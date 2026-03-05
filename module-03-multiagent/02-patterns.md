# 3.2 多Agent设计模式

## 主从模式 (Master-Worker)

### 适用场景
- 任务可以分解为独立子任务
- 需要并行处理大量数据
- 结果需要汇总

### 实现代码

```javascript
// patterns/master-worker.js
class MasterWorkerPattern {
  constructor(workerCount = 4) {
    this.workers = [];
    this.taskQueue = [];
    this.results = [];
    
    // 创建Worker Agent
    for (let i = 0; i < workerCount; i++) {
      this.workers.push({
        id: i,
        agent: new Agent({
          name: `worker-${i}`,
          systemPrompt: '你是一个高效的任务执行者。'
        }),
        busy: false
      });
    }
  }
  
  async execute(tasks, aggregator) {
    console.log(`[Master] 分发 ${tasks.length} 个任务给 ${this.workers.length} 个Worker`);
    
    // 分配任务
    const taskPromises = tasks.map((task, index) => 
      this._assignToWorker(task, index)
    );
    
    // 等待所有任务完成
    const results = await Promise.allSettled(taskPromises);
    
    // 汇总结果
    if (aggregator) {
      return aggregator(results);
    }
    
    return results;
  }
  
  async _assignToWorker(task, taskId) {
    // 找到空闲的Worker
    let worker = this.workers.find(w => !w.busy);
    
    // 如果没有空闲Worker，等待
    while (!worker) {
      await this._sleep(100);
      worker = this.workers.find(w => !w.busy);
    }
    
    // 标记为忙碌
    worker.busy = true;
    
    try {
      console.log(`[Worker-${worker.id}] 开始执行任务 ${taskId}`);
      const result = await worker.agent.execute(task);
      console.log(`[Worker-${worker.id}] 完成任务 ${taskId}`);
      return { taskId, status: 'success', data: result };
    } catch (error) {
      console.error(`[Worker-${worker.id}] 任务 ${taskId} 失败:`, error);
      return { taskId, status: 'error', error: error.message };
    } finally {
      worker.busy = false;
    }
  }
  
  _sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// 使用示例：批量文档处理
async function batchProcessDocuments(documents) {
  const masterWorker = new MasterWorkerPattern(4);
  
  const tasks = documents.map(doc => ({
    type: 'summarize',
    content: doc.content,
    maxLength: 200
  }));
  
  // 自定义汇总逻辑
  const aggregator = (results) => {
    const successful = results.filter(r => r.value?.status === 'success');
    const failed = results.filter(r => r.value?.status === 'error');
    
    return {
      total: results.length,
      successful: successful.length,
      failed: failed.length,
      summaries: successful.map(r => r.value.data)
    };
  };
  
  return await masterWorker.execute(tasks, aggregator);
}

module.exports = { MasterWorkerPattern, batchProcessDocuments };
```

---

## 管道模式 (Pipeline)

### 适用场景
- 数据处理有明确的阶段
- 每个阶段输出是下一阶段输入
- 需要流式处理

### 实现代码

