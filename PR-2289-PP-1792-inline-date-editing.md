# PR Review: PP-1792: Add inline date editing to Order Management dashboard

**PR:** https://github.com/Proofed/B2BWebserver/pull/2289
**Jira:** https://proofed.atlassian.net/browse/PP-1792
**Status:** Code Review
**Branch:** `feature/PP-1792-inline-date-editing-rebased` → `develop`
**Size:** 22 files, +2893 / −195, 8 commits
**Mergeable state (GitHub):** `clean` (was `dirty` at first review — reconciled with develop, see Issue 1)

> **Update (2026-07-16):** Re-checked against the current branch head. **Issues 1, 2, 3, 4 and 5 are now resolved / not-valid** — see the ✅ notes on each. Only low-severity **7, 9** (and optional **6, 8**) remain. Locally re-verified: affected unit tests pass, `typecheck` 0 errors, `lint` 0 errors on changed files.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1. Due date & deadline fields support inline editing | New `DeadlineCellContent` renders two `InlineDeadlineEditor`s (job due date + order deadline), replacing the raw `DeadlineDisplay`s in `tableColumns.tsx` | ✅ Addressed |
| 2.1 Calendar icon shown on hover of a date field | `InlineIconWrapper` is `opacity: 0` → `1` on `DateRow:hover` / `:focus-visible` | ✅ Addressed |
| 3 / 3.1 / 3.2 Icon colour matches date colour state (red when overdue) | `iconColor = isOverdue ? theme.colors.red : color`; `DeadlineDisplay` independently forces `red` when `remainingTime < 0` | ✅ Addressed |
| 4.1 / 4.2 Hovering icon turns it green (matches sidebar) | `&:hover svg, &:focus-visible svg { color: green1 }` | ✅ Addressed |
| 5. Clicking icon opens calendar overlay | `Dropdown` + `DeadlineDatePicker` opened via trigger `onClick={handleOpen}` | ✅ Addressed |
| 6.1 / 6.2 Overlay uses existing job-due-date / order-deadline flows | Reuses `DeadlineDatePicker` + `useUpdateJobReturnBase` (a behaviour-preserving refactor of the sidebar hook) and a new `applyOrderDeadline` mirroring the sidebar | ✅ Addressed (sidebar validation preserved — see Issue 2) |
| 7. User can submit updated date | `onApply={handleApply}` | ✅ Addressed |
| 8.1 / 8.2 On success, dashboard shows updated date + refreshed styling/icon | `refetchDashboard()` + prop-diff `useEffect` clears the per-type loading skeleton | ✅ Addressed (see Issue 7 for a stuck-skeleton edge case) |
| 9. Applies to both due date & deadline wherever displayed | Both editors rendered in the deadline column | ✅ Addressed |
| 10 / 11. Not available for Completed/Cancelled; no icon, non-interactive | `if (FINISHED_ORDER_STATUSES.includes(orderStatus)) return null` | ✅ Addressed |
| 12 / 12.1 Limited to exposing the existing flow; existing validation, permissions & date-update rules unchanged | Existing sidebar flow is preserved (behaviour-preserving refactor); the displayed-Job-Due-Date change is develop's PP-1644 baseline, not this PR | ✅ Addressed — see Issue 2 re-assessment |

**Scope beyond the ticket (flagged):**

- `api/jobs/[jobId]/putJob.ts`, `api/jobs/types.ts` — new PP-1419 fields added to the Job PUT contract. (The unconditional `proofedUserId` forwarding flagged in Issue 3 has since been removed — see Issue 3.)
- `services/orders/utils.ts` — the returnTime→maxReturnTime display of the Job Due Date is **develop's PP-1644 Req 4.1 behaviour**; PP-1792's diff here is a comment only (see Issue 2).
- `OrderJobs/utils.ts` — new `applyChainTimingsFromEditedJob` / `normalizeJobDurations` chain calculator (used by the new inline path).
- The `DeadlineWarningModal` extraction folds in PP-1827's double-click guard for all three call sites (a genuine, reasonable refactor).

---

## Architecture Analysis

The core design is sound and follows repo conventions well:

- **Hook/UI split** is respected: `DeadlineCellContent/index.tsx` is UI-only, with `useInlineOrderData` (shared per-row order fetch + mutations) and `useInlineDateEdit` (per-date-type picker state) in `hooks.ts`.
- **Reuse-first**: `DeadlineDatePicker`, `DeadlineDisplay`, `IconCalendar`, `Dropdown`, and the sidebar's return-time logic are reused rather than duplicated. Extracting `DeadlineWarningModal` into `components/molecules/` removes three copies of identical modal JSX.
- **`useUpdateJobReturn` split** into a handlers-as-params `useUpdateJobReturnBase` core + a thin context-bound wrapper reuses the sidebar's validation without dragging in `useOrderSidebarContext`; the public signature is unchanged, so `JobCard.tsx` is unaffected.

