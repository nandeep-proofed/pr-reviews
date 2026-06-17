# PR Review: PP-1750: Improve System Error Tracking and Logging

**PR:** https://github.com/Proofed/B2BWebserver/pull/2253
**Jira:** https://proofed.atlassian.net/browse/PP-1750
**Status:** Open
**Mergeable state:** ❌ **CONFLICTING** — branch is 12 commits behind `origin/develop` and 4 files conflict (see "Rebase required" below)
**CI status:** no checks reported on latest push
**Last reviewed at:** 2026-06-17 (commit `765f73031` — most recent merge from `develop`)

---

## Resolution Status

Every code-level fix from the prior review round is verified present on the branch head. Test count grew alongside `develop` (now **1292/1292 passing** in `@proofed/shared`, up from the 1008 reported in the previous review).

| # | Issue | Status | Verified on commit |
|---|---|---|---|
| 1 | Caller-supplied `extra` overwritten by axios details | ✅ Resolved — `buildExtras` returns `{ ...extras, ...(context.extra ?? {}) }` so caller wins. Backed by `caller-supplied extra wins over built-in axios fields` test. | `af8f89c78` (carried through `765f73031`) |
| 2 | Duplicated QueryClient + interceptor bootstrap | ✅ Resolved — extracted to `packages/shared/utils/createAppQueryClient.ts`; both `_app.tsx` files import it. | `af8f89c78` |
| 3 | Narrow error boundary scope | ✅ Resolved — `<AppErrorBoundary>` now sits as the outermost wrapper inside `<ThemeProvider>` in both apps, covering `QueryClientProvider`, all context providers, `Layout`, `ModalQueue`, `FeaturesProvider`/`VerifyOnboarding`. Placing it inside `ThemeProvider` is intentional so the fallback UI can resolve theme tokens. | `af8f89c78` |
| 4 | Axios `AxiosHeaders` prototype stripped by spread | ⚠️ **Conditional** — code still spreads `{ ...config, headers: { ...headers, ... } as typeof config.headers }` in `installAxiosCorrelationInterceptor.ts:35-42`. Safe on `develop` today (`axios ^0.27.2`, headers are plain objects). Becomes a real runtime hazard the moment [PR #2261](https://github.com/Proofed/B2BWebserver/pull/2261) (`axios ^0.27.2 → ^1.15.0`, still open and CONFLICTING) merges. See "Blocking follow-up" below. | — |
| 5 | High-cardinality `route` tag | ✅ Resolved — `getSentryRoute()` (new helper at `packages/shared/utils/getSentryRoute.ts`) prefers `Router.router?.pathname` (template `/orders/[id]`) and falls back to `window.location.pathname`. Used uniformly by `installAxiosCorrelationInterceptor`, `AppErrorBoundary`, `createAppQueryClient`. `ErrorPage.getInitialProps` uses `contextData.pathname` (Next's template form during SSR) — same intent, different surface. | `af8f89c78` |
| 6 | Error ID shown in boundary but not in `ErrorPage` | ✅ Resolved — `getInitialProps` returns `{ ...props, errorId }`; the page renders it via `<Styled.ErrorPageErrorId>Error ID: {errorId}</Styled.ErrorPageErrorId>`. | `af8f89c78` |
| 7 | Unsafe `as Error & { statusCode?: number }` cast | ✅ Resolved — narrowed to `as { statusCode?: number } \| null` (error/index.tsx:55-57). | `af8f89c78` |
| 8 | Sentry mock missing v10 API surface | ✅ Resolved — `packages/shared/__mocks__/@sentry/nextjs.ts` exports `withIsolationScope`, `withScope`, `replayIntegration`, `browserTracingIntegration`, `setUser`, `flush`, and a `getCurrentScope()` returning `getScopeData()` (v8+ API). | `af8f89c78` |
| 9 | No test for `withIsolationScope` migration | ✅ Resolved — `withSentryUser.test.ts` (5 tests) including `preserves tag/context writes made inside a nested async handler` which exercises the cross-await semantics. | `af8f89c78` |
| 10 | Module-scope `installAxiosCorrelationInterceptor()` call | ✅ Resolved — both apps invoke it inside `useEffect(..., [])` (creative-portal/_app.tsx:104-107, customer-portal/_app.tsx:90-92). | `af8f89c78` |
| 11 | Stale `/* eslint-disable no-console */` | ✅ Resolved — `sentryContext.ts` no longer carries the directive. | `af8f89c78` |
| 12 | `^10.48.0` caret range | ✅ Resolved — both `package.json` files now pin to exact `"@sentry/nextjs": "10.48.0"`. Note: the original framing ("project convention is exact pinning") is wrong in general — the repo uses caret ranges almost everywhere — but the pre-existing `@sentry/nextjs` entry on `develop` **was** exact-pinned (`"7.73.0"`), so keeping the exact-pin convention after a major-version bump is defensible. | `8dbf0b9de` |
| 13 | No test for `reportError` with non-Error primitives | ✅ Resolved — 5 new primitive tests in `throwSentryError.test.ts`: `null`, `undefined`, string, plain object, and a sentinel that real `Error` instances stay unwrapped (`originalValue` not added). | `af8f89c78` + `c6ee03005` |
| 14 | No direct test for `_app.tsx` QueryCache/MutationCache wiring | ✅ Resolved — `createAppQueryClient.test.ts` (5 tests) covers query `onError`, mutation `onError`, `showDefaultErrorToast` wiring, undefined-route fallback, and configured-instance return. | `af8f89c78` |

### Post-review hardening (carried forward)

| Change | Commit |
|---|---|
| Wrap non-Error `reportError` inputs in synthetic `Error("Non-Error thrown: …")` so Sentry shows searchable titles instead of `<unknown>`; raw value preserved at `extra.originalValue` | `c6ee03005` |
| Pin `@sentry/nextjs` to exact `10.48.0` to prevent silent drift on a freshly-upgraded SDK (closes review #12) | `8dbf0b9de` |
| Wider `reportError` / scrubber hardening across call sites | `db0dc0855` |

> The `/sentry-test` manual verification page (commits `c3cc2397b` and `069833755`) was reverted before merge — the automated suite covers every error-tracking path, and keeping a dev-only page reachable on stage/devtest created noise without lasting value.

### Verification (latest run on `765f73031`)

- `npx turbo run test --filter=@proofed/shared` → **1292/1292 passing**, 130 test files
- `npx turbo run typecheck --filter=@proofed/shared` → no new errors introduced by this PR (pre-existing errors in `Loader`, `Typography`, `iron-session` are orthogonal and exist on `develop`)
- `npx turbo run lint --filter=@proofed/shared --filter=@proofed/creative-portal --filter=@proofed/customer-portal` → clean for PP-1750 files

---

## Rebase required before merge

`gh pr view` reports `mergeStateStatus: DIRTY`, `mergeable: CONFLICTING`. The branch's last `develop` merge was `765f73031`, but `origin/develop` has advanced by 12 commits since (latest tip `054c02f83`). A 3-way merge produces conflicts in **4 files**:

```
apps/creative-portal/components/molecules/tables/TableWithFilters/tableColumns.tsx
apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/DetailedOrderInfo.tsx
apps/customer-portal/package.json
yarn.lock
```

None are core error-tracking files — they're collateral damage from:
- **`tableColumns.tsx` / `DetailedOrderInfo.tsx`** — touched by PR #2306 (`feature/PP-1867: Reorganize order side panel`) on `develop` and by the PP-1750 sweep that migrated `console.error` swallow sites.
- **`apps/customer-portal/package.json`** — develop added/reorganised deps (PR #2232 dead-deps cleanup) while PP-1750 already changed `@sentry/nextjs` pin in the same file.
- **`yarn.lock`** — mechanical, regenerated by `yarn` after the package.json conflict resolves.

These conflicts are tractable (no semantic overlap with the error-tracking work). Merge `develop` into the branch, resolve each one preserving both intents (Sentry v10 pin + dead-deps cleanup; PP-1867 layout + migrated `reportError` call sites), regenerate `yarn.lock`, and re-run the full suite.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1.1 Capture all unhandled React errors | `AppErrorBoundary` wraps the app tree inside `<ThemeProvider>`; shared `_error.tsx` calls `captureUnderscoreErrorException` for SSR errors. | ✅ Addressed |
| 1.2 Errors that interrupt user actions are tracked | `QueryCache`/`MutationCache` global `onError` (via `createAppQueryClient`) + migration of 14 `console.error` swallow sites to `reportError`. | ✅ Addressed |
| 1.3 Contextual metadata (user, route, component, timestamp) | `reportError` accepts `source`, `operation`, `route`, `component`, `queryKey`, `mutationKey`, `httpStatus`, `extra`, `corrId`; `setSentryContext` adds identifiers/request scope. Timestamp is Sentry-default. | ✅ Addressed |
| 2.1 Log internal API failures | Axios response interceptor pushes `corr_id`/`http_status`/`route` into Sentry scope; React Query `onError` reports with axios details. | ✅ Addressed |
| 2.2 Error includes endpoint / status / corr_id / request metadata | `reportError` serializes `url`, `method`, `params`, `requestData`, `responseData`, `responseStatus`, `requestHeaders`, `responseHeaders` when axios error detected. | ✅ Addressed |
| 3.1 Third-party (Wise) failures captured | `useMutation(postCreateRecipient, { onError: () => reportError(..., { source: "wise", operation: "createRecipient" }) })`. | ✅ Addressed |
| 3.2 Service name / endpoint / status | `source` tag + `operation` tag + axios extras provide all three. | ✅ Addressed |
| 4 Centralized monitoring | All paths route through `Sentry.captureException`; corr_id tag joins frontend Sentry with backend Logtail events. | ✅ Addressed |
| 5 Consistent logging coverage | 14 swallow sites migrated; zero `console.error` swallows remain in changed files. | ✅ Addressed |

**Beyond-scope changes also present:**

- Sentry SDK upgrade `7.73.0` → `10.48.0` (major × 3 bumps). Called out as "deferred" in the original PR body but landed in the same PR. High-risk on its own; mitigated here by the expanded mock surface + new tests for the migrated APIs.
- `transpileClientSDK: true` removed (drops IE11 transpilation — IE11 EOL'd 2022).
- Trace sampling changed from 100% everywhere to tiered (devtest 1.0 / stage 0.5 / production 0.1) — good change, reduces Sentry transaction quota burn, but out of original ticket scope.
- Scrubber cleanup (deduplicate `SENSITIVE_FIELDS`, remove redundant re-scrub of the whole request).

---

## Architecture Analysis

The approach is sound: a shared `reportError(error, context)` utility funnels every error through `Sentry.captureException` with consistent tags (`source`, `operation`, `http_status`, `corr_id`) and structured extras. Around this:

- `AppErrorBoundary` — hand-rolled React boundary (correct; no hook equivalent exists). Wrapped INSIDE `<ThemeProvider>` so the fallback UI can read theme tokens; this means a crash *inside* `ThemeProvider` itself would not be caught — acceptable trade-off since Emotion's `ThemeProvider` is functionally inert.
- `installAxiosCorrelationInterceptor` — stamps every outgoing axios request with `x-correlation-id` and pushes scope metadata on response errors. Now invoked inside `useEffect` (no module-scope side effect).
- `QueryCache`/`MutationCache` global `onError` (via `createAppQueryClient`) — catches errors even when per-hook `onError` shadows the default. Also extracts `corr_id` from the failed request's headers and forwards it to `reportError`.
- `_error.tsx` — shared SSR error page re-exported in both apps; surfaces the Sentry event id for support quoting.

The Sentry SDK bump (v7 → v10) is the largest semantic shift. It rewrites `sentryContext.ts` from the legacy `Scope.getContext()` API to `getScopeData().contexts`, replaces `Sentry.runWithAsyncContext` with `Sentry.withIsolationScope` in `withSentryUser.ts`, and switches from class-based `new Sentry.Replay(...)` to functional `Sentry.replayIntegration(...)`. The `withIsolationScope` behavioural change is now covered by `withSentryUser.test.ts`'s nested-async test, which is the right shape — though end-to-end behaviour in production should still be eyeballed in stage before final merge.

---

## Issues Found (all addressed; preserved for changelog/audit)

### 1. `reportError` caller-supplied `extra` is silently overwritten by axios details ✅ FIXED

**[File: packages/shared/utils/throwSentryError.ts]** `buildExtras` now ends with `return { ...extras, ...(context.extra ?? {}) }`. Caller wins. Test: `caller-supplied extra wins over built-in axios fields`.

### 2. Duplicated QueryClient + interceptor bootstrap across both `_app.tsx` files ✅ FIXED

**[File: packages/shared/utils/createAppQueryClient.ts]** Extracted factory returns a configured `QueryClient` with `queryCache`/`mutationCache` wired to `reportError`. Both portals call `createAppQueryClient()` inside `useState`. 5 tests in `createAppQueryClient.test.ts`.

### 3. Error boundary scope is too coarse — Layout / providers are outside it ✅ FIXED

**[File: apps/*/pages/_app.tsx]** `<AppErrorBoundary componentName="...">` is now the outermost wrapper inside `<ThemeProvider>` in both apps. Render crashes in any context provider, `Layout`, `ModalQueue`, `FeaturesProvider`, `VerifyOnboarding`, or any page component are now caught and surfaced via Sentry + fallback UI.

> **Important caveat that's worth keeping in mind:** the boundary intentionally lives *inside* `<ThemeProvider>` so the fallback UI can resolve theme tokens. A throw in `ThemeProvider` itself would escape — Emotion's `ThemeProvider` is functionally inert, so this trade-off is correct.

### 4. Axios interceptor spreads config, likely stripping `AxiosHeaders` prototype ⚠️ DEFERRED

**[File: packages/shared/utils/installAxiosCorrelationInterceptor.ts:35-42]** Code still uses the spread pattern:

```ts
return {
  ...config,
  headers: { ...headers, [CORRELATION_HEADER]: corrId } as typeof config.headers
};
```

Under axios `^0.27.2` (current `develop` and this branch), `config.headers` is a plain object, so the spread is safe today. Under axios `^1.15.0` (queued in [PR #2261](https://github.com/Proofed/B2BWebserver/pull/2261) — still OPEN and itself CONFLICTING), `config.headers` is an `AxiosHeaders` class instance and the spread strips the prototype, breaking any downstream `.set`/`.get`/`.has` call.

This is a **deferred-but-real** concern, not a current bug. See "Blocking follow-up" below for the precise replacement.

### 5. High-cardinality `route` tag from `window.location.pathname` ✅ FIXED

**[File: packages/shared/utils/getSentryRoute.ts]** New helper prefers `Router.router?.pathname ?? Router.pathname` (Next route template) and falls back to `window.location.pathname` only when the router is uninitialised. The `_error.tsx` SSR path uses `contextData.pathname` (also template form). 5 tests in `getSentryRoute.test.ts` cover the fallback chain, the `/_error` skip, and the SSR-undefined case.

### 6. Error ID shown in `AppErrorBoundary` fallback but not in `ErrorPage` ✅ FIXED

**[File: packages/shared/components/pages/error/index.tsx]** `getInitialProps` returns `{ ...props, errorId }`; the page renders `Error ID: {errorId}` when present.

### 7. Unsafe `as Error & { statusCode?: number }` cast in `ErrorPage.getInitialProps` ✅ FIXED

**[File: packages/shared/components/pages/error/index.tsx:55-57]** Narrowed to `as { statusCode?: number } | null`.

### 8. `packages/shared/__mocks__/@sentry/nextjs.ts` was incomplete for v10 API surface ✅ FIXED

**[File: packages/shared/__mocks__/@sentry/nextjs.ts]** Now exports `captureException`, `captureUnderscoreErrorException`, `setTag`, `setContext`, `setUser`, `flush`, `getCurrentScope` (returns `getScopeData`-shaped stub), `withIsolationScope`, `withScope`, `replayIntegration`, `browserTracingIntegration`.

### 9. No test coverage for `runWithAsyncContext` → `withIsolationScope` migration ✅ FIXED

**[File: packages/shared/api/utils/middlewares/withSentryUser.test.ts]** 5 tests cover the wrap, nested-async tag/context propagation, incoming corr-id reuse, user session attachment, and query-param extraction.

### 10. `installAxiosCorrelationInterceptor()` called at module top-level of `_app.tsx` ✅ FIXED

**[File: apps/*/pages/_app.tsx]** Both apps call it inside `useEffect(..., [])`. The `isInstalled` idempotency flag remains as a safety net.

### 11. Stale `/* eslint-disable no-console */` in rewritten `sentryContext.ts` ✅ FIXED

**[File: packages/shared/utils/sentryContext.ts:1]** Removed.

### 12. `@sentry/nextjs` uses `^10.48.0` while neighbouring deps are pinned ✅ FIXED

**[File: apps/*/package.json]** Both now pin to `"@sentry/nextjs": "10.48.0"` (exact).

### 13. No test for `reportError` with non-Error primitive inputs ✅ FIXED

**[File: packages/shared/utils/throwSentryError.test.ts]** New `non-Error primitive inputs` describe block adds 5 tests: `null`, `undefined`, string, plain object — each asserting the synthetic `Error("Non-Error thrown: …")` wrap and `extra.originalValue` storage — plus a sentinel that real `Error` instances stay unwrapped.

### 14. No direct test for `_app.tsx` QueryCache/MutationCache wiring ✅ FIXED

**[File: packages/shared/utils/createAppQueryClient.test.ts]** 5 tests cover the factory's outputs and Sentry forwarding for both queries and mutations.

---

## Tests

- ✅ `throwSentryError` / `reportError`: 22 tests (tags, extras, truncation, axios merge, user extras, returns event id, primitives, caller-wins).
- ✅ `AppErrorBoundary`: 5 tests (children, default fallback, custom fallback, Sentry report, event id render).
- ✅ `ErrorPage`: 11 tests (status code, generic message, getInitialProps scope enrichment, errorId thread-through).
- ✅ `installAxiosCorrelationInterceptor`: 7 tests (install idempotency, header stamping, caller-id passthrough, missing headers, scope push, missing `error.config`).
- ✅ `getSentryRoute`: 5 tests (template pathname, `Router.pathname` fallback, `window.location.pathname` fallback, `/_error` skip, SSR undefined).
- ✅ `createAppQueryClient`: 5 tests (return shape, query onError, mutation onError, toast wiring, undefined-route).
- ✅ `withSentryUser`: 5 tests (isolation-scope wrap, nested async, header corr-id reuse, session user, query-param extraction).
- ✅ `sentryContext` (existing): unchanged tests adjusted for v10 mock shape.
- ❌ No test asserting `AxiosHeaders` instance preservation through the interceptor — gated on issue #4 / PR #2261.
- ❌ No test asserting `SENSITIVE_FIELDS` scrubber still redacts auth/token substrings after the scrubber cleanup. Existing `sentryScrubber.test.ts` covers behaviour but a regression sentinel for the deduplicated paths would be cheap insurance.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Extras overwrite (#1), route cardinality (#5), and error-id parity (#6) all resolved. ⚠️ Axios spread (#4) is safe today (axios 0.27.2) but must be fixed before PR #2261 merges. |
| Regression risk | ✅ Low — Sentry v10 mock surface mirrors actual usage, `withIsolationScope` migration covered by `withSentryUser.test.ts`, primitive `reportError` paths hardened + tested. |
| Tests | ✅ 1292/1292 passing in `@proofed/shared`; 5 new test files (`createAppQueryClient`, `getSentryRoute`, `installAxiosCorrelationInterceptor`, `withSentryUser`, plus expanded `throwSentryError` and `ErrorPage`/`AppErrorBoundary` suites). |
| Code quality | ✅ `_app.tsx` duplication eliminated via shared factory (#2); boundary scope widened (#3); stale directives removed (#11). |
| Mergeable state | ❌ **CONFLICTING** — branch is 12 commits behind `origin/develop`; 4 files conflict (none in error-tracking code). Rebase / merge `develop` required. |

---

## Recommendation

**Approve after rebase.** Every reviewable code concern is addressed. The remaining gates are environmental, not architectural.

### Remaining before merge

1. **Merge `origin/develop` into the branch (or rebase) and resolve the 4 conflicts.** None overlap with error-tracking code; conflict resolution is mechanical:
   - `tableColumns.tsx` / `DetailedOrderInfo.tsx` — keep both the PP-1867 layout changes (from develop) and the migrated `reportError` call sites (from this branch).
   - `apps/customer-portal/package.json` — keep the `@sentry/nextjs: "10.48.0"` pin + the dead-deps cleanup from PR #2232.
   - `yarn.lock` — regenerate via `yarn install` after the JSON resolves.
2. Re-run `npx turbo run test`, `typecheck`, `lint`, and `build` after the merge — expect green across the board.
3. After CI passes, the PR is ready to merge.

### Remaining before PR #2261 (axios 1.x) merges

3. **Update `installAxiosCorrelationInterceptor` to mutate headers via `AxiosHeaders.set()` instead of spreading.** Drop-in replacement:

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

   Can land here as a follow-up commit OR be folded into PR #2261's axios migration. Whichever merges first owns the fix. Add a test that asserts `config.headers instanceof AxiosHeaders` after the interceptor runs.

   Note also: `createAppQueryClient`'s corr-id extraction reads `headers["x-correlation-id"]` which is safe under both axios 0.27 and 1.x (AxiosHeaders exposes property accessors), so no change needed there.

### Optional follow-ups (out of scope)

- Consider splitting future Sentry SDK upgrades into standalone PRs. This one bundles the SDK bump with the error-tracking infra; the bundled approach is well-tested here, but the precedent is risky.
- Add a regression sentinel test for `SENSITIVE_FIELDS` redaction after the scrubber dedup, so future edits can't accidentally widen the leak surface.
- Consider nested boundaries around known crash hotspots (Tiptap editor regions, OrderCreation flow) so a local failure doesn't blank the whole app — the current single boundary catches everything, but a localised fallback would degrade gracefully instead.
