---
name: alemonjs-dev-skill
description: 'ALemonJS 跨平台聊天机器人开发技能。Use when: 开发 ALemonJS 应用、创建消息响应、编写中间件、使用 hooks（useEvent/useMessage/useMention/useSubscribe/useGuild/useChannel/useMember/useRole/useReaction/usePermission/useMedia/useHistory/useMe/useUser/useRequest/useAnnounce/useClient）、构建路由、处理事件、发送消息（Format）、图片渲染（jsxp）、创建平台适配器（FormatEvent/definePlatform）、配置 defineChildren 生命周期、lvyjs 构建工具配置（lvy.config.ts/开发热重载/生产构建/路径别名/静态资源/样式处理/PostCSS/Tailwind）。适用于 bot 开发、插件开发、跨平台适配、项目构建配置。'
argument-hint: '描述你要开发的功能，如 "创建一个 /sign 签到指令" 或 "添加图片回复"'
---

# ALemonJS 跨平台聊天机器人开发

## 适用场景

- 创建新的消息响应指令（如 /hello、/help、/签到）
- 编写中间件（权限校验、日志、限流等）
- 使用 hooks 处理消息、提及、订阅、成员管理、角色管理等
- 构建路由树（嵌套路由、懒加载）
- 发送富文本消息（文本、图片、按钮、Markdown、附件、音视频）
- 使用 jsxp 渲染图片
- 创建或适配新平台（FormatEvent + definePlatform）
- 配置应用生命周期
- 配置 lvyjs 构建工具（lvy.config.ts、路径别名、静态资源、样式、构建选项）
- 项目脚手架搭建与开发环境配置

## 核心概念速查

| 概念             | 说明                                                | 详见                                       |
| ---------------- | --------------------------------------------------- | ------------------------------------------ |
| `defineChildren` | 定义子模块入口（生命周期 + 注册路由）               | [架构参考](./references/architecture.md)   |
| `defineRouter`   | 声明式路由（正则/前缀/精确匹配 + 懒加载）           | [路由参考](./references/routing.md)        |
| `useEvent`       | Hook：事件过滤（替代已废弃的 `createEvent`）        | [Hooks 参考](./references/hooks.md)        |
| `useMessage`     | Hook：发送/编辑/删除/置顶消息                       | [Hooks 参考](./references/hooks.md)        |
| `useMention`     | Hook：获取 @提及用户                                | [Hooks 参考](./references/hooks.md)        |
| `useSubscribe`   | Hook：事件订阅（跨生命周期监听）                    | [Hooks 参考](./references/hooks.md)        |
| `Format`         | 消息格式构建器（链式 API）                          | [消息参考](./references/message-format.md) |
| `FormatEvent`    | 事件构建器（平台适配器用，链式类型安全）            | [事件参考](./references/events.md)         |
| `lvyjs`          | 构建工具（开发热重载 + rollup 生产构建）            | [lvyjs 参考](./references/lvyjs-dev/SKILL.md) |

> **重要**：所有 Hooks 的 `event` 参数现在都是**可选的**。框架通过 `AsyncLocalStorage` 自动注入事件上下文，handler 内部可以直接无参调用 `useMessage()`、`useMention()` 等。传 event 参数仍然兼容。

## 开发环境（lvyjs）

ALemonJS 使用 **lvyjs** 作为开发与构建工具。详细配置见 [lvyjs 参考](./references/lvyjs-dev/SKILL.md)。

### 项目脚手架

```bash
npm create alemonjs    # 创建新项目
cd <项目名>
npm install
npm run dev            # 开发模式（tsx 热重载）
```

### 标准项目结构

```
├── app.ts                 # lvyjs 开发入口
├── index.js               # 生产入口
├── lvy.config.ts          # lvyjs 配置
├── tsconfig.json          # 继承 lvyjs/tsconfig.json
├── alemon.config.yaml     # ALemonJS 平台/PM2 配置
├── jsxp.config.tsx        # 图片预览路由配置
├── postcss.config.mjs     # PostCSS 插件（Tailwind + cssnano）
├── tailwind.config.js     # Tailwind 内容扫描范围
├── src/
│   ├── index.ts           # 应用入口（defineChildren + defineRouter）
│   ├── env.d.ts           # 类型声明
│   ├── response/          # 响应处理器
│   ├── middleware/         # 中间件
│   ├── image/component/   # jsxp 图片组件
│   └── assets/            # 静态资源（CSS、图片、字体）
```