~~The main architectural tension is **timing and duplication with `develop`**.~~ **(Resolved — Issue 1.)** The branch merged `develop` and reconciled the return-time logic with PP-1642/PP-1644; GitHub now reports `clean`.

A secondary concern is **per-row cost**: the deadline cell mounts a (disabled) order query observer, two mutation observers, and two `useInlineDateEdit` instances per order row (Issue 6).

---

## Issues Found

### 1. Branch is behind `develop` and unmergeable; overlaps the merged PP-1642/PP-1644 rework — ✅ RESOLVED

**Severity:** high

**Problem:** GitHub reported `mergeable_state: "dirty"`; the branch predated the merged Dynamic Return Time epic (PP-1642/PP-1644) that reworked the same surfaces.

**Resolution (verified 2026-07-16):** The branch merged current `develop` (`d5e529f9f`) and reconciled the return-time logic across four follow-up commits (`dbe055500`, `2b1c26cc4`, `b2d007204`, `9408aa6f7`). GitHub now reports **`mergeable_state: "clean"`**, and `useUpdateJobReturn.test.ts` (absent at first review) is now present and passing.

### 2. Existing sidebar date-update behaviour changes, contrary to req 12.1 — ✅ RESOLVED / not valid

**Severity:** medium

**Problem (as originally raised):** the review flagged that `updateReturnTime` now uses `applyChainTimingsFromEditedJob` / validates the `maxReturnTime` sequence, and that `resolveCurrentJobDeadline` changes the displayed Job Due Date for every unassigned order.

**Re-assessment (verified 2026-07-16 — not valid):** Both cited changes are develop's already-merged PP-1642/PP-1644 baseline, not this PR:
- **`services/orders/utils.ts`** — PP-1792's diff is a **comment-only change**; the `currentJobReturnTime || currentJobMaxReturnTime` display is develop's **PP-1644 Req 4.1** behaviour.
- **`useUpdateJobReturn.ts`** — a **behaviour-preserving refactor** (Base+wrapper, unchanged public signature, same sidebar handlers). The sidebar's `updateReturnTime` still uses only `updateJobReturnTimes` and does **not** call `applyChainTimingsFromEditedJob` (that's only in the new inline path). The one other edit is `order.dueDateTime` → `order?.dueDateTime` null-safety (identical when `order` is defined).

Req 12.1's "existing sidebar flow unchanged" is upheld. **Residual (not a defect):** a quick sidebar date-edit smoke-test is worthwhile refactor hygiene.

### 3. `putJob.ts` now forwards `proofedUserId` on every job PUT — ✅ RESOLVED

**Severity:** medium

**Problem:** the handler had begun echoing `proofedUserId` on every PUT (risking an unintended re-assignment side effect on unrelated sidebar edits).

**Resolution (verified 2026-07-16):** `putJob.ts` no longer references `proofedUserId`; `jobPutData` is limited to the intended fields. The unconditional forwarding was removed.

### 4. Unhandled crash when the current job is not in the fetched job list (`currentJobIndex === -1`) — ✅ RESOLVED

**[File: DeadlineCellContent/hooks.ts · OrderJobs/hooks/useUpdateJobReturn.ts]**

**Function/Class:** useInlineDateEdit.handleApplyJobDueDate → useUpdateJobReturnBase.updateReturnTime

**Severity:** medium

**Problem:** `currentJobIndex = jobs.findIndex(j => j.id === orderData?.currentJobId)` is `-1` when `currentJobId` is null/0 or absent from `orderJobs`. The job-due-date editor still renders from the row's `currentJobDeadline` prop, so a user could open it and click Apply; `handleApplyJobDueDate` only guarded `!deadlineDate || !orderData?.dueDateTime`, then `updateReturnTime` read `jobs[-1]` → `currentJob.status` → `TypeError: Cannot read properties of undefined`. Confirmed reachable via a list/detail data inconsistency (row exposes a `currentJobDeadline` but the fetched `currentJobId` isn't in `orderJobs`); does not fire on consistent data.

**Resolution (commit `e917142e`, verified 2026-07-16):**
- `updateReturnTime` (shared core) now bails when `jobs[selectedJobIndex]` is undefined, before reading `.status` — protects the sidebar and inline callers alike.
- `handleApplyJobDueDate` early-returns when `currentJobIndex < 0` (added to the guard + deps), so the apply never starts and there's no loading flicker.
- Added a unit test for the `selectedJobIndex === -1` no-op path (`useUpdateJobReturn.test.ts`); the suite passes (28 tests).

### 5. `addJobWithTasks` mixture will be rejected at runtime (documented, not fixed) — ✅ RESOLVED

**Severity:** medium

**Problem:** a `TODO(PP-1419)` acknowledged the mixture's `upcomingJob` PUT lacked the now-required timing fields and would be rejected by OMS.

**Resolution (verified 2026-07-16):** the `TODO(PP-1419)` is gone and the mixture now forwards `returnWindowsMinutes` + `maxReturnTime` (guarded on `insertedJobTiming` windows) for the upcoming job.

