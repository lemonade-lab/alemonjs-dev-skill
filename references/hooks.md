# ALemonJS Hooks 参考

> **重要**：所有 Hooks 的 `event` 参数现在都是**可选的**。框架通过 `AsyncLocalStorage`（`hook-event-context.ts`）自动注入事件上下文，在 handler 内部可以直接无参调用。显式传递 event 仍然兼容。

## useEvent

事件过滤与上下文获取（**替代已废弃的 `createEvent`**）。

```typescript
import { useEvent } from 'alemonjs';

// 无参调用（从 AsyncLocalStorage 自动获取事件）
const [event, next] = useEvent({
  selects: ['message.create'],  // 必填：事件类型过滤
  exact?: '/cmd',               // 精确匹配 MessageText
  prefix?: '/cmd ',             // 前缀匹配 MessageText
  regular?: /pattern/           // 正则匹配 MessageText
});

// 显式传入 event（兼容旧用法）
const [event, next] = useEvent(e, { selects: ['message.create'] });
```

**返回值**：`readonly [UseEventResult<T>, next]`

```typescript
type UseEventResult<T> = {
  current: Events[T];     // 完整事件对象
  value: Events[T]['value']; // 平台原始数据（Object.defineProperty 惰性访问）
  match: {
    selects: boolean;     // 事件类型是否匹配
    exact: boolean;       // 精确匹配结果
    prefix: boolean;      // 前缀匹配结果
    regular: boolean;     // 正则匹配结果
  };
};
```

**标准过滤模式**：

```typescript
export default async () => {
  const [event, next] = useEvent({ selects: ['message.create'] });
  if (!event.match.selects) {
    next(); // 事件类型不匹配，跳过
    return;
  }
  // event.current.MessageText, event.current.UserId 等
};
```

## createEvent（已废弃）

```typescript
/** @deprecated 推荐使用 useEvent */
import { createEvent } from 'alemonjs';

const event = createEvent({ event: e, selects: ['message.create'] });
// 返回扁平对象：{ ...eventFields, selects: bool, prefix: bool, ... }
// 迁移：event.selects → event.match.selects
// 迁移：event.MessageText → event.current.MessageText
```

## useMessage

消息操作的核心 hook。

```typescript
import { useMessage, Format } from 'alemonjs';

const [message] = useMessage(); // event 可选

// 发送消息
message.send({
  format: Format.create().addText('Hello'),
  replyId?: string    // 可选，回复指定消息 ID（默认回复触发消息）
});

// 发送 DataEnums 数组（底层 API）
message.send([Text('Hello')]);

// 编辑消息
message.edit({
  format: Format.create().addText('已编辑'),
  messageId?: string  // 不传则编辑触发消息
});

// 删除消息
message.delete({ messageId?: string }); // 不传则删除触发消息

// 置顶消息
message.pin({ messageId?: string });

// 取消置顶
message.unpin({ messageId?: string });

// 获取消息详情
message.get({ messageId?: string });
```

**返回值**：

- `send()` → `Promise<Result[]>`，每个 Result 包含 `{ code, message, data }`
- `edit/delete/pin/unpin/get` → `Promise<Result>`

## useMention

获取消息中 @提及的用户信息。

```typescript
import { useMention } from 'alemonjs';

const [mention] = useMention(); // event 可选

// 查找所有符合条件的提及用户
const users = await mention.find({
  UserId?: string,
  UserKey?: string,
  UserName?: string,
  IsMaster?: boolean,
  IsBot?: boolean       // 默认排除 bot
});
// 返回: { code, message, data: User[], count: number }

// 查找第一个符合条件的用户
const user = await mention.findOne({ IsMaster: true });
// 返回: { code, message, data: User | null, count: number }
```

**User 类型**：

```typescript
type User = {
  UserId: string;
  UserKey: string;
  UserName?: string;
  UserAvatar?: string;
  IsBot?: boolean;
  IsMaster?: boolean;
};
```