### 核心命令

| 命令              | 说明                                |
| ----------------- | ----------------------------------- |
| `npm run dev`     | `npx lvy app.ts` — 开发热重载      |
| `npm run view`    | `npx lvy app.ts --jsxp` — 图片预览 |
| `npm run build`   | `npx lvy build` — 生产构建         |
| `npm run start`   | PM2 生产启动                        |

### 关键配置（lvy.config.ts）

```typescript
import { defineConfig } from 'lvyjs';
import { dirname, join } from 'path';
import { fileURLToPath } from 'url';
const __dirname = dirname(fileURLToPath(import.meta.url));

export default defineConfig({
  alias: {
    entries: [{ find: '@src', replacement: join(__dirname, 'src') }]
  },
  assets: {
    filter: /\.(png|jpg|jpeg|gif|svg|webp|ico|yaml|txt|ttf|md)$/
  },
  watch: ['src/**/*.{ts,tsx,js,jsx,json,html}'],
  build: {
    typescript: {
      include: ['src/**/*.ts', 'src/**/*.tsx'],
      outDir: 'lib',
      declaration: true
    }
  }
});
```

> **注意**：静态资源和样式文件导入后返回的是**文件绝对路径**（string），不是内容或 URL。

## 开发流程

### 1. 创建新指令（最常见场景）

**步骤**：

1. 在 `src/response/` 下新建处理器文件，`export default` 一个异步函数
2. 在 `src/index.ts` 的 `defineRouter` 中注册路由
3. 使用 `lazy(() => import('./response/xxx'))` 懒加载

**模板 — 简单文本回复**：

```typescript
// src/response/ping.ts
import { useEvent, useMessage, Format } from 'alemonjs';

export default async () => {
  const [event, next] = useEvent({
    selects: ['private.message.create', 'message.create', 'interaction.create', 'private.interaction.create']
  });
  if (!event.match.selects) {
    next();
    return;
  }

  const [message] = useMessage();
  const format = Format.create();
  format.addText('pong!');
  message.send({ format });
};
```

**模板 — 带参数解析**：

```typescript
// src/response/echo.ts
import { useEvent, useMessage, Format } from 'alemonjs';

export default async () => {
  const [event, next] = useEvent({
    selects: ['message.create'],
    prefix: '/echo '
  });
  if (!event.match.selects || !event.match.prefix) {
    next();
    return;
  }

  const text = event.current.MessageText.replace(/^\/echo\s*/, '');
  const [message] = useMessage();
  const format = Format.create();
  format.addText(text || '请输入内容');
  message.send({ format });
};
```

**模板 — 带 @用户**：

```typescript
// src/response/greet.ts
import { useEvent, useMessage, useMention, Format } from 'alemonjs';

export default async () => {
  const [event, next] = useEvent({
    selects: ['message.create']
  });
  if (!event.match.selects) {
    next();
    return;
  }

  const [message] = useMessage();
  const [mention] = useMention();

  const userRes = await mention.findOne();
  if (!userRes.count || !userRes.data) {
    const format = Format.create();
    format.addText('请 @一个用户');
    message.send({ format });
    return;
  }

  const format = Format.create();
  format.addText(`你好, ${userRes.data.UserName || userRes.data.UserId}!`);
  message.send({ format });
};
```

### 2. 注册路由

在 `src/index.ts` 修改 `defineRouter` 数组：