```javascript
// patterns/pipeline.js
class PipelinePattern {
  constructor() {
    this.stages = [];
  }
  
  // 添加处理阶段
  addStage(name, processor, options = {}) {
    this.stages.push({
      name,
      processor,
      parallel: options.parallel || false,
      bufferSize: options.bufferSize || 10
    });
    return this;
  }
  
  async execute(input) {
    let data = input;
    const context = {};
    
    for (const stage of this.stages) {
      console.log(`[Pipeline] 进入阶段: ${stage.name}`);
      
      try {
        if (Array.isArray(data) && stage.parallel) {
          // 并行处理数组元素
          data = await Promise.all(
            data.map(item => stage.processor(item, context))
          );
        } else {
          // 串行处理
          data = await stage.processor(data, context);
        }
        
        context[stage.name] = data;
        console.log(`[Pipeline] 阶段 ${stage.name} 完成`);
      } catch (error) {
        console.error(`[Pipeline] 阶段 ${stage.name} 失败:`, error);
        throw error;
      }
    }
    
    return { result: data, context };
  }
  
  // 流式处理
  async *streamExecute(inputs) {
    for await (const input of inputs) {
      yield await this.execute(input);
    }
  }
}

// 使用示例：文章发布流程
async function articlePublishingPipeline(articleDraft) {
  const pipeline = new PipelinePattern();
  
  const editorAgent = new Agent({ name: 'editor' });
  const seoAgent = new Agent({ name: 'seo' });
  const publisherAgent = new Agent({ name: 'publisher' });
  
  pipeline
    .addStage('edit', async (draft) => {
      return await editorAgent.run(`编辑并改进以下文章：\n${draft}`);
    })
    .addStage('seo-optimize', async (edited) => {
      return await seoAgent.run(`为以下文章生成SEO标题、描述和关键词：\n${edited}`);
    })
    .addStage('publish', async (optimized, context) => {
      return await publisherAgent.run(
        `发布文章到博客平台：\n标题: ${context['seo-optimize'].title}\n内容: ${context.edit}`
      );
    });
  
  return await pipeline.execute(articleDraft);
}

module.exports = { PipelinePattern, articlePublishingPipeline };
```

---

## 投票模式 (Voting)

### 适用场景
- 需要高可靠性决策
- 多个Agent评估同一问题
- 消除单一Agent偏见

### 实现代码

```javascript
// patterns/voting.js
class VotingPattern {
  constructor(agents, options = {}) {
    this.agents = agents;
    this.strategy = options.strategy || 'majority'; // majority, unanimous, weighted
    this.weights = options.weights || agents.map(() => 1);
  }
  
  async decide(question, options) {
    console.log(`[Voting] 对问题进行投票: ${question}`);
    
    // 收集所有Agent的投票
    const votes = await Promise.all(
      this.agents.map(async (agent, index) => {
        const vote = await agent.run(`
          问题: ${question}
          选项: ${options.join(', ')}
          请选择一个最符合的选项，只返回选项名称。
        `);
        return {
          agent: agent.name,
          vote: vote.trim(),
          weight: this.weights[index]
        };
      })
    );
    
    // 统计票数
    const tally = {};
    votes.forEach(({ vote, weight }) => {
      tally[vote] = (tally[vote] || 0) + weight;
    });
    
    console.log('[Voting] 投票结果:', tally);
    
    // 根据策略决定结果
    switch (this.strategy) {
      case 'majority':
        return this._majorityDecision(tally, votes);
      case 'unanimous':
        return this._unanimousDecision(tally, votes);
      case 'weighted':
        return this._weightedDecision(tally, votes);
      default:
        throw new Error(`未知的投票策略: ${this.strategy}`);
    }
  }
  
  _majorityDecision(tally, allVotes) {
    const totalWeight = allVotes.reduce((sum, v) => sum + v.weight, 0);
    const threshold = totalWeight / 2;
    
    for (const [option, count] of Object.entries(tally)) {
      if (count > threshold) {
        return {
          decision: option,
          confidence: count / totalWeight,
          votes: allVotes
        };
      }
    }
    
    // 没有多数，选择票数最多的
    const winner = Object.entries(tally).sort((a, b) => b[1] - a[1])[0];
    return {
      decision: winner[0],
      confidence: winner[1] / totalWeight,
      votes: allVotes,
      note: '无绝对多数，选择相对多数'
    };
  }
  
  _unanimousDecision(tally, allVotes) {
    const options = Object.keys(tally);
    if (options.length === 1) {
      return {
        decision: options[0],
        confidence: 1,
        votes: allVotes
      };
    }
    
    return {
      decision: null,
      confidence: 0,
      votes: allVotes,
      error: '未达成一致意见'
    };
  }
  
  _weightedDecision(tally, allVotes) {
    const totalWeight = allVotes.reduce((sum, v) => sum + v.weight, 0);
    const winner = Object.entries(tally).sort((a, b) => b[1] - a[1])[0];
    
    return {
      decision: winner[0],
      confidence: winner[1] / totalWeight,
      votes: allVotes
    };
  }
}

// 使用示例：代码审查决策
async function codeReviewDecision(codeSnippet) {
  const reviewers = [
    new Agent({ name: 'security-expert', systemPrompt: '你关注代码安全性' }),
    new Agent({ name: 'performance-expert', systemPrompt: '你关注代码性能' }),
    new Agent({ name: 'maintainability-expert', systemPrompt: '你关注代码可维护性' })
  ];
  
  const voting = new VotingPattern(reviewers, {
    strategy: 'majority',
    weights: [1.2, 1.0, 1.0] // 安全专家权重稍高
  });
  
  const result = await voting.decide(
    `审查以下代码是否可以通过：\n${codeSnippet}`,
    ['通过', '需要修改', '拒绝']
  );
  
  return result;
}

module.exports = { VotingPattern, codeReviewDecision };
```

