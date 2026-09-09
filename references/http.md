# AlemonJS HTTP 路由参考

HTTP 能力通过 `defineChildren().register()` 显式返回 `koaRouter`。权限、审计、Body 解析等中间件直接挂在对应 router 上，不依赖隐式目录约定。

## 最小 API

```typescript
import KoaRouter from 'koa-router';
import bodyParser from 'koa-bodyparser';
import { defineChildren } from 'alemonjs';

const router = new KoaRouter({ prefix: '/demo' });
router.use(bodyParser());
router.use(async (ctx, next) => {
  ctx.state.requestId = crypto.randomUUID();
  await next();
});
router.get('/ping', ctx => {
  ctx.body = { code: 200, message: 'pong', requestId: ctx.state.requestId };
});

export default defineChildren({
  register() {
    return { koaRouter: router };
  }
});
```

## 生命周期与错误

- 只有进入 `ready` 的应用才会真正放行 HTTP 服务；未注册/未启用、初始化中、初始化失败、已卸载通常分别为 404、503、500、410。
- HTTP handler 出错会进入 `onHttpError({ ctx, error, appName, path, method, kind })`；已自行写入响应时返回 `'handled'`，避免框架追加默认 500。
- 插件必须先在 `alemon.config.yaml` 的 `apps` 中进入运行时管理，才应公开 HTTP 服务。

## 静态资源与安全

- `/app/` 默认服务应用公开 web 根目录；可在 `package.json` 的 `alemonjs.web.root` 指定构建产物目录。
- HTML、前端路由与 API 请求使用相对路径，避免部署在 `/app` 前缀下时解析为站点根路径。
- 框架会拦截静态路由的 `../` 路径遍历及非法 npm 包名；业务 router 仍要对身份、输入与资源授权负责。
