# PR Review: PP-1750: Error-tracking tag cleanup — Wise operation fix + remove source & redundant route

**PR:** https://github.com/Proofed/B2BWebserver/pull/2351
**Jira:** https://proofed.atlassian.net/browse/PP-1750
**Status:** In Progress
**Branch:** `fix/PP-1750-wise-operation-tag` → `develop`
**Author:** nandeep-proofed

---

## Jira Requirements vs Implementation

PP-1750 is the umbrella "Improve System Error Tracking and Logging" ticket; this PR is a follow-up refinement addressing a QA finding (comment 61506) plus two cleanups.

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| FR1 — React errors captured with user id, route/page, component, timestamp | `AppErrorBoundary` still captures React errors with `component` + `componentStack`; user via `useSentryIdentity`; timestamp automatic. **Route/page now comes from Sentry's automatic `transaction` tag** rather than an explicit `route` tag (the explicit client tag was dropped). | ✅ Addressed (see Issue #2) |
| FR2 — Internal API errors with endpoint, status, corr id, metadata | Unchanged. Cache handler still sends `corrId`, `queryKey`/`mutationKey`, `http_status`; `url`/`method` from axios extras. | ✅ Addressed |
| FR3 — Third-party (Wise) errors with service, operation, status | **Core fix.** `operation=createRecipient` now survives to Sentry via `meta` + single cache-level capture; previously lost to the `dedupe` integration. | ✅ Addressed |
| FR4 — Centralized monitoring (Sentry) | Unchanged. | ✅ Addressed |
| FR5 — Consistent coverage | `source` removal + `route`-extra removal are consistency refinements; all 21 call sites updated. | ✅ Addressed |

**Scope beyond the ticket:** the PR also (a) extracts a shared `ErrorState` molecule and adds a nested page-level `AppErrorBoundary`, and (b) silently renames several `operation` tag values. (a) is a reasonable bundled refinement; (b) is undocumented — see Issue #3.

---

## Architecture Analysis

The change has three parts, all coherent with the existing error-tracking design:

1. **Wise `operation` fix (root cause, correct).** Two `captureException` calls hit the same `Error` instance — the global `MutationCache.onError` (no operation) first, then the mutation's local `onError` (with operation) — and Sentry v10's `dedupe` integration drops the second, so the context-less event won. The fix carries per-call context through React Query `meta: { operation }`, read once in `MutationCache.onError`, and deletes the local `onError`. This collapses to a single capture that keeps the tag. The accompanying regression test (`forwards mutation meta.operation…`) locks the behavior in.

2. **`source` tag removal.** Dropped from `ReportErrorContext`, `buildTags`, and all 21 call sites. Rationale (errors are identifiable by `operation`/`url`/key) is sound.

3. **`route` *extra* removal + `getSentryRoute` deletion.** The `route` Additional-Data extra duplicated Sentry's automatic `transaction` tag, so it's removed everywhere, and the `getSentryRoute` helper is deleted. I verified repo-wide that **every** one of the 12 importers had its `getSentryRoute` import removed — no dangling references remain on the PR branch. The `route` scope **tag** survives on the server middleware and the Next `_error` page (whose `getInitialProps` computes route from Next context, not `getSentryRoute`).

The `ErrorState` extraction is clean: it de-duplicates the near-identical fallback UI that previously lived in both `AppErrorBoundary/styles.ts` and `pages/error/styles.ts`, adds a `fill: "viewport" | "container"` prop, and is well unit-tested. Wrapping each portal's `<Component/>` in a second, `fill="container"` boundary inside the layout is a genuine resilience improvement (a page-render crash no longer blanks the whole app chrome).

---

## Issues Found

### 1. Removing the Wise hook-level `onError` re-enables the default error toast → duplicate / unwanted toast in the payment flow

**[File: apps/creative-portal/services/wise/createRecipient/index.ts]**

**Function/Class:** `useCreateRecipientMutation`

**Severity:** high

**Problem:** The fix replaces the mutation's hook-level `onError` with `meta: { operation }` and removes the callback entirely. But `createAppQueryClient` sets `defaultOptions.mutations.onError = showDefaultErrorToast`. In React Query, a mutation's hook-level `onError` **overrides** the default-options `onError`, whereas a `mutate(vars, { onError })` call-level callback runs **in addition** to it. So previously the local `onError` (reportError) suppressed the default toast; now that it's gone, `showDefaultErrorToast` fires at the observer level **plus** whatever the consumer's call-level `onError` does.

Both consumers already own Wise error UX in their `mutate()` call-level `onError`:
- `PaymentDetailsStep2/hooks.ts:308` — parses the Wise error payload and calls `showDefaultErrorToast(error)` / `showDefaultErrorToast(error, wiseMessage)`.
- `PaymentDetailsStep/hooks.ts:120` — for field-level Wise validation errors (those with a `path`) it calls `setFieldError(...)` and shows **no toast** by design.

**Impact:** After this PR, a Wise `createRecipient` failure shows a generic "Something went wrong" toast from the default handler **in addition** to the consumer's own handling. In `PaymentDetailsStep`, a field-level validation error (e.g. malformed account number) that was previously surfaced only inline via `setFieldError` will now **also** pop a generic toast. This is a user-visible regression in the exact payment flow the PR targets.

