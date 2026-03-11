# 2026-03-11 ACP 协议 & acpx CLI 快速理解与落地

## 0. TL;DR（5条要点摘要）
1. **ACP（Agent Client Protocol）**是一个让「编辑器/客户端」与「代码智能体」之间用**结构化协议（JSON-RPC）**通信的标准，目标类似 LSP：解耦 editor 和 agent。
2. **tmux/PTY**方案本质是“模拟人类敲键盘 + 抓屏解析字符流”，易碎且低效；**ACP**是“API/事件流”，更稳定可控。
3. **acpx** 是一个无头 CLI 客户端：用一套命令表面统一管理多个 ACP agent（gemini/claude/codex/openclaw…），支持持久会话、排队、取消等。
4. “是否需要 API Key”不取决于 acpx，而取决于具体 agent：
   - **Claude（claude-agent-acp）**通常走 Claude Agent SDK → Anthropic API，所以一般需要 `ANTHROPIC_API_KEY`；Claude Code 订阅登录态≠SDK 的 API 鉴权。
   - **Gemini CLI**可用 `GEMINI_API_KEY` 或 OAuth；ACP 模式下更推荐 API Key/非交互鉴权，但实践中可能仍存在 ACP 初始化超时等兼容问题。
5. 今日实操踩坑：Homebrew 版本滞后、Node 版本不匹配、Gemini CLI ACP 初始化超时、OpenClaw 需要启用 acpx 插件与 acp 配置；建议后续按“最小化依赖 + verbose 日志 + 逐个关闭 MCP/扩展”路线排查。

---

## 1. 背景：为什么需要 ACP？
传统在终端里跑 coding agent（Claude Code/Codex 等）时，经常需要：
- 用 PTY 交互（彩色 UI、进度条、交互确认）
- 通过 tmux/脚本自动化：`send-keys` 发送输入、`capture-pane` 抓输出

这会带来典型问题：
- **脆弱**：输出格式/进度条/ANSI 变了就难解析
- **低效**：需要轮询抓屏判断是否完成
- **信息丢失**：只能拿到渲染后的字符，拿不到结构化“计划/工具调用/补丁/状态”

ACP 的目标就是把这类交互升级为**结构化协议**，像“API 事件流”一样可靠。

---

## 2. 核心概念

### 2.1 ACP 是什么
- **ACP（Agent Client Protocol）**：标准化「Client（编辑器/IDE/CLI UI）」与「Agent（代码智能体/编程助手）」之间的通信。
- 采用 **JSON-RPC 2.0**：
  - Methods：请求-响应
  - Notifications：单向事件（进度、tool_call 状态等）

典型生命周期：
1) `initialize`（能力协商）
2) `session/new` 或 `session/load`
3) `session/prompt`（一轮对话/任务）
4) `session/update`（过程中不断推送 plan / message chunk / tool call update）
5) `session/prompt` response 返回 stopReason（end_turn/cancelled…）

### 2.2 Client / Agent / Terminal 的区别（常见误区）
- **Client**：实现 ACP 的那一端（Zed、acpx、某些 IDE/CLI），负责 UI、权限、文件/终端能力等。
- **Agent**：实现 ACP 的另一端（gemini --acp、cursor-agent acp、claude-agent-acp 适配器等）。
- **iTerm/Terminal**：只是字符显示器和输入设备，本身不理解 ACP。

### 2.3 PTY（Pseudo Terminal）是什么
- **PTY = Pseudo TTY / Pseudo Terminal（伪终端）**：让程序以为自己运行在真实终端里，从而输出交互 UI、ANSI 控制序列等。
- tmux 自动化本质就是在操作 PTY。

---

## 3. tmux/PTY vs ACP：原理对比

### 3.1 tmux/PTY（字符流方案）
- 输入：模拟键盘
- 输出：抓屏/解析 ANSI
- 状态：通过文本“猜”出来

优点：通用（任何 CLI 都能跑）
缺点：脆弱、难自动化、难稳定取消/恢复/多会话

### 3.2 ACP（结构化协议方案）
- 输入：`session/prompt`（结构化 content block）
- 输出：`session/update`（plan、tool_call、diff、message chunk…）
- 状态：协议内置 session/stopReason

优点：稳定、事件驱动、天然支持会话/取消/权限请求/结构化 diff

---

## 4. acpx 是什么

### 4.1 一句话定义
**acpx = Agent Client Protocol eXecutable**：无头 CLI 客户端，用统一命令表面驱动多个 ACP agent，避免 PTY scraping。

### 4.2 acpx 能做什么
- **持久会话**：跨多次命令调用保持上下文
- **命名会话**：并行多个工作流（`-s backend` / `-s docs`）
- **排队**：上一个 prompt 没结束也能先提交
- **取消**：协议级 cancel（而不是强 kill）
- **结构化输出**：比抓屏更可解析

### 4.3 常用命令速查
```bash
# 安装
npm install -g acpx@latest

# one-shot（不保存会话）
acpx <agent> exec "你的任务"

# 创建/确保会话
acpx <agent> sessions new
acpx <agent> sessions ensure

# 在当前目录对应会话中对话
acpx <agent> "继续做刚才的事"

# 查看会话与状态
acpx <agent> sessions list
acpx <agent> sessions history
acpx <agent> status

# 取消
acpx <agent> cancel

# verbose 调试
acpx --verbose <agent> exec "debug"
```

