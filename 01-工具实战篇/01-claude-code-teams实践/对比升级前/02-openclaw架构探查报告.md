# OpenClaw 架构探查报告


<!-- TOC -->

# OpenClaw 架构探查报告

## 一、项目概览

### 1.1 项目定位

**OpenClaw** 是一个开源的个人 AI 助手框架,核心特点:


- **多渠道集成**: 支持 WhatsApp/Telegram/Slack/Discord/Signal/iMessage 等 15+ 渠道
- **本地运行**: 数据隐私优先,无需依赖云服务
- **可扩展**: 插件化 Skills 系统,支持自定义能力扩展
- **跨平台**: macOS/iOS/Android/Web 全平台支持

### 1.2 核心价值


| 维度 | 特性 | 竞品对比 |
| --- | --- | --- |
| **隐私性** | 本地运行,数据不出域 | Claude Desktop 云端依赖 |
| **渠道** | 15+ 渠道统一接入 | Cursor 仅支持 IDE |
| **扩展性** | Skills 插件系统 | ChatGPT 封闭生态 |
| **成本** | 自托管,按需付费 | 订阅制 $20+/月 |


---

## 二、项目结构分析

### 2.1 目录结构

```Text
openclaw/
├── packages/          # 核心包
│   ├── gateway/      # 网关服务 (消息路由)
│   ├── agent/        # Agent 核心逻辑
│   ├── channels/     # 渠道适配器
│   ├── skills/       # 内置 Skills
│   ├── memory/       # 上下文管理
│   └── ui/           # Web UI
├── apps/             # 客户端应用
│   ├── macos/        # macOS 原生应用
│   ├── ios/          # iOS 应用
│   └── android/      # Android 应用
├── docs/             # 文档
├── Swabble/          # 语音通话模块
└── assets/           # 静态资源
```

### 2.2 核心模块职责

#### Gateway (网关)


- **职责**: 消息路由、会话管理、认证鉴权
- **关键文件**:

  - `packages/gateway/src/server.ts` - HTTP 服务器
  - `packages/gateway/src/router.ts` - 路由分发
  - `packages/gateway/src/session.ts` - 会话管理


#### Agent (智能体)


- **职责**: 模型调用、工具执行、上下文管理
- **关键文件**:

  - `packages/agent/src/loop.ts` - Agent 执行循环
  - `packages/agent/src/tools.ts` - 工具注册与调用
  - `packages/agent/src/context.ts` - 上下文构建


#### Channels (渠道)


- **职责**: 统一消息格式、适配不同平台协议
- **支持渠道**:

  - WhatsApp (通过 Baileys)
  - Telegram (Grammy Bot)
  - Slack (Bolt SDK)
  - Discord (Discord.js)
  - Signal (signal-cli)
  - iMessage (BlueBubbles)
  - Google Chat
  - Microsoft Teams
  - Matrix
  - Zalo


#### Skills (技能)


- **职责**: 扩展 Agent 能力 (搜索/代码执行/文件操作等)
- **内置 Skills**:

  - Web 搜索 (Brave Search)
  - 代码执行 (Sandbox)
  - 文件操作 (File System)
  - 浏览器控制 (Playwright)
  - 日历管理



---

## 三、核心技术栈

### 3.1 后端技术


| 技术 | 用途 | 版本要求 |
| --- | --- | --- |
| **Node.js** | 运行时 | ≥22 |
| **TypeScript** | 开发语言 | 5.x |
| **Fastify** | HTTP 框架 | 4.x |
| **Prisma** | ORM (SQLite) | 5.x |
| **Zod** | 数据校验 | 3.x |

### 3.2 前端技术


| 技术 | 用途 |
| --- | --- |
| **React** | UI 框架 |
| **Vite** | 构建工具 |
| **Tailwind CSS** | 样式框架 |
| **shadcn/ui** | 组件库 |

### 3.3 AI 集成


| 提供商 | 支持模型 | 认证方式 |
| --- | --- | --- |
| **Anthropic** | Claude Opus/Sonnet/Haiku | OAuth + API Key |
| **OpenAI** | GPT-4/GPT-3.5 | API Key |
| **本地模型** | Ollama/LM Studio | HTTP |


