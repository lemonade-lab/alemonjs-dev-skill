# AlemonJS 事件标准

本文给出通用项目应遵循的事件字段与构建规范。

## 事件分类（通用）

常用处理组合：

```typescript
selects: [
  'message.create',
  'private.message.create',
  'interaction.create',
  'private.interaction.create'
]
```

其余分类：message.update/delete、reaction、channel、guild、member、notice、private 请求。

## 消息事件标准最小集

```typescript
{
  Platform: string;
  value: any;

  UserId: string;
  UserKey: string;
  IsMaster: boolean;
  IsBot: boolean;

  MessageId: string;
  MessageText: string;
  MessageMedia: any[];

  SpaceId: string;
  name: EventKeys;
  Timestamp: number;
}
```

说明：`GuildId/ChannelId/OpenId/ReplyId` 按平台能力选填。

## route 上下文标准

```typescript
{
  __route?: {
    key: string;
    text: string;
    rawArgs: string[];
    parsedArgs: unknown[];
    params: Record<string, unknown>;
  }
}
```

说明：

- 当事件被 Router DSL 命中时，框架会自动写入底层事件对象的 `__route`
- `text` 是当前 scope 归一化后的命令文本
- `params` 来自 `schema` 校验后的命名参数

业务通过 `useEvent()` 读取时，应使用 `event.current.__route` 访问这段上下文。

## useEvent 标准守卫

```typescript
const [event, next] = useEvent({
  selects: ['message.create', 'private.message.create', 'interaction.create', 'private.interaction.create']
});

if (!event.match.selects) {
  next();
  return;
}
```

说明：

- 当前推荐用 `Router.create().group().use()` 做命令匹配
- `useEvent` 更适合作为事件类型过滤守卫，不再作为主命令路由入口

## 标准化建议

- 先保证最小字段集，再追加平台扩展字段
- 自定义字段通过扩展能力注入，避免污染公共字段
- 业务层只读取统一字段，不读取平台原始结构

## 相关文档

- Hook 调用标准：[hooks.md](./hooks.md)
- 路由过滤标准：[routing.md](./routing.md)
