#  OpenCode FAQ

## Architecture & Configuration

### 1. `session/prompt.ts` 中的 `resolveTools` 到底加载了哪些工具？

`resolveTools()` 本质加载两类工具：

#### A. 内置 ToolRegistry 工具

```TypeScript
// 代码块
for (const item of await ToolRegistry.tools(input.model.providerID, input.agent)) {
  tools = tool({ ... execute() { item.execute(args, ctx) } })
}

```

这部分来自 `ToolRegistry.tools(...)`，与文件路径无直接关系，是 opencode 内置/注册的工具集合（read/write/bash 等）。

#### B. MCP 工具

```TypeScript
// 代码块
for (const  of Object.entries(await MCP.tools())) {
  tools = item
}

```


- `resolveTools` **并不会"按路径加载 MCP"**
- 它只是调用 `MCP.tools()`，把已经连接上的 MCP server 暴露出来的工具注入到本轮 LLM 可调用工具里

**结论**：`resolveTools` 加载工具的来源只有两类：`ToolRegistry` + `MCP.tools()`。**Skill 不在这里加载**。


---

### 2. Skill 会加载哪些路径的文件？

Skill 的加载逻辑在：`packages/opencode/src/skill/skill.ts`

#### A. OpenCode 自己的 Skill 路径（Project/Config 目录）

```TypeScript
// 代码块
const OPENCODE_SKILL_GLOB = new Bun.Glob("{skill,skills}/**/SKILL.md")
...
for (const dir of await Config.directories()) {
  for await (const match of OPENCODE_SKILL_GLOB.scan({ cwd: dir, absolute: true, ... })) {
    await addSkill(match)
  }
}

```

会在 `Config.directories()` 返回的**每个配置目录**下扫描：


- `{dir}/skill/**/SKILL.md`
- `{dir}/skills/**/SKILL.md`

`dir` 是"配置目录"，通常对应 `.opencode` 体系。

#### B. 兼容 Claude Code 的 Skills 路径（.claude）

```TypeScript
// 代码块
const CLAUDE_SKILL_GLOB = new Bun.Glob("skills/**/SKILL.md")

// 1) 从当前工作目录向上找 .claude（直到 worktree）
Filesystem.up({ targets: , start: Instance.directory, stop: Instance.worktree })

// 2) 以及全局 ~/.claude
const globalClaude = `${Global.Path.home}/.claude`

```

只要没开禁用开关 `Flag.OPENCODE_DISABLE_CLAUDE_CODE_SKILLS`，就会扫描：


- `{某层级}/.claude/skills/**/SKILL.md`（从 cwd 往上找到的所有 `.claude`）
- `~/.claude/skills/**/SKILL.md`

**核心 Glob 模式**：


| 来源 | Glob 模式 | 路径查找方式 |
| --- | --- | --- |
| OpenCode | `{skill,skills}/**/SKILL.md` | 在 `Config.directories()` 下 |
| Claude Code | `skills/**/SKILL.md` | 在 `.claude` 目录下（project + global） |

#### C. `Config.directories()` 的来源和构成

`Config.directories()` 是 OpenCode 内部硬编码的目录扫描逻辑，**不是用户可配置的选项**。

**代码位置**：`packages/opencode/src/config/config.ts` 第 91-112 行、第 1223-1225 行

**构建逻辑**：

```TypeScript
// 代码块
const directories = ,
    start: Instance.directory, // 从当前工作目录
    stop: Instance.worktree, // 到 git worktree 根
  }),
  ...Filesystem.up({
    // 3. 用户 home 下的 .opencode
    targets: ,
    start: Global.Path.home,
    stop: Global.Path.home,
  }),
]

if (Flag.OPENCODE_CONFIG_DIR) {
  // 4. 环境变量指定的目录（可选）
  directories.push(Flag.OPENCODE_CONFIG_DIR)
}

```

**具体路径解析**：


| 来源 | 实际路径 | 说明 |
| --- | --- | --- |
| `Global.Path.config` | `~/.config/opencode` (Linux/Mac) | 使用 `xdg-basedir` 库，遵循 XDG 规范 |
| 项目级 `.opencode` | `./project/.opencode` → `./parent/.opencode` → ... | 从 cwd 向上查找到 worktree |
| Home `.opencode` | `~/.opencode` | 用户主目录下的 .opencode |
| `OPENCODE_CONFIG_DIR` | 环境变量指定的任意路径 | 可选，手动指定 |

