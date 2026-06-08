# PR Review: feature/PP-1863: Job insertion shifts downstream maxReturnTime, preserves committed returnTime

**PR:** https://github.com/Proofed/B2BWebserver/pull/2310
**Jira:** https://proofed.atlassian.net/browse/PP-1863
**Status:** Code Review

> Note: This PR targets `feature/PP-1644-bulk-actions-order-table`, not `develop`. Numbers below are scoped to commits on top of that base.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1.1 Inserted job has non-null `returnWindowsMinutes` + `maxReturnTime` | Server stamps both from delivery calc; POST omits `returnTime` for unassigned (OMS §24.5) (`addJobWithTasks.ts:172-178`, `addNewJobs.ts:313-327`) | ✅ Addressed |
| 1.2 Start-of-sequence anchor on `orderStartTime + maxReturnWindowsMinutes` | `planJobInsertionTiming.ts:138-143` anchors on `orderCreationDatetime` when `insertAfterJobId == null`, falling back to `now` | ✅ Addressed |
| 1.3 After-job anchor on `previousJob.maxReturnTime + maxReturnWindowMinutes` | `planJobInsertionTiming.ts:144-167` | ✅ Addressed |
| 1.4 When the inserted job has `proofedUserId`, payload must include `returnTime` per anchor rules | **Not implemented.** Server `omit(newJobData, "returnTime")` for every insert (`addJobWithTasks.ts:173`, `addNewJobs.ts:314`); planner never emits a `returnTime`; client hooks only send `status: "In Queue"` and never `proofedUserId`. The UI may not exercise this today, but server-side support is absent. | ❌ Missing |
| 1.5 Unassigned insert must not send `returnTime` | Server strips `returnTime` from POST payload (`omit(newJobData, "returnTime")`) | ✅ Addressed |
| 2.1 Downstream `maxReturnTime` += inserted window | `planJobInsertionTiming.ts:172-185` | ✅ Addressed |
| 2.2 Shift persisted via Job PUT | Server calls `updateJob` with `mergeJobPutBody(...)` (`addJobWithTasks.ts:234-244`, `addNewJobs.ts:425-447`) | ✅ Addressed |
| 3.1 / 3.2 Downstream `returnTime` left unchanged | Planner emits only `maxReturnTime` deltas; tests assert no `returnTime` change | ✅ Addressed |
| 4.1 Inserted job has window + max | Planner hard-block on `windowMinutes <= 0` (`planJobInsertionTiming.ts:123-129`); route also 409s when delivery-calc omits the job | ✅ Addressed |
| 4.2 Sequential integrity across full updated workflow | Planner strict-increasing check on participating max chain (`planJobInsertionTiming.ts:204-222`) | ✅ Addressed |
| 4.3 `Job[final].maxReturnTime <= Order.dueDateTime` | Kept as the existing client-side **overridable warning modal** (per PR description). No server-side hard block. The admin can confirm past-deadline. | ⚠️ Partial (intentional — soft warning by design) |
| 4.4 Assigned downstream `returnTime <= shifted maxReturnTime` | Planner blocks when any participating `returnTime > shifted max` (`planJobInsertionTiming.ts:224-241`) | ✅ Addressed |
| 4.5 Block when previous-job anchor cannot be resolved | Two hard-blocks: unknown anchor id and anchor job missing `maxReturnTime` (`planJobInsertionTiming.ts:147-167`) | ✅ Addressed |
| 4.6 Conflict surfaced to admin | Single route: 409 with reason text. Bulk route: per-order `failed` entries with reason; client toasts a generic warning but **discards the specific reason** in some paths (see Issues #4). | ⚠️ Partial |

**Scope creep:** The PR also includes `planJobRemovalTiming.ts` (151 lines), its 398-line test suite, and integrations in `useDeleteReviewJobAndUpdateReturnJob` and `BulkActionsContextProvider.handleDeleteJobs` (latest commit `a25b5d361: PP-1863: Reflow downstream job hard-stops when a job is skipped`). PP-1863's user story and ACs are strictly about **post-live job insertion**; skip/remove timing is not part of this ticket's spec. The work is logically related (inverse of insertion) but it would belong to a sibling ticket. The base branch (PP-1644) already had a returnTime-diff loop for removal — this PR replaces it with the new maxReturnTime-based planner.

Also folded in:
- `buildJobPutBody.ts` + tests — whole-record PUT helper (consumed by both insertion and removal flows; also by `bulkJobDueDateShift`).
- `putJob.ts` / `putJobs.ts` schema relaxation — fields outside `id`/`orderId` are optional; both routes pick explicit fields and forward verbatim (OMS PUT is whole-record per §24.4).
- `OrderStatusCardInfo` template split into `DeadlineRow` partial — pure refactor.

---

## Architecture Analysis

The core insertion planner (`planJobInsertionTiming.ts`) is a well-shaped pure function: it sorts by `maxReturnTime` to reconstruct the sequence (defensive against arbitrary OMS array order), folds inserts over a working copy, and emits only the deltas the caller needs to write. The compounding case (QA two-config) is handled by chaining: insert #2 anchors on insert #1's synthetic node, downstream shifts accumulate. Returning a discriminated `ok: true | false` with a `reason` string is clean and the route translates it directly to 409.

Moving the timing logic server-side is the right call given:
- Compounding requires a single planner over both inserts; the client snapshot can't express that.
- OMS POST §24.5 / PUT §24.4 contracts are easier to enforce in one place.
- The downstream job list is sourced from a fresh `fetchJobsByOrderId`, not the client snapshot — eliminates the stale-cache write hazard.

The bulk route's per-order grouping with sequential processing is correct, and the partial-success re-plan (`addNewJobs.ts:385-419`) is a thoughtful detail — if insert #2 in an order fails after insert #1 succeeds, the downstream shift is re-planned for the created prefix so other orders' chains aren't over-shifted. Tests cover this explicitly.

`buildJobPutBody.ts` is small and load-bearing — it underpins the new whole-record contract for all PUT consumers (bulk dueDateShift, single updateJobReturn, removal reflow, downstream insertion shift). The "omit `returnTime` when null/undefined" gate is the right shape for the OMS contract.

Test coverage on the new pure planners is strong; the route tests exercise the happy path, the compounding case, the divergent-anchor block, and the partial-write re-plan.

---

## Issues Found

### 1. Req 1.4 (assigned inserted job's `returnTime`) is unimplemented server-side

**[File: apps/creative-portal/api/mixtures/jobs/addJobWithTasks/addJobWithTasks.ts]**

**Function/Class:** addJobWithTasks (and `planJobInsertionTiming` by extension)

**Severity:** medium

**Problem:** Jira Req 1.4 says when the inserted job is created **with** `proofedUserId`, the POST payload must include `returnTime` computed by the participating-job anchor rules. The current handler strips `returnTime` from `newJobData` for every insert via `omit(newJobData, "returnTime")` (line 173) and the planner never emits one. The schema and types still accept `returnTime` and `proofedUserId`, but the server discards them silently.

**Impact:** If the client ever sends an assigned insert (UI work could land that lands in a sibling ticket), OMS will reject the POST per §24.5 (assigned job missing `returnTime`), or — worse — accept an "unassigned" record that the UI shows as assigned, depending on how the route is invoked. Today both `useAddNewJob` and `useBulkAddNewJob` only send `status: "In Queue"`, so this is not user-reachable, but the gap should be closed before any pre-assigned-insertion UI ships.

**Fix:** Either (a) implement Req 1.4 in the planner — emit a `returnTime` for an inserted job that carries a `proofedUserId`, computed per the anchor rules in the ticket; or (b) explicitly call this out as deferred to a follow-up ticket, document it in the PR description, and gate the unassigned-only contract in the schema (e.g. forbid `proofedUserId` on the insert).

---

### 2. `insertAfter` derivation diverges between the two bulk-hook paths

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/hooks/useBulkAddNewJob.ts]**

