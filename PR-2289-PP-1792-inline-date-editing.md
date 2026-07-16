# PR Review: PP-1792: Add inline date editing to Order Management dashboard

**PR:** https://github.com/Proofed/B2BWebserver/pull/2289
**Jira:** https://proofed.atlassian.net/browse/PP-1792
**Status:** Code Review
**Branch:** `feature/PP-1792-inline-date-editing-rebased` → `develop`
**Size:** 22 files, +2893 / −195, 10 commits
**Mergeable state (GitHub):** `clean`

> **Update (2026-07-16):** Re-reviewed and driven to closure against the current branch head. **Issues 1, 2, 3, 4, 5, 7, 9 are resolved / not-valid**; **6 and 8 are valid-but-low, intentionally skipped** (no fix needed unless a concrete need appears). Locally re-verified: affected unit tests pass, `typecheck` 0 errors, `lint` 0 errors on changed files.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1. Due date & deadline fields support inline editing | New `DeadlineCellContent` renders two `InlineDeadlineEditor`s, replacing the raw `DeadlineDisplay`s in `tableColumns.tsx` | ✅ Addressed |
| 2.1 Calendar icon shown on hover of a date field | `InlineIconWrapper` is `opacity: 0` → `1` on `DateRow:hover` / `:focus-visible` | ✅ Addressed |
| 3 / 3.1 / 3.2 Icon colour matches date colour state (red when overdue) | `iconColor = isOverdue ? theme.colors.red : color` | ✅ Addressed |
| 4.1 / 4.2 Hovering icon turns it green (matches sidebar) | `&:hover svg, &:focus-visible svg { color: green1 }` | ✅ Addressed |
| 5. Clicking icon opens calendar overlay | `Dropdown` + `DeadlineDatePicker` opened via `onClick={handleOpen}` | ✅ Addressed |
| 6.1 / 6.2 Overlay uses existing job-due-date / order-deadline flows | Reuses `DeadlineDatePicker` + `useUpdateJobReturnBase` (behaviour-preserving refactor) and a new `applyOrderDeadline` mirroring the sidebar | ✅ Addressed (see Issue 2) |
| 7. User can submit updated date | `onApply={handleApply}` | ✅ Addressed |
| 8.1 / 8.2 On success, dashboard shows updated date + refreshed styling/icon | `refetchDashboard()` + prop-diff `useEffect` + settle-backstop clear the per-type skeleton | ✅ Addressed (Issue 7 backstop added) |
| 9. Applies to both due date & deadline wherever displayed | Both editors rendered in the deadline column | ✅ Addressed |
| 10 / 11. Not available for Completed/Cancelled; no icon, non-interactive | `if (FINISHED_ORDER_STATUSES.includes(orderStatus)) return null` | ✅ Addressed |
| 12 / 12.1 Existing validation, permissions & date-update rules unchanged | Existing sidebar flow preserved (behaviour-preserving refactor); the displayed-Job-Due-Date change is develop's PP-1644 baseline | ✅ Addressed — see Issue 2 |

---

## Architecture Analysis

The core design is sound and follows repo conventions well: the hook/UI split is respected (`DeadlineCellContent/index.tsx` UI-only, logic in `hooks.ts`); components and the sidebar's return-time logic are reused rather than duplicated; the `DeadlineWarningModal` extraction removes three copies of identical modal JSX; and `useUpdateJobReturn` was split into a handlers-as-params `useUpdateJobReturnBase` core + a thin context-bound wrapper with an unchanged public signature.

The original architectural tension — duplication/overlap with `develop`'s PP-1642/PP-1644 — was **resolved by the reconcile** (Issue 1); GitHub reports `clean`. The remaining consideration is per-row cost (Issue 6), assessed as valid-but-low.

---

## Issues Found

### 1. Branch behind `develop` and unmergeable; overlaps merged PP-1642/PP-1644 — ✅ RESOLVED

**Severity:** high. The branch merged current `develop` (`d5e529f9f`) and reconciled the return-time logic (`dbe055500`, `2b1c26cc4`, `b2d007204`, `9408aa6f7`); `mergeable_state` is now `clean` and `useUpdateJobReturn.test.ts` is present and passing.

### 2. Existing sidebar date-update behaviour changes, contrary to req 12.1 — ✅ RESOLVED / not valid

**Severity:** medium. Both cited changes are develop's PP-1642/PP-1644 baseline: `services/orders/utils.ts` is a comment-only diff (the display is PP-1644 Req 4.1), and `useUpdateJobReturn.ts` is a behaviour-preserving refactor (unchanged public signature, same sidebar handlers, `order?.dueDateTime` null-safety only; the sidebar still uses `updateJobReturnTimes`, never the chain calc). Req 12.1 is upheld. Residual: a sidebar smoke-test is good hygiene.

### 3. `putJob.ts` forwards `proofedUserId` on every job PUT — ✅ RESOLVED

**Severity:** medium. `putJob.ts` no longer references `proofedUserId`; `jobPutData` is limited to the intended fields.

### 4. Unhandled crash when the current job is missing (`currentJobIndex === -1`) — ✅ RESOLVED

**Severity:** medium. Resolved in commit `e917142e`: `updateReturnTime` bails when `jobs[selectedJobIndex]` is undefined (before reading `.status`); `handleApplyJobDueDate` early-returns when `currentJobIndex < 0`; unit test added for the `-1` no-op path.

### 5. `addJobWithTasks` mixture would be rejected at runtime — ✅ RESOLVED

