# OpenClaw 架构全景解析


<!-- TOC -->

# OpenClaw 架构全景解析

## 一、项目概览

### 1.1 核心定位

**OpenClaw** 是一个开源的个人 AI 助手框架,实现了以下核心能力:


- **多渠道统一接入**: 支持 15+ 通讯渠道 (WhatsApp/Telegram/Slack/Discord/Signal/iMessage 等)
- **本地优先运行**: 数据隐私保护,无需依赖云服务
- **插件化扩展**: 通过 Skills 系统灵活扩展能力
- **跨平台支持**: macOS/iOS/Android/Web 全平台覆盖

### 1.2 竞品对比


| 维度 | OpenClaw | Claude Desktop | Cursor | ChatGPT |
| --- | --- | --- | --- | --- |
| **部署方式** | 本地/自托管 | 云端 | 云端 | 云端 |
| **隐私性** | 数据不出域 | 云端处理 | 云端处理 | 云端处理 |
| **渠道支持** | 15+ 渠道 | 桌面应用 | IDE 内置 | Web/App |
| **扩展性** | Skills 插件 | 封闭 | 封闭 | 封闭生态 |
| **成本模式** | 自托管按需付费 | $20+/月订阅 | $20+/月订阅 | $20+/月订阅 |


---

## 二、整体架构设计

### 2.1 分层架构

```Text
┌─────────────────────────────────────────┐
│         应用层 (Apps)                   │
│  macOS / iOS / Android / Web UI        │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│       网关层 (Gateway)                  │
│  消息路由 / 会话管理 / 认证鉴权         │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│       智能体层 (Agent)                  │
│  模型调用 / 工具执行 / 上下文管理       │
└─────────────────────────────────────────┘
         ↙                ↘
┌──────────────────┐  ┌──────────────────┐
│ 渠道层 (Channels) │  │ 技能层 (Skills)  │
│  多渠道适配器     │  │  可扩展能力插件   │
└──────────────────┘  └──────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│       存储层 (Storage)                  │
│  SQLite / 向量数据库 / 文件系统         │
└─────────────────────────────────────────┘
```

### 2.2 核心模块职责

#### Gateway (网关模块)

**职责**:


- 消息路由与分发
- 会话状态管理
- 用户认证与鉴权
- HTTP/WebSocket 服务

**关键文件**:


- `packages/gateway/src/server.ts` - Fastify 服务器
- `packages/gateway/src/router.ts` - 路由分发逻辑
- `packages/gateway/src/session.ts` - 会话管理

#### Agent (智能体模块)

**职责**:


- LLM 模型调用
- 工具链执行
- 上下文构建与压缩
- 多轮对话管理

**关键文件**:


- `packages/agent/src/loop.ts` - Agent 执行循环
- `packages/agent/src/tools.ts` - 工具注册与调用
- `packages/agent/src/context.ts` - 上下文管理

#### Channels (渠道模块)

**职责**:


- 统一消息格式
- 适配不同平台协议
- 异步消息处理

**支持渠道**:


- WhatsApp (Baileys)
- Telegram (Grammy Bot)
- Slack (Bolt SDK)
- Discord (Discord.js)
- Signal (signal-cli)
- iMessage (BlueBubbles)
- Google Chat
- Microsoft Teams
- Matrix
- Zalo

#### Skills (技能模块)

**职责**:


- 扩展 Agent 能力
- 提供工具函数
- 支持插件化开发

**内置 Skills**:


- Web 搜索 (Brave Search)
- 代码执行 (Docker Sandbox)
- 文件操作 (File System)
- 浏览器控制 (Playwright)
- 日历管理

#### Memory (记忆模块)

**职责**:


- 上下文压缩
- 会话持久化
- 向量检索 (可选)


---

## 三、技术栈详解

### 3.1 后端技术


| 技术 | 版本 | 用途 |
| --- | --- | --- |
| **Node.js** | ≥22 | 运行时环境 |
| **TypeScript** | 5.x | 开发语言 |
| **Fastify** | 4.x | 高性能 HTTP 框架 |
| **Prisma** | 5.x | ORM (SQLite) |
| **Zod** | 3.x | 数据校验 |

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

## 四、消息路由机制

### 4.1 路由流程

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

### 4.2 关键步骤


1. **Channel Adapter** 接收原始消息
2. **Gateway Router** 根据 `channelId` 和 `userId` 路由
3. **Session Manager** 加载会话上下文
4. **Agent Loop** 执行推理与工具调用
5. **Response Formatter** 格式化响应
6. **Channel Adapter** 发送回复

### 4.3 代码示例

