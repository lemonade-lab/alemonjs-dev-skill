# AlemonJS 路由参考

本文覆盖当前推荐的 Router DSL，同时保留旧版 `defineRouter` 兼容写法，不重复 hooks 和事件字段。

## 推荐入口

```typescript
import { Router, defineChildren } from 'alemonjs';

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

appGroup.use('hello', () => import('./response/hello'));

export default defineChildren({
  register() {
    return {
      responseRouter: router.define
    };
  }
});
```

## 核心模型

- `Router.create(options)`：创建路由实例并声明默认 `events/platforms`
- `router.res(config, ...importers)`：注册顶层前置逻辑或纯事件型兜底处理
- `router.group(options, ...middlewares)`：创建 scope，并给 scope 绑定前缀、中间件、fallback 等规则
- `group.use(config, ...importers)`：注册命令路径与执行 importer 链
- `router.define`：把 DSL 转成底层 `defineRouter(...)` 可消费结构

## res / group / use 的职责边界

推荐把这三层严格分开。

- `router.res(...)`：只放顶层前置逻辑，如维护模式、统一引导、输入预处理、全局拦截
- `router.group(...)`：放共享规则，如 `routeText`、共享中间件、fallback、`keyPolicy`、`duplicateKey`
- `group.use(...)`：只注册具体命令和参数 schema

推荐：

```typescript
router.res({}, () => import('./response/res-maintenance'));

const appGroup = router.group(
  {
    routeText: {
      prefixes: ['/', '#', '＃', '!', '！'],
      stripPrefix: true,
      allowBare: true
    }
  },
  () => import('./middleware/auth')
);

appGroup.use('help', () => import('./response/help'));
```

避免：

- 在 `router.res(...)` 里注册具体命令分发
- 在 `group.use(...)` 里重复声明整组共享规则
- 把 scope 级别鉴权、频控散落到每个 handler

## 路由路径

- 命令路径统一用 `path`
- 支持单词或双词命令，如 `hello`、`user info`
- 路径会自动归一化，前缀 `/ # ＃ ! ！` 会被去掉
- 匹配优先使用双词 key，其次单词 key

```typescript
group.use('help', () => import('./response/help'));
group.use('user info', () => import('./response/user-info'));
```

## Scope 与 routeText

`group({ routeText })` 控制当前作用域内命令文本的解析方式。

```typescript
const appGroup = router.group({
  routeText: {
    prefixes: ['/', '#', '＃', '!', '！'],
    stripPrefix: true,
    allowBare: true
  }
});
```

- `prefixes`：允许的命令前缀
- `stripPrefix`：匹配前是否剥离前缀
- `allowBare`：是否允许无前缀文本也进入命令匹配
- `byPlatform`：可按平台覆写前缀规则

补充约束：

- 内部 key 一律不带前缀，前缀归 `routeText` 处理
- 不要把 `/签到`、`#帮助` 这类字符串直接写进 `group.use(...)`

## 中间件链

scope 或 route 上都可以挂 importer，执行顺序是声明顺序。

```typescript
const adminGroup = router.group(
  {
    path: 'admin',
    routeText: { prefixes: ['/'], stripPrefix: true }
  },
  () => import('./middleware/auth')
);

adminGroup.use('ban', () => import('./response/admin-ban'));
```

- importer 模块默认 `export default`
- handler 签名推荐 `export default async (event, next) => {}`
- `await next()`：进入下一个 importer
- 返回 `false`：终止链路
- 不调用 `next` 且返回 `void`：视为当前链路处理完成

补充：

- 顶层 `router.res(...)` 放行依赖 `next()`，不要指望 `return true`
- 共享中间件优先挂在 `group(...)`，而不是复制到每个 `use(...)`

## 参数校验

命令参数优先放在 `schema`，不要在 handler 里手写分散判断。

```typescript
group.use(
  {
    path: 'user info',
    schema: {
      usage: 'user info <uid>',
      args: [
        {
          name: 'uid',
          rules: [{ required: true }]
        }
      ]
    }
  },
  () => import('./response/user-info')
);
```

- 支持 `string`、`number`、`enum`、`range`、`rest`
- 校验失败会自动回复错误与 `usage`
- 解析结果会挂到底层事件对象；业务层推荐通过 `useRoute()` 读取参数，而不是直接依赖内部 `__route` 挂载细节

## 路由上下文

命中后会把当前命令上下文注入到底层事件对象的 `__route`：

```typescript
{
  key: 'user info',
  text: 'user info 10001',
  rawArgs: ['10001'],
  parsedArgs: ['10001'],
  params: { uid: '10001' }
}
```

业务 handler 优先通过 `useRoute()` 获取这里的上下文，而不是重新解析 `MessageText`。

## fallback 建议

未匹配命令时，scope 可以自动给出相近指令提示。

```typescript
router.group({
  routeText: { prefixes: ['/'], stripPrefix: true },
  fallback: {
    suggest: true,
    allowPrefixMatch: true
  }
});
```

## keyPolicy / duplicateKey / redispatch

这些选项适合放在 `group(...)`，用于约束整组命令行为。

- `keyPolicy.maxWords`：控制命令 key 提取词数，常见值是 `2`
- `duplicateKey`：控制重复 key 的处理策略，常见值是 `warn` 或 `throw`
- `redispatch.maxDepth`：控制重新分发深度，避免递归型链路失控

建议：

- 命令以双词为主的项目，显式配 `keyPolicy: { maxWords: 2 }`
- 多人维护的大项目，重复 key 默认不静默吞掉
- 涉及转义、改写命令文本时，再考虑 `redispatch`

## 旧版 defineRouter 写法

旧项目仍可能直接使用 `defineRouter([...])`：

```typescript
import { defineRouter, lazy } from 'alemonjs';

const middlewareRouter = defineRouter([
  {
    prefix: '/',
    selects: ['message.create', 'private.message.create', 'interaction.create', 'private.interaction.create'],
    handler: lazy(() => import('./middleware/auth'))
  }
]);

const responseRouter = defineRouter([
  {
    exact: '/help',
    selects: ['message.create', 'private.message.create', 'interaction.create', 'private.interaction.create'],
    handler: lazy(() => import('./response/help'))
  },
  {
    prefix: '/admin ',
    selects: ['message.create', 'private.message.create', 'interaction.create', 'private.interaction.create'],
    handler: lazy(() => import('./response/admin'))
  },
  {
    regular: /^(\/|#|＃|!|！)roll\s+\d+$/,
    selects: ['message.create', 'private.message.create'],
    handler: lazy(() => import('./response/roll'))
  }
]);
```

旧版模型特点：

- 单节点支持 `exact`、`prefix`、`regular`
- 单节点内部匹配顺序是 `exact -> prefix -> regular`
- 节点之间按数组顺序匹配，先命中先执行
- 常见工程结构是 `middlewareRouter -> responseRouter`

## 兼容说明

- `defineRouter([...])` 仍是底层兼容结构
- 当前业务层不推荐继续手写 `exact/prefix/regular`
- 旧项目迁移时，优先把命令节点改成 `group.use(path, importer)`

## 相关文档

- API 参数与方法：[hooks.md](./hooks.md)
- 事件类型与字段：[events.md](./events.md)
- 框架执行总览：[architecture.md](./architecture.md)
