# OpenClaw 源码学习 Q&A 精选


<!-- TOC -->

# OpenClaw 源码学习 Q&A 精选

## 一、架构设计类

### Q1: OpenClaw 与 Claude Desktop 的核心区别是什么?

**A:** 三个本质差异:


1. **部署方式**

  - OpenClaw: 本地运行,数据不出域
  - Claude Desktop: 云端依赖



1. **渠道支持**

  - OpenClaw: 15+ 渠道统一接入 (WhatsApp/Telegram/飞书等)
  - Claude Desktop: 仅桌面应用



1. **扩展性**

  - OpenClaw: 插件化 Skills 系统,完全开源
  - Claude Desktop: 封闭生态


**适用场景**:


- 企业内网部署 → OpenClaw
- 多渠道统一助手 → OpenClaw
- 简单个人使用 → Claude Desktop


---

### Q2: Gateway、Agent、Channel 三者的关系?

**A:** 分层协作关系:

```Text
用户消息 (WhatsApp)
    ↓
Channel (统一消息格式)
    ↓
Gateway (路由分发)
    ↓
Agent (AI 处理)
    ↓
Gateway (响应返回)
    ↓
Channel (发送回复)
    ↓
用户接收
```

**职责划分**:


- **Channel**: 适配不同平台协议,统一消息格式
- **Gateway**: 消息路由、会话管理、认证鉴权
- **Agent**: 模型调用、工具执行、上下文管理

**关键设计**:


- Channel 与 Agent 解耦,支持独立扩展
- Gateway 作为中间层,负责协调


---

### Q3: OpenClaw 如何避免 token 溢出?

**A:** 三层压缩策略:

**1. 保留策略**

```Typescript
// 保留系统消息 + 最近 10 条
const systemMessages = messages.filter(m => m.role === 'system');
const recentMessages = messages.slice(-10);
```

**2. 压缩策略**

```Typescript
// 中间历史用 AI 总结
const summary = await model.summarize(middleMessages, {
  maxTokens: 500
});
```

**3. Prompt Caching**

```Typescript
// Anthropic Prompt Caching (减少 50% token 消耗)
const cacheableMessages = messages.slice(0, -2).map(m => ({
  ...m,
  cache_control: { type: 'ephemeral' }
}));
```

**配置建议**:

```Json
{
  "compaction": {
    "enabled": true,
    "maxTokens": 8000,
    "recentCount": 10
  }
}
```


---

### Q4: 如何实现多模型自动降级?

**A:** 优先级 + 重试机制:

```Typescript
const models = ;

async function chatWithFallback(prompt: string) {
  for (const config of models.sort((a, b) => a.priority - b.priority)) {
    try {
      return await callModel(config, prompt);
    } catch (error) {
      console.error(`${config.provider} 失败,尝试降级`);
      continue;
    }
  }
  throw new Error('所有模型调用失败');
}
```

**成本优化**:


- 简单任务用 Haiku/GPT-3.5
- 复杂推理用 Opus/GPT-4
- 本地部署用 Ollama


---

## 二、消息路由类

### Q5: Channel 如何统一不同平台的消息格式?

**A:** 统一抽象层设计:

```Typescript
// 统一消息格式
interface NormalizedMessage {
  id: string;
  channelId: string;
  userId: string;
  content: string;
  timestamp: number;
  metadata: Record<string, any>;
}

// Telegram 转换
function normalizeTelegram(msg: TelegramMessage): NormalizedMessage {
  return {
    id: msg.message_id.toString(),
    channelId: 'telegram',
    userId: msg.from.id.toString(),
    content: msg.text || '',
    timestamp: msg.date * 1000,
    metadata: { chat_id: msg.chat.id }
  };
}

// 飞书转换
function normalizeFeishu(msg: FeishuMessage): NormalizedMessage {
  return {
    id: msg.message_id,
    channelId: 'feishu',
    userId: msg.sender.sender_id.user_id,
    content: JSON.parse(msg.content).text,
    timestamp: parseInt(msg.create_time),
    metadata: { chat_id: msg.chat_id }
  };
}
```

**关键点**:


- 所有 Channel 实现相同接口
- Gateway 只处理统一格式
- 元数据存储平台特有信息


---

### Q6: 如何保证消息顺序性?

**A:** 会话级队列:

```Typescript
class SessionQueue {
  private queues = new Map<string, MessageQueue>();
  
  async process(message: NormalizedMessage) {
    const sessionKey = `${message.channelId}:${message.userId}`;
    
    // 获取或创建队列
    let queue = this.queues.get(sessionKey);
    if (!queue) {
      queue = new MessageQueue();
      this.queues.set(sessionKey, queue);
    }
    
    // 加入队列并等待处理
    return await queue.enqueue(async () => {
      return await agent.process(message);
    });
  }
}
```

