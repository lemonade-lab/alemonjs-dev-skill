# AlemonJS 架构标准

本文面向通用项目，定义统一分层与执行链，不讨论具体业务代码。

## 标准分层

```text
Application Layer
  response / middleware / image component / format builder

Core Layer
  onProcessor -> expendCycle(create/mount/unmount)
  middlewareRouter -> responseRouter
  subscribe(create/mount/unmount)
  AsyncLocalStorage context

Platform Adapter Layer
  事件标准化（FormatEvent）
  平台动作执行（send/manage/client.api）
```

## 标准生命周期

```text
onProcessor
  -> create   (middleware + subscribe.create)
  -> mount    (response + subscribe.mount)
  -> unmount  (subscribe.unmount)
```

## 标准职责边界

- 中间件：准入与放行（鉴权、频控、日志）
- 响应：业务处理与消息输出
- 订阅：跨消息状态等待与收敛
- 适配器：平台差异与字段映射

## 标准执行约定

- `middlewareRouter` 先于 `responseRouter`
- `return true` 继续链路，`void/false` 终止链路
- hooks 默认无参调用，事件上下文由 AsyncLocalStorage 注入

## 标准工程建议

- 单 handler 单职责
- 业务层不直接耦合平台 SDK
- 复杂流程拆为多个路由节点而非巨型函数
- 平台特有逻辑集中在适配器层

## 相关文档

- 路由标准：[routing.md](./routing.md)
- Hook 标准：[hooks.md](./hooks.md)
- 事件标准：[events.md](./events.md)
