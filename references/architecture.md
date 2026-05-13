# AlemonJS 架构标准

本文面向通用项目，定义统一分层与执行链，不讨论具体业务代码。

## 标准分层

```text
Application Layer
  index(router create/group/use) / response / middleware / image component / format builder

Core Layer
  onProcessor -> expendCycle(create/mount/unmount)
  router.define -> scope dispatch -> importer chain
  subscribe(create/mount/unmount)
  AsyncLocalStorage context

Platform Adapter Layer
  事件标准化（FormatEvent）
  平台动作执行（send/manage/client.api）
```

## 标准生命周期

```text
onProcessor
  -> create   (router scope + subscribe.create)
  -> mount    (route dispatch + subscribe.mount)
  -> unmount  (subscribe.unmount)
```

## 应用生命周期钩子

`defineChildren(...)` 除了注册路由，还承载应用初始化和异常卸载时机。

```typescript
export default defineChildren({
  onCreated() {
    // 初始化阶段
  },
  onMounted(store) {
    // 路由与中间件索引已建立
  },
  unMounted(error) {
    // 异常卸载
  }
});
```

推荐职责：

- `onCreated`：放初始化动作、启动前检查、可能阻塞运行的预处理
- `onMounted`：放索引建立后的观察、诊断、注册后检查
- `unMounted(error)`：放异常清理、告警、兜底日志

避免：

- 在 handler 里做一次性初始化
- 在 `onCreated` 里塞入和业务消息处理耦合的命令逻辑
- 用原生全局状态替代可控的生命周期钩子

## 标准职责边界

- index/router：声明事件范围、scope、命令路径与 fallback
- 中间件 importer：准入与放行（鉴权、频控、日志）
- 响应 importer：业务处理与消息输出
- 订阅：跨消息状态等待与收敛
- 适配器：平台差异与字段映射

## 标准执行约定

- Router DSL 先做 scope 命令匹配，再执行 importer 链
- `await next()` 继续链路，`false` 终止链路，`void` 表示当前节点处理完成
- hooks 默认无参调用，事件上下文由 AsyncLocalStorage 注入
- 命中结果会写入底层事件对象，业务通过 `useEvent()` 读取时，优先从 `event.current.__route` 访问

## 标准工程建议

- 单 handler 单职责
- 路由规则集中在 `src/index.ts`
- 业务层不直接耦合平台 SDK
- 复杂流程拆为多个 route/importer，而非巨型函数
- 平台特有逻辑集中在适配器层

## 相关文档

- 路由标准：[routing.md](./routing.md)
- Hook 标准：[hooks.md](./hooks.md)
- 事件标准：[events.md](./events.md)