**Function/Class:** `handleBulkAddNewJob` vs `handleBulkAddMultipleJobs`

**Severity:** medium

**Problem:** The two bulk entry points compute `insertAfter` differently:

```typescript
// handleBulkAddNewJob, line ~262
insertAfter:
  analysis?.previousJob?.jobType === JobType.SERVICE
    ? analysis.previousJob.id
    : undefined,

// handleBulkAddMultipleJobs, line ~404
insertAfter: analysis.previousJob?.id,
```

The first restricts `insertAfter` to a SERVICE-type predecessor only; the second uses any predecessor. They are otherwise structurally identical request builders.

**Impact:** Inserting one job-type via the single-config bulk path vs. the multi-config path can place the new job at a different sequence position, so the planner's anchor (and the resulting hard stop) will differ. If both code paths are still wired up, this is a real behavioural inconsistency. It also makes future maintenance brittle.

**Fix:** Pick one rule and apply it in both places, or factor a `getInsertAfter(analysis)` helper. Looking at the analyzer's `previousJob` (`previousConfig?.jobTypeName ?? lastJob` fallback), the unguarded version (`analysis.previousJob?.id`) matches the server's expectation and the planner's behaviour — recommend dropping the `=== JobType.SERVICE` constraint unless there is a documented reason for it.

