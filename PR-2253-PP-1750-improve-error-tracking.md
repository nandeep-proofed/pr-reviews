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

### 1. Sentry SDK upgraded v7→v10 despite description saying it's deferred ✅ Resolved (2026-06-18)

**[File: apps/creative-portal/package.json, apps/customer-portal/package.json, yarn.lock]**

**Function/Class:** dependency `@sentry/nextjs`

**Severity:** high

**Problem:** Both apps bump `@sentry/nextjs` from `7.73.0` to `10.48.0` (and drop `@sentry/types`), with ~1238 lines of `yarn.lock` churn. The new code structurally depends on v8+ APIs: `Sentry.withIsolationScope` (replaces v7 `runWithAsyncContext`), `Sentry.captureUnderscoreErrorException`, `getCurrentScope().getScopeData()`, and the functional `replayIntegration()`/`browserTracingIntegration()`. Yet the PR description's "Not in scope (deferred)" section explicitly says: *"Sentry SDK v7 → v9 upgrade (breaking API changes, separate ticket)."* The description's commit list (`fc6d94cc0`, `e9b6c76d9`, `c5d41adf8`) is also stale — the branch now has 21 commits.

**Impact:** A v7→v10 jump crosses multiple breaking-change boundaries (the v7→v8 rewrite especially). Because the description says the upgrade is deferred, reviewers and QA will not test it, and all five runtime verification checkboxes in the test plan (error boundary, 500 capture, Wise failure, header propagation, end-to-end corr_id) remain **unchecked**. `CLAUDE.md` still documents "Error Tracking: Sentry 7.73.0". Risk of undetected behavioral regressions in Sentry init, replay, tracing, and SSR capture on Next 14.1.1, plus potential interaction with `@logtail/next` (customer portal).

**Fix:** This is primarily a process/scope correctness issue — the upgrade itself is justified since the implementation requires v8+ APIs. Before merge: (1) correct the PR description to state the upgrade was performed and to what version; (2) update `CLAUDE.md`; (3) run the full runtime test plan on a deployed devtest/stage environment and check the boxes; (4) confirm `@sentry/nextjs@10` peer compatibility with Next 14.1.1 and that the tunnelRoute/sourcemap upload still work. Consider isolating the SDK upgrade into its own commit with a clear message for bisectability.

**Resolution (2026-06-18):**

- ✅ **PR description rewritten** — added a "Dependency change — Sentry SDK v7 → v10" section naming the version jump, the v8+ APIs the new code requires (`withIsolationScope`, `captureUnderscoreErrorException`, `getCurrentScope().getScopeData()`, functional integrations), the `__mocks__/@sentry/nextjs.ts` v10 surface update, and the reviewer asks (runtime plan + `tunnelRoute`/sourcemap check on Next 14.1.1). The `Sentry SDK v7 → v9 upgrade` bullet has been removed from "Not in scope (deferred)". A note acknowledges the branch has grown to 22 commits beyond the three originally listed.
- ✅ **`CLAUDE.md:20`** updated from `Sentry 7.73.0` to `Sentry 10.48.0 (@sentry/nextjs)`.
- ⏳ **Runtime test plan** (5 unchecked items: error boundary, 500 capture, Wise failure, header propagation, end-to-end corr_id) — to be walked through on devtest/stage. The `/sentry-test` dev page added in commit `c3cc2397b` covers items 1–4 directly; item 5 needs a Logtail cross-reference.
- ⏳ **Peer compatibility / tunnelRoute / sourcemap upload** on Next 14.1.1 — to be smoke-tested on devtest deploy.

### 2. Response-error interceptor mutates the global Sentry scope (context bleed) ✅ Resolved (2026-06-18)

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

**Resolution (2026-06-18):**

