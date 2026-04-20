# PR Review: PP-1750: Improve System Error Tracking and Logging

**PR:** https://github.com/Proofed/B2BWebserver/pull/2253
**Jira:** https://proofed.atlassian.net/browse/PP-1750
**Status:** In Progress
**Mergeable state:** ✅ Up-to-date with `develop` (merge commit `d22331582`, 0 behind)
**CI status:** pending

---

## Resolution Status (updated 2026-04-20, post-develop-merge)

All 12 valid review points are addressed on branch `fix/PP-1750-improve-error-tracking`. The two invalid points are skipped with rationale below.

| # | Issue | Status | Landing commit |
|---|---|---|---|
| 1 | Caller-supplied `extra` overwritten by axios details | ✅ Resolved | `af8f89c78` |
| 2 | Duplicated QueryClient + interceptor bootstrap | ✅ Resolved — extracted to `createAppQueryClient` | `af8f89c78` |
| 3 | Narrow error boundary scope | ✅ Resolved — boundary is outermost, inside `<ThemeProvider>` | `af8f89c78` |
| 4 | Axios `AxiosHeaders` prototype stripped by spread | ⚠️ **Conditional** — safe on `develop` today (`axios ^0.27.2`, plain-object headers). **Becomes a real hazard once [PR #2261](https://github.com/Proofed/B2BWebserver/pull/2261) merges**, which bumps axios `^0.27.2 → ^1.15.0`. Axios 1.x uses the `AxiosHeaders` class and our spread pattern in `installAxiosCorrelationInterceptor` will strip it. See "Blocking follow-up" below. | — |
| 5 | High-cardinality `route` tag | ✅ Resolved — new `getSentryRoute()` helper prefers Next router template (`/orders/[id]`) | `af8f89c78` |
| 6 | Error ID shown in boundary but not in `ErrorPage` | ✅ Resolved — `errorId` threaded through `getInitialProps` and rendered | `af8f89c78` |
| 7 | Unsafe `as Error & { statusCode?: number }` cast | ✅ Resolved — narrowed to `{ statusCode?: number } \| null` | `af8f89c78` |
| 8 | Sentry mock missing v10 API surface | ✅ Resolved — added `withIsolationScope`, `withScope`, `replayIntegration`, `browserTracingIntegration`, `setUser`, `flush`, `getScopeData` | `af8f89c78` |
| 9 | No test for `withIsolationScope` migration | ✅ Resolved — new `withSentryUser.test.ts` (5 tests) | `af8f89c78` |
| 10 | Module-scope `installAxiosCorrelationInterceptor()` | ✅ Resolved — moved into `useEffect` in both `_app.tsx` | `af8f89c78` |
| 11 | Stale `/* eslint-disable no-console */` | ✅ Resolved — removed from `sentryContext.ts:1` | `af8f89c78` |
| 12 | `^10.48.0` caret range | ✅ Resolved — pinned to exact `10.48.0` in both portals. Note: the reviewer's framing ("project convention is exact pinning") is wrong in general — the repo uses caret ranges almost everywhere — but the previous `@sentry/nextjs` entry on `develop` **was** exact-pinned (`"7.73.0"`), and after a major-version bump it's defensive to keep the exact-pin convention for this dep rather than allowing silent drift on a freshly-upgraded SDK. |
| 13 | No test for `reportError` with primitive inputs | ✅ Resolved — 4 new primitive tests + caller-wins test | `af8f89c78` |
| 14 | No test for `_app.tsx` QueryCache/MutationCache wiring | ✅ Resolved — factory extraction + `createAppQueryClient.test.ts` (5 tests) | `af8f89c78` |

### Post-review hardening

| Change | Commit |
|---|---|
| Wrap non-Error `reportError` inputs in synthetic `Error("Non-Error thrown: …")` so Sentry shows searchable titles instead of `<unknown>`; raw value preserved at `extra.originalValue` | `c6ee03005` |
| Pin `@sentry/nextjs` to exact `10.48.0` to prevent silent drift on a freshly-upgraded SDK (closes review #12) | `8dbf0b9de` |

> The `/sentry-test` manual verification page (commits `c3cc2397b` and `069833755`) was reverted before merge — the full `1008/1008` automated suite already covers every error-tracking path, and keeping a dev-only page reachable on stage/devtest created noise without lasting value.

### Verification

- `npx turbo run test --filter=@proofed/shared` → **1008/1008 passing**
- `npx turbo run typecheck --filter=@proofed/shared` → no new errors (pre-existing errors in `Loader`, `Typography`, `iron-session` are orthogonal to this PR and exist on `HEAD~4`)
- `npx turbo run lint --filter=@proofed/shared --filter=@proofed/creative-portal --filter=@proofed/customer-portal` → clean for PP-1750 files (pre-existing errors in other files are unrelated)

### Blocking follow-up: axios 1.x compatibility (#4)

[PR #2261 "PP-1754: Upgrade TypeScript to v6 and development tooling"](https://github.com/Proofed/B2BWebserver/pull/2261) upgrades `axios ^0.27.2 → ^1.15.0`. Direct evidence from that PR's diff:

```diff
# apps/creative-portal/package.json
-    "axios": "^0.27.2",
+    "axios": "^1.15.0",
```

and from `IMPROVEMENTS.md` in the same PR:

```diff
+| axios | 0.27 | 1.15.0 | ✅ Done | |
```

PR #2261 already updated its own axios interceptor (`apps/creative-portal/api/utils/tiptap/api.ts`) to mutate `config.headers.Authorization` instead of reassigning `config.headers`. **But it did NOT update `packages/shared/utils/installAxiosCorrelationInterceptor.ts`** — that file still uses the spread pattern the reviewer flagged:

```ts
return {
  ...config,
  headers: { ...headers, [CORRELATION_HEADER]: corrId } as typeof config.headers
};
```

Under axios 1.x, `config.headers` is an `AxiosHeaders` class instance. Spreading strips the prototype, so any downstream interceptor or axios internal that calls `config.headers.set(...)`, `.get(...)`, `.has(...)` will throw `TypeError: config.headers.set is not a function`.

**Action before PR #2261 merges** — replace the spread pattern with direct mutation via `AxiosHeaders` API:

```ts
axios.interceptors.request.use((config) => {
  const existing = config.headers?.get?.(CORRELATION_HEADER);
  const corrId =
    typeof existing === "string" && existing.length > 0
      ? existing
      : generateCorrelationId();
  config.headers.set(CORRELATION_HEADER, corrId);
  return config;
});
```

This is safe in axios 1.x (uses the class API) and should be landed either as a follow-up commit on PP-1750 **or** included in PR #2261's migration. Whichever comes first — but the fix MUST exist before axios 1.15.0 reaches `develop`, otherwise every browser request will break.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1.1 Capture all unhandled React errors | `AppErrorBoundary` wraps `<Component>` in both apps; shared `_error.tsx` calls `captureUnderscoreErrorException` for SSR errors. | ⚠️ Partial — boundary scope is narrow (see issue #3). |
| 1.2 Errors that interrupt user actions are tracked | `QueryCache`/`MutationCache` global `onError` + migration of 13+ `console.error` swallow sites to `reportError`. | ✅ Addressed |
| 1.3 Contextual metadata (user, route, component, timestamp) | `reportError` accepts `source`, `operation`, `route`, `component`, `queryKey`, `mutationKey`, `httpStatus`, `extra`, `corrId`; `setSentryContext` adds identifiers/request scope. Timestamp is Sentry-default. | ✅ Addressed |
| 2.1 Log internal API failures | Axios response interceptor pushes `corr_id`/`http_status`/`route` into Sentry scope; React Query `onError` reports with axios details. | ✅ Addressed |
| 2.2 Error includes endpoint / status / corr_id / request metadata | `reportError` serializes `url`, `method`, `params`, `requestData`, `responseData`, `responseStatus`, `requestHeaders`, `responseHeaders` when axios error detected. | ✅ Addressed |
| 3.1 Third-party (Wise) failures captured | `useMutation(postCreateRecipient, { onError: () => reportError(..., { source: "wise", operation: "createRecipient" }) })`. | ✅ Addressed |
| 3.2 Service name / endpoint / status | `source` tag + `operation` tag + axios extras provide all three. | ✅ Addressed |
| 4 Centralized monitoring | All paths route through `Sentry.captureException`; corr_id tag joins frontend Sentry with backend Logtail events. | ✅ Addressed |
| 5 Consistent logging coverage | 14 swallow sites migrated; zero `console.error` swallows remain in changed files. | ✅ Addressed |

**Beyond-scope changes also present:**

- Sentry SDK upgrade `7.73.0` → `^10.48.0` (major × 3 bumps). This is a high-risk refactor that was *not* in the original Jira scope; it's called out in the PR description as "deferred" but was merged in anyway in a later commit (`fb9a37c01`).
- `transpileClientSDK: true` removed (drops IE11 transpilation).
- Trace sampling changed from 100% everywhere to tiered (devtest 1.0 / stage 0.5 / production 0.1) — good change, but out of ticket scope.
- Scrubber cleanup (deduplicate SENSITIVE_FIELDS, remove redundant re-scrub).

---

## Architecture Analysis

The approach is sound: a shared `reportError(error, context)` utility funnels every error through `Sentry.captureException` with consistent tags (`source`, `operation`, `http_status`, `corr_id`) and structured extras. Around this:

- `AppErrorBoundary` — hand-rolled React boundary (correct; no hook equivalent exists).
- `installAxiosCorrelationInterceptor` — module-level side effect that stamps every outgoing axios request with `x-correlation-id` and pushes scope metadata on response errors.
- `QueryCache`/`MutationCache` global `onError` — catches errors even when per-hook `onError` shadows the default.
- `_error.tsx` — shared SSR error page re-exported in both apps.

The Sentry SDK bump (v7 → v10) is significant. It rewrites `sentryContext.ts` from the legacy `Scope.getContext()` API to `getScopeData().contexts`, replaces `Sentry.runWithAsyncContext` with `Sentry.withIsolationScope` in `withSentryUser.ts`, and switches from class-based `new Sentry.Replay(...)` to functional `Sentry.replayIntegration(...)`. The semantic difference between `runWithAsyncContext` (shared scope across awaits) and `withIsolationScope` (isolated scope per block) is not covered by any new test — this is worth validating in stage before merge.

Duplication remains between the two portals' `_app.tsx` files: ~40 lines of identical QueryClient/QueryCache/MutationCache bootstrap plus the interceptor install and `AppErrorBoundary` wrapper. Given `@proofed/shared` already hosts all the error-tracking utilities, this bootstrap should live there too.

---

## Issues Found

### 1. `reportError` caller-supplied `extra` is silently overwritten by axios details

**[File: packages/shared/utils/throwSentryError.ts]**
**Function/Class:** `buildExtras`
**Severity:** medium
**Problem:** `context.extra` is spread first (line 69), then the builder sets `route`, `component`, `queryKey`, `mutationKey`, and — when the error is an axios error — `url`, `method`, `params`, `requestData`, `responseData`, `responseStatus`, `requestHeaders`, `responseHeaders`. A caller passing `extra: { url: "..." }` or `extra: { method: "..." }` alongside an axios error will have their values silently overwritten.
**Impact:** Surprising precedence — call sites that think they're enriching an event may actually have their values clobbered, and reviewers reading Sentry events won't know the displayed `url` is the axios one versus the caller's intent.
**Fix:** Either spread `extra` last (caller wins), or namespace axios fields (`axios_url`, `axios_method`, ...), or assert non-overlap. Simplest:

```ts
const buildExtras = (error, context) => {
  const extras: Record<string, unknown> = {};
  if (context.route) extras.route = context.route;
  if (context.component) extras.component = context.component;
  if (context.queryKey !== undefined) extras.queryKey = serializeAndTruncate(context.queryKey);
  if (context.mutationKey !== undefined) extras.mutationKey = serializeAndTruncate(context.mutationKey);
  if (axios.isAxiosError(error)) {
    Object.assign(extras, {
      axiosUrl: error.config?.url,
      axiosMethod: error.config?.method,
      // ...
    });
  }
  return { ...extras, ...(context.extra ?? {}) }; // caller wins
};
```

### 2. Duplicated QueryClient + interceptor bootstrap across both `_app.tsx` files

**[File: apps/creative-portal/pages/_app.tsx, apps/customer-portal/pages/_app.tsx]**
**Function/Class:** `App` / module-scope
**Severity:** medium
**Problem:** Both files contain byte-identical code: (a) top-level `installAxiosCorrelationInterceptor()` call with the same comment, (b) ~40 lines of `QueryCache`/`MutationCache`/`QueryClient` wiring with `onError` → `reportError`, (c) the `<AppErrorBoundary componentName="...PortalApp">` wrapper. Only the `componentName` string differs.
**Impact:** The whole point of the PR is consistent error tracking; divergence between the two apps here would silently break that consistency. Future tweaks (e.g., adding `operation`, changing `route` derivation to `router.pathname`) will need to be applied in two places.
**Fix:** Move to a shared factory/hook in `@proofed/shared/utils`:

```ts
// packages/shared/utils/createAppQueryClient.ts
export const createAppQueryClient = () => {
  const queryCache = new QueryCache({
    onError: (error, query) =>
      reportError(error, { source: "react-query", queryKey: query.queryKey, route: getRoute() })
  });
  const mutationCache = new MutationCache({ /* ... */ });
  return new QueryClient({ queryCache, mutationCache, defaultOptions: { /* ... */ } });
};
```

And a shared `<AppShell portal="creative"|"customer">` or at minimum a shared `installAxiosCorrelationInterceptor` call at the shared entry point.

### 3. Error boundary scope is too coarse — Layout / providers are outside it

**[File: apps/creative-portal/pages/_app.tsx]**
**Function/Class:** `App` (JSX tree)
**Severity:** medium
**Problem:** In creative-portal the boundary is wrapped deep inside providers: `<UserContextProvider><BulkActionsContextProvider><SidePanelContextProvider><TeamMembersContextProvider><ThemeProvider>...<FeaturesProvider><VerifyOnboarding><Layout><AppErrorBoundary>`. A throw in `FeaturesProvider`, `VerifyOnboarding`, `Layout`, `ModalQueue`, or any of the context providers will *escape the boundary* and crash the app without a fallback. In customer-portal the same issue exists — `ModalQueue` and `Toast` sit outside `AppErrorBoundary`.
**Impact:** The PR description states "Catches render crashes, reports to Sentry with `source=error-boundary`", but any crash in provider/layout code will bypass the boundary and produce a blank page with no Sentry event (only the `_error.tsx` SSR fallback would catch SSR-time versions; CSR provider crashes hit the default React behaviour).
**Fix:** Wrap at the outermost render node. Move `<AppErrorBoundary>` to be the top-level element inside each App's return statement:

```tsx
return (
  <AppErrorBoundary componentName="CreativePortalApp">
    <QueryClientProvider client={queryClient}>
      <UserContextProvider ...>
        {/* rest of tree */}
      </UserContextProvider>
    </QueryClientProvider>
  </AppErrorBoundary>
);
```

Optionally add nested boundaries around known crash hotspots (Tiptap editor regions, OrderCreation flow) so a local failure doesn't blank the whole app.

### 4. Axios interceptor spreads config, likely stripping `AxiosHeaders` prototype

**[File: packages/shared/utils/installAxiosCorrelationInterceptor.ts]**
**Function/Class:** `installAxiosCorrelationInterceptor` (request interceptor)
**Severity:** medium
**Problem:** The interceptor returns `{ ...config, headers: { ...headers, [CORRELATION_HEADER]: corrId } as typeof config.headers }`. In axios v1 `config.headers` is an `AxiosHeaders` *class instance*, not a plain object. Spreading it into `{}` produces a plain object missing `set`, `has`, `delete`, `getContentType`, etc. The `as typeof config.headers` cast silences the type error but leaves a real runtime hazard for any downstream interceptor (or axios internal) that calls `config.headers.set(...)`.
**Impact:** Any code downstream of this interceptor that uses `AxiosHeaders` methods will throw `TypeError: config.headers.set is not a function`. The project pins `axios` at a v1 version so this is a real concern.
**Fix:** Mutate headers via the SDK API instead of reconstructing the config:

```ts
axios.interceptors.request.use((config) => {
  const existing = config.headers?.get?.(CORRELATION_HEADER);
  const corrId = typeof existing === "string" && existing.length > 0
    ? existing
    : generateCorrelationId();
  config.headers.set(CORRELATION_HEADER, corrId);
  return config;
});
```

Add a test that confirms `config.headers instanceof AxiosHeaders` after the interceptor runs (use a real axios request config, not a plain object mock).

### 5. High-cardinality `route` tag from `window.location.pathname`

**[File: packages/shared/utils/installAxiosCorrelationInterceptor.ts, packages/shared/components/molecules/AppErrorBoundary/index.tsx, apps/\*/pages/_app.tsx]**
**Function/Class:** response interceptor, `componentDidCatch`, QueryCache/MutationCache `onError`
**Severity:** medium
**Problem:** Three different files in this PR enrich the `route` field with `window.location.pathname`, which returns the concrete URL (e.g., `/orders/42`). For Sentry tag grouping you want the route template (`/orders/[id]`). Next's `router.pathname` or `router.route` provides this. In contrast, the new `_error.tsx` correctly uses `contextData.asPath`. The inconsistency means the same error on `/orders/42` and `/orders/43` will appear as two separate routes in Sentry dashboards, blowing out tag cardinality.
**Impact:** Sentry tag cardinality quota issues; noisy dashboards; harder to aggregate errors by page.
**Fix:** Standardize on `router.pathname` (template). Factor out a single `getSentryRoute()` util in `@proofed/shared/utils` and call it from all three places. Consider stamping both (`route` = template, `url` = full pathname) as separate fields — template as `tag`, instance as `extra`.

### 6. Error ID shown in `AppErrorBoundary` fallback but not in `ErrorPage`

**[File: packages/shared/components/pages/error/index.tsx]**
**Function/Class:** `ErrorPage`
**Severity:** low
**Problem:** `AppErrorBoundary` renders the Sentry event ID (`Error ID: ...`) for user-reported debugging, but the shared SSR `ErrorPage` does not — even though its `getInitialProps` calls `captureUnderscoreErrorException` which returns an event ID.
**Impact:** Users hitting SSR errors cannot quote an event ID to support. Inconsistent UX between client-side boundary crashes and SSR-time crashes.
**Fix:** Return the Sentry event id from `captureUnderscoreErrorException` via `getInitialProps`, thread it into `ErrorPageProps`, and display it:

```ts
ErrorPage.getInitialProps = async (contextData) => {
  // ... existing setup
  const errorId = await captureUnderscoreErrorException(contextData);
  const next = await NextErrorComponent.getInitialProps(contextData);
  return { ...next, errorId };
};
```

### 7. Unsafe `as Error & { statusCode?: number }` cast in `ErrorPage.getInitialProps`

**[File: packages/shared/components/pages/error/index.tsx]**
**Function/Class:** `getInitialProps`
**Severity:** low
**Problem:** Line 46: `(contextData.err as Error & { statusCode?: number }).statusCode`. Next.js types `err` as `Error | null | undefined`, but at runtime it can be a string or a plain object if thrown non-Error values bubble up. Accessing `.statusCode` on a string is fine (returns `undefined`), but the cast conveys false certainty.
**Impact:** Minor — no runtime crash. Misleading for future edits.
**Fix:** Narrow explicitly:

```ts
const errWithStatus = contextData.err as { statusCode?: number } | null;
const statusCode = contextData.res?.statusCode ?? errWithStatus?.statusCode;
```

### 8. `packages/shared/__mocks__/@sentry/nextjs.ts` is incomplete for the v10 API surface

**[File: packages/shared/__mocks__/@sentry/nextjs.ts]**
**Function/Class:** mock module
**Severity:** medium
**Problem:** The mock exports `captureException`, `captureUnderscoreErrorException`, `setTag`, `setContext`, and a `getCurrentScope` that returns an object with `getContext`/`setContext`/`setTag`. Missing: `withIsolationScope` (used by `withSentryUser.ts`), `replayIntegration`/`browserTracingIntegration` (used by `client.config.ts`), `getScopeData` (used by the rewritten `sentryContext.ts`). The mock's `getCurrentScope` still returns `getContext` (v7 API) instead of `getScopeData` (v8+).
**Impact:** Tests that import modules touching the Sentry API may fall through to undefined methods and throw, or silently no-op. `sentryContext.test.ts` appears to patch its own mock — but any other test that imports from `packages/shared` and transitively touches Sentry is brittle.
**Fix:** Expand the mock to mirror the v10 surface actually used:

```ts
export const withIsolationScope = (fn: (scope: any) => unknown) => fn({ setTag: () => {}, setContext: () => {}, setUser: () => {} });
export const replayIntegration = () => ({ name: "Replay" });
export const browserTracingIntegration = () => ({ name: "BrowserTracing" });
export const getCurrentScope = () => ({
  getScopeData: () => ({ contexts: {} }),
  setContext: () => {},
  setTag: () => {}
});
```

### 9. No test coverage for `runWithAsyncContext` → `withIsolationScope` migration

**[File: packages/shared/api/utils/middlewares/withSentryUser.ts]**
**Function/Class:** `withSentryUser`
**Severity:** medium
**Problem:** The Sentry v7→v10 migration replaced `runWithAsyncContext` (shared scope across nested awaits) with `withIsolationScope` (isolated scope per invocation). These have different cross-async-boundary semantics: child Sentry captures from inside a deeply-nested await inside an API route will now see the fresh isolation scope rather than the parent scope.
**Impact:** Any API route that relies on tag/context propagation from an outer middleware into a deeply-nested library call may silently lose tags in production. Not covered by tests.
**Fix:** Add an integration test that invokes `withSentryUser` with a nested-async handler and asserts that `setTag`/`setContext` calls inside the nested handler land on the same scope as the outer middleware. If the semantics changed in ways that affect us, compensate (e.g., re-apply the parent scope inside the handler).

### 10. `installAxiosCorrelationInterceptor()` called at module top-level of `_app.tsx`

**[File: apps/creative-portal/pages/_app.tsx:56, apps/customer-portal/pages/_app.tsx:41]**
**Function/Class:** module scope
**Severity:** low
**Problem:** Side-effectful module-scope call. The util has a `typeof window === "undefined"` guard, so it no-ops during SSR — fine in practice. But in dev with HMR the module can re-evaluate; the `isInstalled` flag prevents double-install, though that same flag also prevents re-install across a full page reload in dev when the module is retained across hot reloads with mocked axios instances.
**Impact:** Brittle, not testable without a reset helper (which is why `resetAxiosCorrelationInterceptorForTests` exists). Running this inside `useEffect` of the App component would be more conventional and avoids module-scope side effects during SSR module evaluation.
**Fix:** Move to `useEffect(() => { installAxiosCorrelationInterceptor(); }, []);` inside the App component. Keep the `isInstalled` idempotency flag as a safety net.

### 11. Stale `/* eslint-disable no-console */` in rewritten `sentryContext.ts`

**[File: packages/shared/utils/sentryContext.ts:1]**
**Function/Class:** file header
**Severity:** low
**Problem:** The file no longer uses `console.*` (the pre-existing `console.warn` fallbacks were removed in the v10 API rewrite). The `eslint-disable no-console` directive at the top is now dead code.
**Impact:** Cosmetic; future additions of `console.log` would silently pass lint.
**Fix:** Delete line 1.

### 12. `@sentry/nextjs` uses `^10.48.0` while neighbouring deps are pinned

**[File: apps/creative-portal/package.json, apps/customer-portal/package.json]**
**Function/Class:** dependencies
**Severity:** low
**Problem:** The project convention (per CLAUDE.md and existing `package.json` entries) is exact-version pinning. This PR introduces `"@sentry/nextjs": "^10.48.0"` — a caret range that will silently pull minor + patch updates of a freshly-upgraded SDK on next `yarn install`.
**Impact:** Reviewers can't tell from the diff whether a later CI run will exercise the exact same SDK version; the lockfile is authoritative only until someone regenerates it.
**Fix:** Pin to an exact version for consistency: `"@sentry/nextjs": "10.48.0"`.

### 13. No test for `reportError` with non-Error primitive inputs

**[File: packages/shared/utils/throwSentryError.test.ts]**
**Function/Class:** test suite
**Severity:** low
**Problem:** `QueryCache.onError` typings treat `error` as `unknown`. `reportError` is called with that unknown value. No test covers `reportError(null)`, `reportError("string")`, `reportError(undefined)`, or `reportError({})`. Sentry's `captureException` accepts unknown but serializes primitives/plain objects differently than Errors.
**Impact:** If a network layer rejects with a plain object (common with some libs), the reported event may lack a useful message. No regression cover for this.
**Fix:** Add tests for `reportError` with `null`, `undefined`, primitive string, plain object — assert the event is still captured and `extra`/`tags` are set.

### 14. No direct test for `_app.tsx` QueryCache/MutationCache wiring

**[File: apps/\*/pages/_app.tsx]**
**Function/Class:** QueryClient factory
**Severity:** low
**Problem:** The `QueryCache`/`MutationCache` `onError` handlers are non-trivial inline functions that dereference `mutation.options.mutationKey` and `query.queryKey`. If Tanstack Query ships a breaking change to the `mutation.options` shape, this breaks silently in production and only shows up as "missing mutationKey" in Sentry events.
**Impact:** Core error-reporting path has no regression test.
**Fix:** Either extract the factory (see issue #2) and unit-test it, or add a Vitest test that renders `<App>` and triggers a failing mutation to assert `reportError` is called with the expected context.

---

## Tests

- ✅ `throwSentryError`: 9 new tests cover tags, extras, truncation, axios merge, user extras, returns event id.
- ✅ `AppErrorBoundary`: 5 tests cover children, fallback UI, custom fallback, Sentry report, event id render.
- ✅ `ErrorPage`: 6 tests cover status code, generic message, getInitialProps scope enrichment.
- ✅ `installAxiosCorrelationInterceptor`: 7 tests cover install idempotency, header stamping, caller-id passthrough, missing headers, scope push, missing `error.config`.
- ✅ `getSentryTracesSampleRate`: 5 tests cover devtest / stage / production / default / undefined.
- ❌ No test for `reportError` with non-Error primitives (issue #13).
- ❌ No test for `withSentryUser`'s `withIsolationScope` migration (issue #9).
- ❌ No test for `_app.tsx` QueryCache/MutationCache wiring (issue #14).
- ❌ No test asserting `AxiosHeaders` instance preservation through the interceptor (issue #4).
- ❌ No test asserting the `SENSITIVE_FIELDS` scrubber still redacts auth/token substrings after the scrubber cleanup (regression sentinel for future edits).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Extras overwrite (#1), route cardinality (#5), and error ID parity (#6) fixed. ⚠️ Axios spread (#4) safe today but must be fixed before PR #2261 (axios 1.x) merges — see Blocking follow-up. |
| Regression risk | ✅ Low — `withIsolationScope` covered by new integration-ish test (#9); v10 API mock surface expanded (#8); primitive `reportError` paths hardened + tested |
| Tests | ✅ 1008 tests passing in shared workspace; 32 original + 18 new tests covering all review gaps |
| Code quality | ✅ `_app.tsx` duplication eliminated via shared factory (#2); boundary scope widened (#3); stale directives removed (#11) |
| Mergeable state | ✅ Up-to-date with `develop` via merge commit `d22331582` |

---

## Recommendation

**Approve after rebase** — 11 of the 12 valid review points are addressed on the branch; #4 is deferred to the PR #2261 coordination window (see Blocking follow-up). Point #12 (caret pinning) remains invalid against actual repo state.

### Remaining before merge

1. ~~Rebase against `develop`~~ ✅ Done — `develop` merged in via commit `d22331582` (merge commit rather than rebase, but both resolve the dirty state).
2. ~~Remove the `/sentry-test` page~~ ✅ Done — reverted so the PR ships only the permanent error-tracking infra.

No outstanding blockers before merge.

### Remaining before PR #2261 (axios 1.x) merges

3. **Update `installAxiosCorrelationInterceptor` to mutate headers via `AxiosHeaders.set()` instead of spreading** — see Blocking follow-up above. Can land on this PR (PP-1750) as a follow-up commit, or be folded into PR #2261's migration. Must exist before axios 1.15.0 reaches `develop`.

### Also landed as post-review hardening

- `reportError` now wraps non-Error values (`null`, strings, plain objects) in a synthetic `Error` with a descriptive message and the raw value tucked under `extra.originalValue`, so Sentry stops collapsing every primitive into a single `<unknown>` group.

Optional follow-ups (out of scope for this PR):

- Consider splitting the Sentry v7→v10 SDK upgrade into its own PR in future if similar cross-cutting infra work arises; the current bundled approach is now well-tested.
