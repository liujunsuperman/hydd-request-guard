# Integration Recipes

## Contents

1. Public API checklist
2. Axios install mode
3. Config-first wrapper mode
4. `request(url, config)` wrapper mode
5. Multi-argument SDK wrapper mode
6. WeChat platform mode
7. On-demand (core) mode
8. Config-only bootstrap mode
9. Verification checklist

## Public API Checklist

Default to these stable root exports:

```js
import {
  setupRequestGuard,
  createLoadingKey,
  isLoading,
  subscribeLoading,
  ConsoleRequestGuardLogger,
  RequestGuardLogger,
  CircuitBreakerError
} from '@hydd/request-guard';
```

These documented subpath entries are also public (declared in the package `exports` map) and are the
right choice for mini-programs and bundle-size-sensitive apps:

```js
// No built-in capability; pass an explicit capabilities array.
import { setupRequestGuard } from '@hydd/request-guard/core';
// Capability factories for on-demand assembly.
import { duplicate, retry, circuitBreaker } from '@hydd/request-guard/capabilities';
// Or a single-capability subpath to bundle only what you use.
import duplicate from '@hydd/request-guard/capabilities/duplicate';
// Ready-made WeChat mini-program entry.
import { createWechatRequestGuard } from '@hydd/request-guard/platforms/wechat';
```

Do not import from `src/` or any path not declared in the package `exports` map.

The public controller returned by `setupRequestGuard(...)` can:

- `configure(options)`
- `setRules(rules)`
- `addRule(rule)`
- `clearRules()`
- `clearState()`
- `getStateSnapshot()`
- `createLoadingKey(label?)`
- `isLoading(key)`
- `subscribeLoading(key, listener)`
- `uninstall()` only for Axios installations
- `circuitBreaker.tryRecover/reset/getState/getAllStates`

Always explain integration through these public surfaces only.
Do not answer by describing internal files, internal metadata, or private execution design.

## Loading-State Coordination

Use this only when the user explicitly wants UI state such as:

- button loading
- dialog confirm busy state
- local section skeleton/spinner tied to one guarded request

Recommended pattern:

```js
import axios from 'axios';
import {
  createLoadingKey,
  setupRequestGuard,
  subscribeLoading
} from '@hydd/request-guard';

setupRequestGuard(axios, {
  rules: [{ method: 'post', duplicate: true }]
});

const submitLoadingKey = createLoadingKey('submit-order');

const unsubscribe = subscribeLoading(submitLoadingKey, (loading) => {
  setSubmitLoading(loading);
});

await axios.post('/api/order/submit', payload, {
  requestGuard: {
    duplicate: true,
    loadingKey: submitLoadingKey
  }
});

unsubscribe();
```

Notes:

- Prefer an explicit `loadingKey` over recomputing request identity from UI code.
- Use `requestGuard.loadingKey` only when the request should drive external UI state.
- Keep this additive: do not replace the project's own domain state model if it already has a
  better source of truth.

## Axios Install Mode

Use this when the project already exports a shared Axios instance.

```js
import axios from 'axios';
import { setupRequestGuard, ConsoleRequestGuardLogger } from '@hydd/request-guard';
import { showToast } from './toast';

export const http = axios.create({
  baseURL: '/api'
});

setupRequestGuard(http, {
  notify: ({ message }) => showToast(message),
  logger: new ConsoleRequestGuardLogger({ devOnly: true }),
  rules: [
    {
      method: 'post',
      duplicate: { strategy: 'block', message: 'Request is already in progress.' }
    }
  ]
});
```

Notes:

- Install once at the shared client definition, not in individual feature modules.
- If the repo has multiple Axios clients, either install on each client or bootstrap shared config
  first with config-only mode and then install each client.
- If the project is sensitive to console noise, use `logger: null` instead of the console logger.
- If the project already routes all requests through a guarded shared client, do not wrap that same
  path again just to add another layer. Extend rules or request-level config instead.

## Config-First Wrapper Mode

Use this when the project has a promise-returning request function whose first argument is already a
standard config object or close enough to it.

```js
import { setupRequestGuard } from '@hydd/request-guard';

async function request(config) {
  const response = await fetch(config.url, {
    method: config.method,
    headers: config.headers,
    body: config.data ? JSON.stringify(config.data) : undefined
  });

  return response.json();
}

export const guardedRequest = setupRequestGuard(request, {
  rules: [
    { method: 'post', duplicate: { strategy: 'block', message: 'Submitting...' } }
  ]
});
```

Use this form for `fetch`, `wx.request` promise wrappers, or custom request helpers that already
work with a single config object.

This is often the right fit for:

- shared `fetchJson` helpers
- browser SDK wrappers
- mini-program Promise adapters
- GraphQL fetcher helpers

## `request(url, config)` Wrapper Mode

Use `resolveArgs` when the function signature splits `url` and `config`.

```js
import { setupRequestGuard } from '@hydd/request-guard';

async function request(url, config = {}) {
  return fetch(url, {
    method: config.method,
    headers: config.headers,
    body: config.data ? JSON.stringify(config.data) : undefined
  }).then((res) => res.json());
}

export const guardedRequest = setupRequestGuard(request, {
  resolveArgs(args) {
    const [url, config = {}] = args;
    return {
      args,
      config: { url, ...config }
    };
  },
  rules: [
    { method: 'post', duplicate: { strategy: 'block', message: 'Submitting...' } }
  ]
});
```

