# PR Review: PP-1792: Add inline date editing to Order Management dashboard

**PR:** https://github.com/Proofed/B2BWebserver/pull/2289
**Jira:** https://proofed.atlassian.net/browse/PP-1792
**Status:** Code Review
**Branch:** `feature/PP-1792-inline-date-editing-rebased` → `develop`
**Size:** 24 files, +3058 / −218, 7 commits
**Mergeable state (GitHub):** `dirty` (merge conflicts with `develop`)

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1. Due date & deadline fields support inline editing | New `DeadlineCellContent` renders two `InlineDeadlineEditor`s (job due date + order deadline), replacing the raw `DeadlineDisplay`s in `tableColumns.tsx` | ✅ Addressed |
| 2.1 Calendar icon shown on hover of a date field | `InlineIconWrapper` is `opacity: 0` → `1` on `DateRow:hover` / `:focus-visible` | ✅ Addressed |
| 3 / 3.1 / 3.2 Icon colour matches date colour state (red when overdue) | `iconColor = isOverdue ? theme.colors.red : color`; `DeadlineDisplay` independently forces `red` when `remainingTime < 0`, so text and icon stay consistent | ✅ Addressed |
| 4.1 / 4.2 Hovering icon turns it green (matches sidebar) | `&:hover svg, &:focus-visible svg { color: green1 }` | ✅ Addressed |
| 5. Clicking icon opens calendar overlay | `Dropdown` + `DeadlineDatePicker` opened via trigger `onClick={handleOpen}` | ✅ Addressed |
| 6.1 / 6.2 Overlay uses existing job-due-date / order-deadline flows | Reuses `DeadlineDatePicker` + extracted `useUpdateJobReturnBase` (job due date) and a new `applyOrderDeadline` mirroring the sidebar (order deadline) | ⚠️ Partial — reuses the components but the underlying validation was rewritten (see Issue 2) |
| 7. User can submit updated date | `onApply={handleApply}` | ✅ Addressed |
| 8.1 / 8.2 On success, dashboard shows updated date + refreshed styling/icon | `refetchDashboard()` + prop-diff `useEffect` clears the per-type loading skeleton | ✅ Addressed (see Issue 7 for a stuck-skeleton edge case) |
| 9. Applies to both due date & deadline wherever displayed | Both editors rendered in the deadline column | ✅ Addressed |
| 10 / 11. Not available for Completed/Cancelled; no icon, non-interactive | `if (FINISHED_ORDER_STATUSES.includes(orderStatus)) return null` | ✅ Addressed |
| 12 / 12.1 Limited to exposing the existing flow; existing validation, permissions & date-update rules unchanged | The PR threads the PP-1419 Dynamic Return Time model through the shared Job PUT path, changing the **existing sidebar** update behaviour and the displayed Job Due Date | ⚠️ Partial → see Issue 2 |

**Scope beyond the ticket (flagged):**

- `api/jobs/[jobId]/putJob.ts`, `api/jobs/types.ts` — new PP-1419 fields (`returnWindowsMinutes`, `maxReturnWindowsMinutes`, `maxReturnTime`) added to the Job PUT contract, and `proofedUserId` is now forwarded to OMS (Issue 3).
- `services/orders/utils.ts` — `resolveCurrentJobDeadline` changes what the **Job Due Date** column shows for *every* unassigned order (returnTime → maxReturnTime), not only during inline edit.
- `OrderJobs/utils.ts` — new `applyChainTimingsFromEditedJob` / `normalizeJobDurations` chain calculator.
- The `DeadlineWarningModal` extraction folds in PP-1827's double-click guard for all three call sites (a genuine, reasonable refactor).

---

## Architecture Analysis

The core design is sound and follows repo conventions well:

- **Hook/UI split** is respected: `DeadlineCellContent/index.tsx` is UI-only, with `useInlineOrderData` (shared per-row order fetch + mutations) and `useInlineDateEdit` (per-date-type picker state) in `hooks.ts`. The folder carries `index/hooks/styles/types` + two test files.
- **Reuse-first**: `DeadlineDatePicker`, `DeadlineDisplay`, `IconCalendar`, `Dropdown`, and the sidebar's return-time logic are reused rather than duplicated. Extracting `DeadlineWarningModal` into `components/molecules/` and having the sidebar + bulk wrappers delegate to it removes three copies of identical modal JSX — a good consolidation.
- **`useUpdateJobReturn` split** into a handlers-as-params `useUpdateJobReturnBase` core + a thin context-bound wrapper is a clean way to reuse the sidebar's validation without dragging in `useOrderSidebarContext`. The public signature of `useUpdateJobReturn` is unchanged, so `JobCard.tsx` (its only consumer, line 182) is unaffected.

The main architectural tension is **timing and duplication with `develop`**. This branch's base is older than the merged PP-1642/PP-1644 Dynamic Return Time work (the branch has no `useUpdateJobReturn.test.ts`, which those tickets added). Both this PR and PP-1644 independently add `currentJobMaxReturnTime` and a chain-timing calculator over the same jobs, and both rewrite the return-time validation — so the branches now overlap heavily and GitHub reports the PR as unmergeable. Reconciling those two implementations is the single biggest risk in landing this.

A secondary concern is **per-row cost**: the deadline cell went from two cheap `DeadlineDisplay`s to a component that mounts a (disabled) order query observer, two mutation observers, and two `useInlineDateEdit` instances per order row (Issue 6).

---

## Issues Found

### 1. Branch is behind `develop` and unmergeable; overlaps the merged PP-1642/PP-1644 rework

**[File: (whole PR) — base branch state]**

**Function/Class:** merge / rebase

**Severity:** high

**Problem:** GitHub reports `mergeable_state: "dirty"`. The branch predates the Dynamic Return Time epic (PP-1642 merged 2026-06-09 via #2307, PP-1644 built on it), which is now on `develop` and reworked the exact same surfaces this PR touches: `useUpdateJobReturn` / `updateReturnTime`, `updateJobReturnTimes`, `OrderFromSearch.currentJobMaxReturnTime`, and the `JobPut` type. PP-1644's own PR body even predicted this: *"both branches add `currentJobMaxReturnTime?` to `OrderFromSearch` — expect a 'both added' merge conflict … or rebase onto PP-1792 first."*

**Impact:** The PR cannot be merged as-is. Beyond textual conflicts, there is a real risk of two divergent chain-timing implementations landing (this PR's `applyChainTimingsFromEditedJob` vs. PP-1644's bulk-shift helpers) and of the return-time validation being rewritten twice in incompatible ways.

**Fix:** Rebase onto current `develop` and reconcile the return-time logic with what PP-1642/PP-1644 already merged — ideally converging on a single chain-timing helper and a single `currentJobMaxReturnTime` definition. Re-run the full validation suite after the rebase (the isolated-branch state is not what will actually ship).

### 2. Existing sidebar date-update behaviour changes, contrary to req 12.1

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderJobs/hooks/useUpdateJobReturn.ts]**

**Function/Class:** useUpdateJobReturnBase / updateReturnTime

**Severity:** medium

**Problem:** The ticket states the change is "limited to exposing the existing date update flow inline" and "Existing validation, permissions, and date update rules must remain unchanged" (req 12 / 12.1). In practice `updateReturnTime` now first attempts `applyChainTimingsFromEditedJob`; for "new" orders it validates the `maxReturnTime` sequence (not `returnTime`), and the sidebar's context `updateJobs` (`contexts/orderSidebar/provider.tsx:328`) forwards the *whole* recomputed job — so a sidebar job-due-date change now sends recomputed `maxReturnTime` + cleared `returnTime` for unassigned jobs, where previously it shifted `returnTime` only. Separately, `resolveCurrentJobDeadline` changes the displayed Job Due Date (returnTime → maxReturnTime) for every unassigned order on the dashboard, independent of any inline edit.

