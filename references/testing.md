# AlemonJS 应用离线测试

`alemonjs/testing` 从 2.1.112 起可用。在测试实例内使用真实路由、Hook 与消息构造，平台动作由实例接管。数据库、外部 HTTP 和业务时钟按项目现有方式注入测试依赖。

```ts
import assert from 'node:assert/strict';
import { test } from 'node:test';
import { Format, Router, useMessage } from 'alemonjs';
import { createTestApp } from 'alemonjs/testing';

test('向原群回复', async t => {
  const app = createTestApp();
  t.after(() => app.dispose());

  const router = Router.create({ events: ['message.create'] });
  router.res({ regular: /^\/hello$/ }, async () => async () => {
    const [message] = useMessage();
    await message.send({ format: Format.create().addText('你好') });
  });
  await app.mount({ responseRouter: router.define });

  const result = await app.dispatch(app.events.groupMessage({ text: '/hello', groupId: 'group-1' }));
  assert.equal(result.messages[0]?.text, '你好');
  assert.equal(result.messages[0]?.target?.targetId, 'group-1');
});
```

按项目现有测试命令运行。使用 Node.js 22 和 `tsx` 时可执行 `node --import tsx --test test/*.test.ts`。

- `runHandler(handler, event)` 验证单个处理器。`mount()` 加 `dispatch()` 验证路由、中间件、去重和会话状态。
- `app.actions.on()` 和 `app.apis.on()` 模拟动作和平台 API，处理器返回 `Result[]`。默认模拟发送成功，其他未模拟调用会报错。
- `result.messages` 记录发送尝试，也包括失败。通过 `result.actions[message.actionIndex]` 检查返回码或异常，通过 `nodes` 检查结构化节点。
- `clock.advance(ms)` 推进框架计时器；业务原生计时器和第三方库需要自己的测试时钟。结束后等待 `dispose()`。

[测试 API 与虚拟时钟参考](https://github.com/lemonade-lab/alemonjs/blob/3b582bf5abfc5af1397274f8d47f5661b4b8be3a/packages/alemonjs/docs/testing.md)
