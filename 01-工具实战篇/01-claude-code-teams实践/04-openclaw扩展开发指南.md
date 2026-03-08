# OpenClaw 扩展开发指南


<!-- TOC -->

# OpenClaw 扩展开发指南

## 一、自定义 Channel 开发

### 1.1 Channel 接口定义

```Typescript
interface Channel {
  // 渠道元信息
  readonly id: string;
  readonly name: string;
  
  // 生命周期方法
  connect(): Promise<void>;
  disconnect(): Promise<void>;
  
  // 消息处理
  onMessage(handler: MessageHandler): void;
  sendMessage(userId: string, content: string, options?: SendOptions): Promise<void>;
  
  // 可选方法
  onTyping?(userId: string): void;
  onPresence?(userId: string, status: 'online' | 'offline'): void;
}
```

### 1.2 飞书 Channel 实现示例

```Typescript
import * as lark from '@larksuiteoapi/node-sdk';

export class FeishuChannel implements Channel {
  readonly id = 'feishu';
  readonly name = 'Feishu (飞书)';
  
  private client: lark.Client;
  private messageHandler?: MessageHandler;
  
  constructor(private config: FeishuConfig) {}
  
  async connect(): Promise<void> {
    // 1. 初始化飞书客户端
    this.client = new lark.Client({
      appId: this.config.appId,
      appSecret: this.config.appSecret,
      appType: lark.AppType.SelfBuild
    });
    
    // 2. 启动事件监听
    const eventDispatcher = new lark.EventDispatcher({
      encryptKey: this.config.encryptKey,
      verificationToken: this.config.verificationToken
    });
    
    // 3. 注册消息事件
    eventDispatcher.register({
      'im.message.receive_v1': this.handleIncomingMessage.bind(this)
    });
    
    // 4. 启动 HTTP 服务器
    fastify.post('/webhook/feishu', async (req, reply) => {
      await eventDispatcher.invoke(req.body);
      reply.send({ success: true });
    });
    
    console.log('✅ 飞书 Channel 已连接');
  }
  
  onMessage(handler: MessageHandler): void {
    this.messageHandler = handler;
  }
  
  private async handleIncomingMessage(event: any): Promise<void> {
    const { message, sender } = event;
    
    // 转换为统一格式
    const normalized: NormalizedMessage = {
      id: message.message_id,
      channelId: 'feishu',
      userId: sender.sender_id.user_id,
      content: JSON.parse(message.content).text,
      timestamp: parseInt(message.create_time),
      metadata: {
        chat_id: message.chat_id,
        message_type: message.message_type
      }
    };
    
    // 调用处理器
    if (this.messageHandler) {
      await this.messageHandler(normalized);
    }
  }
  
  async sendMessage(
    userId: string,
    content: string,
    options?: SendOptions
  ): Promise<void> {
    await this.client.im.message.create({
      data: {
        receive_id: userId,
        msg_type: 'text',
        content: JSON.stringify({ text: content })
      },
      params: {
        receive_id_type: 'user_id'
      }
    });
  }
  
  async disconnect(): Promise<void> {
    console.log('飞书 Channel 已断开');
  }
}
```

### 1.3 注册 Channel

```Typescript
// 在配置文件中启用
{
  "channels": {
    "feishu": {
      "enabled": true,
      "appId": "cli_xxx",
      "appSecret": "xxx",
      "encryptKey": "xxx",
      "verificationToken": "xxx"
    }
  }
}

// 在代码中注册
const channelManager = new ChannelManager();
channelManager.register(new FeishuChannel(config.feishu));
```


---

## 二、自定义 Skill 开发

### 2.1 Skill 接口定义

```Typescript
interface Skill {
  // 元信息
  name: string;
  description: string;
  version?: string;
  
  // 参数定义
  parameters: ParameterSchema;
  
  // 执行函数
  execute(params: any, context: ExecutionContext): Promise<any>;
  
  // 可选生命周期
  init?(): Promise<void>;
  cleanup?(): Promise<void>;
  
  // 权限与配置
  requiresApproval?: boolean;
  rateLimit?: {
    maxCalls: number;
    windowSeconds: number;
  };
}
```

### 2.2 示例 1: 天气查询 Skill

```Typescript
export const weatherSkill: Skill = {
  name: 'get_weather',
  description: '查询指定城市的天气信息',
  version: '1.0.0',
  
  parameters: {
    city: {
      type: 'string',
      required: true,
      description: '城市名称 (如: 北京, 上海)'
    },
    unit: {
      type: 'string',
      required: false,
      default: 'celsius',
      enum: ,
      description: '温度单位'
    }
  },
  
  async execute(params) {
    const { city, unit } = params;
    
    // 调用天气 API
    const response = await fetch(
      `https://api.weather.com/v1/weather?city=${city}&unit=${unit}`,
      {
        headers: {
          'Authorization': `Bearer ${process.env.WEATHER_API_KEY}`
        }
      }
    );
    
    const data = await response.json();
    
    // 格式化返回结果
    return `${city}天气: ${data.description}, 温度 ${data.temperature}°${unit === 'celsius' ? 'C' : 'F'}`;
  }
};
```

### 2.3 示例 2: 数据库查询 Skill

```Typescript
import { PrismaClient } from '@prisma/client';

