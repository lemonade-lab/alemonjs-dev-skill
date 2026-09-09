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

- 读业务配置时优先 `getConfigValue()`
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

- 读配置优先走 `getConfigValue()` / `getConfig()`
- 配置变更监听优先 `onWatchConfigValue()`
- 周期任务优先 `setInterval` / `setCron`
- 一次性延迟任务优先 `setTimeout`
- 定时任务、轮询、超时控制默认考虑热重载和自动清理