---

### 3. Anchor-divergence guard rejects legitimate multi-position inserts into the same order

**[File: apps/creative-portal/api/mixtures/jobs/addNewJobs/addNewJobs.ts]**

**Function/Class:** addNewJobs (lines 122-151)

**Severity:** low

**Problem:** The server hard-blocks any order whose batch entries target different `insertAfter` ids: "Multiple jobs inserted into one order must share the same insertion point." This guard exists because `planJobInsertionTiming` chains insert #2 onto insert #1 (the `chainAnchorId` mechanic, line 132), so non-collocated inserts would silently misplace later inserts.

The bulk hook's analyzer (`analyzeOrdersForJobInsertion`) computes `previousJob` per inserted config; for a Review+QA pair on an order with only `[Editing]` jobs, both fall back to `lastJob` = Editing, so the guard passes. But for an order that already has more downstream jobs (e.g., `[Editing, Polishing]`), Review's previousJob is Editing and QA's previousJob falls back to lastJob = Polishing → divergent ids → whole order rejected with a generic message.

**Impact:** Legitimate Review+QA bulk operations on non-canonical workflow shapes will fail with a confusing "must share the same insertion point" error. The user has no obvious recourse short of re-submitting one job at a time.

**Fix:** Either (a) lift the chaining limitation — extend the planner to splice each insert at its specified anchor independently, with downstream re-numbering; or (b) keep the guard but improve the message ("This order has a non-uniform workflow — please add Review and QA one at a time") and surface it through the per-order toast.

---

### 4. Planner reason text is discarded by the client toasts

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/hooks.tsx]**

**Function/Class:** `removeJobAndUpdateReturnTimes` (lines 322-333); `handleDeleteJobs` in `contexts/bulkOrderActionsContext/provider.tsx` (lines 618-622)

**Severity:** low

**Problem:** When `planJobRemovalTiming` returns `{ ok: false, reason: "..." }`, both call sites swallow `reason` and show a generic toast:

- Single-order skip: `showDefaultErrorToast()` (no message at all).
- Bulk skip: `"Couldn't skip the job on N order(s): a later job's committed return time would be exceeded. The remaining orders were skipped."` — fine when there's exactly one failure mode, but `planJobRemovalTiming` also returns `"Skipping would break job due-date ordering."` which is masked.

Insertion has the same shape: the bulk hook's `showBulkAddFailures` does grab the first `error` from the response, so that path is OK; the single-route 409 response carries the planner's reason in `error`, which the catch-block in `useAddNewJob` doesn't surface beyond invalidating the query.

**Impact:** Admins can't tell exactly why a skip or insertion failed — they see a generic error. Reduces troubleshooting clarity.

**Fix:** Pass `removal.reason` (and the 409 `error` in the single-add catch path) into the toast. Example:

```typescript
if (!removal.ok) {
  showToast({ type: "error", text: removal.reason });
  return;
}
```

---

### 5. Three new test files violate the project's prettier rule

**[File: apps/creative-portal/api/utils/jobs/buildJobPutBody.test.ts]**

**Function/Class:** Test file fixture builder (and the same pattern in `planJobInsertionTiming.test.ts:29`, `planJobRemovalTiming.test.ts:29`)

