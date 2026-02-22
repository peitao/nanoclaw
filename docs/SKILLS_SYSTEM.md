# Skill 和 Tool 支持机制

NanoClaw 的技能系统分为两层：
1. **容器技能** - 内置的工具脚本（agent-browser 等）
2. **Skills Engine** - 高级技能管理系统（安装/卸载/更新）

---

## 一、容器技能（Built-in Tools）

### 1.1 什么是容器技能？

容器技能是预装在容器中的 **Bash 脚本**，Agent 可以直接调用。它们为 Agent 提供浏览器自动化、文件操作等能力。

### 1.2 技能存放位置

```
container/skills/
└── agent-browser/
    └── SKILL.md    # 技能定义文件
```

### 1.3 技能格式（SKILL.md）

```yaml
---
name: agent-browser
description: 浏览网页、交互式 Web 应用、填表、截图...
allowed-tools: Bash(agent-browser:*)  # 允许使用的工具
---

# 技能说明和用法
...
```

**关键字段**：
- `name` - 技能名称
- `description` - 技能描述
- `allowed-tools` - 允许 Agent 使用的工具（限制权限）

### 1.4 技能如何同步到容器

在 `container-runner.ts` 中：

```typescript
// 将 container/skills/ 复制到每个群组的 .claude/skills/
const skillsSrc = path.join(process.cwd(), 'container', 'skills');
const skillsDst = path.join(groupSessionsDir, 'skills');

if (fs.existsSync(skillsSrc)) {
  for (const skillDir of fs.readdirSync(skillsSrc)) {
    const srcDir = path.join(skillsSrc, skillDir);
    const dstDir = path.join(skillsDst, skillDir);
    fs.cpSync(srcDir, dstDir, { recursive: true });
  }
}
```

**流程**：
```
启动容器 → 挂载群组目录 → 复制 skills → Agent 可用
```

### 1.5 已有的容器技能

| 技能 | 作用 |
|------|------|
| `agent-browser` | 浏览器自动化（浏览、填表、截图） |

Agent 使用方式：
```bash
agent-browser open https://google.com
agent-browser snapshot -i
agent-browser click @e1
```

---

## 二、Skills Engine（高级技能系统）

### 2.1 概述

Skills Engine 是一个完整的功能：
- 安装/卸载技能
- 版本管理
- 冲突检测
- 自定义修改（custom patches）
- 状态持久化

### 2.2 目录结构

```
skills-engine/
├── index.ts           # 导出所有功能
├── apply.ts           # 安装技能
├── uninstall.ts       # 卸载技能
├── update.ts          # 更新技能
├── state.ts           # 状态管理
├── manifest.ts        # 技能清单验证
├── types.ts           # 类型定义
├── merge.ts           # Git 合并冲突处理
├── resolution-cache.ts # 解决方案缓存
└── ...其他工具模块
```

### 2.3 技能清单（manifest）

每个技能有一个 `manifest.yaml`：

```yaml
skill: my-skill
version: 1.0.0
description: 添加某个功能
core_version: ">=1.0.0"

adds:              # 添加的文件
  - src/new-feature.ts
  - skills/new-skill/

modifies:          # 修改的文件
  - src/index.ts

conflicts:         # 冲突技能
  - other-skill

depends:           # 依赖技能
  - base-skill

structured:        # 结构化修改
  npm_dependencies:
    lodash: ^4.0.0
  env_additions:
    - NEW_API_KEY
  docker_compose_services:
    redis:
      image: redis:latest

file_ops:          # 文件操作
  - type: rename
    from: old.ts
    to: new.ts

test: npm test     # 测试命令
post_apply:        # 安装后脚本
  - npm run build
```

### 2.4 技能状态（state）

安装的技能状态保存在 `.nanoclaw/state.json`：

```json
{
  "skills_system_version": "1.0.0",
  "core_version": "1.2.0",
  "applied_skills": [
    {
      "name": "my-skill",
      "version": "1.0.0",
      "applied_at": "2024-01-01T00:00:00Z",
      "file_hashes": {
        "src/new-feature.ts": "abc123..."
      }
    }
  ],
  "custom_modifications": [],
  "path_remap": {}
}
```

### 2.5 安装流程（applySkill）

