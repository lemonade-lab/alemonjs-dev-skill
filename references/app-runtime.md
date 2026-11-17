# AlemonJS 运行时参考

本文整理框架运行时常用能力，重点覆盖配置读取、配置监听、框架托管定时任务与生命周期清理。

## 配置读取

框架优先通过配置 API 读取 `alemon.config.yaml`，不要在业务代码里到处分散读配置文件。

最小配置示例：

```yaml
port: 17117
input: 'lib/index.js'
serverPort: 18110
autoPort: false # 端口冲突时是否自动递增
```

框架级配置还可按需使用：

- `apps`：启用并纳入运行时管理的子模块；模块配置以包名为 key。
- `processor.repeated_event_time` / `processor.repeated_user_time`：重复事件与连续用户消息的过滤窗口（毫秒）。
- `redirect_text_regular`、`redirect_text_target`、`mapping_text`：将输入文本重定向或映射后再处理。

用户可编辑的业务配置仍应放在应用自身的 `alemon.config.yaml` 命名空间内，不把它们混入环境变量或临时存储。

### getConfigValue

读取当前配置值对象。

```typescript
import { getConfigValue } from 'alemonjs';

const value = getConfigValue();
console.log(value.port);
```

推荐：

- 读取框架全局配置时使用 `getConfigValue()`。读取当前应用配置时使用下节的按包接口。
- 统一在配置模块里收敛读取，不在多个 handler 里重复取值

### getConfig

读取完整配置对象、包信息、启动参数，并支持写回。

```typescript
import { getConfig } from 'alemonjs';

const config = getConfig();
console.log(config.package);
console.log(config.argv);
console.log(config.value);
```

可用于：

- 读取包信息和运行参数
- 修改并保存配置
- 插件化场景下动态扩充配置项

### onWatchConfigValue

监听配置变化。

```typescript
import { onWatchConfigValue } from 'alemonjs';

const unwatch = onWatchConfigValue(value => {
  console.log(value.port);
});
```

推荐：

- 对热更新配置、白名单、开关类能力使用监听
- 需要取消监听时，保存并调用返回的 `unwatch`

## 按包配置和应用顺序

```yaml
apps:
  '@example/storage':
    order: 10
  '@example/business':
    enabled: true
    order: 20
'@example/business':
  greeting: '你好'
```

`apps` 支持布尔值和 `{ enabled, order, command }` 映射，旧的名称数组仍可用。`enabled` 默认 true，`order` 默认取声明位置，从小到大加载，同序号按声明顺序。应用配置内容放在同名顶层节。

`getAppConfigValue<T>()` 返回应用配置，`getAppConfig<T>()` 还提供 `value`、`saveValue()`、`onWatch()`、`reload()` 和 `dispose()`。从调用文件向上找带 `name` 的 `package.json`，查找到当前工作目录为止。定位失败返回空配置并跳过保存。`saveValue()` 替换该应用的整个配置节，`onWatch()` 只通知该节的变更。在 `onDispose()` 清理本应用的监听。

`language: en-US` 切换框架内置提示，默认 `zh-CN`。`withLanguage(language, callback)` 仅覆盖指定回调及其中的异步执行。应用回复由业务自己的文案控制。

## 热加载和启动排查

在机器人应用进程内按已安装的包名调用 `loadAppPackage(name)`、`reloadAppPackage(name)` 或 `unloadAppPackage(name)`。重载和卸载后重新加载需要 Node.js 22.15+ 与 ESM 包，构建完成后再触发。运行时操作即时生效，重启仍按 `apps` 加载。启动时禁用的应用可以手动挂载，框架按 `apps.order` 更新事件处理顺序。

重载会重新导入包内代码，外部依赖继续用现有模块实例。旧应用卸载后，新版完成 `onReady` 才接收事件。初始化失败会抛错并进入 `failed`，修复产物后可再次加载。框架清理登记的路由、订阅、定时任务、上下文和 Expose，应用自建的连接与原生计时器在 `onDispose()` 中清理。

CBP 握手先于应用初始化。发生 `transport_ready_timeout` 时检查主进程或 CBP 服务、连接地址和端口。出现「业务初始化较慢，尚未收到 app_ready」时，检查模块的入口、依赖和生命周期回调。

### QQ 扫码配置

QQ Bot 默认 WebSocket 模式下，缺少完整单机器人凭据且没有 `bots` 项时会发出扫码二维码。扫码授权会重置 AppSecret，旧部署需要更新凭据。认证后适配器将新凭据写入 `alemon.config.yaml` 的 `qq-bot` 节。已授权却再次出现二维码时，检查启动目录、配置文件路径和 `qq-bot` 与 `'@alemonjs/qq-bot'` 的覆盖关系。2.1.46 在有效配置完全未提供账号字段和 `bots` 时，可从原始配置节恢复完整凭据。Webhook 需手动配置。

[QQ 扫码登录参考](https://github.com/lemonade-lab/alemonjs/blob/3b582bf5abfc5af1397274f8d47f5661b4b8be3a/packages/qq-bot/QR_LOGIN.md)

## 发布产物

`alemonc publish` 在临时 Git 工作树构建当前 HEAD，只有已提交内容参与。`--dry-run` 构建并检查产物，在推送之前结束。`--skip-build` 使用已提交的产物。脚本优先级为 `package.json` 的 `alemonjs.build`、`bundle`、`build`。目标为 `release` 或以 `-release` 结尾时，命令将产物和版本标签一起推送。`--branch` 覆盖目标分支，调试时可先用 `--dry-run` 核对。

## 框架托管定时任务

优先使用框架提供的定时任务 API，而不是原生 `setInterval` / `setTimeout`。

原因：

- 插件卸载或热重载时会自动清理
- 任务可统一查看、暂停、恢复
- 更适合 AlemonJS 的应用生命周期

### setInterval / clearInterval

```typescript
import { setInterval, clearInterval } from 'alemonjs';

const id = setInterval(() => {
  console.log('tick');
}, 5000);

clearInterval(id);
```

### setTimeout / clearTimeout

```typescript
import { setTimeout, clearTimeout } from 'alemonjs';

const id = setTimeout(() => {
  console.log('done');
}, 10000);

clearTimeout(id);
```

### setCron

```typescript
import { setCron, clearInterval } from 'alemonjs';

const id = setCron('0 8 * * *', () => {
  console.log('good morning');
});

clearInterval(id);
```

常用 cron：

- `* * * * *`：每分钟
- `0 * * * *`：每小时整点
- `0 8 * * *`：每天早上 8 点
- `0 0 * * 1`：每周一凌晨

### 暂停、恢复、查看

```typescript
import { pauseSchedule, resumeSchedule, listSchedule } from 'alemonjs';

pauseSchedule(id);
resumeSchedule(id);
const all = listSchedule();
```

推荐：

- 调试复杂任务时用 `listSchedule()` 检查当前注册状态
- 需要短暂停运任务时用 `pauseSchedule()`，不要急着删除后重建

## 生命周期与自动清理

框架会自动识别任务归属并在插件卸载、热重载时清理关联任务。

推荐：

- 周期任务优先在 `onCreated()` 注册
- 不要默认手写全局定时器常驻进程
- 交互超时、轮询检查、每日刷新优先使用框架托管任务

## 默认运行时约束

- 读取或监听全局配置使用 `getConfig()` / `onWatchConfigValue()`。包级业务配置使用 `getAppConfig()`。
- 周期任务优先 `setInterval` / `setCron`
- 一次性延迟任务优先 `setTimeout`
- 定时任务、轮询、超时控制默认考虑热重载和自动清理
