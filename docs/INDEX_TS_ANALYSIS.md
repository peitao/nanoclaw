# index.ts 深度解析

这是 NanoClaw 的**主编排器**，是整个系统的核心。它负责消息循环、Agent 调用、状态管理。

---

## 一、文件概览

```
index.ts = 主编排器
├── 消息循环 (startMessageLoop)
├── 消息处理 (processGroupMessages)
├── Agent 运行 (runAgent)
├── 状态管理 (loadState/saveState)
└── 启动入口 (main)
```

---

## 二、全局变量（状态）

```typescript
let lastTimestamp = '';              // 最后处理的消息时间戳（所有群组）
let sessions: Record<string, string> = {};  // 每个群组的会话 ID
let registeredGroups: Record<string, RegisteredGroup> = {};  // 已注册群组
let lastAgentTimestamp: Record<string, string> = {};  // 每个群组最后处理时间
let messageLoopRunning = false;      // 防止重复启动消息循环

let whatsapp: WhatsAppChannel;       // WhatsApp 连接实例
const channels: Channel[] = [];     // 消息通道列表（可扩展多平台）
const queue = new GroupQueue();     // 群组队列（控制并发）
```

**为什么需要这些状态？**
- `lastTimestamp` - 标记已读消息，避免重复处理
- `sessions` - 保持对话上下文，让 Agent 记得之前说了什么
- `registeredGroups` - 知道哪些群组需要响应
- `lastAgentTimestamp` - 每个群组独立游标，支持多群组并行

---

## 三、导入模块

```typescript
import { WhatsAppChannel } from './channels/whatsapp.js';    // WhatsApp 消息通道
import { runContainerAgent, ... } from './container-runner.js'; // 容器管理
import { cleanupOrphans, ensureContainerRuntimeRunning } from './container-runtime.js'; // 容器运行时
import { getAllChats, getNewMessages, ... } from './db.js';  // 数据库操作
import { GroupQueue } from './group-queue.js';              // 消息队列
import { startIpcWatcher } from './ipc.ts';                 // IPC 文件监视
import { findChannel, formatMessages } from './router.ts';   // 消息路由
import { startSchedulerLoop } from './task-scheduler.ts';    // 定时任务
```

---

## 四、状态管理函数

### 4.1 loadState() - 加载状态

```typescript
function loadState(): void {
  // 从数据库恢复上次运行的状态
  lastTimestamp = getRouterState('last_timestamp') || '';
  
  // 解析每个群组的 Agent 处理游标
  const agentTs = getRouterState('last_agent_timestamp');
  lastAgentTimestamp = agentTs ? JSON.parse(agentTs) : {};
  
  // 恢复会话和已注册群组
  sessions = getAllSessions();
  registeredGroups = getAllRegisteredGroups();
}
```

**作用**：程序启动时恢复之前的会话状态，实现持久化。

### 4.2 saveState() - 保存状态

```typescript
function saveState(): void {
  setRouterState('last_timestamp', lastTimestamp);
  setRouterState('last_agent_timestamp', JSON.stringify(lastAgentTimestamp));
}
```

**作用**：每次处理消息后保存状态，确保崩溃后能恢复。

### 4.3 registerGroup() - 注册群组

```typescript
function registerGroup(jid: string, group: RegisteredGroup): void {
  registeredGroups[jid] = group;
  setRegisteredGroup(jid, group);
  
  // 为群组创建目录
  const groupDir = path.join(DATA_DIR, '..', 'groups', group.folder);
  fs.mkdirSync(path.join(groupDir, 'logs'), { recursive: true });
}
```

**作用**：将一个新的群组纳入管理，创建其专属目录。

---

## 五、核心处理函数

### 5.1 processGroupMessages() - 处理群组消息

这是**最核心的函数**，处理一个群组的所有待处理消息。

