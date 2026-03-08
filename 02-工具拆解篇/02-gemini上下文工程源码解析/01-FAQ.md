#  FAQ


# Gemini CLI 内部机制 FAQ

> 本文档深入解析 Gemini
> CLI 的核心加载机制，包括 Skill、MCP、GEMINI.md、上下文压缩等关键系统。

## 目录

1. (#1-skill-加载机制)
2. (#2-mcp-加载机制)
3. (#3-geminimd-处理机制)
4. (#4-上下文压缩机制)
5. (#5-其他上下文来源)
6. (#6-rulespolicies-系统)
7. (#7-压缩豁免说明)

---

## 1. Skill 加载机制

### 1.1 核心文件

| 文件                                       | 职责                         |
| ------------------------------------------ | ---------------------------- |
| `packages/core/src/skills/skillManager.ts` | Skill 生命周期管理、加载协调 |
| `packages/core/src/skills/skillLoader.ts`  | Skill 文件解析、路径扫描     |

### 1.2 加载路径（优先级从低到高）

```
1. 内置 Skills     → packages/core/src/skills/builtin/
2. 扩展 Skills     → ~/.gemini/extensions/{name}/skills/
3. 用户 Skills     → ~/.gemini/skills/
4. 项目 Skills     → {workspace}/.gemini/skills/
```

**后加载的同名 Skill 会覆盖先加载的。**

### 1.3 Skill 文件格式

文件名必须是 `SKILL.md` 或位于子目录 `*/SKILL.md`：

```markdown
---
name: my-skill
description: 这是一个示例技能
---

# 技能指令内容

这里是技能的详细说明和使用方式...
```

**YAML Frontmatter 字段**：

- `name` (必需): Skill 的唯一标识符
- `description` (必需): Skill 的描述，用于 Agent 选择

### 1.4 加载流程

```typescript
// packages/core/src/skills/skillManager.ts
async loadSkills(): Promise<void> {
  // 1. 加载内置 Skills
  await this.loadSkillsFromDirectory(this.builtinSkillsDirectory);

  // 2. 加载扩展 Skills
  for (const extension of this.extensions) {
    await this.loadSkillsFromDirectory(extension.skillsPath);
  }

  // 3. 加载用户 Skills
  await this.loadSkillsFromDirectory(this.userSkillsDirectory);

  // 4. 加载项目 Skills
  await this.loadSkillsFromDirectory(this.projectSkillsDirectory);
}
```

### 1.5 Skill 发现模式

```typescript
// packages/core/src/skills/skillLoader.ts
const SKILL_GLOBS = ;
```

---

## 2. MCP 加载机制

### 2.1 核心文件

| 文件                                            | 职责                          |
| ----------------------------------------------- | ----------------------------- |
| `packages/core/src/tools/mcp-client-manager.ts` | MCP 服务器生命周期管理        |
| `packages/core/src/tools/mcp-client.ts`         | 单个 MCP 服务器连接和工具发现 |
| `packages/cli/src/config/settings.ts`           | 设置加载和合并                |
| `packages/cli/src/config/extension-manager.ts`  | 扩展 MCP 服务器加载           |

### 2.2 配置路径（优先级从低到高）

```
1. Schema 默认值
2. 系统默认设置  → /Library/Application Support/GeminiCli/system-defaults.json (macOS)
                 → /etc/gemini-cli/settings.json (Linux)
                 → C:\ProgramData\gemini-cli\settings.json (Windows)
3. 系统设置     → /Library/Application Support/GeminiCli/settings.json
4. 用户设置     → ~/.gemini/settings.json
5. 工作区设置   → {workspace}/.gemini/settings.json
6. 扩展配置     → ~/.gemini/extensions/{name}/gemini-extension.json
```

### 2.3 配置格式

```json
{
  "mcpServers": {
    "server-name": {
      "command": "/path/to/server",
      "args": ,
      "env": { "API_KEY": "xxx" },
      "cwd": "/working/directory",
      "timeout": 600000,
      "trust": true,
      "includeTools": ,
      "excludeTools": 
    },
    "remote-server": {
      "url": "https://server.com/sse",
      "type": "sse",
      "headers": { "Authorization": "Bearer xxx" }
    }
  }
}
```

### 2.4 传输类型

| 类型      | 配置字段                        | 说明                           |
| --------- | ------------------------------- | ------------------------------ |
| stdio     | `command`, `args`, `env`, `cwd` | 本地进程，通过标准输入输出通信 |
| SSE       | `url`, `type: "sse"`            | Server-Sent Events 远程服务器  |
| HTTP      | `url`, `type: "http"`           | 可流式 HTTP 远程服务器         |
| WebSocket | `tcp`                           | WebSocket 连接                 |

### 2.5 加载流程

```typescript
// packages/core/src/config/config.ts:814
async initialize() {
  this.mcpClientManager = new McpClientManager(
    this.toolRegistry,
    this,
    this.eventEmitter,
  );

  // 并行启动配置的 MCP 服务器和扩展
  await Promise.all();
}

// packages/core/src/tools/mcp-client-manager.ts:269
async startConfiguredMcpServers(): Promise<void> {
  // 安全检查：非信任文件夹不加载 MCP
  if (!this.cliConfig.isTrustedFolder()) {
    return;
  }

  const servers = this.cliConfig.getMcpServers() || {};
  await Promise.all(
    Object.entries(servers).map(() =>
      this.maybeDiscoverMcpServer(name, config),
    ),
  );
}
```

### 2.6 工具发现

```typescript
// packages/core/src/tools/mcp-client.ts:72
async discover(cliConfig: Config): Promise<void> {
  // 发现 MCP 服务器提供的能力
  const prompts = await this.discoverPrompts();
  const tools = await this.discoverTools(cliConfig);
  const resources = await this.discoverResources();

  // 注册工具到全局 ToolRegistry
  for (const tool of tools) {
    this.toolRegistry.registerTool(tool);
  }
}
```

### 2.7 安全控制

```json
{
  "admin": {
    "mcp": {
      "enabled": true,
      "allowed": ,
      "excluded": 
    }
  }
}
```

---

## 3. GEMINI.md 处理机制

### 3.1 核心文件

| 文件                                           | 职责                     |
| ---------------------------------------------- | ------------------------ |
| `packages/core/src/utils/memoryDiscovery.ts`   | GEMINI.md 文件发现和加载 |
| `packages/core/src/services/contextManager.ts` | 上下文管理和整合         |
| `packages/core/src/core/prompts.ts`            | 系统提示构建             |

### 3.2 默认文件名

默认查找 `GEMINI.md`，可通过配置自定义：

```json
{
  "context": {
    "fileName": 
  }
}
```

### 3.3 加载路径

```
1. 全局文件          → ~/.gemini/GEMINI.md
2. 项目根向上遍历     → 从当前目录向上查找直到项目根
3. 子目录向下扫描     → 扫描当前目录下的子目录（有深度限制）
4. 扩展上下文文件     → ~/.gemini/extensions/{name}/context/
```

### 3.4 发现逻辑

```typescript
// packages/core/src/utils/memoryDiscovery.ts
export async function discoverMemories(
  workspaceRoot: string,
  currentDir: string,
  contextFileNames: string[],
): Promise<MemoryFile[]> {
  const memories: MemoryFile[] = [];

  // 1. 加载全局 GEMINI.md
  const globalPath = path.join(os.homedir(), '.gemini', contextFileNames);
  if (await fileExists(globalPath)) {
    memories.push({ path: globalPath, content: await readFile(globalPath) });
  }

  // 2. 从当前目录向上遍历到工作区根
  let dir = currentDir;
  while (dir !== workspaceRoot && dir !== path.dirname(dir)) {
    for (const fileName of contextFileNames) {
      const filePath = path.join(dir, fileName);
      if (await fileExists(filePath)) {
        memories.push({ path: filePath, content: await readFile(filePath) });
      }
    }
    dir = path.dirname(dir);
  }

  // 3. 扫描子目录
  await scanSubdirectories(currentDir, contextFileNames, memories);

  return memories;
}
```

### 3.5 @import 语法

GEMINI.md 支持导入其他文件：

```markdown
# 我的项目说明

@import ./docs/architecture.md @import ./CONVENTIONS.md

## 其他内容...
```

### 3.6 注入位置

GEMINI.md 内容作为系统提示的后缀注入：

```typescript
// packages/core/src/core/prompts.ts
export function getCoreSystemPrompt(config: Config): string {
  const basePrompt = getBaseSystemPrompt();
  const toolPrompts = getToolPrompts(config);
  const memorySuffix = config.getMemorySuffix(); // GEMINI.md 内容

  return `${basePrompt}\n\n${toolPrompts}\n\n${memorySuffix}`;
}
```

### 3.7 内容格式化

```typescript
// 最终注入格式
Instructions from: /path/to/GEMINI.md
<gemini_md_content>
...文件内容...
</gemini_md_content>
```

---

## 4. 上下文压缩机制

### 4.1 核心文件

| 文件                                                   | 职责           |
| ------------------------------------------------------ | -------------- |
| `packages/core/src/services/chatCompressionService.ts` | 压缩策略和执行 |
| `packages/core/src/services/contextManager.ts`         | 压缩触发判断   |

### 4.2 触发条件

```typescript
// packages/core/src/services/contextManager.ts
shouldCompress(): boolean {
  const currentTokens = this.estimateTokenCount();
  const threshold = this.config.getModelTokenLimit() * 0.5; // 50% 阈值
  return currentTokens > threshold;
}
```

**默认阈值**: 当对话 token 数超过模型限制的 50% 时触发压缩。

### 4.3 压缩策略

```typescript
// packages/core/src/services/chatCompressionService.ts
async compressHistory(history: ChatMessage[]): Promise<ChatMessage[]> {
  // 1. 保留最近 30% 的对话历史
  const recentCount = Math.ceil(history.length * 0.3);
  const recentHistory = history.slice(-recentCount);

  // 2. 对旧历史进行摘要
  const oldHistory = history.slice(0, -recentCount);
  const summary = await this.summarizeHistory(oldHistory);

  // 3. 构建压缩后的历史
  return ;
}
```

### 4.4 函数响应截断

```typescript
// 函数响应有 50K token 预算
const FUNCTION_RESPONSE_BUDGET = 50000;

if (responseTokens > FUNCTION_RESPONSE_BUDGET) {
  // 截断但保留最后 30 行
  const lines = response.split('\n');
  const lastLines = lines.slice(-30);
  return `\n${lastLines.join('\n')}`;
}
```

### 4.5 摘要格式

```xml
<state_snapshot>
## 对话摘要

### 用户目标
- 用户想要实现 XXX 功能

### 已完成的工作
1. 分析了 YYY 文件
2. 修改了 ZZZ 配置

### 当前状态
- 正在处理 AAA 问题

### 关键发现
- BBB 模块使用了 CCC 模式
</state_snapshot>
```

---

## 5. 其他上下文来源

### 5.1 环境上下文

```typescript
// packages/core/src/utils/environmentContext.ts
export function getEnvironmentContext(): string {
  return `
Current date: ${new Date().toLocaleDateString()}
Platform: ${process.platform}
Working directory: ${process.cwd()}
Directory structure:
${getDirectoryTree()}
  `;
}
```

### 5.2 MCP 服务器指令

MCP 服务器可以提供 `instructions` 字段，会被添加到上下文：

```typescript
// packages/core/src/tools/mcp-client.ts
async getInstructions(): Promise<string | undefined> {
  const serverInfo = await this.client.getServerInfo();
  return serverInfo.instructions;
}
```

### 5.3 JIT (Just-In-Time) 上下文

当工具访问特定路径时，会动态加载相关的 GEMINI.md：

```typescript
// 当读取 /path/to/file.ts 时
// 自动检查 /path/to/GEMINI.md 和 /path/GEMINI.md
```

### 5.4 上下文优先级

```
1. 系统提示 (Core System Prompt)
2. 环境上下文 (Environment Context)
3. 全局 GEMINI.md
4. 项目 GEMINI.md
5. MCP 服务器指令
6. JIT 上下文
7. 对话历史
```

---

## 6. Rules/Policies 系统

### 6.1 核心文件

| 文件                                      | 职责             |
| ----------------------------------------- | ---------------- |
| `packages/core/src/policy/config.ts`      | Policy 配置管理  |
| `packages/core/src/policy/toml-loader.ts` | TOML 格式解析    |
| `packages/core/src/policy/policies/`      | 内置 Policy 定义 |

### 6.2 加载路径（优先级从低到高）

```
1. 默认 Policies    → packages/core/src/policy/policies/
2. 用户 Policies    → ~/.gemini/policies/
3. 管理员 Policies  → 平台特定系统路径
```

### 6.3 Policy 格式 (TOML)

```toml
# ~/.gemini/policies/my-rules.toml

]
name = "禁止删除重要文件"
priority = 100
condition = "tool == 'delete_file' && path.startsWith('/important/')"
action = "deny"
message = "不允许删除 /important/ 目录下的文件"

]
name = "要求确认危险操作"
priority = 50
condition = "tool == 'run_shell_command' && command.includes('rm -rf')"
action = "confirm"
message = "这是一个危险的删除操作，确定要执行吗？"
```

### 6.4 Policy 操作类型

| Action    | 说明           |
| --------- | -------------- |
| `allow`   | 允许操作       |
| `deny`    | 拒绝操作       |
| `confirm` | 需要用户确认   |
| `warn`    | 显示警告但继续 |

---

## 7. 压缩豁免说明

### 7.1 不被压缩的内容

| 内容               | 原因                               |
| ------------------ | ---------------------------------- |
| **系统提示**       | 每次调用时重新构建，包含 GEMINI.md |
| **GEMINI.md**      | 作为系统提示的一部分，不参与压缩   |
| **Rules/Policies** | 在请求处理层面生效，不在上下文中   |
| **MCP 服务器配置** | 配置级别，不在对话历史中           |
| **Skill 定义**     | 按需加载，不持久化在对话中         |

### 7.2 会被压缩的内容

| 内容             | 压缩方式           |
| ---------------- | ------------------ |
| **用户消息**     | 旧消息被摘要       |
| **助手响应**     | 旧响应被摘要       |
| **工具调用结果** | 超长结果被截断     |
| **JIT 上下文**   | 随对话历史一起压缩 |

### 7.3 压缩流程图

```
用户消息 → Token 计数检查
              ↓
         超过 50% 阈值?
              ↓
        是 → 触发压缩
              ↓
    保留最近 30% 历史 + 摘要旧历史
              ↓
    重建: 系统提示(含GEMINI.md) + 压缩历史 + 新消息
```

---

## 附录：配置示例

### 完整的 settings.json 示例

```json
{
  "context": {
    "fileName": 
  },
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": 
    },
    "github": {
      "command": "npx",
      "args": ,
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  },
  "admin": {
    "mcp": {
      "enabled": true,
      "excluded": 
    }
  }
}
```

### 扩展配置示例

```json
// ~/.gemini/extensions/my-extension/gemini-extension.json
{
  "name": "my-extension",
  "version": "1.0.0",
  "description": "我的自定义扩展",
  "mcpServers": {
    "custom-server": {
      "command": "node",
      "args": 
    }
  },
  "contextFiles": ,
  "skills": "./skills"
}
```

---

## 参考文档

- (./docs/tools/mcp-server.md)
- (./docs/cli/gemini-md.md)
- (./docs/extensions/)

---

_文档生成时间: 2026-01-20_ _基于 gemini-cli 源码分析_




