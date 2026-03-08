# Moltbot 架构分析文档 2026.01.26


> 本文档面向开发者，深入解析 Moltbot 的核心功能、架构原理、消息处理流程和关键代码位置。
>
>

## 1. 项目概述

**Moltbot** 是一个**个人 AI 助手**，运行在本地设备上，通过多种聊天渠道（WhatsApp、Telegram、Slack、Discord、Signal、iMessage 等）与用户交互。

### 核心特性


| 特性 | 说明 |
| --- | --- |
| **多渠道收件箱** | WhatsApp、Telegram、Slack、Discord、Google Chat、Signal、iMessage、Microsoft Teams、WebChat 等 |
| **本地优先 Gateway** | 单一控制平面，管理会话、渠道、工具和事件 |
| **多 Agent 路由** | 将入站消息路由到隔离的 Agent（workspace + 独立会话） |
| **语音交互** | Voice Wake + Talk Mode（macOS/iOS/Android） |
| **可视化画布** | Agent 驱动的 Canvas 工作区 |
| **一等公民工具** | 浏览器控制、Canvas、Nodes、Cron、Sessions 等 |

### 技术栈


- **运行时**: Node.js >= 22
- **语言**: TypeScript (ESM)
- **核心依赖**:

  - `@mariozechner/pi-agent-core` - AI Agent 运行时
  - `@whiskeysockets/baileys` - WhatsApp 连接
  - `grammy` - Telegram Bot
  - `discord.js` - Discord Bot
  - `@slack/bolt` - Slack Bot



---

## 2. 整体架构

```Markdown
// 代码块
┌─────────────────────────────────────────────────────────────────┐
│                        用户消息入口                              │
│  WhatsApp / Telegram / Slack / Discord / Signal / iMessage / Web │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                        Gateway 服务器                            │
│                   ws://127.0.0.1:18789                           │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐    │
│  │  Channel   │ │   Router   │ │  Session   │ │   Tools    │    │
│  │  Registry  │ │            │ │  Manager   │ │  Registry  │    │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘    │
└──────────────────────────────┬──────────────────────────────────┘
                               │
           ┌───────────────────┼───────────────────┐
           ▼                   ▼                   ▼
    ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
    │  Pi Agent   │     │    CLI      │     │  macOS/iOS  │
    │   (RPC)     │     │  Commands   │     │    Nodes    │
    └─────────────┘     └─────────────┘     └─────────────┘

```


---

## 3. 核心模块详解

### 3.1 入口点 - CLI 构建

**文件**: `src/index.ts`

```TypeScript
// 代码块
// 核心导出
export { buildProgram } from "./cli/build-program.js";
export * from "./config/config.js";
export * from "./gateway/protocol/index.js";

```

CLI 通过 `buildProgram()` 构建，提供以下核心命令：


- `moltbot gateway run` - 启动 Gateway 服务器
- `moltbot agent` - 与 Agent 对话
- `moltbot message send` - 发送消息
- `moltbot channels login/status` - 渠道管理
- `moltbot onboard` - 引导式安装

### 3.2 Gateway 服务器

**文件**: `src/gateway/server.impl.ts`

Gateway 是整个系统的**控制平面**，基于 WebSocket 协议，处理所有客户端连接、会话管理和事件分发。

```TypeScript
// 代码块
// 核心类
export class GatewayServerImpl implements GatewayServer {
  // WebSocket 服务器
  private wss: WebSocketServer | null = null;
  
  // 客户端连接管理
  private connections = new Map<string, ClientConnection>();
  
  // 会话管理
  private sessions = new Map<string, SessionState>();
  
  // 工具注册
  private tools = new Map<string, ToolDefinition>();
}

```

**核心功能**:


1. **WebSocket 控制平面** - 管理客户端连接和消息分发
2. **会话管理** - 创建、恢复、紧凑化会话
3. **工具注册** - 动态注册和调用工具
4. **事件系统** - 发布/订阅模式的事件分发
5. **认证鉴权** - Token/密码认证，Tailscale 集成

