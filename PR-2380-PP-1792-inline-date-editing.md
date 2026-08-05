# PR Review: feature/PP-1792 — Inline due date & deadline editing on Order Management dashboard

**PR:** https://github.com/Proofed/B2BWebserver/pull/2380
**Jira:** https://proofed.atlassian.net/browse/PP-1792
**Status:** Code Review (Story, High priority)
**Author:** nandeep-proofed · **Branch:** `feature/PP-1792-inline-date-editing-v2` → `develop`
**Size:** 20 files, +2702 / −1067 (initial); review fixes in commit `608b325f3`

---

## ✅ Review fixes applied (commit `608b325f3`)

All valid findings addressed on `feature/PP-1792-inline-date-editing-v2`:

- **Issue 1 (skeleton stuck) — Resolved.** The per-line skeleton is now driven by the real mutation lifecycle (`onJobMutationStart` sets it, `updateJobs`' `finally` / `onJobMutationSettled` clears it) instead of an optimistic pre-guard `addUpdatingDateType`. A rejected PUT, a guard that blocks before dispatch, or a no-op value can no longer leave the row loading.
- **Issue 2 (floating rejection) — Resolved.** `updateJobs` wraps the dispatch in `try/catch/finally`; the per-call `onError` still toasts, the `catch` stops the fire-and-forget rejection floating.
- **Issue 3 (missing component tests) — Resolved.** Added `InlineJobDueDate` (apply-guard + mode), `InlineOrderDeadline` (Live guard / warning / happy path / skeleton), `DeadlineWarningModal` (double-click guard), and a hook test for skeleton-clear-on-PUT-rejection. +14 tests.
- **Issue 4 (stale order-deadline buffer) — Resolved.** Buffer recomputes via an effect once the lazy fetch resolves; no longer reads `0h 0m` on first open.
- **Req 2 (icon not hover-revealed) — Resolved.** Icon is hidden by default and revealed on `:hover`/`:focus-within` of the date value, implemented with a `data-inline-calendar-trigger` attribute selector (no Emotion component selector, so it survives the test transform). Per-date hover per the ticket wording — confirm against Figma if whole-row hover is intended.
- **Issue 5 (`PP-1792-PLAN.md`) — Resolved.** Removed from the repo root.

**Post-fix gate (run on the v2 branch):** `test` 1775 pass · `typecheck` clean · `lint` clean · `build` 0 warnings.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1. Due date & deadline fields support inline editing | `InlineJobDueDate` (current-job line) + `InlineOrderDeadline` (order deadline line) rendered by `DeadlineCellContent` | ✅ Addressed |
| 2. Hovering a date shows a calendar icon | `IconTrigger` is `opacity:0` by default and revealed on `:hover`/`:focus-within` of the date row (`data-inline-calendar-trigger` selector) — fixed in `608b325f3` | ✅ Addressed (confirm hover scope vs Figma) |
| 3. Icon colour matches the date's colour state (overdue = red) | `iconColor={isOverdue ? theme.colors.red : color}`; `useIsOverdue` mirrors `DeadlineDisplay`'s `remainingTime < 0` red logic | ✅ Addressed |
| 4. Hovering the icon turns it green, matching the side-panel pattern | Reused `IconButton` inherits `currentColor` and hovers to `green1` | ✅ Addressed |
| 5. Clicking the icon opens a calendar overlay | `Dropdown` + `DueDatePicker` / `DeadlineDatePicker` | ✅ Addressed |
| 6. Overlay uses the existing date-change flow (job DD pattern + order deadline pattern) | Job line reuses `useJobDueDatePickers` + `useUpdateJobReturnBase`; order line mirrors `DetailedOrderInfo` order-PUT flow | ✅ Addressed |
| 7. User can submit the updated date | `onApply` on both pickers | ✅ Addressed |
| 8. On success: updated date shown + styling/icon refresh | `refetchDashboard()` invalidates `DASHBOARD_ORDERS_QUERY_KEY` + `SEARCH_ORDERS_FOR_TABLE`; prop-diff effect refreshes state | ✅ Addressed (see Issue 1 for the User-DD skeleton edge) |
| 9. Applies wherever due date/deadline shown in the OM dashboard | `DeadlineCell` in `TableWithFilters` | ✅ Addressed |
| 10 & 11. Not available for Completed/Cancelled — no icon, non-interactive | `DeadlineCellContent` returns `null` when `FINISHED_ORDER_STATUSES.includes(orderStatus)` | ✅ Addressed |
| 12. Existing validation, permissions and rules unchanged | Sidebar logic extracted 1:1 into `useUpdateJobReturnBase` / `useJobDueDatePickers`; two independent diffs confirm behaviour preservation | ✅ Addressed |

**Scope beyond the ticket (justified):** the PR performs two refactor-extractions (`useUpdateJobReturnBase` from `useUpdateJobReturn`; `useJobDueDatePickers` from `JobCard.tsx`) so the dashboard can reuse the sidebar's dynamic-deadline flow context-free. This is necessary to satisfy Req 6/12 and was verified behaviour-preserving. **Out-of-scope artifact:** `PP-1792-PLAN.md` (169 lines) committed at repo root — see Issue 5.

**Known backend dependency (not a PR defect):** the current-job line displays `currentJobDeadline` = `currentJobReturnTime ?? currentJobMaxReturnTime`, which needs Order Search to expose `currentJobMaxReturnTime` (Jira comments 55758 / 57002) and a DB migration to backfill legacy jobs (comment 56729). Until that lands, unassigned-job Job DDs fall back to `returnTime` and legacy-job job-DD PUTs reject (comment 55687) — the latter directly triggers Issue 1 below.

---

## Architecture Analysis

The approach is sound and reuse-first, matching the project's conventions:

- **Two clean extractions.** `useUpdateJobReturnBase` is the context-free validation/dispatch core; `useUpdateJobReturn` is now a thin wrapper that binds `useOrderSidebarContext`. `useJobDueDatePickers` lifts all picker orchestration (seeds, TZ frames, bounds, buffers, pristine, inline errors, the three update fns) out of `JobCard.tsx`. Both extractions were diffed old-vs-new line-by-line: **no validation rule, bound, buffer, dispatch payload, default, boolean, or error path changed** on the sidebar path. The only deltas are inert additions for the dashboard (`seedDateTime`, `onJobMutationSettled`, `order | undefined`, `index === -1`) that resolve to identical values when the sidebar omits them.
- **Row-level shared data.** `useInlineOrderData(orderId)` does one lazy (`enabled:false`) order fetch per row, shared by both editors, and owns the per-line skeleton set, the local prop-driven deadline-warning modal, and the context-free handlers. Good separation; `DeadlineCellContent` is `memo`'d and UI-only.
- **Design-token reuse.** Icon colour, overdue red, and the `DeadlineDisplay` coalescing all reuse existing atoms/utilities rather than reimplementing.

The residual risk lives entirely in the **new dashboard-only paths**, and that's where the issues below concentrate — chiefly the optimistic per-line skeleton lifecycle.

---

## Issues Found

### 1. User-DD inline edit can leave a permanent loading skeleton on the row — ✅ RESOLVED (`608b325f3`)

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/DeadlineCellContent/partials/InlineJobDueDate/index.tsx]**

