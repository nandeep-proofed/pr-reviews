# PR Review: PP-1644 — Dynamic Return Time — Creative Portal Bulk Actions & Order Table

**PR:** https://github.com/Proofed/B2BWebserver/pull/2307
**Jira:** https://proofed.atlassian.net/browse/PP-1644
**Status:** Code Review (Jira)
**Base branch:** `feature/PP-1642-job-edit-mode` (stacks on PP-1642)
**Head:** `bb71dcc4f` — `feature/PP-1644-bulk-actions-order-table`

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Req 1.1–1.3 — Bulk shift moves every job's `maxReturnTime` by `delta`, never `returnTime` | `buildShiftedJobMaxReturnTimes` (api/utils/jobs/bulkJobDueDateShift.ts) emits one `JobPut` per affected job with only `maxReturnTime`; Shift mode iterates current job + downstream, Target mode emits only the current job. Wired through `BulkDeadlineModal/hooks.ts` `handleChangeDueDates` Jobs branch. | ✅ Addressed |
| Req 2.1.a — `returnTime <= shifted maxReturnTime` for every assigned job | `validateJobMaxReturnShift` walks affected jobs and emits `returnTimeExceedsMax` conflicts; surfaced inline through `getShiftConflictMessage`. | ✅ Addressed |
| Req 2.1.b — `shifted Job[final].maxReturnTime <= newDueDate` | Implemented as a warn-and-confirm (not a hard block) via the `setDeadlineModalProps` / `showDeadlineWarningModal` flow — mirrors the per-job picker. Matches the spec's intent ("…landing after the order deadline raises the existing warn-and-confirm modal" — PR description). | ⚠️ Partial (warn vs block — confirm with PM if hard-block was required) |
| Req 2.1.c — Sequential integrity: `Job[N].maxReturnTime > Job[N-1].maxReturnTime` | In Shift mode preserved structurally (all downstream jobs move by same delta). In Target mode the new `downstreamCeiling` conflict guards against pushing the current job past the next unassigned job's user-DD ceiling (`nextMax − nextWindow`). Strict pairwise sequencing (max[N] > max[N-1] for the **next assigned** job's hard stop) is **not** explicitly validated for Target mode. | ⚠️ Partial |
| Req 2.2 — Block bulk and surface conflict for manual resolution | Inline error rendered via `setError(...)` derived from `getShiftConflictMessage`; modal stays open. | ✅ Addressed |
| Req 3 — Bulk target due-date validation (assigned returnTime ≤ target; sequential integrity) | Same `validateJobMaxReturnShift` with `isTarget: true`; `downstreamCeiling` guards next unassigned job. | ⚠️ Partial (no explicit next-assigned-job `maxReturnTime` ceiling check) |
| Req 4.1 — Deadline sort key = `effectiveDeadline = returnTime ?? maxReturnTime` | New optional `currentJobMaxReturnTime` on `OrderFromSearch`; `mapOrderFromApiToTable` derives `currentJobEffectiveDeadline = currentJobReturnTime \|\| currentJobMaxReturnTime`; `sortByTimeRemaining` keys on `currentJobEffectiveDeadline ?? deadline`. | ✅ Addressed |
| Req 4.2 — Deadline column visually distinguishes locked vs hard-stop | `DeadlineDisplay` gains optional `placeholder` (defaults to `"—"` from `tableColumns.tsx` for the current-job line when `currentJobDeadline` is empty). Display of the hard-stop value itself is owned by PP-1792 per PR description. | ✅ Addressed (placeholder only) |
| Req 5.1 — Overdue filter matches assigned jobs whose `returnTime < now`; excludes unassigned | `applyServerSideOrderFilters` Overdue branch now hard-requires `order.currentJobReturnTime` (UTC-parsed) and compares to `currentDate`. Old fallback to `order.dueDateTime` removed. | ✅ Addressed |
| Testing Notes: At-Risk / Due-Today filters | Explicitly out of scope per PR description and ticket's Requirements §5 (which only lists Overdue). | ➖ Not in scope |

