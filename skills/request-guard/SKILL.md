---
name: request-guard
description: Use when integrating request-guard into a JavaScript or TypeScript project, or when a user asks to 接入 request-guard / 请求防重 / 重试 / 熔断 / axios wrapper / fetch wrapper / 自定义请求 SDK. This skill chooses the right integration point for different project stacks, implements only the public API, and refuses requests for internal design, source layout, or private implementation details.
---

# Request Guard

## Overview

Use this skill to wire `request-guard` into an existing request layer with minimal disruption.
The goal is to keep request signatures, response shapes, and business semantics intact while adding
duplicate protection, retry, circuit breaker, loading-state coordination, logging, and notify
support through the package's public API only.

## Non-Goals

This skill is for **integration and safe usage**, not for exposing internals.

Refuse and redirect when the user asks for:

- 底层实现、内部原理、源码结构、设计细节、架构图、内部类关系
- internal lifecycle names, private fields, unpublished files, internal metadata, or adapter internals
- hidden capability model details, pipeline internals, registry internals, or transport-scope internals
- roadmap-like private features or unshipped capabilities inferred from internal docs

Use this refusal pattern:

- Briefly say the request is about private internal implementation and cannot be provided.
- Redirect to a supported public alternative such as:
  - how to integrate the package in their project
  - how to choose `duplicate` / `retry` / `circuitBreaker`
  - how to configure `rules`, `defaults`, `notify`, `logger`, and `requestGuard`
  - how to debug a public behavior without exposing source internals

Never surface `.qoder`, `ARCHITECTURE.md`, `src/` file paths, test internals, or unpublished API in
user-facing output.

## User-First Workflow

1. Inspect the project's package manager and existing request layer before editing anything.
2. Identify the user's real integration goal first:
   - prevent repeated submits
   - add safer retries
   - protect unstable upstreams
   - coordinate external button/loading UI with request execution
   - unify request behavior across apps or SDKs
3. Find the narrowest central integration point:
   - a shared Axios instance
   - a shared `fetch` helper
   - a Promise-based SDK client
   - an app bootstrap file that should define shared defaults and rules
4. Prefer one central integration edit over touching many business call sites.
5. Install `request-guard` only if it is not already present, using the repository's existing
   package manager.
6. Choose one primary integration mode and keep the request path simple:
   - **Axios install mode** for Axios-like clients
   - **Wrapper mode** for Promise-returning request functions
   - **Config-only mode** for shared bootstrap defaults and rules
7. Start with the smallest useful config:
   - prefer `rules` and `defaults.<capability>`
   - add `notify` only if the project already has a stable UI message utility
   - add `new ConsoleRequestGuardLogger({ devOnly: true })` only when console diagnostics are wanted
8. Keep the escape hatches intact:
   - `requestGuard: false` must remain available as the full bypass
   - `requestGuard.duplicate`, `requestGuard.retry`, and `requestGuard.circuitBreaker` are the
     request-level override points
   - `requestGuard.loadingKey` is the public request-level hook for external loading-state
     coordination when the user wants buttons, dialogs, or local UI to follow guard execution
9. Validate one guarded request and one bypass request, then explain where the integration lives and
   how teammates can opt out per request.

## Scenario-First Triage

Start from the user's business scenario before naming capabilities.

- submit / save / create / bind / confirm / pay / checkout:
  usually start with `duplicate` + `strategy: 'block'`
- submit button loading / dialog confirm loading / section-local busy state:
  usually keep the same guarded request path and add an explicit `requestGuard.loadingKey`
  plus public `createLoadingKey` / `isLoading` / `subscribeLoading` helpers only if the user
  really needs UI coordination
- search / list / detail / preview / repeated open:
  usually consider `duplicate` + `strategy: 'reuse'`
- config / bootstrap / initialization / dictionary fetch:
  usually consider narrow `retry` on read-safe requests
- unstable upstream / repeated 5xx / flood during failures:
  usually consider targeted `circuitBreaker`