## useSubscribe

跨生命周期的事件订阅机制，用于实现"等待用户回复"等交互模式。

```typescript
import { useSubscribe } from 'alemonjs';

// 仅传 selects（event 自动从上下文获取）
const [subscribe] = useSubscribe(['message.create']);

// 显式传入 event（兼容旧用法）
const [subscribe] = useSubscribe(event, ['message.create']);

// 在 create 阶段注册订阅
const reg = subscribe.create(
  async (ev, next) => {
    // ev 是新的事件，满足 keys 匹配条件时触发
  },
  ['UserId'] // 监听条件：事件上哪些 key 的值需要匹配（仅支持基础数据类型）
);
// 返回: { selects: T[], choose: 'create', id: string }

// 在 mount 阶段注册
subscribe.mount(callback, keys);

// 在 unmount 阶段注册
subscribe.unmount(callback, keys);

// 取消订阅
subscribe.cancel(reg);
```

**订阅匹配机制**：

```
注册时存储 keys 的值（如 UserId='123'）
    │
新事件到达 → 对比事件上相同 key 的值
    │
全部匹配 → 触发 callback
```

## useGuild

服务器/公会管理。

```typescript
import { useGuild } from 'alemonjs';

const [guild] = useGuild(); // event 可选

// 获取服务器信息
await guild.info({ guildId?: string }); // → Result<GuildInfo | null>

// 获取 Bot 加入的服务器列表
await guild.list(); // → Result<GuildInfo[]>

// 更新服务器设置
await guild.update({ name?: string, guildId?: string }); // → Result

// 退出服务器
await guild.leave({ guildId?: string, isDismiss?: boolean }); // → Result
```

## useChannel

频道管理。

```typescript
import { useChannel } from 'alemonjs';

const [channel] = useChannel(); // event 可选

// 获取频道信息
await channel.info({ channelId?: string }); // → Result<ChannelInfo | null>

// 获取频道列表
await channel.list({ guildId?: string }); // → Result<ChannelInfo[]>

// 创建频道
await channel.create({ name: string, type?: string, parentId?: string, guildId?: string });

// 更新频道
await channel.update({ channelId: string, name?: string, topic?: string, position?: number });

// 删除频道
await channel.delete({ channelId: string });
```

## useMember

成员管理。

```typescript
import { useMember } from 'alemonjs';

const [member] = useMember(); // event 可选

// 获取成员信息
await member.info({ userId: string, guildId?: string }); // → Result<MemberInfo | null>

// 获取成员列表（支持分页）
await member.list({ guildId?: string, pagination?: PaginationParams });

// 搜索成员
await member.search({ keyword: string, guildId?: string, limit?: number });

// 踢出成员
await member.kick({ userId: string, guildId?: string }); // → Result

// 封禁成员
await member.ban({ userId: string, guildId?: string, reason?: string, duration?: number });

// 解封成员
await member.unban({ userId: string, guildId?: string }); // → Result
```

## useRole

角色管理。

```typescript
import { useRole } from 'alemonjs';

const [role] = useRole(); // event 可选

// 获取角色列表
await role.list({ guildId?: string }); // → Result<RoleInfo[]>

// 创建角色
await role.create({ name: string, color?: number, permissions?: string, guildId?: string });

// 更新角色
await role.update({ roleId: string, name?: string, color?: number, permissions?: string, guildId?: string });

// 删除角色
await role.remove({ roleId: string, guildId?: string }); // → Result

// 为用户分配角色
await role.assign({ roleId: string, userId: string, guildId?: string }); // → Result

// 撤销用户角色
await role.revoke({ roleId: string, userId: string, guildId?: string }); // → Result
```

## usePermission

频道权限管理。