Extras beyond Jira scope (worth flagging):
- **`getBulkActionsData` fetch strategy rewrite** — switched from `fetchAssignedJobsByJobSequence` (which dropped unassigned/in-queue jobs and broke Apply silently) to `fetchJob(id)` per `jobSequence` entry. Necessary to make Req 1 work, but it's a real behavior change worth highlighting.
- **`pages/jobs/utils.ts` `jobItemToJobTableItem`** — the legacy fallback `jobItem.returnTime` is removed; `effective` and `isOverdue` now always read `maxReturnTime`. Same intent as bb71dcc4f's "remove dead legacy-order return-time handling" but reaches into the Available/Assigned-Jobs surfaces. Confirm no remaining legacy orders flow through this path.
- **`useUpdateJobReturn.ts`** — guard relaxed from `if (!job || !job.maxReturnTime) return;` to `if (!job) return;`. Now relies on `maxReturnTime` always being present. See Issue #3.

---

## Architecture Analysis

The PR is structured along clean seams:

1. **Pure helpers** (`api/utils/jobs/bulkJobDueDateShift.ts`) encapsulate shift math + validation as deterministic functions over `Job[]`, returning a discriminated conflict structure. This is unit-tested independently of the React hook and keeps the modal hook declarative.

2. **Hook wiring** in `BulkDeadlineModal/hooks.ts` follows a clear validate → block → warn → apply pipeline for the Jobs branch. The Orders branch is unchanged.

3. **Data layer** — `OrderFromSearch` gains `currentJobMaxReturnTime?` and `OrderTableSchema` gains `currentJobEffectiveDeadline?`. The split between sort key (`currentJobEffectiveDeadline`, `returnTime`-first) and display value (`currentJobDeadline`, owned by PP-1792 to be `maxReturnTime`-first) is intentional but worth re-confirming with PM — for an assigned job these can diverge.

4. **Fetch rewrite** — `getBulkActionsData` now fans out per-job lookups instead of the OMS multi-search. This makes the data complete for unassigned jobs but trades batched calls for individual ones; per-order load grows from O(chunks of 5) to O(jobs). Acceptable for typical workflows but worth tracking.

The split into `Shift` / `Target` modes is consistent with PP-1642's sidebar picker semantics — admins won't have to learn two mental models.

---

## Issues Found

### 1. Inline error in handleChangeDate Target branch checks the wrong job

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/BulkToolbarActions/Modals/BulkDeadlineModal/hooks.ts]**

**Function/Class:** `useBulkDeadlineModal` → `handleChangeDate` (line 331–347)

**Severity:** medium

**Problem:** In Target mode the synchronous check inside `handleChangeDate` evaluates `currentJobs.some((job) => isBefore(new Date(job.maxReturnTime), currentDate))` — i.e. "any current job's existing max is before the picked date." Whenever the admin picks a later date than the current job's current `maxReturnTime` (a perfectly normal Target operation), this fires and calls `setError("Must be before next due date")`. The companion `useEffect` (line 74–113) does the *correct* check against the **next** job's `maxReturnTime`, but it only ever calls `setError(...)`; it never clears a previously-set error.

**Impact:** Picking a valid Target date that happens to be later than the current job's existing hard stop puts a misleading inline error on the modal until the admin re-picks (or until the apply-time `validateJobMaxReturnShift` does the real check). The Apply button is still gated correctly via the conflict-list path in `handleChangeDueDates`, but the inline UX flashes a false-negative message. The duplicated check is the root cause — both `handleChangeDate` and the `useEffect` try to validate the picker, and they disagree.

**Fix:** Delete the in-`handleChangeDate` validation branch (Target mode) and rely on the existing `useEffect` to set/clear the error, OR replace the body with a `currentJobs[i].nextJob.maxReturnTime` lookup. Also have the `useEffect` `setError(undefined)` on the success branch so stale errors clear when the admin picks a different date.

```ts
// In handleChangeDate, drop the broken isTargetSelected branch:
if (isTargetSelected) return; // useEffect handles validation
```

And in the useEffect:

```ts
if (hasJobWithDueDateBeforeCurrentDate) {
  setError("Must be before next due date");
} else {
  setError(undefined); // <-- clear stale errors
}
```

---

### 2. Per-job fetch swallows failures silently; bulk operates on a quietly-incomplete jobs array

**[File: apps/creative-portal/api/mixtures/orders/getBulkActionsData/getBulkActionsData.ts]**

**Function/Class:** `getBulkActionsData` (line 59–71)

**Severity:** medium

**Problem:** The new per-job fan-out wraps each `fetchJob(id, requesterId)` in a `try/catch` that returns `null` on any failure, then filters nulls out:

```ts
const fetchedJobs = await Promise.all(
  jobIds.map(async (jobId) => {
    try {
      return await fetchJob(jobId, requesterId);
    } catch (error) {
      return null;
    }
  })
);
const orderJobs = fetchedJobs.filter((job): job is Job => job != null);
```

If even one job in `jobSequence` fails to fetch (transient OMS 5xx, network blip, auth race), the rest of the pipeline proceeds with `orderJobs` missing that entry. `bulkJobDueDateShift` then computes a `delta` and shifted `maxReturnTime` values **without that job in the sequence** — so:
- In Shift mode the missing job is silently skipped from the `JobPut[]`, leaving an out-of-sequence hard stop.
- The `validateJobMaxReturnShift` `returnTimeExceedsMax` check cannot detect a conflict on the missing job because it doesn't see it.
- The `downstreamCeiling` check uses the wrong "next job" if the next job is the one that failed.

The PR description's "fixes a silent no-op on Apply" replaced the empty-array silent failure with a partial-array silent failure that's harder to detect.

**Impact:** Bulk shifts could land an order in an internally-inconsistent state without surfacing an error. Hard to reproduce, but the failure mode is data corruption (sequence integrity), not a visible UX glitch.

**Fix:** Fail the bulk action when any job in `jobSequence` cannot be fetched. Log + return the order with `orderJobs: null/empty`, and have the hook treat that order as ineligible and surface a per-order error, or reject the whole bulk request.

```ts
const fetchedJobs = await Promise.all(
  jobIds.map((jobId) => fetchJob(jobId, requesterId))
);
// throws if any reject — handled by the outer try/catch + handleEndpointError
const orderJobs = fetchedJobs;
```

If partial recovery is desired, mark the order with an explicit `hasIncompleteSequence: true` and have `handleChangeDueDates` filter it out before validating.

---

### 3. useUpdateJobReturn now dereferences maxReturnTime without a guard

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderJobs/hooks/useUpdateJobReturn.ts]**

**Function/Class:** `useUpdateJobReturn` callback (line 186–191)

**Severity:** low

**Problem:** The guard changed from `if (!job || !job.maxReturnTime) return;` to `if (!job) return;`, and the next line still does `parseUtcDateString(job.maxReturnTime)`. If `maxReturnTime` is ever empty/undefined (e.g. an edge order from before the migration, or an OMS payload regression), `parseUtcDateString` will produce an `Invalid Date` and downstream comparisons silently misbehave.

**Impact:** Low under normal conditions because every job now carries a hard stop, but it's a hidden assumption that no longer matches the previous defensive contract. If you hit it in production, the inline picker flashes a wrong validation state with no error in the console.

**Fix:** Either keep the original guard or assert at the boundary:

```ts
if (!job?.maxReturnTime) return;
```

It's a one-line revert with no functional downside.

---

### 4. `downstreamCeiling` constraint ignores Canceled "next" jobs

**[File: apps/creative-portal/api/utils/jobs/bulkJobDueDateShift.ts]**

**Function/Class:** `validateJobMaxReturnShift` (line 119–143)

**Severity:** low

**Problem:** `getDownstreamJobConstraint(orderJobs[currentJobIndex + 1])` is passed the *raw* next slot in the sequence. The `isParticipating` filter (`status !== "Canceled"`) is only applied inside `getAffectedJobs`; it is not applied when picking the downstream ceiling. If the immediate next job is Canceled but a participating job lives further downstream, the Target check either:
- uses the Canceled job's `maxReturnTime` minus its `returnWindowsMinutes` (artificially tight ceiling), or
- returns `undefined` because the Canceled job has a `returnTime` set (skipping the check entirely when a real, further-downstream unassigned job should have constrained it).

**Impact:** Edge case (Canceled mid-sequence is rare), but produces either false-positive blocks or false-negative passes. Both fail closed silently — admin retries or hits a server-side validation later.

**Fix:** Walk forward to the first participating job before calling `getDownstreamJobConstraint`:

```ts
const nextParticipating = orderJobs
  .slice(currentJobIndex + 1)
  .find(isParticipating);
const downstream = getDownstreamJobConstraint(nextParticipating);
```

---

### 5. Sort key falls back to order due date — but only on lookup, not on `OrderFromSearch` rows missing both timing values

**[File: apps/creative-portal/services/orders/utils.ts]**

