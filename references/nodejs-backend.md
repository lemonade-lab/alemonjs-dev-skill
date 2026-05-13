# Node.js 后端通用参考

本文面向 AlemonJS 业务开发时常见的 Node.js 后端工程问题，强调通用常识与默认判断，不绑定具体 ORM、HTTP 框架或存储实现。

## 适用范围

- 命令处理、订阅回调、定时任务、平台事件消费
- 时间计算、冷却、签到、过期、重试、限流
- 配置加载、参数校验、错误处理、日志埋点
- 数据写入、一致性控制、外部接口调用

## 默认工程判断

- 默认假设所有外部输入都不可信。
- 默认假设所有外部 IO 都可能超时、失败、重复、返回脏数据。
- 默认把业务规则写成可复用工具或中间件，不散落在多个 handler。
- 默认先定义边界条件，再写 happy path。
- 默认优先保证可维护和可审计，其次才是少写几行代码。

## 时间与时区

时间逻辑优先使用 `dayjs`，不要在业务里散落 `Date` 原生细节和手写字符串比较。

推荐依赖与插件：

```typescript
import dayjs from 'dayjs';
import utc from 'dayjs/plugin/utc';
import timezone from 'dayjs/plugin/timezone';
import duration from 'dayjs/plugin/duration';
import customParseFormat from 'dayjs/plugin/customParseFormat';
import isSameOrAfter from 'dayjs/plugin/isSameOrAfter';
import isSameOrBefore from 'dayjs/plugin/isSameOrBefore';

dayjs.extend(utc);
dayjs.extend(timezone);
dayjs.extend(duration);
dayjs.extend(customParseFormat);
dayjs.extend(isSameOrAfter);
dayjs.extend(isSameOrBefore);
```

默认规则：

- 存储、比较、过期判断优先使用时间戳或统一时区的 `dayjs` 对象。
- 展示给用户时再格式化，不在业务判断里比较格式化后的字符串。
- 涉及“今天/明天/本周/月初/月末”时，必须先定义业务时区。
- 涉及每日刷新、签到重置、冷却结束时，统一使用 `startOf/endOf` 表达边界。
- 不手写 `24 * 60 * 60 * 1000` 这类魔法时长，统一封装常量或使用 `dayjs.duration()`。
- 用户输入时间时先校验格式，再解析。

推荐：

```typescript
const now = dayjs().tz('Asia/Shanghai');
const resetAt = now.endOf('day');
const expired = dayjs(expireAt).isSameOrBefore(now);
const cooldownMs = dayjs.duration(10, 'minute').asMilliseconds();
```

避免：

```typescript
const expired = new Date(expireAt).toDateString() === new Date().toDateString();
const next = Date.now() + 24 * 60 * 60 * 1000;
```

常见判断：

- 每日次数重置：按业务时区的 `startOf('day')` 或 `endOf('day')`
- 周期性奖励：按明确时区的 `startOf('week')`，不要依赖宿主默认 locale
- 冷却时间：保存结束时间戳或最近执行时间戳，不保存“剩余秒数”
- 任务超时：比较绝对时间，不比较累计字符串状态

## 输入校验与类型收敛

所有进入 handler 的数据都应尽快变成“可直接使用”的窄类型。

默认规则：

- 路由参数优先走 `schema`，不要在 handler 里重复拆字符串。
- 数字、布尔、枚举、时间、用户 ID 都要显式解析。
- 区分空字符串、`undefined`、`null`、`NaN`。
- 分页、limit、时间范围、批量输入都要设上限。
- 第三方接口返回值进入业务前先做字段存在性和类型判断。

推荐：

```typescript
const [event] = useEvent();

const uid = String(event.current.__route?.params?.uid ?? '').trim();
if (!uid) return false;

const page = Number(event.current.__route?.params?.page ?? 1);
if (!Number.isInteger(page) || page < 1 || page > 100) return false;
```

避免：

```typescript
const page = +event.current.__route?.params?.page || 1;
```

## 错误处理