---

## 5. 各 agent 的 ACP 支持方式：native vs adapter

### 5.1 native（原生支持 ACP）
- **Gemini CLI**：`gemini --acp`
- **Cursor**：`cursor-agent acp`
- **Copilot CLI**：`copilot --acp --stdio`
- **Qwen Code**：`qwen --acp`

### 5.2 adapter（通过适配器桥接）
- **Claude**：`claude-agent-acp`（Zed 团队适配器）→ Claude Agent SDK → Anthropic API
- **Codex**：`codex-acp`（Zed 适配器）→ Codex CLI
- **Pi**：通过 `pi-acp` 适配器

结论：是否“能 ACP”不是看 acpx，而是看 agent 是否 native/adapter 方案成熟。

---

## 6. 鉴权/Key：为什么经常“必须要 API KEY”

### 6.1 acpx 不决定鉴权
acpx 是 ACP 客户端，**不直接调用模型 API**。它启动的 agent 进程怎么鉴权，取决于 agent。

### 6.2 Claude 的情况（acpx claude）
- 常见链路：acpx → `claude-agent-acp` → Claude Agent SDK → Anthropic API
- 因此通常需要：`ANTHROPIC_API_KEY`
- Claude Code 订阅/登录态（你直接在终端跑 `claude`）一般不能直接复用到 SDK。

### 6.3 Gemini 的情况（acpx gemini）
- Gemini CLI 可以走 `GEMINI_API_KEY` 或 OAuth。
- ACP/无交互场景下，通常更建议 API Key（避免卡在浏览器授权）。

---

## 7. 今日实操记录（踩坑与解决）

### 7.1 Homebrew 版本滞后
- brew 的 `gemini-cli` 一度停留在 **0.32.1**，不支持 `--acp`（只提供 `--experimental-acp` 或更早没有）。
- npm 上已发布 **0.33.0**，并提供正式 `--acp`。

处理：卸载 brew 版本，改用 npm 安装：
```bash
brew uninstall gemini-cli
npm install -g @google/gemini-cli@latest
```

### 7.2 Node 版本不匹配导致 gemini-cli 运行报错
- 旧 node（例如 v17.x）会在运行新 gemini-cli 时触发语法/regex 特性不支持。

处理：升级 node（brew 或其他方式），确保 `node --version` >= 18。

### 7.3 Gemini CLI 普通模式可用，但 ACP 模式 initialize 超时
- `gemini -p "..."` 能正常返回
- 但 `acpx gemini exec "..."` 会在 ACP `initialize` 阶段超时

可能原因（待进一步验证）：
- gemini --acp 启动时加载 MCP servers/扩展导致卡住
- ACP 协议版本/实现兼容问题
- 权限/认证链路仍有交互等待

建议排查路径：
1) `acpx --verbose gemini exec "..."` 看更详细日志
2) 以最小配置启动 gemini（禁用扩展/MCP，或使用干净 profile）
3) 检查 `~/.gemini/settings.json` 中 MCP server 是否异常（例如某个 server “Connection closed”）

### 7.4 OpenClaw 与 ACP/acpx 插件配置（用于 OpenClaw 内部跑 ACP）
- OpenClaw 要跑 ACP runtime（acpx backend）需要启用插件 + 配置：
  - 启用 acpx 插件
  - `acp.enabled=true`
  - `acp.backend=acpx`
  - `acp.allowedAgents` 包含 gemini/claude/codex...
  - 非交互权限：`plugins.entries.acpx.config.permissionMode=approve-all`（视需求）

注意：修改后需要重启 Gateway 才生效。

---

## 8. FAQ

### Q1：ACP 是 Zed 发起的吗？
是。Zed 官网写的“Created by Zed, grown by a community”。治理上目前 **Zed 与 JetBrains 联合治理**，并计划走向独立基金会。

### Q2：我在 iTerm 里跑 `claude` 的界面是不是 acpx？
不是。那是 Claude Code 自己的 PTY 交互界面。acpx 是另一个 CLI 客户端，需要你显式运行 `acpx ...`。

### Q3：acpx 一定要 API Key 吗？
不一定。acpx 不决定鉴权；但很多 agent 的 ACP/SDK 路径天然依赖 API 鉴权，所以“看起来像必须”。

### Q4：为什么 Gemini `--acp` 你这边还是不稳定？
Gemini CLI ACP 可能还在快速迭代；同时本地 MCP/扩展/认证链路也可能影响启动。建议走 verbose + 最小配置排查。

---

## 9. 参考链接
- ACP 官方文档：https://agentclientprotocol.com/
- Zed ACP 介绍页：https://zed.dev/acp
- acpx 仓库：https://github.com/openclaw/acpx
- Gemini CLI：https://github.com/google-gemini/gemini-cli
- claude-agent-acp（Zed 适配器）：https://github.com/zed-industries/claude-agent-acp

