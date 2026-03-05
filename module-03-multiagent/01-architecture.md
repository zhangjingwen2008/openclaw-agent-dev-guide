# 模块3: 多Agent协作实战

## 3.1 多Agent架构设计

### 为什么需要多Agent？

单Agent的局限：
- 上下文窗口有限
- 难以并行处理多个任务
- 专业领域深度不足

多Agent的优势：
- **专业化**: 每个Agent专注特定领域
- **并行化**: 同时处理多个子任务
- **容错性**: 单个失败不影响整体
- **可扩展**: 动态添加新Agent

### 典型架构模式

```
┌─────────────────────────────────────────┐
│           Orchestrator                  │
│         (协调者/调度器)                  │
└──────────────┬──────────────────────────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌───────┐  ┌───────┐  ┌───────┐
│Agent A│  │Agent B│  │Agent C│
│研究   │  │写作   │  │审核   │
└───┬───┘  └───┬───┘  └───┬───┘
    │          │          │
    └──────────┼──────────┘
               ▼
        ┌─────────────┐
        │ Shared Store│
        │  (共享记忆)  │
        └─────────────┘
```

---

## 3.2 实现协调者模式

### 基础协调器

```javascript
// multi-agent/orchestrator.js
class Orchestrator {
  constructor() {
    this.agents = new Map();
    this.sharedStore = new Map();
    this.eventBus = new EventEmitter();
  }
  
  // 注册Agent
  register(name, agent, config = {}) {
    this.agents.set(name, {
      instance: agent,
      config: {
        role: config.role || 'worker',
        capabilities: config.capabilities || [],
        priority: config.priority || 1
      }
    });
    
    // 订阅Agent事件
    agent.on('complete', (result) => {
      this._handleAgentComplete(name, result);
    });
    
    agent.on('error', (error) => {
      this._handleAgentError(name, error);
    });
  }
  
  // 分发任务
  async dispatch(task) {
    console.log(`[Orchestrator] 分发任务: ${task.type}`);
    
    // 根据任务类型选择合适的Agent
    const suitableAgents = this._findSuitableAgents(task);
    
    if (suitableAgents.length === 0) {
      throw new Error(`没有Agent能处理任务类型: ${task.type}`);
    }
    
    // 选择优先级最高的
    const selected = suitableAgents.sort((a, b) => 
      b.config.priority - a.config.priority
    )[0];
    
    // 执行任务
    const result = await selected.instance.execute(task);
    
    // 保存到共享存储
    this.sharedStore.set(`task:${task.id}`, {
      agent: selected.name,
      result,
      timestamp: Date.now()
    });
    
    return result;
  }
  
  // 并行执行多个任务
  async dispatchParallel(tasks) {
    console.log(`[Orchestrator] 并行分发 ${tasks.length} 个任务`);
    
    const promises = tasks.map(task => this.dispatch(task));
    const results = await Promise.allSettled(promises);
    
    return results.map((result, index) => ({
      taskId: tasks[index].id,
      status: result.status,
      data: result.status === 'fulfilled' ? result.value : result.reason
    }));
  }
  
  // 工作流编排
  async executeWorkflow(workflow) {
    const results = {};
    
    for (const step of workflow.steps) {
      console.log(`[Orchestrator] 执行步骤: ${step.name}`);
      
      // 解析依赖
      const dependencies = step.dependsOn?.map(depId => results[depId]) || [];
      
      // 构建任务
      const task = {
        id: step.id,
        type: step.type,
        input: step.input,
        context: dependencies
      };
      
      // 执行
      const result = await this.dispatch(task);
      results[step.id] = result;
      
      // 检查是否需要人工介入
      if (step.requiresApproval && !await this._requestApproval(step, result)) {
        throw new Error(`步骤 ${step.name} 未通过审批`);
      }
    }
    
    return results;
  }
  
  _findSuitableAgents(task) {
    const suitable = [];
    
    for (const [name, { config }] of this.agents) {
      if (config.capabilities.includes(task.type)) {
        suitable.push({ name, config });
      }
    }
    
    return suitable;
  }
  
  _handleAgentComplete(agentName, result) {
    this.eventBus.emit('agent:complete', { agent: agentName, result });
  }
  
  _handleAgentError(agentName, error) {
    console.error(`[Orchestrator] Agent ${agentName} 错误:`, error);
    this.eventBus.emit('agent:error', { agent: agentName, error });
  }
  
  async _requestApproval(step, result) {
    // 实际实现中这里可以发送通知等待人工确认
    console.log(`[Orchestrator] 等待审批: ${step.name}`);
    return true; // 简化示例
  }
}

module.exports = Orchestrator;
```

