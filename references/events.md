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

## useEvent 标准守卫

```typescript
const [event, next] = useEvent({ selects: ['message.create', 'private.message.create', 'interaction.create', 'private.interaction.create'], prefix: '/cmd' });
if (!event.match.selects || !event.match.prefix) {
  next();
  return;
}
```

## 标准化建议

- 先保证最小字段集，再追加平台扩展字段
- 自定义字段通过扩展能力注入，避免污染公共字段
- 业务层只读取统一字段，不读取平台原始结构

## 相关文档

- Hook 调用标准：[hooks.md](./hooks.md)
- 路由过滤标准：[routing.md](./routing.md)