**Severity:** low

**Problem:** ESLint reports 3 `prettier/prettier` errors introduced by this PR. Each new test file uses the pattern `} as Job)` where prettier wants `}) as Job`:

```
api/utils/jobs/buildJobPutBody.test.ts:21:4
api/utils/jobs/planJobInsertionTiming.test.ts:29:4
api/utils/jobs/planJobRemovalTiming.test.ts:29:4
```

(The other 4 lint errors in PR-touched files — `OrderManagment/hooks.tsx:427-428,501` and `bulkOrderActionsContext/provider.tsx:912` — are pre-existing on the PP-1644 base branch per `git blame`, so not attributable to this PR. The broader repo-level lint failures in `packages/wysiwyg` are also pre-existing on the base.)

**Impact:** CLAUDE.md requires "0 lint errors" before commit. These are PR-introduced.

**Fix:** Run `yarn lint:fix` against the three test files, or apply the small formatting nudge manually:

```typescript
const buildJob = (overrides: Partial<Job> = {}): Job =>
  ({
    id: 1,
    // ...
    ...overrides
  }) as Job;
```

---

### 6. `OrderJobData.upcomingJobs` is now dead

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/hooks/useBulkAddNewJob.ts]**

**Function/Class:** `OrderJobData` (line 31) / `analyzeOrdersForJobInsertion` (lines 121-136)

**Severity:** low

**Problem:** The hook computes `upcomingJobs` from `upcomingJobConfigurations`/`upcomingJobTypes` and includes it in every `OrderJobData`. Pre-PP-1863, this was sent to the server in the bulk payload. Per the PR description ("the now-unused `upcomingJobs` / `serviceConfigurations` fields dropped from the wire"), the server now derives the downstream list from its own fresh `fetchJobsByOrderId`, so the client computation is dead weight.

**Impact:** Dead code on a hot path (recomputed on every `orders`/`jobs`/`serviceConfigurations` change). Minor — but defaults to drift over time.

**Fix:** Drop `upcomingJobs` from `OrderJobData` and the analyzer's `return` block; also drop `upcomingJobConfigurations` / `upcomingJobTypes` / `currentJobConfig` / `nextConfigIndex` if they have no other consumers.

---

### 7. `buildWorkflowTasksForInsertion` fan-out can misrepresent fixed-rate workflow rows

**[File: apps/creative-portal/api/utils/deliveryCalculation/buildWorkflowTasksForInsertion.ts]**

**Function/Class:** `buildWorkflowTasksForInsertion`

**Severity:** low

**Problem:** For non-inserted rows, the helper falls back to `workItemSize ?? service.minimumTaskQuantity` as the quantity. The own comment acknowledges this is "a pragmatic alignment that covers the common file-drop and CSV cases where every workflow row is sized off the document; truly fixed-rate rows (priced per unit, no document length) are not distinguished here."

**Impact:** For service configurations whose duration is independent of document length (per-unit pricing), the delivery-calc payload will overstate the quantity, which inflates returned `maxReturnWindowsMinutes` for *non-inserted* rows. The inserted row's window is unaffected (it overrides the quantity from `jobTaskData.length`), so the downstream shift driven by the inserted row stays correct. But if any code path consumes a non-inserted row's window from this same response, it would be wrong.

