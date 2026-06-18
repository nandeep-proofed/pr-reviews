# PR Review: PP-1750: Improve System Error Tracking and Logging

**PR:** https://github.com/Proofed/B2BWebserver/pull/2253
**Jira:** https://proofed.atlassian.net/browse/PP-1750
**Status:** Code Review (Jira) · PR open, `mergeable_state: clean`
**Branch:** `fix/PP-1750-improve-error-tracking` → `develop`
**Size:** 54 files, +3571 / −1189, 21 commits

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1.1 All unhandled React errors captured | `AppErrorBoundary` (`componentDidCatch → reportError`, `source=error-boundary`) wraps both apps' root; shared `ErrorPage`/`_error.tsx` captures SSR errors via `captureUnderscoreErrorException` | ✅ Addressed |
| 1.2 Errors interrupting user actions tracked | 14 `console.error` swallow sites migrated to `reportError`; react-query `QueryCache`/`MutationCache` `onError` | ⚠️ Partial — one swallow site missed (see Issue 3) |
| 1.3 Contextual metadata (user, route, component, timestamp) | user via `useSentryIdentity`/`withSentryUser`; route via `getSentryRoute`; component via `componentName`; timestamp by Sentry | ✅ Addressed (SSR route caveat — Issue 6) |
| 2.1 Internal API failures logged | `QueryCache`/`MutationCache` global `onError → reportError` (`source=react-query`) fires even when per-hook `onError` shadows defaults | ✅ Addressed |
| 2.2 endpoint, HTTP status, correlation ID, request metadata | `buildExtras` adds url/method/params/requestData/responseData; `http_status` tag auto-derived from axios; `x-correlation-id` via interceptor | ✅ Addressed (SSR/custom-instance gaps — Issue 5) |
| 3.1 Third-party (Wise) failures captured | `wise/createRecipient` `onError → reportError` (`source=wise`, `operation=createRecipient`) | ✅ Addressed |
| 3.2 service name, operation, status/message | `source=wise`, `operation=createRecipient`, `http_status` auto-derived | ✅ Addressed |
| 4 Centralized monitoring across all 3 surfaces | All paths route to Sentry with a `source` tag distinguishing react/api/third-party | ✅ Addressed |
| 5 Consistent coverage | Broad migration across both apps | ⚠️ Partial — BriefStep swallow missed; SSR/custom-axios uncovered |

**Scope beyond Jira (flagged):**
- **Major undocumented dependency upgrade** — `@sentry/nextjs` `7.73.0 → 10.48.0` (Issue 1). The PR description lists this as *deferred*.
- Sentry config hardening (env-aware `tracesSampleRate`, `debug:false`, `enabled` gate, removal of `transpileClientSDK`/`hideSourceMaps`, scrubber cleanup) — sensible quota/PII follow-ups, but beyond the strict ticket scope.
- `TASKS_COMPLETED.md` deleted (−653 lines) — consistent with the team decision to drop that log, but an unrelated deletion bundled into this PR.

---

## Architecture Analysis

The approach is sound and well-layered. A single `reportError(error, context)` core in `throwSentryError.ts` is the funnel for every surface (error boundary, react-query caches, third-party calls, migrated swallow sites), with `throwSentryError` preserved as a thin backward-compatible wrapper. Cross-cutting concerns are factored into reusable shared utilities (`createAppQueryClient`, `installAxiosCorrelationInterceptor`, `getSentryRoute`, `baseInit`) so both portals stay in lock-step — a good fit for the monorepo's `@proofed/shared` convention. PII handling is thoughtful: query/mutation keys and HTTP bodies are pre-scrubbed *before* serialization because the global `beforeSend` scrubber can't recurse into already-stringified extras. Non-Error throws are wrapped in synthetic Errors so Sentry can group/title them. The `AppErrorBoundary` correctly sits inside `ThemeProvider` so its styled fallback has theme access.

