# 20260304 - Tmux 内多 Agent 直接协作方案（失败）

## 方案概述

**核心思路**：在 Tmux 内部创建多个 Panel，每个 Panel 运行一个 CLI Agent，让它们通过共享的文件系统或标准输入输出直接交互。

## 架构设计

```
┌─────────────────────────────────────────────────┐
│         Tmux Session: research-team             │
├─────────────────┬─────────────────┬─────────────┤
│   Panel 0       │   Panel 1       │   Panel 2   │
│  Architect      │   Reviewer      │   Scribe    │
│  (ccd)          │   (codex)       │   (mccd)    │
│                 │                 │             │
│  写方案 ────────>│  审查方案 ──────>│  发布到KM   │
└─────────────────┴─────────────────┴─────────────┘
         ▲                 │                │
         └─────────────────┴────────────────┘
              通过文件或管道通信
```

## 技术实现

### 1. 环境准备

```bash
tmux new-session -d -s research-team "ccd"
tmux split-window -h -t research-team "codex"
tmux split-window -h -t research-team "mccd"
```

### 2. 尝试的通信方式

**方式 A：共享文件系统**
- Architect 写入 `draft.md`
- Reviewer 读取 `draft.md`，写入 `review.md`
- 循环直到 Reviewer 批准
- Scribe 读取最终的 `draft.md`，发布到 KM

**方式 B：标准输入输出重定向**

```bash
tmux pipe-pane -t research-team:0 "tee architect_output.txt"
tmux send-keys -t research-team:1 "cat architect_output.txt | codex review" Enter
```

**方式 C：Tmux 的 send-keys 串联**

```bash
tmux send-keys -t research-team:0 "设计方案" Enter
# 等待完成...
tmux send-keys -t research-team:1 "审查 draft.md" Enter
```

## 失败原因分析

### 1. 缺乏中控协调
三个 Agent 都是"哑终端"，不知道什么时候该开始工作、上游是否完成、下游是否准备好接收。本质上还是"人工调度"。

### 2. 状态同步困难
无法实时感知各 Agent 的执行状态，需要频繁切换 Panel 查看输出，容易出现"死锁"。

### 3. 错误处理缺失
当某个 Agent 失败时，整个流程卡住，需要人工介入。

### 4. CLI Agent 的交互模式不适合自动化
`ccd`、`codex`、`mccd` 都是交互式 CLI 工具，不是"守护进程"，无法持续监听文件变化。

## 核心问题总结

根本问题在于：**试图让"工具"直接协作，而不是让"智能体"协作**。

- CLI Agent 是工具：它们需要人（或智能体）来操作
- OpenClaw 是智能体：它能理解任务、做决策、调度工具

**类比**：
- ❌ 失败方案 = 把锤子、锯子、钉子放在一起，期望它们自己组装成桌子
- ✅ 成功方案 = 木匠（OpenClaw）拿着锤子、锯子、钉子，按照图纸组装桌子

## 经验教训

1. **明确"智能体"与"工具"的边界**：`ccd`、`codex`、`mccd` 是工具，不是智能体，需要一个智能体来调度。
2. **协作需要"中控"**：多 Agent 协作必须有一个"大脑"来统筹全局。
3. **状态可观测性是关键**：自动化的前提是"可观测"，看不见就无法控制。
4. **分步执行优于一次性执行**：复杂任务要拆解，小步快跑，及时反馈。

最终，我们放弃了这个方案，转而采用"OpenClaw 中控 + Tmux 调度"的方案，并取得了成功。

**核心教训**：不要试图让工具直接协作，而是让智能体调度工具。
