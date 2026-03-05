# 模块4: 外部集成与扩展

## 4.1 REST API 集成

### HTTP客户端封装

```javascript
// integration/http-client.js
const axios = require('axios');

class HttpClient {
  constructor(baseConfig = {}) {
    this.client = axios.create({
      timeout: baseConfig.timeout || 30000,
      headers: {
        'Content-Type': 'application/json',
        ...baseConfig.headers
      },
      ...baseConfig
    });
    
    // 请求拦截器
    this.client.interceptors.request.use(
      config => {
        console.log(`[HTTP] ${config.method.toUpperCase()} ${config.url}`);
        return config;
      },
      error => Promise.reject(error)
    );
    
    // 响应拦截器
    this.client.interceptors.response.use(
      response => response,
      error => {
        console.error('[HTTP] 请求失败:', error.message);
        return Promise.reject(this._formatError(error));
      }
    );
  }
  
  async get(url, config = {}) {
    const response = await this.client.get(url, config);
    return response.data;
  }
  
  async post(url, data, config = {}) {
    const response = await this.client.post(url, data, config);
    return response.data;
  }
  
  async put(url, data, config = {}) {
    const response = await this.client.put(url, data, config);
    return response.data;
  }
  
  async delete(url, config = {}) {
    const response = await this.client.delete(url, config);
    return response.data;
  }
  
  _formatError(error) {
    if (error.response) {
      return {
        type: 'response',
        status: error.response.status,
        data: error.response.data,
        message: error.message
      };
    }
    if (error.request) {
      return {
        type: 'request',
        message: '网络请求失败，请检查连接'
      };
    }
    return {
      type: 'unknown',
      message: error.message
    };
  }
}

module.exports = HttpClient;
```

### API Skill 模板

```javascript
// skills/api-skill-template/index.js
const HttpClient = require('../../integration/http-client');

class APISkill {
  constructor(config) {
    this.client = new HttpClient({
      baseURL: config.baseURL,
      headers: {
        'Authorization': `Bearer ${config.apiKey}`
      }
    });
    this.config = config;
  }
  
  validate(params) {
    // 子类实现验证逻辑
    return { valid: true };
  }
  
  async execute(params) {
    const validation = this.validate(params);
    if (!validation.valid) {
      return { success: false, error: validation.errors };
    }
    
    try {
      const result = await this._callAPI(params);
      return { success: true, data: result };
    } catch (error) {
      return { 
        success: false, 
        error: `API调用失败: ${error.message}` 
      };
    }
  }
  
  async _callAPI(params) {
    // 子类实现具体API调用
    throw new Error('子类必须实现 _callAPI 方法');
  }
}

module.exports = APISkill;
```

---

## 4.2 数据库集成

### 多数据库支持

```javascript
// integration/database-factory.js
class DatabaseFactory {
  static create(config) {
    switch (config.type) {
      case 'postgresql':
        return new PostgreSQLAdapter(config);
      case 'mysql':
        return new MySQLAdapter(config);
      case 'mongodb':
        return new MongoDBAdapter(config);
      case 'sqlite':
        return new SQLiteAdapter(config);
      default:
        throw new Error(`不支持的数据库类型: ${config.type}`);
    }
  }
}

// PostgreSQL 适配器
class PostgreSQLAdapter {
  constructor(config) {
    const { Pool } = require('pg');
    this.pool = new Pool({
      host: config.host,
      port: config.port || 5432,
      database: config.database,
      user: config.user,
      password: config.password
    });
  }
  
  async query(sql, params) {
    const client = await this.pool.connect();
    try {
      const result = await client.query(sql, params);
      return result.rows;
    } finally {
      client.release();
    }
  }
  
  async close() {
    await this.pool.end();
  }
}

// MongoDB 适配器
class MongoDBAdapter {
  constructor(config) {
    this.config = config;
    this.client = null;
    this.db = null;
  }
  
  async connect() {
    const { MongoClient } = require('mongodb');
    this.client = new MongoClient(this.config.uri);
    await this.client.connect();
    this.db = this.client.db(this.config.database);
  }
  
  async find(collection, query, options = {}) {
    return await this.db.collection(collection)
      .find(query)
      .limit(options.limit || 100)
      .toArray();
  }
  
  async insert(collection, documents) {
    return await this.db.collection(collection).insertMany(documents);
  }
  
  async close() {
    await this.client.close();
  }
}

module.exports = { DatabaseFactory, PostgreSQLAdapter, MongoDBAdapter };
```

---

## 4.3 消息队列集成

### Redis Pub/Sub