```typescript
import { defineChildren, defineRouter, lazy, logger } from 'alemonjs';

const responseRouter = defineRouter([
  {
    // 匹配方式三选一或组合使用
    exact: '/ping', // 精确匹配（最快，O(1)）
    // prefix: '/echo',               // 前缀匹配（O(n)）
    // regular: /^\/help/,            // 正则匹配（缓存编译）
    selects: ['private.message.create', 'message.create', 'interaction.create', 'private.interaction.create'],
    handler: lazy(() => import('./response/ping'))
  },
  {
    regular: /hello/,
    selects: ['message.create'],
    handler: lazy(() => import('./response/greet')),
    // 嵌套子路由
    children: [
      {
        exact: '/hello world',
        handler: lazy(() => import('./response/hello-world'))
      }
    ]
  }
]);

export default defineChildren({
  register() {
    return { responseRouter };
  },
  onCreated() {
    logger.info('应用启动完成');
  }
});
```

### 3. 发送富文本消息

```typescript
import { Format, FormatButtonGroup, FormatMarkDown } from 'alemonjs';

// 纯文本
const f1 = Format.create();
f1.addText('Hello World');

// 图片（Buffer | 带协议字符串）
const f2 = Format.create();
f2.addImage(buffer); // Buffer（自动转 base64://）
f2.addImage('https://example.com/img.png'); // URL
f2.addImage('base64://...'); // Base64
f2.addImage('file:///path/to/local.png'); // 本地文件

// 按钮组
const buttons = new FormatButtonGroup();
buttons
  .addRow()
  .addButton('确认', 'confirm') // BT(title, data?, options?)
  .addButton('取消', 'cancel')
  .addRow()
  .addButton('帮助', 'help', { type: 'command', autoEnter: true });
const f3 = Format.create();
f3.addText('请选择：');
f3.addButtonGroup(buttons);

// Markdown
const md = new FormatMarkDown();
md.addTitle('标题').addText('正文内容').addBold('加粗').addLink('ALemonJS', 'https://alemonjs.com');
const f4 = Format.create();
f4.addMarkdown(md);

// 附件 / 音频 / 视频
const f6 = Format.create();
f6.addAttachment('https://example.com/file.pdf', { filename: 'doc.pdf' });
f6.addAudio('https://example.com/audio.mp3');
f6.addVideo('https://example.com/video.mp4');

// 组合消息（链式调用）
const f5 = Format.create();
f5.addText('标题').addImage(buffer).addText('描述文字');
```

### 4. 图片渲染（jsxp）

```typescript
// src/response/card.ts
import { renderComponentToBuffer } from 'jsxp';
import { useEvent, useMessage, Format } from 'alemonjs';
import CardComponent from '@src/image/component/Card';

export default async () => {
  const [event, next] = useEvent({ selects: ['message.create'] });
  if (!event.match.selects) {
    next();
    return;
  }

  const [message] = useMessage();
  const img = await renderComponentToBuffer('/card', CardComponent, {
    title: 'Hello',
    data: event.current.MessageText
  });

  const format = Format.create();
  if (typeof img !== 'boolean') {
    format.addImage(img);
  } else {
    format.addText('图片渲染失败');
  }
  message.send({ format });
};
```

```tsx
// src/image/component/Card.tsx
import React from 'react';
import { LinkStyleSheet } from 'jsxp';
import css from '@src/assets/main.css';

export default function Card({ title, data }: { title: string; data: string }) {
  return (
    <html>
      <head>
        <LinkStyleSheet src={css} />
      </head>
      <body>
        <div className='p-4 bg-white rounded-lg shadow'>
          <h1 className='text-2xl font-bold text-blue-500'>{title}</h1>
          <p className='mt-2 text-gray-700'>{data}</p>
        </div>
      </body>
    </html>
  );
}
```

### 5. 编写中间件

```typescript
// src/index.ts
import { defineChildren, defineRouter, lazy } from 'alemonjs';

const middlewareRouter = defineRouter([
  {
    prefix: '/',
    selects: ['message.create'],
    handler: lazy(() => import('./middleware/auth'))
  }
]);

const responseRouter = defineRouter([
  /* ... */
]);

export default defineChildren({
  register() {
    return { responseRouter, middlewareRouter };
  }
});
```

```typescript
// src/middleware/auth.ts
export default async () => {
  const [event] = useEvent({ selects: ['message.create'] });
  // return true  → 继续执行下一个 handler
  // return void/false → 终止链
  if (event.current.IsMaster) {
    return true; // 放行
  }
  // 鉴权失败，不调用 next，链终止
};
```

