# User Scenario Playbook

## Contents

1. How to use this playbook
2. Submit, save, pay, and checkout
3. Search, list, detail, and preview
4. Upload and import
5. Config, bootstrap, and initialization
6. Unstable upstream or repeated 5xx
7. Shared SDK and platform rollout
8. Opt-out and local override
9. Safe response shape

## How To Use This Playbook

When the user asks for help, start from their business language, not from package jargon.

Good first questions to answer internally:

- What user action is hurting today?
- Where is the shared request entry for that action?
- What is the smallest public config that solves it?

Then answer with:

1. where to integrate
2. what starter config to use
3. how to opt out per request

## Submit, Save, Pay, And Checkout

Common user wording:

- "防止重复提交"
- "支付按钮别重复点"
- "保存时避免多次请求"
- "下单接口做防重"

Recommended first move:

- capability: `duplicate`
- starter strategy: `block`
- integration point: shared Axios client or shared request wrapper

Starter rule:

```js
{
  method: 'post',
  duplicate: {
    strategy: 'block',
    message: 'Request is already in progress.'
  }
}
```

When to go further:

- add a `keyGenerator` when business identity matters more than the full request body
- add a `loadingKey` when the user also wants button/dialog loading to follow the guarded request
- add idempotency header only when the backend or gateway supports it

Public loading-state pattern:

```js
import { createLoadingKey, subscribeLoading } from '@hydd/request-guard';

const submitLoadingKey = createLoadingKey('submit-order');

subscribeLoading(submitLoadingKey, (loading) => {
  setSubmitLoading(loading);
});

await request({
  url: '/api/order/submit',
  method: 'POST',
  data: formData,
  requestGuard: {
    duplicate: {
      strategy: 'block'
    },
    loadingKey: submitLoadingKey
  }
});
```

## Search, List, Detail, And Preview

Common user wording:

- "列表频繁刷新"
- "搜索接口别重复发"
- "详情页连续打开会打很多次"

Recommended first move:

- capability: `duplicate`
- starter strategy: `reuse`
- integration point: shared read-request helper

Starter rule:

```js
{
  url: /\/search|\/list|\/detail|\/preview/,
  duplicate: {
    strategy: 'reuse'
  }
}
```

Use `reuse` only when duplicate callers should share the same result instead of failing loudly.

## Upload And Import

Common user wording:

- "上传别重复点"
- "导入接口防重"
- "大文件上传避免重复提交"

Recommended first move:

- capability: `duplicate`
- starter strategy: `block`
- extra caution: only add idempotency header if server support is real

Starter rule:

```js
{
  url: /\/upload|\/import/,
  duplicate: {
    strategy: 'block',
    message: 'Upload is already in progress.'
  }
}
```

## Config, Bootstrap, And Initialization

Common user wording:

- "初始化接口偶发失败"
- "配置拉取想自动重试"
- "字典接口网络波动时兜一下"

Recommended first move:

- capability: `retry`
- scope it narrowly to read-safe requests

Starter rule:

```js
{
  url: /\/config|\/bootstrap|\/dictionary/,
  retry: {
    attempts: 2,
    delay: 300,
    backoff: 'exponential'
  }
}
```

Avoid turning on broad retry for all write requests.

## Unstable Upstream Or Repeated 5xx

Common user wording:

- "某个上游挂了会一直打"
- "失败时前端疯狂重试"
- "风险服务波动时想保护一下"

Recommended first move:

- capability: `circuitBreaker`
- target only the affected upstream or endpoint group

Starter rule:

```js
{
  url: /\/risk|\/core-upstream/,
  circuitBreaker: {
    failCount: 3,
    window: 30000,
    recoverDelay: 10000
  }
}
```

Do not present circuit breaker as a blanket default for every request in the app.

## Shared SDK And Platform Rollout

Common user wording:

- "这是平台能力，很多项目都要接"
- "我们有一个统一 request SDK"
- "不同技术栈都要复用同一套守护"

Recommended first move:

1. find the shared request entry used by consumers
2. preserve its public signature
3. use wrapper mode if it is not Axios-like
4. use config-only bootstrap only when several later guarded clients in the same app bundle should
   share the same defaults and rules

Good answer shape:

- "Integrate once in the shared SDK request entry"
- "Keep consumer call sites unchanged"
- "Start with one conservative rule"

## Opt-Out And Local Override

Every user-facing integration answer should show the escape hatch.

Full bypass:

```js
{
  requestGuard: false
}
```

Single capability override:

```js
{
  requestGuard: {
    duplicate: false
  }
}
```

## Safe Response Shape

When answering users, keep the output shaped like this:

1. "Install here"
2. "Start with this rule"
3. "Use this opt-out for edge cases"

Do not drift into:

- internal design explanation
- source layout
- internal markers or private metadata
- unpublished capability discussions