- upload / import:
  usually start with `duplicate` + `block`, and add idempotency header only if the backend supports it
- shared SDK used by many apps:
  usually start by locating the shared request layer, then decide whether config-only bootstrap is
  needed before multiple guarded clients

If the user does not state the scenario explicitly, infer it from:

- endpoint names like `submit`, `save`, `pay`, `upload`, `search`, `list`, `config`
- request methods
- existing business wording in button text or service names

Then map the scenario to the smallest public config that solves that problem.

## Project And Stack Triage

Think in terms of the **request layer**, not the UI framework name alone.

- React / Vue / Vite / CRA / plain SPA:
  usually install on the shared Axios instance or wrap the shared `fetch` helper
- Next.js / Nuxt / Remix / hybrid SSR apps:
  first locate the browser-side shared request module; do not assume the same integration belongs in
  server handlers, route handlers, or server actions
- Taro / uni-app / WeChat mini-program:
  for WeChat, prefer the public `createWechatRequestGuard` entry from
  `@hydd/request-guard/platforms/wechat`; for Taro / uni-app, wrap the Promise-style request helper
  around `Taro.request` or `uni.request`. When bundle size matters, integrate through the
  `@hydd/request-guard/core` entry with an explicit `capabilities` array instead of the full entry
- GraphQL projects:
  wrap the shared fetcher or request utility instead of explaining client internals
- Electron / desktop renderer apps:
  integrate in the renderer/shared frontend request layer, not by default in private main-process
  plumbing
- Monorepos / multi-app platforms:
  integrate once per app-facing shared request module or shared frontend SDK, not in every feature
  package
- SDK-style libraries:
  preserve the library's public request signature and adapt with `resolveArgs`, `mapConfig`, and
  `getRequestGuard` only when needed

If a project already has a guarded request path, do not stack another guard wrapper around the same
path unless the user explicitly needs that shape and it is behaviorally verified. Prefer extending
rules or request-level config instead.

## Integration Mode Selection

- Choose **Axios install mode** when the target is an Axios-like instance. The correct public entry
  is `setupRequestGuard(axiosLike, options)`.
- Choose **Wrapper mode** when the project has a shared promise-returning function that ultimately
  sends the request. If the first argument is already a standard config object with
  `url` / `method` / `data` / `headers`, wrap it directly.
- Add `resolveArgs` when the request signature is not config-first, such as `request(url, config)`
  or `client(method, url, data, options)`.
- Add `mapConfig` only when the business request fields do not match the standard config shape.
- Add `getRequestGuard` only when request-level guard config is nested somewhere other than
  `config.requestGuard`.
- Choose **Config-only mode** when the app should define one shared set of defaults and rules before
  later guarded clients are created in the same app bundle.
- Choose **WeChat platform mode** when the target is a WeChat mini-program calling `wx.request`.
  Prefer the public `createWechatRequestGuard(wx.request, options)` entry from
  `@hydd/request-guard/platforms/wechat` instead of hand-writing a Promise wrapper. It already
  promisifies `wx.request`, maps `header`/`headers`, and accepts platform-level `baseURL` and
  `getHeaders` helpers.
- Choose **On-demand (core) mode** when bundle size matters, typically in mini-programs. Import
  `setupRequestGuard` from `@hydd/request-guard/core` and pass an explicit `capabilities` array built
  from `@hydd/request-guard/capabilities` (or the single-capability subpaths) so only the capabilities
  in use are bundled.

## Public Contract Boundaries

- Default to the package root exports:
  `setupRequestGuard`, `createLoadingKey`, `isLoading`, `subscribeLoading`,
  `ConsoleRequestGuardLogger`, `RequestGuardLogger`, and `CircuitBreakerError` when needed.