### 6. 使用订阅（useSubscribe）

```typescript
// src/response/confirm.ts
import { useEvent, useMessage, useSubscribe, Format } from 'alemonjs';

export default async () => {
  const [event, next] = useEvent({ selects: ['message.create'] });
  if (!event.match.selects) {
    next();
    return;
  }

  const [message] = useMessage();
  const [subscribe] = useSubscribe(['message.create']);

  // 发送确认提示
  const format = Format.create();
  format.addText('确认操作？回复 "是" 或 "否"');
  message.send({ format });

  // 注册订阅：等待同一用户在 create 阶段的下条消息
  const reg = subscribe.create(
    async (ev, next) => {
      if (ev.MessageText === '是') {
        const [msg] = useMessage(ev);
        msg.send({ format: Format.create().addText('已确认！') });
      } else {
        const [msg] = useMessage(ev);
        msg.send({ format: Format.create().addText('已取消。') });
      }
      // 处理完毕后取消订阅
      subscribe.cancel(reg);
    },
    ['UserId'] // 监听的 event key（仅匹配同一用户）
  );
};
```

### 7. 主动消息（MessageDirect）

```typescript
import { MessageDirect, Format } from 'alemonjs';

const direct = MessageDirect.create();

// 向频道发送
await direct.sendToChannel({
  SpaceId: 'channel_id_123',
  format: Format.create().addText('定时通知')
});

// 向用户私信
await direct.sendToUser({
  OpenID: 'user_id_456',
  format: Format.create().addText('私信通知')
});
```

### 8. 消息操作（编辑/删除/置顶）

```typescript
import { useEvent, useMessage, Format } from 'alemonjs';

export default async () => {
  const [event, next] = useEvent({ selects: ['message.create'] });
  if (!event.match.selects) { next(); return; }

  const [message] = useMessage();

  // 编辑消息
  await message.edit({
    format: Format.create().addText('已编辑的内容'),
    messageId: 'target_message_id' // 不传则编辑触发消息
  });

  // 删除消息
  await message.delete({ messageId: 'target_message_id' }); // 不传则删除触发消息

  // 置顶/取消置顶
  await message.pin();    // 置顶触发消息
  await message.unpin();  // 取消置顶触发消息

  // 获取消息详情
  const res = await message.get({ messageId: 'some_id' });
};
```

### 9. 管理操作（服务器/频道/成员/角色）

```typescript
import { useEvent, useGuild, useChannel, useMember, useRole } from 'alemonjs';

export default async () => {
  const [event, next] = useEvent({ selects: ['message.create'] });
  if (!event.match.selects) { next(); return; }

  // 服务器管理
  const [guild] = useGuild();
  const guildInfo = await guild.info();
  const guildList = await guild.list();

  // 频道管理
  const [channel] = useChannel();
  const channelInfo = await channel.info();
  const channels = await channel.list();
  await channel.create({ name: '新频道' });

  // 成员管理
  const [member] = useMember();
  const memberInfo = await member.info({ userId: '123' });
  const members = await member.list();
  await member.kick({ userId: '123' });
  await member.ban({ userId: '123', reason: '违规', duration: 3600 });

  // 角色管理
  const [role] = useRole();
  const roles = await role.list();
  await role.create({ name: '管理员', color: 0xff0000 });
  await role.assign({ roleId: 'r1', userId: 'u1' });
};
```

## 关键规则

### useEvent 返回值结构

```typescript
const [event, next] = useEvent({
  selects: ['message.create'],  // 必填：事件类型过滤
  exact?: '/cmd',               // 精确匹配 MessageText
  prefix?: '/cmd',              // 前缀匹配 MessageText
  regular?: /pattern/           // 正则匹配 MessageText
});

// event.current  — 原始事件对象（包含 MessageText, UserId, GuildId 等）
// event.value    — 平台原始数据（event.current.value 的快捷访问）
// event.match    — 匹配结果
//   .selects     — boolean：事件类型是否匹配
//   .exact       — boolean：精确匹配是否成功
//   .prefix      — boolean：前缀匹配是否成功
//   .regular     — boolean：正则匹配是否成功
// next()         — 跳过当前 handler，进入下一个路由节点
```