**原理**:


- 每个会话一个独立队列
- 消息按顺序处理
- 避免并发冲突


---

### Q7: 如何处理长时间运行的任务?

**A:** 异步响应 + 状态推送:

```Typescript
async function handleLongTask(message: NormalizedMessage) {
  // 1. 立即响应
  await channel.sendMessage(
    message.userId,
    '⏳ 任务已接收,正在处理中...'
  );
  
  // 2. 后台执行
  const taskId = uuid();
  taskManager.run(taskId, async () => {
    const result = await executeTask(message);
    
    // 3. 完成后推送
    await channel.sendMessage(
      message.userId,
      `✅ 任务完成:\n${result}`
    );
  });
  
  return `任务ID: ${taskId}`;
}
```

**适用场景**:


- 代码执行 (>30 秒)
- 大文件处理
- 复杂分析任务


---

## 三、Agent 执行类

### Q8: Agent 执行循环的最大迭代次数是多少?

**A:** 默认 10 次,可配置:

```Typescript
const MAX_ITERATIONS = 10;

async function agentLoop(message: string) {
  let iteration = 0;
  
  while (iteration < MAX_ITERATIONS) {
    iteration++;
    
    const response = await model.chat(prompt);
    
    if (!response.toolCalls) {
      return response; // 无工具调用,结束
    }
    
    await executeTools(response.toolCalls);
  }
  
  throw new Error('达到最大迭代次数');
}
```

**为什么限制迭代次数?**


- 防止死循环
- 控制成本
- 避免超时

**如何处理未完成的任务?**


- 返回中间结果
- 建议拆分任务
- 记录日志用于调试


---

### Q9: 工具调用失败如何重试?

**A:** 指数退避 + 最大重试:

```Typescript
async function executeWithRetry(
  skill: Skill,
  params: any,
  maxRetries = 3
): Promise<any> {
  let lastError: Error;
  
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await skill.execute(params);
    } catch (error) {
      lastError = error;
      
      // 指数退避: 2^i 秒
      await sleep(Math.pow(2, i) * 1000);
      
      console.log(`重试 ${i + 1}/${maxRetries}: ${skill.name}`);
    }
  }
  
  // 所有重试失败
  throw new Error(`工具调用失败: ${lastError.message}`);
}
```

**重试策略**:


- 网络错误 → 重试
- 参数错误 → 不重试
- 速率限制 → 延长等待时间


---

### Q10: 如何在工具间传递上下文?

**A:** ExecutionContext:

```Typescript
interface ExecutionContext {
  userId: string;
  sessionId: string;
  previousResults: Map<string, any>; // 前置工具结果
  metadata: Record<string, any>;
}

// Skill 1: 搜索
const searchSkill: Skill = {
  name: 'search',
  async execute(params, context) {
    const results = await search(params.query);
    
    // 保存到上下文
    context.previousResults.set('search_results', results);
    
    return results;
  }
};

// Skill 2: 总结 (依赖搜索结果)
const summarizeSkill: Skill = {
  name: 'summarize',
  async execute(params, context) {
    // 读取前置结果
    const searchResults = context.previousResults.get('search_results');
    
    return await model.summarize(searchResults);
  }
};
```


---

## 四、Skills 开发类

### Q11: Skill 参数如何验证?

**A:** 使用 Zod 或手动验证:

**方案 1: Zod**

```Typescript
import { z } from 'zod';

const paramsSchema = z.object({
  city: z.string().min(1, '城市名称不能为空'),
  unit: z.enum().default('celsius')
});

export const weatherSkill: Skill = {
  name: 'get_weather',
  async execute(params) {
    const validated = paramsSchema.parse(params);
    return await fetchWeather(validated);
  }
};
```

**方案 2: 手动验证**

```Typescript
function validateParams(input: any, schema: ParameterSchema): any {
  const validated: any = {};
  
  for (const  of Object.entries(schema)) {
    const value = input;
    
    // 必填检查
    if (def.required && value === undefined) {
      throw new Error(`缺少必填参数: ${key}`);
    }
    
    // 类型检查
    if (value !== undefined && typeof value !== def.type) {
      throw new Error(`参数 ${key} 类型错误`);
    }
    
    validated = value ?? def.default;
  }
  
  return validated;
}
```


---

### Q12: 如何限制 Skill 的调用频率?

**A:** 速率限制器:

```Typescript
class RateLimiter {
  private counters = new Map<string, number>();
  
  async check(key: string, limit: number, windowMs: number): Promise<void> {
    const count = this.counters.get(key) || 0;
    
    if (count >= limit) {
      throw new Error('超过速率限制');
    }
    
    this.counters.set(key, count + 1);
    
    // 窗口结束后重置
    setTimeout(() => {
      this.counters.delete(key);
    }, windowMs);
  }
}

// 使用
export const limitedSkill: Skill = {
  name: 'limited_skill',
  rateLimit: {
    maxCalls: 10,
    windowSeconds: 60
  },
  async execute(params, context) {
    const key = `${context.userId}:limited_skill`;
    await rateLimiter.check(key, 10, 60000);
    
    return await doWork(params);
  }
};
```