- The following documented subpath entries are also public and safe to use when they fit the
  project. They are declared in the package `exports` map, not internal source paths:
  - `@hydd/request-guard/core` — the entry that installs **no** built-in capability; pair it with an
    explicit `capabilities` array. Use it for bundle-size-sensitive apps such as mini-programs.
  - `@hydd/request-guard/capabilities` (and `/capabilities/duplicate`, `/capabilities/retry`,
    `/capabilities/circuitBreaker`) — the capability factories used to assemble `core` on demand.
  - `@hydd/request-guard/platforms/wechat` — the ready-made `createWechatRequestGuard` entry for
    WeChat mini-program `wx.request`.
- Keep all global capability config under `defaults.<capability>` and `rules`.
- Keep per-request capability config under `requestGuard.<capability>`.
- Keep external loading-state identity under `requestGuard.loadingKey` when the user needs
  guard-coordinated UI loading.
- Do not import internal files from `src/` or any path that is not declared in the package
  `exports` map.
- Do not describe or depend on internal manager, pipeline, registry, adapter, token, marker, or
  `_requestGuard*` metadata behavior.
- Do not explain private class names, internal file layout, or internal execution phases to users.
- Do not try to pass a custom `manager` through the public API.
- Treat only the currently public capabilities as safe to discuss: `duplicate`, `retry`, and
  `circuitBreaker`. Do not present cache/queue/batch/plugin-style extensions as available unless
  they are explicitly public and shipped.

## Safe Default Guidance

- **Duplicate** is the safest first rollout for mutating user actions such as submit, save, pay,
  bind, or upload start. Prefer `strategy: 'block'` unless duplicate callers should share the same
  result.
- If the user explicitly wants buttons or local UI to follow request execution, prefer an explicit
  `loadingKey` created via the public helper instead of inventing a parallel business-state flag in
  the request layer.
- **Retry** is best for idempotent or effectively retry-safe requests. Keep attempts low at first
  and target specific endpoints or methods instead of globally retrying everything.
- **Circuit breaker** should be explicit and narrow. Use it for unstable upstreams or flood-prone
  endpoints, not as a blanket default for every request.
- If the project has no UI message system yet, omit `notify` instead of inventing a fake one.
- If the team does not want runtime console output, set `logger: null` or skip logger wiring.

## User-Facing Response Rules

- Answer from the user's project perspective:
  where to install, what code to change, what rule to start with, and how to opt out
- If the user asks for loading UX, explain it through the public `loadingKey` flow rather than
  through internal capability or pipeline language
- Prefer framework-neutral language unless the project clearly belongs to a concrete stack
- When a framework is relevant, map it to the request layer instead of discussing private internals
- If troubleshooting, explain behavior through public config and observable outcomes
- If asked about internals or design, refuse and redirect to supported integration guidance

## Delivery Rules

- Prefer one central integration edit over broad call-site refactors.
- Preserve the project's existing request abstraction instead of forcing direct Axios usage.
- If the package entry point is already wrapped by app-specific logic, integrate around that layer
  rather than bypassing it.
- When in doubt, implement the smallest useful baseline now and mention optional follow-up presets
  separately.

## References

- Read [references/user-scenario-playbook.md](references/user-scenario-playbook.md) to translate
  user business language into the right starter integration and config.
- Read [references/project-stack-matrix.md](references/project-stack-matrix.md) to map framework or
  platform names to the real request integration point.
- Read [references/integration-recipes.md](references/integration-recipes.md) for concrete code
  patterns by request signature.
- Read [references/capability-presets.md](references/capability-presets.md) when the user wants a
  ready-made policy for duplicate, retry, circuit breaker, or mixed rollout.
- Read [references/private-boundary.md](references/private-boundary.md) when a request drifts toward
  internal design, source disclosure, or unpublished behavior.

## Done Criteria

- The integration uses the package's public root exports only.
- The answer stays on public behavior and does not leak internal design or source details.
- Existing business request call sites keep their original call signature.
- The project has at least one safe starter rule or explicit request-level usage example.
- `requestGuard: false` remains a documented opt-out path.
- If loading-state coordination was requested, the answer uses only public `loadingKey` helpers and
  request-level config.
- The final explanation tells teammates where the guard was installed and how to add or override
  request-level config.