The main architectural caveats are (a) the response interceptor mutating the **global** Sentry scope (context bleed — Issue 2), and (b) the gap between the PR's stated scope and the actual SDK major-version jump that the new code structurally depends on (Issue 1).

---

## Issues Found

### 1. Sentry SDK upgraded v7→v10 despite description saying it's deferred

**[File: apps/creative-portal/package.json, apps/customer-portal/package.json, yarn.lock]**

**Function/Class:** dependency `@sentry/nextjs`

**Severity:** high

**Problem:** Both apps bump `@sentry/nextjs` from `7.73.0` to `10.48.0` (and drop `@sentry/types`), with ~1238 lines of `yarn.lock` churn. The new code structurally depends on v8+ APIs: `Sentry.withIsolationScope` (replaces v7 `runWithAsyncContext`), `Sentry.captureUnderscoreErrorException`, `getCurrentScope().getScopeData()`, and the functional `replayIntegration()`/`browserTracingIntegration()`. Yet the PR description's "Not in scope (deferred)" section explicitly says: *"Sentry SDK v7 → v9 upgrade (breaking API changes, separate ticket)."* The description's commit list (`fc6d94cc0`, `e9b6c76d9`, `c5d41adf8`) is also stale — the branch now has 21 commits.

**Impact:** A v7→v10 jump crosses multiple breaking-change boundaries (the v7→v8 rewrite especially). Because the description says the upgrade is deferred, reviewers and QA will not test it, and all five runtime verification checkboxes in the test plan (error boundary, 500 capture, Wise failure, header propagation, end-to-end corr_id) remain **unchecked**. `CLAUDE.md` still documents "Error Tracking: Sentry 7.73.0". Risk of undetected behavioral regressions in Sentry init, replay, tracing, and SSR capture on Next 14.1.1, plus potential interaction with `@logtail/next` (customer portal).

**Fix:** This is primarily a process/scope correctness issue — the upgrade itself is justified since the implementation requires v8+ APIs. Before merge: (1) correct the PR description to state the upgrade was performed and to what version; (2) update `CLAUDE.md`; (3) run the full runtime test plan on a deployed devtest/stage environment and check the boxes; (4) confirm `@sentry/nextjs@10` peer compatibility with Next 14.1.1 and that the tunnelRoute/sourcemap upload still work. Consider isolating the SDK upgrade into its own commit with a clear message for bisectability.

### 2. Response-error interceptor mutates the global Sentry scope (context bleed)

**[File: packages/shared/utils/installAxiosCorrelationInterceptor.ts]**

**Function/Class:** `installAxiosCorrelationInterceptor` (response-error handler)

**Severity:** medium

**Problem:** On every response error the interceptor calls `setSentryContext({ corr_id, http_status, route })`, which sets **tags on the current scope** (`Sentry.setTag`). In the browser there is a single shared scope, so these tags persist globally and attach to *subsequent, unrelated events* until overwritten.

**Impact:** A failed request stamps `corr_id=A` / `http_status=500` / `route=/x` onto the global scope. A later, unrelated render error caught by `AppErrorBoundary` (which sets `route` but not `corr_id`/`http_status`) is then reported carrying the **stale** `corr_id=A` and `http_status=500`. This produces mislabeled events, wrong correlation IDs, and phantom HTTP statuses — directly undermining the ticket's triage goal. The mutation is also largely redundant: `reportError` already attaches `corr_id` (read from the request header) and auto-derives `http_status` per-event from axios errors via `buildTags`.

**Fix:** Don't mutate the global scope from the interceptor. Either remove the `setSentryContext` call and rely on `reportError`'s per-event tagging, or, if you want the interceptor itself to report, capture inside an isolated scope:

```typescript
axios.interceptors.response.use(
  (response) => response,
  (error: AxiosError) => {
    Sentry.withScope((scope) => {
      scope.setTag("corr_id", corrId);
      scope.setTag("http_status", error.response?.status);
      scope.setTag("route", getSentryRoute());
      // optionally Sentry.captureException(error) here
    });
    return Promise.reject(error);
  }
);
```

