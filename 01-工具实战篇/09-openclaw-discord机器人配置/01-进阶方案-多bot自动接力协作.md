# 进阶方案：OpenClaw Discord 多 Bot 自动接力协作（支持迭代反馈）


<!-- TOC -->

# OpenClaw Discord 多 Bot 接力配置教程

## 背景

在 Discord 中运行多个 OpenClaw Bot 时，如果不做消息路由控制，所有 Bot 都会同时响应用户消息，导致混乱。本教程展示如何通过 **OpenClaw Hook** 实现多 Bot 接力模式，并支持 **Architect ↔ Reviewer 迭代反馈循环**。

## 目标架构

实现以下消息流：

```Text
用户 → Architect → Reviewer → 判断：
                              ├─ 需要修改 → Architect（修改后再给 Reviewer）
                              └─ 通过评审 → Code Bot / Scribe Bot
```

**路由规则：**


1. 用户消息 → 只有 **Architect** Bot 响应
2. Architect Bot 消息 → 只有 **Reviewer** Bot 响应
3. Reviewer Bot 消息 → 根据内容判断：

  - **包含修改反馈关键词**（需要修改/建议调整/有问题/需要优化等）→ **Architect** Bot 响应（进入修改循环）
  - **包含通过关键词**（通过/没问题/LGTM/可以等）→ 根据是否包含代码关键词：

    - 包含代码关键词 → **Code** Bot 响应
    - 不包含代码关键词 → **Scribe** Bot 响应



**打断机制：**


- 用户可以通过 @提及任意 Bot 来打断接力链，让该 Bot 直接响应

**迭代反馈示例：**

```Text
用户: "帮我设计一个用户登录系统"
  ↓
Architect: "我设计了以下架构：..."
  ↓
Reviewer: "建议调整数据库设计，需要添加密码加密"
  ↓
Architect: "已修改，增加了 bcrypt 加密..."
  ↓
Reviewer: "通过，可以开始实现代码了"
  ↓
Code: "开始编写登录系统代码..."
```


---

## 配置步骤

### 1. 创建 Hook 目录

```Bash
mkdir -p ~/.openclaw/hooks/discord-relay
```

### 2. 创建 HOOK.md

创建文件 `~/.openclaw/hooks/discord-relay/HOOK.md`：

```Markdown
---
name: discord-relay
description: "Discord multi-bot relay orchestration with review loop"
metadata:
  {
    "openclaw":
      {
        "emoji": "🔗",
        "events": ,
        "requires": { "channels":  },
        "install": ,
      },
  }
---

# Discord Multi-Bot Relay Hook

## Purpose

Orchestrate multiple Discord bots in a relay pattern with iterative review loop:
1. User → Architect Bot
2. Architect Bot → Reviewer Bot  
3. Reviewer Bot → Architect Bot (if needs revision) OR Code/Scribe Bot (if approved)

## Interrupt Mechanism

If a user explicitly @mentions a bot, that bot can respond immediately (breaks the relay chain).

## Configuration

None required. The hook automatically detects bot names and routes messages accordingly.

## How It Works

### Rule 1: User Messages
- Only the **Architect** bot responds to user messages
- All other bots are blocked

### Rule 2: Architect Bot Messages
- Only the **Reviewer** bot responds to Architect's messages
- All other bots are blocked

### Rule 3: Reviewer Bot Messages
- If message contains **revision keywords** (需要修改/建议调整/有问题/需要优化/请修改/需要改进/可以改进/建议):
  - **Architect** bot responds (enters revision loop)
- If message contains **approval keywords** (通过/没问题/可以/LGTM/looks good/approved/同意/确认):
  - Check for code keywords (代码/实现/开发/编程/code/implement/develop):
    - If yes: **Code** bot responds
    - If no: **Scribe** bot responds

### Override: @Mentions
- If a user @mentions any bot directly, that bot can respond immediately
- This allows manual intervention in the relay chain
```

**关键点：**


- `events: ` 指定 Hook 监听入站消息事件
- `requires: { "channels":  }` 限制只在 Discord 渠道生效
- YAML frontmatter 格式是必须的

