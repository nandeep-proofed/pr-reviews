# PR Review: feature/PP-1792 — Inline due date & deadline editing on Order Management dashboard

**PR:** https://github.com/Proofed/B2BWebserver/pull/2380
**Jira:** https://proofed.atlassian.net/browse/PP-1792
**Status:** Code Review (Story, High priority)
**Author:** nandeep-proofed · **Branch:** `feature/PP-1792-inline-date-editing-v2` → `develop`
**Size:** 20 files, +2702 / −1067, 3 commits

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1. Due date & deadline fields support inline editing | `InlineJobDueDate` (current-job line) + `InlineOrderDeadline` (order deadline line) rendered by `DeadlineCellContent` | ✅ Addressed |
| 2. Hovering a date shows a calendar icon | `IconTrigger` renders an `IconButton` calendar — but **persistently**, not revealed on hover (no `opacity:0`/`:hover` reveal in `styles.ts` or the parent table) | ⚠️ Partial — verify vs Figma |
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

### 1. User-DD inline edit can leave a permanent loading skeleton on the row

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

### 2. `bundle.onApply()` rejection is unhandled

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/DeadlineCellContent/partials/InlineJobDueDate/index.tsx]**

**Function/Class:** `InlineJobDueDate` `handleApply`

**Severity:** low

**Problem:** `bundle.onApply()` is fire-and-forget. For the User DD path it transitively awaits `updateJobs`, which rejects when the job PUT fails. The error is surfaced to the user via the per-call `showDefaultErrorToast`, but the rejected promise itself is never awaited or caught, producing an unhandled promise rejection (noisy in the console / Sentry, and coupled to Issue 1's stuck skeleton).

**Impact:** Console/Sentry noise on every failed User-DD edit; no user-facing corruption. Low on its own, but resolving Issue 1 should also make the rejection explicit rather than floating.

**Fix:** Once `updateJobs` clears the skeleton in a `finally` (Issue 1), also swallow/park the rejection at the call site so it doesn't float — e.g. make the handler `async` and `await bundle.onApply().catch(() => {})`, or have `onApply` return the promise so the caller can attach the handler.

---

### 3. New logic-heavy components have no direct unit tests

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/DeadlineCellContent/partials/InlineJobDueDate/index.tsx, .../InlineOrderDeadline/index.tsx, apps/creative-portal/components/molecules/DeadlineWarningModal/index.tsx]**

**Function/Class:** `InlineJobDueDate`, `InlineOrderDeadline`, `DeadlineWarningModal`

**Severity:** low

**Problem:** The suite covers `useInlineOrderData`, `useJobDueDatePickers` (bundle contract), `useUpdateJobReturnBase` (handler contract), `DeadlineCellContent`, and `DeadlineCell`. But the two components that hold the most nuanced new logic have **zero direct tests** — `DeadlineCellContent`'s test mocks both inline editors out entirely. Untested behaviour includes: the `currentJobIndex < 0` apply guard, the fetch-on-open chaining, the optimistic skeleton lifecycle (Issue 1), `InlineOrderDeadline`'s Live-status guard, the before-last-job warning branch, the order PUT, the buffer math, and `disableApply`. `DeadlineWarningModal`'s PP-1827 double-click guard is also untested.

**Impact:** The regression that would have caught Issue 1 (a stuck skeleton after a failed apply) is exactly the kind these missing tests would cover. Project rule: every PR must include tests for new code.

**Fix:** Add component tests for `InlineJobDueDate` (apply blocked when `currentJobIndex < 0`; skeleton cleared on mutation failure) and `InlineOrderDeadline` (non-Live order → error toast, no mutation; before-last-job deadline → warning staged; happy path → `mutateOrder` with the new `dueDateTime`). A small test for `DeadlineWarningModal`'s double-click guard is cheap and valuable.

---

### 4. Order-deadline picker buffer is stale on first open

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

### 5. `PP-1792-PLAN.md` committed to the repo root

**[File: PP-1792-PLAN.md]**

**Severity:** low