`resolveArgs` is enough when the request-specific guard config still lives at `config.requestGuard`
and the merged config can be passed straight back to the original function.

## Multi-Argument SDK Wrapper Mode

Use this when the request layer looks like `client(method, url, data, options)` or when field names
are not standard.

```js
import { setupRequestGuard } from '@hydd/request-guard';

async function httpClient(method, url, data, options = {}) {
  return sdk.request(method, url, data, options);
}

export const guardedClient = setupRequestGuard(httpClient, {
  resolveArgs(args) {
    const [method, url, data, options = {}] = args;

    return {
      args,
      config: {
        method,
        url,
        data,
        headers: options.headers,
        requestGuard: options.requestGuard
      },
      configIndex: 3,
      toRequestConfig(guardedConfig) {
        return {
          ...options,
          headers: guardedConfig.headers,
          requestGuard: guardedConfig.requestGuard
        };
      }
    };
  },
  rules: [
    { method: 'post', duplicate: { strategy: 'block', message: 'Submitting...' } }
  ]
});
```

When the business config uses non-standard field names, add `mapConfig` and `getRequestGuard`:

```js
const guardedSdk = setupRequestGuard(sdkRequest, {
  resolveArgs(args) {
    return { args, config: args[0] };
  },
  mapConfig(config) {
    return {
      url: config.api,
      method: (config.type || 'GET').toUpperCase(),
      data: config.payload,
      headers: config.extra?.headers
    };
  },
  getRequestGuard(config) {
    return config.extra?.guard;
  }
});
```

Add `onBeforeRequest`, `onAfterResponse`, or `onAfterError` only when the project already has a
real need such as token injection, response normalization, or error reporting.

Prefer adapting the user's existing public SDK signature over replacing it with a new config shape.

## WeChat Platform Mode

Use this for WeChat mini-programs instead of hand-writing a `wx.request` Promise wrapper. The public
`createWechatRequestGuard` entry promisifies `wx.request`, normalizes `header`/`headers`, and accepts
platform-level `baseURL` and `getHeaders` helpers.

```js
import { createWechatRequestGuard } from '@hydd/request-guard/platforms/wechat';
import { duplicate, retry } from '@hydd/request-guard/capabilities';

export const request = createWechatRequestGuard(wx.request, {
  capabilities: [duplicate(), retry()],
  baseURL: 'https://api.example.com',
  getHeaders() {
    return { token: wx.getStorageSync('token') };
  },
  notify(payload) {
    wx.showToast({ title: payload.message, icon: 'none' });
  },
  rules: [
    { method: 'post', duplicate: { strategy: 'block', message: '请勿重复提交' } },
    { method: 'get', retry: true }
  ]
});

await request({ url: '/order/submit', method: 'POST', data: { orderId: '12345' } });
```

Notes:

- `baseURL` and `getHeaders` are platform-level conveniences for request mapping only; they are not
  core capability config.
- `createWechatRequestGuard` is built on the on-demand `core` runtime, so pass `capabilities`
  explicitly to control exactly what gets bundled.
- A guarded call always returns a `Promise`, not the raw `RequestTask`. If a specific call needs the
  task's `abort()`, surface it from inside your own helper or call the un-guarded helper for that one
  request (see project-stack-matrix.md).

## On-Demand (core) Mode

Use this when bundle size matters — typically mini-programs — and you do not want the full entry that
auto-installs `duplicate`, `retry`, and `circuitBreaker`. Import `setupRequestGuard` from
`@hydd/request-guard/core` and pass an explicit `capabilities` array.

```js
import { setupRequestGuard } from '@hydd/request-guard/core';
import duplicate from '@hydd/request-guard/capabilities/duplicate';

export const guardedRequest = setupRequestGuard(wxRequest, {
  capabilities: [duplicate()],
  defaults: {
    duplicate: { message: '请勿重复提交' }
  },
  rules: [{ method: 'post', duplicate: true }]
});
```

Notes:

- `core` installs no capability by default; if `rules` / `defaults` / `requestGuard` reference a
  capability that was not assembled, the dev build warns and ignores that config.
- Keep the capability factories (`duplicate()` / `retry()`) for declaring **what to install**, and
  put project default parameters in `defaults.<capability>`.

## Config-Only Bootstrap Mode

Use this when multiple later installations should share one set of defaults and rules.

```js
import { setupRequestGuard, ConsoleRequestGuardLogger } from '@hydd/request-guard';

setupRequestGuard({
  logger: new ConsoleRequestGuardLogger({ devOnly: true }),
  rules: [
    { method: 'post', duplicate: { strategy: 'block', message: 'Submitting...' } },
    { url: /\/config\//, retry: { attempts: 2, delay: 300, backoff: 'exponential' } }
  ]
});

export const guardedFetch = setupRequestGuard(fetchRequest);
export const guardedSdk = setupRequestGuard(sdkRequest, { resolveArgs });
```

Later `setupRequestGuard(...)` calls in the same app bundle reuse this shared guard configuration, so
bootstrap shared defaults and rules early if multiple guarded clients should behave consistently.

## Verification Checklist

- Confirm the installation lives in one central request-layer file.
- Exercise one guarded request that should trigger the chosen capability.
- Exercise one request with `requestGuard: false` and confirm it bypasses the guard path.
- If Axios mode was installed, confirm `uninstall()` is available on the returned controller.
- If wrapper mode was installed, confirm the wrapped function keeps the original business signature.
- Confirm the answer and code changes stay on public behavior and do not leak private implementation
  details.