### 3. 创建 handler.ts

创建文件 `~/.openclaw/hooks/discord-relay/handler.ts`：

```Typescript
/**
 * Discord Multi-Bot Relay Handler with Review Loop
 */

export default async function handler(context) {
  const { event, message, sender, agentId, mentions, logger } = context;
  
  logger.info(` Processing message from ${sender?.name || 'unknown'} for agent ${agentId}`);
  
  // 如果用户明确 @ 了某个 Bot，允许该 Bot 响应（打断机制）
  if (mentions && mentions.includes(agentId)) {
    logger.info(` User mentioned ${agentId}, allowing response`);
    return { allow: true };
  }
  
  // 获取发送者信息
  const senderName = sender?.name || sender?.username || '';
  const senderIsBot = sender?.bot || false;
  
  // 规则 1: 用户消息 → 只有 Architect 响应
  if (!senderIsBot) {
    if (agentId === 'architect') {
      logger.info(` User message → Architect responds`);
      return { allow: true };
    } else {
      logger.info(` User message → Block ${agentId}`);
      return { allow: false };
    }
  }
  
  // 规则 2: Architect Bot 消息 → 只有 Reviewer 响应
  if (senderName.includes('Architect')) {
    if (agentId === 'reviewer') {
      logger.info(` Architect message → Reviewer responds`);
      return { allow: true };
    } else {
      logger.info(` Architect message → Block ${agentId}`);
      return { allow: false };
    }
  }
  
  // 规则 3: Reviewer Bot 消息 → 判断是反馈修改还是通过评审
  if (senderName.includes('Reviewer')) {
    const messageText = message?.content || message?.text || '';
    
    // 检测是否需要 Architect 修改（包含修改、调整、优化等关键词）
    const needsRevision = /需要修改|建议调整|需要优化|请修改|需要改进|有问题|不够|可以改进|建议|修改建议/i.test(messageText);
    
    if (needsRevision && agentId === 'architect') {
      logger.info(` Reviewer feedback → Architect revises`);
      return { allow: true };
    }
    
    // 检测是否通过评审（包含通过、可以、没问题等关键词）
    const approved = /通过|没问题|可以|LGTM|looks good|approved|同意|确认/i.test(messageText);
    
    if (approved) {
      // 根据内容决定 Code 或 Scribe
      const needsCode = /代码|实现|开发|编程|code|implement|develop/i.test(messageText);
      
      if (needsCode && agentId === 'code') {
        logger.info(` Reviewer approved (needs code) → Code responds`);
        return { allow: true };
      } else if (!needsCode && agentId === 'scribe') {
        logger.info(` Reviewer approved (no code) → Scribe responds`);
        return { allow: true };
      }
    }
    
    // 默认：如果 Reviewer 的消息既不是反馈也不是通过，阻止所有 Bot
    logger.info(` Reviewer message (unclear intent) → Block ${agentId}`);
    return { allow: false };
  }
  
  // 默认：阻止响应（避免循环）
  logger.info(` Default block for ${agentId}`);
  return { allow: false };
}
```

**关键点：**


- Handler 必须返回 `{ allow: true }` 或 `{ allow: false }` 来控制消息是否传递给 Agent
- `context` 包含 `message`, `sender`, `agentId`, `mentions`, `logger` 等字段
- 使用 TypeScript 语法（OpenClaw 通过 jiti 自动编译）
- **新增：** 支持 Reviewer → Architect 反馈循环

### 4. 验证 Hook 配置

```Bash
openclaw hooks list
```

应该看到：

```Text
✓ ready  │ 🔗 discord-relay       │ Discord multi-bot relay orchestration │ openclaw-managed
```

### 5. 重启 Gateway

```Bash
openclaw gateway restart
```

### 6. 检查日志确认 Hook 已注册

```Bash
tail -n 100 ~/.openclaw/logs/gateway.log | grep discord-relay
```

应该看到：

```Text
 Registered hook: discord-relay -> message:inbound
```


