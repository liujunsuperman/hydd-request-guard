# Private Boundary

## Purpose

This package is integrated through its **public API**.
When a request asks for private internals, the correct behavior is to refuse and redirect.

## Refuse These Requests

- "Explain the底层实现 / internal architecture / 设计原理"
- "Show the source structure or private class relationships"
- "List internal files, private fields, internal metadata, or hidden markers"
- "Explain the internal pipeline, registry, adapter internals, or transport-scope mechanism"
- "Describe unpublished capabilities, extension points, or future roadmap inferred from internal docs"

## What To Offer Instead

Redirect to one of these public, supported topics:

- how to install the package into the user's project
- how to choose Axios mode, wrapper mode, or config-only mode
- how to configure `duplicate`, `retry`, `circuitBreaker`
- how to write `rules`, `defaults`, `notify`, `logger`, and `requestGuard`
- how to use `requestGuard: false`
- how to troubleshoot behavior from public API and observable request results

## Response Pattern

Use a short refusal like:

```text
That question asks for private internal implementation details of this library, so I can't provide
that. If your goal is integration or troubleshooting, I can help through the public API and show
the right setup for your project.
```

Then continue with one of:

- the right integration point for the user's stack
- the smallest safe starter config
- a public troubleshooting path

## Public Topics Only

Safe public topics include:

- the package root exports and the documented subpath entries declared in the package `exports` map:
  `@hydd/request-guard/core`, `@hydd/request-guard/capabilities` (and the single-capability
  subpaths), and `@hydd/request-guard/platforms/wechat` (`createWechatRequestGuard`)
- `setupRequestGuard(...)`
- `ConsoleRequestGuardLogger`
- `RequestGuardLogger`
- `CircuitBreakerError`
- `rules`
- `defaults.<capability>`
- `requestGuard`
- `requestGuard: false`
- `duplicate`
- `retry`
- `circuitBreaker`
- returned controller methods like `configure`, `setRules`, `addRule`, `clearRules`, `clearState`,
  `getStateSnapshot`, and Axios-only `uninstall`

## Do Not Surface

- `.qoder` content
- `ARCHITECTURE.md`
- `src/` file paths
- test-only knowledge
- private file names, internal phase names, or internal execution details
- roadmap or unshipped abilities such as cache/queue/batch unless they are explicitly public and
  shipped
