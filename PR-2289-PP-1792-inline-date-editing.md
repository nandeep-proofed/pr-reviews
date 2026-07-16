# PR Review: PP-1792: Add inline date editing to Order Management dashboard

**PR:** https://github.com/Proofed/B2BWebserver/pull/2289
**Jira:** https://proofed.atlassian.net/browse/PP-1792
**Status:** Code Review
**Branch:** `feature/PP-1792-inline-date-editing-rebased` → `develop`
**Size:** 22 files, +2893 / −195, 8 commits
**Mergeable state (GitHub):** `clean` (was `dirty` at first review — reconciled with develop, see Issue 1)

> **Update (2026-07-16):** Re-checked against the current branch head (`d5e529f9f`). **Issues 1, 2, 3 and 5 are now resolved / not-valid** — see the ✅ notes on each. Issues 4, 6, 7, 8, 9 still stand. Locally re-verified on the current branch: affected unit tests (126) pass, `typecheck` 0 errors, `lint` 0 errors on all changed files.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1. Due date & deadline fields support inline editing | New `DeadlineCellContent` renders two `InlineDeadlineEditor`s (job due date + order deadline), replacing the raw `DeadlineDisplay`s in `tableColumns.tsx` | ✅ Addressed |
| 2.1 Calendar icon shown on hover of a date field | `InlineIconWrapper` is `opacity: 0` → `1` on `DateRow:hover` / `:focus-visible` | ✅ Addressed |
| 3 / 3.1 / 3.2 Icon colour matches date colour state (red when overdue) | `iconColor = isOverdue ? theme.colors.red : color`; `DeadlineDisplay` independently forces `red` when `remainingTime < 0`, so text and icon stay consistent | ✅ Addressed |
| 4.1 / 4.2 Hovering icon turns it green (matches sidebar) | `&:hover svg, &:focus-visible svg { color: green1 }` | ✅ Addressed |
| 5. Clicking icon opens calendar overlay | `Dropdown` + `DeadlineDatePicker` opened via trigger `onClick={handleOpen}` | ✅ Addressed |
| 6.1 / 6.2 Overlay uses existing job-due-date / order-deadline flows | Reuses `DeadlineDatePicker` + `useUpdateJobReturnBase` (a behaviour-preserving refactor of the sidebar hook) and a new `applyOrderDeadline` mirroring the sidebar | ✅ Addressed (sidebar validation preserved — see Issue 2) |
| 7. User can submit updated date | `onApply={handleApply}` | ✅ Addressed |
| 8.1 / 8.2 On success, dashboard shows updated date + refreshed styling/icon | `refetchDashboard()` + prop-diff `useEffect` clears the per-type loading skeleton | ✅ Addressed (see Issue 7 for a stuck-skeleton edge case) |
| 9. Applies to both due date & deadline wherever displayed | Both editors rendered in the deadline column | ✅ Addressed |
| 10 / 11. Not available for Completed/Cancelled; no icon, non-interactive | `if (FINISHED_ORDER_STATUSES.includes(orderStatus)) return null` | ✅ Addressed |
| 12 / 12.1 Limited to exposing the existing flow; existing validation, permissions & date-update rules unchanged | Existing sidebar flow is preserved (behaviour-preserving refactor); the displayed-Job-Due-Date change is develop's PP-1644 baseline, not this PR | ✅ Addressed — see Issue 2 re-assessment |

**Scope beyond the ticket (flagged):**

- `api/jobs/[jobId]/putJob.ts`, `api/jobs/types.ts` — new PP-1419 fields (`returnWindowsMinutes`, `maxReturnWindowsMinutes`, `maxReturnTime`) added to the Job PUT contract. (The unconditional `proofedUserId` forwarding flagged in Issue 3 has since been removed — see Issue 3.)
- `services/orders/utils.ts` — the returnTime→maxReturnTime display of the Job Due Date is **develop's PP-1644 Req 4.1 behaviour**; PP-1792's diff here is a comment only (see Issue 2).
- `OrderJobs/utils.ts` — new `applyChainTimingsFromEditedJob` / `normalizeJobDurations` chain calculator (used by the new inline path).
- The `DeadlineWarningModal` extraction folds in PP-1827's double-click guard for all three call sites (a genuine, reasonable refactor).

---

## Architecture Analysis

The core design is sound and follows repo conventions well:

- **Hook/UI split** is respected: `DeadlineCellContent/index.tsx` is UI-only, with `useInlineOrderData` (shared per-row order fetch + mutations) and `useInlineDateEdit` (per-date-type picker state) in `hooks.ts`. The folder carries `index/hooks/styles/types` + two test files.
- **Reuse-first**: `DeadlineDatePicker`, `DeadlineDisplay`, `IconCalendar`, `Dropdown`, and the sidebar's return-time logic are reused rather than duplicated. Extracting `DeadlineWarningModal` into `components/molecules/` and having the sidebar + bulk wrappers delegate to it removes three copies of identical modal JSX — a good consolidation.
- **`useUpdateJobReturn` split** into a handlers-as-params `useUpdateJobReturnBase` core + a thin context-bound wrapper is a clean way to reuse the sidebar's validation without dragging in `useOrderSidebarContext`. The public signature of `useUpdateJobReturn` is unchanged, so `JobCard.tsx` (its only consumer, line 182) is unaffected.

~~The main architectural tension is **timing and duplication with `develop`**.~~ **(Resolved — Issue 1.)** The branch has since merged `develop` and reconciled the return-time logic with the merged PP-1642/PP-1644 Dynamic Return Time work; GitHub now reports the PR as `clean`.

A secondary concern is **per-row cost**: the deadline cell went from two cheap `DeadlineDisplay`s to a component that mounts a (disabled) order query observer, two mutation observers, and two `useInlineDateEdit` instances per order row (Issue 6).

---

## Issues Found

### 1. Branch is behind `develop` and unmergeable; overlaps the merged PP-1642/PP-1644 rework — ✅ RESOLVED

**[File: (whole PR) — base branch state]**

**Function/Class:** merge / rebase

**Severity:** high

**Problem:** GitHub reported `mergeable_state: "dirty"`. The branch predated the Dynamic Return Time epic (PP-1642 merged 2026-06-09 via #2307, PP-1644 built on it), which reworked the exact same surfaces this PR touches: `useUpdateJobReturn` / `updateReturnTime`, `updateJobReturnTimes`, `OrderFromSearch.currentJobMaxReturnTime`, and the `JobPut` type.

**Resolution (verified 2026-07-16):** The branch merged current `develop` (`d5e529f9f`) and reconciled the return-time logic across four follow-up commits — `dbe055500` "Merge conflicts resolve", `2b1c26cc4` "branch Job Due Date on current job assignment status", `b2d007204` "prefer currentJobMaxReturnTime for dashboard Job Due Date", `9408aa6f7` "display + edit currentJobMaxReturnTime when OMS supplies it". GitHub now reports **`mergeable_state: "clean"`**, and `useUpdateJobReturn.test.ts` (present on `develop` via PP-1642, absent at first review) is now on the branch and passing.

### 2. Existing sidebar date-update behaviour changes, contrary to req 12.1 — ✅ RESOLVED / not valid

**[File: apps/creative-portal/.../OrderJobs/hooks/useUpdateJobReturn.ts · apps/creative-portal/services/orders/utils.ts]**

**Function/Class:** useUpdateJobReturnBase / updateReturnTime · resolveCurrentJobDeadline

**Severity:** medium

**Problem (as originally raised):** The ticket states the change is "limited to exposing the existing date update flow inline" and "existing validation, permissions, and date update rules must remain unchanged" (req 12 / 12.1). The review flagged that `updateReturnTime` now attempts `applyChainTimingsFromEditedJob` / validates the `maxReturnTime` sequence, and that `resolveCurrentJobDeadline` changes the displayed Job Due Date (returnTime → maxReturnTime) for every unassigned order.

**Re-assessment (verified 2026-07-16 against the current reconciled branch — not valid):** Both cited behaviour changes are develop's already-merged PP-1642/PP-1644 baseline, not introduced by this PR:

- **`services/orders/utils.ts`** — PP-1792's diff vs develop is a **comment-only change**. The displayed-Job-Due-Date fallback `currentJobReturnTime || currentJobMaxReturnTime` is develop's behaviour, labelled **PP-1644 Req 4.1** in the code. Not this PR.
- **`useUpdateJobReturn.ts`** — the change is a **behaviour-preserving refactor**: `useUpdateJobReturn` is split into a handlers-as-params `useUpdateJobReturnBase` core + a thin wrapper whose **public signature is unchanged** and which passes the **same** sidebar context handlers (`setDeadlineModalProps`, `showDeadlineWarningModal`, `updateJobs`). The sidebar's `updateReturnTime` still imports/uses only `updateJobReturnTimes` (legacy shift) and does **not** call `applyChainTimingsFromEditedJob` — that chain calculator is used **only in the new inline dashboard path** (`DeadlineCellContent`), never the sidebar flow. The one other edit is `order.dueDateTime` → `order?.dueDateTime` null-safety, which is identical when `order` is defined (always true in the sidebar).

The original review was written against the pre-reconcile (`dirty`) branch, where PP-1792 had independent chain-timing wiring. After merging develop + the reconcile commits the branch converged on develop's model, so req 12.1's "existing sidebar flow unchanged" is **upheld**.

**Residual (not a defect):** because a shared hook was refactored, a quick QA smoke-test of the sidebar date-edit is worthwhile refactor hygiene.

### 3. `putJob.ts` now forwards `proofedUserId` on every job PUT — ✅ RESOLVED

**[File: apps/creative-portal/api/jobs/[jobId]/putJob.ts]**

**Function/Class:** putJob

**Severity:** medium

**Problem:** The handler had begun sending `proofedUserId` to OMS whenever the request body carried one. Because the sidebar's `updateJobs` posts the full `Job` object, existing non-date PUT flows would also echo `proofedUserId` — risking an unintended (re)assignment side effect if OMS treats it as such.

**Resolution (verified 2026-07-16):** `putJob.ts` no longer references `proofedUserId` — `jobPutData` is now limited to the intended fields (`maxReturnTime`, `returnWindowsMinutes`, etc.). The unconditional forwarding was removed from the shared route handler.

### 4. Unhandled crash when the current job is not in the fetched job list (`currentJobIndex === -1`)

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/DeadlineCellContent/hooks.ts]**