```javascript
// integration/redis-messaging.js
const redis = require('redis');

class RedisMessaging {
  constructor(config) {
    this.publisher = redis.createClient(config);
    this.subscriber = redis.createClient(config);
    this.handlers = new Map();
  }
  
  async connect() {
    await this.publisher.connect();
    await this.subscriber.connect();
    
    // 设置消息处理器
    this.subscriber.on('message', (channel, message) => {
      const handler = this.handlers.get(channel);
      if (handler) {
        try {
          const data = JSON.parse(message);
          handler(data);
        } catch (error) {
          console.error('消息解析失败:', error);
        }
      }
    });
  }
  
  async subscribe(channel, handler) {
    this.handlers.set(channel, handler);
    await this.subscriber.subscribe(channel);
    console.log(`[Redis] 订阅频道: ${channel}`);
  }
  
  async publish(channel, message) {
    const data = typeof message === 'string' ? message : JSON.stringify(message);
    await this.publisher.publish(channel, data);
  }
  
  async disconnect() {
    await this.publisher.quit();
    await this.subscriber.quit();
  }
}

module.exports = RedisMessaging;
```

---

## 4.4 Webhook 集成

### Webhook 服务器

```javascript
// integration/webhook-server.js
const express = require('express');
const crypto = require('crypto');

class WebhookServer {
  constructor(config = {}) {
    this.app = express();
    this.port = config.port || 3000;
    this.secret = config.secret; // 用于签名验证
    this.handlers = new Map();
    
    this.app.use(express.json());
    this.app.use(express.raw({ type: 'application/json' }));
    
    // 注册路由
    this.app.post('/webhook/:name', this._handleWebhook.bind(this));
    
    // 健康检查
    this.app.get('/health', (req, res) => {
      res.json({ status: 'ok', timestamp: new Date().toISOString() });
    });
  }
  
  register(name, handler, options = {}) {
    this.handlers.set(name, {
      handler,
      verifySignature: options.verifySignature || false,
      secret: options.secret
    });
    console.log(`[Webhook] 注册处理器: ${name}`);
  }
  
  async _handleWebhook(req, res) {
    const { name } = req.params;
    const registration = this.handlers.get(name);
    
    if (!registration) {
      return res.status(404).json({ error: 'Webhook not found' });
    }
    
    // 验证签名
    if (registration.verifySignature && registration.secret) {
      const signature = req.headers['x-signature'];
      const payload = req.body;
      
      if (!this._verifySignature(payload, signature, registration.secret)) {
        return res.status(401).json({ error: 'Invalid signature' });
      }
    }
    
    try {
      const result = await registration.handler(req.body, req.headers);
      res.json({ success: true, data: result });
    } catch (error) {
      console.error(`[Webhook] 处理失败 (${name}):`, error);
      res.status(500).json({ error: error.message });
    }
  }
  
  _verifySignature(payload, signature, secret) {
    const hmac = crypto.createHmac('sha256', secret);
    hmac.update(typeof payload === 'string' ? payload : JSON.stringify(payload));
    const expected = hmac.digest('hex');
    return crypto.timingSafeEqual(
      Buffer.from(signature),
      Buffer.from(expected)
    );
  }
  
  start() {
    return new Promise((resolve) => {
      this.server = this.app.listen(this.port, () => {
        console.log(`[Webhook] 服务器启动在端口 ${this.port}`);
        resolve();
      });
    });
  }
  
  stop() {
    return new Promise((resolve) => {
      if (this.server) {
        this.server.close(resolve);
      } else {
        resolve();
      }
    });
  }
}

// 使用示例
async function setupWebhooks() {
  const server = new WebhookServer({ port: 3000, secret: 'my-secret' });
  
  // GitHub webhook
  server.register('github', async (payload) => {
    console.log('收到GitHub事件:', payload.action);
    // 触发Agent处理
    const agent = new Agent();
    await agent.run(`处理GitHub事件: ${JSON.stringify(payload)}`);
  }, { verifySignature: true, secret: 'github-webhook-secret' });
  
  // Stripe支付webhook
  server.register('stripe', async (payload) => {
    console.log('收到Stripe事件:', payload.type);
    // 处理支付相关逻辑
  }, { verifySignature: true });
  
  await server.start();
  return server;
}

module.exports = { WebhookServer, setupWebhooks };
```

---

## 4.5 第三方服务集成示例

### Slack 集成

```javascript
// skills/slack-integration/index.js
const { WebClient } = require('@slack/web-api');

class SlackSkill {
  constructor(config) {
    this.client = new WebClient(config.token);
    this.defaultChannel = config.defaultChannel;
  }
  
  async sendMessage(text, options = {}) {
    const channel = options.channel || this.defaultChannel;
    
    try {
      const result = await this.client.chat.postMessage({
        channel,
        text,
        blocks: options.blocks,
        attachments: options.attachments
      });
      
      return {
        success: true,
        ts: result.ts,
        channel: result.channel
      };
    } catch (error) {
      return {
        success: false,
        error: error.message
      };
    }
  }
  
  async uploadFile(filePath, options = {}) {
    const channel = options.channel || this.defaultChannel;
    
    try {
      const result = await this.client.files.uploadV2({
        channel_id: channel,
        file: filePath,
        title: options.title,
        initial_comment: options.comment
      });
      
      return { success: true, fileId: result.file.id };
    } catch (error) {
      return { success: false, error: error.message };
    }
  }
}

module.exports = SlackSkill;
```

---

*下一章: [02-automation.md](02-automation.md) - 自动化工作流*