**Fix:** Keep a no-op hook-level `onError` so the default toast stays suppressed, while still routing `operation` via `meta`. This matches the established pattern in this repo (`services/orders/create/index.ts`, `services/orders/createOrderNew`, `services/users/index.ts`, `services/self/phoneNumber` all use `onError: () => {}` precisely to suppress the default toast when the consumer handles errors). A no-op `onError` does **not** call `reportError`, so it does not reintroduce the dedupe problem — the single cache-level capture with `meta.operation` is preserved.

```typescript
export const useCreateRecipientMutation = () =>
  useMutation(postCreateRecipient, {
    meta: { operation: "createRecipient" },
    // Consumers own Wise error UX (inline field errors + specific
    // toasts), so suppress the default `showDefaultErrorToast`. A no-op
    // here does NOT re-capture the error, so the single cache-level
    // capture (which reads `meta.operation`) is unaffected.
    onError: () => {}
  });
```

Please confirm with a quick manual test: trigger a Wise `createRecipient` failure (e.g. a 422) from both Payment steps and verify only one toast (the specific one) appears, and that field-level errors remain inline-only.

---

### 2. PR description contradicts the implementation regarding `getSentryRoute` and the client `route` tag

**[File: packages/shared/components/molecules/AppErrorBoundary/index.tsx]**

**Function/Class:** `AppErrorBoundary.componentDidCatch` (and the PR description)

**Severity:** medium

**Problem:** The PR description (point 3) states the `route` scope **tag** set via `setSentryContext` "(client `useSentryIdentity`/`AppErrorBoundary`/error page …) is intentionally **kept**, along with the `getSentryRoute` helper." The diff does the opposite for the client boundary:
- `getSentryRoute.ts` and its test are **deleted**.
- `AppErrorBoundary` removes `route` from its `setSentryContext({ … })` call, so it no longer sets a `route` tag.
- `useSentryIdentity` never set a `route` tag in the first place (it only sets user/env/release) — verified in `packages/shared/hooks/useSentryIdentity.ts`.

**Impact:** Client-side React errors caught by `AppErrorBoundary` no longer carry the explicit `route` Sentry tag; they fall back to Sentry's automatic `transaction` tag (the route template). This is functionally acceptable (the `_error` page and server middleware still set the `route` tag, and `transaction` covers FR1's "route/page"), so it is **not a functional gap** — but the description misrepresents what the code does, which will confuse QA/reviewers verifying the route metadata and anyone reasoning about the change later.

**Fix:** Update the PR description to state that `getSentryRoute` is removed, the client `AppErrorBoundary` route tag is dropped, and client React errors now rely on the automatic `transaction` tag. Confirm with the team that `transaction` is an acceptable substitute for the explicit client `route` tag.

---

### 3. Undocumented `operation` tag renames make the new *primary* identifier vaguer

**[File: apps/creative-portal/contexts/teamMembersContext/provider.tsx (+ JobCard.tsx, OrderTemplates/hooks.ts, notifications/hooks.ts)]**

**Function/Class:** multiple `reportError` call sites

**Severity:** medium

**Problem:** Beyond removing `source`, the PR silently changes several `operation` values to a nearby variable/handler name — not mentioned anywhere in the PR description:
- `teamMembersContext`: `loadFiltersFromStorage` → `savedFiltersJson`; `saveFiltersToStorage` → `teamFilterFields` (these are local *variable* names, not operations).
- `OrderTemplates`: `checkLiveOrdersBeforeDelete` → `liveOrders` (a local variable name).
- `JobCard`: `updateCurrentJobAfterAbort` → `handleAbortSuccess` (the enclosing handler — but the failing operation is the `mutateOrderProps.mutateAsync` job update, not "abort success").
- `notifications`: `initializeNotificationPreferences` → `updateOrgGroup` (matches the failing mutation — this one is reasonable).

**Impact:** The PR's stated rationale for removing `source` is that `operation`/`url`/key already identify errors — which makes `operation` the **primary** subsystem identifier going forward. Renaming several values to vaguer nouns (`savedFiltersJson`, `teamFilterFields`, `liveOrders`) undercuts that rationale and degrades Sentry grouping/searchability for those errors.

**Fix:** Keep the descriptive verb-phrase `operation` values (or, if the renames are intentional, call them out in the PR description with the reasoning). At minimum revert the three variable-name renames to operation-describing labels.

---

### 4. `meta.operation` is read as `unknown` via a cast

**[File: packages/shared/utils/createAppQueryClient.ts]**

**Function/Class:** `MutationCache.onError`

**Severity:** low