---

## 验证测试

### 测试场景 1：用户消息


1. 在 Discord 频道发送消息："帮我分析一下这个项目"
2. **预期结果：** 只有 Architect Bot 响应

### 测试场景 2：Architect → Reviewer


1. Architect Bot 回复后
2. **预期结果：** 只有 Reviewer Bot 响应

### 测试场景 3：Reviewer 反馈修改 → Architect


1. Reviewer Bot 回复："建议调整架构设计，需要优化性能"
2. **预期结果：** Architect Bot 响应并修改方案
3. 修改后 Reviewer 再次评审（可以多次循环）

### 测试场景 4：Reviewer 通过 → Code Bot


1. Reviewer Bot 回复："通过，可以开始实现代码了"
2. **预期结果：** 只有 Code Bot 响应

### 测试场景 5：Reviewer 通过 → Scribe Bot


1. Reviewer Bot 回复："没问题，可以写文档了"
2. **预期结果：** 只有 Scribe Bot 响应

### 测试场景 6：@提及打断


1. 用户发送："@Code Bot 帮我写个函数"
2. **预期结果：** Code Bot 直接响应（跳过 Architect 和 Reviewer）

### 测试场景 7：多轮迭代

```Text
用户: "设计一个用户系统"
  ↓
Architect: "设计方案 v1..."
  ↓
Reviewer: "需要修改：缺少权限管理"
  ↓
Architect: "设计方案 v2（增加权限）..."
  ↓
Reviewer: "建议调整：权限粒度太粗"
  ↓
Architect: "设计方案 v3（细化权限）..."
  ↓
Reviewer: "通过，可以实现代码了"
  ↓
Code: "开始编写代码..."
```


---

## 常见问题

### Q1: Hook 显示 ready 但没有生效？

**检查步骤：**


1. 确认 `HOOK.md` 使用了正确的 YAML frontmatter 格式
2. 确认 `events: ` 字段存在
3. 重启 Gateway 后检查日志：
```Bash
tail -n 100 ~/.openclaw/logs/gateway.log | grep "Registered hook: discord-relay"
```



### Q2: Reviewer 反馈后 Architect 没有响应？

**可能原因：**


- Reviewer 的消息中没有包含修改关键词（需要修改/建议调整/有问题等）
- 检查日志确认关键词匹配：
```Bash
openclaw logs --follow | grep discord-relay
```



**解决方案：**


- 确保 Reviewer 的反馈包含明确的修改关键词
- 或者调整 `handler.ts` 中的 `needsRevision` 正则表达式

### Q3: Reviewer 说"通过"后没有 Bot 响应？

**可能原因：**


- Reviewer 的消息中既包含"通过"又包含"需要修改"，导致逻辑冲突
- 消息中没有明确的代码关键词，导致无法判断是 Code 还是 Scribe

**解决方案：**


- 确保 Reviewer 的通过消息清晰明确
- 建议格式："通过，可以开始实现代码了" 或 "通过，可以写文档了"

### Q4: 如何调试 Hook？

查看实时日志：

```Bash
openclaw logs --follow | grep discord-relay
```

日志会显示每条消息的路由决策：

```Text
 Processing message from Reviewer for agent architect
 Reviewer feedback → Architect revises
```

### Q5: 如何禁用 Hook？

```Bash
openclaw hooks disable discord-relay
```

或在配置文件中：

```Json
{
  "hooks": {
    "internal": {
      "entries": {
        "discord-relay": { "enabled": false }
      }
    }
  }
}
```

### Q6: 如何自定义关键词？

编辑 `~/.openclaw/hooks/discord-relay/handler.ts`，修改正则表达式：

```Typescript
// 修改反馈关键词
const needsRevision = /需要修改|建议调整|你的自定义关键词/i.test(messageText);

// 修改通过关键词
const approved = /通过|没问题|你的自定义关键词/i.test(messageText);

// 修改代码关键词
const needsCode = /代码|实现|你的自定义关键词/i.test(messageText);
```

修改后重启 Gateway：

