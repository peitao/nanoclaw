# NanoClaw 架构文档

## 一、项目概述

**NanoClaw** 是一个轻量级的个人 AI 助手，通过 WhatsApp 与你对话，Agent 在隔离的 Linux 容器中安全运行。

### 核心特性
- 📱 WhatsApp 消息交互（@Andy 触发）
- 🐳 容器隔离执行（安全运行 Bash 命令）
- 👥 群组隔离（每个群独立文件系统）
- ⏰ 定时任务（cron 表达式）
- 🌐 网页搜索和浏览器自动化

---

## 二、核心模块

### 2.1 消息循环 (index.ts)

**职责**：主编排器，状态管理、消息循环、Agent 调用

```
WhatsApp → 消息轮询 → 消息处理 → Agent 容器 → 响应发送
```

**主要流程**：
1. `startMessageLoop()` - 持续轮询新消息
2. `processGroupMessages()` - 处理群组消息
3. `runAgent()` - 启动容器运行 Agent

**关键状态**：
- `lastTimestamp` - 最后处理的消息时间戳
- `lastAgentTimestamp` - 每个群组最后 Agent 处理的时间戳
- `sessions` - 每个群组的会话 ID（保持对话上下文）
- `registeredGroups` - 已注册的群组

---

### 2.2 容器管理 (container-runner.ts)

**职责**：启动 Agent 容器并挂载目录

**核心函数**：
- `runContainerAgent()` - 启动容器运行 Agent
- `buildVolumeMounts()` - 构建卷挂载
- `writeGroupsSnapshot()` - 写群组快照到容器
- `writeTasksSnapshot()` - 写任务快照到容器

**卷挂载规则**：
| 群组 | 挂载 |
|------|------|
| main | 项目根目录 + main 组目录 |
| 其他 | 组目录 (只读 global) |

**隔离机制**：
- 每个群组有独立的工作目录 (`/workspace/group`)
- main 群组可访问所有群组信息
- 其他群组只能看到自己的目录

---

### 2.3 消息通道 (channels/whatsapp.ts)

**职责**：WhatsApp 连接、认证、收发消息

**核心类**：`WhatsAppChannel`

**主要方法**：
- `connect()` - 连接 WhatsApp
- `sendMessage(jid, text)` - 发送消息
- `onMessage` - 消息回调

**认证方式**：WhatsApp QR 码认证

---

### 2.4 定时任务 (task-scheduler.ts)

**职责**：定时任务调度

**支持类型**：
- `cron` - Cron 表达式
- `interval` - 间隔时间（毫秒）
- `once` - 一次性任务

**核心函数**：
- `startSchedulerLoop()` - 启动调度循环
- `runTask()` - 执行任务

---

### 2.5 群组队列 (group-queue.ts)

**职责**：管理群组消息队列，避免并发冲突

**核心功能**：
- 消息入队/出队
- 控制并发（同时只处理一个群组）
- 管理容器 stdin/stdout

---

### 2.6 数据库 (db.ts)

**职责**：SQLite 数据库操作

**主要表**：
- `messages` - 消息存储
- `chats` - 聊天元数据
- `sessions` - 会话状态
- `tasks` - 定时任务
- `registered_groups` - 已注册群组

---

### 2.7 IPC 监视 (ipc.ts)

**职责**：通过文件系统触发 Agent 任务

**工作方式**：
1. 监视 `data/ipc/` 目录
2. 写入文件触发任务
3. Agent 读取并执行

---

## 三、数据流

### 3.1 消息处理流程

```
1. WhatsApp 收到消息
      ↓
2. storeMessage() 存入数据库
      ↓
3. startMessageLoop() 检测到新消息
      ↓
4. 检查是否需要触发词（非 main 群组）
      ↓
5. 获取未处理消息（从 lastAgentTimestamp）
      ↓
6. 格式化消息 formatMessages()
      ↓
7. GroupQueue 判断是否已有活跃容器
      ↓
   ├─ 有容器 → 发送消息到 stdin
   └─ 无容器 → 启动新容器 runAgent()
      ↓
8. 容器运行 Agent（Claude Code）
      ↓
9. 捕获输出，通过 WhatsApp 发送
      ↓
10. 保存会话状态和游标
```

### 3.2 定时任务流程

```
1. startSchedulerLoop() 轮询检查任务
      ↓
2. getDueTasks() 获取到期任务
      ↓
3. 对每个任务：
   ├─ 写任务快照到组目录
   ├─ 启动容器
   ├─ 运行 Agent
   └─ 发送结果到群组
      ↓
4. 更新任务状态和下次运行时间
```

---

## 四、容器隔离机制

### 4.1 为什么用容器？

- **安全**：Bash 命令只在容器内执行，不影响宿主机
- **隔离**：每个群组独立文件系统，互不干扰
- **状态**：每个群组有独立会话，保持对话上下文

### 4.2 容器内能做什么？

- 搜索网页、抓取内容
- 执行代码（npm, python, etc.）
- 读写群组目录文件
- 访问互联网

### 4.3 容器不能做什么？

- 访问宿主机的敏感目录
- 影响其他群组的容器

---

## 五、关键配置文件

### 5.1 环境变量 (.env)

```
ANTHROPIC_API_KEY=sk-...
WHATSAPP_SESSION_DIR=./store
DATA_DIR=./data
GROUPS_DIR=./groups
```

### 5.2 群组目录结构

```
groups/
├── main/           # 主群组（管理频道）
│   ├── CLAUDE.md   # 记忆文件
│   └── logs/
├── global/         # 全局记忆（所有群组可读）
└── <group-folder>/ # 各群组独立目录
    ├── CLAUDE.md
    └── logs/
```

---

## 六、核心文件索引

| 文件 | 职责 |
|------|------|
| `src/index.ts` | 主编排器：状态管理、消息循环、Agent 调用 |
| `src/channels/whatsapp.ts` | WhatsApp 连接、认证、收发消息 |
| `src/container-runner.ts` | 启动 Agent 容器并挂载目录 |
| `src/container-runtime.ts` | 容器运行时管理（启动/停止） |
| `src/db.ts` | SQLite 数据库操作 |
| `src/task-scheduler.ts` | 定时任务调度 |
| `src/group-queue.ts` | 群组消息队列管理 |
| `src/ipc.ts` | IPC 文件监视与任务处理 |
| `src/router.ts` | 消息格式化与路由 |
| `src/types.ts` | 类型定义 |
| `src/config.ts` | 配置常量 |

---

## 七、设计哲学

- **小到能看懂** — 整个代码库几分钟就能读完
- **安全靠 OS 级隔离** — 不是靠应用层权限检查，而是靠容器
- **为单个用户而建** — 不是通用框架，而是为个人需求量身定做
- **AI 原生** — 没有安装向导，一切靠 Claude Code 引导
- **技能而非功能** — 社区贡献的是 Skill 脚本，用户按需执行
