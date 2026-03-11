# OpenClaw Agent 与 Session 架构详解

## 一、核心疑问

### Q1: 如何实现 Webchat 和 Discord 的 Session 隔离？
**问题场景**：在使用 OpenClaw 时，发现 Discord 的聊天记录会进入 Webchat 控制台的 session，导致两个渠道的对话混在一起，无法实现独立的对话上下文。

### Q2: Agent 和 Session 的关系是什么？
**困惑点**：不清楚 Agent（智能体）和 Session（会话）之间的层级关系，以及它们如何影响对话隔离。

### Q3: 如何在不同渠道使用不同的 Agent？
**需求**：希望在 Webchat 控制台和 Discord 中使用不同的 Agent，但不知道如何配置和切换。

---

## 二、核心概念解析

### 2.1 Agent（智能体）

**定义**：Agent 是一个"角色定义"或"配置模板"，类似于一个具有特定能力和行为的虚拟助手。

**核心属性**：
- 使用的模型（如 Claude Sonnet 4.5、GPT-5.2）
- Workspace 路径
- 行为规则和 Prompt
- 工具权限

**示例**：
```json
{
  "id": "main",
  "name": "默认助手",
  "model": {
    "primary": "aicodemirror-claude/claude-sonnet-4-5-20250929"
  }
}
```

### 2.2 Session（会话）

**定义**：Session 是一次具体的对话实例，包含完整的对话历史和上下文。

**核心特征**：
- 每个 Session 都关联到一个 Agent
- Session 的唯一标识：`agent:channel:identifier`
- 对话历史存储在独立的 JSONL 文件中

**示例**：
```
session key: agent:main:main
transcript: /Users/xxx/.openclaw/agents/main/sessions/423f8d06-bd54-4b39-9848-722c53610477.jsonl
```

### 2.3 Binding（绑定规则）

**定义**：Binding 定义了哪个 Channel（渠道）使用哪个 Agent 作为默认响应者。

**配置示例**：
```json
{
  "bindings": [
    {
      "agentId": "main",
      "match": {
        "channel": "webchat"
      }
    },
    {
      "agentId": "discord-main",
      "match": {
        "channel": "discord",
        "accountId": "main"
      }
    }
  ]
}
```

### 2.4 关系图

```
Agent (配置模板)
    ↓
  绑定到 Channel (通过 Binding)
    ↓
  创建 Session (对话实例)
```

**关键点**：
1. 一个 Agent 可以有多个 Session（比如多个用户同时对话）
2. 一个 Session 只属于一个 Agent
3. Session 隔离 = 对话历史隔离
4. Agent 隔离 = 更彻底的隔离（包括配置、行为、上下文）

---

## 三、问题根源分析

### 3.1 原始配置（问题状态）

```json
{
  "bindings": [
    {
      "agentId": "main",
      "match": {
        "channel": "webchat"
      }
    },
    {
      "agentId": "main",
      "match": {
        "channel": "discord"
      }
    }
  ]
}
```

**问题**：
- `main` agent 同时绑定了 `webchat` 和 `discord` 两个 channel
- 虽然技术上会创建两个不同的 session，但 OpenClaw 的路由逻辑可能会让同一个 agent 的不同 channel 消息进入同一个 session context
- 导致 Discord 的消息出现在 Webchat 的对话中

### 3.2 架构示意图（问题状态）

```
main agent
  ├─ 绑定 webchat → session: agent:main:webchat-xxx
  └─ 绑定 discord  → session: agent:main:discord-xxx
     ❌ 两个 session 可能共享上下文
```

---

## 四、正确的配置方案

### 4.1 方案 B：创建独立的 Discord Agent（推荐）

**核心思路**：为 Discord 创建一个专门的 Agent（`discord-main`），实现完全独立的 Agent 和 Session。

#### 步骤 1：在 `agents.list` 中添加新 Agent

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "name": "默认助手",
        "model": {
          "primary": "aicodemirror-claude/claude-sonnet-4-5-20250929"
        }
      },
      {
        "id": "discord-main",
        "name": "Discord 助手",
        "model": {
          "primary": "aicodemirror-claude/claude-sonnet-4-5-20250929"
        }
      }
    ]
  }
}
```

#### 步骤 2：修改 Binding 规则

```json
{
  "bindings": [
    {
      "agentId": "main",
      "match": {
        "channel": "webchat"
      }
    },
    {
      "agentId": "discord-main",
      "match": {
        "channel": "discord",
        "accountId": "main"
      }
    }
  ]
}
```

#### 步骤 3：应用配置并重启

使用 OpenClaw 的 `config.patch` 工具：

```javascript
mcp_gateway({
  action: "config.patch",
  note: "已创建独立的 discord-main agent，webchat 和 Discord 现在使用不同的 session。",
  raw: `{
    agents: {
      list: [
        // ... 添加 discord-main
      ]
    },
    bindings: [
      // ... 修改绑定规则
    ]
  }`
})
```

### 4.2 最终架构（修复后）

```
main agent
  └─ 绑定 webchat → session: agent:main:webchat-xxx

