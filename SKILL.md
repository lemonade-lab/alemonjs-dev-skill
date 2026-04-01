---
name: alemonjs-dev-skill
description: 'AlemonJS 通用开发技能。Use when: 新建或维护大多数 AlemonJS 项目、统一 handler/router/hooks 写法、快速落地消息/图片/中间件/订阅能力、按标准结构组织工程、接入 lvyjs 构建链路。'
argument-hint: '描述目标功能与场景，如 "加一个可扩展的签到命令" 或 "统一项目路由和中间件写法"'
---

# AlemonJS 通用项目技能

目标：面向大部分需求与项目，默认采用统一标准写法，优先可维护、可扩展、可迁移。

## 快速导航

| 主题 | 文件 | 用途 |
| --- | --- | --- |
| 架构标准 | [references/architecture.md](./references/architecture.md) | 生命周期、执行链、项目分层 |
| 路由标准 | [references/routing.md](./references/routing.md) | defineRouter 规则与匹配顺序 |
| Hook 标准 | [references/hooks.md](./references/hooks.md) | 17 hooks 用法与职责边界 |
| 消息标准 | [references/message-format.md](./references/message-format.md) | Format/Markdown/Button 统一写法 |
| 事件标准 | [references/events.md](./references/events.md) | EventKeys 与 FormatEvent 构建 |
| 构建标准 | [references/lvyjs-dev.md](./references/lvyjs-dev.md) | lvyjs 开发/构建/别名/资源 |
| 图片标准 | [references/jsxp-dev.md](./references/jsxp-dev.md) | jsxp 组件与渲染链路 |

## 统一标准写法（默认）

### 1. 目录标准

```text
src/
  index.ts              # defineChildren + register
  response/*.ts         # 响应处理器（一个文件一个能力）
  middleware/*.ts       # 中间件处理器（一个文件一个能力）
  image/component/*.tsx # 图片组件
  assets/*              # CSS/图片/字体等静态资源
```

### 2. handler 标准

- 统一签名：`export default async () => {}`
- 统一开头：`useEvent` 做过滤守卫
- 不匹配必须：`next(); return;`
- 统一返回：`true` 继续链；`void/false` 终止链

```typescript
import { useEvent, useMessage, Format } from 'alemonjs';

export default async () => {
  const [event, next] = useEvent({ selects: ['message.create'] });
  if (!event.match.selects) {
    next();
    return;
  }

  const [message] = useMessage();
  await message.send({ format: Format.create().addText('ok') });
};
```

### 3. router 标准

- `exact` 优先，`prefix` 次之，`regular` 最后
- handler 必须 `lazy(() => import(...))`
- 中间件与响应分离注册

```typescript
import { defineChildren, defineRouter, lazy } from 'alemonjs';

const middlewareRouter = defineRouter([
  { prefix: '/', selects: ['message.create'], handler: lazy(() => import('./middleware/auth')) }
]);

const responseRouter = defineRouter([
  { exact: '/ping', selects: ['message.create'], handler: lazy(() => import('./response/ping')) }
]);

export default defineChildren({
  register() {
    return { middlewareRouter, responseRouter };
  }
});
```

### 4. 消息标准

- 默认使用 `Format.create()` 构建消息
- 按钮只通过 `FormatButtonGroup`
- 链接只通过 `FormatMarkDown.addLink(text, url)`

```typescript
import { Format, FormatButtonGroup, FormatMarkDown } from 'alemonjs';

const btn = new FormatButtonGroup();
btn.addRow().addButton('确认', 'ok');

const md = new FormatMarkDown().addText('说明').addLink('文档', 'https://example.com');

const format = Format.create().addText('标题').addButtonGroup(btn).addMarkdown(md);
```

### 5. 构建标准

- 开发入口：`app.ts`
- 生产入口：`index.js`
- 别名：统一 `@src`
- 资源导入返回文件绝对路径

## 适配大多数项目的三层能力模型

- 基础层：命令响应、参数解析、文本/图片回复
- 业务层：中间件鉴权、订阅交互、管理类 hooks
- 平台层：FormatEvent 适配器扩展、平台 API 透传

建议默认从基础层开始，按需启用业务层与平台层。

## 默认开发流程

1. 定义能力边界：一个 handler 只做一件事
2. 写 `response/*.ts`：先过滤后执行
3. 在 `src/index.ts` 注册路由
4. 需要鉴权时加 `middleware/*.ts`
5. 需要交互状态时加 `useSubscribe`
6. 需要图片输出时接入 jsxp

## 常用命令

```bash
npm create alemonjs
npm run dev
npm run view
npm run build
npm run start
```

## 通用约束（强约定）

- 不在 handler 顶层写平台分支逻辑，平台差异放适配器层
- 不把路由匹配写进业务逻辑，统一放 `useEvent` 守卫
- 不把消息拼装散落在多处，统一 `Format` 链式构建
- 不让单个 handler 过长，超过 80 行应拆分

## 迁移与兼容

- `createEvent` 迁移到 `useEvent`
- `Format.addLink` 迁移到 `FormatMarkDown.addLink`
- hooks 默认无参调用，旧写法传 event 仍兼容
