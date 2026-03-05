# 模块5: 生产部署与优化

## 5.1 部署架构设计

### 生产环境架构

```
┌─────────────────────────────────────────────────────────┐
│                      Load Balancer                       │
│                     (Nginx/HAProxy)                      │
└───────────────────────┬─────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Agent Node  │ │  Agent Node  │ │  Agent Node  │
│     #1       │ │     #2       │ │     #N       │
└──────┬───────┘ └──────┬───────┘ └──────┬───────┘
       │                │                │
       └────────────────┼────────────────┘
                        ▼
              ┌─────────────────┐
              │   Redis Cluster │
              │  (Shared State) │
              └─────────────────┘
                        │
       ┌────────────────┼────────────────┐
       ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ PostgreSQL   │ │   MongoDB    │ │ Elasticsearch│
│ (Relational) │ │  (Document)  │ │   (Search)   │
└──────────────┘ └──────────────┘ └──────────────┘
```

---

## 5.2 Docker 部署

### Dockerfile

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

# 安装依赖
COPY package*.json ./
RUN npm ci --only=production

# 复制代码
COPY . .

# 生产镜像
FROM node:20-alpine

WORKDIR /app

# 安装运行时依赖
RUN apk add --no-cache curl

# 从builder复制
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/src ./src
COPY --from=builder /app/config ./config

# 非root用户运行
RUN addgroup -g 1001 -S nodejs
RUN adduser -S agent -u 1001
USER agent

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

EXPOSE 3000

CMD ["node", "src/index.js"]
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  agent:
    build: .
    container_name: openclaw-agent
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - REDIS_URL=redis://redis:6379
      - DB_URL=postgresql://postgres:5432/agentdb
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    volumes:
      - ./data:/app/data
      - ./logs:/app/logs
    depends_on:
      - redis
      - postgres
    networks:
      - agent-network
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M

  redis:
    image: redis:7-alpine
    container_name: agent-redis
    restart: unless-stopped
    volumes:
      - redis-data:/data
    networks:
      - agent-network

  postgres:
    image: postgres:15-alpine
    container_name: agent-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: agent
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: agentdb
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - agent-network

  nginx:
    image: nginx:alpine
    container_name: agent-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - agent
    networks:
      - agent-network

volumes:
  redis-data:
  postgres-data:

networks:
  agent-network:
    driver: bridge
```

---

## 5.3 Kubernetes 部署

### Deployment 配置

```yaml
# k8s/agent-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openclaw-agent
  labels:
    app: openclaw-agent
spec:
  replicas: 3
  selector:
    matchLabels:
      app: openclaw-agent
  template:
    metadata:
      labels:
        app: openclaw-agent
    spec:
      containers:
      - name: agent
        image: your-registry/openclaw-agent:latest
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: agent-secrets
              key: redis-url
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: agent-secrets
              key: openai-key
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: data-volume
          mountPath: /app/data
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: agent-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: openclaw-agent-service
spec:
  selector:
    app: openclaw-agent
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: openclaw-agent-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: openclaw-agent
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## 5.4 性能优化

### 连接池配置

```javascript
// config/database.js
module.exports = {
  development: {
    pool: {
      min: 2,
      max: 10
    }
  },
  production: {
    pool: {
      min: 5,
      max: 50,
      acquireTimeoutMillis: 30000,
      createTimeoutMillis: 30000,
      destroyTimeoutMillis: 5000,
      idleTimeoutMillis: 30000,
      reapIntervalMillis: 1000,
      createRetryIntervalMillis: 200
    }
  }
};
```

### 缓存策略

```javascript
// utils/cache.js
const NodeCache = require('node-cache');

class CacheManager {
  constructor() {
    this.localCache = new NodeCache({ stdTTL: 600 });
    this.redis = null;
  }
  
  async init(redisClient) {
    this.redis = redisClient;
  }
  
  async get(key) {
    // 先查本地缓存
    let value = this.localCache.get(key);
    if (value) return value;
    
    // 再查Redis
    if (this.redis) {
      value = await this.redis.get(key);
      if (value) {
        // 回填本地缓存
        this.localCache.set(key, JSON.parse(value));
        return JSON.parse(value);
      }
    }
    
    return null;
  }
  
  async set(key, value, ttl = 600) {
    // 写入本地缓存
    this.localCache.set(key, value, ttl);
    
    // 写入Redis
    if (this.redis) {
      await this.redis.setex(key, ttl, JSON.stringify(value));
    }
  }
  
  async del(key) {
    this.localCache.del(key);
    if (this.redis) {
      await this.redis.del(key);
    }
  }
}

module.exports = new CacheManager();
```

---

## 5.5 监控与日志

### 指标收集

```javascript
// monitoring/metrics.js
const promClient = require('prom-client');

const metrics = {
  requestDuration: new promClient.Histogram({
    name: 'http_request_duration_seconds',
    help: 'HTTP请求持续时间',
    labelNames: ['method', 'route', 'status'],
    buckets: [0.1, 0.5, 1, 2, 5]
  }),
  
  activeAgents: new promClient.Gauge({
    name: 'active_agents_total',
    help: '当前活跃的Agent数量'
  }),
  
  taskCounter: new promClient.Counter({
    name: 'tasks_processed_total',
    help: '处理的任务总数',
    labelNames: ['status']
  }),
  
  llmTokens: new promClient.Counter({
    name: 'llm_tokens_used_total',
    help: 'LLM使用的token数',
    labelNames: ['model', 'type']
  })
};

module.exports = metrics;
```

### 日志配置

```javascript
// config/logger.js
const winston = require('winston');

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { 
    service: 'openclaw-agent',
    version: process.env.npm_package_version
  },
  transports: [
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    }),
    new winston.transports.File({ 
      filename: 'logs/error.log', 
      level: 'error' 
    }),
    new winston.transports.File({ 
      filename: 'logs/combined.log' 
    })
  ]
});

module.exports = logger;
```

---

## 5.6 安全最佳实践

### API密钥管理

```javascript
// security/api-keys.js
const crypto = require('crypto');

class APIKeyManager {
  constructor() {
    this.keys = new Map();
  }
  
  generateKey(clientId, permissions = []) {
    const key = crypto.randomBytes(32).toString('hex');
    const hash = crypto.createHash('sha256').update(key).digest('hex');
    
    this.keys.set(hash, {
      clientId,
      permissions,
      createdAt: Date.now(),
      lastUsed: null,
      usageCount: 0
    });
    
    // 只返回一次原始key
    return { key, hash };
  }
  
  validateKey(key) {
    const hash = crypto.createHash('sha256').update(key).digest('hex');
    const record = this.keys.get(hash);
    
    if (!record) return { valid: false };
    
    record.lastUsed = Date.now();
    record.usageCount++;
    
    return {
      valid: true,
      clientId: record.clientId,
      permissions: record.permissions
    };
  }
  
  revokeKey(hash) {
    return this.keys.delete(hash);
  }
}

module.exports = APIKeyManager;
```

---

*教程全部完成！*
