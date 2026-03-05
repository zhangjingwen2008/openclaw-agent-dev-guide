# 模块2: 技能系统深度解析

## 2.1 内置技能全览

OpenClaw 提供了丰富的内置技能，开箱即用：

| 技能名称 | 功能 | 使用场景 |
|---------|------|---------|
| `search` | 网页搜索 | 获取实时信息 |
| `code` | 代码执行 | 运行Python/JS代码 |
| `browser` | 浏览器控制 | 网页操作、截图 |
| `file` | 文件操作 | 读写本地文件 |
| `memory` | 记忆管理 | 持久化存储 |

### 启用内置技能

```yaml
# openclaw.config.yaml
skills:
  - name: "search"
    enabled: true
    config:
      engine: "google"  # google/bing/baidu
      maxResults: 5
      
  - name: "code"
    enabled: true
    config:
      allowedLanguages: ["python", "javascript"]
      timeout: 30000
      
  - name: "browser"
    enabled: true
    config:
      headless: true
      defaultTimeout: 10000
```

---

## 2.2 Search Skill 深度使用

### 基础搜索

```javascript
const agent = new Agent();

// 简单查询
const result = await agent.run('搜索最新的AI新闻');
```

### 高级配置

```javascript
// 自定义搜索参数
const searchConfig = {
  site: 'github.com',        // 限定站点
  fileType: 'pdf',           // 文件类型
  timeRange: 'week',         // 时间范围
  region: 'cn'               // 地区
};

const result = await agent.run(
  '查找本周发布的机器学习论文',
  { search: searchConfig }
);
```

### 实战：竞品分析助手

```javascript
// skills/competitor-analyzer/index.js
module.exports = {
  name: 'competitor-analyzer',
  description: '分析竞争对手产品',
  
  async execute({ productName }) {
    const agent = new Agent();
    
    // 1. 搜索竞品信息
    const searchResult = await agent.run(
      `搜索 ${productName} 的功能介绍、定价、用户评价`
    );
    
    // 2. 访问官网获取详细信息
    const websiteInfo = await agent.run(
      `访问 ${productName} 官网，提取核心功能和定价方案`
    );
    
    // 3. 生成分析报告
    const analysis = await agent.run(
      `基于以下信息生成竞品分析报告：\n${searchResult}\n${websiteInfo}`
    );
    
    return {
      success: true,
      data: {
        product: productName,
        analysis: analysis,
        sources: searchResult.sources
      }
    };
  }
};
```

---

## 2.3 Code Skill 实战编程

### 安全执行环境

Code Skill 在沙箱环境中运行代码，确保安全：

```javascript
// 安全的代码执行
const result = await agent.run(`
  帮我计算斐波那契数列的前20项
  要求：使用Python，展示代码和结果
`);
```

### 数据分析实战

```javascript
// skills/data-analyzer/index.js
const fs = require('fs');

module.exports = {
  name: 'data-analyzer',
  
  async execute({ filePath, analysisType }) {
    // 读取CSV文件
    const csvContent = fs.readFileSync(filePath, 'utf8');
    
    const agent = new Agent();
    
    // 使用Code Skill进行数据分析
    const analysis = await agent.run(`
      请分析以下CSV数据，进行${analysisType}分析：
      \n\n${csvContent}
      \n\n要求：
      1. 使用pandas加载数据
      2. 展示数据统计摘要
      3. 生成可视化图表（matplotlib）
      4. 输出关键洞察
    `);
    
    return {
      success: true,
      data: analysis
    };
  }
};
```

---

## 2.4 Browser Skill 网页自动化

### 基本操作

```javascript
// 打开网页并提取信息
const result = await agent.run(`
  打开 https://example.com
  提取页面标题和所有链接
  截图保存到 ./screenshot.png
`);
```

### 表单自动填写

```javascript
// skills/form-filler/index.js
module.exports = {
  name: 'form-filler',
  
  async execute({ url, formData }) {
    const agent = new Agent();
    
    await agent.run(`
      访问 ${url}
      按以下信息填写表单：
      ${JSON.stringify(formData, null, 2)}
      提交表单并确认成功
    `);
    
    return { success: true };
  }
};
```

---

## 2.5 File Skill 文件管理

### 读写操作

```javascript
// 写入文件
await agent.run('
  创建一个配置文件 config.json，包含：
  - 服务器地址
  - API密钥占位符
  - 日志级别设置
');

// 批量处理
await agent.run('
  读取 ./data 目录下所有 .txt 文件
  统计每个文件的行数和词数
  生成汇总报告保存到 summary.md
');
```

---

## 2.6 Memory Skill 持久化记忆

### 长期记忆存储

```javascript
// 保存重要信息
await agent.run('
  记住：用户的名字是景文，偏好使用飞书沟通
  这是长期记忆，下次对话还要记得
');

// 检索记忆
await agent.run('用户之前告诉过我他的名字吗？');
```

### 结构化记忆

```javascript
// skills/user-profile/index.js
module.exports = {
  name: 'user-profile',
  
  async execute({ action, data }) {
    const memory = new Memory();
    
    if (action === 'save') {
      await memory.set(`user:${data.userId}`, data);
      return { success: true };
    }
    
    if (action === 'get') {
      const profile = await memory.get(`user:${data.userId}`);
      return { success: true, data: profile };
    }
  }
};
```

---

## 2.7 技能组合策略

### 多技能协作示例

```javascript
// 复杂任务：研究一个公司
async function researchCompany(companyName) {
  const agent = new Agent();
  
  // 1. 搜索基本信息
  const basicInfo = await agent.run(
    `搜索 ${companyName} 的公司简介、成立时间、融资情况`
  );
  
  // 2. 访问官网深入了解
  const websiteData = await agent.run(
    `访问 ${companyName} 官网，提取产品信息和团队介绍`
  );
  
  // 3. 分析数据生成报告
  const report = await agent.run(
    `基于以下信息生成投资分析报告：\n${basicInfo}\n${websiteData}`
  );
  
  // 4. 保存到文件
  await agent.run(
    `将以下报告保存到 reports/${companyName}-analysis.md：\n${report}`
  );
  
  return report;
}
```

---

## 2.8 技能调试技巧

### 开启调试模式

```yaml
# openclaw.config.yaml
debug: true
logLevel: verbose
```

### 查看技能调用日志

```bash
# 查看详细日志
openclaw run --verbose

# 只查看特定技能的日志
openclaw run --skill-filter=search,code
```

### 单元测试技能

```javascript
// tests/weather-skill.test.js
const weatherSkill = require('../skills/weather');

async function test() {
  const result = await weatherSkill.execute({ city: '北京' });
  
  console.assert(result.success === true, '应该成功');
  console.assert(result.data.temperature !== undefined, '应该有温度数据');
  
  console.log('✓ 测试通过');
}

test();
```

---

*下一章: [02-custom-skills.md](02-custom-skills.md) - 开发自定义高级技能*