---

### Q13: Skill 如何请求用户授权?

**A:** 授权流程:

```Typescript
export const sensitiveSkill: Skill = {
  name: 'sensitive_skill',
  requiresApproval: true,
  
  async execute(params, context) {
    // 1. 请求授权
    const approved = await requestApproval({
      userId: context.userId,
      skill: 'sensitive_skill',
      params,
      message: '该操作将访问敏感数据,是否授权?'
    });
    
    if (!approved) {
      throw new Error('用户拒绝授权');
    }
    
    // 2. 执行操作
    return await accessSensitiveData(params);
  }
};

// 授权请求
async function requestApproval(request: ApprovalRequest): Promise<boolean> {
  // 发送授权请求到用户
  await channel.sendMessage(
    request.userId,
    `${request.message}\n\n回复 "yes" 授权, "no" 拒绝`
  );
  
  // 等待用户响应 (30 秒超时)
  const response = await waitForUserResponse(request.userId, 30000);
  
  return response.toLowerCase() === 'yes';
}
```


---

## 五、自动化任务类

### Q14: Cron 表达式如何编写?

**A:** 标准格式:

```Text
┌───────────── 分钟 (0-59)
│ ┌───────────── 小时 (0-23)
│ │ ┌───────────── 日 (1-31)
│ │ │ ┌───────────── 月 (1-12)
│ │ │ │ ┌───────────── 星期 (0-6, 0=周日)
│ │ │ │ │
* * * * *
```

**常用示例**:

```Yaml
# 每天 18:00
"0 18 * * *"

# 工作日 9:00
"0 9 * * 1-5"

# 每周五 17:00
"0 17 * * 5"

# 每 5 分钟
"*/5 * * * *"

# 每月 1 号凌晨 2:00
"0 2 1 * *"
```

**在线工具**: https://crontab.guru


---

### Q15: 如何监听 GitHub PR 事件?

**A:** Webhook 配置:

**1. GitHub 配置**

```Text
Settings → Webhooks → Add webhook
- Payload URL: https://your-domain/webhook/github
- Content type: application/json
- Events: Pull requests
```

**2. OpenClaw 处理**

```Typescript
webhookServer.register('/webhook/github', async (event) => {
  // 验证签名
  if (!verifyGitHubSignature(event)) {
    throw new Error('Invalid signature');
  }
  
  // 处理 PR 事件
  if (event.action === 'opened' && event.pull_request) {
    await agent.execute({
      message: `新 PR: ${event.pull_request.title}`,
      context: {
        repo: event.pull_request.base.repo.full_name,
        prNumber: event.pull_request.number
      }
    });
  }
});
```

**3. 签名验证**

```Typescript
function verifyGitHubSignature(event: any): boolean {
  const signature = event.headers;
  const secret = process.env.GITHUB_WEBHOOK_SECRET;
  
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(JSON.stringify(event.body))
    .digest('hex');
  
  return signature === `sha256=${expectedSignature}`;
}
```


---

## 六、性能优化类

### Q16: 如何减少 AI 调用成本?

**A:** 四种策略:

**1. Prompt Caching**

```Typescript
// 缓存系统消息和 Skill 定义
const cachedMessages = ;

// 减少 50% 输入 token 成本
```

**2. 模型分级**

```Typescript
function selectModel(complexity: string) {
  return {
    low: 'claude-haiku',     // $0.25/1M tokens
    medium: 'claude-sonnet',  // $3/1M tokens
    high: 'claude-opus'       // $15/1M tokens
  };
}
```

**3. 结果缓存**

```Typescript
const cache = new Map();

async function cachedCall(prompt: string, ttl = 3600) {
  const key = hash(prompt);
  
  if (cache.has(key)) {
    return cache.get(key);
  }
  
  const result = await model.chat(prompt);
  cache.set(key, result);
  
  setTimeout(() => cache.delete(key), ttl * 1000);
  
  return result;
}
```

**4. 批量请求**

```Typescript
// 合并多个小请求
const results = await model.batch();
```


---

### Q17: 如何提升响应速度?

**A:** 三个方向:

**1. 异步处理**

```Typescript
// 立即响应,后台执行
await channel.sendMessage(userId, '⏳ 处理中...');

// 后台执行
processInBackground(async () => {
  const result = await longRunningTask();
  await channel.sendMessage(userId, `✅ ${result}`);
});
```

**2. 并发工具调用**

