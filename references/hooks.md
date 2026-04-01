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
  selects: ['message.create'],
  exact?: '/cmd',
  prefix?: '/cmd ',
  regular?: /pattern/
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

### useMention

```typescript
const [mention] = useMention();
await mention.find(options?);
await mention.findOne(options?);
```

### useSubscribe

```typescript
const [subscribe] = useSubscribe(['message.create']);
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

## 账号与请求（无需 event）

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

## 相关文档

- 事件字段与类型：[events.md](./events.md)
- 消息构建器：[message-format.md](./message-format.md)
- 路由执行规则：[routing.md](./routing.md)
