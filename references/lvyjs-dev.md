# lvyjs 开发构建速查

## 命令

```bash
npx lvy app.ts         # 开发模式
npx lvy app.ts --jsxp  # 开发 + 图片预览
npx lvy build          # 生产构建 (src -> lib)
```

## 推荐 scripts

```json
{
  "dev": "npx lvy app.ts",
  "view": "npx lvy app.ts --jsxp",
  "build": "npx lvy build"
}
```

## 最小配置

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
  styles: {
    filter: /\.(css|scss|less|sass)$/
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

## 推荐 app.ts 内容

```ts
import { start } from 'alemonjs';
import { createServer } from 'jsxp';
// --jsxp 启动 jsxp 服务
if (process.argv.includes('--jsxp')) {
  void createServer();
} else {
  // 如果脚本是js写的，用 src/index.js
  start('src/index.ts');
}
```

## 关键规则

- 静态资源导入返回绝对路径字符串。
- 样式导入返回编译后 CSS 文件路径。
- `@src` 别名需同时配置在 `lvy.config.ts` 与 `tsconfig.json`。
- 开发入口是 `app.ts`，生产入口是 `index.js`。

## 生态协同

- JSX 图片设计规范见：[jsxp-dev.md](./jsxp-dev.md)
- 主技能入口见：[../SKILL.md](../SKILL.md)
