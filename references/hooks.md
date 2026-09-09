# AlemonJS Hooks 参考

本文聚焦业务可直接调用的 hooks，结合官方 `docs/alemonjsDocs/core/hook.mdx`、`response.md` 与 `data-type.md` 的能力面整理推荐写法。事件字段详见 events，消息结构详见 message-format。

## 使用约定

- 所有 hooks 的 `event` 参数都可选。
- response handler 内默认依赖 AsyncLocalStorage 自动注入上下文，优先无参调用。
- 业务代码内部读取当前事件时，默认优先 `const [event, next] = useEvent()`，不要直接依赖 handler 形参上的 `event`。
- 业务代码内部读取命令上下文时，默认优先 `const [route] = useRoute()`。
- 非 response 上下文、订阅回调、工具函数中需要显式传入 `event`。
- 当前推荐把命令匹配交给 `Router.create().group().use()`，hooks 负责读上下文和执行平台动作。

## 核心入口

### useEvent

`useEvent` 用于安全读取当前事件和 `next` 回调，也兼容旧版 `exact/prefix/regular` 守卫。

```typescript
const [event, next] = useEvent({
  selects: ['message.create', 'private.message.create', 'interaction.create', 'private.interaction.create']
});
```

返回结构：

```typescript
{
  current,
  value,
  match: { selects, exact, prefix, regular }
}
```

字段说明：

- `event.current`：标准化后的当前事件对象，业务优先读这里
- `event.value`：原始平台数据
- `event.match`：当前守卫条件的命中结果

推荐：

- 需要读取 `UserId/GuildId/ChannelId/MessageId` 时，优先从 `event.current` 读取
- 不把 handler 形参当作默认事件入口，业务内部统一先 `useEvent()`
- 需要放行旧链路或订阅链路时，再显式调用 `next()`

### useRoute

`useRoute` 用于安全读取当前命令的路由上下文，是读取 `key/text/params/rawArgs/parsedArgs` 的默认入口。

```typescript
const [route] = useRoute();

if (!route.matched) {
  return;
}

const uid = route.param('uid');
const page = route.param('page');
```

返回结构分两种：

- 未命中：`matched: false`，其余字段为空快照
- 已命中：`matched: true`，可读取 `key`、`text`、`sourceText`、`rewrittenText`、`rawArgs`、`parsedArgs`、`params`

推荐：

- 命令参数优先走 `route.param(name)` 或 `route.params`
- 需要判断某参数是否存在时，用 `route.hasParam(name)`
- 需要比较原始输入和重写后的命令文本时，读 `sourceText` / `rewrittenText`
- 不在业务代码里直接依赖 `event.current.__route`

旧版兼容：

```typescript
const [event, next] = useEvent({
  selects: ['message.create', 'private.message.create', 'interaction.create', 'private.interaction.create'],
  exact: '/cmd',
  prefix: '/cmd ',
  regular: /pattern/
});

if (!event.match.selects || !event.match.regular) {
  next();
  return;
}
```

旧版返回结构通常会包含：

```typescript
{
  current,
  value,
  match: { selects, exact, prefix, regular }
}
```

`next` 周期语义：

- `next()`：当前周期继续后续匹配
- `next(true)`：下一个周期继续
- `next(true, true)`：下下个周期继续

### useMessage

`useMessage` 用于消息发送、编辑、删除、置顶、取详情，是业务里最常用的动作 hook。

```typescript
const [message] = useMessage();

await message.send({ format, replyId? });
await message.edit({ format, messageId? });
await message.delete({ messageId? });
await message.pin({ messageId? });
await message.unpin({ messageId? });
await message.get({ messageId? });
```

常见搭配：

```typescript
import { useRoute, useMessage, Format } from 'alemonjs';

export default async () => {
  const [route] = useRoute();
  const [message] = useMessage();
  const uid = String(route.param('uid') ?? '');

  await message.send({
    format: Format.create().addText(`uid=${uid}`)
  });
};
```

补充约定：

- `send` 可传 `{ format, replyId? }`，也可直接传 `DataEnums[]`
- 消息发送返回通常是 `Result[]`，一个 `format` 可能被平台拆成多次实际 API 调用
- 格式降级和平台兼容由框架负责，业务层只构建 `Format`
- 发送过程会在当前事件上记录 `_sendAttempted`、`_sendSucceeded`、`_lastSendError`

发送状态语义：