- ✅ **Response-error interceptor removed** in `packages/shared/utils/installAxiosCorrelationInterceptor.ts`. The request interceptor (which stamps `x-correlation-id`) is unchanged. The response side previously called `setSentryContext({ corr_id, http_status, route })` on every failure, mutating the global scope; that call is gone.
- ✅ **Per-event tagging covers the same fields.** `reportError` (`packages/shared/utils/throwSentryError.ts:57,60-69,80`) already attaches `corr_id` (from the request header, forwarded by `createAppQueryClient`'s `QueryCache`/`MutationCache` onError), auto-derives `http_status` from axios errors, and includes `route` in extras — all scoped to the single event.
- ✅ **Tests updated** (`installAxiosCorrelationInterceptor.test.ts`). Two response-interceptor tests removed. Added a regression guard asserting `Sentry.setTag` / `Sentry.setContext` are never called from the interceptor and that the response interceptor is no longer installed. 6/6 tests pass; lint + typecheck clean.
- ℹ️ **Trade-off:** direct `axios.get/post` calls that bypass `reportError` (e.g., fire-and-forget calls outside react-query) lose the per-event `corr_id`/`http_status` tags they previously got from the global mutation. Missing tags on those events is strictly better than the wrong tags on subsequent unrelated events. If any such call needs Sentry context, it should be wrapped in `try/catch` + `reportError`.

### 3. "Zero console.error swallows remain in runtime app code" is inaccurate ✅ Resolved (2026-06-18)

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

**Resolution (2026-06-18):**

- ✅ **Swallow migrated** in `apps/creative-portal/components/organisms/NewOrderForm/partials/BriefStep/hooks.tsx` (`onFilesAccepted` catch block). Tags match the customer-portal counterpart: `source: "supportDocuments"`, `operation: "onFilesAccepted"`, `route: getSentryRoute()`, plus `extra: { selectedOrderIndexesBrief, ordersCount }` so reviewers can tell whether the failure was on a single order, a selection, or the all-orders write.
- ✅ **Repo-wide sweep clean.** `grep "console.error"` across `apps/` + `packages/shared` (excluding `*.test.*`, `__mocks__/`, the Sentry scrubber's own diagnostic logging, and `scripts/`) returns zero hits. The PR description's "Zero `console.error` swallows remain in runtime app code" is now actually accurate.
- ✅ **Typecheck + lint pass** on the changed file. No new tests added — this is a behavioural migration of a catch block, matched to the customer-portal pattern that already has equivalent coverage.

### 4. Axios request/response headers attached to extras aren't header-aware-scrubbed ✅ Resolved (2026-06-18)

**[File: packages/shared/utils/throwSentryError.ts]**

**Function/Class:** `buildExtras`

**Severity:** low

**Problem:** `extras.requestHeaders = error.config?.headers` and `extras.responseHeaders = error.response?.headers` are attached raw. They are only scrubbed later by the global `scrubSensitiveData` (key-substring match), which redacts `authorization`/`auth`/`token` but **not** `cookie` / `set-cookie` — those are handled exclusively by `scrubHeaderValues`, which `sentryScrubber` applies to `request.headers`/metadata, **not** to `extra.*`.

**Impact:** In the browser this is largely moot (JS can't read `Set-Cookie`; the `Cookie` header isn't in `config.headers`). But `reportError` also runs on the SSR/server path (e.g. `tiptapAiChanges`, and any future server caller), where axios configs can carry forwarded cookies — those values could land unredacted in `extra.requestHeaders`.

**Fix:** Either drop full header objects from extras (they're noisy), or run them through `scrubHeaderValues` / strip `cookie`/`set-cookie` before attaching.

**Resolution (2026-06-18):**

- ✅ **Scrub at the source.** Added `scrubHeadersForExtras` in `packages/shared/utils/throwSentryError.ts` — a two-pass helper that runs `scrubSensitiveData` (redacts whole header keys like `Authorization`/`apiKey`/`token`) and then `scrubHeaderValues` (parses the `cookie` header and redacts sensitive cookies). Both `extras.requestHeaders` and `extras.responseHeaders` go through it before being attached, so PII can't reach Sentry even on the SSR/server path where forwarded session cookies would otherwise land raw.
- ✅ **`scrubHeaderValues` exported** from `packages/shared/utils/sentryScrubber.ts` so it can be reused at the call site. No behaviour change inside the scrubber itself.
- ✅ **Tests added** in `throwSentryError.test.ts`: one asserting `Authorization`/`x-api-key` are `[REDACTED]` while non-sensitive headers (`Content-Type`, `x-request-id`) survive; one asserting `cookie: tracking=ok; session=...; theme=dark` is scrubbed to redact only `session`. 28/28 tests pass.
- ℹ️ Defense in depth: `sentryScrubber.beforeSend` still re-scrubs `event.extra` recursively, so even if a future caller bypasses the helper the scrubber catches the obvious sensitive keys — the new helper just closes the cookie-parsing gap that `scrubSensitiveData` alone couldn't handle.

### 5. Custom axios instance and SSR requests bypass the correlation interceptor ✅ Resolved (2026-06-18) — documented, no code fix

**[File: packages/shared/utils/installAxiosCorrelationInterceptor.ts, apps/creative-portal/api/utils/tiptap/api.ts]**

**Function/Class:** `installAxiosCorrelationInterceptor`

**Severity:** low

**Problem:** The interceptor patches only the global `axios` default and is a no-op during SSR (`typeof window === "undefined"`). `tiptapApiClient = axios.create(...)` is a separate instance, and `getServerSideProps`/API-route outbound calls run server-side — none receive the `x-correlation-id` header or the response-error handling.

**Impact:** Requirement 2 (correlation ID on internal API requests) isn't covered for SSR-originated calls or custom-instance traffic. tiptap is third-party/server-side so this is mostly acceptable, but the gap should be acknowledged rather than implied-complete.

**Fix:** Document the browser-only scope explicitly, and/or install the request interceptor on shared custom instances and the server axios path if SSR correlation is in scope.

**Resolution (2026-06-18):**

- ✅ **JSDoc on `installAxiosCorrelationInterceptor` expanded** with explicit "Scope" and "Out of scope" sections naming every uncovered path: custom axios instances (`axios.create()`), server-side outbound axios (`getServerSideProps`, API → API), and `fetch`/other HTTP clients. The interceptor's actual behaviour is unchanged.
- ✅ **`tiptapApiClient` annotated** in `apps/creative-portal/api/utils/tiptap/api.ts` with a comment explaining it is intentionally unpatched — third-party tiptap.cloud endpoint, own auth header, doesn't propagate `x-correlation-id`. Errors from callers are already reported with `source: "tiptap-ai-changes"` via direct `reportError` calls (migrated in commit `fc6d94cc0`).
- ℹ️ **Server-side outbound correlation NOT added** in this PR. The right fix there is request-scoped corr_id threading (read `req.headers["x-correlation-id"]` already surfaced by `withSentryUser`/`withApiMiddleware`, forward to outbound axios calls manually — or pull in AsyncLocalStorage). Both options are non-trivial design work and out of scope for PP-1750. The JSDoc now flags this gap so reviewers know to file a follow-up if SSR ↔ API correlation becomes a requirement.
- 📋 **Inbound chain (frontend → API) is already complete** via existing code: this interceptor stamps `x-correlation-id` on outbound browser requests; `withSentryUser` (`packages/shared/api/utils/middlewares/withSentryUser.ts:27-34`) reads it back on the server and forwards into Sentry scope + response header. So Requirement 2 is satisfied for the dominant path — only SSR-originated outbound traffic is uncovered, and that's acknowledged rather than implied-complete.

### 6. `ErrorPage.getInitialProps` route likely resolves to `/_error`

**[File: packages/shared/components/pages/error/index.tsx]**

**Function/Class:** `ErrorPage.getInitialProps`

**Severity:** low

**Problem:** `const route = contextData.pathname ?? contextData.asPath;` with a comment claiming `pathname` is the errored route's template. For the Pages-Router error page, `ctx.pathname` is typically `/_error`, so `route` would tag every SSR error as `/_error` rather than the real route template (`asPath` holds the concrete URL).

**Impact:** The `route` tag/context on SSR-captured errors loses its grouping value. Low severity but worth a quick check against a real deployed SSR error.

**Fix:** Verify the actual `contextData.pathname` value on a deployed SSR error; if it's `/_error`, prefer `contextData.asPath` (or derive the template another way).

### 7. `getSentryRoute` fallback returns concrete pathname (tag cardinality) ✅ Resolved (2026-06-18)

**[File: packages/shared/utils/getSentryRoute.ts]**

**Function/Class:** `getSentryRoute`

**Severity:** low

**Problem:** When the Next router isn't initialised it falls back to `window.location.pathname` (e.g. `/orders/42`). As a `reportError` *extra* this is fine, but `route` is also promoted to a **tag** via `setSentryContext` (interceptor + error boundary), and concrete paths inflate tag cardinality.

**Impact:** Minor Sentry tag-cardinality growth in the fallback case; the primary path correctly uses the route template.

**Fix:** Acceptable as-is; optionally only use the concrete path for an extra and avoid tagging it.

**Resolution (2026-06-18):**

- ✅ **Concrete-pathname fallback removed** in `packages/shared/utils/getSentryRoute.ts`. The function now returns `undefined` when no route template is available (Next router not initialised, pathname is `/_error`, etc.) instead of returning `window.location.pathname`. Updated JSDoc explains why and notes the route template is the only "tag-safe" value.
- ✅ **Tag-promotion sites stay clean automatically.** `setSentryContext` (`sentryContext.ts:51-53`) already ignores `undefined` fields, so `setTags({ route: undefined })` is a no-op — the high-cardinality concrete path never reaches the tag set. `AppErrorBoundary` and `ErrorPage` (the only remaining callers that promote `route` to a tag after Issue 2's interceptor removal) needed no changes.
- ✅ **Extras-only callers are unaffected.** `createAppQueryClient`, `useSupportDocuments` (both portals), `BriefStep/hooks.tsx`, `JobCard.tsx`, etc. all forward `route` as a `reportError` extra; the `buildExtras` builder simply skips undefined.
- ✅ **Two tests updated** in `getSentryRoute.test.ts`: the "fallback to window.location.pathname" and "/_error" cases now assert `undefined` (renamed to call out the tag-cardinality reason). 5/5 tests pass. Full shared suite stays green (1306/1306).
- ℹ️ **What's lost:** the concrete URL as a `reportError` extra in the router-not-ready edge case. Sentry's automatic browser instrumentation (`request.url`, `transaction.name`, navigation breadcrumbs) still captures the concrete URL on each event, so triage information isn't lost — just consolidated to the SDK-provided fields.
- 🧹 **Side fix:** the same edit pass restored `_hint?: EventHint` / `_hint?: BreadcrumbHint` parameters that an earlier linter run had over-pruned from `sentryScrubber` and `beforeBreadcrumb` (Sentry's API expects 2-arg callbacks; tests pass two). Now both match the Sentry signature again.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `vitest run` on `@proofed/shared` (full suite) | ✅ 1306/1306 pass | Run after the Issue 7 / Issue 4 edits; confirms no regression from the scrubber/getSentryRoute changes. |
| `tsc --noEmit` on `@proofed/shared` | ✅ Clean | Run after every issue's edit. |
| `tsc --noEmit` on `@proofed/creative-portal` | ✅ Clean | Run after Issues 3, 5, 7. |
| `tsc --noEmit` on `@proofed/customer-portal` | ✅ Clean | Run after Issue 7 (cross-app `getSentryRoute` signature change). |
| `eslint --max-warnings 0` on every touched file | ✅ Clean | Applied iteratively; lint-staged also ran via pre-commit hook. |
| `npx turbo run build` (full branch) | ⏭️ Not run this session | Re-run on devtest deploy — required by CLAUDE.md before merge, especially important given the SDK v7→v10 upgrade. |
| Runtime test plan (5 PR-body checkboxes) | ⏭️ Pending | Must walk through on devtest/stage; `/sentry-test` dev page covers 4/5. |

> The scoped validation above covers the **review fixes** (commit `716ae6719`). The whole-branch `turbo run build` and the runtime test plan still need to run on a deployed devtest before merge, per CLAUDE.md.

---

## Tests

- ✅ 9 new test files added by the PR itself (~1,520 lines): `throwSentryError` (564), `ErrorPage` (214), `withSentryUser` (165), `baseInit` (131), `installAxiosCorrelationInterceptor` (128), `createAppQueryClient` (119), `AppErrorBoundary` (110), `getSentryRoute` (66), `keys` (25). Strictly **8 new** files + the expanded `throwSentryError.test.ts` (was modified, not new) — the original review wording was slightly off and is corrected here.
- ✅ `__mocks__/@sentry/nextjs.ts` updated to the v10 surface (`withIsolationScope`, `getCurrentScope().getScopeData`, `captureUnderscoreErrorException`, functional integrations) — consistent with the upgrade.
- ✅ Meets the project rule that new code ships with tests, for the new shared utilities.
- ✅ **Review-fix tests added 2026-06-18** (commit `716ae6719`):
  - `throwSentryError.test.ts` — +2 cases: header keys (`Authorization`, `x-api-key`) redact to `[REDACTED]` while neutral keys (`Content-Type`, `x-request-id`) survive; cookie-string parsing redacts only the sensitive cookie (`session=...`) inside `cookie: tracking=ok; session=...; theme=dark`.
  - `installAxiosCorrelationInterceptor.test.ts` — −2 cases removed (response-interceptor tests for the now-deleted handler), +2 added including a regression guard asserting the interceptor never calls `Sentry.setTag`/`setContext`.
  - `getSentryRoute.test.ts` — 2 cases updated to assert `undefined` on the router-not-ready and `/_error` paths (was: concrete `window.location.pathname`).
  - Full `@proofed/shared` suite: 1306/1306 pass.
- ⚠️ No regression test asserts that the migrated swallow sites (tiptapAiChanges, serializerUtils, JobCard, notifications, OrderTemplates, teamMembersContext, useSupportDocuments, BriefStep) now call `reportError` — these behavioural migrations are unverified by tests.
- ⚠️ The five **runtime** verification items in the PR test plan are all unchecked. The `/sentry-test` dev page (commit `c3cc2397b`) covers items 1–4; item 5 (corr_id cross-reference Sentry ↔ Logtail) needs a deployed env.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Resolved — Issues 1, 2, 3, 4, 5, 7 all fixed 2026-06-18 in commit `716ae6719` |
| Regression risk | ⚠️ Low/medium — Sentry v7→v10 upgrade now disclosed in the PR + CLAUDE.md (Issue 1); runtime test plan still pending on devtest/stage |
| Tests | ✅ Strong unit coverage for new utils + 5 regression-guard cases added with the fixes; migrations remain unverified at runtime |
| Code quality | ✅ Good — clean layering, reuse-first, thoughtful PII handling; review fixes preserved the same style |
| Validation suite | ✅ Scoped (typecheck/lint/test on touched files + full `@proofed/shared` suite) clean. ⏭️ Whole-branch `turbo run build` still owed. |
| Open issues | ⚠️ Issue 6 (`ErrorPage.getInitialProps` route may resolve to `/_error`) still open — flagged as "verify on real deployed SSR error," low severity. |
| Mergeable state | ✅ Clean (GitHub); branch has 1 unpushed commit (`716ae6719`). Runtime test plan + whole-branch build still required before merge. |

---

## Recommendation

Original verdict was **request changes**. After the 2026-06-18 fix pass (commit `716ae6719`) the substantive code defects are resolved; what remains is one low-severity verification item and the standard pre-merge checks.

1. ~~**Fix the PR description and `CLAUDE.md`** to reflect the actual `@sentry/nextjs` 7.73.0 → 10.48.0 upgrade; remove it from "deferred."~~ ✅ Done 2026-06-18 — PR description rewritten + `CLAUDE.md` updated. Still owed: complete the runtime test plan on a deployed env and confirm tunnelRoute/sourcemap upload (Issue 1).
2. ~~**Remove the global `setSentryContext` mutation** from the axios response interceptor (or scope it with `withScope`) to stop cross-event context bleed (Issue 2).~~ ✅ Done 2026-06-18 — response-error interceptor removed entirely; per-event tagging via `reportError` covers the same fields; test suite updated with a regression guard.
3. ~~**Migrate the remaining swallow** in `NewOrderForm/partials/BriefStep/hooks.tsx` so creative/customer portals are consistent (Issue 3).~~ ✅ Done 2026-06-18 — migrated to `reportError` with matching tags; repo-wide sweep confirms zero `console.error` swallows remain in runtime code.
4. **Run the mandatory whole-branch validation suite** before merge — `yarn install` first (deps changed), then `npx turbo run test / typecheck / lint / build`. Required by CLAUDE.md. Scoped checks ran clean during the fix pass but the full pipeline still needs to execute.
5. *(Optional / low)* ~~Scrub or drop header objects in extras (Issue 4);~~ ✅ Done 2026-06-18. ~~document the interceptor's browser-only scope (Issue 5);~~ ✅ Done 2026-06-18 — JSDoc expanded with Scope / Out-of-scope sections; `tiptapApiClient` annotated. **Verify the SSR `route` value on a real deployed error (Issue 6 — still open).** ~~Kill the concrete-path fallback in `getSentryRoute` (Issue 7).~~ ✅ Done 2026-06-18 — fallback now returns `undefined`; `setSentryContext` skips the tag automatically.

---

## Fix commit & related notes

- **Commit:** `716ae6719` — `PP-1750: Address PR review — fix scope mutation, headers, route cardinality` — 12 files changed, +220 / −81. Local-only; not yet pushed to `origin/fix/PP-1750-improve-error-tracking`.
- **One unrelated file swept in:** `packages/shared/config/sentry/client.config.ts` carried a pre-existing working-tree change that removed `googletagmanager`/`hotjar`/`stripe` from `THIRD_PARTY_DENY_URLS`. The reviewer attempted to unstage it before commit, but lint-staged's processing pipeline re-included it. Unrelated to any of the seven review issues; either intentional (then keep) or accidental (then amend the commit to restore those three regex patterns). Flagged to the PR author for a decision.
- **Review-doc location:** `pr-reviews/PR-2253-PP-1750-improve-error-tracking.md` (this file) is gitignored — these resolution annotations are local-only and won't ship with the PR. If reviewers elsewhere need them, copy/paste the relevant Resolution blocks into the PR conversation.
