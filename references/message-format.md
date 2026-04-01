# ALemonJS 消息格式参考

## Format 类

主消息构建器，链式 API。

```typescript
import { Format } from 'alemonjs';

const format = Format.create();

// 文本
format.addText('Hello World');
format.addText('提示', { style: 'info' }); // 可选 options

// 图片（Buffer 或带协议字符串）
format.addImage(buffer);                          // Buffer → 自动转 base64://
format.addImage('https://example.com/img.png');   // URL
format.addImage('base64://...');                  // Base64
format.addImage('file:///path/to/local.png');     // 本地路径

// 提及用户
format.addMention(userId);                // @指定用户
format.addMention();                      // @所有人
format.addMention(userId, { belong: 'user' }); // 指定 belong

// 按钮组（传入 FormatButtonGroup 实例）
format.addButtonGroup(buttonGroup);

// Markdown（传入 FormatMarkDown 实例）
format.addMarkdown(mdInstance);

// 原始 Markdown 文本
format.addMarkdownOriginal('**bold** _italic_');

// 附件（带协议字符串）
format.addAttachment('https://example.com/file.pdf', { filename: 'doc.pdf' });

// 音频
format.addAudio('https://example.com/audio.mp3');

// 视频
format.addVideo('https://example.com/video.mp4');

// 吸收另一个 Format
format.absorb(otherFormat);

// 清空
format.clear();

// 获取内部数据
format.value; // DataEnums[]
```

**注意**：`Format` 类**没有** `addBreak()` 方法（只有 `FormatMarkDown` 有）。`addLink()` 已废弃（链接是 Markdown 语法，请用 `FormatMarkDown.addLink()`）。

## FormatButtonGroup

按钮组构建器。

```typescript
import { FormatButtonGroup } from 'alemonjs';

const buttons = new FormatButtonGroup();
// 或: Format.createButtonGroup();

buttons
  .addRow()
  .addButton('确认', 'confirm_data')     // BT(title, data?, options?)
  .addButton('取消', 'cancel_data')
  .addRow()
  .addButton('帮助', 'help_data', { type: 'command', autoEnter: true });

// 使用
format.addButtonGroup(buttons);

// 合并
buttons.absorb(otherButtons);

// 清空
buttons.clear();
```

**`addButton` 参数说明**：

```typescript
// 签名：BT(title: string, data?: string, options?: ButtonOptions)
type ButtonOptions = {
  data?: string;       // command 数据
  toolTip?: string;    // 禁用时的提示
  autoEnter?: boolean; // 是否自动回车
  type?: 'command' | 'link' | 'call'; // 按钮类型
};

// 简写用法
addButton('确认', 'confirm')              // title + data
addButton('确认')                          // 仅 title
addButton('确认', 'data', { type: 'link' }) // 完整参数
```

## FormatMarkDown

结构化 Markdown 构建器。

```typescript
import { FormatMarkDown } from 'alemonjs';

const md = new FormatMarkDown();
// 或: Format.createMarkdown();

md.addTitle('标题')                    // # 标题
  .addSubtitle('副标题')               // ## 副标题
  .addText('正文')
  .addContent('原始内容')              // 不做任何处理的原始文本
  .addBold('加粗')
  .addItalic('斜体')                   // 下划线风格
  .addItalicStar('斜体')               // 星号风格
  .addStrikethrough('删除线')
  .addLink('显示文本', 'https://example.com')  // 注意：第一参数是文本，第二参数是 URL
  .addImage('https://img.url', { width: 200, height: 100 })  // 图片
  .addCode('console.log(1)', { language: 'ts' })
  .addList('项目1', '项目2', '项目3')  // 列表（可变参数）
  .addBlockquote('引用')
  .addDivider()                        // ---
  .addNewline()                        // 换行
  .addNewline(true)                    // 多行换行
  .addBreak()                          // 换行（等同 addNewline()）
  .addMention(userId)                  // @提及
  .addMention()                        // @所有人
  .addButton('操作', { data: 'action' }); // 内联按钮

// 使用
format.addMarkdown(md);

// 合并
md.absorb(otherMd);

// 清空
md.clear();
```

## 底层数据类型（DataEnums）

Format 内部使用的数据联合类型：

| 类型                 | 说明         | 工厂函数                                  |
| -------------------- | ------------ | ----------------------------------------- |
| `DataText`           | 纯文本       | `Text(value, options?)`                   |
| `DataImage`          | 图片         | `Image(buffer \| string)`                |
| `DataButton`         | 单个按钮     | `BT(title, data?, options?)`              |
| `DataButtonRow`      | 按钮行       | `BT.row(...buttons)`                      |
| `DataButtonGroup`    | 按钮组       | `BT.group(...rows)`                       |
| `DataMarkDown`       | Markdown     | `MD(...items)`                            |
| `DataMention`        | @提及        | `Mention(userId?, options?)`              |
| `DataAttachment`     | 附件         | `Attachment(url, options?)`               |
| `DataAudio`          | 音频         | `Audio(url)`                              |
| `DataVideo`          | 视频         | `Video(url)`                              |
| `DataMarkdownOriginal` | 原始 MD   | `MarkdownOriginal(text)`                  |

**废弃类型**：

| 废弃                 | 替代                     |
| -------------------- | ----------------------- |
| `DataLink`           | `FormatMarkDown.addLink()` |
| `DataImageURL`       | `Image(url)`            |
| `DataImageFile`      | `Image(path)`           |

## 发送消息

```typescript
const [message] = useMessage();

// 推荐方式
message.send({
  format: Format.create().addText('Hello'),
  replyId: event.current.MessageId // 默认自动回复触发消息
});

// 底层方式
message.send([Text('Hello'), Image(buffer)]);
```

## 图片渲染（jsxp）

将 React 组件渲染为图片 Buffer：

```typescript
import { renderComponentToBuffer } from 'jsxp';

const buffer = await renderComponentToBuffer(
  '/route',              // 路由标识
  ComponentFunction,     // React 组件
  { props }              // Props
);

// buffer: Buffer | boolean (失败时为 false)
```

组件模板：

```tsx
import React from 'react';
import { LinkStyleSheet, BackgroundImage } from 'jsxp';
import css from '@src/assets/main.css';

export default function Card({ title }: { title: string }) {
  return (
    <html>
      <head>
        <LinkStyleSheet src={css} />
      </head>
      <body>
        <div className='p-4'>{title}</div>
      </body>
    </html>
  );
}
```

样式使用 Tailwind CSS，通过 `@src/assets/main.css` 引入。