### 3. "Zero console.error swallows remain in runtime app code" is inaccurate

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/BriefStep/hooks.tsx]**

**Function/Class:** support-document add handler (line ~300)

**Severity:** medium

**Problem:** A swallowing `catch` still logs only to console:

```typescript
} catch (error) {
  // eslint-disable-next-line no-console
  console.error("Support document add failed", error);
  showToast({ type: "error", text: "Failed to add support documents" });
}
```

This is the creative-portal analogue of the customer-portal `useSupportDocuments` swallow that the PR **did** migrate (`source=supportDocuments`). The PR description claims "Zero `console.error` swallows remain in runtime app code."

**Impact:** Support-document add failures in the creative portal stay invisible to Sentry, and the same failure mode is tracked in one app but not the other — contradicting requirement 5 (consistent coverage) and the PR's own completeness claim.

**Fix:** Migrate this site to `reportError`:

```typescript
} catch (error) {
  reportError(error, {
    source: "supportDocuments",
    operation: "onFilesAccepted",
    route: getSentryRoute()
  });
  showToast({ type: "error", text: "Failed to add support documents" });
}
```

### 4. Axios request/response headers attached to extras aren't header-aware-scrubbed

**[File: packages/shared/utils/throwSentryError.ts]**

**Function/Class:** `buildExtras`

**Severity:** low

**Problem:** `extras.requestHeaders = error.config?.headers` and `extras.responseHeaders = error.response?.headers` are attached raw. They are only scrubbed later by the global `scrubSensitiveData` (key-substring match), which redacts `authorization`/`auth`/`token` but **not** `cookie` / `set-cookie` — those are handled exclusively by `scrubHeaderValues`, which `sentryScrubber` applies to `request.headers`/metadata, **not** to `extra.*`.

**Impact:** In the browser this is largely moot (JS can't read `Set-Cookie`; the `Cookie` header isn't in `config.headers`). But `reportError` also runs on the SSR/server path (e.g. `tiptapAiChanges`, and any future server caller), where axios configs can carry forwarded cookies — those values could land unredacted in `extra.requestHeaders`.

