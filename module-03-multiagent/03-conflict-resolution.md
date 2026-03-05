# 3.3 冲突解决与一致性

## 资源竞争处理

### 分布式锁实现

```javascript
// multi-agent/distributed-lock.js
class DistributedLock {
  constructor(redisClient) {
    this.redis = redisClient;
    this.locks = new Map();
  }
  
  async acquire(resourceId, ttl = 30000) {
    const lockKey = `lock:${resourceId}`;
    const token = `${Date.now()}-${Math.random()}`;
    
    // 尝试获取锁
    const acquired = await this.redis.set(lockKey, token, {
      NX: true, // 仅当不存在时才设置
      PX: ttl   // 过期时间（毫秒）
    });
    
    if (acquired === 'OK') {
      this.locks.set(resourceId, token);
      
      // 启动自动续期
      this._startRenewal(resourceId, token, ttl);
      
      return {
        success: true,
        release: () => this.release(resourceId, token)
      };
    }
    
    return { success: false, reason: '资源已被锁定' };
  }
  
  async release(resourceId, token) {
    const lockKey = `lock:${resourceId}`;
    const currentToken = await this.redis.get(lockKey);
    
    // 验证token，防止误释放别人的锁
    if (currentToken === token) {
      await this.redis.del(lockKey);
      this.locks.delete(resourceId);
      return { success: true };
    }
    
    return { success: false, reason: '无法释放不属于自己的锁' };
  }
  
  _startRenewal(resourceId, token, ttl) {
    // 每1/3 TTL时间续期一次
    const interval = setInterval(async () => {
      const lockKey = `lock:${resourceId}`;
      const current = await this.redis.get(lockKey);
      
      if (current === token) {
        await this.redis.pexpire(lockKey, ttl);
      } else {
        clearInterval(interval);
      }
    }, ttl / 3);
  }
}

module.exports = DistributedLock;
```

---

## 决策投票机制

```javascript
// multi-agent/consensus.js
class ConsensusProtocol {
  constructor(agents, options = {}) {
    this.agents = agents;
    this.threshold = options.threshold || Math.ceil(agents.length * 2 / 3);
    this.timeout = options.timeout || 30000;
  }
  
  async propose(proposal) {
    console.log(`[Consensus] 发起提案: ${proposal.id}`);
    
    const votes = [];
    const promises = this.agents.map(agent => 
      this._requestVote(agent, proposal).then(vote => {
        votes.push({ agent: agent.name, vote });
      }).catch(error => {
        votes.push({ agent: agent.name, error: error.message });
      })
    );
    
    // 等待所有投票或超时
    await Promise.race([
      Promise.all(promises),
      this._sleep(this.timeout)
    ]);
    
    // 统计结果
    const accepts = votes.filter(v => v.vote === 'accept').length;
    const rejects = votes.filter(v => v.vote === 'reject').length;
    
    console.log(`[Consensus] 投票结果: ${accepts} 接受, ${rejects} 拒绝`);
    
    if (accepts >= this.threshold) {
      return {
        status: 'committed',
        proposal: proposal,
        votes: votes
      };
    }
    
    return {
      status: 'rejected',
      reason: `未达到共识阈值 (${accepts}/${this.threshold})`,
      votes: votes
    };
  }
  
  async _requestVote(agent, proposal) {
    const response = await agent.run(`
      请对以下提案进行投票：
      ID: ${proposal.id}
      内容: ${proposal.content}
      
      请回复 "accept" 或 "reject"，并简要说明理由。
    `);
    
    // 解析投票结果
    if (response.toLowerCase().includes('accept')) {
      return 'accept';
    }
    return 'reject';
  }
  
  _sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

module.exports = ConsensusProtocol;
```

---

## 死锁检测与恢复

```javascript
// multi-agent/deadlock-detector.js
class DeadlockDetector {
  constructor() {
    this.waitGraph = new Map(); // agent -> [agents it's waiting for]
    this.resourceOwners = new Map(); // resource -> agent
    this.waitingForResource = new Map(); // agent -> resource
  }
  
  // Agent请求资源
  requestResource(agentId, resourceId) {
    const owner = this.resourceOwners.get(resourceId);
    
    if (!owner) {
      // 资源空闲，直接分配
      this.resourceOwners.set(resourceId, agentId);
      return { granted: true };
    }
    
    if (owner === agentId) {
      return { granted: true, alreadyOwned: true };
    }
    
    // 资源被占用，加入等待图
    this.waitingForResource.set(agentId, resourceId);
    
    if (!this.waitGraph.has(agentId)) {
      this.waitGraph.set(agentId, new Set());
    }
    this.waitGraph.get(agentId).add(owner);
    
    // 检测死锁
    const cycle = this._detectCycle();
    if (cycle) {
      console.error('[Deadlock] 检测到死锁:', cycle);
      return { 
        granted: false, 
        deadlock: true, 
        cycle,
        resolution: this._resolveDeadlock(cycle)
      };
    }
    
    return { granted: false, waiting: true };
  }
  
  // Agent释放资源
  releaseResource(agentId, resourceId) {
    if (this.resourceOwners.get(resourceId) === agentId) {
      this.resourceOwners.delete(resourceId);
      this.waitGraph.delete(agentId);
      this.waitingForResource.delete(agentId);
      
      // 通知等待的Agent
      return { released: true };
    }
    
    return { released: false, reason: '不是资源所有者' };
  }
  
  _detectCycle() {
    const visited = new Set();
    const recursionStack = new Set();
    
    const dfs = (node, path) => {
      visited.add(node);
      recursionStack.add(node);
      path.push(node);
      
      const neighbors = this.waitGraph.get(node) || new Set();
      for (const neighbor of neighbors) {
        if (!visited.has(neighbor)) {
          const cycle = dfs(neighbor, path);
          if (cycle) return cycle;
        } else if (recursionStack.has(neighbor)) {
          // 发现环
          const cycleStart = path.indexOf(neighbor);
          return path.slice(cycleStart);
        }
      }
      
      path.pop();
      recursionStack.delete(node);
      return null;
    };
    
    for (const node of this.waitGraph.keys()) {
      if (!visited.has(node)) {
        const cycle = dfs(node, []);
        if (cycle) return cycle;
      }
    }
    
    return null;
  }
  
  _resolveDeadlock(cycle) {
    // 策略：终止优先级最低的Agent
    const victim = cycle[0]; // 简化：选择环中第一个
    
    console.log(`[Deadlock] 选择牺牲者: ${victim}`);
    
    // 释放该Agent持有的所有资源
    for (const [resource, owner] of this.resourceOwners.entries()) {
      if (owner === victim) {
        this.releaseResource(victim, resource);
      }
    }
    
    return { victim, action: 'terminated' };
  }
}

module.exports = DeadlockDetector;
```

---

*模块3完成*
