---
name: lvyjs-dev
description: 'lvyjs 构建工具配置与开发流程。USE FOR: 配置 lvy.config.ts、路径别名、静态资源导入、样式文件处理、PostCSS/Tailwind 配置、开发热重载、生产构建、tsconfig 继承、项目脚手架搭建。'
---

# lvyjs — ALemonJS 开发构建工具

基于 tsx + rollup 构建的 Node.js 应用开发与打包工具，是 ALemonJS 项目的标准构建工具链。

## 核心命令

```bash
npx lvy <入口文件>          # 开发模式（tsx 实时编译 + 热重载）
npx lvy <入口文件> --jsxp   # 开发模式 + jsxp 图片预览服务
npx lvy build               # 生产构建（rollup 打包 src → lib）
```

对应 package.json scripts：

```json
{
  "dev": "npx lvy app.ts",
  "view": "npx lvy app.ts --jsxp",
  "build": "npx lvy build"
}
```

## 入口文件 (app.ts)

lvyjs 的入口文件不是 `src/index.ts`，而是项目根目录的 `app.ts`：

```typescript
// app.ts
import { start } from 'alemonjs';
import { createServer } from 'jsxp';

if (process.argv.includes('--jsxp')) {
  void createServer(); // 启动 jsxp 图片预览服务
} else {
  start('src/index.ts'); // 启动 ALemonJS 应用
}
```

生产环境入口 `index.js`：

```javascript
// index.js
import { start } from 'alemonjs';
start(); // 使用 lib/ 下的编译产物
```

## lvy.config.ts 配置详解

```typescript
import { defineConfig } from 'lvyjs';
import { dirname, join } from 'path';
import { fileURLToPath } from 'url';
const __dirname = dirname(fileURLToPath(import.meta.url));

export default defineConfig({
  // 环境变量注入到 process.env
  env: {
    MY_VAR: 'value'
  },

  // 路径别名
  alias: {
    entries: [
      { find: '@src', replacement: join(__dirname, 'src') }
    ]
  },

  // 静态资源识别（默认: .png|.jpg|.jpeg|.gif|.svg|.webp|.ico）
  assets: {
    filter: /\.(png|jpg|jpeg|gif|svg|webp|ico|yaml|txt|ttf|md)$/
  },

  // 样式文件识别（默认: .css|.scss|.less|.sass）
  styles: {
    filter: /\.(css|scss|less|sass)$/
  },

  // 文件监听（开发模式自动重启）
  watch: ['src/**/*.{ts,tsx,js,jsx,json,html}'],
  // 或完整写法：
  // watch: {
  //   paths: ['src/**/*.yaml', 'config'],
  //   delay: 1000  // 防抖延迟 ms，默认 500
  // },

  // 生产构建配置
  build: {
    typescript: {
      include: ['src/**/*.ts', 'src/**/*.tsx'],
      outDir: 'lib',
      removeComments: true,
      declaration: true
    },
    commonjs: {},           // CJS 处理，设为 false 禁用
    RollupOptions: {},      // 自定义 Rollup 配置
    OutputOptions: {
      input: 'src',         // 输入目录，默认 src
      dir: 'lib'            // 输出目录，默认 lib
    }
  }
});
```

配置文件查找优先级：`lvy.config.ts` > `lvy.config.js` > `lvy.config.mjs` > `lvy.config.cjs` > `lvy.config.tsx`

## tsconfig.json

继承 lvyjs 内置的 TypeScript 配置：

```json
{
  "compilerOptions": {
    "paths": {
      "@src/*": ["./src/*"]
    }
  },
  "include": ["src/**/*", "jsxp.config.tsx"],
  "extends": "lvyjs/tsconfig.json"
}
```

类型声明文件 `src/env.d.ts`：

```typescript
/// <reference types="lvyjs/env" />
/// <reference types="alemonjs/env" />
```

## 静态资源导入

导入图片等静态资源时返回文件的**绝对路径**（string 类型）：

```typescript
import { readFileSync } from 'fs';
// 返回绝对路径
import img_logo from '@src/assets/logo.png';
const data = readFileSync(img_logo);
```

