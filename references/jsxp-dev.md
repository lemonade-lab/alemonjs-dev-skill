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
import URL_SCSS from '@src/assets/css/input.scss';
import URL_TTT from '@src/assets/font/tttgbnumber.ttf';
import classNames from 'classnames';
import React from 'react';

const HTML = (props: React.DetailedHTMLProps<React.HTMLAttributes<HTMLBodyElement>, HTMLBodyElement> & {}) => {
  const { children, className, ...reSet } = props;

  return (
    <html className='p-0 m-0'>
      <head>
        <link type='text/css' rel='stylesheet' href={URL_SCSS} />
        <meta httpEquiv='content-type' content='text/html;charset=utf-8' />
        <style
          dangerouslySetInnerHTML={{
            __html: `
              @font-face {
                font-family: 'tttgbnumber';
                src: url('${URL_TTT}'); 
                font-weight: normal; 
                font-style: normal; 
              }
              body { 
                font-family: 'tttgbnumber', 
                system-ui, sans-serif; 
              }
            `
          }}
        />
      </head>
      <body className={classNames('p-0 m-0 w-full text-center', className)} {...reSet}>
        {children}
      </body>
    </html>
  );
};

export default HTML;
```

## handler 示例

```typescript
import { renderComponentIsHtmlToBuffer } from 'jsxp';
import { useMessage, Format } from 'alemonjs';
import Card from '@src/image/component/Card';
export default async () => {
    const [message] = useMessage();
    const img = await renderComponentIsHtmlToBuffer(Card, { data: 'hello' });
    if (typeof img !== 'boolean') {
       message.send({ format: Format.create().addImage(img) });
   }
}
```

## 约束

- 组件都用HTML进行包裹

- 设置图片宽度要在HTML组件上进行设置，且根据各自情况设定宽度大小来确保最佳

```tsx
<HTML style={{ width: 'px' }}></HTML>
```

- 背景图效果务必是在左上角开始自然放大

- 不能出现纯白/纯黑背景

- 不能出现白底白字，黑底黑字

- 增删改组件务必在 jsxp.config.tsx 上补充或同步调整路由

- 尽量避免纯文本的图片，如果本地有icon，能带上icon、图片等素材的要带上

## 联动文档

- 构建与别名： [lvyjs-dev.md](./lvyjs-dev.md)
- 消息结构： [message-format.md](./message-format.md)
- 主技能入口： [../SKILL.md](../SKILL.md)