```typescript
async function processGroupMessages(chatJid: string): Promise<boolean> {
  // 1. 获取群组信息
  const group = registeredGroups[chatJid];
  if (!group) return true;
  
  // 2. 找到对应的消息通道
  const channel = findChannel(channels, chatJid);
  
  // 3. 判断是否为主群组
  const isMainGroup = group.folder === MAIN_GROUP_FOLDER;
  
  // 4. 获取未处理的消息（从上次游标到现在）
  const sinceTimestamp = lastAgentTimestamp[chatJid] || '';
  const missedMessages = getMessagesSince(chatJid, sinceTimestamp, ASSISTANT_NAME);
  
  if (missedMessages.length === 0) return true;
  
  // 5. 非主群组需要触发词
  if (!isMainGroup && group.requiresTrigger !== false) {
    const hasTrigger = missedMessages.some(m => TRIGGER_PATTERN.test(m.content.trim()));
    if (!hasTrigger) return true;  // 没有触发词，不处理
  }
  
  // 6. 格式化消息为 prompt
  const prompt = formatMessages(missedMessages);
  
  // 7. 保存旧游标（出错时回滚）
  const previousCursor = lastAgentTimestamp[chatJid];
  lastAgentTimestamp[chatJid] = missedMessages[missedMessages.length - 1].timestamp;
  saveState();
  
  // 8. 设置空闲超时（长时间无输出则关闭 stdin）
  const resetIdleTimer = () => {
    idleTimer = setTimeout(() => {
      queue.closeStdin(chatJid);  // 关闭容器 stdin
    }, IDLE_TIMEOUT);
  };
  
  // 9. 运行 Agent
  const output = await runAgent(group, prompt, chatJid, async (result) => {
    // 10. 处理流式输出
    if (result.result) {
      // 移除 <internal>...</internal> 内部推理内容
      const text = raw.replace(/<internal>[\s\S]*?<\/internal>/g, '').trim();
      await channel.sendMessage(chatJid, text);  // 发送回复
      outputSentToUser = true;
      resetIdleTimer();
    }
  });
  
  // 11. 错误处理：已发送消息则不回滚，未发送则回滚重试
  if (output === 'error' || hadError) {
    if (!outputSentToUser) {
      lastAgentTimestamp[chatJid] = previousCursor;  // 回滚游标
      saveState();
    }
    return false;
  }
  
  return true;
}
```

**流程图**：
```
收到消息 → 获取未处理消息 → 检查触发词 → 格式化 → 运行 Agent → 发送回复 → 保存游标
```

### 5.2 runAgent() - 运行 Agent

```typescript
async function runAgent(
  group: RegisteredGroup,   // 群组信息
  prompt: string,           // 格式化后的消息
  chatJid: string,         // 群组 JID
  onOutput?: (output: ContainerOutput) => Promise<void>  // 输出回调
): Promise<'success' | 'error'> {
  
  const isMain = group.folder === MAIN_GROUP_FOLDER;
  const sessionId = sessions[group.folder];  // 获取已有会话
  
  // 1. 写任务快照（让容器知道有哪些定时任务）
  const tasks = getAllTasks();
  writeTasksSnapshot(group.folder, isMain, tasks);
  
  // 2. 写可用群组快照（让 main 群组知道所有群组）
  const availableGroups = getAvailableGroups();
  writeGroupsSnapshot(group.folder, isMain, availableGroups, ...);
  
  // 3. 包装回调，追踪新会话 ID
  const wrappedOnOutput = onOutput ? async (output) => {
    if (output.newSessionId) {
      sessions[group.folder] = output.newSessionId;
      setSession(group.folder, output.newSessionId);
    }
    await onOutput(output);
  } : undefined;
  
  // 4. 调用容器运行 Agent
  const output = await runContainerAgent(group, {
    prompt,
    sessionId,
    groupFolder: group.folder,
    chatJid,
    isMain,
  }, (proc, containerName) => {
    queue.registerProcess(chatJid, proc, containerName, group.folder);
  }, wrappedOnOutput);
  
  // 5. 保存新会话 ID
  if (output.newSessionId) {
    sessions[group.folder] = output.newSessionId;
    setSession(group.folder, output.newSessionId);
  }
  
  return output.status === 'error' ? 'error' : 'success';
}
```

**关键点**：
- **会话保持**：`sessionId` 让 Agent 记得之前的对话
- **快照机制**：通过文件共享任务和群组信息给容器
- **流式输出**：通过回调实时发送 Agent 的输出

---

## 六、消息循环

### 6.1 startMessageLoop() - 消息轮询主循环

