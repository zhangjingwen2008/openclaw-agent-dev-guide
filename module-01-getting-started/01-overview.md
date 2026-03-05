# 模块1: 环境搭建与初识

## 1.1 OpenClaw 是什么？

OpenClaw 是一个开源的 AI Agent 框架，让你能够：
- 🤖 构建自主运行的智能代理
- 🔧 通过 Skills 扩展代理能力
- 👥 让多个代理协作完成任务
- 🌐 集成各种外部服务和API

### 核心设计理念

```
Agent = LLM + Memory + Tools + Planning
```

OpenClaw 将这些要素封装成简洁的接口，让你专注于业务逻辑而非底层实现。

## 1.2 架构概览

```
┌─────────────────────────────────────────┐
│           Your Application              │
├─────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │ Agent 1 │  │ Agent 2 │  │ Agent N │ │
│  └────┬────┘  └────┬────┘  └────┬────┘ │
│       └─────────────┼─────────────┘     │
│                     ▼                   │
│            ┌─────────────┐              │
│            │ Skill Store │              │
│            └──────┬──────┘              │
│                   │                     │
│       ┌───────────┼───────────┐         │
│       ▼           ▼           ▼         │
│   ┌───────┐   ┌───────┐   ┌───────┐    │
│   │Search │   │Code   │   │Browser│    │
│   │Skill  │   │Skill  │   │Skill  │    │
│   └───────┘   └───────┘   └───────┘    │
└─────────────────────────────────────────┘
```

## 1.3 安装与配置

### 系统要求
- Node.js 18+ 
- Git
- 至少 4GB RAM

### 安装步骤

```bash
# 1. 安装 OpenClaw CLI
npm install -g openclaw-cn

# 2. 验证安装
openclaw --version

# 3. 初始化工作空间
openclaw init my-first-agent
cd my-first-agent
```

### 配置文件详解

`openclaw.config.yaml`:

```yaml
agent:
  name: "my-assistant"
  model: "gpt-4"
  systemPrompt: |
    You are a helpful assistant.

skills:
  - name: "web-search"
    enabled: true
  - name: "code-execution"
    enabled: true

memory:
  type: "local"
  path: "./memory"
```

## 1.4 第一个 "Hello World" Agent

创建 `hello-world.js`:

```javascript
const { Agent } = require('openclaw');

const agent = new Agent({
  name: 'greeter',
  systemPrompt: 'You are a friendly greeting bot.'
});

async function main() {
  const response = await agent.run('Say hello to the world!');
  console.log(response);
}

main();
```

运行:
```bash
node hello-world.js
```

预期输出:
```
Hello, World! 👋 I'm excited to help you on your journey with OpenClaw!
```

## 1.5 核心概念解析

### Agent (代理)
代理是执行任务的主体，包含：
- **LLM**: 大脑，负责推理和决策
- **Memory**: 记忆，保存上下文和历史
- **Skills**: 技能，扩展能力边界

### Skill (技能)
技能是可复用的功能模块：
- 内置技能: Search, Code, Browser, File, etc.
- 自定义技能: 你可以开发自己的技能

### Session (会话)
会话管理一次完整的交互：
- 维护对话上下文
- 追踪工具调用
- 记录执行历史

## 下一步

进入 [02-environment-setup.md](02-environment-setup.md)，我们将详细配置开发环境。

---

*本章节完成度: 80% | 预计完成时间: 30分钟*