**Impact:** The existing sidebar deadline-change flow and the at-rest dashboard Job Due Date column behave differently after this PR. This may be intentional/necessary given PP-1419, but it exceeds the ticket's stated "unchanged" guarantee and should be explicitly QA'd against the sidebar and dashboard for both new-model and legacy orders — not assumed safe.

**Fix:** Confirm with the ticket owner that the behavioural change to the existing sidebar flow and the displayed Job Due Date is intended, and add it to the manual test plan (sidebar job-due-date change; dashboard Job Due Date value for Offered/In Queue/On Hold orders). Ensure the manual "existing sidebar date editing still works unchanged" checklist item is actually exercised on legacy (pre-PP-1419) orders.

### 3. `putJob.ts` now forwards `proofedUserId` on every job PUT

**[File: apps/creative-portal/api/jobs/[jobId]/putJob.ts]**

**Function/Class:** putJob

**Severity:** medium

**Problem:** The previous handler never sent `proofedUserId` to OMS. It now does whenever the request body carries one:

```typescript
if (body.proofedUserId) {
  jobPutData.proofedUserId = body.proofedUserId;
}
```

Because the sidebar's `updateJobs` posts the full `Job` object, existing non-date PUT flows (e.g. editing pay/comment via the sidebar) will now also echo `proofedUserId` on the PUT — a field they did not send before.

**Impact:** If OMS interprets `proofedUserId` on a Job PUT as an (re)assignment operation rather than an idempotent echo, unrelated sidebar edits could have assignment side effects. If OMS treats it as idempotent (same assignee), it's harmless. This needs confirmation against OMS semantics before merge.

**Fix:** Verify with the OMS contract (PP-1419 §24) that echoing the current `proofedUserId` on PUT is idempotent for already-assigned jobs. If it is not, scope the `proofedUserId` forwarding to the code paths that actually intend (re)assignment rather than adding it unconditionally in the shared route handler.

### 4. Unhandled crash when the current job is not in the fetched job list (`currentJobIndex === -1`)

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/DeadlineCellContent/hooks.ts]**

**Function/Class:** useInlineDateEdit.handleApplyJobDueDate → useUpdateJobReturnBase.updateReturnTime

**Severity:** medium

**Problem:** `currentJobIndex` is `jobs.findIndex((job) => job.id === orderData?.currentJobId)`, which is `-1` if `currentJobId` is null/0 or absent from `orderJobs`. `handleApplyJobDueDate` only guards `if (!deadlineDate || !orderData?.dueDateTime) return;`. It then calls `updateReturnTime`, whose first line dereferences `jobs[selectedJobIndex].status`:

```typescript
if (jobs[selectedJobIndex].status === "Approved") { ... }  // jobs[-1] is undefined → throws
```

The picker still renders (it falls back to `initialDate` from the `currentJobDeadline` prop), so a user can open it and click Apply even when the fetched order has no matching current job.

**Impact:** A `TypeError: Cannot read properties of undefined (reading 'status')` inside the render tree of the affected row on backend data inconsistency (currentJobId missing from `orderJobs`). No test covers `currentJobIndex === -1`.

**Fix:** Guard the apply path, e.g. `if (!deadlineDate || !orderData?.dueDateTime || currentJobIndex < 0) return;` (and/or bail in `updateReturnTime` when `selectedJobIndex < 0`). Add a unit test for the `-1` case.

### 5. `addJobWithTasks` mixture will be rejected at runtime (documented, not fixed)

**[File: apps/creative-portal/api/mixtures/jobs/addJobWithTasks/addJobWithTasks.ts]**

**Function/Class:** addJobWithTasks

**Severity:** medium

**Problem:** The PR adds a comment acknowledging the problem rather than resolving it:

```
// TODO(PP-1419): OMS requires returnWindowsMinutes, maxReturnWindowsMinutes and
// maxReturnTime on every Job PUT. This mixture's yup schema does not yet validate
// those fields on `upcomingJob`, so this call will be rejected at runtime.
```

This mixture calls the `updateJob` service directly (bypassing `putJob.ts`) with a payload lacking the now-required fields.