**Function/Class:** `InlineJobDueDate` `handleApply` (in concert with `useInlineOrderData.updateJobs` and `useUpdateJobReturnBase.updateReturnTime`)

**Severity:** medium

**Problem:** `handleApply` sets the optimistic skeleton *before* dispatch:

```typescript
const handleApply = () => {
  if (currentJobIndex < 0) return;
  addUpdatingDateType("jobDueDate");
  bundle.onApply();            // not awaited, not caught
};
```

The **Job DD** path (`updateMaxReturnTime` → internal `mutatePartialJob`) is safe: the mutation is created with `onSettled: onJobMutationSettled`, which fires on **both success and error** and calls `removeUpdatingDateType("jobDueDate")`. The **User DD** path is not symmetric — `updateReturnTime` dispatches via the injected `updateJobs` handler, which has **no settle hook**:

```typescript
// useInlineOrderData.updateJobs
const updateJobs = useCallback(async (jobsToUpdate: Job[]) => {
  await Promise.all(jobsToUpdate.map((updatedJob) =>
    mutateJob(updatedJob, { onError: (error) => showDefaultErrorToast(error) })
  ));
  await refetchDashboard();     // ← skipped if the PUT rejects
}, [mutateJob, refetchDashboard]);
```

So for the User DD path the skeleton is cleared **only** by the container's prop-diff effect after a successful, value-changing refetch. It is never cleared when:
- **the job PUT rejects** — `mutateAsync` rejects even with `onError`, so `Promise.all` rejects, `refetchDashboard()` never runs, the prop never changes, and the skeleton sticks (plus an unhandled rejection — see Issue 2). This is a *live* scenario: legacy jobs currently reject with `returnWindowsMinutes: Required` (Jira comment 55687), and any transient 5xx/network error hits the same path.
- **an early-return guard fires without dispatching** — `updateReturnTime` returns after only a toast for the `status === "Approved"` guard and the out-of-sequence guard (neither is mirrored by a `disableApply`/inline-error, so both are reachable). `updateJobs` is never called, so nothing clears the skeleton.
- **the edit resolves to the same displayed value** — refetch returns an unchanged `currentJobDeadline`, so the prop-diff effect never fires.