### Handler 返回值约定

```
return true   → 继续执行链中的下一个 handler
return void   → 停止当前链（默认行为）
return false  → 停止当前链
```

### next() 的含义

- 在路由 handler 中：`next()` 表示当前 handler 不处理，跳到下一个路由节点
- 在中间件中：`return true` 放行到下一个中间件或响应处理器

### 事件过滤模式

始终在 handler 开头用 `useEvent` + `if (!event.match.selects)` 过滤事件类型：

```typescript
export default async () => {
  const [event, next] = useEvent({
    selects: ['message.create'] // 只处理公域消息
  });
  if (!event.match.selects) {
    next(); // 不是目标事件类型，放过
    return;
  }
  // 处理逻辑...
};
```

### 路由匹配优先级

```
exact (精确) > prefix (前缀) > regular (正则)
```

性能排序：exact O(1) → prefix O(n) → regular（缓存编译后匹配）

### defineChildren 生命周期

```
defineChildren({
  register()    → 注册路由/中间件（返回 { responseRouter, middlewareRouter }）
  onCreated()   → 注册完成后触发（初始化资源）
  onMounted()   → 挂载完成后触发（接收 store 参数）
  unMounted(e)  → 卸载时触发（清理资源，e 为错误对象）
})
```

## 项目结构约定

```
├── app.ts                 # lvyjs 开发入口（dev 模式）
├── index.js               # 生产入口（build 后使用）
├── lvy.config.ts          # lvyjs 构建配置
├── tsconfig.json          # TypeScript 配置（继承 lvyjs/tsconfig.json）
├── alemon.config.yaml     # 平台登录 + PM2 部署配置
├── jsxp.config.tsx        # 图片预览路由
├── postcss.config.mjs     # PostCSS 插件
├── tailwind.config.js     # Tailwind CSS 配置
├── src/
│   ├── index.ts           # 应用入口：defineChildren + defineRouter
│   ├── env.d.ts           # 类型声明（/// <reference types="lvyjs/env" /> + alemonjs/env）
│   ├── response/          # 响应处理器（每个文件 export default 一个函数）
│   │   ├── help.ts
│   │   └── sign.ts
│   ├── middleware/         # 中间件处理器
│   │   └── auth.ts
│   ├── image/             # jsxp 图片组件
│   │   └── component/
│   │       ├── Html.tsx    # HTML 外壳组件（引入样式）
│   │       └── Card.tsx
│   └── assets/            # 静态资源（CSS、图片、字体）
│       ├── main.css       # Tailwind 入口样式
│       └── root.css       # 主题变量
```

## 完整 Hooks 速查表

| Hook            | 需要 event | 主要方法                                                   |
| --------------- | ---------- | ---------------------------------------------------------- |
| `useEvent`      | 自动       | 返回 `[{ current, value, match }, next]`                   |
| `useMessage`    | 可选       | `send`, `edit`, `delete`, `pin`, `unpin`, `get`            |
| `useMention`    | 可选       | `find(options?)`, `findOne(options?)`                      |
| `useSubscribe`  | 可选       | `create`, `mount`, `unmount`, `cancel`                     |
| `useGuild`      | 可选       | `info`, `list`, `update`, `leave`                          |
| `useChannel`    | 可选       | `info`, `list`, `create`, `update`, `delete`               |
| `useMember`     | 可选       | `info`, `list`, `kick`, `ban`, `unban`, `search`           |
| `useRole`       | 可选       | `list`, `create`, `update`, `remove`, `assign`, `revoke`   |
| `usePermission` | 可选       | `get`, `set`                                               |
| `useReaction`   | 可选       | `add`, `remove`, `list`                                    |
| `useMedia`      | 可选       | `upload`, `sendChannel`, `sendUser`                        |
| `useHistory`    | 可选       | `list`                                                     |
| `useMe`         | 不需要     | `info`, `guilds`, `threads`, `friends`                     |
| `useUser`       | 不需要     | `info`                                                     |
| `useRequest`    | 不需要     | `friend`, `guild`                                          |
| `useAnnounce`   | 可选       | `set`, `remove`                                            |
| `useClient`     | 可选       | 递归代理，调用 `client.api.xxx()` 透传到平台 SDK           |