### 6. Every dashboard row now mounts an order query + mutation observers

**Severity:** low

**Problem:** for every rendered row the cell instantiates `useOrderByIdQuery` (disabled), `useOrderPutMutation`, `useJobMutation`, `useQueryClient`, `useZonedTime`, plus two `useInlineDateEdit` instances. Overhead, not a correctness bug.

**Fix:** Optional — lazily create the query/mutations only when a picker first opens, or memoise `InlineDeadlineEditor`. Not a blocker.

### 7. Loading skeleton can stick if the post-mutation dashboard value is unchanged

**Severity:** low

**Problem:** the per-type skeleton is cleared only by the prop-diff `useEffect`. If a mutation succeeds but the recomputed value is identical, the prop never changes and the skeleton is not cleared. `refetchDashboard`'s one-shot reference-equality retry mitigates but does not guarantee a prop change.

**Status (2026-07-16):** ❌ Still open (low).

**Fix:** Also clear the updating state on mutation settle (success/error) as a backstop.

### 8. `putJob.ts` performs no runtime validation of the now-required fields

**Severity:** low

**Problem:** `parseReqBody<JobPut>(req)` is a type assertion only; a caller omitting the PP-1419 fields forwards `undefined` and gets an opaque OMS 4xx instead of a clear BFF 400.

**Fix:** Optional — add a yup schema / guards returning `handleBadRequest` when absent.

### 9. Order-deadline warning path skips the "must be Live" guard

**[File: DeadlineCellContent/hooks.ts]**

**Function/Class:** applyOrderDeadline

**Severity:** low

**Problem:** when the new order deadline falls before the last job's return time, the warning-confirm callback calls `mutateOrder(...)` directly; the `status !== "Live"` check only exists on the non-warning branch, so a non-Live (e.g. Paused) order can be mutated through the warning path.

**Status (2026-07-16):** ❌ Still open — the `status !== "Live"` check is still after the warning branch.

**Fix:** Perform the `status === "Live"` check once up front, before branching, so both paths enforce it.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| Affected unit tests (creative-portal) | ✅ | Pass on the current branch — incl. the new `selectedJobIndex === -1` test (Issue 4). Re-run 2026-07-16 |
| `turbo run typecheck` (creative-portal) | ✅ | 0 errors (current branch) |
| `lint` (changed files) | ✅ | 0 errors (current branch) |
| `turbo run build` | ⏭️ Not run | — |

---

## Tests

- ✅ New unit tests added for `DeadlineCellContent`, `applyChainTimingsFromEditedJob`, `resolveCurrentJobDeadline`, plus a `tableColumns.test.tsx` mock addition.
- ✅ Good coverage of the tricky paths: the async-fetch race guard, Target-mode error, order-status guard, chain cascade, and the null-return legacy fallback.
- ✅ **Issue 4 gap closed** — `selectedJobIndex === -1` no-op path now covered.
- ⚠️ The component/hook tests mock nearly every dependency, so they verify wiring more than real integration.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ The unguarded crash path (Issue 4) is fixed; core inline flow correct. Issues 2 & 3 resolved |
| Regression risk | ✅ Low — Issues 1, 2, 3 resolved; sidebar flow is a behaviour-preserving refactor |
| Tests | ✅ New tests added with good path coverage; ⚠️ heavily mocked |
| Code quality | ✅ Clean hook/UI split, good reuse, sensible `DeadlineWarningModal` consolidation |
| Validation suite | ✅ affected tests / typecheck / lint green (build not run) |
| Mergeable state | ✅ `clean` |

---

## Recommendation

**Approve with minor follow-ups** — all high/medium issues are resolved; only two low-severity items remain.

1. ~~**Blocker — resolve Issue 1.**~~ ✅ RESOLVED — reconciled with develop, `clean`.
2. **Run the full validation suite** (`test` / `typecheck` / `lint` / `build`) before sign-off per CLAUDE.md. Affected tests / typecheck / lint are green here.
3. ~~**Confirm OMS `proofedUserId` semantics (Issue 3).**~~ ✅ RESOLVED — no longer forwarded.
4. ~~**Fix the `currentJobIndex === -1` guard (Issue 4).**~~ ✅ RESOLVED — guarded in `updateReturnTime` + `handleApplyJobDueDate`, with a test (commit `e917142e`).
5. ~~**Decide on `addJobWithTasks` (Issue 5).**~~ ✅ RESOLVED — mixture now forwards the PP-1419 fields.
6. ~~**Confirm the changed sidebar/dashboard behaviour (Issue 2).**~~ ✅ RE-ASSESSED as not-valid — behaviour-preserving refactor; develop's PP-1644 baseline. A sidebar smoke-test is still good hygiene.
7. Address low-severity **9** (Live-status guard) and **7** (skeleton backstop) as small follow-ups; treat **6** & **8** as optional.
