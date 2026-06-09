# PR Review: PP-1640: dynamic job return time in customer-portal order creation

**PR:** https://github.com/Proofed/B2BWebserver/pull/2292
**Jira:** https://proofed.atlassian.net/browse/PP-1640
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1.1 Delivery Calculation response exposes `returnWindowsMinutes` + `maxReturnWindowsMinutes` per job; both retrieved | Added to `DeliveryCalculationType.jobs` and `CreateOrderAndJobsProps.deadlineWithOrderedJobs.jobs`; flow through `getDeadline` → `getDeadlineWithOrderedJobs` untouched (no reshaping) | ✅ Addressed |
| 2.1 `Job[1].maxReturnTime = orderStartTime + Job[1].maxReturnWindowsMinutes` | `calculateJobsReturnTime` seeds `previousMaxReturn = orderStartTime`, then `addMinutes(previousMaxReturn, maxReturnWindowsMinutes)` | ✅ Addressed |
| 2.2 `Job[N].maxReturnTime = Job[N-1].maxReturnTime + Job[N].maxReturnWindowsMinutes` | Recurrence carries `previousMaxReturn = maxReturn` each iteration | ✅ Addressed |
| 3.1 Each Job POST includes `returnWindowsMinutes` + `maxReturnTime` | `addJob` body now `{ orderId, returnWindowsMinutes, maxReturnTime }` | ⚠️ Partial — field-name caveat (see Issue 1) |
| 3.2 `returnTime` must not be sent | Removed from `AddJobBE` and the `addJob` call; old accumulator deleted | ✅ Addressed |
| 4.1 Every job has non-null `returnWindowsMinutes` + `maxReturnTime` | `isMissingWindow` guards both window inputs, throws `JobTimingValidationError` | ✅ Addressed |
| 4.2 Sequential integrity `Job[N].maxReturnTime > Job[N-1].maxReturnTime` | `maxReturn.getTime() <= previousMaxReturn.getTime()` throws | ✅ Addressed |
| 4.3 `Job[final].maxReturnTime <= Order.dueDateTime` | Final compare `finalMaxReturn > dueDate` throws | ✅ Addressed |
| 4.4 On failure, order not submitted + conflict surfaced to customer | Validation runs in `createOrder` (parent) before any side effect; `missingTiming` pre-flight throws before the order POST; `statusCode: 400` surfaces via `handleEndpointError` on both the JSON and streamed endpoints | ✅ Addressed |

**Scope beyond the ticket:** `safeCreateOrderAndJobs` was deleted (88 lines). It has no callers anywhere in the codebase, so removal is safe cleanup — and arguably forced, since its signature would no longer compile without the new required `jobTimings` prop. Worth a one-line mention in the PR description.

---

## Architecture Analysis

The change cleanly extracts the timing logic into a pure, well-tested shared util (`packages/shared/utils/calculateJobsTime/`) and splits responsibilities correctly:

- **`createOrder.ts` (parent orchestrator)** computes `orderStartTime` (group anchor vs. `now`) and calls `calculateJobsReturnTime` **before** `createOrderAndJobs`. This is the key correctness property the ticket asks for: validation throws before any side-effecting call, so a timing-rule failure never produces an orphan order.
- **`createOrderAndJobs.ts`** receives pre-computed `jobTimings`, does a defensive `missingTiming` pre-flight (still before the order POST), then attaches each job's `returnWindowsMinutes` + `maxReturnTime` by `jobTypeId` lookup.
- **`JobTimingValidationError`** carries `statusCode = 400`; `handleEndpointError` reads `(error as ApiError).statusCode`, so both the JSON route (`createOrderEndpoint`) and the streamed route (`createStreamEndpoint`, the file-upload path) return a 4xx.

The recurrence matches the spec exactly, and the boundary conditions (`<=` for strict-increasing, `>` for due-date) are correct. The unit test suite (217 lines) is thorough and covers single/multi-job, each validation rule, the grouped anchor, and the `statusCode`.

The main residual risks are not in the pure helper but in the **data it is fed**: the relationship between `deadlineWithOrderedJobs.jobs` and `jobConfigurationsWithServices`, the workflow ordering, and the real-world magnitude of `maxReturnWindowsMinutes` vs. the order deadline.

---

## Issues Found

### 1. Job POST field name may not match the backend contract

**[File: apps/customer-portal/api/utils/jobs/addJob.ts]**
**Function/Class:** AddJobBE / addJob
**Severity:** medium
**Problem:** The payload sends `returnWindowsMinutes` (plural "Windows"). The Jira **requirement text** also uses `returnWindowsMinutes`, but the Jira **API spec (§6.8 CLIENT PORTAL JOB CREATION POST)** code block shows the field as `returnWindowMinutes` (singular "Window"):