**Fix:** Today this is fine because the route only reads the inserted row's windows. Worth a follow-up to either (a) distinguish fixed-rate services in the response and use the proper minimum quantity, or (b) explicitly document this is "inserted-row only" and never read the other rows.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ✅ | 1139/1139 pass across 123 files (creative-portal + shared + wysiwyg + storybook). |
| `npx turbo run typecheck` | ✅ | 0 errors across all 5 workspaces. |
| `npx turbo run lint` | ❌ (PR-attributable: 3) | 3 prettier violations in new PR test files (Issue #5). 4 additional lint errors in PR-touched files are pre-existing on base (PP-1644). The wider repo lint failures (`packages/wysiwyg`, many other `creative-portal` files) are pre-existing on the PP-1644 base branch and not caused by this PR. |
| `npx turbo run build` | ✅ | Clean — `apps/creative-portal` builds, all routes generated. |

---

## Tests

- ✅ `planJobInsertionTiming.test.ts` — 11 cases including after-job anchor, downstream shift, returnTime preservation, QA two-insert compounding, start-of-sequence anchor, shuffled OMS order, mid-sequence id reordering, canceled jobs unaffected, and the three hard-block paths (no-anchor, unknown anchor, non-positive window).
- ✅ `planJobRemovalTiming.test.ts` — 11 cases covering re-chain, downstream collapse, Review+QA pair removal, shuffled-input reconstruction, canceled-job skip, first-job removal anchoring on `creationDatetime`, last-job removal, assigned-deadline conflict block, and empty-input no-op.
- ✅ `addJobWithTasks.test.ts` — PP-1827 duplicate-prevention + PP-1863 new cases (computed hard stop with no `returnTime`, downstream shift preserves committed `returnTime`, 409 on no-anchor) + PP-1641 delivery-calc lookup with distinct max/standard windows.
- ✅ `addNewJobs.test.ts` — single-insert + QA compounding + per-order partial-success + anchor-divergence guard + partial-write re-plan + per-order delivery-calc lookup.
- ✅ `buildJobPutBody.test.ts` — whole-record contract, `returnTime` omission for null/undefined, exclusion of non-PUT fields.
- ✅ `buildWorkflowTasksForInsertion.test.ts` — covers the workflow-row fan-out with `workItemSize` and `minimumTaskQuantity` fallbacks.
- ✅ `bulkJobDueDateShift.test.ts` — updated to validate complete-body JobPut shape (whole-record contract).
- ❌ No test exercises Req 1.4 (assigned inserted job's `returnTime`) — consistent with the gap in Issue #1.
- ⚠️ No UI test coverage for the new `DeadlineRow` partial, but it's a thin display component.
- ⚠️ Manual checklist in the PR description is **not yet checked off** ("Manual testing completed" — unchecked).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ (Req 1.4 unimplemented; bulk hook insertAfter inconsistency; anchor-divergence may reject legitimate workflows) |
| Regression risk | ⚠️ Medium (large server-side refactor of two POST routes; new whole-record PUT contract changes the shape of every consumer; partial-success bulk semantics) |
| Tests | ✅ Strong unit coverage on planners; route happy-path + key edge cases covered; manual checklist not yet completed |
| Code quality | ✅ Well-commented, well-typed; pure-function design for planners; thoughtful partial-success re-plan |
| Validation suite | ❌ Lint (3 PR-introduced prettier errors in new test files); test/typecheck/build all pass |
| Mergeable state | ⚠️ GitHub reports `clean`, but per CLAUDE.md the 3 PR-introduced lint errors must be fixed first; targets PP-1644 base, not develop (will auto-retarget once base lands) |

---

## Recommendation

**Request changes** — none of the issues are critical correctness defects on the unassigned-insert path the UI actually exercises today, but the PR cannot land cleanly under CLAUDE.md's "0 lint errors" rule and the Req 1.4 gap should be acknowledged in writing before merge.

1. **Fix the 3 prettier violations** in the new test files (Issue #5). One-line nudge each, easy.
2. **Address Req 1.4** (Issue #1) — either implement the assigned-insert `returnTime` calculation in the planner or call out explicitly in the PR description that this is deferred to a follow-up ticket, and tighten the schema to reject `proofedUserId` on the insert until that ships.
3. **Reconcile `insertAfter` between the two bulk hook entry points** (Issue #2) — pick one rule and apply it uniformly.
4. **Improve error surfacing** for both skip and single-add 409 paths so the planner's `reason` reaches the user (Issue #4).
5. **Complete the manual checklist** in the PR description (POST/PUT payload screenshots) before merge — server-side logic that touches OMS contracts is worth proving on a real environment.
6. **Either rename the PR / split out** the `planJobRemovalTiming` work (skip reflow) into its own ticket, or expand PP-1863's scope to include it explicitly — right now it's bundled silently against a ticket whose AC says nothing about removal.
7. Optionally address Issues #3, #6, #7 in follow-ups; they aren't blockers for this PR's primary objective.
