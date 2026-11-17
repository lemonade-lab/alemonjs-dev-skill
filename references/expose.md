# Expose 插件能力

同一个 AlemonJS 运行实例内，应用通过协议、提供者和动作调用共享业务能力。为聊天指令保留 Router 入口，为插件调用注册明确的数据契约和授权。

## 注册结构化动作

在 `counter-plugin` 的应用入口注册只读动作。

```ts
import { defineChildren, Expose } from 'alemonjs';

const expose = Expose.create().provide({
  name: 'example.counter',
  version: '1',
  description: '读取计数',
  actions: {
    read: Expose.action<null, { count: number }>({
      description: '返回当前计数',
      effect: 'read',
      inputSchema: { type: 'null' },
      outputSchema: {
        type: 'object',
        properties: { count: { type: 'integer', minimum: 0 } },
        required: ['count'],
        additionalProperties: false
      },
      authorize: (_input, context) => context.callerApp === 'report-plugin',
      handler: () => ({ count: 0 })
    })
  }
});

export default defineChildren({
  register() {
    return { expose };
  }
});
```

对象契约要求明确写出 `additionalProperties: false`。使用框架支持的 JSON Schema 子集，避免使用 `$ref`、组合类型和 `pattern`。涉及玩家权益的动作，由提供者另外校验具体玩家、渠道及授权范围。

## 调用、失败和清理

- 用 `Expose.create().schema()` 查看各提供者的契约，用 `client.call('example.counter', 'counter-plugin', 'read', null, { timeoutMs: 5000 })` 调用。在示例中需从 `report-plugin` 的事件处理器或生命周期调用。
- `call()` 要求输入输出契约完整。成功结果的 `data` 是通过输出校验的 `unknown`，按对应契约读取。旧动作可继续使用 `invoke()`。
- 错误结果的 `outcome: 'not_started'` 表示处理器尚未开始。`'unknown'` 表示已经开始，应查询业务结果或使用幂等标识。超时或中止通过 `context.signal` 通知处理器。
- `watch()` 在动作成功且通过输出校验后通知，跳过标为 `effect: 'read'` 的动作。应用上下文内的订阅随卸载清理。缓存的提供者句柄在卸载或重载后失效。

## 后台调用

在所属应用的事件处理器或生命周期中用 `createBackgroundRunner(callback)` 封装函数。之后调用返回函数时，框架保留应用身份，并清除玩家事件、事件链和父 Expose 调用上下文。应用负责调度、持久化和业务授权。需要代表玩家执行动作时，把经验证的授权范围和业务操作 ID 显式传给提供者。

[完整接口参考](https://github.com/lemonade-lab/alemonjs/blob/3b582bf5abfc5af1397274f8d47f5661b4b8be3a/packages/alemonjs/docs/expose.md)