**Impact:** After a failed or no-op inline User-DD edit, that row's job-due-date line shows a loading skeleton indefinitely until the user reloads / the row remounts. Because the legacy-job rejection is a known current backend state, this is reachable today, not just theoretically.

**Fix:** Give the User-DD dispatch the same settle guarantee the Job-DD path has. Minimum — clear the skeleton in a `finally` on the handler:

```typescript
const updateJobs = useCallback(
  async (jobsToUpdate: Job[]) => {
    try {
      await Promise.all(
        jobsToUpdate.map((updatedJob) =>
          mutateJob(updatedJob, {
            onError: (error) => showDefaultErrorToast(error)
          })
        )
      );
      await refetchDashboard();
    } finally {
      removeUpdatingDateType("jobDueDate");
    }
  },
  [mutateJob, refetchDashboard, removeUpdatingDateType]
);
```

That covers the server-error and unchanged-value cases. For the guard-failure cases (where `updateJobs` is never reached), prefer not setting the optimistic skeleton until a dispatch is actually confirmed — e.g. mirror `InlineOrderDeadline`, which only calls `addUpdatingDateType` inside `runOrderMutation` *after* its guards pass, rather than in `handleApply` before them. Alternatively, have `onApply`/the core signal whether a dispatch was initiated so the caller can roll the skeleton back.

---

### 2. `bundle.onApply()` rejection is unhandled — ✅ RESOLVED (`608b325f3`)

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/DeadlineCellContent/partials/InlineJobDueDate/index.tsx]**

**Function/Class:** `InlineJobDueDate` `handleApply`

**Severity:** low

**Problem:** `bundle.onApply()` is fire-and-forget. For the User DD path it transitively awaits `updateJobs`, which rejects when the job PUT fails. The error is surfaced to the user via the per-call `showDefaultErrorToast`, but the rejected promise itself is never awaited or caught, producing an unhandled promise rejection (noisy in the console / Sentry, and coupled to Issue 1's stuck skeleton).

**Impact:** Console/Sentry noise on every failed User-DD edit; no user-facing corruption. Low on its own, but resolving Issue 1 should also make the rejection explicit rather than floating.

**Fix:** Once `updateJobs` clears the skeleton in a `finally` (Issue 1), also swallow/park the rejection at the call site so it doesn't float — e.g. make the handler `async` and `await bundle.onApply().catch(() => {})`, or have `onApply` return the promise so the caller can attach the handler.

---

### 3. New logic-heavy components have no direct unit tests — ✅ RESOLVED (`608b325f3`)

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/DeadlineCellContent/partials/InlineJobDueDate/index.tsx, .../InlineOrderDeadline/index.tsx, apps/creative-portal/components/molecules/DeadlineWarningModal/index.tsx]**

**Function/Class:** `InlineJobDueDate`, `InlineOrderDeadline`, `DeadlineWarningModal`

**Severity:** low

**Problem:** The suite covers `useInlineOrderData`, `useJobDueDatePickers` (bundle contract), `useUpdateJobReturnBase` (handler contract), `DeadlineCellContent`, and `DeadlineCell`. But the two components that hold the most nuanced new logic have **zero direct tests** — `DeadlineCellContent`'s test mocks both inline editors out entirely. Untested behaviour includes: the `currentJobIndex < 0` apply guard, the fetch-on-open chaining, the optimistic skeleton lifecycle (Issue 1), `InlineOrderDeadline`'s Live-status guard, the before-last-job warning branch, the order PUT, the buffer math, and `disableApply`. `DeadlineWarningModal`'s PP-1827 double-click guard is also untested.

**Impact:** The regression that would have caught Issue 1 (a stuck skeleton after a failed apply) is exactly the kind these missing tests would cover. Project rule: every PR must include tests for new code.

**Fix:** Add component tests for `InlineJobDueDate` (apply blocked when `currentJobIndex < 0`; skeleton cleared on mutation failure) and `InlineOrderDeadline` (non-Live order → error toast, no mutation; before-last-job deadline → warning staged; happy path → `mutateOrder` with the new `dueDateTime`). A small test for `DeadlineWarningModal`'s double-click guard is cheap and valuable.

---

### 4. Order-deadline picker buffer is stale on first open — ✅ RESOLVED (`608b325f3`)

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/DeadlineCellContent/partials/InlineOrderDeadline/index.tsx]**

**Function/Class:** `InlineOrderDeadline` (icon `onClick`)

**Severity:** low

**Problem:** The order is fetched lazily on open, but the buffer is computed synchronously in the same click from `lastJobReturnTime`, which is still `""` until the fetch resolves:

```typescript
onClick={() => {
  fetchOrder();                                   // async — resolves later
  setDeadline(toZonedTime(`${dateTime}Z`, timeZone));
  setCurrentBuffer(getBufferMinutes(dateTime, lastJobReturnTime)); // lastJob still null here
}}
```