```Bash
openclaw gateway restart
```


---

## 注意事项


1. **Bot 名称匹配：** Handler 通过 `senderName.includes('Architect')` 等方式识别 Bot，确保 Discord Bot 的显示名称包含对应关键词
2. **Agent ID 匹配：** 配置文件中的 agent ID 必须与 Handler 中的字符串完全一致
3. **Hook 优先级：** `message:inbound` Hook 在消息到达 Agent 之前执行，返回 `{ allow: false }` 会阻止消息传递
4. **性能影响：** Hook 在每条消息上执行，保持 Handler 逻辑简单高效
5. **关键词冲突：** 避免在同一条消息中同时使用"需要修改"和"通过"等冲突关键词
6. **迭代次数：** 理论上 Architect ↔ Reviewer 可以无限循环，建议在 Reviewer 的 Prompt 中限制迭代次数


---

## 扩展思路

### 添加迭代次数限制

可以在 Handler 中添加状态跟踪，限制 Architect ↔ Reviewer 的循环次数：

```Typescript
// 需要外部状态存储（如 Redis 或文件）
const iterationCount = getIterationCount(conversationId);
if (iterationCount > 3) {
  logger.warn(` Max iterations reached, forcing approval`);
  // 强制进入 Code/Scribe 阶段
}
```

### 添加更多 Bot 角色

修改 `handler.ts`，增加新的路由规则：

```Typescript
// 规则 4: Code Bot 消息 → Tester Bot 响应
if (senderName.includes('Code')) {
  if (agentId === 'tester') {
    logger.info(` Code message → Tester responds`);
    return { allow: true };
  }
}

// 规则 5: Tester Bot 消息 → 判断测试结果
if (senderName.includes('Tester')) {
  const testPassed = /测试通过|all tests passed|no bugs/i.test(messageText);
  const testFailed = /测试失败|bugs found|需要修复/i.test(messageText);
  
  if (testFailed && agentId === 'code') {
    logger.info(` Test failed → Code fixes`);
    return { allow: true };
  } else if (testPassed && agentId === 'scribe') {
    logger.info(` Test passed → Scribe documents`);
    return { allow: true };
  }
}
```

### 基于频道的路由

```Typescript
const channelId = context.channelId;
if (channelId === '1234567890') {
  // 特定频道的路由规则（如开发频道使用严格评审）
  const needsRevision = /需要修改|建议调整|有任何问题/i.test(messageText);
} else {
  // 其他频道使用宽松评审
  const needsRevision = /严重问题|必须修改/i.test(messageText);
}
```

### 时间窗口控制

```Typescript
const hour = new Date().getHours();
if (hour >= 22 || hour < 8) {
  // 夜间跳过 Reviewer，直接进入实现阶段
  if (senderName.includes('Architect') && agentId === 'code') {
    logger.info(` Night mode: skip review`);
    return { allow: true };
  }
}
```

### 添加通知机制

```Typescript
// 当迭代次数过多时，通知管理员
if (iterationCount > 5) {
  await context.notify({
    channel: 'admin-alerts',
    message: `⚠️ Architect-Reviewer 循环超过 5 次，可能需要人工介入`
  });
}
```


---

## 参考资料


- (https://docs.openclaw.ai/automation/hooks "OpenClaw Hooks 文档")
- (https://docs.openclaw.ai/tools/plugin "OpenClaw Plugin 文档")
- (https://docs.openclaw.ai/channels/discord "Discord Bot 配置")


---

## 更新日志

### v2.0 (2026-03-05)


- ✅ 新增 Architect ↔ Reviewer 迭代反馈循环
- ✅ 支持 Reviewer 反馈修改后 Architect 重新设计
- ✅ 新增关键词检测：修改反馈关键词 + 通过关键词
- ✅ 更新测试场景，增加多轮迭代示例
- ✅ 新增常见问题 Q2-Q6

### v1.0 (2026-03-05)


- ✅ 初始版本：单向接力模式
- ✅ 用户 → Architect → Reviewer → Code/Scribe