```
{
  "orderId": 123,
  "returnWindowMinutes": 120,
  "maxReturnTime": "2026-05-09T14:20:00Z"
}
```

**Impact:** `prepareServiceAxiosConfigWithData` forwards the body keys verbatim. If the backend expects `returnWindowMinutes`, the standard window is silently dropped (the field is ignored), and the job persists without it — a silent data-loss bug that unit tests can't catch because they only exercise the pure helper, not the wire payload.
**Fix:** Confirm the exact field name against the live `POST /api/jobs` contract (or the delivery-calc service that owns the field). The delivery-calc response field is `returnWindowsMinutes`, which is good corroboration, but the spec's singular form should be reconciled before merge. Verify in DevTools that the persisted job carries the window value.

### 2. New accumulation walks the full delivery-calc job list, not the persisted set

**[File: apps/customer-portal/api/utils/mixtures/orders/createOrder/createOrder.ts]**
**Function/Class:** createOrder → calculateJobsReturnTime
**Severity:** medium
**Problem:** `calculateJobsReturnTime` accumulates over **every** entry in `deadlineWithOrderedJobs.jobs`. The previous code accumulated `currentJob?.duration` only for job types that had a matching `jobConfigurationsWithServices` entry (it iterated job configs and `find`-ed the job). If the delivery-calc response ever returns a job type that has no corresponding job config (the `sortByJobsInOrder` map explicitly handles `Return`/`Other` types, hinting this is possible), that phantom job now:
- contributes its `maxReturnWindowsMinutes` to the cumulative total, inflating every later `maxReturnTime` and the final due-date check, yet
- is never persisted (the `addJob` loop iterates `jobConfigurationsWithServices`, and the `missingTiming` check only flags *configs without timings*, never *timings without configs*).

**Impact:** If `jobs` ⊋ `jobConfigurationsWithServices`, valid orders can be 400-blocked by phantom accumulation, or persisted jobs receive a `maxReturnTime` that bakes in a job the customer isn't being charged for. This is a behavioral change from the old per-config accumulation.
**Fix:** Confirm `deadlineWithOrderedJobs.jobs` is guaranteed 1:1 (by `jobTypeId`) with `jobConfigurationsWithServices`. If not guaranteed, filter the jobs fed to the recurrence to the configured set, e.g.:

```typescript
const configuredJobTypeIds = new Set(
  jobConfigurationsWithServices.map((config) => config.jobTypeId)
);
const jobTimings = calculateJobsReturnTime({
  jobs: deadlineWithOrderedJobs.jobs.filter((job) =>
    configuredJobTypeIds.has(job.jobTypeId)
  ),
  orderDueDateTime: deadlineWithOrderedJobs.deadline,
  orderStartTime
});
```

### 3. Validation feasibility for grouped orders is unverified against real data

**[File: packages/shared/utils/calculateJobsTime/calculateJobsReturnTime.ts]**
**Function/Class:** calculateJobsReturnTime (final due-date check)
**Severity:** medium
**Problem:** For grouped orders, `orderStartTime = deadline − totalOrderDuration` (no slack). The final-job rule then requires `Σ maxReturnWindowsMinutes ≤ totalOrderDuration`. If `maxReturnWindowsMinutes` is, by design, a *hard-stop window that exceeds the per-job standard duration*, the sum will exceed `totalOrderDuration` and **every grouped order is rejected with a 400**. Non-grouped orders have some slack (the deadline includes weekend/business-hour padding beyond `now + totalOrderDuration`), but the same tension exists if the max windows are large.
**Impact:** Over-strict validation could block legitimate order creation in production — a customer-facing 400 on the happy path. This can't be caught by the current unit tests, which use hand-picked window values.
**Fix:** Manually verify against real delivery-calc responses (especially a multi-job grouped order) that legitimate orders pass. Confirm the intended semantics: is `maxReturnWindowsMinutes` ≤ the job's contribution to `totalOrderDuration`? Document the expected relationship in a code comment so the invariant is explicit.

### 4. New branching logic has no integration test coverage

