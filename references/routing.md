# AlemonJS 路由参考

本文仅覆盖路由匹配与执行规则，不重复 hooks 和事件字段。

## defineRouter

```typescript
import { defineRouter, lazy } from 'alemonjs';

const router = defineRouter([
  {
    exact?: string,
    prefix?: string,
    regular?: RegExp,
    selects?: EventKeys | EventKeys[],
    handler: lazy(() => import('./response/xxx')),
    children?: ResponseRoute[]
  }
]);
```

## 匹配顺序

单节点内部：
1. `exact`
2. `prefix`
3. `regular`

节点之间：按数组顺序，先匹配先执行。

## 执行顺序

`middlewareRouter -> responseRouter`

中间件约定：
- `return true`：继续到下一个节点
- `return void/false`：终止

## lazy 约束

- handler 必须使用 `lazy(() => import(...))`
- 目标模块必须 `export default`

## 嵌套路由

父节点匹配后再进入 children，常用于：
- 统一鉴权
- 管理命令分组

## 推荐 handler 模式

```typescript
export default async () => {
  const [event, next] = useEvent({ selects: ['message.create'] });
  if (!event.match.selects) {
    next();
    return;
  }
  // business
};
```

## 相关文档

- API 参数与方法：[hooks.md](./hooks.md)
- 事件类型与字段：[events.md](./events.md)
- 框架执行总览：[architecture.md](./architecture.md)