**关键方法处理器目录**: `src/gateway/methods/`

### 3.3 渠道注册表

**文件**: `src/channels/registry.ts`

管理所有聊天渠道的注册、启动和消息处理。

```TypeScript
// 代码块
// 渠道注册表结构
export interface ChannelRegistry {
  // 注册渠道
  register(channel: Channel): void;
  
  // 启动所有渠道
  startAll(): Promise<void>;
  
  // 获取渠道状态
  getStatus(channelId: string): ChannelStatus;
}

```

**支持的渠道实现**:


| 渠道 | 目录 | 核心库 |
| --- | --- | --- |
| WhatsApp | `src/whatsapp/` | `@whiskeysockets/baileys` |
| Telegram | `src/telegram/` | `grammy` |
| Discord | `src/discord/` | `discord.js` |
| Slack | `src/slack/` | `@slack/bolt` |
| Signal | `src/signal/` | `signal-cli` |
| iMessage | `src/imessage/` | `imsg` |
| Web | `src/web/` | 内置 WebSocket |

**扩展渠道**: `extensions/` 目录


- `extensions/msteams/` - Microsoft Teams
- `extensions/matrix/` - Matrix
- `extensions/bluebubbles/` - BlueBubbles
- `extensions/zalo/` - Zalo

### 3.4 消息路由系统

**文件**: `src/routing/resolve-route.ts`

将入站消息路由到正确的 Agent 和 Session。

```TypeScript
// 代码块
// 路由解析
export function resolveRoute(params: {
  channel: string;
  chatId: string;
  userId: string;
  isGroup: boolean;
  config: MoltbotConfig;
}): RouteResult {
  // 1. 解析 session key
  // 2. 确定目标 agent
  // 3. 检查权限 (allowlist/pairing)
  // 4. 返回路由结果
}

```

**路由规则**:


- **DM 消息**: 路由到 `main` session 或根据 allowlist 处理
- **群组消息**: 根据 activation mode (mention/always) 决定是否响应
- **多 Agent**: 根据配置路由到不同的 agent workspace

### 3.5 Agent 运行时

**文件**: `src/agents/pi-embedded-runner/run.ts`

嵌入式 Pi Agent 执行器，处理 AI 对话的核心逻辑。

```TypeScript
// 代码块
export async function runEmbeddedPiAgent(
  params: RunEmbeddedPiAgentParams,
): Promise<EmbeddedPiRunResult> {
  // 1. 解析 session lane（队列隔离）
  // 2. 解析模型和认证
  // 3. 构建 payload
  // 4. 执行 attempt（含重试和 failover）
  // 5. 返回结果
}

```

**关键子模块**:


| 文件 | 功能 |
| --- | --- |
| `run/attempt.ts` | 单次执行尝试 |
| `run/payloads.ts` | 构建请求 payload |
| `compact.ts` | 会话紧凑化（上下文压缩） |
| `history.ts` | 历史消息限制 |
| `system-prompt.ts` | 系统提示词构建 |
| `tool-split.ts` | 工具分割处理 |

### 3.6 Session 管理

**文件**: `src/gateway/sessions-resolve.ts`, `src/gateway/session-utils.ts`

```TypeScript
// 代码块
// Session 解析
export function resolveSessionKeyFromResolveParams(params: {
  cfg: MoltbotConfig;
  p: SessionsResolveParams;
}): SessionsResolveResult;

// Session Key 结构
// 格式: {channel}:{chatId} 或 {agentId}:{channel}:{chatId}
// 例如: telegram:123456789, main:whatsapp:+1234567890

```

**Session 类型**:


- `main` - 主会话（1:1 DM）
- `group:{chatId}` - 群组会话
- `spawned:{parentId}:{childId}` - 派生会话

### 3.7 媒体处理

**文件**: `src/media/store.ts`

```TypeScript
// 代码块
// 媒体存储
export const MEDIA_MAX_BYTES = 5 * 1024 * 1024; // 5MB

export async function ensureMediaDir(): Promise<string>;
export async function cleanOldMedia(ttlMs?: number): Promise<void>;
export function extractOriginalFilename(filePath: string): string;

```