**`Global.Path.config 的定义`**（`packages/opencode/src/global/index.ts`）：

```TypeScript
// 代码块
import { xdgConfig } from "xdg-basedir"

const config = path.join(xdgConfig!, "opencode")  // ~/.config/opencode

export namespace Global {
  export const Path = {
    config,  // ~/.config/opencode
    home: os.homedir(),  // ~
    ...
  }
}

```

**完整的目录扫描顺序示例**：

假设你在 `/home/user/projects/myapp` 目录运行 opencode：

```Markdown
// 代码块
1. ~/.config/opencode                         ← Global.Path.config (XDG 规范)
2. /home/user/projects/myapp/.opencode        ← 项目级（当前目录）
3. /home/user/projects/.opencode              ← 项目级（父目录，如存在）
4. /home/user/.opencode                       ← Home 级
5. ${OPENCODE_CONFIG_DIR}                     ← 环境变量（如设置）

```

**Skill 最终会在以下路径查找**：


- `~/.config/opencode/skill/**/SKILL.md`
- `~/.config/opencode/skills/**/SKILL.md`
- `./.opencode/skill/**/SKILL.md`
- `./.opencode/skills/**/SKILL.md`
- `~/.opencode/skill/**/SKILL.md`
- `~/.opencode/skills/**/SKILL.md`
- `${OPENCODE_CONFIG_DIR}/skill/**/SKILL.md`（如设置环境变量）
- `${OPENCODE_CONFIG_DIR}/skills/**/SKILL.md`（如设置环境变量）

**用户可以通过以下方式影响 directories**：


1. **创建目录** - 在上述路径创建 `.opencode` 文件夹
2. **环境变量** - 设置 `OPENCODE_CONFIG_DIR` 添加额外目录
3. **XDG 环境变量** - 修改 `XDG_CONFIG_HOME` 改变全局配置路径


---

### 3. MCP 会"加载哪些路径"？

**重要**：MCP 不是靠扫描目录加载的，而是来自**配置**。

#### MCP 加载机制

```TypeScript
// 代码块
const cfg = await Config.get()
const config = cfg.mcp ?? {}
Object.entries(config).map(async () => { ... create(key, mcp) ... })

```

MCP 的"加载来源" = `Config.get().mcp` 配置项，每个 entry 有 `type: "remote" | "local"` 等字段：

#### Remote MCP


- 通过 HTTP/SSE/streamable-http 连接远程服务
- 配置项包含 `mcp.url`、可选的 `mcp.headers`、OAuth 配置等

#### Local MCP


- 通过 stdio 启动本地子进程
- 配置项包含 `mcp.command`（可执行文件/命令）
- 工作目录 `cwd = Instance.directory`（当前工作目录）
- 环境变量继承 `process.env` 并叠加 `mcp.environment`

**结论**：本地 MCP 的"路径感"体现在 `mcp.command` 和 `cwd`，而非按目录扫描。


---

### 4. `AGENTS.md` 在 OpenCode 处理用户消息时如何生效？

`AGENTS.md` 的注入点在 `SystemPrompt.custom()`（`packages/opencode/src/session/system.ts`）。

每次 LLM 调用前，这些内容会被拼进系统提示词：

```TypeScript
// 代码块
system: 

```

#### SystemPrompt.custom() 的查找规则

规则文件收集来源分三类：

##### A. 本地向上查找（只取**第一种命中的文件类型**）

```TypeScript
// 代码块
const LOCAL_RULE_FILES = 
for (const localRuleFile of LOCAL_RULE_FILES) {
  const matches = await Filesystem.findUp(localRuleFile, Instance.directory, Instance.worktree)
  if (matches.length > 0) { add all matches; break }
}

```

**含义**：