**Problem:** A 169-line implementation-planning document is added at the repository root. It's design scaffolding rather than shipped code and doesn't belong in the app tree.

**Impact:** Repo clutter; unrelated to the feature.

**Fix:** Remove it from the PR (or relocate to `docs/` if the team wants to retain it).

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ Skipped — user opted out | Not run. PR claims 1759 creative-portal tests pass. |
| `npx turbo run typecheck` | ⏭️ Skipped — user opted out | Not run. |
| `npx turbo run lint` | ⏭️ Skipped — user opted out | Not run. PR claims ESLint clean. |
| `npx turbo run build` | ⏭️ Skipped — user opted out | Not run. PR claims `turbo run build` — 0 warnings. |

> Validation suite was **not executed** for this review (reviewer opted out; the PR is on the v2 branch and validating it would require a fresh detached worktree + `yarn install`). Re-run all four against `feature/PP-1792-inline-date-editing-v2` before merging.

---

## Tests

- ✅ `useUpdateJobReturnBase.test.ts` — injected-handler contract (dispatch via `updateJobs`, warning-modal routing, undefined-order tolerance)
- ✅ `useJobDueDatePickers.test.ts` — bundle contract (seed-before-fetch, re-seed on reopen, assigned bundle, missing-timing)
- ✅ `DeadlineCellContent/hooks.test.tsx` — `useInlineOrderData` (stage/open/close warning, PUT-per-job)
- ✅ `DeadlineCellContent/index.test.tsx` — bulk skeleton, finished-order gating, editor rendering, prop-diff skeleton clear
- ✅ `DeadlineCell/index.test.tsx` — group-header rendering + prop delegation
- ✅ `tableColumns.test.tsx` — column prop plumbing
- ❌ `InlineJobDueDate` — no direct test (Issue 3)
- ❌ `InlineOrderDeadline` — no direct test (Issue 3)
- ❌ `DeadlineWarningModal` — no direct test (Issue 3)
- ⏭️ Existing-suite pass/fail not verified (validation skipped)

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ One reachable defect (Issue 1 — stuck skeleton on failed/no-op User-DD edit) |
| Regression risk | ✅ Low — both sidebar extractions independently verified behaviour-preserving |
| Tests | ⚠️ Hooks well covered; the two logic-heavy new components untested |
| Code quality | ✅ Clean, reuse-first, well-documented; minor housekeeping (Issue 5) |
| Validation suite | ⏭️ Skipped — user opted out (re-run before merge) |
| Mergeable state | ✅ Clean (GitHub `mergeable_state: clean`); validation not independently verified |

---

## Recommendation

**Approve with changes** — the architecture is solid and the risky refactors are safe, but Issue 1 is a reachable UX defect and validation was not run.

1. **Fix Issue 1** — give the User-DD dispatch a settle guarantee (clear the `jobDueDate` skeleton in a `finally` on `updateJobs`, and stop setting the optimistic skeleton before the guard-gated dispatch). This is the one blocking item; the stuck skeleton is reachable today via the documented legacy-job PUT rejection.
2. **Address Issue 2** — handle/await the `onApply` rejection so failed edits don't float an unhandled promise.
3. **Add the missing component tests** (Issue 3) — at minimum `InlineJobDueDate`'s apply-guard + skeleton-clear-on-failure and `InlineOrderDeadline`'s Live-status/warning/happy paths; this satisfies the "tests for new code" rule and locks in the Issue 1 fix.
4. **Verify Req 2 against Figma** — the calendar icon currently renders persistently; confirm whether the design calls for hover-to-reveal (PR checklist "UI elements match designs" is still unchecked).
5. **Run the mandatory validation suite** (`test` / `typecheck` / `lint` / `build`) on the PR branch — it was skipped for this review and is a hard gate per CLAUDE.md.
6. **Housekeeping** — remove `PP-1792-PLAN.md` (Issue 5), and confirm the pending backend `currentJobMaxReturnTime` field + legacy-job migration land before this is validated on staging (display + legacy-edit correctness depend on them).
