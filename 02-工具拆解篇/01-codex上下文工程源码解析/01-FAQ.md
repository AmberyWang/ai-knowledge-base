#  FAQ


# Codex CLI 常见问题 (FAQ)

本文档回答 Codex CLI (codex-rs) 使用中的常见问题，帮助你理解配置加载、上下文管理和压缩机制。

---

## 目录

1. (#1-skills-加载机制)
2. (#2-mcp-服务器配置)
3. (#3-agentsmd-处理机制)
4. (#4-上下文添加机制)
5. (#5-上下文压缩机制)

---

## 1. Skills 加载机制

### 1.1 Skills 加载哪些路径？

按优先级从高到低：

| 优先级 | 路径                          | 说明            |
| ------ | ----------------------------- | --------------- |
| 1      | `<repo>/.codex/skills/`       | 仓库级 Skills   |
| 2      | `$CODEX_HOME/skills/`         | 用户级 Skills   |
| 3      | `$CODEX_HOME/skills/.system/` | 系统内置 Skills |
| 4      | `/etc/codex/skills/`          | 管理员级 Skills |

> `$CODEX_HOME` 默认为 `~/.codex`

### 1.2 Skills 发现机制是什么？

**扫描方式**：BFS（广度优先搜索）遍历目录

**扫描限制**：

- 最大深度：6 层
- 最大目录数：2000 个
- 跳过：`.git`、`node_modules`、隐藏目录（除 `.codex`）

**识别标志**：目录中包含 `SKILL.md` 文件

### 1.3 Skill 文件结构

```
my-skill/
├── SKILL.md      # 必需：技能定义（YAML frontmatter + Markdown 正文）
└── SKILL.toml    # 可选：UI 元数据（图标、描述等）
```

**SKILL.md 格式**：

```markdown
---
name: my-skill
description: 技能描述
---

# 技能内容

这里是技能的详细说明和指令...
```

### 1.4 Skills 何时注入到对话？

**触发条件**：用户消息中明确提到技能名称

**注入格式**：作为 `<skill>` XML 标签注入到上下文

```xml
<skill name="my-skill">
技能内容...
</skill>
```

**重要**：Skills 不会自动注入，只有用户主动引用时才会加载。

---

## 2. MCP 服务器配置

### 2.1 MCP 配置加载路径

按优先级从高到低：

| 优先级 | 来源         | 说明                                 |
| ------ | ------------ | ------------------------------------ |
| 1      | 运行时参数   | CLI 参数或 API 传入                  |
| 2      | 仓库级配置   | `<repo>/.codex/config.toml`          |
| 3      | 树遍历配置   | 从 CWD 向上查找 `.codex/config.toml` |
| 4      | 当前目录配置 | `./codex.toml`                       |
| 5      | 用户级配置   | `~/.codex/config.toml`               |
| 6      | 系统级配置   | `/etc/codex/config.toml`             |

### 2.2 MCP 配置格式

在 `config.toml` 中使用 `` 表配置：

**stdio 传输（本地进程）**：

```toml

command = "node"
args = 
env = { API_KEY = "xxx" }
```

**streamable_http 传输（远程服务）**：

```toml

url = "https://example.com/mcp"
```

### 2.3 MCP 工具命名规则

MCP 服务器提供的工具按以下格式命名：

```
mcp__<服务器名>__<工具名>
```

例如：`mcp__my-server__search`

### 2.4 MCP 生命周期管理

- **连接管理**：由 `McpConnectionManager` 统一管理
- **延迟连接**：首次使用时才建立连接
- **运行时刷新**：支持动态重新加载配置

---

## 3. AGENTS.md 处理机制

### 3.1 AGENTS.md 搜索路径

**搜索策略**：从 Git 仓库根目录到当前工作目录（CWD）逐层查找

**搜索边界**：不会跨越 Git 仓库边界

**示例**：

```
/project/          <- Git 根目录，搜索 AGENTS.md
/project/src/      <- 继续搜索
/project/src/lib/  <- CWD，最后搜索
```

### 3.2 文件名优先级

同一目录下，按以下优先级选择：

1. `AGENTS.override.md` - 最高优先级（用于本地覆盖）
2. `AGENTS.md` - 标准文件名
3. `project_doc_fallback_filenames` 配置的其他文件名

### 3.3 能否在 AGENTS.md 中引用其他文件？

**不支持**。

Codex CLI 直接读取 AGENTS.md 的原始内容，没有实现 `@include`、`#import` 或路径引用语法。

**替代方案**：

- 将所有内容直接写在 AGENTS.md 中
- 使用 Skills 机制拆分可复用的指令
- 使用 MCP 服务器提供动态上下文

### 3.4 AGENTS.md 大小限制

- **默认限制**：`project_doc_max_bytes = 4096` 字节
- **可配置**：在 `config.toml` 中修改 `project_doc_max_bytes`

超过限制的内容会被截断。

### 3.5 多个 AGENTS.md 如何合并？

合并顺序（从先到后）：

1. `config.user_instructions` - 配置文件中的用户指令
2. 项目文档（AGENTS.md）- 按目录层级合并
3. Skills 章节 - 用户引用的技能
4. `ChildAgentsMd` 消息 - 子代理的额外指令

---

## 4. 上下文添加机制

### 4.1 初始上下文构建顺序

会话开始时，上下文按以下顺序构建：

```
1. DeveloperInstructions (权限说明)
   ↓
2. DeveloperInstructions (开发者指令)
   ↓
3. 协作模式说明
   ↓
4. UserInstructions (AGENTS.md 内容)
   ↓
5. EnvironmentContext (环境信息，XML 格式)
```

### 4.2 各类上下文的处理方式

| 上下文类型            | 传递方式                | 说明                 |
| --------------------- | ----------------------- | -------------------- |
| BaseInstructions      | API `instructions` 字段 | 不在消息历史中       |
| DeveloperInstructions | 消息历史                | 权限、开发者指令     |
| UserInstructions      | 消息历史                | AGENTS.md 内容       |
| EnvironmentContext    | 消息历史                | 环境变量、工作目录等 |
| Skills                | 消息历史                | 用户引用时注入       |
| 工具定义              | API `tools` 字段        | 不在消息历史中       |

### 4.3 如何添加自定义上下文？

**方式 1：AGENTS.md**

- 在项目根目录创建 `AGENTS.md`
- 写入项目特定的指令和上下文

**方式 2：Skills**

- 在 `~/.codex/skills/` 创建技能目录
- 添加 `SKILL.md` 文件
- 对话中提及技能名触发注入

**方式 3：MCP 服务器**

- 配置 MCP 服务器提供动态工具
- 工具可以返回额外上下文信息

**方式 4：配置文件**

- 在 `config.toml` 中设置 `user_instructions`

---

## 5. 上下文压缩机制

### 5.1 何时触发压缩？

**触发条件**：对话 Token 数超过 `model_auto_compact_token_limit` 配置值

**Token 估算**：字节数 ÷ 4

### 5.2 哪些内容始终保留？

以下内容**永不压缩**：

| 内容类型       | 说明                                    |
| -------------- | --------------------------------------- |
| 系统指令       | BaseInstructions、DeveloperInstructions |
| AGENTS.md      | UserInstructions                        |
| Skills 列表    | 可用技能清单                            |
| MCP 配置       | 服务器和工具定义                        |
| GhostSnapshots | 文件快照引用                            |

### 5.3 哪些内容会被压缩？

| 内容类型       | 压缩策略                              |
| -------------- | ------------------------------------- |
| 用户消息       | 保留最近的，最多 20k tokens，倒序选择 |
| Assistant 回复 | 仅保留最后一条                        |
| 工具调用       | **全部丢弃**                          |
| 工具输出       | **全部丢弃**                          |
| Shell 命令     | **全部丢弃**                          |

### 5.4 压缩对 Skills 和 MCP 的影响

**Skills**：

- Skills 列表（可用技能清单）始终保留
- 已注入的 Skill 内容作为消息历史的一部分，可能被压缩

**MCP**：

- MCP 服务器配置始终保留
- MCP 工具定义始终保留
- MCP 工具调用的输出会被丢弃

### 5.5 如何调整压缩行为？

在 `config.toml` 中配置：

```toml
# 触发压缩的 Token 阈值
model_auto_compact_token_limit = 100000

# 压缩时保留的最大用户消息 Token 数
# (代码中硬编码为 20000，暂不可配置)
```

---

## 附录：配置文件示例

```toml
# ~/.codex/config.toml

# 用户指令（会添加到上下文）
user_instructions = "始终使用中文回复"

# AGENTS.md 最大字节数
project_doc_max_bytes = 8192

# 压缩触发阈值
model_auto_compact_token_limit = 100000

# MCP 服务器配置

command = "node"
args = 
```

---

## 相关文档

- (./core_process.md) - Codex CLI 核心流程详解
- (./docs/config.md) - 配置文件完整参考
- (./docs/skills.md) - Skills 开发指南




