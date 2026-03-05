# 4.2 自动化工作流

## 定时任务调度

### Cron 表达式配置

```yaml
# workflows/scheduled-tasks.yaml
workflows:
  daily-report:
    schedule: "0 9 * * *"  # 每天上午9点
    steps:
      - name: collect-data
        skill: database
        action: query
        params:
          sql: "SELECT * FROM metrics WHERE date = CURRENT_DATE - 1"
      
      - name: generate-report
        skill: code
        action: analyze
        params:
          template: daily-report-template
      
      - name: send-email
        skill: email-sender
        params:
          to: "manager@company.com"
          subject: "每日数据报告"
    
  weekly-cleanup:
    schedule: "0 2 * * 0"  # 每周日凌晨2点
    steps:
      - name: cleanup-logs
        skill: file
        action: delete
        params:
          pattern: "logs/*.log"
          olderThan: "30d"
      
      - name: backup-database
        skill: database
        action: backup
        params:
          destination: "s3://backups/weekly/"
```

### 动态任务调度器

```javascript
// integration/task-scheduler.js
const cron = require('node-cron');

class TaskScheduler {
  constructor() {
    this.tasks = new Map();
    this.running = false;
  }
  
  register(name, schedule, workflow) {
    if (!cron.validate(schedule)) {
      throw new Error(`无效的cron表达式: ${schedule}`);
    }
    
    const task = cron.schedule(schedule, async () => {
      console.log(`[Scheduler] 执行任务: ${name}`);
      try {
        await this._executeWorkflow(workflow);
      } catch (error) {
        console.error(`[Scheduler] 任务 ${name} 失败:`, error);
        await this._handleFailure(name, error);
      }
    }, {
      scheduled: false // 先不启动
    });
    
    this.tasks.set(name, {
      task,
      schedule,
      workflow,
      lastRun: null,
      nextRun: null
    });
    
    return this;
  }
  
  start() {
    this.running = true;
    for (const [name, { task }] of this.tasks) {
      task.start();
      console.log(`[Scheduler] 已启动: ${name}`);
    }
  }
  
  stop() {
    this.running = false;
    for (const [name, { task }] of this.tasks) {
      task.stop();
      console.log(`[Scheduler] 已停止: ${name}`);
    }
  }
  
  async _executeWorkflow(workflow) {
    const orchestrator = new Orchestrator();
    
    for (const step of workflow.steps) {
      console.log(`[Workflow] 执行步骤: ${step.name}`);
      
      const result = await orchestrator.dispatch({
        type: step.skill,
        action: step.action,
        ...step.params
      });
      
      if (!result.success) {
        throw new Error(`步骤 ${step.name} 失败: ${result.error}`);
      }
    }
  }
  
  async _handleFailure(taskName, error) {
    // 发送告警通知
    const alertSkill = new SlackSkill(config.slack);
    await alertSkill.sendMessage(`
      ⚠️ 定时任务失败\n任务: ${taskName}\n错误: ${error.message}\n时间: ${new Date().toISOString()}
    `);
  }
  
  getStatus() {
    const status = [];
    for (const [name, info] of this.tasks) {
      status.push({
        name,
        schedule: info.schedule,
        running: this.running,
        lastRun: info.lastRun,
        nextRun: info.task.nextDates().toDate()
      });
    }
    return status;
  }
}

module.exports = TaskScheduler;
```

---

## 事件驱动架构

### 事件总线实现