在 JSX 图片组件中使用：

```tsx
import { BackgroundImage } from 'jsxp';
import img_logo from '@src/assets/alemonjs.png';

<BackgroundImage url={img_logo} size="100% auto">
  <div>内容</div>
</BackgroundImage>
```

## 样式文件导入

导入样式文件时返回编译后 CSS 文件的**绝对路径**：

```typescript
import { readFileSync } from 'fs';
import cssURL from '@src/assets/main.css';
const cssContent = readFileSync(cssURL, 'utf-8');
```

在 JSX 组件中通过 jsxp 的 `LinkStyleSheet` 引入：

```tsx
import { LinkStyleSheet } from 'jsxp';
import css_output from '@src/assets/main.css';

export default function Html({ children }) {
  return (
    <html>
      <head>
        <LinkStyleSheet src={css_output} />
      </head>
      <body>{children}</body>
    </html>
  );
}
```

### CSS 中的引用

```css
/* 支持别名路径 */
@import url('@src/assets/root.css');
/* 支持相对路径 */
@import url('./other.css');
```

SCSS/LESS 需额外安装：`npm install less sass -D`

## PostCSS 配置

通过 `postcss.config.mjs` 配置，内置支持 `postcss-import`、`postcss-url`、`autoprefixer`：

```javascript
// postcss.config.mjs
export default {
  plugins: {
    tailwindcss: {},
    cssnano: { preset: 'default' }
  }
};
```

## Tailwind CSS 配置

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
export default {
  content: ['./src/**/*.{jsx,tsx}']
};
```

标准 CSS 入口文件模板（`src/assets/main.css`）：

```css
@import url('@src/assets/root.css');
@tailwind base;
@tailwind components;
@tailwind utilities;

body {
  @apply m-0 p-0 flex flex-col;
}
```

## jsxp 图片预览配置

```tsx
// jsxp.config.tsx
import React from 'react';
import { defineConfig } from 'jsxp';
import MyComponent from '@src/image/component/help';

export default defineConfig({
  routes: {
    '/': {
      component: <MyComponent data="预览数据" />
    }
  }
});
```

使用 `npm run view` 启动预览服务（默认 http://localhost:8080），可在浏览器中实时查看图片组件效果。

## 生产部署

### 构建

```bash
npm run build    # src/ → lib/
```

### PM2 部署

```yaml
# alemon.config.yaml
pm2:
  apps:
    - name: 'qq-bot'
      script: 'node index.js --login qq-bot'
      env:
        NODE_ENV: 'production'
```

```bash
npm run start    # npx pm2 startOrRestart pm2.config.cjs
npm run stop     # npx pm2 stop pm2.config.cjs
npm run delete   # npx pm2 delete pm2.config.cjs
```

## VS Code 调试配置

项目模板包含 `.vscode/launch.json`：

```json
{
  "configurations": [
    {
      "name": "dev",
      "type": "node",
      "request": "launch",
      "runtimeArgs": ["dev"],
      "runtimeExecutable": "yarn"
    },
    {
      "name": "view",
      "type": "node",
      "request": "launch",
      "runtimeArgs": ["dev", "--view"],
      "runtimeExecutable": "yarn",
      "preLaunchTask": "open-browser"
    }
  ]
}
```

## 关键规则

- **入口文件是 `app.ts`**（开发模式）和 `index.js`（生产模式），不是 `src/index.ts`
- **静态资源导入返回绝对路径字符串**，不是文件内容或 URL
- **样式导入返回编译后 CSS 文件路径**，不是样式内容
- **路径别名 `@src`** 必须同时在 `lvy.config.ts` 和 `tsconfig.json` 中配置
- **`watch` 配置** 仅在开发模式有效，用于非 TS/TSX 文件的变化检测
- **`build.typescript` 选项** 直接传递给 `@rollup/plugin-typescript`
- **配置文件中** 使用 `fileURLToPath(import.meta.url)` 获取 `__dirname`（ESM 环境）