discord-main agent
  └─ 绑定 discord → session: agent:discord-main:discord-xxx

✅ 两个完全独立的 agent，各自管理自己的 session
```

---

## 五、Agent 协作机制

### 5.1 跨 Agent 调用：sessions_spawn

**核心能力**：任何 Agent 都可以通过 `sessions_spawn` 调用其他 Agent，实现"AI 团队"协作。

**使用场景**：
- 在 Webchat 中临时调用 `architect` 设计架构
- 在 Discord 中调用 `main` 处理特定任务
- 多个 Agent 并行执行不同任务

**示例**：
```javascript
sessions_spawn({
  runtime: "subagent",
  agentId: "architect",  // 指定其他 agent
  task: "帮我设计一个微服务架构"
})
```

### 5.2 Agent 协作规则

**可用性**：
- ✅ `openclaw.json` 里配置的所有 agent，都可以通过 `sessions_spawn` 调用
- ✅ 不受 channel binding 限制（binding 只影响"默认响应者"）
- ✅ 可以同时派发多个任务给不同 agent

**限制**：
- 避免循环调用（A 调用 B，B 又调用 A）
- Sub-agent 的并发数有限制（默认 `maxConcurrent: 8`）

### 5.3 实际协作示例

```
用户：帮我做一个项目
main agent：好的，我来协调：
   - 用 architect 设计架构
   - 用 code 实现功能
   - 用 reviewer 审查代码
   - 用 scribe 写文档
   (同时派发，并行执行)
```

---

## 六、常见问题 FAQ

### Q1: 执行 `/new` 会发生什么？
**答**：`/new` 命令会在当前 agent 下创建一个全新的 session。
- ✅ 保持在同一个 agent
- ✅ 创建新的 session（全新的对话历史）
- ✅ 之前的 session 会被保存，可以随时切回去

### Q2: 如何在 Webchat 中切换 Agent？
**答**：Webchat 不支持直接切换 agent，但可以通过以下方式：
1. **使用 sub-agent**（推荐）：通过 `sessions_spawn` 临时调用其他 agent
2. **修改配置**：修改 binding 规则，将 webchat 绑定到其他 agent（需要重启）

### Q3: Discord 和 Webchat 可以互相调用对方的 Agent 吗？
**答**：可以！任何 agent 都可以通过 `sessions_spawn` 调用其他 agent，包括：
- `discord-main` 可以调用 `main`
- `main` 可以调用 `discord-main`
- 任何 agent 都可以调用任何其他 agent

### Q4: 如何验证 Session 是否隔离？
**答**：使用 `sessions_list` 工具查看当前活跃的 session：
```javascript
mcp_sessions_list({
  limit: 10,
  messageLimit: 1
})
```
检查不同 channel 的 session key 是否不同。

---

## 七、最佳实践建议

### 7.1 Session 隔离策略

1. **不同渠道使用不同 Agent**：为每个主要渠道（Webchat、Discord、Telegram）创建独立的 agent
2. **共享 Workspace**：所有 agent 共享同一个 workspace（MEMORY.md、AGENTS.md 等），保持知识一致性
3. **定期清理 Session**：使用 `/new` 命令定期清理过长的对话历史

### 7.2 Agent 设计原则

1. **职责明确**：每个 agent 应有明确的职责（如 architect、reviewer、code）
2. **模型匹配**：根据任务类型选择合适的模型（如代码审查用 GPT-5.2，架构设计用 Claude Sonnet）
3. **协作优先**：优先使用 `sessions_spawn` 进行 agent 协作，而不是创建过多独立 agent

### 7.3 配置管理建议

1. **使用 config.patch**：修改配置时使用 `config.patch` 而不是 `config.apply`，避免覆盖整个配置
2. **备份配置文件**：在修改前备份 `openclaw.json`
3. **验证配置**：修改后使用 `openclaw doctor --non-interactive` 验证配置正确性

---

## 八、总结

### 核心要点

1. **Agent = 角色定义**，Session = 对话实例
2. **Binding 决定默认响应者**，但不限制 agent 协作
3. **完全隔离 = 不同 Agent + 不同 Session**
4. **sessions_spawn 是跨 agent 协作的钥匙**

### 配置检查清单

- [ ] 为每个主要渠道创建独立的 agent
- [ ] 修改 bindings 规则，确保每个渠道绑定到正确的 agent
- [ ] 使用 `config.patch` 应用配置并重启
- [ ] 验证 session 隔离是否生效（通过 `sessions_list`）
- [ ] 测试跨 agent 协作功能（通过 `sessions_spawn`）

---

**文档版本**：v1.0  
**最后更新**：2026-03-11  
**适用版本**：OpenClaw 2026.2.17+  
**学城链接**：https://km.sankuai.com/page/2750089650