---

## 3.3 实战：内容创作团队

### 场景描述
创建一个自动化内容创作团队：
- **研究员**: 收集资料、整理信息
- **写手**: 撰写文章
- **编辑**: 润色修改
- **审核员**: 质量把关

### 代码实现

```javascript
// multi-agent/content-team.js
const { Agent } = require('openclaw');
const Orchestrator = require('./orchestrator');

// 创建各角色Agent
const researcher = new Agent({
  name: 'researcher',
  systemPrompt: `你是专业的研究员。你的任务是：
1. 深入搜索和收集主题相关信息
2. 整理关键数据和观点
3. 提供结构化的研究报告
输出格式：Markdown，包含引用来源`
});

const writer = new Agent({
  name: 'writer',
  systemPrompt: `你是资深撰稿人。你的任务是：
1. 基于研究资料撰写高质量文章
2. 确保逻辑清晰、语言流畅
3. 符合目标受众的阅读习惯
输出格式：完整的文章，含标题和小标题`
});

const editor = new Agent({
  name: 'editor',
  systemPrompt: `你是严格的编辑。你的任务是：
1. 检查文章的语法和错别字
2. 优化句子结构和表达方式
3. 确保风格统一、用词准确
输出：修改后的文章 + 修改说明`
});

const reviewer = new Agent({
  name: 'reviewer',
  systemPrompt: `你是质量审核员。你的任务是：
1. 评估文章的整体质量
2. 检查是否符合发布标准
3. 给出通过/不通过的结论和改进建议
输出：审核报告（通过/不通过 + 理由）`
});

// 初始化协调器
const orchestrator = new Orchestrator();

orchestrator.register('researcher', researcher, {
  role: 'specialist',
  capabilities: ['research', 'data-collection'],
  priority: 2
});

orchestrator.register('writer', writer, {
  role: 'specialist',
  capabilities: ['writing', 'content-creation'],
  priority: 2
});

orchestrator.register('editor', editor, {
  role: 'specialist',
  capabilities: ['editing', 'proofreading'],
  priority: 1
});

orchestrator.register('reviewer', reviewer, {
  role: 'gatekeeper',
  capabilities: ['review', 'quality-check'],
  priority: 3
});

// 定义内容创作工作流
async function createArticle(topic) {
  const workflow = {
    steps: [
      {
        id: 'research',
        name: '资料研究',
        type: 'research',
        input: { topic }
      },
      {
        id: 'writing',
        name: '文章撰写',
        type: 'writing',
        input: { topic },
        dependsOn: ['research']
      },
      {
        id: 'editing',
        name: '编辑润色',
        type: 'editing',
        dependsOn: ['writing']
      },
      {
        id: 'review',
        name: '质量审核',
        type: 'review',
        dependsOn: ['editing'],
        requiresApproval: true
      }
    ]
  };
  
  try {
    const results = await orchestrator.executeWorkflow(workflow);
    
    return {
      success: true,
      article: results.writing,
      review: results.review,
      metadata: {
        createdAt: new Date().toISOString(),
        topic,
        agentsInvolved: ['researcher', 'writer', 'editor', 'reviewer']
      }
    };
  } catch (error) {
    return {
      success: false,
      error: error.message,
      partialResults: error.partialResults
    };
  }
}

// 使用示例
createArticle('人工智能在医疗领域的应用')
  .then(result => {
    if (result.success) {
      console.log('✅ 文章创作完成！');
      console.log('审核结果:', result.review);
    } else {
      console.log('❌ 创作失败:', result.error);
    }
  });

module.exports = { createArticle, orchestrator };
```

---

## 3.4 消息传递与状态同步

### 共享内存实现

