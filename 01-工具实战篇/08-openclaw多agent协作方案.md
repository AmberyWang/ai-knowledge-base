# 20260305 - OpenClaw 多 Agent 协作方案（成功）

## 方案概述

**核心思路**：OpenClaw 作为中控（Orchestrator），通过 Tmux 调度多个 CLI Agent 协作完成复杂任务。

## 架构设计

```
┌─────────────────────────────────────────────────┐
│           OpenClaw (Main Session)               │
│              中控 / 调度器                        │
└──────────────┬──────────────────────────────────┘
               │
               │ tmux send-keys / capture-pane
               │
    ┌──────────┴──────────┬──────────────┐
    │                     │              │
┌───▼────┐          ┌────▼───┐      ┌───▼────┐
│Architect│         │Reviewer│      │ Scribe │
│(ccd)   │         │(codex) │      │ (mccd) │
└────────┘          └────────┘      └────────┘
  写方案              审查方案         发布到KM
```

## 技术实现

### 1. 环境准备

```bash
tmux new-session -d -s orchestrator -n architect
tmux new-window -t orchestrator:1 -n reviewer
tmux new-window -t orchestrator:2 -n scribe
```

### 2. 启动 Agent

```bash
tmux send-keys -t orchestrator:architect "ccd" Enter
tmux send-keys -t orchestrator:reviewer "codex" Enter
tmux send-keys -t orchestrator:scribe "mccd" Enter
```

### 3. 任务调度流程

OpenClaw 的核心职责：
1. **发送指令**：通过 `tmux send-keys` 向目标窗口发送任务
2. **读取输出**：通过 `tmux capture-pane` 获取 Agent 的执行结果
3. **决策路由**：根据输出内容决定下一步操作

```bash
# 1. 向 Architect 发送任务
tmux send-keys -t orchestrator:architect "设计上下文工程体系方案" Enter

# 2. 等待并读取结果
sleep 30
tmux capture-pane -t orchestrator:architect -p > architect_output.txt

# 3. 将结果发给 Reviewer
tmux send-keys -t orchestrator:reviewer "审查以下方案：$(cat architect_output.txt)" Enter

# 4. 读取 Review 意见
sleep 20
tmux capture-pane -t orchestrator:reviewer -p > review_feedback.txt

# 5. 根据反馈决定：修改 or 发布
if grep -q "LGTM" review_feedback.txt; then
    tmux send-keys -t orchestrator:scribe "将方案写入 KM" Enter
else
    tmux send-keys -t orchestrator:architect "根据以下意见修改：$(cat review_feedback.txt)" Enter
fi
```

## 实战案例

### 任务目标

针对 KM 文档中的"上下文工程体系"部分，设计可落地的技术方案并发布到 KM。

### 执行流程

| 阶段 | Agent | 输出 | 耗时 |
|------|-------|------|------|
| 架构设计 | Architect | `context_engineering_design.md`（17KB） | ~5分钟 |
| 方案评审 | Reviewer | 3 条核心改进建议 | ~1分钟 |
| 迭代修改 | Architect | 完整技术设计文档（17KB） | ~3分钟 |
| 发布到 KM | Scribe | KM 子页面 | ~2.5分钟 |

**总耗时**：约 12 分钟（全自动）

## 关键成功因素

### 1. 明确的角色分工
- **Architect**：专注于方案设计，不做评审
- **Reviewer**：专注于质量把关，不做实现
- **Scribe**：专注于发布，不做内容修改

### 2. OpenClaw 的中控能力
- **状态感知**：通过 `capture-pane` 实时了解 Agent 执行状态
- **智能决策**：根据输出内容判断是否需要迭代
- **异常处理**：当 Agent 卡住时，能够中断并重试

### 3. 分步执行策略
- 避免一次性生成大文档
- 分章节迭代，每次只生成一个章节
- 每次操作后立即检查文件大小和输出状态

## 优势与局限

### 优势
1. 完全自动化：从需求到发布，无需人工干预
2. 可观测性强：可随时进入 Tmux 查看 Agent 状态
3. 易于调试：每个 Agent 独立运行，问题定位清晰
4. 可扩展性好：可轻松增加新的 Agent 角色

### 局限
1. 依赖 Tmux 环境
2. 需要手动启动（首次需创建 Tmux 会话并启动 Agent）
3. 状态轮询有一定延迟

## 适用场景

- ✅ 文档生成与发布（技术方案、设计文档）
- ✅ 代码审查与迭代（生成代码 → 审查 → 修改 → 合并）
- ✅ 知识沉淀流程（提取 → 结构化 → 发布）
- ❌ 需要实时交互的场景
- ❌ 对延迟敏感的任务

## 总结

这套方案的核心价值在于：**将 OpenClaw 的智能决策能力与 CLI Agent 的专业能力结合起来**，实现了真正的"AI 协作"而非"AI 对话"。