```Typescript
// Gateway Router 核心逻辑
async function routeMessage(message: IncomingMessage) {
  // 1. 提取路由信息
  const { channelId, userId, content } = message;
  
  // 2. 加载会话
  const session = await sessionManager.load({ channelId, userId });
  
  // 3. 调用 Agent
  const response = await agent.execute({
    message: content,
    context: session.context
  });
  
  // 4. 更新会话
  await session.update(response.context);
  
  // 5. 返回响应
  return response.content;
}
```


---

## 五、Agent 执行循环

### 5.1 执行流程

```Typescript
async function agentLoop(message: string, context: Context) {
  while (true) {
    // 1. 构建 Prompt
    const prompt = buildPrompt(message, context);
    
    // 2. 调用模型
    const response = await model.chat(prompt);
    
    // 3. 检查工具调用
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

### 5.2 关键特性


- **迭代式工具调用**: 支持多轮工具链
- **上下文压缩**: 超过 token 限制时自动压缩
- **错误重试**: 工具失败时自动重试
- **并发控制**: 限制同时执行的工具数量


---

## 六、上下文管理策略

### 6.1 压缩机制

```Typescript
async function compactContext(messages: Message[]) {
  if (tokenCount(messages) < MAX_TOKENS) {
    return messages;
  }
  
  // 策略 1: 保留最近 N 条消息
  const recent = messages.slice(-10);
  
  // 策略 2: 提取关键信息
  const summary = await model.summarize(messages.slice(0, -10));
  
  return ;
}
```

### 6.2 优化建议


- 启用自动压缩: `compaction.enabled = true`
- 设置压缩阈值: `compaction.maxTokens = 8000`
- 保留关键消息: 系统消息和最近 10 条
- 使用 Prompt Caching 减少 token 消耗


---

## 七、Skills 插件机制

### 7.1 Skill 定义

```Typescript
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

### 7.2 注册流程

```Typescript
// Agent 启动时注册 Skills
agent.registerSkill(webSearchSkill);
agent.registerSkill(codeExecutionSkill);
agent.registerSkill(fileSystemSkill);
```

### 7.3 自定义 Skill 开发

```Typescript
export const myCustomSkill: Skill = {
  name: 'my_custom_skill',
  description: '自定义能力描述',
  parameters: {
    input: { type: 'string', required: true }
  },
  execute: async (params) => {
    // 实现自定义逻辑
    return `处理结果: ${params.input}`;
  }
};
```


---

## 八、部署架构

### 8.1 单机部署

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

### 8.2 Docker 部署

```Bash
docker run -d \
  -p 18789:18789 \
  -v ~/.openclaw:/root/.openclaw \
  openclaw/openclaw:latest
```

### 8.3 系统服务

**macOS (launchd)**:

```Bash
openclaw onboard --install-daemon
```

**Linux (systemd)**:

```Bash
openclaw gateway --install-service
```


---

## 九、核心配置

### 9.1 主配置文件

位置: `~/.openclaw/config.json`

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

### 9.2 环境变量

```Bash
# .env
ANTHROPIC_API_KEY=sk-ant-xxx
OPENAI_API_KEY=sk-xxx
BRAVE_API_KEY=BSAxxxxx
```


---

## 十、性能优化

### 10.1 模型选择


| 场景 | 推荐模型 | 原因 |
| --- | --- | --- |
| 日常对话 | Claude Sonnet | 性价比高 |
| 复杂推理 | Claude Opus | 能力最强 |
| 快速响应 | GPT-3.5 Turbo | 速度快 |
| 本地部署 | Ollama Llama3 | 无需 API |

### 10.2 缓存策略


- 启用 Prompt Caching (Anthropic)
- 缓存系统提示词和 Skill 定义
- 减少 50% token 消耗


---

## 十一、安全机制

### 11.1 沙箱隔离


- 代码执行使用 Docker 沙箱
- 文件访问限制在 workspace 目录
- 网络请求需显式授权

### 11.2 权限控制

```Json
{
  "security": {
    "allowedUsers": ,
    "restrictedSkills": ,
    "requireApproval": true
  }
}
```

### 11.3 审计日志


- 所有工具调用记录到 `~/.openclaw/logs/audit.log`
- 包含用户、时间、参数、结果


---

## 十二、参考资源

### 官方资源


- 官方文档: https://docs.openclaw.ai/concepts/architecture
- GitHub: https://github.com/openclaw/openclaw
- Discord 社区: https://discord.gg/clawd

### 源码关键文件


- Gateway: `packages/gateway/src/server.ts`
- Agent: `packages/agent/src/loop.ts`
- Channels: `packages/channels/telegram/index.ts`
- Skills: `packages/skills/web-search/index.ts`


---

**文档版本**: v1.0   **更新时间**: 2026-02-12   **维护团队**: OpenClaw 研究小组


