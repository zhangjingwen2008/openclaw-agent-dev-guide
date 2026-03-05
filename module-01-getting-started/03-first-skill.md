# 1.3 开发你的第一个 Skill

## 什么是 Skill？

Skill 是 OpenClaw 中扩展 Agent 能力的基本单元。一个 Skill 本质上是一个 JavaScript/TypeScript 模块，它：
- 定义输入参数
- 执行特定任务
- 返回结构化结果

## Skill 结构

```javascript
// my-skill.js
module.exports = {
  name: 'weather',
  description: 'Get weather information for a city',
  
  parameters: {
    city: {
      type: 'string',
      description: 'City name',
      required: true
    }
  },
  
  async execute({ city }) {
    // 实现逻辑
    const weather = await fetchWeather(city);
    return {
      temperature: weather.temp,
      condition: weather.desc
    };
  }
};
```

## 实战: 天气查询 Skill

### Step 1: 创建文件

```bash
mkdir -p skills/weather
touch skills/weather/index.js
```

### Step 2: 编写代码

```javascript
// skills/weather/index.js
const axios = require('axios');

module.exports = {
  name: 'weather',
  description: '查询指定城市的天气信息',
  
  parameters: {
    city: {
      type: 'string',
      description: '城市名称（中文或英文）',
      required: true
    }
  },
  
  async execute({ city }) {
    try {
      // 使用免费天气API
      const response = await axios.get(
        `https://wttr.in/${encodeURIComponent(city)}?format=j1`
      );
      
      const current = response.data.current_condition[0];
      
      return {
        success: true,
        data: {
          city: city,
          temperature: current.temp_C + '°C',
          feelsLike: current.FeelsLikeC + '°C',
          humidity: current.humidity + '%',
          description: current.lang_zh[0].value,
          wind: current.windspeedKmph + ' km/h'
        }
      };
    } catch (error) {
      return {
        success: false,
        error: `无法获取 ${city} 的天气信息: ${error.message}`
      };
    }
  }
};
```

### Step 3: 注册 Skill

在 `openclaw.config.yaml` 中添加:

```yaml
skills:
  - name: "weather"
    path: "./skills/weather"
    enabled: true
```

### Step 4: 测试

```javascript
// test-weather.js
const { Agent } = require('openclaw');

const agent = new Agent();

async function test() {
  const result = await agent.run('北京今天天气怎么样？');
  console.log(result);
}

test();
```

运行:
```bash
node test-weather.js
```

预期输出:
```
北京今天的天气是：晴天，气温 15°C，体感温度 13°C，湿度 45%，风速 12 km/h。
```

## Skill 最佳实践

### 1. 错误处理
始终优雅地处理错误，返回有用的错误信息：

```javascript
async execute(params) {
  try {
    // ... 业务逻辑
  } catch (error) {
    return {
      success: false,
      error: `操作失败: ${error.message}`,
      suggestion: '请检查网络连接或稍后重试'
    };
  }
}
```

### 2. 参数验证
在 execute 开始前验证参数：

```javascript
async execute(params) {
  if (!params.city || params.city.trim() === '') {
    return {
      success: false,
      error: '城市名称不能为空'
    };
  }
  // ...
}
```

### 3. 结果格式化
返回结构化的数据，便于 Agent 理解和展示：

```javascript
return {
  success: true,
  data: {
    // 核心数据
  },
  summary: '一句话总结',  // 方便 LLM 理解
  details: '详细信息'     // 可选的补充说明
};
```

## 进阶: 带状态的 Skill

有些 Skill 需要维护状态（如数据库连接、缓存等）：

```javascript
// skills/database/index.js
class DatabaseSkill {
  constructor() {
    this.connection = null;
  }
  
  async init() {
    this.connection = await createConnection();
  }
  
  async execute({ query }) {
    if (!this.connection) {
      await this.init();
    }
    return this.connection.query(query);
  }
  
  async cleanup() {
    if (this.connection) {
      await this.connection.close();
    }
  }
}

module.exports = new DatabaseSkill();
```

## 练习

尝试开发以下 Skills：
1. 📅 日历管理 Skill - 添加/查询日程
2. 📝 笔记记录 Skill - 保存和检索笔记
3. 🔍 网页搜索 Skill - 集成搜索引擎API

完成后，你将掌握 OpenClaw 最核心的扩展机制！

---

*下一章: [04-understanding-memory.md](04-understanding-memory.md) - 深入理解 Memory 系统*
