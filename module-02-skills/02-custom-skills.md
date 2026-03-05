# 2.2 开发自定义高级技能

## 技能开发框架

### 标准结构模板

```javascript
// skills/template/index.js
class Skill {
  constructor(config = {}) {
    this.config = {
      timeout: 30000,
      retries: 3,
      ...config
    };
    this.initialized = false;
  }
  
  // 初始化（可选）
  async init() {
    // 建立连接、加载资源等
    this.initialized = true;
  }
  
  // 参数验证
  validate(params) {
    const errors = [];
    
    // 必需参数检查
    if (!params.requiredField) {
      errors.push('缺少必需参数: requiredField');
    }
    
    // 类型检查
    if (params.count && typeof params.count !== 'number') {
      errors.push('count 必须是数字');
    }
    
    return errors.length > 0 ? { valid: false, errors } : { valid: true };
  }
  
  // 核心执行逻辑
  async execute(params) {
    // 1. 验证参数
    const validation = this.validate(params);
    if (!validation.valid) {
      return {
        success: false,
        error: validation.errors.join(', ')
      };
    }
    
    // 2. 确保已初始化
    if (!this.initialized) {
      await this.init();
    }
    
    // 3. 执行业务逻辑（带重试）
    let lastError;
    for (let i = 0; i < this.config.retries; i++) {
      try {
        const result = await this._doExecute(params);
        return { success: true, data: result };
      } catch (error) {
        lastError = error;
        await this._sleep(1000 * (i + 1)); // 指数退避
      }
    }
    
    // 4. 全部重试失败
    return {
      success: false,
      error: `执行失败（重试${this.config.retries}次）: ${lastError.message}`
    };
  }
  
  // 实际业务逻辑（子类实现）
  async _doExecute(params) {
    throw new Error('子类必须实现 _doExecute 方法');
  }
  
  // 清理资源（可选）
  async cleanup() {
    this.initialized = false;
  }
  
  _sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

module.exports = Skill;
```

---

## 实战：邮件发送技能

```javascript
// skills/email-sender/index.js
const nodemailer = require('nodemailer');

class EmailSenderSkill {
  constructor(config = {}) {
    this.config = {
      host: config.host || process.env.SMTP_HOST,
      port: config.port || 587,
      user: config.user || process.env.SMTP_USER,
      pass: config.pass || process.env.SMTP_PASS,
      secure: config.secure || false
    };
    this.transporter = null;
  }
  
  async init() {
    this.transporter = nodemailer.createTransport({
      host: this.config.host,
      port: this.config.port,
      secure: this.config.secure,
      auth: {
        user: this.config.user,
        pass: this.config.pass
      }
    });
    
    // 验证连接
    await this.transporter.verify();
  }
  
  validate(params) {
    const required = ['to', 'subject'];
    const missing = required.filter(field => !params[field]);
    
    if (missing.length > 0) {
      return {
        valid: false,
        errors: [`缺少必需参数: ${missing.join(', ')}`]
      };
    }
    
    // 邮箱格式验证
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    const recipients = Array.isArray(params.to) ? params.to : [params.to];
    
    for (const email of recipients) {
      if (!emailRegex.test(email)) {
        return {
          valid: false,
          errors: [`无效的邮箱地址: ${email}`]
        };
      }
    }
    
    return { valid: true };
  }
  
  async execute(params) {
    const validation = this.validate(params);
    if (!validation.valid) {
      return { success: false, error: validation.errors[0] };
    }
    
    if (!this.transporter) {
      await this.init();
    }
    
    try {
      const info = await this.transporter.sendMail({
        from: this.config.user,
        to: params.to,
        subject: params.subject,
        text: params.text,
        html: params.html,
        attachments: params.attachments
      });
      
      return {
        success: true,
        data: {
          messageId: info.messageId,
          accepted: info.accepted,
          rejected: info.rejected
        }
      };
    } catch (error) {
      return {
        success: false,
        error: `邮件发送失败: ${error.message}`
      };
    }
  }
}

module.exports = EmailSenderSkill;
```

### 配置示例

```yaml
# openclaw.config.yaml
skills:
  - name: "email-sender"
    path: "./skills/email-sender"
    enabled: true
    config:
      host: "smtp.gmail.com"
      port: 587
      user: "your-email@gmail.com"
      # 密码从环境变量读取
```

---

## 实战：数据库操作技能