---

## 四、核心机制深度解析

### 4.1 消息路由机制

```Mermaid
graph LR
    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    F --> E
    E --> G
    G --> B
    B --> H
```

**关键流程:**


1. **Channel Adapter** 接收原始消息 (WhatsApp/Telegram 等)
2. **Gateway Router** 根据 `channelId` 和 `userId` 路由
3. **Session Manager** 加载会话上下文
4. **Agent Loop** 执行推理与工具调用
5. **Response Formatter** 格式化响应 (Markdown/富文本)
6. **Channel Adapter** 发送回复

### 4.2 Agent 执行循环

```Typescript
// 简化版 Agent Loop
async function agentLoop(message: string, context: Context) {
  while (true) {
    // 1. 构建 Prompt
    const prompt = buildPrompt(message, context);
    
    // 2. 调用模型
    const response = await model.chat(prompt);
    
    // 3. 检查是否需要工具调用
    if (response.toolCalls) {
      const results = await executeTools(response.toolCalls);
      context.addToolResults(results);
      continue; // 继续循环
    }
    
    // 4. 返回最终响应
    return response.content;
  }
}
```

**关键特性:**


- **迭代式工具调用**: 支持多轮工具链
- **上下文压缩**: 超过 token 限制时自动压缩
- **错误重试**: 工具失败时自动重试

### 4.3 上下文管理策略

```Typescript
// 上下文压缩示例
async function compactContext(messages: Message[]) {
  if (tokenCount(messages) < MAX_TOKENS) {
    return messages;
  }
  
  // 策略 1: 保留最近 N 条消息
  const recent = messages.slice(-10);
  
  // 策略 2: 提取关键信息
  const summary = await model.summarize(messages);
  
  return ;
}
```

### 4.4 Skills 插件机制

**Skill 定义示例:**

```Typescript
// packages/skills/web-search/index.ts
export const webSearchSkill: Skill = {
  name: 'web_search',
  description: 'Search the web using Brave Search API',
  parameters: {
    query: { type: 'string', required: true },
    count: { type: 'number', default: 5 }
  },
  execute: async (params) => {
    const results = await braveSearch(params.query, params.count);
    return formatResults(results);
  }
};
```

**注册流程:**

```Typescript
// Agent 启动时注册 Skills
agent.registerSkill(webSearchSkill);
agent.registerSkill(codeExecutionSkill);
agent.registerSkill(fileSystemSkill);
```


---

## 五、扩展机制分析

### 5.1 自定义 Channel

**实现步骤:**


1. 创建 Channel Adapter:

```Typescript
// packages/channels/custom/index.ts
export class CustomChannel implements Channel {
  async connect() {
    // 连接到目标平台
  }
  
  async onMessage(handler: MessageHandler) {
    // 监听消息
  }
  
  async sendMessage(userId: string, content: string) {
    // 发送消息
  }
}
```


1. 注册到 Gateway:

```Typescript
gateway.registerChannel('custom', new CustomChannel());
```

### 5.2 自定义 Skill

**实现步骤:**


1. 定义 Skill:

```Typescript
export const mySkill: Skill = {
  name: 'my_custom_skill',
  description: '自定义能力描述',
  parameters: { /* 参数定义 */ },
  execute: async (params) => {
    // 实现逻辑
  }
};
```


1. 在配置文件中启用:

```Json
{
  "skills": {
    "enabled": 
  }
}
```

### 5.3 Cron 任务

**配置示例:**

```Yaml
# cron.yml
jobs:
  - name: daily_summary
    schedule: "0 9 * * *"  # 每天 9:00
    action:
      type: agent
      message: "生成昨日工作总结"
      channel: telegram
      userId: "@admin"
```


---

## 六、部署架构

### 6.1 单机部署

```Text
┌─────────────────────────────┐
│   OpenClaw Gateway (Node)   │
│   - HTTP Server (18789)     │
│   - SQLite Database         │
│   - Cron Scheduler          │
└─────────────────────────────┘
        ↓          ↑
    消息路由    响应返回
        ↓          ↑
┌─────────────────────────────┐
│  Channels (多渠道适配器)     │
│  - WhatsApp/Telegram/Slack  │
│  - Discord/Signal/iMessage  │
└─────────────────────────────┘
```

