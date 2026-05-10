# AlemonJS Hooks 参考

本文聚焦业务可直接调用的 hooks。事件字段详见 events，消息结构详见 message-format。

## 使用约定

- 所有 hooks 的 `event` 参数都可选。
- handler 内默认依赖 AsyncLocalStorage 自动注入上下文。
- 推荐无参调用：`useMessage()`、`useMention()`。

## 核心入口

### useEvent

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
  match: { selects }
}
```

说明：

- 当前推荐把命令匹配交给 `Router.create().group().use()`
- `useEvent` 主要用于读取当前事件和在非命中场景下 `next()`
- 若当前 handler 已由 Router DSL 命中，命令与参数优先从 `event.__route` 读取

旧版兼容：

```typescript
const [event, next] = useEvent({
  selects: ['message.create', 'private.message.create', 'interaction.create', 'private.interaction.create'],
  exact?: '/cmd',
  prefix?: '/cmd ',
  regular?: /pattern/
});
```

旧版返回结构通常会包含：

```typescript
{
  current,
  value,
  match: { selects, exact, prefix, regular }
}
```

### useMessage

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
export default async (event) => {
  const [message] = useMessage();
  const uid = String(event.__route?.params?.uid ?? '');

  await message.send({ format });
};
```

### useMention

```typescript
const [mention] = useMention();
await mention.find(options?);
await mention.findOne(options?);
```

### useSubscribe

```typescript
const [subscribe] = useSubscribe(['message.create', 'private.message.create', 'interaction.create', 'private.interaction.create']);
const reg = subscribe.create(callback, ['UserId']);
subscribe.mount(callback, keys);
subscribe.unmount(callback, keys);
subscribe.cancel(reg);
```

## 管理类 hooks

```typescript
useGuild: info list update leave
useChannel: info list create update delete
useMember: info list search kick ban unban
useRole: list create update remove assign revoke
usePermission: get set
useReaction: add remove list
useAnnounce: set remove
```

## 媒体与历史

```typescript
useMedia: upload sendChannel sendUser
useHistory: list
```

## 账号与请求

```typescript
useMe: info guilds threads friends
useUser: info
useRequest: friend guild
```

## 透传客户端

```typescript
const [client] = useClient<PlatformAPI>();
await client.api.use.send(params);
```

## 主动消息

```typescript
const direct = MessageDirect.create();
await direct.sendToChannel({ SpaceId, format, replyId? });
await direct.sendToUser({ OpenID, format });
```

## route 上下文

命令通过 Router DSL 命中后，可直接从事件上拿到路由上下文：

```typescript
event.__route?.key;
event.__route?.text;
event.__route?.rawArgs;
event.__route?.parsedArgs;
event.__route?.params;
```

推荐：

- 不重复解析 `event.MessageText`
- 不在业务 handler 里自己判断前缀
- 命令参数优先依赖 `schema + event.__route.params`

## 相关文档

- 事件字段与类型：[events.md](./events.md)
- 消息构建器：[message-format.md](./message-format.md)
- 路由执行规则：[routing.md](./routing.md)