- 从 `Instance.directory` 往上到 `Instance.worktree`
- 如果找到 `AGENTS.md`，就把找到的所有层级都加进去，然后 **break**（不会再去找 (http://CLAUDE.md "CLAUDE.md")）
- 只有没找到 (http://AGENTS.md "AGENTS.md")，才会尝试 (http://CLAUDE.md "CLAUDE.md")，以此类推

##### B. 全局规则文件（只取**第一个存在的**）

```TypeScript
// 代码块
const GLOBAL_RULE_FILES = 
if (!disable_claude_prompt) add ~/.claude/CLAUDE.md
if (Flag.OPENCODE_CONFIG_DIR) add ${OPENCODE_CONFIG_DIR}/AGENTS.md

for (const globalRuleFile of GLOBAL_RULE_FILES) {
  if (exists) { add; break }
}

```

##### C. `config.instructions` 显式指引

支持 glob / 绝对路径 / ~/ / URL：

```TypeScript
// 代码块
if (config.instructions) {
  for (let instruction of config.instructions) {
    // 支持 http/https URLs
    if (instruction.startsWith("https://") || instruction.startsWith("http://")) {
      urls.push(instruction)
      continue
    }
    // 支持 ~/ 路径
    if (instruction.startsWith("~/")) {
      instruction = path.join(os.homedir(), instruction.slice(2))
    }
    // 支持绝对路径 glob 或相对 glob
    // ...处理并添加到 paths 集合
  }
}

```

最终会把每个文件内容包装成：

```Markdown
// 代码块
Instructions from: <path-or-url>
<file-content>

```

并注入 system prompt。

#### 查找优先级总结


| 优先级 | 来源 | 行为 |
| --- | --- | --- |
| 1 | 本地 `AGENTS.md` | 向上查找，**第一个找到的文件类型就 break** |
| 2 | 本地 `CLAUDE.md` | 只在没找到 (http://AGENTS.md "AGENTS.md") 时查找 |
| 3 | 本地 `CONTEXT.md` | 只在没找到前两者时查找（deprecated） |
| 4 | 全局 `~/.opencode/config/AGENTS.md` | 单一全局文件 |
| 5 | 全局 `~/.claude/CLAUDE.md` | 单一全局文件（如果未禁用） |
| 6 | `${OPENCODE_CONFIG_DIR}/AGENTS.md` | 环境变量指定目录 |
| 7 | `config.instructions` 数组 | 显式指引的所有文件/URL |

#### (http://AGENTS.md "AGENTS.md") 和 instructions 内容会被压缩吗？

**不会。** OpenCode 会**原样读取并注入**所有规则文件内容，不做任何压缩、截断或摘要处理。

**代码实现**（`packages/opencode/src/session/system.ts` 第 119-131 行）：

```TypeScript
// 代码块
const foundFiles = Array.from(paths).map(
  (p) =>
    Bun.file(p)
      .text() // 直接读取原始文本，不压缩
      .catch(() => "")
      .then((x) => "Instructions from: " + p + "\n" + x), // 原样拼接
)
const foundUrls = urls.map(
  (url) =>
    fetch(url, { signal: AbortSignal.timeout(5000) })
      .then((res) => (res.ok ? res.text() : ""))
      .catch(() => "")
      .then((x) => (x ? "Instructions from: " + url + "\n" + x : "")), // URL 内容也原样注入
)
return Promise.all().then((result) => result.filter(Boolean))

```

**处理流程**：


1. `Bun.file(p).text()` - 读取文件完整原始内容
2. 拼接前缀 `"Instructions from: <path>\n"`
3. 原样注入 system prompt

**需要注意的外部限制**：


| 限制来源 | 说明 | 建议 |
| --- | --- | --- |
| LLM Context Window | instructions 太长可能超出模型上下文限制 | 控制单文件大小，避免过多冗余 |
| Token 计费 | 每次 LLM 调用都会消耗 instructions 的 token | 精简内容，只保留必要规则 |
| API 请求大小 | 某些 provider 可能有请求体大小限制 | 拆分大文件，使用多个小文件 |

**最佳实践**：


- 保持 `AGENTS.md` 精简（建议 < 2000 行）
- 使用 `config.instructions` 按需加载，而非全部写入一个文件
- 避免在规则文件中包含大量示例代码（改用引用路径让 LLM 自行读取）


---

### 5. 在 `AGENTS.md` 里添加路径指引是否 OK？

**可以，但要搞清楚它不会被"特殊解析"成 include。**

#### 现有实现

当前框架中：


- `AGENTS.md` 的内容会被**原样作为 system instructions** 注入
- 系统**不会自动识别**类似 `include: ./foo.md` 然后去读文件（除非自己改代码实现）

#### 三种方案对比


| 方案 | 实现方式 | 确定性 | 说明 |
| --- | --- | --- | --- |
| 1. `config.instructions` | 在配置中显式指引额外规则文件 | ⭐⭐⭐ 最高 | 框架级实现，会真实读取内容注入系统提示 |
| 2. 直接写入 `AGENTS.md` | 把所有规则/指引内容直接写进 (http://AGENTS.md "AGENTS.md") | ⭐⭐⭐ 最高 | 简单粗暴，但文件可能变很大 |
| 3. 在 `AGENTS.md` 中写路径指引 | "请参考 ./specs/project.md" | ⭐⭐ 中等 | 依赖 LLM 是否执行，不是强制机制 |
| 4. 实现 include 语法 | 修改 `SystemPrompt.custom()` 解析 (http://AGENTS.md "AGENTS.md") 中的 include 指令 | ⭐⭐⭐ 最高 | 需要改代码实现，但最灵活 |

#### 推荐做法

按确定性从高到低：


1. **使用 config.instructions**（推荐）

  - 在 `.opencode/config.json` 或 `.opencode/config.yaml` 中添加：
```JSON
// 代码块
{
  "instructions": 
}

```


  - 这样框架会自动读取所有这些文件内容并注入系统提示

2. **在 AGENTS.md 中直接编写所有规则**

  - 简单直接，但文件维护成本较高

3. **在 AGENTS.md 中添加文本指引**

  - 如 "关于项目架构，请参考 CORE_ARCHITECTURE.md"
  - 但这依赖 LLM 是否实际去阅读

4. **修改框架实现 include 指令**

  - 如果需要"像真正的 include 一样强制加载"，需要改 `SystemPrompt.custom()` 让它解析 (http://AGENTS.md "AGENTS.md") 中的特殊语法



---

## Configuration Files

### 规则文件的优先级和加载顺序

OpenCode 支持多种规则文件，按以下顺序加载（只取第一类命中的）：


1. **项目级规则**（从工作目录向上到 worktree）

  - `AGENTS.md` → `CLAUDE.md` → `CONTEXT.md`

2. **全局规则**（只取第一个存在的）

  - `~/.opencode/config/AGENTS.md`
  - `~/.claude/CLAUDE.md`（可禁用）
  - `${OPENCODE_CONFIG_DIR}/AGENTS.md`（如指定）

3. **显式指引**（`config.instructions` 数组中的所有文件/URL）

### config.instructions 配置指南

`config.instructions` 是在 OpenCode 配置文件中明确指定需要加载的规则文件列表。与自动查找不同，这里列出的**所有文件内容都会被读取并注入系统提示词**。

#### 配置位置

配置可以放在以下任何位置（按优先级）：


1. **项目配置**（最高优先级）

  - `./opencode.jsonc` 或 `./opencode.json`（项目根目录及向上的任何层级）

2. **.opencode 目录配置**

  - `./.opencode/opencode.jsonc` 或 `./.opencode/opencode.json`
  - `~/.opencode/opencode.jsonc` 或 `~/.opencode/opencode.json`

3. **全局配置**

  - `~/.opencode/config/opencode.json` 等

4. **环境变量指定**

  - `OPENCODE_CONFIG` - 自定义配置文件路径
  - `OPENCODE_CONFIG_DIR` - 自定义配置目录
  - `OPENCODE_CONFIG_CONTENT` - JSON 字符串形式的内联配置


#### 完整配置示例

```JSON
// 代码块
{
  "instructions": 
}

```

#### 支持的路径格式


| 格式 | 说明 | 例子 |
| --- | --- | --- |
| 相对路径 | 相对于项目根目录或配置文件所在目录 | `STYLE_GUIDE.md`, `./specs/api.md` |
| Glob 模式 | 使用 `*` 匹配多个文件 | `./specs/*.md`, `docs/**/*.md` |
| `~/` 路径 | 用户主目录下的文件 | `~/my-rules.md`, `~/.opencode/custom.md` |
| 绝对路径 | 完整的文件系统路径 | `/etc/opencode/rules.md` |
| HTTP/HTTPS URL | 远程文件 | `https://my-docs.com/guidelines.md` |

#### 处理流程

当 OpenCode 启动时，对于 `instructions` 数组中的每一项：


1. **路径解析**

  - 相对路径：相对于配置文件所在目录
  - `~/` 路径：展开为用户主目录
  - URL：使用 fetch 获取内容

2. **文件读取**

  - 对于本地文件路径：读取文件内容
  - 对于 URL：发送 HTTP GET 请求（5 秒超时）

3. **内容注入**

  - 每个文件内容会被包装成：
```Markdown
// 代码块
Instructions from: <路径或URL>
<文件内容>

```


  - 注入到系统提示词中

4. **数组合并**

  - 多个配置来源的 `instructions` 会合并（不重复）
  - 合并顺序：remote config → global config → project config → CLI env


#### 实际使用示例

**示例 1：项目中的本地文件**

文件结构：

```Markdown
// 代码块
project/
├── opencode.json
├── AGENTS.md
└── docs/
    ├── architecture.md
    └── api-guide.md

```

配置：

```JSON
// 代码块
{
  "instructions": 
}

```

**示例 2：混合本地和远程**

```JSON
// 代码块
{
  "instructions": 
}

```

**示例 3：使用 Glob 模式**

```JSON
// 代码块
{
  "instructions": 
}

```

#### 错误处理


- **文件不存在**：警告日志，继续处理其他文件
- **URL 请求失败**：5 秒超时后忽略，继续处理
- **配置文件语法错误**：会抛出 `ConfigJsonError` 错误

#### JSONC 特殊语法

OpenCode 支持在配置文件中使用 JSONC 格式（带注释和尾逗号）。还支持特殊替换：

```Markdown
// 代码块
{
  // 环境变量替换
  "instructions": ,
}

```

### 如何在项目中添加自定义指引

#### 方法一：使用 (http://AGENTS.md "AGENTS.md")（推荐简单项目）

在项目根目录创建 `AGENTS.md`，内容格式参考 `packages/opencode/AGENTS.md`。


- 优点：自动检测，无需显式配置
- 缺点：只能包含一个文件

#### 方法二：使用 config.instructions（推荐复杂项目）

在 `opencode.json` 或 `.opencode/opencode.json` 中添加：

```JSON
// 代码块
{
  "instructions": 
}

```


- 优点：灵活，支持多文件、glob、URL、环境变量
- 缺点：需要显式配置

#### 方法三：使用全局 (http://AGENTS.md "AGENTS.md")（用户级）

在 `~/.opencode/config/AGENTS.md` 下创建（应用于所有项目）。


- 优点：一次配置，全局生效
- 缺点：只能包含一个文件，优先级较低

#### 方法四：使用环境变量（临时/CI/CD）

```Shell
// 代码块
export OPENCODE_CONFIG_CONTENT='{"instructions":}'
opencode

```


---

## Tools & Extensions

### Skill 和 MCP 的区别是什么？


| 特性 | Skill | MCP |
| --- | --- | --- |
| 加载方式 | 文件扫描 glob | 配置项 + 连接 |
| 文件格式 | (http://SKILL.md "SKILL.md")（Markdown） | 服务端定义工具 |
| 加载路径 | `.opencode/skill/`, `.claude/skills/` | 无路径，通过 Config.mcp 配置 |
| 用途 | 工作流定义/命令模板 | 外部服务工具集成 |
| 连接方式 | N/A | HTTP/stdio |


---

## Troubleshooting

### Q: 我的 (http://AGENTS.md "AGENTS.md") 为什么没有被加载？

检查以下几点：


1. **文件位置**

  - 是否在项目根目录或某个中间层目录？（从当前工作目录向上查找）
  - 或是否在 `~/.opencode/config/` 或 `~/.claude/` 下？

2. **文件名**

  - 确保是 `AGENTS.md`（区分大小写，在 macOS/Linux 上）
  - 如果项目有其他规则文件（`CLAUDE.md`），只会加载第一种类型

3. **内容**

  - 确保内容有效（例如 YAML frontmatter 正确）
  - 使用 `opencode debug` 检查加载情况


### Q: 如何自定义 Skill 的加载路径？

当前不支持自定义加载路径。Skill 只会从以下位置加载：


- `.opencode/` 下的 `{skill,skills}/**/SKILL.md`
- `.claude/` 下的 `skills/**/SKILL.md`

如果需要其他路径，建议在 `.opencode` 中创建软链接或复制文件。

### Q: MCP server 启动失败，怎么调试？

检查以下几点：


1. **配置正确性**
```JSON
// 代码块
{
  "mcp": {
    "my-server": {
      "type": "local",
      "command": ,
      "enabled": true
    }
  }
}

```


2. **查看日志**

  - 查看 OpenCode 日志输出
  - 检查 `mcp.timeout` 配置（默认 30 秒）

3. **测试连接**

  - 对于远程 MCP：检查 URL 和网络连接
  - 对于本地 MCP：检查命令是否可执行

4. **环境变量**

  - 确保本地 MCP 的 `mcp.environment` 配置正确



---

## Advanced Topics

### 系统提示词注入流程

每次 LLM 调用时，系统提示词的组成顺序如下：

```TypeScript
// 代码块
system: 

```

### 规则文件的优先级排序

当多个规则文件被加载时：


1. 本地项目级 (http://AGENTS.md "AGENTS.md")（可能多层）
2. 全局 (http://AGENTS.md "AGENTS.md")（单一）
3. config.instructions 中的所有文件/URL

后加载的规则会追加到系统提示词中，可能会覆盖或补充前面的规则。

### 配置合并行为详解

OpenCode 配置的合并规则重要理解，特别是当配置来自多个来源时：

#### 配置源优先级


| 优先级（低→高） | 来源 | 覆盖规则 |
| --- | --- | --- |
| 1 | 远程预配置 | 被所有本地配置覆盖 |
| 2 | `~/.opencode/config/opencode.json` (global) | 被项目配置覆盖 |
| 3 | `OPENCODE_CONFIG` env var | 被 content env var 覆盖 |
| 4 | `.opencode/opencode.json` or `opencode.json` (project) | 被 content env var 覆盖 |
| 5 | `OPENCODE_CONFIG_CONTENT` env var (highest) | 无 |

#### 对象字段的合并规则

对于大多数配置字段（如 `model`, `providers` 等）：


- **后面的配置完全覆盖前面的**
- 不会进行字段级合并

示例：

```JSON
// 代码块
// 全局配置
{ "model": { "provider": "claude", "options": { "a": 1, "b": 2 } } }

// 项目配置
{ "model": { "provider": "openai", "options": { "a": 10 } } }

// 结果：项目配置完全覆盖全局
{ "model": { "provider": "openai", "options": { "a": 10 } } }
// "b": 2 会丢失

```

#### 数组字段的合并规则（特殊）

**`instructions 数组是特例`**，使用 `mergeConfigConcatArrays()` 函数特殊处理：

```TypeScript
// 代码块
// 全局
{ "instructions":  }

// 项目
{ "instructions":  }

// 结果：两个数组连接
{ "instructions":  }

```

这样做的好处是：全局规则始终应用 + 项目规则追加。

#### 实际例子

**场景**：同时有全局配置和项目配置

**全局**: `~/.opencode/config/opencode.json`

```JSON
// 代码块
{
  "model": { "provider": "claude" },
  "instructions": 
}

```

**项目**: `./opencode.json`

```JSON
// 代码块
{
  "model": { "provider": "openai" },
  "instructions": 
}

```

**最终结果**：

```JSON
// 代码块
{
  "model": { "provider": "openai" },
  "instructions": 
}

```

解释：


- `model.provider` 被项目的 `openai` 覆盖全局的 `claude`
- `instructions` 数组被**合并**：全局规则 + 项目规则


---

## Complete Configuration Examples

### Minimal Project Setup

**File: opencode.json** (project root)

```JSON
// 代码块
{
  "instructions": 
}

```

**File: AGENTS.md** (project root)

```Markdown
// 代码块
# My Project Guidelines

You are an AI coding agent working on this project.

## Code Style

- Use 2-space indentation
- Prefer const over let
- Keep functions under 50 lines

## Architecture

This is a Bun + TypeScript project using the MDP framework.

```

**Result**: When you run `opencode` in this project, both files are loaded into system prompt.


---

### Medium Project Setup (Recommended for Most)

**File structure**:

```Markdown
// 代码块
my-project/
├── opencode.json
├── AGENTS.md
├── STYLE_GUIDE.md
├── CORE_ARCHITECTURE.md
└── specs/
    ├── api.md
    └── database.md

```

**File: opencode.json**

```JSON
// 代码块
{
  "instructions": 
}

```

**File: AGENTS.md**

```Markdown
// 代码块
# Project: My Service

## Overview

This is a TypeScript service built with MDP framework.

## What This Agent Can Do

- Modify source code in `src/`
- Run tests with `bun test`
- Deploy to staging with `bun run deploy:staging`

## What This Agent MUST NOT Do

- Modify database migrations directly (use schema tool)
- Delete files without asking
- Run destructive commands in production

```

**File: STYLE_GUIDE.md**

```Markdown
// 代码块
# Code Style

## TypeScript

- Use strict mode
- Avoid `any` type
- Prefer interfaces over types for public APIs

## Formatting

- Use Prettier with defaults
- 80-character line limit for comments

```

**File: CORE_ARCHITECTURE.md**

```Markdown
// 代码块
# Architecture

## Project Structure

- `src/` - Application code
- `test/` - Unit and integration tests
- `scripts/` - Build and deployment scripts

## Technology Stack

- Runtime: Bun
- Language: TypeScript
- Framework: MDP
- Database: PostgreSQL with Zebra ORM

```

**File: specs/api.md**

```Markdown
// 代码块
# API Specification

## Authentication

All endpoints require Bearer token in `Authorization` header.

## Error Codes

- 400: Bad Request
- 401: Unauthorized
- 500: Internal Server Error

```

**Result**: When you run `opencode` in this project:


1. `AGENTS.md` is loaded (agent capabilities)
2. `STYLE_GUIDE.md` is loaded (code conventions)
3. `CORE_ARCHITECTURE.md` is loaded (system design)
4. All `.md` files in `specs/` are loaded (API docs)

All content is concatenated into system prompt in order.


---

### Advanced Setup (Multiple Environments)

**File structure**:

```Markdown
// 代码块
my-project/
├── opencode.json              # Base config
├── .opencode/
│   ├── opencode.prod.json     # Production overrides
│   └── opencode.dev.json      # Development overrides
├── AGENTS.md
└── docs/
    ├── shared/
    │   ├── style.md
    │   └── architecture.md
    └── prod/
        ├── deployment.md
        └── monitoring.md

```

**File: opencode.json** (base - always loaded)

```JSON
// 代码块
{
  "instructions": 
}

```

**File: .opencode/opencode.dev.json** (development - loaded when present)

```JSON
// 代码块
{
  "instructions": 
}

```

**File: .opencode/opencode.prod.json** (production - loaded when present)

```JSON
// 代码块
{
  "instructions": 
}

```

**Usage**:

```Shell
// 代码块
# Development (loads opencode.json + .opencode/opencode.dev.json)
opencode

# Production (loads .opencode/opencode.prod.json + remote URL)
opencode --config .opencode/opencode.prod.json

```

**Config loading order** (higher priority wins, arrays merge):


| Priority | Source |
| --- | --- |
| 1 (lowest) | `opencode.json` |
| 2 | `.opencode/opencode.json` |
| 3 | `.opencode/opencode.dev.json` or `.opencode/opencode.prod.json` |
| 4 (highest) | `OPENCODE_CONFIG_CONTENT` env var |

**Result**: Base rules always applied, with environment-specific additions.


---

### User Global Setup

**File: ~/.opencode/config/opencode.json** (affects ALL projects)

```JSON
// 代码块
{
  "instructions": 
}

```

**File: ~/.opencode/config/personal-guidelines.md**

```Markdown
// 代码块
# My Personal Guidelines

## Preferences

- I prefer TypeScript over JavaScript
- Always use kebab-case for file names
- Create unit tests for new functions

## Security

- Never log API keys or tokens
- Always use HTTPS
- Validate user input on server side

```

**Result**: These rules apply to every OpenCode session, for any project.

**Override in project**: Project `opencode.json` instructions take precedence over global.


---

### Using Remote Configuration

**File: opencode.json** (project root)

```JSON
// 代码块
{
  "instructions": 
}

```

**What happens**:


1. At startup, OpenCode fetches both URLs
2. Content is read with 5-second timeout
3. If URL fails, warning is logged but process continues
4. Content is injected into system prompt

**Best practices**:


- Use stable documentation URLs
- Ensure URLs are accessible from your network
- Provide fallback local copies if possible


---

### Using Environment Variables

**Inline config** (temporary, for one session):

```Shell
// 代码块
export OPENCODE_CONFIG_CONTENT='{"instructions":}'
opencode

```

**Custom config directory**:

```Shell
// 代码块
export OPENCODE_CONFIG_DIR=~/.opencode/prod-configs
opencode
# Will look for ${OPENCODE_CONFIG_DIR}/AGENTS.md

```

**Custom config file**:

```Shell
// 代码块
opencode --config /path/to/custom-opencode.json

```

**Env vars in config** (JSONC syntax):

```Markdown
// 代码块
{
  // Reference environment variables
  "instructions": ,
}

```


---

### Real-World Example: MDP Project

**File: opencode.json**

```JSON
// 代码块
{
  "instructions": 
}

```

**File: AGENTS.md**

```Markdown
// 代码块
# MDP Service Development

## Build & Test

- Build: `bun run build` or `./gradlew build`
- Test: `bun test` or `./gradlew test`
- Dev: `bun dev`

## Code Generation

- Generate API code: `./script/gen-api.ts`
- Generate Database: `./script/gen-db.ts`

## Deployment

- Staging: `mdp deploy --env test`
- Production: `mdp deploy --env prod`

## Framework Rules

- Use MDP agents from `packages/opencode`
- Follow the three-layer architecture (API → Service → DAO)
- Always add error handling for external calls
- Use Zebra for database access

## Do NOT

- Commit secrets or API keys
- Modify generated code directly
- Use raw SQL queries

```

**File: STYLE_GUIDE.md**

```Markdown
// 代码块
# Code Style for MDP Services

## Java/Kotlin

- Use 2-space indentation
- Follow Google Java Style Guide
- Add @NonNull/@Nullable annotations
- Write JavaDoc for public methods

## TypeScript (if applicable)

- Use 2-space indentation
- Strict null checks enabled
- No implicit any
- Use interfaces for contracts

## Git

- Commit messages: "action: description"
- Example: "fix: handle null user in profile service"

```

**File: CORE_ARCHITECTURE.md**

```Markdown
// 代码块
# Architecture

## System Design

- API Layer: HTTP endpoints
- Service Layer: Business logic
- DAO Layer: Database access via Zebra
- Cache: Redis for frequently accessed data

## Key Classes

- `UserService` - User management
- `OrderService` - Order processing
- `NotificationService` - Event notifications

## Technology Stack

- Language: Java 17 / TypeScript
- Framework: MDP + Spring
- Database: PostgreSQL (Zebra ORM)
- Cache: Redis
- Message Queue: Kafka (MAFKA)

```

**Result**:


- Agent has complete context of MDP framework
- Knows project-specific conventions
- Can follow deployment procedures
- Has access to external documentation


---

## Configuration Priority Reference

When multiple config sources exist, they are applied in this order (later overrides):


1. **Well-known remote configs** (if applicable)
2. **Global config**: `~/.opencode/config/opencode.json`
3. **Environment variable**: `OPENCODE_CONFIG`
4. **Project config** (向上查找): `opencode.json` or `.opencode/opencode.json`
5. **Env var content**: `OPENCODE_CONFIG_CONTENT` (JSON string)

**For instructions array specifically**: Arrays from multiple configs are **concatenated**, not replaced.


---

### Quick Reference: Where to Put What


| Goal | Location | Example |
| --- | --- | --- |
| Project rules | `./AGENTS.md` | Capabilities, constraints |
| Code style | `./STYLE_GUIDE.md` | Indentation, naming conventions |
| Architecture | `./CORE_ARCHITECTURE.md` | System design, data flow |
| API docs | `./specs/api.md` | Endpoints, request/response |
| Database | `./specs/database.md` | Schema, relationships |
| Deployment | `./scripts/deployment.md` | How to deploy, environments |
| Personal rules | `~/.opencode/config/personal-guidelines.md` | Your preferences (global) |
| Remote docs | URL in `config.instructions` | Company standards, shared guidelines |


---

## Related Documentation


- (./packages/opencode/AGENTS.md "AGENTS.md") - OpenCode agent guidelines
- (./STYLE_GUIDE.md "STYLE_GUIDE.md") - Project code style guide
- (./CORE_ARCHITECTURE.md "CORE_ARCHITECTURE.md") - Project architecture
- (./README.md "README.md") - Project overview


