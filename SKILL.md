---
name: alemonjs-dev-skill
description: 'AlemonJS 通用开发技能。Use when: 新建或维护大多数 AlemonJS 项目、统一 Router DSL / handler / hooks 写法、快速落地消息/图片/中间件/订阅能力、按标准结构组织工程、接入 lvyjs 构建链路。'
argument-hint: '描述目标功能与场景，如 "加一个可扩展的签到命令" 或 "统一项目路由和中间件写法"'
---

# AlemonJS 通用项目技能

目标：面向大部分需求与项目，默认采用统一标准写法，优先可维护、可扩展、可迁移。

当前推荐写法基于 `Router.create().group().use()`，最终通过 `router.define` 挂到 `defineChildren().register()`。
同时保留旧版 `defineRouter([...])` 参考，用于维护旧项目和迁移场景。

## 快速导航

| 主题 | 文件 | 用途 |
| --- | --- | --- |
| 架构标准 | [references/architecture.md](./references/architecture.md) | 生命周期、执行链、项目分层 |
| 路由标准 | [references/routing.md](./references/routing.md) | Router DSL、scope、参数校验、fallback |
| Hook 标准 | [references/hooks.md](./references/hooks.md) | 常用 hooks 与 `event.current.__route` 上下文 |
| 消息标准 | [references/message-format.md](./references/message-format.md) | Format/Markdown/Button 统一写法 |
| 事件标准 | [references/events.md](./references/events.md) | EventKeys、通用事件字段、路由上下文 |
| 构建标准 | [references/lvyjs-dev.md](./references/lvyjs-dev.md) | lvyjs 开发/构建/别名/资源 |
| 图片标准 | [references/jsxp-dev.md](./references/jsxp-dev.md) | jsxp 组件与渲染链路 |
| 运行时标准 | [references/app-runtime.md](./references/app-runtime.md) | 配置读取、定时任务、热重载清理 |
| 后端通用 | [references/nodejs-backend.md](./references/nodejs-backend.md) | 时间、校验、错误、配置、并发与日志 |

## 统一标准写法（默认）

### 1. 目录标准

```text
src/
  index.ts              # Router.create + defineChildren register
  response/*.ts         # 响应处理器（一个文件一个能力）
  middleware/*.ts       # 复用中间件（鉴权、频控、日志等）
  image/component/*.tsx # 图片组件
  assets/*              # CSS/图片/字体等静态资源
```

### 2. index/router 标准

- 统一入口：`const router = Router.create({ events })`
- 顶层前置：`router.res(...)` 只放维护模式、统一引导、输入预处理这类入口逻辑
- 统一分组：`router.group({ routeText })`
- 统一注册：`group.use('cmd', () => import('./response/cmd'))`
- 统一导出：`responseRouter: router.define`
- 能在路由层表达的规则，不要下沉到 handler 重复判断

```typescript
import { Router, defineChildren, logger } from 'alemonjs';
import expose from './expose';

const router = Router.create({
  events: ['message.create', 'private.message.create', 'interaction.create', 'private.interaction.create']
});

const appGroup = router.group({
  routeText: {
    prefixes: ['/', '#', '＃', '!', '！'],
    stripPrefix: true,
    allowBare: true
  }
});

appGroup.use('help', () => import('./response/help'));

export default defineChildren({
  register() {
    return {
      responseRouter: router.define,
      expose
    };
  },
  onCreated() {
    logger.info('本地测试启动');
  }
});
```

### 3. handler 标准

- 统一签名：`export default async () => {}` 或 `export default async (_event, _next) => {}`
- 路由命中后默认不需要再手动判断命令名
- 需要放行后续 importer 时调用 `await next()`
- 默认返回 `void`；明确终止可返回 `false`
- handler 内部读取当前事件时，优先 `const [event, next] = useEvent()` 安全获取上下文
- 参数与命令上下文优先从 `event.current.__route` 读取

```typescript
import { useEvent, useMessage, Format } from 'alemonjs';

export default () => {
  const [event] = useEvent();
  const [message] = useMessage();
  const name = String(event.current.__route?.params?.name ?? 'world');

  await message.send({
    format: Format.create().addText(`hello ${name}`)
  });
};
```

### 4. 路由注册标准

- 命令优先使用 `group.use('path', importer)`
- 多词命令直接写空格路径，如 `group.use('admin ban', importer)`
- 需要参数时使用对象配置并声明 `schema`
- 需要共用鉴权或日志时，把 middleware importer 作为 `group(...middlewares)` 或 `use(...importers)` 的前置 importer
- 纯事件型兜底逻辑才使用 `router.res(...)`

```typescript
const appGroup = router.group(
  {
    routeText: {
      prefixes: ['/', '#', '＃', '!', '！'],
      stripPrefix: true,
      allowBare: true
    },
    fallback: {
      suggest: true,
      allowPrefixMatch: true
    }
  },
  () => import('./middleware/auth')
);

appGroup.use(
  {
    path: 'user info',
    schema: {
      usage: 'user info <uid>',
      args: [{ name: 'uid', rules: [{ required: true }] }]
    }
  },
  () => import('./response/user-info')
);
```

### 5. 消息标准

- 默认使用 `Format.create()` 构建消息
- `Format` 是统一消息容器，文本、Markdown、按钮、媒体优先都挂到同一个 `format` 上
- 按钮推荐写法以官网文档为准：`Format.create().addButtonGroup(Format.createButtonGroup()...)`
- Markdown 推荐入口是 `Format.createMarkdown()`