```javascript
// integration/event-bus.js
class EventBus {
  constructor() {
    this.subscribers = new Map();
    this.middleware = [];
  }
  
  use(middleware) {
    this.middleware.push(middleware);
  }
  
  subscribe(eventType, handler, options = {}) {
    if (!this.subscribers.has(eventType)) {
      this.subscribers.set(eventType, []);
    }
    
    const subscription = {
      handler,
      filter: options.filter,
      priority: options.priority || 0
    };
    
    this.subscribers.get(eventType).push(subscription);
    
    // 按优先级排序
    this.subscribers.get(eventType).sort((a, b) => b.priority - a.priority);
    
    // 返回取消订阅函数
    return () => {
      const subs = this.subscribers.get(eventType);
      const index = subs.indexOf(subscription);
      if (index > -1) subs.splice(index, 1);
    };
  }
  
  async publish(event) {
    // 运行中间件
    let processedEvent = event;
    for (const middleware of this.middleware) {
      processedEvent = await middleware(processedEvent);
      if (!processedEvent) return; // 中间件可以拦截事件
    }
    
    const { type } = processedEvent;
    const subscribers = this.subscribers.get(type) || [];
    
    // 并行执行所有订阅者
    const promises = subscribers.map(async ({ handler, filter }) => {
      // 检查过滤器
      if (filter && !filter(processedEvent)) return;
      
      try {
        await handler(processedEvent);
      } catch (error) {
        console.error(`[EventBus] 处理器错误 (${type}):`, error);
      }
    });
    
    await Promise.all(promises);
  }
  
  // 一次性订阅
  once(eventType, handler) {
    const unsubscribe = this.subscribe(eventType, (event) => {
      unsubscribe();
      handler(event);
    });
    return unsubscribe;
  }
}

// 使用示例
const eventBus = new EventBus();

// 添加日志中间件
eventBus.use(async (event) => {
  console.log(`[Event] ${event.type}:`, event.payload);
  return event;
});

// 订阅新订单事件
eventBus.subscribe('order.created', async (event) => {
  const { orderId, amount } = event.payload;
  
  // 触发库存检查
  await inventoryAgent.checkStock(orderId);
  
  // 触发支付处理
  if (amount > 0) {
    await paymentAgent.processPayment(orderId);
  }
}, { priority: 10 });

// 发布事件
eventBus.publish({
  type: 'order.created',
  payload: { orderId: '12345', amount: 99.99 },
  timestamp: Date.now()
});

module.exports = EventBus;
```

---

## 实战：完整自动化流程

### 智能客服系统

```javascript
// workflows/customer-service.js
class CustomerServiceWorkflow {
  constructor() {
    this.eventBus = new EventBus();
    this.setupEventHandlers();
  }
  
  setupEventHandlers() {
    // 新消息到达
    this.eventBus.subscribe('message.received', async (event) => {
      const { customerId, message, channel } = event.payload;
      
      // 1. 意图识别
      const intent = await this.intentAgent.classify(message);
      
      // 2. 路由到对应处理流程
      switch (intent.category) {
        case 'inquiry':
          await this.handleInquiry(customerId, message);
          break;
        case 'complaint':
          await this.handleComplaint(customerId, message);
          break;
        case 'order':
          await this.handleOrderQuery(customerId, message);
          break;
        default:
          await this.escalateToHuman(customerId, message);
      }
    });
    
    // 工单状态变更
    this.eventBus.subscribe('ticket.updated', async (event) => {
      const { ticketId, status, assignee } = event.payload;
      
      if (status === 'resolved') {
        // 发送满意度调查
        await this.surveyAgent.sendSurvey(ticketId);
      }
    });
  }
  
  async handleInquiry(customerId, message) {
    // 查询知识库
    const answer = await this.knowledgeAgent.search(message);
    
    // 生成回复
    const response = await this.replyAgent.generate({
      context: answer,
      tone: 'friendly',
      language: 'zh'
    });
    
    // 发送回复
    await this.sendMessage(customerId, response);
  }
  
  async handleComplaint(customerId, message) {
    // 创建高优先级工单
    const ticket = await this.ticketAgent.create({
      customerId,
      type: 'complaint',
      priority: 'high',
      content: message
    });
    
    // 立即通知人工客服
    await this.notificationAgent.alert({
      channel: 'slack',
      message: `🚨 紧急投诉工单: ${ticket.id}`,
      mention: '@support-leads'
    });
    
    // 安抚客户
    await this.sendMessage(customerId, 
      '非常抱歉给您带来不好的体验，我已为您创建优先处理工单，专员将尽快联系您。'
    );
  }
  
  async sendMessage(customerId, content) {
    await this.eventBus.publish({
      type: 'message.send',
      payload: { customerId, content }
    });
  }
}

module.exports = CustomerServiceWorkflow;
```

---

*模块4完成*