**媒体模块结构**:


| 文件 | 功能 |
| --- | --- |
| `store.ts` | 媒体文件存储和管理 |
| `fetch.ts` | 远程媒体下载 |
| `parse.ts` | 媒体消息解析 |
| `mime.ts` | MIME 类型检测 |
| `audio.ts` | 音频处理 |
| `image-ops.ts` | 图像操作 |

### 3.8 Plugin SDK

**文件**: `src/plugin-sdk/index.ts`

为插件开发提供的 SDK 接口。

```TypeScript
// 代码块
// 插件 SDK 导出
export * from "./plugin-context.js";
export * from "./plugin-types.js";
export * from "./channel-adapter.js";

```

**插件能力**:


- 自定义渠道适配器
- 工具注册
- 事件监听
- 配置扩展


---

## 4. 消息处理流程

```Markdown
// 代码块
用户发送消息
      │
      ▼
┌─────────────────┐
│ Channel Adapter │  ← Telegram/WhatsApp/Discord 等渠道适配器
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Message Parse  │  ← 解析消息内容、媒体附件
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Route Resolve  │  ← 解析路由：channel + chatId → session key
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Permission Check│  ← 检查 allowlist / pairing policy
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Session Resolve │  ← 获取或创建 session
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Agent Runner   │  ← 调用 Pi Agent 处理
└────────┬────────┘
         │
         ├── Tool Calls ──► 工具执行
         │
         ▼
┌─────────────────┐
│ Response Stream │  ← 流式响应
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Channel Send    │  ← 发送回复到原渠道
└─────────────────┘

```


---

## 5. 关键代码索引

### 5.1 入口和配置


| 功能 | 文件路径 |
| --- | --- |
| CLI 入口 | `src/index.ts` |
| CLI 构建 | `src/cli/build-program.ts` |
| 配置类型 | `src/config/config.ts` |
| 默认配置 | `src/agents/defaults.ts` |

### 5.2 Gateway 核心


| 功能 | 文件路径 |
| --- | --- |
| Gateway 服务器实现 | `src/gateway/server.impl.ts` |
| WebSocket 协议 | `src/gateway/protocol/index.ts` |
| 方法处理器 | `src/gateway/methods/*.ts` |
| Session 工具 | `src/gateway/session-utils.ts` |
| Session 解析 | `src/gateway/sessions-resolve.ts` |
| Session 补丁 | `src/gateway/sessions-patch.ts` |

### 5.3 渠道实现


| 渠道 | 目录 |
| --- | --- |
| 渠道注册表 | `src/channels/registry.ts` |
| WhatsApp | `src/whatsapp/` |
| Telegram | `src/telegram/` |
| Discord | `src/discord/` |
| Slack | `src/slack/` |
| Signal | `src/signal/` |
| iMessage | `src/imessage/` |
| Web | `src/web/` |

### 5.4 Agent 运行时


| 功能 | 文件路径 |
| --- | --- |
| Agent 运行入口 | `src/agents/pi-embedded-runner/run.ts` |
| 执行尝试 | `src/agents/pi-embedded-runner/run/attempt.ts` |
| Payload 构建 | `src/agents/pi-embedded-runner/run/payloads.ts` |
| 会话紧凑化 | `src/agents/pi-embedded-runner/compact.ts` |
| 历史限制 | `src/agents/pi-embedded-runner/history.ts` |
| 系统提示 | `src/agents/pi-embedded-runner/system-prompt.ts` |
| 模型认证 | `src/agents/model-auth.ts` |
| Failover 处理 | `src/agents/failover-error.ts` |

### 5.5 路由系统


| 功能 | 文件路径 |
| --- | --- |
| 路由解析 | `src/routing/resolve-route.ts` |
| Session Label | `src/sessions/session-label.ts` |

### 5.6 媒体处理