```typescript
import { Format } from 'alemonjs';

const md = Format.createMarkdown().addText('说明');
const bt = Format.createButtonGroup().addRow().addButton('确认', 'ok');

const format = Format.create()
  .addMarkdown(md)
  .addButtonGroup(bt);
```

### 6. 构建标准

- 开发入口：`app.ts`
- 生产入口：`index.js`
- 别名：统一 `@src`
- 资源导入返回的是文件绝对路径

## 旧版写法保留策略

- 新项目默认使用 `Router.create().group().use()`
- 旧项目如果已经大量使用 `defineRouter([...])`，可以继续按原结构维护
- 迁移旧项目时优先做增量迁移，不要求一次性全部改完
- 在同一个文档里保留旧版示例，但要明确它是“兼容写法”而不是当前首选

旧版典型结构：

```typescript
import { defineChildren, defineRouter, lazy } from 'alemonjs';

const middlewareRouter = defineRouter([
  {
    prefix: '/',
    selects: ['message.create', 'private.message.create', 'interaction.create', 'private.interaction.create'],
    handler: lazy(() => import('./middleware/auth'))
  }
]);

const responseRouter = defineRouter([
  {
    exact: '/ping',
    selects: ['message.create', 'private.message.create', 'interaction.create', 'private.interaction.create'],
    handler: lazy(() => import('./response/ping'))
  }
]);

export default defineChildren({
  register() {
    return { middlewareRouter, responseRouter };
  }
});
```

## 适配大多数项目的三层能力模型

- 基础层：命令响应、参数解析、文本/图片回复
- 业务层：scope 中间件、订阅交互、管理类 hooks
- 平台层：FormatEvent 适配器扩展、平台 API 透传

建议默认从基础层开始，按需启用业务层与平台层。

## 默认开发流程

1. 定义能力边界：一个 handler 只做一件事
2. 在 `src/index.ts` 创建 `Router.create({ events })`
3. 用 `router.res(...)` 放顶层前置逻辑，不在这里注册具体命令
4. 用 `group({ routeText })` 定义命令作用域、共享中间件与前缀规则
5. 用 `group.use(...)` 注册命令，参数写进 `schema`
6. 需要鉴权或日志时，把 middleware 作为 importer 串到 group 或 route
7. 需要交互状态时加 `useSubscribe`
8. 需要图片输出时接入 jsxp

## 常用命令

```bash
npm create alemonjs
npm run dev
npm run view
npm run build
```

## 通用约束

- 不在 handler 顶层重复做命令匹配，命中交给 Router DSL
- 不在 handler 顶层写平台分支逻辑，平台差异放适配器层
- 不把消息拼装散落在多处，统一 `Format` 链式构建
- 命令参数优先走 `schema` 校验，不手写零散字符串判断
- 业务内部读取当前事件时，默认优先 `useEvent()`，不要直接依赖 handler 形参上的 `event`
- 顶层前置逻辑放 `router.res(...)`，共享规则放 `group(...)`，具体命令放 `group.use(...)`
- 时间逻辑优先使用 `dayjs` 统一处理，先定义时区与边界，再写业务判断
- 外部输入默认不可信，参数、环境变量、第三方响应都要显式校验和收敛类型
- 预期错误与系统错误分开处理，给用户的文案和给日志的上下文分开设计
- 外部 IO 默认加超时和失败预期，涉及重复事件、限额、签到、回调时优先考虑幂等

## 迁移与兼容

- 旧 `defineRouter([...])` 业务写法迁移到 `Router.create().group().use()`
- `defineRouter` 仍是底层导出结构，但不再作为业务层推荐入口
- 旧项目继续保留 `middlewareRouter + responseRouter` 结构是可接受的
- `createEvent` 迁移到 `useEvent`
- hooks 默认无参调用，旧写法传 event 仍兼容

## 需要 Node.js 后端常识时

- 时间、时区、冷却、过期、每日刷新：读取 [references/nodejs-backend.md](./references/nodejs-backend.md) 的“时间与时区”
- 输入校验、schema 边界、环境变量收敛：读取 [references/nodejs-backend.md](./references/nodejs-backend.md) 的“输入校验与类型收敛”“配置与环境变量”
- 错误处理、日志、外部请求、重试：读取 [references/nodejs-backend.md](./references/nodejs-backend.md) 的“错误处理”“日志与可观测性”“异步 IO 与外部依赖”
- 签到、次数、库存、订阅状态、重复回调：读取 [references/nodejs-backend.md](./references/nodejs-backend.md) 的“幂等、并发与状态一致性”

## 需要运行时能力时

- 初始化、挂载、异常卸载：读取 [references/architecture.md](./references/architecture.md) 的“应用生命周期钩子”
- 配置读取、配置监听、配置写回：读取 [references/app-runtime.md](./references/app-runtime.md) 的“配置读取”
- 定时任务、cron、暂停恢复、自动清理：读取 [references/app-runtime.md](./references/app-runtime.md) 的“框架托管定时任务”

## 开发结束根据当前环境尝试进行检查

- ts check

```ts
npm install -g @typescript/native-preview
tsgo
```

- lint check

```bash
npm install -g eslint
eslint src --ext .ts,.tsx,.js --fix --max-warnings=0
```