export const dbQuerySkill: Skill = {
  name: 'query_database',
  description: '查询数据库 (仅支持 SELECT 语句)',
  requiresApproval: true, // 需要用户授权
  
  parameters: {
    query: {
      type: 'string',
      required: true,
      description: 'SQL 查询语句'
    }
  },
  
  private prisma: PrismaClient,
  
  async init() {
    this.prisma = new PrismaClient();
    console.log('✅ 数据库连接已建立');
  },
  
  async execute(params, context) {
    const { query } = params;
    
    // 安全检查: 只允许 SELECT
    if (!query.trim().toLowerCase().startsWith('select')) {
      throw new Error('仅支持 SELECT 查询');
    }
    
    // 审计日志
    console.log(` User: ${context.userId}, Query: ${query}`);
    
    // 执行查询
    const result = await this.prisma.$queryRawUnsafe(query);
    
    // 格式化结果
    return JSON.stringify(result, null, 2);
  },
  
  async cleanup() {
    await this.prisma.$disconnect();
    console.log('数据库连接已关闭');
  }
};
```

### 2.4 示例 3: 文件上传 Skill

```Typescript
export const fileUploadSkill: Skill = {
  name: 'upload_file',
  description: '上传文件到云存储',
  
  parameters: {
    filePath: {
      type: 'string',
      required: true,
      description: '本地文件路径'
    },
    bucket: {
      type: 'string',
      required: false,
      default: 'default',
      description: '存储桶名称'
    }
  },
  
  rateLimit: {
    maxCalls: 10,
    windowSeconds: 60 // 每分钟最多 10 次
  },
  
  async execute(params) {
    const { filePath, bucket } = params;
    
    // 1. 验证文件存在
    if (!fs.existsSync(filePath)) {
      throw new Error(`文件不存在: ${filePath}`);
    }
    
    // 2. 读取文件
    const fileBuffer = await fs.readFile(filePath);
    const fileName = path.basename(filePath);
    
    // 3. 上传到 S3
    const s3 = new AWS.S3();
    const result = await s3.upload({
      Bucket: bucket,
      Key: fileName,
      Body: fileBuffer
    }).promise();
    
    return `✅ 文件已上传: ${result.Location}`;
  }
};
```


---

## 三、Cron 任务开发

### 3.1 配置文件格式

```Yaml
# cron.yml
jobs:
  # 每日工作总结
  - name: daily_summary
    schedule: "0 18 * * 1-5"  # 工作日 18:00
    action:
      type: agent
      message: |
        基于今日 Git 提交和飞书日历生成工作总结:
        1. 完成的任务
        2. 参加的会议
        3. 明日计划
      channel: feishu
      userId: "@me"
  
  # 周报生成
  - name: weekly_report
    schedule: "0 17 * * 5"  # 周五 17:00
    action:
      type: agent
      message: "生成本周工作周报"
      channel: feishu
      userId: "@me"
  
  # 定时备份
  - name: backup_database
    schedule: "0 2 * * *"  # 每天 02:00
    action:
      type: webhook
      url: "https://backup-service.internal/trigger"
      method: POST
      body:
        database: "production"
```

### 3.2 动态添加 Cron 任务

```Typescript
class CronManager {
  async addJob(config: CronJobConfig): Promise<void> {
    // 验证 cron 表达式
    if (!this.validateCronExpression(config.schedule)) {
      throw new Error('无效的 cron 表达式');
    }
    
    // 创建任务
    const job = new CronJob(config.schedule, async () => {
      await this.executeJob(config);
    });
    
    // 保存到数据库
    await db.cronJobs.create({
      data: {
        name: config.name,
        schedule: config.schedule,
        action: JSON.stringify(config.action),
        enabled: true
      }
    });
    
    // 启动任务
    job.start();
    
    console.log(`✅ Cron 任务已添加: ${config.name}`);
  }
  