## 可用事件类型（EventKeys）

| 事件                         | 说明                   |
| ---------------------------- | ---------------------- |
| `message.create`             | 公域消息创建           |
| `private.message.create`     | 私域消息创建           |
| `interaction.create`         | 公域交互（按钮点击等） |
| `private.interaction.create` | 私域交互               |
| `message.update`             | 消息更新               |
| `message.delete`             | 消息删除               |
| `private.message.update`     | 私聊消息编辑           |
| `private.message.delete`     | 私聊消息删除           |
| `message.reaction.add`       | 表情回应添加           |
| `message.reaction.remove`    | 表情回应移除           |
| `message.pin`                | 消息置顶               |
| `channel.create`             | 频道创建               |
| `channel.delete`             | 频道删除               |
| `channel.update`             | 频道更新               |
| `guild.join`                 | 加入服务器             |
| `guild.exit`                 | 退出服务器             |
| `guild.update`               | 服务器更新             |
| `member.add`                 | 成员加入               |
| `member.remove`              | 成员离开               |
| `member.ban`                 | 成员封禁               |
| `member.unban`               | 成员解封               |
| `member.update`              | 成员信息更新           |
| `notice.create`              | 公域通知               |
| `private.notice.create`      | 私域通知               |
| `private.friend.add`         | 好友添加请求           |
| `private.friend.remove`      | 好友删除               |
| `private.guild.add`          | 入群申请               |

**常用事件组合**：

```typescript
// 所有带消息体的事件（EventsMessageCreateKeys）
selects: ['message.create', 'private.message.create', 'interaction.create', 'private.interaction.create']
```

## 开发命令

```bash
npm run dev          # npx lvy app.ts — 开发模式（tsx 热重载）
npm run view         # npx lvy app.ts --jsxp — 图片组件预览（localhost:8080）
npm run build        # npx lvy build — 生产构建（src/ → lib/）
npm run app          # node index.js — 运行生产构建产物
npm run start        # PM2 生产启动
npm run stop         # PM2 停止
npm run review       # npx alemonc start — 审核 UI
```

## 常见错误排查

| 错误                           | 原因                              | 解决                                          |
| ------------------------------ | --------------------------------- | --------------------------------------------- |
| `module has no default export` | handler 文件没有 `export default` | 确保 `export default async () => {}`          |
| 消息不触发                     | `selects` 未匹配事件类型          | 检查 `useEvent` 中的 `selects` 数组           |
| 路由不匹配                     | 正则/前缀/精确匹配失败            | 检查 `defineRouter` 中的匹配条件              |
| `next()` 后依然执行            | `next()` 后未 `return`            | 在 `next()` 之后加 `return`                   |
| 图片渲染返回 boolean           | jsxp 组件出错                     | 检查组件引用路径和 CSS 导入                   |
| `event is not object`          | hook 在无上下文的环境中调用       | 确保在 handler 内部调用 hooks（有事件上下文） |

## 废弃 API 迁移指南

| 废弃 API                      | 替代方案                                      |
| ------------------------------ | --------------------------------------------- |
| `createEvent({ event: e, selects })` | `useEvent({ selects })`                |
| `event.selects`（布尔值）      | `event.match.selects`                         |
| `event.MessageText`           | `event.current.MessageText`                   |
| `useMessage(event)`           | `useMessage()`（event 可选）                  |
| `useMention(event)`           | `useMention()`（event 可选）                  |
| `useSubscribe(event, selects)` | `useSubscribe(selects)`（event 可选）         |
| `useSend(event)`              | `useMessage()`                                |
| `useObserver(selects)`        | `useSubscribe(selects)`                       |
| `Image.url()` / `Image.file()` | `Image(buffer \| string)`                   |
| `sendToChannel()` / `sendToUser()` | `MessageDirect.create().sendToChannel()` |
| `Format.addLink()`            | `FormatMarkDown.addLink()`（链接是 MD 语法） |