**Function/Class:** `mapOrderFromApiToTable` (line 116–117) + `sortByTimeRemaining` (utils.ts line 130)

**Severity:** low

**Problem:** `currentJobEffectiveDeadline = order.currentJobReturnTime || order.currentJobMaxReturnTime` correctly handles OMS's `""` for unassigned current jobs. But the test at `services/orders/utils.test.ts:142+` confirms it ends up `undefined` when both fields are missing — and `sortByTimeRemaining` then uses `item.currentJobEffectiveDeadline || item.deadline` as the fallback. That's fine when `deadline` is present; if an order ever lands without `deadline` either, the sort key parses as `new Date("Z")` → `Invalid Date` → `NaN`, and the comparator produces `NaN` results that mix order.

**Impact:** Unlikely in practice (an order without any deadline shouldn't reach the table), but the fallback chain has no terminating safety. Consider whether to drop such rows from the sort or anchor on a known sentinel.

**Fix:** Guard in `getEffectiveSortDeadline` with a sentinel (`new Date(0)` or `Infinity`) instead of relying on `new Date("undefinedZ")`:

```ts
const getEffectiveSortDeadline = (item: OrdersTableItem): Date => {
  const raw = item.currentJobEffectiveDeadline || item.deadline;
  return raw ? new Date(`${raw}Z`) : new Date(8640000000000000); // sort to end
};
```

---

### 6. Lint failures (Prettier) in newly added test files

**[File: multiple test files added by this PR]**

**Function/Class:** N/A — formatting only

**Severity:** medium (blocker per CLAUDE.md "zero failures and zero warnings")

**Problem:** `npx turbo run lint` on `pr-2307-review` reports **157 errors** in `@proofed/creative-portal`; develop reports 150. The PR introduces **7 new prettier errors**, all of the form `as Type)` vs `) as Type` placement, in:

- `api/mixtures/orders/getBulkActionsData/getBulkActionsData.test.ts:81, 92`
- `api/orders/utils.test.ts:112, 229`
- `api/utils/jobs/bulkJobDueDateShift.test.ts:34`
- `components/molecules/tables/TableWithFilters/partials/BulkToolbarActions/Modals/BulkDeadlineModal/hooks.test.ts:73, 83`
- `components/molecules/tables/TableWithFilters/utils.test.ts:111`
- `services/orders/utils.test.ts:29`

**Impact:** CLAUDE.md mandates "0 lint errors" pre-commit. The wysiwyg lint failures are pre-existing baseline noise, but the 7 new errors above were introduced by this PR's test scaffolding (`{...} as Job` inside `()` parens). They are all auto-fixable.

**Fix:** Run `yarn lint:fix` (or `yarn lint --fix`) — every error is `prettier/prettier` and trivially auto-correctable. Example:

```ts
// Before
const buildJob = (overrides: Partial<Job> = {}): Job =>
  ({
    id: 1,
    // ...
    ...overrides
  } as Job);

// After (auto-fixed)
const buildJob = (overrides: Partial<Job> = {}): Job =>
  ({
    id: 1,
    // ...
    ...overrides
  }) as Job;
```

---

### 7. `handleUpdateJobs` is not awaited inside try/catch — bulk error toast unreachable for async failures

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/partials/BulkToolbarActions/Modals/BulkDeadlineModal/hooks.ts]**

**Function/Class:** `handleChangeDueDates` (line 398–500)

**Severity:** low

**Problem:** Inside `try { ... } catch { showToast(...) }`, both call sites for `handleUpdateJobs(jobsToUpdate, JOB_DUE_DATE_SUCCESS_MESSAGE)` (line 476–478, 489–492) are fire-and-forget. The surrounding try/catch will never observe an async rejection from those calls; the "Failed to update bulk deadlines" toast is effectively unreachable for the Jobs branch.

**Impact:** If `handleUpdateJobs` already surfaces its own error toast on failure, this is cosmetic. If it doesn't, bulk-update failures silently close the modal with no user-facing signal. Worth one grep through `useBulkActionsContext` to confirm.

