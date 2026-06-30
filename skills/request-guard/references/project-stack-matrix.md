# Project Stack Matrix

## Contents

1. How to use this matrix
2. SPA projects
3. SSR and hybrid projects
4. Mini-program and multi-end projects
5. GraphQL and SDK-style projects
6. Monorepo and platform projects
7. What not to do

## How To Use This Matrix

Do not choose the integration point from the framework name alone.
Choose it from the **shared request entry used by the app**.

The usual order is:

1. Find the request entry used by most business calls.
2. Check whether it is Axios-like or a Promise-returning function.
3. Install once at that shared layer.
4. Add the smallest useful rules for the user's real pain point.

## SPA Projects

### React / Vue / Vite / CRA / plain web app

Most common integration points:

- `src/api/http.ts`
- `src/utils/request.ts`
- `src/services/request.ts`
- shared `axios.create(...)` client
- shared `fetchJson(...)` helper

Recommended shape:

- shared Axios instance: `setupRequestGuard(http, options)`
- shared fetch helper or SDK function: `setupRequestGuard(requestFn, options)`

Typical starter rule:

```js
{
  method: 'post',
  duplicate: { strategy: 'block', message: 'Submitting...' }
}
```

## SSR And Hybrid Projects

### Next.js / Nuxt / Remix / hybrid apps

First decide which request runtime the user actually means:

- browser-side page/app requests
- server-side route handlers
- server actions or backend-only clients

Default recommendation:

- integrate the browser-side shared request module first
- do not assume private server plumbing should expose the same guidance unless the user explicitly
  wants that runtime integrated

Good integration points:

- client-side `lib/http.ts`
- browser `api/client.ts`
- shared frontend SDK used by pages/components

Be careful:

- do not install separately in many page modules
- do not mix multiple guard setups on the same request path when one shared client is enough

## Mini-Program And Multi-End Projects

### Taro / uni-app / WeChat mini-program / similar wrappers

These usually need wrapper mode around a Promise-style request helper.

For **WeChat mini-programs**, prefer the public platform entry instead of hand-writing the wrapper:

```js
import { createWechatRequestGuard } from '@hydd/request-guard/platforms/wechat';
import { duplicate, retry } from '@hydd/request-guard/capabilities';

export const request = createWechatRequestGuard(wx.request, {
  capabilities: [duplicate(), retry()],
  baseURL: 'https://api.example.com',
  getHeaders() {
    return { token: wx.getStorageSync('token') };
  },
  rules: [{ method: 'post', duplicate: true }]
});
```

For **Taro / uni-app** (or when you intentionally keep your own wrapper), wrap the Promise-style
helper yourself and pass it to the on-demand `core` entry to keep the bundle small:

```js
import { setupRequestGuard } from '@hydd/request-guard/core';
import duplicate from '@hydd/request-guard/capabilities/duplicate';

function request(config) {
  return new Promise((resolve, reject) => {
    uni.request({ ...config, success: resolve, fail: reject });
  });
}

const guardedRequest = setupRequestGuard(request, {
  capabilities: [duplicate()],
  rules: [{ method: 'post', duplicate: true }]
});
```

If the app already has `Taro.request`, `uni.request`, or `wx.request` wrapped in a shared helper,
install on that helper instead of every page-level call site.

#### Keeping `RequestTask` / `abort()` available (mini-program / RN)

`wx.request` / `Taro.request` / `uni.request` return a `RequestTask` (with `abort()`,
`onHeadersReceived`, etc.) **synchronously** when the request starts. A guarded request always
resolves to a `Promise`, so that task object is not the return value of `guardedRequest(...)`.
This is expected: the guard standardizes every call into a Promise so duplicate/retry/circuit-breaker
can compose. It does **not** require changing the package — surface the task from inside your own
request helper, which is where `wx.request` is actually called.

Recommended pattern: let callers pass a `onRequestTask` callback and hand the task back from the
helper:

```js
function request(config) {
  return new Promise((resolve, reject) => {
    const task = wx.request({
      ...config,
      success: resolve,
      fail: reject
    });
    // 把底层 task 交还业务，让需要中断的调用方拿到 abort 句柄。
    config?.onRequestTask?.(task);
  });
}

const guardedRequest = setupRequestGuard(request, options);

// 业务侧需要中断时：
let pending;
guardedRequest({ url: '/api/feed', onRequestTask: (task) => { pending = task; } });
// later: pending?.abort();
```

If a specific call must keep the raw `RequestTask` as the return value (e.g. it never needs
duplicate/retry and only needs `abort`), call the original un-guarded helper directly for that one
call instead of routing it through the guard.

#### Circuit-breaker persistence falls back to memory off-browser

`circuitBreaker.storage: 'session' | 'indexedDB'` only persists where `sessionStorage` / `indexedDB`
actually exist. In WeChat/Alipay mini-programs, React Native, and most SSR runtimes these globals are
absent, so the guard **automatically falls back to in-memory storage** (state is not persisted across
app restarts) and emits one warning through the configured `logger`
(`phase: circuitBreaker:storageDowngrade`, with `requestedStorage` / `effectiveStorage`). The default
is already `'memory'`, so this only matters if you explicitly opted into persistence. On these
platforms, keep `storage: 'memory'` (the default) and treat circuit-breaker state as per-session.

## GraphQL And SDK-Style Projects

### GraphQL clients

Do not explain or expose client internals.
Instead, locate one of these public integration points:

- shared fetcher
- request utility used by the GraphQL client
- SDK request bridge

Typical guidance:

- if the project has a `requestGraphQL(query, variables, options)` helper, wrap that helper
- if the signature is non-standard, add `resolveArgs` and `mapConfig`

### SDK-style request clients

If the project exposes a custom request signature such as:

- `client(method, url, data, options)`
- `sdk.request({ api, type, payload, extra })`

Use wrapper mode and preserve the original public signature.
Only add:

- `resolveArgs` when the function is not config-first
- `mapConfig` when field names are non-standard
- `getRequestGuard` when request-level guard config is nested elsewhere

## Monorepo And Platform Projects

### Shared frontend platform / multiple apps

Recommended order:

1. Find the shared request module consumed by the app packages.
2. Install the guard there if multiple apps truly share the same request semantics.
3. Otherwise install once per app-local request layer.
4. Use config-only bootstrap only when multiple later guarded clients in the same app bundle should
   share the same defaults and rules.

Good targets:

- shared frontend SDK package
- shared browser request client
- per-app request entry if apps have different request semantics

## What Not To Do

- Do not install the guard separately in every feature module.
- Do not wrap a request function that is already clearly behind a guarded shared client unless the
  behavior is intentionally different and verified.
- Do not start by explaining private internals, architecture, or source layout.
- Do not present unpublished capabilities or roadmap ideas as if users can rely on them today.
