# Capability Presets

## Contents

1. Choosing the first capability
2. Duplicate presets
3. Retry presets
4. Circuit breaker presets
5. Mixed rollout presets
6. Precedence and opt-out
7. Runtime cleanup patterns
8. Common mistakes

## Choosing the First Capability

Start small and choose the capability that matches the user's real pain:

- Repeated clicks, duplicate form submits, repeated payment taps:
  start with `duplicate`
- Intermittent network or config/bootstrap fetch failures:
  start with `retry`
- A flaky upstream that can be flooded during failures:
  start with `circuitBreaker`
- If the user says "just make it safe by default", start with a narrow `duplicate` rule on
  mutating requests and mention retry/circuit breaker as follow-up options

## Duplicate Presets

### Form Submit Baseline

```js
{
  method: 'post',
  duplicate: {
    strategy: 'block',
    message: 'Please do not submit again.'
  }
}
```

Use for:

- form submission
- save/update actions
- bind/unbind actions
- upload start

### Shared Result Baseline

```js
{
  url: /\/search|\/detail|\/preview/,
  duplicate: {
    strategy: 'reuse'
  }
}
```

Use `reuse` only when duplicate callers should wait for and reuse the same result. It is a better
fit for duplicate reads or repeated "open detail" actions than for destructive writes.

### Business-Key Duplicate

```js
{
  duplicate: {
    strategy: 'block',
    keyGenerator(config) {
      return `order:${config.data.orderId}`;
    }
  }
}
```

Use this when the backend treats a business key as the real idempotency boundary.

### Idempotency Header Opt-In

```js
{
  defaults: {
    duplicate: {
      idempotencyKeyHeader: {
        enabled: true,
        headerName: 'Idempotency-Key'
      }
    }
  }
}
```

Use this only when the server or gateway understands idempotency headers.

## Retry Presets

### Safe Read Retry

```js
{
  method: 'get',
  retry: {
    attempts: 2,
    delay: 300,
    backoff: 'exponential'
  }
}
```

Good first choice for configuration fetches, dictionary data, and idempotent reads.

### Status-Code Scoped Retry

```js
{
  url: /\/config|\/bootstrap/,
  retry: {
    attempts: 3,
    delay: 200,
    backoff: 'exponential',
    statusCodes: [408, 429, 500, 502, 503, 504]
  }
}
```

Use this when the project wants more explicit retry boundaries.

### Custom Retry Decision

```js
{
  retry: {
    attempts: 3,
    delay: 250,
    canRetry({ status, code, config }) {
      if (config.url?.includes('/payment')) return false;
      return status === 503 || code === 'ECONNABORTED';
    }
  }
}
```

Use this when the rules must reflect business safety rather than just transport status.

Avoid:

- global retry on all mutating requests
- high default attempt counts
- retrying business validation errors

## Circuit Breaker Presets

### Critical Upstream Protection

```js
{
  url: /\/core\/upstream\//,
  circuitBreaker: {
    failCount: 3,
    window: 30000,
    recoverDelay: 10000
  }
}
```

Use this for endpoints that can fail in bursts and trigger a flood of repeated traffic.

### Domain-Scoped Protection

```js
{
  url: /^https:\/\/risk-api\.example\.com\//,
  circuitBreaker: {
    type: 'domain',
    failCount: 3,
    window: 30000,
    recoverDelay: 15000,
    probeCount: 1
  }
}
```

Use `type: 'domain'` when one host or domain group is the unit of failure, not a single path.

Keep circuit breaker targeted. It should be opt-in for specific unstable upstreams, not a blanket
default for every request in the app.

## Mixed Rollout Presets

### Safe Starter Bundle

```js
{
  rules: [
    {
      method: 'post',
      duplicate: { strategy: 'block', message: 'Request is already in progress.' }
    },
    {
      url: /\/config|\/bootstrap/,
      retry: { attempts: 2, delay: 300, backoff: 'exponential' }
    }
  ]
}
```

This is a strong default when the user wants broad but conservative coverage.

### Payment-Oriented Bundle

```js
{
  defaults: {
    duplicate: {
      idempotencyKeyHeader: {
        enabled: true,
        headerName: 'Idempotency-Key'
      }
    }
  },
  rules: [
    {
      url: /\/payment|\/checkout/,
      duplicate: {
        strategy: 'block',
        message: 'Payment is being processed.'
      }
    }
  ]
}
```

Use only when the server-side contract supports idempotency headers.

## Precedence and Opt-Out

The stable precedence order is:

```text
requestGuard.<capability> > rules > defaults.<capability>
```

The full bypass is:

```js
{
  requestGuard: false
}
```

Examples:

```js
await request({
  url: '/api/order',
  method: 'POST',
  requestGuard: {
    duplicate: false
  }
});

await request({
  url: '/api/health',
  method: 'GET',
  requestGuard: false
});
```

## Runtime Cleanup Patterns

### Logout or app reset

Use the returned controller when the app needs to clear in-flight guard state during logout, account
switch, or app reset:

```js
requestGuardController.clearState();
```

Good times to use this:

- logout
- tenant switch
- route/app teardown where stale waiting state would be confusing

### Runtime rule change

If the app needs to swap rules after initialization, prefer public controller methods:

```js
requestGuardController.setRules([
  { method: 'post', duplicate: true }
]);
```

Use `clearRules()` when you want later requests to stop matching global rules without changing
existing business request code.

## Common Mistakes

- Installing the guard separately in many feature files instead of one shared request layer
- Importing internal files from `src/` instead of using public package entries (the package root or
  a documented subpath like `@hydd/request-guard/core`, `/capabilities`, or `/platforms/wechat`)
- Inventing top-level fields like `duplicateMessage` or `retryAttempts` instead of using
  `defaults.<capability>` or `requestGuard.<capability>`
- Enabling `reuse` for business actions that should fail loudly on repeated submits
- Retrying unsafe write operations by default
- Forgetting to explain `requestGuard: false` as the opt-out path for edge cases
- Answering with private implementation details instead of staying on public integration behavior