```typescript
async function applySkill(skillDir: string): Promise<ApplyResult> {
  const manifest = readManifest(skillDir);  // 1. 读取清单
  
  // 2. 预检查
  checkSystemVersion(manifest);      // 系统版本兼容？
  checkCoreVersion(manifest);         // 核心版本兼容？
  checkDependencies(manifest);       // 依赖满足？
  checkConflicts(manifest);          // 有冲突？
  
  // 3. 检测 drift（文件是否被修改过）
  for (const relPath of manifest.modifies) {
    if (hasDrift(resolvedPath)) {
      driftFiles.push(relPath);
    }
  }
  
  // 4. 创建备份
  if (driftFiles.length > 0) {
    createBackup();
  }
  
  // 5. 执行文件操作
  executeFileOps(manifest.file_ops);
  
  // 6. 合并修改（处理冲突）
  for (const relPath of manifest.modifies) {
    mergeFile(relPath);  // 使用 git merge / rerere
  }
  
  // 7. 结构化修改
  if (manifest.structured) {
    mergeNpmDependencies(manifest.structured.npm_dependencies);
    mergeEnvAdditions(manifest.structured.env_additions);
    runNpmInstall();
  }
  
  // 8. 记录状态
  recordSkillApplication(manifest, fileHashes);
  
  // 9. 运行测试
  if (manifest.test) {
    execSync(manifest.test);
  }
  
  return { success: true, skill, version };
}
```

### 2.6 核心功能

| 功能 | 函数 | 说明 |
|------|------|------|
| 安装 | `applySkill()` | 安装技能，处理合并冲突 |
| 卸载 | `uninstallSkill()` | 卸载技能，回滚更改 |
| 更新 | `previewUpdate()` | 预览更新 |
| 更新 | `applyUpdate()` | 应用更新 |
| 验证 | `checkConflicts()` | 检测冲突 |
| 备份 | `createBackup()` | 创建备份 |
| 恢复 | `restoreBackup()` | 恢复备份 |

---

## 三、工具调用机制

### 3.1 Agent 如何使用工具

Agent 运行在容器中，可以直接调用 Bash 命令：

```bash
# 调用浏览器技能
agent-browser open https://example.com

# 调用文件系统技能（如果有）
ls /workspace/group
```

### 3.2 权限控制

通过 `allowed-tools` 限制 Agent 能用的工具：

```yaml
allowed-tools: 
  - Bash(agent-browser:*)   # 只能调用 agent-browser
  - Bash(readonly:*)        # 只能读文件
  - Bash(npm:*)             # 只能运行 npm
```

### 3.3 工具来源

1. **容器内置** - `container/skills/` 下的脚本
2. **系统命令** - 容器中的 npm、git、node 等
3. **群组目录** - `/workspace/group/` 下的文件

---

## 四、整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    NanoClaw 主进程                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Skills Engine (skills-engine/)          │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌────────┐ │   │
│  │  │ apply   │  │uninstall│  │ update  │  │ state  │ │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └────────┘ │   │
│  └─────────────────────────────────────────────────────┘   │
└───────────────────────┬─────────────────────────────────────┘
                        │ 启动容器
                        ↓
┌─────────────────────────────────────────────────────────────┐
│                    容器内环境                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              .claude/skills/                         │   │
│  │  ┌─────────────┐  ┌────────────────────────────┐    │   │
│  │  │agent-browser│  │  (其他技能)                │    │   │
│  │  │   SKILL.md  │  │      SKILL.md             │    │   │
│  │  └─────────────┘  └────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Claude Code Agent                                  │   │
│  │  - 读取 SKILL.md 了解技能用法                        │   │
│  │  - 调用 agent-browser 等命令                        │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 五、总结

| 层级 | 位置 | 作用 | 示例 |
|------|------|------|------|
| **容器技能** | `container/skills/` | 预装工具脚本 | `agent-browser` |
| **Skills Engine** | `skills-engine/` | 高级包管理 | 安装/卸载/更新 |
| **权限控制** | `allowed-tools` | 限制 Agent 能力 | 只允许特定命令 |

**调用链**：
```
用户消息 → index.ts → container-runner.ts → 启动容器 
→ Agent 读取 skills → 调用 agent-browser 命令 → 执行任务
```

这样 Agent 就能在安全的容器环境中，为你完成网页浏览、文件操作等各种任务！