```javascript
// skills/database/index.js
const { Pool } = require('pg');

class DatabaseSkill {
  constructor(config = {}) {
    this.pool = new Pool({
      host: config.host || process.env.DB_HOST,
      port: config.port || 5432,
      database: config.database || process.env.DB_NAME,
      user: config.user || process.env.DB_USER,
      password: config.password || process.env.DB_PASS,
      max: 20, // 最大连接数
      idleTimeoutMillis: 30000
    });
  }
  
  validate(params) {
    if (!params.operation) {
      return { valid: false, errors: ['必须指定 operation'] };
    }
    
    const validOps = ['query', 'insert', 'update', 'delete'];
    if (!validOps.includes(params.operation)) {
      return {
        valid: false,
        errors: [`无效的操作: ${params.operation}. 支持: ${validOps.join(', ')}`]
      };
    }
    
    if (!params.sql && params.operation === 'query') {
      return { valid: false, errors: ['query 操作需要 sql 参数'] };
    }
    
    return { valid: true };
  }
  
  async execute(params) {
    const validation = this.validate(params);
    if (!validation.valid) {
      return { success: false, error: validation.errors[0] };
    }
    
    const client = await this.pool.connect();
    
    try {
      let result;
      
      switch (params.operation) {
        case 'query':
          result = await client.query(params.sql, params.values);
          return {
            success: true,
            data: {
              rows: result.rows,
              rowCount: result.rowCount
            }
          };
          
        case 'insert':
          const insertSQL = this._buildInsert(params.table, params.data);
          result = await client.query(insertSQL.sql, insertSQL.values);
          return {
            success: true,
            data: { insertedId: result.rows[0]?.id }
          };
          
        case 'update':
          const updateSQL = this._buildUpdate(params.table, params.data, params.where);
          result = await client.query(updateSQL.sql, updateSQL.values);
          return {
            success: true,
            data: { updatedCount: result.rowCount }
          };
          
        case 'delete':
          const deleteSQL = this._buildDelete(params.table, params.where);
          result = await client.query(deleteSQL.sql, deleteSQL.values);
          return {
            success: true,
            data: { deletedCount: result.rowCount }
          };
      }
    } catch (error) {
      return { success: false, error: error.message };
    } finally {
      client.release();
    }
  }
  
  _buildInsert(table, data) {
    const keys = Object.keys(data);
    const values = Object.values(data);
    const placeholders = values.map((_, i) => `$${i + 1}`).join(', ');
    
    return {
      sql: `INSERT INTO ${table} (${keys.join(', ')}) VALUES (${placeholders}) RETURNING id`,
      values
    };
  }
  
  _buildUpdate(table, data, where) {
    const setClause = Object.keys(data).map((key, i) => `${key} = $${i + 1}`).join(', ');
    const values = [...Object.values(data)];
    
    let whereClause = '';
    if (where) {
      const whereKeys = Object.keys(where);
      whereClause = 'WHERE ' + whereKeys.map((key, i) => `${key} = $${values.length + i + 1}`).join(' AND ');
      values.push(...Object.values(where));
    }
    
    return {
      sql: `UPDATE ${table} SET ${setClause} ${whereClause}`,
      values
    };
  }
  
  _buildDelete(table, where) {
    const whereKeys = Object.keys(where);
    const whereClause = whereKeys.map((key, i) => `${key} = $${i + 1}`).join(' AND ');
    
    return {
      sql: `DELETE FROM ${table} WHERE ${whereClause}`,
      values: Object.values(where)
    };
  }
  
  async cleanup() {
    await this.pool.end();
  }
}

module.exports = DatabaseSkill;
```

---

## 技能打包与发布

### 创建 package.json

```json
{
  "name": "openclaw-skill-email",
  "version": "1.0.0",
  "description": "Email sending skill for OpenClaw",
  "main": "index.js",
  "keywords": ["openclaw", "skill", "email"],
  "author": "Your Name",
  "license": "MIT",
  "dependencies": {
    "nodemailer": "^6.9.0"
  },
  "peerDependencies": {
    "openclaw": ">=2.0.0"
  }
}
```

### 发布到 npm

```bash
# 1. 登录 npm
npm login

# 2. 发布
npm publish

# 3. 安装使用
npm install openclaw-skill-email
```

---

## 技能最佳实践清单

- [ ] 完善的参数验证
- [ ] 优雅的错误处理
- [ ] 支持重试机制
- [ ] 资源正确释放
- [ ] 详细的日志记录
- [ ] 完整的单元测试
- [ ] 清晰的文档说明
- [ ] 版本兼容性声明

---

*下一章: [03-skill-composition.md](03-skill-composition.md) - 技能组合与编排*