```javascript
// multi-agent/shared-memory.js
class SharedMemory {
  constructor(options = {}) {
    this.store = new Map();
    this.subscribers = new Map();
    this.ttl = options.ttl || 3600000; // 默认1小时过期
  }
  
  // 设置值
  set(key, value, options = {}) {
    const entry = {
      value,
      timestamp: Date.now(),
      ttl: options.ttl || this.ttl,
      accessCount: 0
    };
    
    this.store.set(key, entry);
    
    // 通知订阅者
    this._notifySubscribers(key, value, 'set');
    
    return true;
  }
  
  // 获取值
  get(key) {
    const entry = this.store.get(key);
    
    if (!entry) return undefined;
    
    // 检查是否过期
    if (Date.now() - entry.timestamp > entry.ttl) {
      this.store.delete(key);
      return undefined;
    }
    
    entry.accessCount++;
    return entry.value;
  }
  
  // 原子更新
  async update(key, updater) {
    const current = this.get(key);
    const updated = await updater(current);
    this.set(key, updated);
    return updated;
  }
  
  // 订阅变化
  subscribe(key, callback) {
    if (!this.subscribers.has(key)) {
      this.subscribers.set(key, new Set());
    }
    this.subscribers.get(key).add(callback);
    
    // 返回取消订阅函数
    return () => {
      this.subscribers.get(key).delete(callback);
    };
  }
  
  // 发布消息
  publish(channel, message) {
    this._notifySubscribers(channel, message, 'publish');
  }
  
  _notifySubscribers(key, value, type) {
    const callbacks = this.subscribers.get(key);
    if (callbacks) {
      callbacks.forEach(cb => {
        try {
          cb(value, type);
        } catch (error) {
          console.error('Subscriber error:', error);
        }
      });
    }
  }
  
  // 清理过期数据
  cleanup() {
    const now = Date.now();
    for (const [key, entry] of this.store) {
      if (now - entry.timestamp > entry.ttl) {
        this.store.delete(key);
      }
    }
  }
}

module.exports = SharedMemory;
```

---

## 3.5 故障处理与重试

```javascript
// multi-agent/resilience.js
class ResilientAgent {
  constructor(agent, options = {}) {
    this.agent = agent;
    this.maxRetries = options.maxRetries || 3;
    this.retryDelay = options.retryDelay || 1000;
    this.circuitBreaker = {
      failures: 0,
      threshold: options.circuitThreshold || 5,
      timeout: options.circuitTimeout || 60000,
      state: 'CLOSED' // CLOSED, OPEN, HALF_OPEN
    };
  }
  
  async execute(task) {
    // 检查熔断器状态
    if (this.circuitBreaker.state === 'OPEN') {
      if (Date.now() - this.circuitBreaker.lastFailure < this.circuitBreaker.timeout) {
        throw new Error('Circuit breaker is OPEN');
      }
      this.circuitBreaker.state = 'HALF_OPEN';
    }
    
    let lastError;
    
    for (let attempt = 0; attempt < this.maxRetries; attempt++) {
      try {
        const result = await this.agent.execute(task);
        
        // 成功，重置熔断器
        if (this.circuitBreaker.state === 'HALF_OPEN') {
          this.circuitBreaker.state = 'CLOSED';
          this.circuitBreaker.failures = 0;
        }
        
        return result;
      } catch (error) {
        lastError = error;
        
        console.log(`[ResilientAgent] 尝试 ${attempt + 1}/${this.maxRetries} 失败: ${error.message}`);
        
        if (attempt < this.maxRetries - 1) {
          await this._sleep(this.retryDelay * Math.pow(2, attempt)); // 指数退避
        }
      }
    }
    
    // 全部重试失败，触发熔断
    this.circuitBreaker.failures++;
    this.circuitBreaker.lastFailure = Date.now();
    
    if (this.circuitBreaker.failures >= this.circuitBreaker.threshold) {
      this.circuitBreaker.state = 'OPEN';
      console.error('[ResilientAgent] 熔断器打开');
    }
    
    throw lastError;
  }
  
  _sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

module.exports = ResilientAgent;
```

---

*下一章: [02-patterns.md](02-patterns.md) - 多Agent设计模式*