### 6.2 Docker 部署

```Bash
docker run -d \
  -p 18789:18789 \
  -v ~/.openclaw:/root/.openclaw \
  openclaw/openclaw:latest
```

### 6.3 系统服务 (Daemon)

**macOS (launchd):**

```Bash
openclaw onboard --install-daemon
```

**Linux (systemd):**

```Bash
openclaw gateway --install-service
```


---

## 七、关键配置文件

### 7.1 主配置文件

**位置:** `~/.openclaw/config.json`

```Json
{
  "gateway": {
    "port": 18789,
    "host": "0.0.0.0"
  },
  "models": {
    "default": "anthropic/claude-opus-4",
    "fallback": 
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "token": "YOUR_BOT_TOKEN"
    }
  },
  "skills": {
    "enabled": 
  }
}
```

### 7.2 环境变量

```Bash
# .env
ANTHROPIC_API_KEY=sk-ant-xxx
OPENAI_API_KEY=sk-xxx
BRAVE_API_KEY=BSAxxxxx
```


---

## 八、性能优化建议

### 8.1 上下文压缩


- 启用自动压缩: `compaction.enabled = true`
- 设置压缩阈值: `compaction.maxTokens = 8000`
- 保留关键消息: 系统消息和最近 10 条

### 8.2 模型选择


| 场景 | 推荐模型 | 原因 |
| --- | --- | --- |
| 日常对话 | Claude Sonnet | 性价比高 |
| 复杂推理 | Claude Opus | 能力最强 |
| 快速响应 | GPT-3.5 Turbo | 速度快 |
| 本地部署 | Ollama Llama3 | 无需 API |

### 8.3 缓存策略


- 启用 Prompt 缓存 (Anthropic Prompt Caching)
- 缓存系统提示词和 Skill 定义
- 减少 50% token 消耗


---

## 九、安全机制

### 9.1 沙箱隔离


- 代码执行使用 Docker 沙箱
- 文件访问限制在 workspace 目录
- 网络请求需显式授权

### 9.2 权限控制

```Json
{
  "security": {
    "allowedUsers": ,
    "restrictedSkills": ,
    "requireApproval": true
  }
}
```

### 9.3 审计日志


- 所有工具调用记录到 `~/.openclaw/logs/audit.log`
- 包含用户、时间、参数、结果


---

## 十、常见问题排查

### 10.1 Channel 连接失败

```Bash
# 检查 Gateway 状态
openclaw status

# 查看日志
openclaw logs --channel telegram

# 重启 Gateway
openclaw gateway restart
```

### 10.2 模型调用失败

```Bash
# 测试模型连接
openclaw models test

# 查看失败原因
openclaw logs --level error
```

### 10.3 上下文溢出

```Bash
# 手动压缩会话
openclaw memory compact --session <id>

# 清理历史
openclaw memory clear --before 30d
```


---

## 十一、参考资源

### 11.1 官方文档


- 架构文档: https://docs.openclaw.ai/concepts/architecture
- Agent 循环: https://docs.openclaw.ai/concepts/agent-loop
- Skills 开发: https://docs.openclaw.ai/cli/skills

### 11.2 源码关键文件


| 模块 | 关键文件 | 说明 |
| --- | --- | --- |
| Gateway | `packages/gateway/src/server.ts` | HTTP 服务器 |
| Agent | `packages/agent/src/loop.ts` | 执行循环 |
| Channels | `packages/channels/telegram/index.ts` | Telegram 适配器 |
| Skills | `packages/skills/web-search/index.ts` | Web 搜索 |

### 11.3 社区资源


- GitHub: https://github.com/openclaw/openclaw
- Discord: https://discord.gg/clawd
- DeepWiki: https://deepwiki.com/openclaw/openclaw


---

**更新时间:** 2026-02-12   **维护者:** openclaw 学习团队   **版本:** v1.0