```typescript
import { usePermission } from 'alemonjs';

const [permission] = usePermission(); // event 可选

// 获取用户在频道中的权限
await permission.get({ userId: string, channelId?: string }); // → Result

// 设置用户在频道中的权限
await permission.set({ userId: string, allow?: string, deny?: string, channelId?: string }); // → Result
```

## useReaction

表情回应管理。

```typescript
import { useReaction } from 'alemonjs';

const [reaction] = useReaction(); // event 可选

// 添加表情回应
await reaction.add({ emojiId: string, messageId?: string }); // → Result

// 移除表情回应
await reaction.remove({ emojiId: string, messageId?: string }); // → Result

// 获取某个表情的回应用户列表
await reaction.list({ emojiId: string, messageId?: string, limit?: number }); // → Result
```

## useMedia

媒体文件管理。

```typescript
import { useMedia } from 'alemonjs';

const [media] = useMedia(); // event 可选

type MediaType = 'image' | 'audio' | 'video' | 'file';

// 上传媒体文件（仅上传，不发送）
await media.upload({ type: MediaType, url?: string, data?: string, name?: string });

// 发送媒体到频道
await media.sendChannel({ type: MediaType, url?: string, data?: string, channelId?: string });

// 发送媒体到用户
await media.sendUser({ userId: string, type: MediaType, url?: string, data?: string });
```

## useHistory

消息历史记录。

```typescript
import { useHistory } from 'alemonjs';

const [history] = useHistory(); // event 可选

// 获取频道消息历史
await history.list({
  channelId?: string,
  limit?: number,
  before?: string,  // 在此消息之前
  after?: string    // 在此消息之后
}); // → Result
```

## useMe

获取当前机器人信息（无需 event）。

```typescript
import { useMe } from 'alemonjs';

const [me] = useMe();

// 机器人信息
await me.info(); // → Result<User | null>

// 加入的服务器列表
await me.guilds(); // → Result<GuildInfo[]>

// 私聊线程列表
await me.threads(); // → Result

// 好友列表
await me.friends(); // → Result
```

## useUser

用户信息查询（无需 event）。

```typescript
import { useUser } from 'alemonjs';

const [user] = useUser();

// 获取用户信息
await user.info({ userId: string }); // → Result<User | null>
```

## useRequest

请求处理——好友请求、入群请求等（无需 event）。

```typescript
import { useRequest } from 'alemonjs';

const [request] = useRequest();

// 处理好友请求
await request.friend({ flag: string, approve: boolean, remark?: string }); // → Result

// 处理加群/加服务器请求
await request.guild({ flag: string, subType: string, approve: boolean, reason?: string }); // → Result
```

## useAnnounce

频道公告管理。

```typescript
import { useAnnounce } from 'alemonjs';

const [announce] = useAnnounce(); // event 可选

// 设置公告
await announce.set({ messageId: string, channelId?: string, guildId?: string }); // → Result

// 删除公告（传 messageId 或 'all' 删除所有）
await announce.remove({ messageId?: string, guildId?: string }); // → Result
```

## useClient

透明代理，直接调用平台 SDK API。通过递归 Proxy 实现嵌套属性访问。

```typescript
import { useClient } from 'alemonjs';

// 无参调用
const [client] = useClient();

// 带类型约束（传入 API 类型类）
const [client] = useClient<PlatformAPI>();

// 调用平台 API（键路径自动拼接，发送到 Platform 进程）
const result = await client.api.use.send(params);
// → 实际发送 action: 'client.api', key: 'api.use.send', params: [params]
```

## MessageDirect（主动消息）

不依赖事件上下文，主动向频道或用户发送消息。

```typescript
import { MessageDirect, Format } from 'alemonjs';

const direct = MessageDirect.create();

// 向频道发送
await direct.sendToChannel({
  SpaceId: string,     // 空间 ID
  format: Format | DataEnums[],
  replyId?: string
});

// 向用户私信
await direct.sendToUser({
  OpenID: string,      // 用户 ID
  format: Format | DataEnums[]
});
```