**Severity:** medium. The `TODO(PP-1419)` is gone; the mixture now forwards `returnWindowsMinutes` + `maxReturnTime` (guarded on `insertedJobTiming` windows) for the upcoming job.

### 6. Every dashboard row mounts an order query + mutation observers — ⚠️ Valid but low (skipped)

**Severity:** low. Confirmed real per-row overhead (no virtualization), **but** the queries are disabled (no network), the observers are lightweight, the set is month-scoped and rendered in batches, and group-header/finished rows don't pay the full cost. Overhead, not a correctness bug; no profiling evidence of jank. **Skipped** (won't-fix). Optional wins if ever needed: gate `useInlineOrderData` off finished rows in `DeadlineCell`; lazily create the query + mutations on first picker-open.

### 7. Loading skeleton can stick if the post-mutation value is unchanged — ✅ RESOLVED

**[File: DeadlineCellContent/hooks.ts]**

**Severity:** low

**Problem:** the per-type skeleton was cleared only by the prop-diff `useEffect`; a successful mutation whose recomputed value was identical (re-applying the same date, or structural-sharing returning the same string) left the skeleton stuck until an unrelated refresh.

**Resolution (commit `4f26d155`, verified 2026-07-16):** added a settle-backstop — both `applyOrderDeadline` (order deadline) and the `updateJobs` handler (job due date) now `removeUpdatingDateType(...)` in a `finally`, so the skeleton always clears on mutation settle. The prop-diff effect remains the primary (flash-free) clear on a real change. Unit test added asserting the skeleton clears on a successful order-deadline update.

### 8. `putJob.ts` performs no runtime validation of the now-required fields — ⚠️ Valid but low (skipped)

**Severity:** low. `parseReqBody<JobPut>(req)` is a type assertion only, so a caller omitting the PP-1419 fields forwards `undefined` and gets an opaque OMS 4xx instead of a clear BFF 400. **Skipped** — diagnostics-only (a schema wouldn't fix any behaviour, just change one error code to another), no in-app caller that should send the fields omits them, and it's consistent with the route's pre-existing no-schema pattern (not a regression). Optional follow-up if BFF-level error clarity is later wanted.

### 9. Order-deadline warning path skipped the "must be Live" guard — ✅ RESOLVED

**[File: DeadlineCellContent/hooks.ts]**

**Severity:** low

**Problem:** in `applyOrderDeadline`, the `status !== "Live"` check lived only on the non-warning branch, so a non-Live (e.g. Paused) order whose new deadline fell before the last job's return time could be mutated via the warning-confirm callback.

**Resolution (commit `4f26d155`, verified 2026-07-16):** the Live-status check is hoisted above the warning branch, so both paths enforce it. Unit test added: a non-Live order with a deadline before the last job now hits the Live toast — no warning modal, no mutate.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| Affected unit tests (creative-portal) | ✅ | Pass on the current branch — incl. the new Issue 4 (`-1`), Issue 7 (settle) and Issue 9 (Live-guard) tests. Re-run 2026-07-16 |
| `turbo run typecheck` (creative-portal) | ✅ | 0 errors (current branch) |
| `lint` (changed files) | ✅ | 0 errors (current branch) |
| `turbo run build` | ⏭️ Not run | Run before final sign-off |

---

## Tests

- ✅ New unit tests added for `DeadlineCellContent`, `applyChainTimingsFromEditedJob`, `resolveCurrentJobDeadline`, plus `tableColumns.test.tsx`.
- ✅ Good coverage of the tricky paths: async-fetch race guard, Target-mode error, order-status guard, chain cascade, legacy fallback.
- ✅ **Review-driven gaps closed:** `selectedJobIndex === -1` no-op (Issue 4), skeleton settle-backstop (Issue 7), non-Live warning-path guard (Issue 9).
- ⚠️ The component/hook tests mock nearly every dependency, so they verify wiring more than real integration.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Crash path (4), stuck skeleton (7) and Live-guard gap (9) all fixed; core flow correct |
| Regression risk | ✅ Low — Issues 1, 2, 3 resolved; sidebar flow is a behaviour-preserving refactor |
| Performance | ⚠️ Low — per-row observer overhead (Issue 6), disabled/bounded; skipped |
| Tests | ✅ Good path coverage incl. the three review-driven additions; ⚠️ heavily mocked |
| Code quality | ✅ Clean hook/UI split, good reuse, sensible `DeadlineWarningModal` consolidation |
| Validation suite | ✅ affected tests / typecheck / lint green (full `test` + `build` still to run) |
| Mergeable state | ✅ `clean` |

---

## Recommendation

**Approve** — pending the two standard pre-merge steps below. All high/medium issues are resolved; both remaining low items (6, 8) are intentionally skipped with rationale.

1. **Run the full validation suite** (`turbo run test` + `build`) on the reconciled branch before merge, per CLAUDE.md. Affected tests / typecheck / lint are green here.
2. **Manual QA pass** of the test-plan checklist (sidebar / bulk / inline / PP-1789 & PP-1840 sanity), plus a quick sidebar date-edit smoke-test (the shared hook was refactored — Issue 2) and the inline flow on a legacy (pre-PP-1419) order.

_Resolved this cycle: Issue 4 (`e917142e`), Issues 7 & 9 (`4f26d155`). Issues 2, 3, 5 confirmed resolved in code; Issue 1 via the develop reconcile. Issues 6 & 8 skipped (valid but low)._