```Typescript
// 并行执行多个工具
const results = await Promise.all();
```

**3. 本地模型**

```Typescript
// 简单任务用本地模型
if (isSimpleTask(message)) {
  return await ollamaModel.chat(message); // 无网络延迟
}
```


---

## 七、调试与监控类

### Q18: 如何调试 Skill 执行?

**A:** 多层日志:

```Typescript
export const debugSkill: Skill = {
  name: 'debug_skill',
  async execute(params, context) {
    console.log('=== Skill 开始执行 ===');
    console.log('参数:', JSON.stringify(params, null, 2));
    console.log('上下文:', JSON.stringify(context, null, 2));
    
    try {
      const result = await doWork(params);
      console.log('执行成功:', result);
      return result;
    } catch (error) {
      console.error('执行失败:', error);
      throw error;
    } finally {
      console.log('=== Skill 执行结束 ===');
    }
  }
};
```

**日志级别**:

```Bash
# 开启 DEBUG 日志
export LOG_LEVEL=debug
openclaw gateway start

# 查看日志
tail -f ~/.openclaw/logs/debug.log
```


---

### Q19: 如何监控系统运行状态?

**A:** 监控面板:

```Typescript
// 性能指标
class PerformanceMonitor {
  private metrics = {
    totalRequests: 0,
    successfulRequests: 0,
    failedRequests: 0,
    avgResponseTime: 0,
    activeConnections: 0
  };
  
  recordRequest(duration: number, success: boolean) {
    this.metrics.totalRequests++;
    
    if (success) {
      this.metrics.successfulRequests++;
    } else {
      this.metrics.failedRequests++;
    }
    
    // 更新平均响应时间
    this.metrics.avgResponseTime = 
      (this.metrics.avgResponseTime * (this.metrics.totalRequests - 1) + duration) 
      / this.metrics.totalRequests;
  }
  
  getMetrics() {
    return {
      ...this.metrics,
      successRate: this.metrics.successfulRequests / this.metrics.totalRequests
    };
  }
}

// 暴露监控接口
fastify.get('/metrics', async () => {
  return performanceMonitor.getMetrics();
});
```

**Dashboard 展示**:

```Typescript
// Web UI 展示实时指标
function MetricsDashboard() {
  const  = useState(null);
  
  useEffect(() => {
    const interval = setInterval(async () => {
      const data = await fetch('/metrics').then(r => r.json());
      setMetrics(data);
    }, 1000);
    
    return () => clearInterval(interval);
  }, []);
  
  return (
    <div>
      <h2>系统监控</h2>
      <p>总请求数: {metrics?.totalRequests}</p>
      <p>成功率: {(metrics?.successRate * 100).toFixed(2)}%</p>
      <p>平均响应时间: {metrics?.avgResponseTime.toFixed(0)}ms</p>
    </div>
  );
}
```


---

## 八、常见问题类

### Q20: OpenClaw 是否支持中文?

**A:** 完全支持。


- 使用 Claude/GPT-4 等多语言模型
- 中文理解能力强
- 文档和配置均支持中文


---

### Q21: 是否可以完全离线运行?

**A:** 可以,使用本地模型:

```Json
{
  "models": {
    "default": "ollama/llama3",
    "fallback": []
  }
}
```

**限制**:


- 本地模型能力弱于云端
- 需要 GPU 加速
- 模型体积大 (数 GB)


---

### Q22: 如何迁移现有的 ChatGPT 对话?

**A:** 导入工具:

```Typescript
const importChatGPT: Skill = {
  name: 'import_chatgpt',
  async execute({ conversationJson }) {
    const messages = parseConversation(conversationJson);
    
    // 创建新会话
    const session = await Session.create('imported');
    
    // 导入消息历史
    for (const msg of messages) {
      session.addMessage({
        role: msg.role,
        content: msg.content
      });
    }
    
    await session.save();
    
    return `✅ 已导入 ${messages.length} 条消息`;
  }
};
```


---

## 九、学习资源

### 官方文档


- 架构: https://docs.openclaw.ai/concepts/architecture
- Agent Loop: https://docs.openclaw.ai/concepts/agent-loop
- Skills: https://docs.openclaw.ai/cli/skills

### 社区资源


- GitHub: https://github.com/openclaw/openclaw
- Discord: https://discord.gg/clawd
- DeepWiki: https://deepwiki.com/openclaw/openclaw

### 团队文档


- 架构解析: https://km.sankuai.com/page/2748203784
- 核心原理: https://km.sankuai.com/page/2747794151
- 扩展开发: https://km.sankuai.com/page/2748480295


---

**文档版本**: v1.0   **更新时间**: 2026-02-12   **维护团队**: OpenClaw 研究小组   **反馈渠道**: 欢迎在飞书群或学城评论区补充问题


