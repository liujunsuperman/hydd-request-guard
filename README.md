<div align="center">

# @hydd/request-guard

**全平台可用、能力自由组合的前端请求守护层**

一套配置，管好所有请求。几行代码给项目加上系统级请求守护能力。

[![npm version](https://img.shields.io/npm/v/@hydd/request-guard.svg)](https://www.npmjs.com/package/@hydd/request-guard)
[![license](https://img.shields.io/npm/l/@hydd/request-guard.svg)](./LICENSE)
[![website](https://img.shields.io/badge/官网-happydidi.cn-blue)](https://happydidi.cn)

</div>

---

## 介绍

`@hydd/request-guard` 把散落在各处拦截器里的请求治理逻辑——防重在 A 拦截器、重试在 B 拦截器、熔断靠手动变量——统一收拢成一套**可声明、可组合、可观测**的治理管道，让你用几行代码给项目加上一整套系统级请求守护能力。

### 核心优势

- 🌐 **全平台可用** — axios / fetch / 小程序 / React Native 全平台通用，同一套配置跨平台复用
- 🧩 **能力自由组合** — 防重、重试、熔断按需引入，同时生效互不冲突
- 📐 **声明式规则** — 用 `rules` 声明哪些请求需要什么治理，业务代码零改动
- 📢 **统一观测出口** — 一个 `notify` 接 Toast，一个 `logger` 接开发日志，治理行为可追踪
- 🛡️ **安全无感** — 守护层只做加法不做减法，内置三层降级保护，永远不会拦住该发的请求

### 与手写拦截器的对比

| | 手写拦截器 | request-guard |
|---|---|---|
| 防重 + 重试 + 熔断一起用 | 各能力互相打架，需手动协调 | 自由组合，自动协调 |
| 跨平台 | axios 拦截器搬到 fetch/小程序就废 | 同一套配置，全平台通用 |
| 出问题排查 | 拦截器静默吞错，盲猜 | 统一日志出口，可追踪可上报 |
| 业务侵入 | 每个请求手动加 flag/变量 | 声明式规则，业务代码零改动 |

### 能力矩阵

| 治理能力 | 状态 | 说明 |
|----------|------|------|
| 防重复请求（Duplicate） | ✅ 已内置 | block 阻断或 reuse 复用重复请求 |
| 请求重试（Retry） | ✅ 已内置 | 失败请求按规则自动重试，指数退避、429 Retry-After 自动识别 |
| 熔断守护（Circuit Breaker） | ✅ 已内置 | 接口持续失败自动熔断，支持半开试探自动恢复 |
| 请求缓存（Cache） | 📋 规划中 | 响应级缓存与 stale-while-revalidate |
| 优先级队列（Queue） | 📋 规划中 | 请求排队与优先级调度 |
| 分组合并（Batch） | 📋 规划中 | 相邻请求合并发出 |

---

## 安装

```bash
npm install @hydd/request-guard
# 或
pnpm add @hydd/request-guard
# 或
yarn add @hydd/request-guard
```

---

## 快速开始

一行代码给所有 POST 请求加上防重保护：

```javascript
import axios from 'axios';
import { setupRequestGuard } from '@hydd/request-guard';

// 一行接入：所有 POST 请求自动防重，开发环境自动输出诊断日志
setupRequestGuard(axios, {
  dev: true,                                    // 开发环境输出日志（生产环境自动静默）
  rules: [{ method: 'post', duplicate: true }]  // 所有 POST 自动防重
});

// 业务代码无需任何改动
axios.post('/api/order/submit', { orderId: '12345' });
// 快速双击提交，第二次请求会被自动拦截，控制台输出 duplicate.triggered 日志
```

### 渐进式：加上提示和更多能力

```javascript
const requestManager = setupRequestGuard(axios, {
  dev: true,  // 开发环境输出诊断日志（不传 logger 时自动用内置日志）

  // 用户提示出口：防重命中、熔断拦截时通知用户
  notify: (payload) => Toast.show(payload.message),

  // 全局默认配置：各能力的默认参数
  defaults: {
    duplicate: { strategy: 'block', message: '请勿重复提交' },
    retry: { attempts: 3, delay: 200, backoff: 'exponential' }
  },

  // 规则：声明哪些请求自动启用什么治理
  rules: [
    { method: 'post', duplicate: true },        // POST 自动防重
    { method: 'get', retry: true }              // GET 失败自动重试
  ]
});
```

---

## 三大接入方式

| 你的情况 | 用这个 | 说明 |
|----------|--------|------|
| 我用 axios | Axios 安装模式 | 传一个 axios 实例，所有请求自动受保护 |
| 我用 fetch / 小程序 / 自定义 SDK | Wrapper 模式 | 把请求函数包一层，调用方式不变 |
| 我想在别处统一管规则 | 纯配置模式 | 不接管请求函数，只配置规则，多处共享 |

三种方式的 `rules`、`defaults`、`notify`、`logger` 写法完全一致，区别只在"如何接管请求函数"。

### Axios 安装模式（推荐）

```javascript
import axios from 'axios';
import { setupRequestGuard } from '@hydd/request-guard';

const requestManager = setupRequestGuard(axios, {
  dev: true,
  notify: (payload) => Toast.show(payload.message),
  rules: [{ method: 'post', duplicate: true }]
});

// requestManager 提供：configure / setRules / addRule / clearRules / clearState /
// getStateSnapshot / createLoadingKey / isLoading / subscribeLoading / circuitBreaker / uninstall
```

### Wrapper 模式（fetch / 小程序 / 自定义 SDK）

```javascript
import { setupRequestGuard } from '@hydd/request-guard/core';
import { duplicate, retry } from '@hydd/request-guard/capabilities';

// 把 wx.request 封装成 Promise 风格
function wxRequest(config) {
  return new Promise((resolve, reject) => {
    wx.request({ ...config, success: resolve, fail: reject });
  });
}

const guardedRequest = setupRequestGuard(wxRequest, {
  capabilities: [duplicate(), retry()],
  notify: (payload) => wx.showToast({ title: payload.message, icon: 'none' }),
  rules: [
    { method: 'post', duplicate: { strategy: 'block', message: '正在提交中' } },
    { method: 'get', retry: { attempts: 2, delay: 500 } }
  ]
});

// 使用方式和原来一样
await guardedRequest({
  url: '/api/order/submit',
  method: 'POST',
  data: { orderId: '12345' }
});
```

### 微信小程序平台快捷入口

不想手动封装 `wx.request`？用平台快捷入口：

```javascript
import { createWechatRequestGuard } from '@hydd/request-guard/platforms/wechat';
import { duplicate, retry } from '@hydd/request-guard/capabilities';

export const request = createWechatRequestGuard(wx.request, {
  capabilities: [duplicate(), retry()],
  baseURL: 'https://api.example.com',
  getHeaders() { return { token: wx.getStorageSync('token') }; },
  notify(payload) { wx.showToast({ title: payload.message, icon: 'none' }); },
  rules: [{ method: 'post', duplicate: true }]
});
```

### 纯配置模式

不接管请求函数，只设置全局共享配置，适合在路由守卫、应用初始化等入口统一管理规则：

```javascript
const requestManager = setupRequestGuard({
  notify: (payload) => Toast.show(payload.message),
  rules: [
    { method: 'post', duplicate: { strategy: 'block', message: '正在处理' } },
    { url: /\/api\/config/, retry: { attempts: 3, delay: 1000 } }
  ]
});

// 后续可动态管理
requestManager.addRule({ url: /\/api\/payment/, duplicate: { strategy: 'block' } });
requestManager.clearState();  // 路由切换时清空状态
```

---

## 全量入口与按需入口

`@hydd/request-guard` 是全量入口，默认安装 `duplicate`、`retry`、`circuitBreaker`。

小程序等包体积敏感项目从 `@hydd/request-guard/core` 接入，只打包真正用到的能力：

```javascript
import { setupRequestGuard } from '@hydd/request-guard/core';
import duplicate from '@hydd/request-guard/capabilities/duplicate';

const guardedRequest = setupRequestGuard(wxRequest, {
  capabilities: [duplicate()],
  rules: [{ method: 'post', duplicate: true }]
});
```

---

## 配置优先级

```
请求级配置（最高） > 规则匹配（首个命中生效） > 全局默认（最低）
```

- **全局默认**（`defaults`）放项目通用参数
- **规则匹配**（`rules`）按 URL / 方法批量声明治理策略
- **请求级**（`config.requestGuard`）给单个请求特殊处理，优先级最高，不受 `excludeMethods` 限制
- 任何时候 `requestGuard: false` 可完全绕过守护层

---

## 能力详解

### 防重复请求（Duplicate）

防止同一请求被重复发送。表单提交用 block 阻断，数据查询用 reuse 复用。

| 策略 | 首次请求 | 重复请求 | 适合场景 |
|------|----------|----------|----------|
| **block**（阻断） | 正常发送 | 直接拒绝，抛出 `RequestGuardBlockedError` | 表单提交、下单、支付 |
| **reuse**（复用） | 正常发送 | 等待首次请求结果，共享响应 | 列表查询、数据加载、配置查询 |

```javascript
// block：表单提交防双击
await axios.post('/api/order/submit', orderData, {
  requestGuard: {
    duplicate: {
      strategy: 'block',
      message: '订单提交中，请稍候',
      keyGenerator(config) { return `order:${config.data.orderId}`; }  // 以订单 ID 判重
    }
  }
});

// reuse：列表查询自动合并
const fetchList = (params) => axios.get('/api/list', {
  params,
  requestGuard: { duplicate: { strategy: 'reuse', compareFields: ['method', 'url', 'params'] } }
});
// 组件 A 和 B 同时发起相同查询，实际只发出 1 次请求，两者拿到相同响应
```

**全局默认配置**（`defaults.duplicate`）：

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| strategy | `'block'` | 默认策略 |
| compareFields | `['method','baseURL','url','params','data']` | 判重比较字段 |
| excludeMethods | `['GET','OPTIONS','HEAD']` | 规则匹配排除的方法 |
| compareHeaders | `[]` | 进入判重的 header 白名单 |
| hashAlgorithm | `'auto'` | 哈希算法（auto / cyrb53 / manual） |
| maxCacheSize | `100` | 单策略在途记录硬上限 |
| message | `'请勿重复提交'` | 默认提示文案 |
| keyGenerator | `null` | 自定义 key 生成函数 |

### 请求重试（Retry）

网络抖动、服务端瞬断时，自动重试失败请求，业务代码无需任何感知。

**默认行为**：
- `attempts: 3`（首次 + 2 次重试），`delay: 300` + `backoff: 'exponential'`
- 默认排除 POST/PUT/PATCH/DELETE（写操作不自动重试）
- 只重试白名单内的状态码：408, 429, 500, 502, 503, 504
- 429 自动解析 `Retry-After` header 覆盖延迟时间

```javascript
// 全局开启
setupRequestGuard(axios, {
  defaults: { retry: { attempts: 3, delay: 200, backoff: 'exponential' } }
});

// 请求级覆盖
await axios.get('/api/config', {
  requestGuard: { retry: { attempts: 5, delay: 500, backoff: 'linear' } }
});

// 自定义重试判断（canRetry 优先级最高）
await axios.get('/api/data', {
  requestGuard: {
    retry: {
      attempts: 3,
      canRetry({ status, code }) { return status === 503 || code === 'ECONNABORTED'; }
    }
  }
});
```

| 退避算法 | 公式 | 示例（delay=200） |
|------|------|------|
| `fixed` | `delay` | 200, 200, 200 |
| `linear` | `delay × attemptIndex` | 200, 400, 600 |
| `exponential` | `delay × 2^(attemptIndex-1)` | 200, 400, 800 |

### 熔断守护（Circuit Breaker）

接口持续失败时自动熔断，避免无意义请求打爆服务端，支持半开试探自动恢复。

```
正常 (NORMAL) → 连续失败达阈值 → 熔断 (CIRCUIT_BREAKER) → 等待 recoverDelay → 半开 (HALF_OPEN) → 试探成功 → 正常
```

```javascript
// 接口级熔断（默认）
setupRequestGuard(axios, {
  defaults: {
    circuitBreaker: {
      type: 'request',      // 接口级：每个 URL 独立计数
      failCount: 5,         // 连续失败 5 次触发
      window: 60000,        // 60 秒时间窗口
      recoverDelay: 30000,  // 30 秒后自动半开试探
      statusCodes: [500, 502, 503, 504]
    }
  }
});

// 域名级熔断：整个服务挂了时全停
requestManager.addRule({
  url: /\/api\/payment/,
  circuitBreaker: { type: 'domain', failCount: 3, recoverDelay: 60000, storage: 'session' }
});

// 手动控制
requestManager.circuitBreaker.getState('POST:/api/payment/create');  // 查看状态
requestManager.circuitBreaker.reset('POST:/api/payment/create');      // 手动重置
```

---

## 组合配置

多能力同时生效，互不冲突：

```javascript
setupRequestGuard(axios, {
  defaults: {
    duplicate: { strategy: 'block', message: '请勿重复提交' },
    retry: { attempts: 3, delay: 200, backoff: 'exponential' },
    circuitBreaker: { failCount: 5, window: 60000, recoverDelay: 30000 }
  },
  rules: [
    { method: 'post', duplicate: true },                                              // POST 防重
    { url: /\/api\/(list|search)/, method: 'get', duplicate: { strategy: 'reuse' }, retry: { attempts: 2 } }, // 查询复用+重试
    { url: /\/api\/(payment|order)/, circuitBreaker: { type: 'domain', failCount: 3 } } // 支付域熔断
  ]
});
```

> [!TIP]
> **默认 excludeMethods 互补设计**
> duplicate 默认排除 GET/OPTIONS/HEAD（读操作该重试不该去重），retry 默认排除 POST/PUT/PATCH/DELETE（写操作该去重不该盲目重试）。两者并集覆盖全部常见方法，单条规则同时配 `duplicate` 和 `retry` 时，任一方法都只会命中其中一个能力。需要同方法同时生效，用请求级显式配置。

---

## 请求级配置

对单个请求单独配置，优先级最高：

```javascript
// 防重 + 重试 + 熔断同时生效
await axios.post('/api/payment/create', data, {
  requestGuard: {
    duplicate: { strategy: 'block', message: '支付处理中...' },
    retry: { attempts: 2, delay: 1000 },
    circuitBreaker: { failCount: 3 }
  }
});

// 禁用单个能力
await axios.post('/api/special', data, { requestGuard: { duplicate: false } });

// 完全绕过守护层
await axios.post('/api/internal', data, { requestGuard: false });
```

---

## 规则系统

用 `rules` 声明"哪些请求需要什么治理"，按声明顺序首个命中生效：

```javascript
setupRequestGuard(axios, {
  rules: [
    { url: '/api/order/submit', duplicate: { strategy: 'block' } },           // 精确匹配
    { url: /\/submit$/, method: 'post', duplicate: true },                    // 正则 + 方法
    { url: /\/api\/(list|search)/, methods: ['get','post'], duplicate: { strategy: 'reuse' } }, // 多方法
    (config) => config.url.includes('payment')                                // 函数式规则
      ? { duplicate: { strategy: 'block', message: '支付处理中' } }
      : null,
    { match: (config) => config.headers?.['X-Idempotent'] === 'true',         // 自定义匹配器
      duplicate: { strategy: 'block' } }
  ]
});
```

---

## 消息与日志系统

```javascript
setupRequestGuard(axios, {
  dev: true,  // 不传 logger 时自动用内置开发日志
  notify: (payload) => Toast.show(payload.message)  // 用户提示出口
});
```

**notify 触发时机**：

| 触发 | capability | 说明 |
|------|-----------|------|
| duplicate 命中 | `duplicate` | block 拒绝 / reuse 等待均触发 |
| 熔断拦截 | `circuitBreaker` | 熔断打开，请求被拒绝 |
| 熔断状态变更 | `circuitBreaker` | 状态在 NORMAL/CIRCUIT_BREAKER/HALF_OPEN 间切换 |
| retry | — | 默认不触发 notify，仅走 logger |

**logger 事件**：

| 事件名 | 等级 | 说明 |
|--------|------|------|
| `duplicate.triggered` | warn | 命中重复请求 |
| `retry.scheduled` | info | 准备重试 |
| `retry.exhausted` | warn | retry 生命周期停止 |
| `circuitBreaker.blocked` | warn | 请求被熔断拦截 |
| `circuitBreaker.stateChange` | warn | 熔断状态变更 |
| `circuitBreaker.halfOpenProbe` | info | 半开放行试探 |
| `circuitBreaker.recovered` | info | 熔断恢复 |
| `requestGuard.internalError` | error | 守护层内部异常 |

需要接入自定义日志时，继承 `RequestGuardLogger` 重写 `write(event)`：

```javascript
import { RequestGuardLogger, setupRequestGuard } from '@hydd/request-guard';

class RemoteLogger extends RequestGuardLogger {
  write(event) {
    // event: { level, event, message, namespace, timestamp, data }
    monitor.report(event.event, event.data);
  }
}

setupRequestGuard(axios, { logger: new RemoteLogger({ level: 'warn' }) });
```

---

## 全局配置 API

`setupRequestGuard` 返回的 `RequestGuardController`：

| 方法 | 说明 |
|------|------|
| `configure(options)` | 更新全局配置（defaults / rules / notify / logger） |
| `use(capabilityDescriptor)` | 装配能力 descriptor |
| `setRules(rules)` / `addRule(rule)` / `clearRules()` | 管理规则 |
| `clearState()` | 清空所有能力状态（路由切换/登出时用） |
| `getStateSnapshot()` | 获取当前状态快照 |
| `createLoadingKey()` / `isLoading(key)` / `subscribeLoading(key, fn)` | 外部 UI loading 协作 |
| `circuitBreaker.tryRecover(key)` / `reset(key)` / `getState(key)` / `getAllStates()` | 熔断器手动控制 |
| `uninstall()` | 卸载守护，恢复原始行为（仅 Axios 模式） |

### 与外部按钮 loading 协作

让按钮的 loading 跟随这次请求的真实执行态，不用自己维护标志位、也不怕重试/异常时忘了复位。

**Vue 3**

```js
const loading = ref(false);
const submitKey = createLoadingKey('submit-order');

// 请求开始 loading=true，结束（含重试、报错）自动 false
subscribeLoading(submitKey, (v) => (loading.value = v));

// 发请求时带上 loadingKey
http.post('/api/order/submit', data, { requestGuard: { loadingKey: submitKey } });
```

**React**

```js
const [loading, setLoading] = useState(false);
const submitKey = useRef(createLoadingKey('submit-order')).current;

// 请求开始 loading=true，结束（含重试、报错）自动 false
useEffect(() => subscribeLoading(submitKey, setLoading), []);

// 发请求时带上 loadingKey
http.post('/api/order/submit', data, { requestGuard: { loadingKey: submitKey } });
```

> 并发引用计数、`isLoading(key)` 同步查询见文档 [Loading 协作 API](https://happydidi.cn/api/loading)。

---

## 安全降级

守护层只做加法不做减法，内置多层降级保护：

1. **全链路兜底** — 内部任何环节异常都被 catch 住，不会向业务代码抛出
2. **异常自动放行** — 守护层出问题时，请求直接正常发出，跟没装一样
3. **请求级开关** — `requestGuard: false` 随时绕过守护层
4. **配置传错降级** — 配置项拼错时静默降级为默认值，dev 环境告警

---

## 守护卸载与生命周期

| 函数 | 干什么 | 什么时候用 |
|------|--------|-----------|
| `clearState()` | 清空运行时状态 | 路由切换、组件销毁、用户登出 |
| `uninstall()` | 完全卸载守护层 | 退出应用、热更新重装 |
| `clearRules()` | 只清空规则，不动状态 | 临时关闭规则匹配 |

```javascript
// SPA 路由切换：清空状态，防止上一页的请求残留影响下一页
router.beforeEach(() => { requestManager.clearState(); });

// 用户登出：先清状态，再卸载
function logout() { requestManager.clearState(); requestManager.uninstall(); }

// 热更新重装
if (import.meta.hot) { import.meta.hot.dispose(() => requestManager.uninstall()); }
```

---

## 错误类型

推荐统一用 `error.name` 字符串判别：

| 错误名（`error.name`） | 所属能力 | 触发时机 |
|--------|----------|----------|
| `RequestGuardBlockedError` | duplicate | block 策略命中重复请求 |
| `RequestGuardCancelledError` | duplicate | reuse 等待方被 `clearState()` 取消 |
| `RequestGuardCacheEvictedError` | duplicate | 容量满拒绝新 key |
| `RequestGuardRetryCancelledError` | retry | waiting 状态被 `clearState()` 取消 |
| `RequestGuardCircuitBreakerError` | circuitBreaker | 熔断器打开时请求被拒绝 |

> [!WARNING]
> **短路错误不经过响应拦截器**
> duplicate block、熔断拦截等"请求未发出"的短路错误，**不会进入 `axios.interceptors.response.use`**。用 `notify` 出口或请求级 `.catch` 处理。

---

## 常见业务场景

### 电商下单防重

```javascript
setupRequestGuard(axios, {
  notify: (payload) => Toast.show(payload.message),
  defaults: { duplicate: { strategy: 'block', message: '订单处理中，请勿重复提交' } }
});

await axios.post('/api/order/create', orderData, {
  requestGuard: {
    duplicate: {
      strategy: 'block',
      keyGenerator(config) { return `order:create:${config.data?.orderId}`; }
    }
  }
});
```

### 列表查询复用 + 自动重试

```javascript
setupRequestGuard(axios, {
  rules: [{
    url: /\/api\/(list|search|query)/, method: 'get',
    duplicate: { strategy: 'reuse', compareFields: ['method', 'url', 'params'] },
    retry: { attempts: 3, delay: 500, backoff: 'exponential' }
  }]
});
// 相同查询只发 1 次请求，失败自动重试，所有等待方拿到相同结果
```

### 多租户 header 去重

```javascript
setupRequestGuard(axios, {
  defaults: { duplicate: { strategy: 'block', headerFields: ['X-Tenant-Id', 'X-Store-Id'] } },
  rules: [{ method: 'post', duplicate: true }]
});
// 不同租户的相同请求互不干扰
```

### 文件上传 FormData 去重

```javascript
setupRequestGuard(axios, {
  rules: [{
    url: /\/api\/upload/, method: 'post',
    duplicate: {
      strategy: 'block', message: '文件上传中，请勿重复提交',
      formDataResolver(data) {
        const entries = [];
        data.forEach((value, key) => {
          if (value instanceof File) entries.push(`${key}:${value.name}:${value.size}:${value.lastModified}`);
          else entries.push(`${key}:${value}`);
        });
        return entries.join('|');
      }
    }
  }]
});
```

---

## AI 接入 Skill（可选）

如果你使用 **Claude Code** 或 **Codex**，可以让 AI 直接按你的技术栈和业务场景帮你接入。本包内置了一个
Skill，会自动选择合适的接入方式（Axios 安装 / Wrapper / 微信平台入口 / 纯配置 / 按需 core 入口），生成
最小可用规则，并保留 `requestGuard: false` 等逃生口。Skill 只覆盖**公开接入能力**，不会泄露内部实现。

Skill 随 npm 包一起分发，安装后位于：

```text
node_modules/@hydd/request-guard/skills/request-guard/
```

把该目录拷到你的 AI 工具技能目录即可：

- Claude Code：`.claude/skills/request-guard/`（项目级）或 `~/.claude/skills/request-guard/`（用户级）
- Codex：`$CODEX_HOME/skills/request-guard/`

然后直接描述需求，例如：

```text
用 request-guard 给我的 React + axios 项目接入表单防重复
用 request-guard 保留我现有 sdk.request(method, url, data, options) 签名，接入支付防重
```

详细安装与用法见文档 [AI 接入 Skill](https://happydidi.cn/recipes/codex-skill)。

---

## License

[MIT](./LICENSE)