```typescript
async function startMessageLoop(): Promise<void> {
  if (messageLoopRunning) return;  // 防止重复启动
  messageLoopRunning = true;
  
  logger.info(`NanoClaw running (trigger: @${ASSISTANT_NAME})`);
  
  while (true) {  // 无限循环
    try {
      // 1. 获取所有已注册群组的 JID
      const jids = Object.keys(registeredGroups);
      
      // 2. 查询新消息（从 lastTimestamp 之后）
      const { messages, newTimestamp } = getNewMessages(jids, lastTimestamp, ASSISTANT_NAME);
      
      if (messages.length > 0) {
        // 3. 立即推进已读游标
        lastTimestamp = newTimestamp;
        saveState();
        
        // 4. 按群组分组消息
        const messagesByGroup = new Map<string, NewMessage[]>();
        for (const msg of messages) {
          messagesByGroup.get(msg.chat_jid)?.push(msg) || messagesByGroup.set(msg.chat_jid, [msg]);
        }
        
        // 5. 处理每个群组的消息
        for (const [chatJid, groupMessages] of messagesByGroup) {
          const group = registeredGroups[chatJid];
          if (!group) continue;
          
          // 6. 检查是否需要触发词
          const needsTrigger = !isMainGroup && group.requiresTrigger !== false;
          if (needsTrigger) {
            const hasTrigger = groupMessages.some(m => TRIGGER_PATTERN.test(m.content.trim()));
            if (!hasTrigger) continue;  // 跳过非触发消息
          }
          
          // 7. 获取所有未处理的消息（包含非触发消息作为上下文）
          const allPending = getMessagesSince(chatJid, lastAgentTimestamp[chatJid] || '', ASSISTANT_NAME);
          const messagesToSend = allPending.length > 0 ? allPending : groupMessages;
          
          // 8. 尝试发送到现有容器，或创建新容器
          if (queue.sendMessage(chatJid, formatted)) {
            // 发送到现有容器
            lastAgentTimestamp[chatJid] = messagesToSend[messagesToSend.length - 1].timestamp;
            saveState();
            channel.setTyping?.(chatJid, true);  // 显示"正在输入"
          } else {
            // 队列创建新容器
            queue.enqueueMessageCheck(chatJid);
          }
        }
      }
    } catch (err) {
      logger.error({ err }, 'Error in message loop');
    }
    
    // 9. 等待下次轮询
    await new Promise(resolve => setTimeout(resolve, POLL_INTERVAL));
  }
}
```

**核心逻辑**：
1. **轮询**：`POLL_INTERVAL` 间隔检查新消息
2. **去重**：按群组分，避免重复处理
3. **触发词**：非主群组需要 @Andy 才会响应
4. **上下文**：即使没有触发词，消息也会被积累作为上下文
5. **队列**：有活跃容器就直接发消息，没有就创建新的

### 6.2 recoverPendingMessages() - 启动恢复

```typescript
function recoverPendingMessages(): void {
  // 检查每个群组是否有未处理的消息
  // 崩溃恢复：程序崩溃时可能消息已读但未处理
  for (const [chatJid, group] of Object.entries(registeredGroups)) {
    const pending = getMessagesSince(chatJid, lastAgentTimestamp[chatJid] || '', ASSISTANT_NAME);
    if (pending.length > 0) {
      queue.enqueueMessageCheck(chatJid);  // 加入队列处理
    }
  }
}
```

---

## 七、启动入口 main()

```typescript
async function main(): Promise<void> {
  // 1. 确保容器运行时运行
  ensureContainerSystemRunning();
  
  // 2. 初始化数据库
  initDatabase();
  loadState();
  
  // 3. 设置优雅关闭
  const shutdown = async (signal) => {
    await queue.shutdown(10000);  // 等待队列处理完成
    for (const ch of channels) await ch.disconnect();
    process.exit(0);
  };
  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT', () => shutdown('SIGINT'));
  
  // 4. 创建消息通道
  const channelOpts = {
    onMessage: (jid, msg) => storeMessage(msg),  // 收到消息存入数据库
    onChatMetadata: (jid, ts, name, channel, isGroup) => storeChatMetadata(...),
    registeredGroups: () => registeredGroups,
  };
  
  whatsapp = new WhatsAppChannel(channelOpts);
  channels.push(whatsapp);
  await whatsapp.connect();
  
  // 5. 启动定时任务调度器
  startSchedulerLoop({
    registeredGroups: () => registeredGroups,
    getSessions: () => sessions,
    queue,
    onProcess: (groupJid, proc, containerName, groupFolder) => 
      queue.registerProcess(groupJid, proc, containerName, groupFolder),
    sendMessage: async (jid, text) => {
      const channel = findChannel(channels, jid);
      if (channel) await channel.sendMessage(jid, formatOutbound(text));
    },
  });
  
  // 6. 启动 IPC 监视器
  startIpcWatcher({
    sendMessage: (jid, text) => findChannel(channels, jid).sendMessage(jid, text),
    registeredGroups: () => registeredGroups,
    registerGroup,
    syncGroupMetadata: () => whatsapp?.syncGroupMetadata(),
    getAvailableGroups,
    writeGroupsSnapshot,
  });
  
  // 7. 设置队列处理函数
  queue.setProcessMessagesFn(processGroupMessages);
  
  // 8. 恢复未处理消息
  recoverPendingMessages();
  
  // 9. 启动消息循环
  startMessageLoop();
}
```