**Fix:** Either `await` the calls inside try, or document that `handleUpdateJobs` owns the error UX (and remove the dead `catch` here).

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ✅ | 1091/1091 pass across 119 files (`@proofed/creative-portal`); other packages cached/clean |
| `npx turbo run typecheck` | ✅ | 0 errors across all 5 typechecked workspaces |
| `npx turbo run lint` | ❌ | 157 errors in `@proofed/creative-portal` (develop baseline: 150 → **7 new from this PR**, all prettier formatting); 62 errors in `@proofed/wysiwyg-editor` (pre-existing). All PR-introduced errors are auto-fixable via `yarn lint:fix`. |
| `npx turbo run build` | ✅ | Clean — `creative-portal` and `wysiwyg-editor` both build with 0 type warnings |

---

## Tests

- ✅ `bulkJobDueDateShift.test.ts` — covers Shift mode delta cascade, Target single-job, returnTime preservation, Canceled-skip, both conflict types, message variants. Solid coverage of the new helper.
- ✅ `BulkDeadlineModal/hooks.test.ts` — covers the orchestration: clean apply, blocked-with-inline-error, warn-and-defer; Orders mode regression test. Mock setup is principled (real `bulkJobDueDateShift` runs).
- ✅ `DeadlineDisplay/index.test.tsx` — placeholder behavior for undefined / empty-string / present dateTime.
- ✅ `TableWithFilters/utils.test.ts` — `sortByTimeRemaining` asc/desc, current-job-vs-order priority, undefined fallback.
- ✅ `api/orders/utils.test.ts` — Overdue filter: past returnTime matches, unassigned excluded, future returnTime excluded even when order due date is past.
- ✅ `services/orders/utils.test.ts` — `currentJobEffectiveDeadline` locked-first, fallback to maxReturnTime on `""`, undefined when both missing.
- ✅ `getBulkActionsData.test.ts` — rewrites cover fan-out per-id fetch, unassigned-job inclusion, large sequence (11 jobs), individual-failure skip behavior.
- ✅ `OrderJobs/utils.test.ts` — `getDownstreamJobConstraint` now keys off `jobType` not `description` (catches the original "undefined's standard window" bug).
- ⚠️ No test confirms the **Issue #1** broken inline-error path; the hook test only covers the apply pipeline, not the picker's per-change validation feedback.
- ⚠️ No test for the **Issue #2** partial-fetch silent-failure scenario; consider adding a `mockFetchJob.mockRejectedValueOnce(...)` case that asserts bulk does not silently proceed.
- ✅ Validation suite: 1091/1091 tests pass.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Two medium concerns (Issue #1 inline-error UX bug, Issue #2 silent partial-fetch); behavior is otherwise correct |
| Regression risk | ⚠️ Medium — `getBulkActionsData` fetch strategy rewrite + `jobItemToJobTableItem` legacy-removal touch broadly-used paths |
| Tests | ✅ Comprehensive for the new code; small gaps around the issues above |
| Code quality | ✅ Pure helpers + declarative hook wiring; clear separation of sort key vs. display value |
| Validation suite | ❌ Lint fails (Issue #6 — 7 new prettier errors); test/typecheck/build pass |
| Mergeable state | ❌ Per CLAUDE.md: do not commit until lint is clean. GitHub `mergeable_state` reports `clean` against the base branch, but the project's own pre-commit gate is failing. |

---

## Recommendation

**Request changes (mostly small).** The architecture and Jira coverage are solid; the issues are concentrated in two real bugs plus housekeeping.

1. **Blocker:** Run `yarn lint:fix` to clear the 7 new prettier errors in the new test files (Issue #6). CLAUDE.md mandates 0 lint failures before commit.
2. **Fix:** Remove the broken Target-mode inline check in `handleChangeDate`, and have the companion `useEffect` clear the error on its success branch (Issue #1).
3. **Fix:** Decide on a failure policy for `getBulkActionsData` per-job fetch failures — either fail loud or mark the order ineligible (Issue #2). The current silent-partial behavior is worse than the bug it replaced.
4. **Nice-to-have:** Restore the `maxReturnTime` guard in `useUpdateJobReturn.ts` (Issue #3), filter Canceled when picking the downstream ceiling (Issue #4), and either `await` `handleUpdateJobs` or remove the dead catch (Issue #7).
5. **Confirm with PM:** Req 2.1.b is implemented as a warn-and-confirm rather than a hard block — verify this matches the intended UX. Also confirm the sort-vs-display split (Req 4.1 uses `returnTime`-first sort while PP-1792 will show `maxReturnTime`-first) is acceptable.
6. **Manual verification:** PR description still has the "yarn dev visual check" testing item unchecked — confirm in a browser before merging.