**Problem:** `operation: mutation.options.meta?.operation as string | undefined` casts an `unknown` (React Query's `MutationMeta` defaults to `Record<string, unknown>`). This typechecks today, but the cast hides typos in the `meta` key and offers no compile-time guarantee that `operation` is a string.

**Impact:** Minor. A future caller writing `meta: { operaton: "…" }` (typo) or `meta: { operation: 123 }` would not be caught.

**Fix (optional):** Augment React Query's `MutationMeta` once in a shared `.d.ts` so `meta.operation` is typed, dropping the cast:

```typescript
declare module "@tanstack/react-query" {
  interface MutationMeta {
    operation?: string;
  }
}
```

---

## Validation Checks

> Run in-place on the PR tip (`848f2fc7e`), reusing existing `node_modules` (the PR changes **no** `package.json`/`yarn.lock`). `TIPTAP_PRO_TOKEN` is unset, so a fresh-worktree install was not possible. `core.autocrlf=true` on this Windows checkout. Build was stopped early at the user's request. Note: this repo reports **0 CI check runs** on the PR — there is no CI gate.

| Check | Result | Notes |
|---|---|---|
| `npx turbo run typecheck` | ✅ | 0 errors across all 5 typecheck tasks (shared, creative-portal, customer-portal, wysiwyg, storybook). Confirms the `meta.operation` cast and `ErrorStateFill` type wiring compile. |
| `npx turbo run test` | ⚠️ | 1 failure, **not PR-attributable**: `api/orders/createNew/__tests__/jobTimings.test.ts` — `Hook timed out in 10000ms` (a `beforeAll`/`beforeEach` timeout, not an assertion), in a file this PR does not touch, while build+lint ran concurrently (CPU contention). creative-portal: 1673/1674 pass; all other workspaces pass. PR-relevant suites (throwSentryError, createAppQueryClient, AppErrorBoundary, ErrorState, sentryContext) pass. |
| `npx turbo run lint` | ⚠️ | `customer-portal` failed with 22 `prettier/prettier` errors — **all in files this PR does not touch** (OrderPriceSection, Header, useDeliveryCalculationsWithDocuments, CustomDelivery, ShippingOptions, etc.). Pre-existing/environmental (installed prettier `3.5.3` matches the lock; `autocrlf` CRLF involvement). The PR's own 28 changed files pass `prettier --check` ("All matched files use Prettier code style!"). |
| `npx turbo run build` | ⏸️ | Not completed — terminated early at the user's request to stop gate checks. Inconclusive. |

---

## Tests

- ✅ New `ErrorState` molecule has a dedicated test (6 cases: heading/button, statusCode message, generic message, errorId present/absent, `onReload` click).
- ✅ `createAppQueryClient.test.ts` adds the key regression test — `forwards mutation meta.operation so the single cache-level capture carries it` — plus updates the no-source/no-route expectations.
- ✅ `throwSentryError.test.ts`, `sentryContext.test.ts`, `AppErrorBoundary.test.tsx` updated to drop `source`/`route` assertions; obsolete `getSentryRoute.test.ts` removed with the helper.
- ❌ **No test covers the toast behavior of the Wise mutation** — the regression in Issue #1 (default toast now fires) is exactly the kind of thing a `PaymentDetailsStep` hook test asserting "showToast/showDefaultErrorToast called once" would catch. The existing `PaymentDetailsStep2/hooks.test.ts` mocks the mutation, so it won't catch the default-handler path. Add coverage when fixing #1.
- ⚠️ Local validation suite is not all-green, but the test/lint failures are pre-existing/environmental and not attributable to this PR (see Validation Checks).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Wise `operation` fix is correct, but removing the local `onError` introduces a duplicate/unwanted toast (Issue #1) |
| Regression risk | ⚠️ Medium — user-visible double toast / unexpected toast on field validation in the Wise payment flow |
| Tests | ⚠️ Good coverage for the Sentry-context changes; missing coverage for the Wise toast behavior |
| Code quality | ✅ Clean `ErrorState` extraction, no dangling `getSentryRoute` refs, correct dedupe root-cause fix |
| Validation suite | ⚠️ typecheck pass; test/lint have only pre-existing/environmental failures (not PR-attributable); build not completed |
| Mergeable state | ✅ Clean (GitHub reports no merge conflicts) |

---

## Recommendation

**Request changes** — the Sentry `operation`-tag fix is correct and the refactor is clean, but there is one real code defect in the PR's own target flow.

1. **(Blocker) Fix Issue #1** — add `onError: () => {}` to `useCreateRecipientMutation` so the default `showDefaultErrorToast` stays suppressed; keep `meta: { operation }`. Add a hook test asserting the toast count, and manually verify both Payment steps.
2. **(Should) Update the PR description (Issue #2)** — it claims `getSentryRoute` and the client `AppErrorBoundary` route tag are kept; both are removed. Confirm the automatic `transaction` tag is the accepted substitute for the client boundary.
3. **(Should) Revert / document the `operation` renames (Issue #3)** — keep descriptive labels for the new primary identifier.
4. **(Optional) Issue #4** — augment `MutationMeta` to drop the `unknown` cast.
5. **(Process) Re-run the validation suite from a clean checkout** (`yarn install` with `TIPTAP_PRO_TOKEN`, LF line endings) for an authoritative all-green; the local test/lint failures here are pre-existing/environmental, but this repo has no CI gate, so a clean local run is the only safety net.