- 调用 `message.send(...)` 前，会先把当前事件标记为 `_sendAttempted = true`
- 只要返回结果里存在任意 `ResultCode.Ok`，会标记 `_sendSucceeded = true`，并把 `_lastSendError = null`
- 若返回结果全部失败，或发送过程中抛错，会更新 `_lastSendError`

适合用于：

- 判断某个 handler 是否至少尝试过回复
- 给重试、兜底回复、失败告警提供依据
- 在复杂链路里避免重复发送

### useMention

`useMention` 用于解析当前消息中的 @ 提及数据。

```typescript
const [mention] = useMention();
const one = await mention.findOne();
const many = await mention.find({ IsBot: false });
```

常用过滤项：

- `UserId`
- `UserKey`
- `UserName`
- `IsMaster`
- `IsBot`

说明：

- `find()` 返回所有匹配提及项
- `findOne()` 返回第一个匹配项
- 默认常见场景下会排除 bot，写业务时不要假设一定能拿到用户

### useSubscribe

`useSubscribe` 用于在某个事件周期里持续观察后续事件，适合临时监听、一次性观察和 create/mount/unmount 生命周期订阅。

```typescript
const [subscribe] = useSubscribe(['message.create', 'private.message.create']);

const sub = subscribe.mount(
  async (event, next) => {
    const [message] = useMessage(event);
    const text = event.MessageText;

    if (text === '123456') {
      await message.send({ format: Format.create().addText('密码正确') });
      return;
    }

    await message.send({ format: Format.create().addText('密码不正确') });
    next();
  },
  ['UserId']
);

subscribe.cancel(sub);
```

订阅时机：

- `subscribe.create(callback, keys)`：响应体创建时触发
- `subscribe.mount(callback, keys)`：中间件之后、响应之前触发
- `subscribe.unmount(callback, keys)`：响应之后触发
- `subscribe.cancel(sub)`：取消订阅

`keys` 说明：

- 用于指定事件匹配键，如 `['UserId']`、`['UserId', 'ChannelId']`
- 只有这些字段值相同的事件，才会命中对应订阅回调

订阅回调里的 `next` 语义：

- `next()`：保持订阅
- `next(true)`：保持订阅且传递给下一个订阅
- `next(true, true)`：保持订阅且传递给下一个周期

推荐：

- 订阅一定配超时和取消逻辑，避免悬空状态
- 多轮、分步且需要保存状态的流程优先使用 `createContext`；它提供作用域、状态、冲突和过期控制，见 [context.md](./context.md)
- 订阅回调里如果要发消息或读上下文，显式 `useMessage(event)`

## 管理类 hooks

### useMember

成员管理，常用于查成员、踢人、封禁、禁言、改名片、头衔。

```typescript
const [member] = useMember();

await member.info({ userId: '10001' });
await member.list({ guildId: '20001' });
await member.search({ keyword: '管理', limit: 20 });
await member.kick({ userId: '10001' });
await member.ban({ userId: '10001', reason: '违规', duration: 3600 });
await member.unban({ userId: '10001' });
await member.mute({ userId: '10001', duration: 60 });
await member.admin({ userId: '10001', enable: true });
await member.card({ userId: '10001', card: '新昵称' });
await member.title({ userId: '10001', title: 'VIP', duration: -1 });
```

### useChannel

频道管理。

```typescript
const [channel] = useChannel();

await channel.info({ channelId: '30001' });
await channel.list({ guildId: '20001' });
await channel.create({ name: '新频道', guildId: '20001' });
await channel.update({ channelId: '30001', name: '改名', topic: '新话题' });
await channel.delete({ channelId: '30001' });
```

### useGuild

服务器或公会管理。

```typescript
const [guild] = useGuild();

await guild.info({ guildId: '20001' });
await guild.list();
await guild.update({ guildId: '20001', name: '新名称' });
await guild.leave({ guildId: '20001', isDismiss: false });
await guild.mute({ guildId: '20001', enable: true });
```

### useRole

角色管理。

```typescript
const [role] = useRole();

await role.list({ guildId: '20001' });
await role.create({ name: 'VIP', color: 0xff0000, guildId: '20001' });
await role.update({ roleId: '40001', name: '超级VIP' });
await role.delete({ roleId: '40001' });
await role.assign({ userId: '10001', roleId: '40001' });
await role.remove({ userId: '10001', roleId: '40001' });
```

### usePermission

频道权限管理。

```typescript
const [permission] = usePermission();

await permission.get({ userId: '10001', channelId: '30001' });
await permission.set({ userId: '10001', channelId: '30001', allow: '1', deny: '0' });
```

### useReaction

消息表情回应管理。