| 功能 | 文件路径 |
| --- | --- |
| 媒体存储 | `src/media/store.ts` |
| 媒体下载 | `src/media/fetch.ts` |
| 媒体解析 | `src/media/parse.ts` |
| MIME 检测 | `src/media/mime.ts` |
| 音频处理 | `src/media/audio.ts` |

### 5.7 工具系统


| 功能 | 文件路径 |
| --- | --- |
| 工具定义 | `src/agents/tools/` |
| 浏览器控制 | `src/browser/` |
| Canvas | `src/canvas-host/` |
| Cron 调度 | `src/cron/` |

### 5.8 扩展插件


| 插件 | 目录 |
| --- | --- |
| Microsoft Teams | `extensions/msteams/` |
| Matrix | `extensions/matrix/` |
| BlueBubbles | `extensions/bluebubbles/` |
| Zalo | `extensions/zalo/` |
| Voice Call | `extensions/voice-call/` |
| LINE | `extensions/line/` |


---

## 6. 配置结构

配置文件位置: `~/.clawdbot/moltbot.json`

```Markdown
// 代码块
{
  // Agent 配置
  "agent": {
    "model": "anthropic/claude-opus-4-5",
    "workspace": "~/clawd"
  },
  
  // Gateway 配置
  "gateway": {
    "port": 18789,
    "bind": "loopback",
    "auth": {
      "mode": "token"
    }
  },
  
  // 渠道配置
  "channels": {
    "telegram": {
      "botToken": "...",
      "allowFrom": 
    },
    "whatsapp": {
      "allowFrom": 
    },
    "discord": {
      "token": "..."
    }
  },
  
  // 安全配置
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "non-main"
      }
    }
  }
}

```


---

## 7. 安全模型

### DM 策略


| 策略 | 说明 |
| --- | --- |
| `pairing` | 默认。未知发送者收到配对码，需 `moltbot pairing approve` 批准 |
| `open` | 开放模式，需显式配置 `"*"` 在 allowlist |

### 沙箱模式


- **main session**: 工具在主机运行，Agent 有完整访问权限
- **non-main sessions**: 工具在 Docker 沙箱中运行


---

## 8. 开发命令

```Shell
// 代码块
# 安装依赖
pnpm install

# 开发模式运行
pnpm moltbot gateway run --verbose

# 构建
pnpm build

# 测试
pnpm test

# Lint
pnpm lint

```


---

## 9. 附录：目录结构

```Markdown
// 代码块
moltbot/
├── src/
│   ├── index.ts              # 入口点
│   ├── cli/                   # CLI 命令
│   ├── gateway/               # Gateway 服务器
│   │   ├── server.impl.ts     # 核心实现
│   │   ├── protocol/          # WebSocket 协议
│   │   ├── methods/           # 方法处理器
│   │   └── session-*.ts       # Session 管理
│   ├── channels/              # 渠道抽象
│   ├── routing/               # 消息路由
│   ├── agents/                # Agent 运行时
│   │   ├── pi-embedded-runner/# Pi Agent 执行器
│   │   ├── tools/             # 工具定义
│   │   └── model-auth.ts      # 模型认证
│   ├── media/                 # 媒体处理
│   ├── telegram/              # Telegram 渠道
│   ├── whatsapp/              # WhatsApp 渠道
│   ├── discord/               # Discord 渠道
│   ├── slack/                 # Slack 渠道
│   ├── signal/                # Signal 渠道
│   ├── imessage/              # iMessage 渠道
│   ├── web/                   # Web 渠道
│   ├── browser/               # 浏览器控制
│   ├── canvas-host/           # Canvas 工作区
│   ├── plugin-sdk/            # 插件 SDK
│   └── config/                # 配置管理
├── extensions/                # 扩展插件
│   ├── msteams/
│   ├── matrix/
│   ├── bluebubbles/
│   └── ...
├── apps/                      # 原生应用
│   ├── macos/
│   ├── ios/
│   └── android/
├── docs/                      # 文档
└── dist/                      # 构建输出

```



