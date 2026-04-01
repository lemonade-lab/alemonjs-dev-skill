# ALemonJS 事件参考

## 事件类型总览

所有事件类型由 `EventKeys` 联合类型定义（27 种事件）：

### 消息事件

| EventKey                 | 类型                        | 说明                    |
| ------------------------ | --------------------------- | ----------------------- |
| `message.create`         | `PublicEventMessageCreate`  | 公域消息（群/频道消息） |
| `private.message.create` | `PrivateEventMessageCreate` | 私域消息（私聊）        |
| `message.update`         | `PublicEventMessageUpdate`  | 消息编辑                |
| `message.delete`         | `PublicEventMessageDelete`  | 消息删除                |
| `message.pin`            | `PublicEventMessagePin`     | 消息置顶                |
| `private.message.update` | `PrivateEventMessageUpdate` | 私聊消息编辑            |
| `private.message.delete` | `PrivateEventMessageDelete` | 私聊消息删除            |

### 表情回应事件

| EventKey                  | 说明         |
| ------------------------- | ------------ |
| `message.reaction.add`    | 表情回应添加 |
| `message.reaction.remove` | 表情回应移除 |

### 交互事件

| EventKey                     | 说明                   |
| ---------------------------- | ---------------------- |
| `interaction.create`         | 公域交互（按钮点击等） |
| `private.interaction.create` | 私域交互               |

### 频道/服务器事件

| EventKey         | 说明       |
| ---------------- | ---------- |
| `channel.create` | 频道创建   |
| `channel.delete` | 频道删除   |
| `channel.update` | 频道更新   |
| `guild.join`     | 加入服务器 |
| `guild.exit`     | 退出服务器 |
| `guild.update`   | 服务器更新 |

### 成员事件

| EventKey        | 说明         |
| --------------- | ------------ |
| `member.add`    | 成员加入     |
| `member.remove` | 成员离开     |
| `member.ban`    | 成员封禁     |
| `member.unban`  | 成员解封     |
| `member.update` | 成员信息更新 |

### 通知/请求事件

| EventKey                | 说明         |
| ----------------------- | ------------ |
| `notice.create`         | 公域通知     |
| `private.notice.create` | 私域通知     |
| `private.friend.add`    | 好友添加请求 |
| `private.friend.remove` | 好友删除     |
| `private.guild.add`     | 入群申请     |

### 事件分组常量

```typescript
// 带消息体的事件（可用于路由匹配和消息处理）
type EventsMessageCreateKeys =
  | 'message.create'
  | 'private.message.create'
  | 'interaction.create'
  | 'private.interaction.create';
```

## 事件对象通用字段

框架自动注入的基础字段（`AutoFields`），所有字段**可选**：

```typescript
type AutoFields = {
  CreateAt?: number;   // 事件创建时间戳
  DeviceId?: string;   // 设备/实例 ID
};
```

所有事件类型都附加了 `Expansion` 扩展类型：

```typescript
type Expansion = { [key: string]: any };
// 允许平台适配器通过 FormatEvent.add<E>() 注入扩展字段
```

## 消息事件核心字段

公域消息 `message.create` 的字段组成：

```typescript
// Platform 基础
Platform: string;            // 平台标识 ('discord', 'qq-bot', 'kook' 等)
BotId?: string;              // 机器人 ID
value: any;                  // 平台原始事件数据（不可枚举）

// 用户信息（User 类型）
UserId: string;              // 用户 ID
UserKey: string;             // 用户唯一 Key
UserName?: string;           // 用户名（可选）
UserAvatar?: string;         // 用户头像 URL（可选）
IsMaster: boolean;           // 是否管理员
IsBot: boolean;              // 是否机器人

// 消息信息
MessageId: string;           // 消息 ID
ReplyId?: string;            // 回复消息 ID（可选）
MessageText: string;         // 消息文本内容
MessageMedia: any[];         // 媒体数据

// 空间信息（Guild + Channel）
GuildId: string;             // 服务器/群 ID
SpaceId: string;             // 空间 ID（框架统一标识）
ChannelId: string;           // 频道 ID

// 开放平台
OpenId: string;              // 开放 ID

// 框架注入
name: EventKeys;             // 事件类型名称
Timestamp: number;           // 时间戳（由 FormatEvent 注入）
CreateAt?: number;           // 创建时间（AutoFields）
DeviceId?: string;           // 设备 ID（AutoFields）
```

## 常用事件类型组合

```typescript
// 所有消息类型（公域 + 私域 + 交互）— 最常用
selects: ['message.create', 'private.message.create', 'interaction.create', 'private.interaction.create']

// 仅公域消息
selects: ['message.create']

// 包含交互（按钮回调）
selects: ['message.create', 'interaction.create']
```

## useEvent 过滤

在 handler 内对事件进行过滤（替代已废弃的 `createEvent`）：

```typescript
import { useEvent } from 'alemonjs';

export default async () => {
  const [event, next] = useEvent({
    selects: ['message.create'],
    exact: '/hello'
  });

  // event.match.selects === true  → 事件类型匹配
  // event.match.exact === true    → 精确匹配成功
  // event.match.prefix === false  → 未设置前缀匹配
  // event.match.regular === false → 未设置正则匹配

  if (!event.match.selects) {
    next();
    return;
  }

  // 访问事件字段
  event.current.MessageText;
  event.current.UserId;
  event.value;  // 平台原始数据
};
```

## FormatEvent 事件构建器（平台适配器使用）

`FormatEvent` 用于平台适配器构建标准化事件对象，提供**类型安全**的链式 API——根据事件类型 `T` 自动约束可用方法。

```typescript
import { FormatEvent, wrapEvent } from 'alemonjs';

// 基础构建
const event = FormatEvent.create('message.create')
  .addPlatform({ Platform: 'discord', value: rawData })
  .addGuild({ GuildId: 'g1', SpaceId: 's1' })
  .addChannel({ ChannelId: 'c1' })
  .addUser({ UserId: 'u1', UserKey: 'k1', IsMaster: false, IsBot: false })
  .addMessage({ MessageId: 'm1', ReplyId: 'r1' })
  .addText({ MessageText: 'hello' })
  .addMedia({ MessageMedia: [] })
  .addOpen({ OpenId: 'o1' })
  .value;

// 添加自定义扩展字段（自动加 _ 前缀存储）
const event2 = FormatEvent.create('message.create')
  .addPlatform({ Platform: 'qq', value: raw })
  .add<{ rawType: string }>({ rawType: 'GROUP_AT_MESSAGE_CREATE' })
  .value;

// wrapEvent：包装事件为只读代理，访问扩展字段时自动解析 _ 前缀
const wrapped = wrapEvent<{ rawType: string }>(event2);
wrapped.rawType; // 'GROUP_AT_MESSAGE_CREATE'（实际读取 _rawType）
```

**可用方法约束**：不同事件类型只允许调用对应的 `add*` 方法。例如 `private.message.delete` 只有 `addPlatform` 和 `addMessage`，调用 `addGuild` 会产生类型错误。
