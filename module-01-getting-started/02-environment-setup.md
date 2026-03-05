# 1.2 环境配置详解

## 目录
- [本地开发环境](#本地开发环境)
- [Docker部署](#docker部署)
- [云服务器配置](#云服务器配置)
- [IDE推荐与配置](#ide推荐与配置)

---

## 本地开发环境

### macOS 安装

```bash
# 使用 Homebrew
brew install node@20

# 安装 OpenClaw
npm install -g openclaw-cn

# 创建项目
mkdir ~/openclaw-projects
cd ~/openclaw-projects
openclaw init my-agent
```

### Linux (Ubuntu/Debian) 安装

```bash
# 安装 Node.js
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# 验证
node --version  # v20.x.x
npm --version   # 10.x.x

# 安装 OpenClaw
sudo npm install -g openclaw-cn
```

### Windows 安装

```powershell
# 使用 Chocolatey
choco install nodejs

# 或使用官方安装包从 nodejs.org 下载

# 安装 OpenClaw
npm install -g openclaw-cn
```

---

## Docker部署

### Dockerfile示例

```dockerfile
FROM node:20-alpine

WORKDIR /app

# 安装 OpenClaw
RUN npm install -g openclaw-cn

# 复制项目文件
COPY . .

# 安装依赖
RUN npm install

# 暴露端口（如果需要Web界面）
EXPOSE 3000

CMD ["openclaw", "start"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  openclaw-agent:
    build: .
    container_name: my-agent
    volumes:
      - ./data:/app/data
      - ./memory:/app/memory
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - NODE_ENV=production
    restart: unless-stopped
```

### 运行

```bash
# 构建并启动
docker-compose up -d

# 查看日志
docker-compose logs -f

# 进入容器
docker exec -it my-agent sh
```

---

## 云服务器配置

### 推荐配置

| 用途 | CPU | 内存 | 存储 | 预估月费 |
|------|-----|------|------|----------|
| 个人开发 | 2核 | 4GB | 50GB SSD | ¥50-100 |
| 小型生产 | 4核 | 8GB | 100GB SSD | ¥150-300 |
| 企业级 | 8核+ | 16GB+ | 500GB+ SSD | ¥500+ |

### 阿里云 ECS 快速部署

```bash
# 1. 购买ECS实例（推荐 Ubuntu 22.04）
# 2. SSH登录
ssh root@your-server-ip

# 3. 安装基础软件
apt update && apt upgrade -y
apt install -y git curl vim

# 4. 安装 Node.js
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs

# 5. 安装 PM2（进程管理）
npm install -g pm2

# 6. 安装 OpenClaw
npm install -g openclaw-cn

# 7. 初始化项目
mkdir -p /opt/openclaw
cd /opt/openclaw
openclaw init production-agent

# 8. 使用 PM2 启动
cd production-agent
pm2 start openclaw --name "agent"
pm2 save
pm2 startup
```

---

## IDE推荐与配置

### VS Code (强烈推荐)

#### 必装插件
- **ESLint** - 代码规范检查
- **Prettier** - 代码格式化
- **GitLens** - Git增强
- **Markdown All in One** - Markdown支持
- **YAML** - 配置文件高亮

#### 工作区配置 `.vscode/settings.json`

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "eslint.validate": ["javascript", "typescript"],
  "files.exclude": {
    "node_modules": true,
    ".git": true
  },
  "search.exclude": {
    "node_modules": true
  }
}
```

#### 调试配置 `.vscode/launch.json`

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Agent",
      "program": "${workspaceFolder}/src/index.js",
      "env": {
        "NODE_ENV": "development"
      }
    }
  ]
}
```

### Cursor (AI辅助编程)

特别适合 OpenClaw 开发，因为：
- 内置 AI 代码补全
- 支持自然语言生成代码
- 可以理解 OpenClaw 的 API 模式

配置提示词 `.cursorrules`:

```
You are helping develop an OpenClaw agent.
Key concepts:
- Agents use skills to perform tasks
- Skills are reusable modules
- Memory persists across sessions
- Always handle errors gracefully
```

---

## 环境变量管理

### 开发环境 `.env.development`

```bash
# LLM配置
OPENAI_API_KEY=sk-your-key-here
OPENAI_MODEL=gpt-4

# 调试模式
DEBUG=true
LOG_LEVEL=debug

# 本地存储
MEMORY_PATH=./memory
DATA_PATH=./data
```

### 生产环境 `.env.production`

```bash
# LLM配置
OPENAI_API_KEY=sk-prod-key-here
OPENAI_MODEL=gpt-4-turbo

# 生产优化
DEBUG=false
LOG_LEVEL=warn

# 持久化存储
MEMORY_PATH=/var/lib/openclaw/memory
DATA_PATH=/var/lib/openclaw/data
```

### 加载环境变量

```javascript
require('dotenv').config({
  path: process.env.NODE_ENV === 'production' 
    ? '.env.production' 
    : '.env.development'
});
```

---

## 验证安装

运行诊断命令：

```bash
openclaw doctor
```

预期输出：
```
✓ Node.js version: v20.11.0
✓ npm version: 10.2.4
✓ OpenClaw CLI: 2.5.1
✓ Git installed
✓ Default config valid
✓ Can connect to OpenAI API

All checks passed! 🎉
```

---

## 常见问题

### Q: 安装时权限错误？
```bash
# 使用 npx 避免全局安装权限问题
npx openclaw-cn init my-agent
```

### Q: 如何切换 Node.js 版本？
```bash
# 使用 nvm
nvm install 20
nvm use 20
nvm alias default 20
```

### Q: 国内网络慢？
```bash
# 切换 npm 镜像
npm config set registry https://registry.npmmirror.com
```

---

环境配置完成！接下来进入 [03-first-agent.md](03-first-agent.md)，创建你的第一个真正有用的 Agent。
