# ALemonJS Dev Skill

ALemonJS 跨平台聊天机器人开发的 AI 编程助手 Skill，为 AI 编码工具提供 ALemonJS 框架的完整开发知识，包括路由、Hooks、消息格式、事件系统、JSX 卡片渲染，以及 Node.js 后端通用工程常识。

支持 GitHub Copilot、Cursor、Windsurf 等主流 AI 编程助手。

## 安装

### GitHub Copilot

将此仓库克隆到项目的 `.github/skills/` 目录下：

```bash
git clone --depth=1 https://github.com/lemonade-lab/alemonjs-dev-skill .github/skills/alemonjs-dev
```

或使用 Git Submodule 跟踪上游更新：

```bash
git submodule add https://github.com/lemonade-lab/alemonjs-dev-skill .github/skills/alemonjs-dev
```

### Cursor

将此仓库克隆到项目的 `.cursor/skills/` 目录下：

```bash
git clone --depth=1 https://github.com/lemonade-lab/alemonjs-dev-skill .cursor/skills/alemonjs-dev
```

或在 `.cursor/rules` 中引用 SKILL.md 内容。

### Windsurf

将此仓库克隆到项目根目录并在 `.windsurfrules` 中引用：

```bash
git clone --depth=1 https://github.com/lemonade-lab/alemonjs-dev-skill .windsurf/skills/alemonjs-dev
```

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