**启动顺序**：
```
容器运行时 → 数据库 → 消息通道 → 调度器 → IPC → 消息循环
```

---

## 八、数据流全景图

```
┌─────────────────────────────────────────────────────────────┐
│                        WhatsApp                             │
└─────────────────────┬───────────────────────────────────────┘
                      │ 收到消息
                      ↓
┌─────────────────────────────────────────────────────────────┐
│                    storeMessage()                            │
│                  存入 SQLite 数据库                          │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│                 startMessageLoop()                          │
│              轮询检测新消息                                  │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ 1. getNewMessages() 查询新消息                          ││
│  │ 2. 按群组分组                                            ││
│  │ 3. 检查触发词                                            ││
│  │ 4. 获取历史消息（作为上下文）                             ││
│  │ 5. 发送到队列或创建新容器                                 ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│              processGroupMessages()                          │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ 1. getMessagesSince() 获取未处理消息                    ││
│  │ 2. formatMessages() 格式化 prompt                       ││
│  │ 3. runAgent() 启动容器                                  ││
│  │ 4. 捕获流式输出                                          ││
│  │ 5. channel.sendMessage() 发送回复                       ││
│  │ 6. 保存游标 lastAgentTimestamp                         ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│                    runAgent()                               │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ 1. writeTasksSnapshot() 写任务快照                       ││
│  │ 2. writeGroupsSnapshot() 写群组快照                     ││
│  │ 3. runContainerAgent() 启动容器运行 Claude              ││
│  │ 4. 返回结果和新的 sessionId                             ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

---

## 九、关键设计决策

### 9.1 为什么要用游标（timestamp）而不是已读标记？

```typescript
lastAgentTimestamp[chatJid] = missedMessages[missedMessages.length - 1].timestamp;
```

- **支持重试**：出错时可以回滚到之前的游标重新处理
- **增量处理**：只处理新增消息，不用每次全量扫描
- **崩溃恢复**：通过对比数据库中的消息和游标位置，发现遗漏

### 9.2 为什么要用 GroupQueue？

```typescript
if (queue.sendMessage(chatJid, formatted)) {
  // 发送到现有容器
} else {
  // 创建新容器
}
```

- **复用容器**：群组可能连续发多条消息，复用容器保持上下文
- **控制并发**：同时只处理一个群组，避免资源竞争
- **空闲超时**：超时后关闭 stdin，让容器自动结束

### 9.3 为什么要用快照文件？

```typescript
writeTasksSnapshot(group.folder, isMain, tasks);
writeGroupsSnapshot(group.folder, isMain, availableGroups, ...);
```

- **容器隔离**：容器内无法直接访问主进程的内存数据
- **文件共享**：通过共享目录让容器读取任务和群组信息
- **权限控制**：main 群组可以看到所有群组，其他群组只能看到自己的

---

## 十、完整变量清单

| 变量 | 类型 | 作用 |
|------|------|------|
| `lastTimestamp` | string | 全局最后已读消息时间戳 |
| `sessions` | Record | 每个群组的会话 ID |
| `registeredGroups` | Record | 已注册的群组信息 |
| `lastAgentTimestamp` | Record | 每个群组的 Agent 处理游标 |
| `messageLoopRunning` | boolean | 防止重复启动消息循环 |
| `whatsapp` | WhatsAppChannel | WhatsApp 连接实例 |
| `channels` | Channel[] | 消息通道列表 |
| `queue` | GroupQueue | 群组消息队列 |

---

## 十一、总结

index.ts 是整个 NanoClaw 的大脑：

1. **消息循环** - 持续轮询 WhatsApp 新消息
2. **触发机制** - 非主群组需要 @Andy 触发
3. **Agent 运行** - 在隔离容器中运行 Claude
4. **状态持久化** - 保存会话和游标，支持崩溃恢复
5. **队列管理** - 控制并发，复用容器，保持上下文

理解这个文件，就理解了 NanoClaw 的核心逻辑！
