# 20260305 - OpenClaw 多 Discord 机器人配置完整教程

## 目标

在一个 Discord 服务器中配置多个机器人，每个机器人对应 OpenClaw 中的不同 Agent，实现多角色协作。

## 前置准备

- Discord 账号
- OpenClaw 已安装并运行
- 对 Discord 服务器有管理员权限

---

## 第一步：在 Discord 开发者门户创建机器人

### 1.1 创建应用并添加机器人

1. 访问 [Discord 开发者门户](https://discord.com/developers/applications)
2. 点击右上角 **新建应用程序**
3. 输入应用名称（例如：`Architect Bot`），点击 **创建**
4. 在左侧菜单点击 **机器人**
5. 点击 **添加机器人** → 确认
6. 在 **令牌（TOKEN）** 区域，点击 **复制** 按钮
7. **妥善保存这个 Token**

### 1.2 启用必需的权限

在 **机器人** 页面向下滚动，找到 **特权网关意图**：
- ✅ **消息内容意图**（MESSAGE CONTENT INTENT）— 必须启用
- ✅ **服务器成员意图**（SERVER MEMBERS INTENT）— 推荐启用

### 1.3 生成邀请链接并邀请机器人

1. 在左侧菜单点击 **OAuth2** → **URL 生成器**
2. 在 **范围（SCOPES）** 中勾选：`bot`、`applications.commands`
3. 在 **机器人权限** 中勾选：查看频道、发送消息、读取消息历史、嵌入链接、附加文件
4. 复制生成的 URL，在浏览器中打开
5. 选择您的服务器，点击 **授权**

### 1.4 重复创建其他机器人

按照 1.1-1.3 步骤创建：Reviewer Bot、Scribe Bot、Code Bot 等。

---

## 第二步：获取 Discord 服务器 ID

1. 打开 Discord 客户端 → 用户设置 → 高级 → 开启 **开发者模式**
2. 右键点击服务器图标 → **复制服务器 ID**

---

## 第三步：配置 OpenClaw

### 3.1 配置多个 Discord 账号

```bash
openclaw config set channels.discord.accounts.main.token "YOUR_MAIN_BOT_TOKEN"
openclaw config set channels.discord.accounts.architect.token "YOUR_ARCHITECT_BOT_TOKEN"
openclaw config set channels.discord.accounts.reviewer.token "YOUR_REVIEWER_BOT_TOKEN"
openclaw config set channels.discord.accounts.scribe.token "YOUR_SCRIBE_BOT_TOKEN"
openclaw config set channels.discord.accounts.code.token "YOUR_CODE_BOT_TOKEN"
```

### 3.2 配置服务器白名单

```bash
openclaw config set 'channels.discord.guilds.YOUR_GUILD_ID' '{"requireMention":false,"channels":{"general":{"allow":true}}}'
```

### 3.3 配置 Agent 绑定（关键步骤！）

在 `~/.openclaw/openclaw.json` 的 `bindings` 部分：

```json
"bindings": [
  {
    "agentId": "architect",
    "match": {
      "channel": "discord",
      "account": "architect"
    }
  },
  {
    "agentId": "reviewer",
    "match": {
      "channel": "discord",
      "account": "reviewer"
    }
  }
]
```

**关键点**：
- `agentId` 必须与 `agents.list` 中的 `id` 一致
- `match.account` 必须与 `channels.discord.accounts` 中的账号名一致
- **不要遗漏 `account` 字段**

---

## 第四步：重启 OpenClaw Gateway

```bash
openclaw gateway restart
openclaw status
```

---

## 第五步：在 Discord 中测试

```text
@Architect Bot 设计一个用户登录系统的技术方案
@Reviewer Bot 审查上面的方案，提出改进建议
@Scribe Bot 整理上面的讨论，生成最终文档
@Code Bot 根据上面的方案，写出核心代码
```

---

## 故障排查

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| 机器人显示离线 | Token 错误或 Gateway 未重启 | 检查 Token，重启 Gateway |
| 机器人不响应 | 未启用消息内容意图 | 在开发者门户启用 |
| 多个机器人响应同一条消息 | bindings 缺少 account 字段 | 确保每个 binding 有 account |

---

## 最佳实践

1. **模型选择**：架构师/审查员用 Claude Sonnet，开发者用 Codex，文档管理员用 Claude Sonnet
2. **安全建议**：定期轮换 Token，使用 allowlist 限制服务器，不要在公开仓库提交 Token