**Impact:** Any flow that adds a job with tasks and updates the following (`upcomingJob`) job will fail at OMS. This is framed as a pre-existing/parallel PP-1419 concern (not introduced by this PR's route change), but merging with a known "will be rejected at runtime" path is risky if that path is live.

**Fix:** Confirm whether the add-job-with-tasks path is currently exercised. If it is, fix it here (forward the PP-1419 fields + extend the yup schema) rather than deferring; if it genuinely cannot fire yet, ensure the PP-1419 follow-up ticket referenced in the TODO exists and is linked.

### 6. Every dashboard row now mounts an order query + mutation observers

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/DeadlineCellContent/hooks.ts]**

**Function/Class:** useInlineOrderData (called once per row via DeadlineCellContent)

**Severity:** low

**Problem:** The deadline cell replaced two cheap `DeadlineDisplay`s with a component that, for every rendered order row, instantiates `useOrderByIdQuery` (disabled), `useOrderPutMutation`, `useJobMutation`, `useQueryClient`, `useZonedTime`, plus two `useInlineDateEdit` instances (each with its own `useUpdateJobReturnBase` + `useLatestRef` effects). The hooks run even for rows that immediately `return null` (finished orders) because hooks must precede the early return.

**Impact:** For a large dashboard (tens–hundreds of rows) this adds a proportional number of disabled query observers and mutation observers. The queries are disabled (no network), so this is overhead rather than a correctness bug, but it is materially heavier than the previous cell and worth a quick profiler check on a big result set.

**Fix:** Optional. If profiling shows cost, consider lazily creating the query/mutations only when a picker first opens, or memoising `InlineDeadlineEditor`. Not a blocker.

### 7. Loading skeleton can stick if the post-mutation dashboard value is unchanged

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/DeadlineCellContent/index.tsx]**

**Function/Class:** DeadlineCellContent (prop-diff useEffect) + useInlineOrderData.updatingDateTypes

**Severity:** low

**Problem:** `addUpdatingDateType(...)` shows the per-type skeleton, and it is cleared **only** by the `useEffect` that fires when the `deadline` / `currentJobDeadline` prop reference changes after `refetchDashboard()`. If the mutation succeeds but the recomputed dashboard value is identical to the prior one (e.g. same effective date, or React Query structural-sharing returns the same string), the prop never changes and the skeleton is not cleared.

**Impact:** In a narrow case the cell can show its loading skeleton indefinitely until an unrelated refresh. `refetchDashboard` mitigates via a one-shot reference-equality retry, but does not guarantee the prop changes.

**Fix:** Also clear the updating state on mutation settle (success/error) as a backstop, rather than relying solely on the downstream prop diff.

### 8. `putJob.ts` performs no runtime validation of the now-required fields

**[File: apps/creative-portal/api/jobs/[jobId]/putJob.ts]**

**Function/Class:** putJob

**Severity:** low

**Problem:** `const body = await parseReqBody<JobPut>(req)` is a type assertion only. `JobPut` now types `returnWindowsMinutes` / `maxReturnWindowsMinutes` / `maxReturnTime` as required, but nothing enforces them at runtime; a caller that omits them forwards `undefined` (dropped from the JSON body).

**Impact:** A malformed/legacy caller gets an opaque OMS 4xx instead of a clear BFF 400, making failures harder to diagnose. Consistent with the route's pre-existing lack of a yup schema, so not a regression — just a robustness gap that the new required-field contract makes more likely to bite.

**Fix:** Optional — add a yup schema (or explicit guards) validating the PP-1419 fields and returning `handleBadRequest` when absent.

### 9. Order-deadline warning path skips the "must be Live" guard

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/DeadlineCellContent/hooks.ts]**

**Function/Class:** applyOrderDeadline

**Severity:** low

**Problem:** When the new order deadline falls before the last job's return time, the code shows the warning modal whose confirm callback calls `mutateOrder(...)` directly. The `orderData.status !== "Live"` check only exists on the non-warning branch below it, so a non-Live (e.g. Paused) order can be mutated through the warning-confirm path.