  private async executeJob(config: CronJobConfig): Promise<void> {
    try {
      if (config.action.type === 'agent') {
        await this.executeAgentAction(config.action);
      } else if (config.action.type === 'webhook') {
        await this.executeWebhookAction(config.action);
      }
    } catch (error) {
      console.error(`❌ 任务执行失败: ${config.name}`, error);
      
      // 发送告警
      await this.sendAlert(config.name, error);
    }
  }
}
```


---

## 四、Webhook 集成

### 4.1 GitHub PR 监听

```Typescript
// 注册 GitHub Webhook
webhookServer.register('/webhook/github', async (event) => {
  if (event.action === 'opened' && event.pull_request) {
    const pr = event.pull_request;
    
    // 触发代码审查 Agent
    await agent.execute({
      message: `请审查 PR: ${pr.title}\n${pr.html_url}`,
      context: {
        repo: pr.base.repo.full_name,
        prNumber: pr.number
      }
    });
  }
});
```

### 4.2 Gmail 新邮件监听

```Typescript
// 使用 Gmail PubSub
const { google } = require('googleapis');

async function setupGmailWatch() {
  const gmail = google.gmail('v1');
  
  // 订阅邮箱变更
  await gmail.users.watch({
    userId: 'me',
    requestBody: {
      topicName: 'projects/my-project/topics/gmail-notifications'
    }
  });
}

// 处理 PubSub 消息
pubsub.subscription('gmail-sub').on('message', async (message) => {
  const { emailAddress, historyId } = JSON.parse(message.data);
  
  // 获取新邮件
  const emails = await fetchNewEmails(historyId);
  
  for (const email of emails) {
    // 触发 Agent 处理
    await agent.execute({
      message: `新邮件: ${email.subject}\n${email.snippet}`,
      context: { emailId: email.id }
    });
  }
  
  message.ack();
});
```


---

## 五、UI 扩展开发

### 5.1 自定义 Dashboard Widget

```Typescript
// Widget 组件
export function CustomWidget() {
  const  = useState(null);
  
  useEffect(() => {
    // 调用 OpenClaw API
    fetch('/api/custom-data')
      .then(res => res.json())
      .then(setData);
  }, []);
  
  return (
    <Card>
      <CardHeader>
        <CardTitle>自定义数据</CardTitle>
      </CardHeader>
      <CardContent>
        {data && (
          <div>
            <p>统计: {data.count}</p>
            <Chart data={data.series} />
          </div>
        )}
      </CardContent>
    </Card>
  );
}

// 注册 Widget
dashboardRegistry.register({
  id: 'custom-widget',
  component: CustomWidget,
  defaultPosition: { x: 0, y: 0, w: 6, h: 4 }
});
```

### 5.2 自定义设置页面

```Typescript
export function CustomSettingsPage() {
  const  = useState({});
  
  const handleSave = async () => {
    await fetch('/api/config', {
      method: 'PUT',
      body: JSON.stringify(config)
    });
  };
  
  return (
    <div>
      <h2>自定义配置</h2>
      <Form>
        <Input
          label="API Key"
          value={config.apiKey}
          onChange={(e) => setConfig({ ...config, apiKey: e.target.value })}
        />
        <Button onClick={handleSave}>保存</Button>
      </Form>
    </div>
  );
}

// 注册设置页面
settingsRegistry.register({
  id: 'custom-settings',
  title: '自定义设置',
  component: CustomSettingsPage
});
```


---

## 六、测试与调试

### 6.1 Skill 单元测试

```Typescript
import { describe, it, expect } from 'vitest';

describe('weatherSkill', () => {
  it('应该返回天气信息', async () => {
    const result = await weatherSkill.execute({
      city: '北京',
      unit: 'celsius'
    }, {
      userId: 'test-user'
    });
    
    expect(result).toContain('北京天气');
    expect(result).toContain('°C');
  });
  
  it('应该处理无效城市', async () => {
    await expect(
      weatherSkill.execute({ city: 'InvalidCity' }, {})
    ).rejects.toThrow('城市不存在');
  });
});
```

### 6.2 Channel 集成测试

```Typescript
describe('FeishuChannel', () => {
  let channel: FeishuChannel;
  
  beforeEach(() => {
    channel = new FeishuChannel(mockConfig);
  });
  
  it('应该正确连接', async () => {
    await channel.connect();
    expect(channel.isConnected()).toBe(true);
  });
  
  it('应该发送消息', async () => {
    const spy = vi.spyOn(channel.client.im.message, 'create');
    
    await channel.sendMessage('user-123', 'Hello');
    
    expect(spy).toHaveBeenCalledWith({
      data: {
        receive_id: 'user-123',
        msg_type: 'text',
        content: JSON.stringify({ text: 'Hello' })
      }
    });
  });
});
```

### 6.3 调试日志

```Typescript
// 开启详细日志
process.env.LOG_LEVEL = 'debug';

