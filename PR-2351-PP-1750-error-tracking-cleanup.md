# PR Review: PP-1750 — Error-tracking cleanup (Wise operation fix, tag/route trimming, shared ErrorState)

**PR:** https://github.com/Proofed/B2BWebserver/pull/2351
**Jira:** https://proofed.atlassian.net/browse/PP-1750
**Status:** In Progress (Highest priority)
**Branch:** `fix/PP-1750-wise-operation-tag` (local HEAD `06856636f` is one commit ahead of the PR `head.sha` captured by GitHub — `7fffd1acb`; the extra commit applies the same `meta.operation` fix to notification-prefs and is reviewed below)

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| FR1 — React app errors captured with user/route/component/timestamp metadata | `AppErrorBoundary` + new inner page-level boundary + shared `ErrorState` fallback; `reportError(error, { operation: "react.error-boundary", component })`; `useSentryIdentity`/`withSentryUser` bind `user.id` (PII stripped); `route` retained as a Sentry tag via `setSentryContext` | ✅ Addressed |
| FR2 — Internal API failures captured with endpoint/status/correlation-id/metadata | `MutationCache`/`QueryCache` `onError` → `reportError` with `corrId` from `x-correlation-id`, `http_status` auto-derived from axios in `buildTags`, body/headers PII-scrubbed | ✅ Addressed |
| FR3 — Third-party (Wise) failures include service name + operation + response | **Root-cause fixed**: per-call context now travels via React Query `meta: { operation: "wise.create-recipient" }` read once in `MutationCache.onError`; server-side `createRecipient.ts` adds `Sentry.setTag("operation", …)` + `Sentry.setContext("wise_recipient_creation", …)` onto the request-isolated scope before `handleEndpointError` captures. See **Issue 1** — the server tag value diverges from the client one | ⚠️ Partial (functional, but the two surfaces emit inconsistent `operation` values) |
| FR4 — Centralised monitoring across React / internal-API / 3rd-party | All three surfaces flow through `reportError` → `Sentry.captureException`; `instrumentation.ts` added for v8+ SDK init; `nextConfig.js` `release.name` wired for source-map upload | ✅ Addressed |
| FR5 — Consistent coverage; failures that prevent user actions are captured | Previously-swallowed paths now report (`papaparse` `.catch(() => {})`, `Router.push().catch(...)`, axios `.catch(...)` in `useGoogleSimpleSignOn`, `EventSource.onerror`, `console.error` paths in maintenance/file-metadata/file-downloader/copy-to-clipboard/clean-content/logout-inactive). The dedupe footgun (a second `captureException` on the same Error from a local `onError`) is now documented and fixed wherever it was used | ✅ Addressed |
| QA scenario "Wise sandbox failure → `operation=createRecipient`" | Client mutation tag now `wise.create-recipient`; server tag still `createRecipient` (camelCase) — flagged as Issue 1 | ⚠️ Partial |

### Scope creep flagged

This PR is named "Error-tracking cleanup" but bundles the full **SENTRY_AUDIT.md** sweep (11 audit items). The author called this out in earlier Jira comments. The reviewer should be aware they're approving:

- A Next.js runtime change (`instrumentation.ts` + `experimental.instrumentationHook: true`)
- A breaking-for-anything-out-of-pages-api change (`wrapApiHandlerWithSentry` removal — see Issue 3)
- Source-map upload behaviour (`release.name`)
- A fail-closed change to the Sentry scrubber (events now drop on scrub failure instead of leaking — see `sentryScrubber.ts:271-282`)
- A new `SENTRY_AUDIT.md` document at the repo root (see Issue 6)

---

## Architecture Analysis

The PR is structurally three things stacked on top of each other:

