# jsxp 图片开发速查

## 渲染链路

```text
handler
-> renderComponentIsHtmlToBuffer(Component, props)
-> jsxp 渲染
-> PNG Buffer
-> Format.addImage(buffer)
-> message.send({ format })
```

## 组件约定

- 组件输出完整 html/head/body 结构，或使用统一 Html 外壳。
- 样式通过 LinkStyleSheet 引入编译后的 CSS 文件。
- 资源（图片、字体）统一使用 @src 别名导入。

## 最小组件模板

```tsx
import React from 'react';
import { LinkStyleSheet } from 'jsxp';
import css from '@src/assets/main.css';

export default function Card({ data }: { data: string }) {
  return (
    <html>
      <head><LinkStyleSheet src={css} /></head>
      <body>
        <div className='p-4'>{data}</div>
      </body>
    </html>
  );
}
```

## handler 示例

```typescript
import { renderComponentIsHtmlToBuffer } from 'jsxp';
import { useMessage, Format } from 'alemonjs';
import Card from '@src/image/component/Card';

const [message] = useMessage();
const img = await renderComponentIsHtmlToBuffer(Card, { data: 'hello' });
if (typeof img !== 'boolean') {
  message.send({ format: Format.create().addImage(img) });
}
```

## 联动文档

- 构建与别名： [lvyjs-dev.md](./lvyjs-dev.md)
- 消息结构： [message-format.md](./message-format.md)
- 主技能入口： [../SKILL.md](../SKILL.md)