// 在 Skill 中输出调试信息
export const debugSkill: Skill = {
  name: 'debug_skill',
  async execute(params, context) {
    console.log('=== Skill 执行 ===');
    console.log('参数:', JSON.stringify(params, null, 2));
    console.log('上下文:', JSON.stringify(context, null, 2));
    
    const result = await doSomething(params);
    
    console.log('结果:', result);
    console.log('===================');
    
    return result;
  }
};
```


---

## 七、部署与发布

### 7.1 打包 Skill 插件

```Json
// package.json
{
  "name": "openclaw-skill-weather",
  "version": "1.0.0",
  "main": "dist/index.js",
  "openclaw": {
    "type": "skill",
    "skills": 
  },
  "scripts": {
    "build": "tsc",
    "test": "vitest"
  }
}
```

```Bash
# 构建
npm run build

# 发布到 npm
npm publish
```

### 7.2 安装插件

```Bash
# 安装 Skill 插件
openclaw skills install openclaw-skill-weather

# 启用 Skill
openclaw skills enable weather

# 列出已安装的 Skills
openclaw skills list
```

### 7.3 本地开发模式

```Bash
# 链接本地插件
cd ~/my-skill
npm link

openclaw skills link ~/my-skill

# 热重载
openclaw dev --watch
```


---

## 八、最佳实践

### 8.1 错误处理

```Typescript
export const robustSkill: Skill = {
  name: 'robust_skill',
  async execute(params) {
    try {
      return await doWork(params);
    } catch (error) {
      // 记录详细错误
      console.error('Skill 执行失败:', {
        skill: 'robust_skill',
        params,
        error: error.message,
        stack: error.stack
      });
      
      // 返回友好错误消息
      throw new Error(`执行失败: ${error.message}`);
    }
  }
};
```

### 8.2 参数验证

```Typescript
import { z } from 'zod';

const paramsSchema = z.object({
  city: z.string().min(1, '城市名称不能为空'),
  unit: z.enum().default('celsius')
});

export const validatedSkill: Skill = {
  name: 'validated_skill',
  async execute(params) {
    // 使用 Zod 验证
    const validated = paramsSchema.parse(params);
    
    return await processWeather(validated);
  }
};
```

### 8.3 性能优化

```Typescript
// 使用缓存避免重复请求
const cache = new Map();

export const cachedSkill: Skill = {
  name: 'cached_skill',
  async execute(params) {
    const cacheKey = JSON.stringify(params);
    
    // 检查缓存
    if (cache.has(cacheKey)) {
      const cached = cache.get(cacheKey);
      if (Date.now() - cached.timestamp < 60000) { // 1 分钟
        return cached.result;
      }
    }
    
    // 执行实际请求
    const result = await fetchData(params);
    
    // 更新缓存
    cache.set(cacheKey, {
      result,
      timestamp: Date.now()
    });
    
    return result;
  }
};
```


---

## 九、示例项目

### 9.1 货架值班助手

**需求**: 监听告警,自动分析并推送到飞书

**实现**:

```Typescript
// 1. 创建告警监听 Skill
export const alertSkill: Skill = {
  name: 'monitor_alerts',
  async execute() {
    const alerts = await fetchAlerts();
    
    for (const alert of alerts) {
      // 使用 AI 分析告警
      const analysis = await agent.execute({
        message: `分析告警: ${JSON.stringify(alert)}`
      });
      
      // 推送到飞书
      await feishuChannel.sendMessage(
        '@oncall',
        `🚨 告警通知\n${analysis.content}`
      );
    }
  }
};

// 2. 配置定时任务
// cron.yml
jobs:
  - name: check_alerts
    schedule: "*/5 * * * *"  # 每 5 分钟
    action:
      type: skill
      skill: monitor_alerts
```

### 9.2 团队知识库

**需求**: 将学城文档接入 OpenClaw,实现智能问答

**实现**:

```Typescript
// 1. 索引文档 Skill
export const indexDocsSkill: Skill = {
  name: 'index_docs',
  async execute({ urls }) {
    for (const url of urls) {
      const content = await fetchKMDoc(url);
      const chunks = splitIntoChunks(content);
      const embeddings = await model.embed(chunks);
      
      await vectorDB.upsert({ url, embeddings });
    }
  }
};

// 2. 知识检索 Skill
export const searchKnowledgeSkill: Skill = {
  name: 'search_knowledge',
  async execute({ query }) {
    const embedding = await model.embed(query);
    const results = await vectorDB.search(embedding, { topK: 3 });
    
    return results.map(r => ({
      title: r.metadata.title,
      url: r.metadata.url,
      snippet: r.content.slice(0, 200)
    }));
  }
};
```


---

## 十、参考资源

### 官方文档


- Skills 开发: https://docs.openclaw.ai/cli/skills
- Channels 开发: https://docs.openclaw.ai/concepts/channels
- Cron 任务: https://docs.openclaw.ai/automation/cron-jobs

### 示例代码


- GitHub: https://github.com/openclaw/openclaw/tree/main/packages/skills
- 社区插件: https://github.com/openclaw/plugins


---

**文档版本**: v1.0   **更新时间**: 2026-02-12   **维护团队**: OpenClaw 研究小组