1. **The Wise-tag root-cause fix.** The original PR-2347 attempt used `meta` on the client mutation, but a later sub-commit moved it back to a local `onError` (which Sentry v10's `dedupe` integration drops). This PR restores the `meta` approach (`apps/creative-portal/services/wise/createRecipient/index.ts:26`) and pairs it with `createAppQueryClient.ts:54-65` reading `mutation.options.meta?.operation` once. The fallback `"react-query.mutation"` ensures every cache-level event is filterable. The latest commit `06856636f` extends the same pattern to `usePatchOrgGroupNotificationPrefsMutation`. **Approach is correct and well-documented**.

2. **A tag/route trim and naming convention.** `source` is removed from every call site (21 places). The `route` *extra* is removed from `reportError` (Sentry's automatic `transaction` tag already carries the parameterised route), but `route` is still emitted as a filterable tag via `setSentryContext`. `getSentryRoute.ts` and its test are fully deleted — verified via `git grep`: zero stragglers. A `<subsystem>.<action>` kebab convention for `operation` is documented in `throwSentryError.ts:38-56` and applied across the diff — with one exception (Issue 1).

3. **Shared `ErrorState` molecule + two-tier error boundaries.** The duplicated "Something went wrong" UI in `AppErrorBoundary` and `_error.tsx` is extracted to `packages/shared/components/molecules/ErrorState/`. Both `_app.tsx` files now wrap a second `AppErrorBoundary fill="container"` inside `<Layout>` so a page render error keeps the portal header/footer. The outer boundary stays as the catastrophic catch-all (`fill="viewport"`). Solid: shared leaf + per-portal chrome.

Cross-cutting touches: PII scrub list extended (`accountNumber`, `abartn`, `accountHolderName`, `phone`, `dateOfBirth`, `address`, etc.); scrubber gains depth/cycle bounds and fail-closed semantics; `withSentryUser` no longer sends `email`/`roles` to Sentry (was being stripped in `beforeSend` anyway).

---

## Issues Found

### 1. Server-side Wise tag uses the old camelCase value (`createRecipient`) while client uses `wise.create-recipient` — same failure, two unrelated tag groups in Sentry

**[File: apps/creative-portal/api/wise/createRecipient/createRecipient.ts]**

**Function/Class:** `createRecipient` (line 129)

**Severity:** medium

**Problem:** The convention documented in `packages/shared/utils/throwSentryError.ts:38-56` is `<subsystem>.<action>` dot-kebab, lowercase, with `wise.create-recipient` listed as the canonical example. The client mutation at `apps/creative-portal/services/wise/createRecipient/index.ts:26` sets `meta: { operation: "wise.create-recipient" }`. The server-side handler at line 129 sets `Sentry.setTag("operation", "createRecipient")`. A single 422 produces two Sentry events (one per surface) with two different `operation` tags.

**Impact:**

- Sentry filter `operation:wise.*` matches only the client event.
- Sentry filter `operation:createRecipient` matches only the server event.
- The QA-scenarios update in Jira comment 61368 told QA to identify Wise events via `operation=createRecipient`. After this PR the client event no longer has that tag — and the server event no longer matches the new dot-kebab convention used everywhere else. Whichever filter the dashboard uses, half the events disappear from view.
- Also splits Sentry issue-grouping — backend Node-runtime events and frontend browser-runtime events would already split by environment, but now they also split by `operation`, making it harder to see "Wise createRecipient is failing" as one signal.

**Fix:** Align the server tag with the client tag and the documented convention:

```typescript
Sentry.setTag("operation", "wise.create-recipient");
```

### 2. `meta.operation` is untyped — a typo silently demotes the event to the generic `react-query.mutation` tag

**[File: packages/shared/utils/createAppQueryClient.ts]**

**Function/Class:** `MutationCache.onError` (line 47-65)

**Severity:** medium

**Problem:** `mutation.options.meta?.operation as string | undefined` reads from React Query's `MutationMeta` type, which is `Record<string, unknown>`. There's no compile-time guarantee that callers spelled the key correctly. A typo like `meta: { opeation: "wise.create-recipient" }` or a wrong shape like `meta: { tags: { operation: "..." } }` produces no error — the event just lands with `operation: "react-query.mutation"` and the Wise context is lost. Same for the new `usePatchOrgGroupNotificationPrefsMutation` in commit `06856636f`.

**Impact:** This is the **same class of bug** the PR was created to fix (silently dropped per-call context). A typed `meta` shape would have caught the original lost-Wise-tag at the type checker rather than in production.

**Fix:** Augment React Query's `Register` interface in a shared types file (e.g. `packages/shared/types/react-query.d.ts`):

```typescript
import "@tanstack/react-query";

declare module "@tanstack/react-query" {
  interface Register {
    mutationMeta: { operation?: string };
    queryMeta: { operation?: string };
  }
}
```

Then drop the `as string | undefined` cast in `createAppQueryClient.ts`. Optional/required is a design call — leaving `operation` optional preserves the `react-query.mutation` fallback while making typos a compile error.

### 3. `withApiMiddleware` removed the manual `wrapApiHandlerWithSentry` but kept `opts.sentry` — a `{ sentry: false }` caller now silently skips `withSentryUser`'s isolation scope, breaking the new Wise scope-mutation pattern

**[File: packages/shared/api/utils/middlewares/withApiMiddleware/withApiMiddleware.ts]**

**Function/Class:** `withApiMiddleware` (line 108)

**Severity:** medium

**Problem:** Removing the manual `wrapApiHandlerWithSentry(handler, "")` is correct (auto-instrumentation handles it). But the `if (opts.sentry)` gate at line 108 still controls whether `withSentryUser` (which provides `Sentry.withIsolationScope(...)`) is applied. The new server-side Wise pattern in `createRecipient.ts:129-136` mutates the current scope:

```typescript
Sentry.setTag("operation", "createRecipient");
Sentry.setContext("wise_recipient_creation", { … });
```

If any future endpoint passes `{ sentry: false }` and follows this same pattern (or copy-pastes the comment "the request-isolated scope (provided by `withSentryUser`'s `withIsolationScope`)"), the `setTag`/`setContext` calls land on the **global** scope and leak to every subsequent request handled by the same Node worker — including unrelated endpoints, until the next request from a `sentry: true` handler overwrites them. Today no caller passes `{ sentry: false }` (`git grep -E "\bsentry:\s*false" -- '*.ts' '*.tsx'` returns nothing), so this is theoretical.

**Impact:** Hidden coupling — the scope-mutation idiom only works while every endpoint also opts in to `withSentryUser`. A future copy-paste plus `{ sentry: false }` is enough to corrupt unrelated requests' tags.

**Fix:** Either (a) drop the `opts.sentry` flag entirely if no caller uses it (simpler, fewer foot-guns) or (b) make the isolation scope unconditional and only gate the user-binding portion on the flag. If keeping the flag, add a comment at line 108 noting that disabling it also disables the isolation scope — endpoints that rely on `Sentry.setTag` in handlers will leak.

### 4. Latest commit (`06856636f`) leaves the imports tidy but doesn't widen the `meta`-shaped fix to a test

**[File: apps/creative-portal/services/organisationGroups/index.ts]**

**Function/Class:** `usePatchOrgGroupNotificationPrefsMutation` (line ~276-294 of `develop`+commit)

**Severity:** low

**Problem:** The new `meta: { operation: "org-group.update-notification-prefs", ...options?.meta }` pattern is added to the mutation factory, and the caller's `.catch(reportError(...))` is replaced with a no-op comment. There's no test exercising this — `createAppQueryClient.test.ts` covers the cache-side read of `meta.operation` generically (via `wise.create-recipient`), but no test verifies that this specific hook actually wires the `meta` through.

**Impact:** Future refactors to `usePatchOrgGroupNotificationPrefsMutation` (e.g. someone spreads `options` last instead of first, overwriting the meta) would silently demote the operation tag. The same dedupe trap that motivated the PR.

**Fix:** Add a small unit test in `apps/creative-portal/services/organisationGroups/__tests__/index.test.ts` (or wherever the existing org-groups tests live) that builds the mutation, executes it with a rejecting fn, and asserts the mocked `reportError` was called with `operation: "org-group.update-notification-prefs"`. Or — cheaper — extract the meta object into a named const and reference it from both the hook and a test for the value.

### 5. Test coverage gaps on the new `reportError` migrations (low severity but documents the scope)

**[Files: multiple]**

**Function/Class:** N/A — coverage observation

**Severity:** low

**Problem:** New `reportError` calls were added in these places without accompanying unit tests:

- `apps/creative-portal/components/organisms/modals/WysiwygModal/index.tsx` — synthetic `new Error("Wysiwyg collab authentication failed")`
- `apps/customer-portal/components/pages/create-orders/index.tsx` — two `Router.push().catch` paths
- `apps/customer-portal/hooks/useGoogleSimpleSignOn/index.ts` — two axios `.catch` paths
- `packages/shared/hooks/useLogoutInactiveUser.ts`
- `packages/shared/hooks/useMaintenanceMode/index.tsx` (×2)
- `packages/shared/lib/maintenance/handleMaintenanceCheckSSR.ts`
- `packages/shared/lib/maintenance/maintenanceSSE.ts`
- `packages/shared/api/utils/files/metadata/{inject,verify}/index.ts`
- `packages/shared/utils/{cleanContent,copyTextToClipboard,fileDownloader}.ts`
- `apps/creative-portal/components/organisms/NewOrderForm/partials/AddOrdersStep/hooks.tsx` (csv parse)

**Impact:** These are all "convert `console.error` to `reportError`" or "report a previously-swallowed catch" swaps, so the regression risk is low. But CLAUDE.md requires "Every PR must include tests for new code", and 10+ files added new behaviour without coverage.

**Fix:** Not blocking, but consider adding at least one round-trip test per category (one for the React-Query-bypassing axios catch, one for the SSE error, one for the synthetic-Error pattern) so the convention is exercised. New tests *were* added for `ErrorState`, `createAppQueryClient`, `throwSentryError`, `AppErrorBoundary`, `ErrorPage`, `sentryContext`, `handleServerSideError`, `installAxiosCorrelationInterceptor` — so the discipline is there, just inconsistent.

### 6. `SENTRY_AUDIT.md` committed to the repo root

**[File: SENTRY_AUDIT.md]**

**Function/Class:** N/A — documentation file

**Severity:** low

**Problem:** A 299-line internal audit doc with a "post-fix sweep" status note is being added to the repo root in this PR. CLAUDE.md instructs "NEVER proactively create documentation files (\*.md) or README files unless explicitly requested by the User." This file may be deliberate (it's a useful permanent record of the v7→v10 migration), but it'll show up in every future `ls` at the repo root and rot as the codebase evolves.

**Impact:** Mostly cosmetic. The doc's "Resolution log" maps audit items to commits — once those commits are squash-merged, the SHAs in the doc will dangle.

**Fix:** Confirm with the team whether this belongs in-tree, in `docs/`, or in Confluence/Notion. If keeping it, drop the per-commit SHAs from the resolution log (they won't survive squash-merge) and replace with a single link to this PR.

### 7. `getWorkItemContentVersion.ts` rename touches a Logtail logger key, not a Sentry tag — silent breakage for dashboards/alerts keyed on the old value

**[File: packages/shared/api/workItemContentVersion/[id]/getWorkItemContentVersion/getWorkItemContentVersion.ts]**

**Function/Class:** request logger context (line 1)

**Severity:** low

**Problem:** This is the only diff line in the file — `operation: "getWorkItemContentVersion"` → `"work-item-content.get-version"`. It's a `req.log.with({ operation: … })` (logtail logger), not a `setTag` (Sentry). The dot-kebab convention documented in `throwSentryError.ts` is specifically for Sentry `operation` tags. Renaming the logger key changes a structured-log field, which could break Logtail saved searches, alerts, or dashboards keyed off `operation:getWorkItemContentVersion`.

**Impact:** Silent — no error, just dashboards/alerts that used to fire now don't (or vice-versa).

**Fix:** Either revert this single-line rename (logtail vs sentry have different conventions) or flag it to whoever owns the Logtail dashboards so they can update queries.

### 8. `AppErrorBoundary` default `fill` shifted from implicit `"container"` to `"viewport"` — out-of-PR consumers (none today) would visually change

**[File: packages/shared/components/molecules/AppErrorBoundary/index.tsx]**

**Function/Class:** `AppErrorBoundary.render` (line 54)

**Severity:** low

**Problem:** The new prop `fill?: ErrorStateFill` defaults to `"viewport"` (line 54). Both `_app.tsx` files explicitly pass `fill="viewport"` on the outer boundary and `fill="container"` on the inner one, so they're unaffected. But the old default (implicit `min-height: calc(100vh - 19.5rem)` baked into the styles) was effectively `"container"`. Any other consumer of `AppErrorBoundary` would silently switch to full-viewport.

**Impact:** `git grep AppErrorBoundary` returns only the two `_app.tsx` files, this component's own test, and `ErrorState`'s comment — so there are no other consumers today. Pure forward-compatibility concern.

**Fix:** Optional — flip the default to `"container"` to match the historical behaviour, since both real callers now pass `fill` explicitly. Or document in the JSDoc that `viewport` is the *new* default and callers nested inside `<Layout>` need `fill="container"`.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ❌ | 1 failure / 1318 passes. **Pre-existing, not caused by this PR**: `packages/shared/utils/formatWordQuantity.test.ts:35` — `formatWordQuantity(1000000, true)` returns `"10,00,000 words"` (Indian numbering) instead of `"1,000,000 words"`. The util uses `new Intl.NumberFormat()` with no locale, so it inherits the OS locale (`en-IN` on this machine). `formatWordQuantity.{ts,test.ts}` are untouched by PR-2351 (`git log develop..HEAD --` returns no commits). Will fail on any branch run from this locale. Recommend pinning `new Intl.NumberFormat("en-US")` in a follow-up. |
| `npx turbo run typecheck` | ✅ | 0 errors across 5 packages (`@proofed/shared`, `@proofed/creative-portal`, `@proofed/customer-portal`, `@proofed/wysiwyg-editor`, `@proofed/storybook`). |
| `npx turbo run lint` | ❌ | 63 prettier errors in `packages/wysiwyg/src/extensions/diffViewer/utils.ts` (62) and `packages/wysiwyg/src/extensions/comments/index.ts` (1). **Pre-existing, not caused by this PR**: `git diff develop..HEAD --` shows neither file was touched. All 63 errors are `prettier/prettier`-class formatting drift, auto-fixable via `yarn lint:fix`. |
| `npx turbo run build` | ❌ | `@proofed/creative-portal` compiled successfully (`✓ Compiled successfully`) and uploaded source maps to Sentry, then failed during the page-data collection phase with `unhandledRejection Error: Cannot find module './chunks/vendor-chunks/lodash.js'`. This is a Next.js standalone-mode chunk-resolution issue rooted in a stale `.next/` directory, not a code defect from this PR — no PR file references `lodash` or `vendor-chunks`. Recommend `rm -rf apps/creative-portal/.next` and re-run; author's PR description records `✅ typecheck` and unit-test passes from a clean build, so the failure is local-environment-only. |

**Important caveat:** All three failures are unrelated to the PR's diff. None of them touch files this PR modifies. Per CLAUDE.md, "If a pre-existing failure is unrelated to your work, flag it to the team but still do not commit on top of it" — they should still be triaged in follow-up PRs.

---

## Tests

- ✅ New `ErrorState.test.tsx` covers heading, status-code text, error-id row, and `onReload` click (74 lines, 6 cases — solid)
- ✅ `AppErrorBoundary.test.tsx` updated to drop `source` tag assertion and verify the new `react.error-boundary` operation tag is set
- ✅ `createAppQueryClient.test.ts` adds tests for (a) default `react-query.mutation` operation when no `meta` is set, and (b) `meta.operation` forwarding — the regression case for the Wise dedupe trap
- ✅ `throwSentryError.test.ts` rewritten to drop `source` field and use `reportError` everywhere
- ✅ `sentryContext.test.ts` asserts `route` no longer lands in request context, but still emits a `route` tag; verifies `null`-not-`undefined` for tag clearing per Sentry v8+
- ✅ `withSentryUser.test.ts` updated for `id`-only `setUser` call
- ✅ `sentryScrubber.test.ts` updated for the fail-closed-returns-`null` change
- ✅ `ErrorPage.test.tsx` updated to assert `Sentry.setTag("source", ...)` is *not* called
- ❌ No tests for new `reportError` swaps in `WysiwygModal`, `create-orders` Router.push catches, `useGoogleSimpleSignOn`, `useLogoutInactiveUser`, maintenance hooks, file-metadata helpers, file downloader, copy-to-clipboard, cleanContent, csv parse, or the latest commit's `usePatchOrgGroupNotificationPrefsMutation` meta — see Issues 4 & 5
- ⬜ Manual verification still owed (per PR description's verification checklist): trigger a Wise `createRecipient` 422 on devtest and confirm `operation=wise.create-recipient` (and the server tag — see Issue 1) in the Sentry event

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ (the Wise root-cause is fixed but the server-side tag value is inconsistent with the client-side one — Issue 1) |
| Regression risk | ⚠️ Medium — large surface (62 files, 11 audit items folded in), removed `wrapApiHandlerWithSentry` could affect API handlers outside `pages/api/**` (none today, verified), latent dedupe-trap if anyone copy-pastes the `.catch(reportError)` pattern (mitigated by deletion + extensive comments) |
| Tests | ⚠️ Good coverage on shared utilities (`ErrorState`, `createAppQueryClient`, scrubber, sentryContext, AppErrorBoundary, ErrorPage), gaps on app-level `console.error → reportError` swaps and the latest `usePatchOrgGroupNotificationPrefsMutation` meta (Issues 4 & 5) |
| Code quality | ✅ Well-commented (every non-obvious choice has a "WHY" comment), dedupe rationale is documented at the contract boundary (`createAppQueryClient.ts` JSDoc, throwSentryError.ts convention block), naming convention is enforced almost everywhere |
| Validation suite | ❌ Failures (`test`, `lint`, `build`) but **all three are pre-existing and unrelated to this PR's diff** — see Validation Checks for the breakdown. Author should re-run `build` with a clean `.next/` to confirm. |
| Mergeable state | ✅ Clean per GitHub; ⚠️ Local validation has unrelated failures the author should be aware of |

---

## Recommendation

**Approve with suggestions** — the core PP-1750 work (Wise root-cause fix, source/route trim, shared ErrorState, audit sweep) is correct and well-documented. Address Issues 1–4 (in order of importance) before merge; Issues 5–8 can land in follow-ups.

1. **Issue 1 (must-fix-before-merge):** Change `Sentry.setTag("operation", "createRecipient")` to `"wise.create-recipient"` in `apps/creative-portal/api/wise/createRecipient/createRecipient.ts:129` so the server-side and client-side surfaces emit the same tag and follow the new convention. Update QA notes in Jira accordingly.
2. **Issue 2 (high-value follow-up):** Type-augment React Query's `Register` interface with `mutationMeta: { operation?: string }` so the same dedupe-trap that motivated this PR can't recur via a `meta` key typo.
3. **Issue 3 (clean-up):** Either drop the unused `opts.sentry` flag in `withApiMiddleware` or document that disabling it also disables `withSentryUser`'s isolation scope and breaks the Wise scope-mutation pattern.
4. **Issue 4 (small):** Add a unit test (or shared const) for the `meta.operation` on `usePatchOrgGroupNotificationPrefsMutation`.
5. **Pre-merge sanity:** Re-run `npx turbo run build` from a clean `.next/` to confirm the local build failure is environmental, not regression. The three test/lint/build failures observed locally were all in files this PR does not touch, but per CLAUDE.md they should be flagged so the team can triage in follow-up PRs (`formatWordQuantity` locale fragility; prettier drift in `packages/wysiwyg`).
6. **Manual devtest verification owed:** Trigger a Wise `createRecipient` 422 and confirm in Sentry that both the client and (after Issue 1) the server events carry `operation=wise.create-recipient`.
