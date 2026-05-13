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
- `Format` 没有 `addBreak()`。
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
  .addBreak()
  .addMention(userId?)
  .addButton('操作', { data: 'action' });
```

## 最小发送示例

```typescript
const [message] = useMessage();
message.send({ format: Format.create().addText('Hello') });
```

## 相关文档

- 业务调用入口：[hooks.md](./hooks.md)
- 图片卡片规范：[jsxp-dev.md](./jsxp-dev.md)