`currentBuffer` is only recomputed on a subsequent `handleChangeDate`, so on the very first open the picker shows a `0h 0m` buffer until the user nudges the date.

**Impact:** Cosmetic — the buffer preview reads `0h 0m` momentarily on first open. Self-corrects on any date change. No effect on what gets saved.

**Fix:** Recompute the buffer from `lastJob` once the order resolves (e.g. seed `currentBuffer` in an effect keyed on `lastJobReturnTime`/`isOrderFetching`, or derive it with `useMemo` instead of holding it in state).

---

### 5. `PP-1792-PLAN.md` committed to the repo root — ✅ RESOLVED (removed)

**[File: PP-1792-PLAN.md]**

**Severity:** low

**Problem:** A 169-line implementation-planning document is added at the repository root. It's design scaffolding rather than shipped code and doesn't belong in the app tree.

**Impact:** Repo clutter; unrelated to the feature.

**Fix:** Remove it from the PR (or relocate to `docs/` if the team wants to retain it).

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `yarn app:creative-portal test` | ✅ Pass | 1775 creative-portal tests pass (post-fix, incl. +14 new). |
| `npx turbo run typecheck` | ✅ Pass | Clean across all workspaces. |
| `npx turbo run lint` | ✅ Pass | ESLint clean (0 warnings, `--max-warnings 0`). |
| `yarn build:creative-portal` | ✅ Pass | Build completes with 0 type warnings. |

> Validation suite was **executed on `feature/PP-1792-inline-date-editing-v2`** after the review fixes (commit `608b325f3`).

---

## Tests

- ✅ `useUpdateJobReturnBase.test.ts` — injected-handler contract (dispatch via `updateJobs`, warning-modal routing, undefined-order tolerance)
- ✅ `useJobDueDatePickers.test.ts` — bundle contract (seed-before-fetch, re-seed on reopen, assigned bundle, missing-timing)
- ✅ `DeadlineCellContent/hooks.test.tsx` — `useInlineOrderData` (stage/open/close warning, PUT-per-job)
- ✅ `DeadlineCellContent/index.test.tsx` — bulk skeleton, finished-order gating, editor rendering, prop-diff skeleton clear
- ✅ `DeadlineCell/index.test.tsx` — group-header rendering + prop delegation
- ✅ `tableColumns.test.tsx` — column prop plumbing
- ✅ `InlineJobDueDate.test.tsx` — apply-guard (`currentJobIndex < 0`), job/user mode, skeleton (added `608b325f3`)
- ✅ `InlineOrderDeadline.test.tsx` — non-Live guard, before-last-job warning, happy-path PUT, skeleton (added `608b325f3`)
- ✅ `DeadlineWarningModal.test.tsx` — confirm fires callback + close, double-click guard (added `608b325f3`)
- ✅ `hooks.test.tsx` — skeleton cleared on PUT rejection + mutation-start/settle (added `608b325f3`)
- ✅ Full suite verified: 1775 pass

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Issue 1 resolved (skeleton now tied to the mutation lifecycle) |
| Regression risk | ✅ Low — both sidebar extractions independently verified behaviour-preserving |
| Tests | ✅ Hooks + the three new components now covered (+14 tests) |
| Code quality | ✅ Clean, reuse-first, well-documented; `PP-1792-PLAN.md` removed |
| Validation suite | ✅ Run on the v2 branch — test / typecheck / lint / build all pass |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve** — all review findings resolved in `608b325f3`; the architecture is solid and the risky refactors are verified behaviour-preserving.

Resolved:
1. ✅ **Issue 1** — skeleton now driven by the mutation lifecycle (`onJobMutationStart` + `updateJobs` `finally` / `onJobMutationSettled`), so it can't stick on PUT-reject, guard-block, or no-op.
2. ✅ **Issue 2** — `updateJobs` catches/settles the dispatch; the rejection no longer floats.
3. ✅ **Issue 3** — component tests added for `InlineJobDueDate`, `InlineOrderDeadline`, `DeadlineWarningModal`, plus skeleton-clear-on-rejection.
4. ✅ **Issue 4** — order-deadline buffer recomputes when the fetch resolves.
5. ✅ **Req 2** — calendar icon revealed on hover/focus of the date value.
6. ✅ **Issue 5** — `PP-1792-PLAN.md` removed.
7. ✅ **Validation** — `test` / `typecheck` / `lint` / `build` all pass on the v2 branch.

Remaining (outside this PR / for confirmation before staging):
- **Req 2 hover scope** — implemented per-date hover per the ticket wording; confirm against Figma if whole-row hover is intended.
- **Backend deps** — the pending Order Search `currentJobMaxReturnTime` field + legacy-job migration must land for display + legacy-edit correctness (Jira comments 55758 / 57002 / 56729 / 55687).