**Function/Class:** useInlineDateEdit.handleApplyJobDueDate → useUpdateJobReturnBase.updateReturnTime

**Severity:** medium

**Problem:** `currentJobIndex` is `jobs.findIndex((job) => job.id === orderData?.currentJobId)`, which is `-1` if `currentJobId` is null/0 or absent from `orderJobs`. `handleApplyJobDueDate` only guards `if (!deadlineDate || !orderData?.dueDateTime) return;`. It then calls `updateReturnTime`, whose first lines are `const currentJob = jobs[selectedJobIndex];` then `if (currentJob.status === "Approved")` — `jobs[-1]` is `undefined` → throws.

**Impact:** A `TypeError: Cannot read properties of undefined (reading 'status')` inside the render tree of the affected row on backend data inconsistency (currentJobId missing from `orderJobs`). No test covers `currentJobIndex === -1`.

**Status (2026-07-16):** ❌ **Still open** — no `currentJobIndex < 0` / `selectedJobIndex < 0` guard has been added.

**Fix:** Guard the apply path, e.g. `if (!deadlineDate || !orderData?.dueDateTime || currentJobIndex < 0) return;` (and/or bail in `updateReturnTime` when `selectedJobIndex < 0`). Add a unit test for the `-1` case.

### 5. `addJobWithTasks` mixture will be rejected at runtime (documented, not fixed) — ✅ RESOLVED

**[File: apps/creative-portal/api/mixtures/jobs/addJobWithTasks/addJobWithTasks.ts]**

**Function/Class:** addJobWithTasks

**Severity:** medium

**Problem:** The PR had added a `TODO(PP-1419)` comment acknowledging that the mixture calls `updateJob` directly with a payload lacking the now-required `returnWindowsMinutes` / `maxReturnWindowsMinutes` / `maxReturnTime`, so the `upcomingJob` PUT would be rejected at OMS.

**Resolution (verified 2026-07-16):** The `TODO(PP-1419)` is gone and the mixture now forwards the PP-1419 timing fields for the upcoming job — it guards on `insertedJobTiming?.maxReturnWindowsMinutes`/`returnWindowsMinutes`, derives the window, and sends `returnWindowsMinutes` + `maxReturnTime` on the PUT.

### 6. Every dashboard row now mounts an order query + mutation observers

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/DeadlineCellContent/hooks.ts]**

**Function/Class:** useInlineOrderData (called once per row via DeadlineCellContent)

**Severity:** low

**Problem:** For every rendered order row the cell instantiates `useOrderByIdQuery` (disabled), `useOrderPutMutation`, `useJobMutation`, `useQueryClient`, `useZonedTime`, plus two `useInlineDateEdit` instances. The queries are disabled (no network), so this is overhead rather than a correctness bug, but it is materially heavier than the previous cell.

**Fix:** Optional. If profiling shows cost, lazily create the query/mutations only when a picker first opens, or memoise `InlineDeadlineEditor`. Not a blocker.

### 7. Loading skeleton can stick if the post-mutation dashboard value is unchanged

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/DeadlineCellContent/index.tsx]**

**Function/Class:** DeadlineCellContent (prop-diff useEffect) + useInlineOrderData.updatingDateTypes

**Severity:** low

**Problem:** `addUpdatingDateType(...)` shows the per-type skeleton, cleared **only** by the `useEffect` that fires when the `deadline` / `currentJobDeadline` prop reference changes after `refetchDashboard()`. If the mutation succeeds but the recomputed value is identical, the prop never changes and the skeleton is not cleared. `refetchDashboard` mitigates via a one-shot reference-equality retry but does not guarantee the prop changes.

**Fix:** Also clear the updating state on mutation settle (success/error) as a backstop, rather than relying solely on the downstream prop diff.

### 8. `putJob.ts` performs no runtime validation of the now-required fields