```typescript
const [reaction] = useReaction();

await reaction.add({ emojiId: '👍' });
await reaction.remove({ emojiId: '👍', messageId: '50001' });
await reaction.list({ emojiId: '👍', messageId: '50001', limit: 20 });
```

### useAnnounce

频道公告管理。

```typescript
const [announce] = useAnnounce();

await announce.set({ messageId: '50001', channelId: '30001', guildId: '20001' });
await announce.remove({ messageId: '50001', guildId: '20001' });
```

## 媒体与历史

### useMedia

媒体上传与主动发送。

```typescript
const [media] = useMedia();

await media.upload({ type: 'image', url: 'https://example.com/image.png' });
await media.sendChannel({ type: 'image', url: 'https://example.com/image.png', channelId: '30001' });
await media.sendUser({ userId: '10001', type: 'audio', url: 'https://example.com/test.mp3' });
await media.send({
  target: { targetId: '30001', scope: 'channel' },
  type: 'image',
  filePath: '/tmp/image.png'
});
```

`type` 常用值：

- `'image'`
- `'audio'`
- `'video'`
- `'file'`

媒体来源可为 `url`、`data`、`filePath` 或 `fileId`，一次操作只能提供其中一种。调用后通过返回结果的 `code` 判断平台是否成功处理。

### useConnection

查询当前机器人连接状态。

```typescript
const [{ getStatus }] = useConnection();
const result = await getStatus({ BotId: 'bot-id' });
```

### useInteraction

确认按钮、命令等平台交互事件。省略 `InteractionId` 时会从当前事件读取。

```typescript
const [{ ack }] = useInteraction();
const result = await ack({ InteractionId: 'interaction-id', code: 0 });
```

连接和交互能力不被平台支持时通常返回 `Warn`；业务必须检查 `result.code`，而不是只依赖异常控制流。

### useHistory

消息历史查询。

```typescript
const [history] = useHistory();

await history.list({ channelId: '30001', limit: 50, before: '50001' });
```

## 账号与请求

### useMe

当前 Bot 账号信息。

```typescript
const [me] = useMe();

await me.info();
await me.guilds();
await me.threads();
await me.friends();
```

### useUser

用户信息查询。

```typescript
const [user] = useUser();

await user.info({ userId: '10001' });
```

### useRequest

处理好友请求、入群请求等平台侧申请。

```typescript
const [request] = useRequest();

await request.friend({ flag: 'friend-flag', approve: true, remark: '备注' });
await request.guild({ flag: 'guild-flag', subType: 'add', approve: true, reason: '同意入群' });
```

## 透传客户端

`useClient` 用于调用平台特有 API。仅在业务必须依赖平台原生能力、且通用 hooks 不够表达时使用。

```typescript
import { API, platform } from '@alemonjs/qq-bot';
import { useClient, useEvent } from 'alemonjs';

export default () => {
  const [event] = useEvent();

  if (event.current.Platform === platform) {
    const [client] = useClient(API);
    client.usersMe();
  }
};
```

推荐：

- 先判断平台，再调用对应平台 API
- 平台特有逻辑收敛到适配器层或独立 service，不散落在业务 handler

## 主动消息

需要脱离当前响应上下文主动发消息时，可使用 `MessageDirect.create()`。

```typescript
const direct = MessageDirect.create();

await direct.sendToChannel({ SpaceId: '30001', format });
await direct.sendToUser({ OpenId: '10001', format });
```

## route 上下文

命令通过 Router DSL 命中后，推荐通过 `useRoute()` 读取路由上下文：

```typescript
const [route] = useRoute();

route.key;
route.text;
route.sourceText;
route.rewrittenText;
route.rawArgs;
route.parsedArgs;
route.params;
```

推荐：

- 不重复解析 `event.MessageText`
- 不在业务 handler 里自己判断前缀
- 命令参数优先依赖 `schema + useRoute()`
- 业务里读取命令参数、分页、目标用户、时间范围时，先从 `route.param(name)` 或 `route.params` 收敛类型

## 执行周期速记

```text
subscribe(create)
  -> middleware
  -> subscribe(mount)
  -> response
  -> subscribe(unmount)
```

说明：

- 中间件和订阅都可能影响后续是否继续执行
- 不调用 `next()` 往往意味着当前链路在此结束
- 多轮交互时先明确自己挂在哪个周期，再决定是否穿透到后续订阅或后续周期

## 相关文档

- 事件字段与类型：[events.md](./events.md)
- 消息构建器：[message-format.md](./message-format.md)
- 路由执行规则：[routing.md](./routing.md)
