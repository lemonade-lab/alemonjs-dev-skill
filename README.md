# ALemonJS Dev Skill

ALemonJS 跨平台聊天机器人开发的 AI 编程助手 Skill，为 AI 编码工具提供 ALemonJS 框架的完整开发知识，包括路由、Hooks、消息格式、事件系统、JSX 卡片渲染，以及 Node.js 后端通用工程常识。

支持 GitHub Copilot、Cursor、Windsurf 等主流 AI 编程助手。

## 安装

### 克隆

```bash
git clone git clone --depth=1 https://github.com/lemonade-lab/alemonjs-dev-skill ./.agents/skills
```

### 连接到各个Agent

可通过link，以copilot为例

```bash
sudo ln -s "$PWD/.agents/skills" "$PWD/.github/skills"
```
  
- GitHub Copilot

.github/skills

- Claude Code

.claude/skills

- Cursor
  
.cursor/skills/

- Windsurf

.windsurf/skills/

- trae

.trae/skills

- codex

.codex/skills


## 快速开始

```bash
npm create alemonjs@latest -y
cd alemonjs
npm install yarn -g --registry=https://registry.npmmirror.com
yarn install
```

- 升级相关依赖到最新

```bash
npm install alemonjs -g --registry=https://registry.npmmirror.com
alemonc upgrade
```