**[File: apps/creative-portal/api/jobs/[jobId]/putJob.ts]**

**Function/Class:** putJob

**Severity:** low

**Problem:** `const body = await parseReqBody<JobPut>(req)` is a type assertion only. `JobPut` types the PP-1419 fields as required, but nothing enforces them at runtime; a caller that omits them forwards `undefined`. A malformed/legacy caller gets an opaque OMS 4xx instead of a clear BFF 400.

**Fix:** Optional — add a yup schema (or explicit guards) validating the PP-1419 fields and returning `handleBadRequest` when absent.

### 9. Order-deadline warning path skips the "must be Live" guard

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/DeadlineCellContent/hooks.ts]**

**Function/Class:** applyOrderDeadline

**Severity:** low

**Problem:** When the new order deadline falls before the last job's return time, the code shows the warning modal whose confirm callback calls `mutateOrder(...)` directly. The `orderData.status !== "Live"` check only exists on the non-warning branch below it, so a non-Live (e.g. Paused) order can be mutated through the warning-confirm path.

**Status (2026-07-16):** ❌ **Still open** — the `status !== "Live"` check is still positioned after the warning branch.

**Fix:** Perform the `status === "Live"` check once up front (before branching into the warning vs. direct path) so both paths enforce it.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| Affected unit tests (creative-portal) | ✅ | **126/126 pass** on the current branch — re-run 2026-07-16 |
| `turbo run typecheck` (creative-portal) | ✅ | 0 errors (current branch) |
| `lint` (all 22 changed files) | ✅ | 0 errors (current branch) |
| `turbo run build` | ⏭️ Not run | — |

Post-rebase note (now satisfied): `useUpdateJobReturn.test.ts` — present on `develop` (PP-1642), absent at first review — is now on the branch and passing against the reconciled `useUpdateJobReturnBase`.

---

## Tests

- ✅ New unit tests added: `DeadlineCellContent` (`DeadlineCellContent.test.tsx` + `hooks.test.ts`), `applyChainTimingsFromEditedJob` (`OrderJobs/utils.test.ts`), `resolveCurrentJobDeadline` (`services/orders/utils.test.ts`), plus a `tableColumns.test.tsx` mock addition.
- ✅ Good coverage of the tricky paths: the async-fetch race guard (`userInteractedRef`), Target-mode "before next due date" error, order-status guard, chain cascade across assigned/unassigned jobs, and the null-return legacy fallback.
- ⚠️ The component/hook tests mock nearly every dependency, so they verify wiring more than real integration.
- ❌ No test for `currentJobIndex === -1` (Issue 4, still open).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ One unguarded crash path remains (Issue 4); core inline flow otherwise correct. Issues 2 & 3 resolved |
| Regression risk | ✅ Low → Issue 1 (develop overlap), Issue 2 (sidebar refactor is behaviour-preserving) and Issue 3 (proofedUserId) all resolved |
| Tests | ✅ New tests added with good path coverage; ⚠️ heavily mocked; one gap (Issue 4) |
| Code quality | ✅ Clean hook/UI split, good reuse, sensible `DeadlineWarningModal` consolidation |
| Validation suite | ✅ affected tests / typecheck / lint green on current branch (build not run) |
| Mergeable state | ✅ `clean` (Issue 1 resolved) |

---

## Recommendation

**Request changes** — down to a very short list; the blocker and most mediums are cleared.

1. ~~**Blocker — resolve Issue 1 first.**~~ ✅ **RESOLVED** — branch reconciled with merged PP-1642/PP-1644, `mergeable_state: clean`, `useUpdateJobReturn.test.ts` present and passing.
2. **Run the full validation suite** (`test` / `typecheck` / `lint` / `build`) on the reconciled branch. Affected tests / typecheck / lint are green here; the full `turbo run test` + `build` should still be run before sign-off per CLAUDE.md.
3. ~~**Confirm OMS semantics for `proofedUserId` on PUT** (Issue 3).~~ ✅ **RESOLVED** — no longer forwarded from `putJob.ts`.
4. **Fix the `currentJobIndex === -1` guard (Issue 4)** and add a test — the one remaining code defect.
5. ~~**Decide on `addJobWithTasks` (Issue 5).**~~ ✅ **RESOLVED** — mixture now forwards the PP-1419 fields.
6. ~~**Confirm the changed sidebar/dashboard behaviour (Issue 2)** with the ticket owner.~~ ✅ **RE-ASSESSED as not-valid** — the sidebar hook change is a behaviour-preserving refactor and the displayed-Job-Due-Date change is develop's PP-1644 baseline. A quick sidebar date-edit smoke-test is still good hygiene.
7. Address low-severity items 9 (Live-status guard) and 7 (skeleton backstop) as small follow-ups; treat 6 & 8 as optional.