**[File: apps/customer-portal/api/utils/mixtures/orders/createOrder/helpers/createOrderAndJobs.ts]**
**Function/Class:** createOrderAndJobs (missingTiming pre-flight) / createOrder (orderStartTime)
**Severity:** low
**Problem:** The pure helper is excellently covered, but the new glue is not: (a) the `missingTiming` throw path, (b) the grouped-vs-non-grouped `orderStartTime` computation in `createOrder.ts`, and (c) the `addJob` payload reshaping. CLAUDE.md requires tests for new code.
**Impact:** A regression in the wiring (e.g., wrong anchor for grouped orders, or a `jobTypeId` mismatch) would slip through. Issues 1–3 above are precisely the kind of thing an integration test would surface.
**Fix:** Add a test for `createOrderAndJobs` that (1) asserts `addJob` is called with `{ returnWindowsMinutes, maxReturnTime }` and no `returnTime`, and (2) asserts `JobTimingValidationError` is thrown — and no `addOrder`/`addJob` is called — when a config has no timing. A small test on `createOrder.ts` covering the grouped anchor would close the gap on Issue 3.

### 5. `isMissingWindow` type predicate is inaccurate for `NaN`

**[File: packages/shared/utils/calculateJobsTime/calculateJobsReturnTime.ts]**
**Function/Class:** isMissingWindow
**Severity:** low
**Problem:** The signature is `value is null | undefined`, but the body returns `true` for `Number.isNaN(value)` as well — and `NaN` is typed `number`, not `null | undefined`. The predicate narrows incorrectly. It happens to be harmless here because the guard only gates a `throw`, so the narrowed type is never used downstream.
**Impact:** Cosmetic / future-maintenance footgun. No runtime effect today.
**Fix:** Drop the type predicate and return a plain `boolean`, or rename to reflect that it also rejects `NaN`:

```typescript
const isMissingWindow = (value: number | null | undefined): boolean =>
  value == null || Number.isNaN(value);
```

### 6. Negative `returnWindowsMinutes` is not rejected

**[File: packages/shared/utils/calculateJobsTime/calculateJobsReturnTime.ts]**
**Function/Class:** calculateJobsReturnTime
**Severity:** low
**Problem:** A negative `maxReturnWindowsMinutes` is implicitly caught by the strict-increasing check (it would move `maxReturn` backward). But a negative `returnWindowsMinutes` (the standard window, not used in the recurrence) passes `isMissingWindow` and is forwarded to the backend unchanged.
**Impact:** Edge case; only reachable if the delivery-calc service returns a negative standard window. Low likelihood.
**Fix:** Optionally treat `value < 0` as invalid in the window check, or rely on backend validation. Acceptable to defer.

---

## Tests

- ✅ Pure helper `calculateJobsReturnTime` has 217 lines of unit tests: single-job, multi-job accumulation + strict-increasing assertion, missing `returnWindowsMinutes`, missing `maxReturnWindowsMinutes`, first-job zero window, non-strictly-increasing, final > dueDateTime, empty jobs, custom grouped anchor, and `statusCode === 400`.
- ✅ Each Jira validation rule (4.1–4.3) maps to a dedicated failing test.
- ❌ No test for the `missingTiming` pre-flight throw in `createOrderAndJobs`.
- ❌ No test for the `addJob` payload shape change (`returnWindowsMinutes`/`maxReturnTime` present, `returnTime` absent).
- ❌ No test for `createOrder.ts` grouped vs. non-grouped `orderStartTime` anchor.
- ⚠️ Manual test plan needed: real delivery-calc data for a grouped multi-job order (Issue 3), and a DevTools check of the `POST /api/jobs` body field name (Issue 1).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Helper is correct; data-feed assumptions (Issues 2 & 3) need confirmation |
| Regression risk | ⚠️ Medium — accumulation semantics changed; `safeCreateOrderAndJobs` removal is safe (no callers) |
| Tests | ⚠️ Pure helper excellent; integration glue untested |
| Code quality | ✅ Clean extraction, good error typing, pre-flight ordering correct |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions** — the core logic is well-structured and the unit tests are strong. Resolve these before merge:

1. **Confirm the `POST /api/jobs` field name** (`returnWindowsMinutes` vs `returnWindowMinutes`) against the live backend and verify in DevTools (Issue 1).
2. **Confirm `deadlineWithOrderedJobs.jobs` is 1:1 with `jobConfigurationsWithServices`**, or filter the recurrence input to the configured job types (Issue 2).
3. **Manually validate a grouped multi-job order** end-to-end to confirm the due-date rule doesn't block legitimate orders (Issue 3).
4. **Add integration tests** for the `missingTiming` throw and the `addJob` payload shape (Issue 4).
5. Minor: tidy the `isMissingWindow` predicate (Issue 5); note the `safeCreateOrderAndJobs` removal in the PR description.