**Fix:** Either drop full header objects from extras (they're noisy), or run them through `scrubHeaderValues` / strip `cookie`/`set-cookie` before attaching.

### 5. Custom axios instance and SSR requests bypass the correlation interceptor

**[File: packages/shared/utils/installAxiosCorrelationInterceptor.ts, apps/creative-portal/api/utils/tiptap/api.ts]**

**Function/Class:** `installAxiosCorrelationInterceptor`

**Severity:** low

**Problem:** The interceptor patches only the global `axios` default and is a no-op during SSR (`typeof window === "undefined"`). `tiptapApiClient = axios.create(...)` is a separate instance, and `getServerSideProps`/API-route outbound calls run server-side — none receive the `x-correlation-id` header or the response-error handling.

**Impact:** Requirement 2 (correlation ID on internal API requests) isn't covered for SSR-originated calls or custom-instance traffic. tiptap is third-party/server-side so this is mostly acceptable, but the gap should be acknowledged rather than implied-complete.

**Fix:** Document the browser-only scope explicitly, and/or install the request interceptor on shared custom instances and the server axios path if SSR correlation is in scope.

### 6. `ErrorPage.getInitialProps` route likely resolves to `/_error`

**[File: packages/shared/components/pages/error/index.tsx]**

**Function/Class:** `ErrorPage.getInitialProps`

**Severity:** low

**Problem:** `const route = contextData.pathname ?? contextData.asPath;` with a comment claiming `pathname` is the errored route's template. For the Pages-Router error page, `ctx.pathname` is typically `/_error`, so `route` would tag every SSR error as `/_error` rather than the real route template (`asPath` holds the concrete URL).

**Impact:** The `route` tag/context on SSR-captured errors loses its grouping value. Low severity but worth a quick check against a real deployed SSR error.

**Fix:** Verify the actual `contextData.pathname` value on a deployed SSR error; if it's `/_error`, prefer `contextData.asPath` (or derive the template another way).

### 7. `getSentryRoute` fallback returns concrete pathname (tag cardinality)

**[File: packages/shared/utils/getSentryRoute.ts]**

**Function/Class:** `getSentryRoute`

**Severity:** low

**Problem:** When the Next router isn't initialised it falls back to `window.location.pathname` (e.g. `/orders/42`). As a `reportError` *extra* this is fine, but `route` is also promoted to a **tag** via `setSentryContext` (interceptor + error boundary), and concrete paths inflate tag cardinality.

**Impact:** Minor Sentry tag-cardinality growth in the fallback case; the primary path correctly uses the route template.

**Fix:** Acceptable as-is; optionally only use the concrete path for an extra and avoid tagging it.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ Skipped — user opted out | Re-run before merge — required by CLAUDE.md |
| `npx turbo run typecheck` | ⏭️ Skipped — user opted out | Re-run before merge |
| `npx turbo run lint` | ⏭️ Skipped — user opted out | Re-run before merge |
| `npx turbo run build` | ⏭️ Skipped — user opted out | Re-run before merge — important given the SDK major upgrade |

> Validation suite was **not run** (user opted out). The SDK v7→v10 upgrade makes a clean local `test/typecheck/lint/build` (after `yarn install`) especially important before merge.

---

## Tests

- ✅ 9 new test files added (~1,520 lines): `throwSentryError` (564), `ErrorPage` (214), `withSentryUser` (165), `baseInit` (131), `installAxiosCorrelationInterceptor` (128), `createAppQueryClient` (119), `AppErrorBoundary` (110), `getSentryRoute` (66), `keys` (25).
- ✅ `__mocks__/@sentry/nextjs.ts` updated to the v10 surface (`withIsolationScope`, `getCurrentScope().getScopeData`, `captureUnderscoreErrorException`, functional integrations) — consistent with the upgrade.
- ✅ Meets the project rule that new code ships with tests, for the new shared utilities.
- ⚠️ No regression test asserts that the migrated swallow sites (tiptapAiChanges, serializerUtils, JobCard, notifications, OrderTemplates, teamMembersContext, useSupportDocuments, etc.) now call `reportError` — these behavioral migrations are unverified by tests.
- ⚠️ The five **runtime** verification items in the PR test plan are all unchecked.
- ❌ Automated validation suite not executed this review (user opted out).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Mostly correct; global-scope context bleed (Issue 2) and a missed swallow site (Issue 3) |
| Regression risk | ⚠️ Medium — driven by the undisclosed Sentry v7→v10 major upgrade (Issue 1) |
| Tests | ⚠️ Strong unit coverage for new utils; migrations + runtime paths unverified |
| Code quality | ✅ Good — clean layering, reuse-first, thoughtful PII handling |
| Validation suite | ⏭️ Skipped — user opted out (re-run before merge) |
| Mergeable state | ✅ Clean (GitHub) — but validation not run; treat as unverified |

---

## Recommendation

**Request changes** (primarily description/scope accuracy + two code fixes), then re-validate.

1. **Fix the PR description and `CLAUDE.md`** to reflect the actual `@sentry/nextjs` 7.73.0 → 10.48.0 upgrade; remove it from "deferred." Complete the runtime test plan on a deployed env (Issue 1).
2. **Remove the global `setSentryContext` mutation** from the axios response interceptor (or scope it with `withScope`) to stop cross-event context bleed (Issue 2).
3. **Migrate the remaining swallow** in `NewOrderForm/partials/BriefStep/hooks.tsx` so creative/customer portals are consistent (Issue 3).
4. **Run the mandatory validation suite** (`yarn install` first — deps changed — then `test` / `typecheck` / `lint` / `build`); all must pass before merge per CLAUDE.md. This was not run in this review.
5. *(Optional / low)* Scrub or drop header objects in extras (Issue 4); document the interceptor's browser-only scope (Issue 5); verify the SSR `route` value (Issue 6).
