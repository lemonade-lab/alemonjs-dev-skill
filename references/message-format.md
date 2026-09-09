# AlemonJS 消息格式参考

本文只描述消息构建器，不重复 hooks 与事件内容。

## Format

```typescript
const format = Format.create();

format.addText('text');
format.addImage(bufferOrUrl);
format.addMention(userId?);
format.addButtonGroup(...rows);
format.addMarkdown(md);
format.addMarkdownOriginal('**raw**');
format.addAttachment(url, options?);
format.addAudio(url);
format.addVideo(url);
format.absorb(other);
format.clear();
```

注意：
- 链接请用 Markdown 构建器的 `addLink()`，不要用废弃 `Format.addLink()`。
- `Format` 是统一消息容器。业务层优先从 `Format.create()` 出发构建整条消息。
- 按钮与 Markdown 的推荐写法以官网文档示例为准：按钮使用 `Format.createButtonGroup()`，Markdown 使用 `Format.createMarkdown()`。

## 按钮推荐写法

```typescript
const format = Format.create().addButtonGroup(
  Format.createButtonGroup()
    .addRow()
    .addButton('确认', 'confirm')
    .addButton('取消', 'cancel')
    .addRow()
    .addButton('帮助', 'help', { type: 'command', autoEnter: true })
);
```

- 一个按钮组最多 5 行、每行最多 5 个按钮；超过不保证能发送或渲染。
- 需要跳转链接时用 `{ type: 'link' }`，平台交互可用 `{ type: 'call' }`；`autoEnter`、`showList`、`toolTip` 仅按平台能力生效。

## Markdown 推荐写法

```typescript
const md = Format.createMarkdown();
md.addTitle('标题')
  .addSubtitle('副标题')
  .addText('正文')
  .addBold('加粗')
  .addItalic('斜体')
  .addStrikethrough('删除线')
  .addLink('显示文本', 'https://example.com')
  .addImage('https://img.url', { width: 200, height: 100 })
  .addCode('console.log(1)', { language: 'ts' })
  .addList('a', 'b')
  .addBlockquote('引用')
  .addDivider()
  .addNewline()
  .addMention(userId?)
  .addButton('操作', { data: 'action' });
```

## 最小发送示例

```typescript
const [message] = useMessage();
message.send({ format: Format.create().addText('Hello') });
```

## 发送与降级

- 媒体（Image/Audio/Video/Attachment）优先于 Markdown/按钮，Markdown/按钮优先于普通文本。
- 有媒体时，支持 caption 的平台会合并低优先级内容；不支持时框架可能拆成多次实际发送。因此不要把一次 `message.send({ format })` 当作一次平台 API 调用。
- Markdown 与 Text 属于同一信息层：支持 Markdown 时合并为 Markdown；否则 Markdown 降级为文本。
- 不支持的 ButtonGroup、Markdown、音视频、附件会逐步降级为文本。业务层只需构建语义正确的 `Format`，并检查 action 返回结果。

## 相关文档

- 业务调用入口：[hooks.md](./hooks.md)
- 图片卡片规范：[jsxp-dev.md](./jsxp-dev.md)
