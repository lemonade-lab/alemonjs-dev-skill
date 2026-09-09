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

`defineChildren(...)` 除了注册路由，还承载应用初始化、进入可服务状态与统一清理时机。

```typescript
export default defineChildren({
  onCreated() {
    // 轻量初始化
  },
  async onMounted(store) {
    // 兼容：索引建立后、ready 前
  },
  async onReady(store) {
    // 最终依赖检查；抛错则不进入 ready
  },
  async onDispose(error) {
    // 关闭连接、停止轮询、释放资源
  }
});
```

推荐职责：

- `onCreated`：轻量初始化与静态配置读取；不要在模块顶层或这里放重型阻塞逻辑
- `onMounted`：兼容初始化入口，发生在索引建立后、`ready` 前
- `onReady`：新项目的最终依赖校验、ready 日志、依赖应用已可服务后才能启动的后台逻辑
- `onDispose(error)`：新项目的统一清理入口
- `unMounted(error)`：兼容旧卸载写法；新代码使用 `onDispose`

### 事件、HTTP 与状态回调

- `onEventStart({ event, name })`：trace、入口日志等轻量通知
- `onEventError({ event, error, appName, phase })`：处理 `middleware`、`response`、`subscribe`、`route` 阶段错误；仅返回 `'continue'` 才继续当前事件链
- `onEventFinished({ reason, duration, hasSendAttempted, hasSendSucceeded, lastSendError })`：审计、耗时和发送结果收口
- `onHttpError({ ctx, error, appName, path, method, kind })`：HTTP 处理错误；自己写完响应后返回 `'handled'`
- `onRuntimeStatusChange({ appName, previousStatus, status, error })`：观察 `discovered`、`loading`、`ready`、`failed`、`disposed`

这些是通知与生命周期回调，不应当作为新的业务中间件系统。除 `onEventError` 和 `onHttpError` 外，回调不改变框架默认流程。

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

### 自定义平台边界

只有在开发平台适配器时才使用 `definePlatform`、`cbpPlatform`、`FormatEvent` 和 `createResult`：适配器负责把原始平台事件构建成标准事件、执行 CBP action，并将平台结果归一化。普通业务应用继续优先使用通用 hooks；平台 SDK 调用收敛在适配器或独立 service，避免散落进 handler。

## 标准执行约定

- Router DSL 先做 scope 命令匹配，再执行 importer 链
- `await next()` 继续链路，`false` 终止链路，`void` 表示当前节点处理完成
- hooks 默认无参调用，事件上下文由 AsyncLocalStorage 注入
- 命中结果会写入底层事件对象；业务层读取路由上下文时，优先通过 `useRoute()` 获取只读快照

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