---

## 市场模式 (Market-based)

### 适用场景
- Agent能力差异大
- 需要动态任务分配
- 引入竞争机制提高效率

### 实现代码

```javascript
// patterns/market.js
class MarketPattern {
  constructor() {
    this.agents = new Map(); // agent -> capabilities & cost
    this.taskHistory = [];
  }
  
  registerAgent(agent, capabilities, costFunction) {
    this.agents.set(agent, {
      capabilities: new Set(capabilities),
      costFunction,
      reputation: 1.0,
      completedTasks: 0,
      failedTasks: 0
    });
  }
  
  async assignTask(task) {
    // 找出能处理此任务的Agent
    const candidates = [];
    
    for (const [agent, info] of this.agents) {
      if (this._canHandle(info.capabilities, task.requirements)) {
        const estimatedCost = info.costFunction(task);
        const adjustedCost = estimatedCost / info.reputation; // 声誉好的成本更低
        
        candidates.push({
          agent,
          cost: adjustedCost,
          reputation: info.reputation
        });
      }
    }
    
    if (candidates.length === 0) {
      throw new Error('没有Agent能处理此任务');
    }
    
    // 选择成本最低的（模拟竞价）
    candidates.sort((a, b) => a.cost - b.cost);
    const winner = candidates[0];
    
    console.log(`[Market] 任务分配给 ${winner.agent.name}, 预估成本: ${winner.cost}`);
    
    try {
      const startTime = Date.now();
      const result = await winner.agent.execute(task);
      const duration = Date.now() - startTime;
      
      // 更新声誉
      this._updateReputation(winner.agent, true, duration);
      
      this.taskHistory.push({
        task,
        agent: winner.agent.name,
        cost: winner.cost,
        duration,
        success: true
      });
      
      return result;
    } catch (error) {
      this._updateReputation(winner.agent, false);
      
      this.taskHistory.push({
        task,
        agent: winner.agent.name,
        success: false,
        error: error.message
      });
      
      throw error;
    }
  }
  
  _canHandle(capabilities, requirements) {
    return requirements.every(req => capabilities.has(req));
  }
  
  _updateReputation(agent, success, duration) {
    const info = this.agents.get(agent);
    
    if (success) {
      info.completedTasks++;
      info.reputation = Math.min(2.0, info.reputation * 1.05);
    } else {
      info.failedTasks++;
      info.reputation = Math.max(0.1, info.reputation * 0.8);
    }
  }
  
  getMarketStats() {
    const stats = [];
    for (const [agent, info] of this.agents) {
      stats.push({
        name: agent.name,
        reputation: info.reputation,
        completed: info.completedTasks,
        failed: info.failedTasks,
        successRate: info.completedTasks / (info.completedTasks + info.failedTasks)
      });
    }
    return stats.sort((a, b) => b.reputation - a.reputation);
  }
}

module.exports = { MarketPattern };
```

---

*下一章: [03-conflict-resolution.md](03-conflict-resolution.md) - 冲突解决与一致性*