错误处理的目标不是“别报错”，而是“报错时能定位、能降级、对用户稳定”。

默认规则：

- 预期错误与系统错误分开处理。
- 给用户的错误文案保持稳定，不暴露底层堆栈和敏感信息。
- `try/catch` 放在明确边界上，不要把整个模块全部吞掉。
- 日志记录业务上下文，如路由、用户、参数摘要、耗时。
- 只对幂等场景做重试。

推荐模式：

```typescript
const [event] = useEvent();

try {
  await service.run(input);
} catch (error) {
  logger.error({
    err: error,
    route: event.current.__route?.key,
    userId: event.current.UserId
  }, 'service run failed');

  await message.send({
    format: Format.create().addText('处理失败，请稍后重试')
  });
}
```

## 配置与环境变量

配置应集中读取、集中校验，不要在业务代码里到处直接访问 `process.env`。

默认规则：

- 密钥、域名、开关、超时、重试次数、时区都走配置层。
- 启动时校验必填配置，不要等到运行时报错。
- 默认值要保守，危险能力默认关闭。
- 不把环境变量解析逻辑散落在 handler 和 service 里。

推荐：

- 建一个统一 `config` 模块导出已校验配置。
- 时区作为显式配置项，而不是依赖宿主机器默认设置。
- 超时、重试、限流参数统一收口。

## 幂等、并发与状态一致性

机器人、回调、订阅、定时任务都要默认考虑重复触发和并发写入。

默认规则：

- 写操作先判断幂等键或唯一约束。
- 涉及次数、库存、资格、签到、奖励发放时，优先使用原子更新。
- 不依赖“先查再写”保证正确性，除非有锁或事务边界。
- 定时任务和平台回调默认可能重复投递。
- 重试前先确认操作是否幂等。

高风险场景：

- 签到奖励重复发放
- 扣次数和发奖励分离导致部分成功
- 重复消费同一事件
- 多请求同时更新同一份用户状态

## 日志与可观测性

日志的目标是支撑排障，不是简单打印“执行到这里了”。

默认规则：

- 关键入口记录结构化日志。
- 外部调用记录目标、耗时、结果状态。
- 不记录密钥、token、完整敏感载荷。
- 高频路径避免打印大对象和大数组。
- 错误日志优先带上下文字段，而不是只打一句字符串。

建议字段：

- `route`
- `eventType`
- `userId`
- `requestId` 或事件唯一键
- `durationMs`
- `status`

## 异步 IO 与外部依赖

默认把数据库、HTTP、文件、缓存都当作不稳定依赖处理。

默认规则：

- 所有外部 IO 都设置超时。
- 只对可重试错误做有限重试，并使用退避。
- 批量请求和并发请求要限流，不无上限 `Promise.all(...)`。
- 外部依赖返回结果进入业务前先做收敛和降级判断。
- 资源句柄、订阅、定时器、临时文件要明确释放。

## 数据与查询习惯

即使不绑定数据库框架，也应保持基本工程判断。

默认规则：

- 查询字段最小化，不默认全量字段。
- 分页要有限制，排序要明确。
- 创建时间、更新时间、删除状态要有统一语义。
- 唯一约束类业务不要只靠代码判断。
- 批处理优先使用批量接口，不循环单条写入。

## 安全基础

这是后端默认常识，不应当依赖调用方自觉。

默认规则：

- 不信任任何客户端输入。
- 不拼接 SQL、shell、路径、重定向地址。
- 上传和下载路径要校验。
- 敏感操作先鉴权再执行。
- 对高频命令、登录态接口、回调入口考虑限流和防刷。

## 与 AlemonJS 业务结合时的建议

- 命令参数优先放在 `group.use({ schema })`，不要在 handler 里再拆一遍消息文本。
- 时间、冷却、过期、签到、每日重置建议抽成独立 `time` 工具模块。
- 频控、鉴权、日志、幂等判断优先中间件化。
- 对外回复文案保持稳定，内部错误细节只进日志。
- 平台字段兼容留在适配器层，业务层只消费统一事件字段。
