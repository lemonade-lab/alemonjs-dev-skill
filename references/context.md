# AlemonJS 事件上下文

事件上下文用于多轮对话、分步表单、验证码确认等跨消息流程。状态默认只在内存中保存；重启后需要保留的数据应写入数据库或 Redis。

## 何时使用

- 需要按用户或频道隔离会话、保存步骤状态、设置超时或处理重复打开：使用 `createContext`。
- 只需短暂观察后续事件，或需要 create/mount/unmount 订阅时机：使用 `useSubscribe`。

## 最小模式

```typescript
import { createContext, configureContext, defineChildren, useMessage, Format } from 'alemonjs';

const registerContext = createContext({
  name: 'register',
  events: ['message.create', 'private.message.create'],
  scope: ['UserId'],
  expiresIn: '5m',
  initialState: () => ({ step: 'email', email: '' }),
  handlers: {
    email(event, state) {
      state.email = event.MessageText;
      const [message] = useMessage(event);
      message.send({ format: Format.create().addText('请输入 confirm 确认') });
      registerContext.confirm();
    },
    confirm(event, _state, action) {
      if (event.MessageText === 'confirm') {
        action.close();
        return;
      }
      action.pass(); // 保留上下文，同时让当前事件进入普通路由
    }
  }
});

export default defineChildren({
  register: () => ({
    responseContent: configureContext({ contexts: { register: registerContext } })
  })
});
```

## 关键规则

- `name` 必须唯一；每个 handler 名会成为打开下一步的动作，例如 `registerContext.confirm()`。
- `scope` 决定会话隔离维度；事件缺少所需字段不会命中。
- `expiresIn` 可用毫秒数或 `'5m'` 形式；`conflict` 用 `replace` 或 `reject` 处理同一作用域的重复会话。
- `initialState` 可用对象或工厂函数；`action.payload` 读取打开时的载荷，`action.signal` 感知关闭、替换、过期或卸载。
- 注册到 `middlewareContent` 表示在中间件路由前运行；注册到 `responseContent` 表示在响应路由前运行。同一上下文不能同时注册两处。
- `onError` 使用 `close` 或 `keep` 决定 handler 出错后是否保留上下文。
