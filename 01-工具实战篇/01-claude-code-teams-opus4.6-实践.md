# 20260212 Claude Code Teams+opus4.6 实践 for 解析源码

# 一、配置

参考 [Agent Teams 文档](https://code.claude.com/docs/en/agent-teams#choose-a-display-mode)

```JSON
// ~/.claude/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

# 二、使用

**Prompt**
帮我组建一个团队带我学习openclaw源码、核心亮点、关键模块的核心原理、并在看代码过程中看是否可以基于它的能力寻找和发现如何基于openclaw扩展开发出为自己或团队日常工作提效的做功点（可以是基于openclaw的扩展能力做，也可以是修改openclaw源码复刻一个垂直场景等等不设限）：

1. 一个负责提出问题
2. 一个负责探查源码并回答问题
3. 一个负责发现机会发现产品的可能性
4. 一个负责记录成一系列的学城文档