**Impact:** Inconsistent enforcement of the Live-status rule between the two order-deadline branches. Low risk because finished orders are already excluded at the cell level, but Paused/other non-Live states are not.

**Fix:** Perform the `status === "Live"` check once up front (before branching into the warning vs. direct path) so both paths enforce it.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ Skipped | Skipped — user opted out |
| `npx turbo run typecheck` | ⏭️ Skipped | Skipped — user opted out |
| `npx turbo run lint` | ⏭️ Skipped | Skipped — user opted out |
| `npx turbo run build` | ⏭️ Skipped | Skipped — user opted out |

Static review only. The PR self-reports typecheck 0 errors, 921 passing tests, and clean lint on the isolated branch — **not verified in this review**, and not representative of the post-rebase merge state (Issue 1). A concern worth a targeted check post-rebase: `useUpdateJobReturn.test.ts` exists on `develop` (added by PP-1642) but **not** on this branch, so after rebasing, those existing tests will run against the rewritten `useUpdateJobReturnBase` for the first time.

---

## Tests

- ✅ New unit tests added: 25 for `DeadlineCellContent` (`DeadlineCellContent.test.tsx` + `hooks.test.ts`), 8 for `applyChainTimingsFromEditedJob` (`OrderJobs/utils.test.ts`), 7 for `resolveCurrentJobDeadline` (`services/orders/utils.test.ts`), plus a `tableColumns.test.tsx` mock addition. Meets the "every PR must include tests" requirement.
- ✅ Good coverage of the tricky paths: the async-fetch race guard (`userInteractedRef`), Target-mode "before next due date" error, order-status guard, chain cascade across assigned/unassigned jobs, and the null-return legacy fallback.
- ⚠️ The component/hook tests mock nearly every dependency (services, query client, date utils, child components), so they verify wiring more than real integration.
- ❌ No test for `currentJobIndex === -1` (Issue 4).
- ❌ Not verified running: the four mandatory checks were skipped at the user's request, and the existing `useUpdateJobReturn.test.ts` interaction post-rebase (see Validation note) is untested here.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ One unguarded crash path (Issue 4) + an OMS-contract question (Issue 3); core inline flow otherwise correct |
| Regression risk | ⚠️ Medium — changes to the existing sidebar update flow (Issue 2), new `proofedUserId` forwarding (Issue 3), and heavy overlap with merged PP-1642/PP-1644 (Issue 1) |
| Tests | ✅ New tests added with good path coverage; ⚠️ heavily mocked, one gap |
| Code quality | ✅ Clean hook/UI split, good reuse, sensible `DeadlineWarningModal` consolidation |
| Validation suite | ⏭️ Skipped — user opted out (must be run before merge) |
| Mergeable state | ❌ Dirty — merge conflicts with `develop` (Issue 1) |

---

## Recommendation

**Request changes.**

1. **Blocker — resolve Issue 1 first.** Rebase onto current `develop` and reconcile the return-time logic with the already-merged PP-1642/PP-1644 Dynamic Return Time work (single chain-timing helper, single `currentJobMaxReturnTime` definition). This must precede any final sign-off because it changes what actually ships.
2. **Run the mandatory validation suite** (`test` / `typecheck` / `lint` / `build`) **after the rebase** — this review skipped it at the user's request, and per CLAUDE.md a PR cannot be approved on unverified checks. Pay special attention to `useUpdateJobReturn.test.ts` (present on `develop`, absent here) now exercising the rewritten `useUpdateJobReturnBase`.
3. **Confirm OMS semantics for `proofedUserId` on PUT** (Issue 3) and for the changed sidebar/dashboard behaviour (Issue 2) with the ticket owner; add both to the manual test plan for legacy and new-model orders.
4. **Fix the `currentJobIndex === -1` guard** (Issue 4) and add a test.
5. **Decide on `addJobWithTasks`** (Issue 5) — fix it now if the path is live, otherwise ensure the PP-1419 follow-up is tracked and linked.
6. Address the low-severity items (7 & 9) as small follow-ups; treat 6 & 8 as optional.
