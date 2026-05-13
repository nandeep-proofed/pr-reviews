# PR Review: PP-1640: dynamic job return time in customer-portal order creation

**PR:** https://github.com/Proofed/B2BWebserver/pull/2292
**Jira:** https://proofed.atlassian.net/browse/PP-1640
**Status:** In Progress

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Delivery Calculation response includes `returnWindowsMinutes` and `maxReturnWindowsMinutes` per job; both must be retrieved | Both fields added to `DeliveryCalculationType` and `CreateOrderAndJobsProps` | ✅ Addressed |
| `Job[1].maxReturnTime = orderStartTime + Job[1].maxReturnWindowsMinutes` | `computeJobReturnTimings` sets `previousMaxReturn = orderStartTime`, first job: `addMinutes(orderStartTime, maxReturnWindowsMinutes)` | ✅ Addressed |
| `Job[N].maxReturnTime = Job[N-1].maxReturnTime + Job[N].maxReturnWindowsMinutes` | Accumulator pattern in `computeJobReturnTimings` loop | ✅ Addressed |
| Job POST must include `returnWindowsMinutes` and `maxReturnTime` | `addJob` body updated to send both fields | ✅ Addressed |
| `returnTime` must not be sent | `returnTime` removed from `AddJobBE` and POST body | ✅ Addressed |
| Validation: every job has non-null window fields | `isMissingWindow` guard throws `JobTimingValidationError` | ✅ Addressed |
| Validation: `Job[N].maxReturnTime > Job[N-1].maxReturnTime` for consecutive jobs | Monotonicity check applies to all jobs including the first | ✅ Addressed |
| Validation: `Job[final].maxReturnTime <= Order.dueDateTime` | Post-loop check; throws on violation | ✅ Addressed |
| Validation failures must not produce an orphan order | `computeJobReturnTimings` + pre-flight cross-check both run before `addOrder` | ✅ Addressed |
| Failures throw with `statusCode: 400` surfaced to client | `JobTimingValidationError` has `statusCode: 400`; `handleEndpointError` matches via duck-typed `statusCode` property | ✅ Addressed |
| Grouped order start anchor = group deadline − total duration | `subMinutes(deadline, totalOrderDuration)` in `createOrder.ts` | ✅ Addressed |
| Non-grouped order anchored to "now" | `orderStartTime = new Date()` for non-grouped path | ✅ Addressed |

---

## Architecture Analysis

The approach cleanly separates timing computation into a pure, side-effect-free `computeJobReturnTimings` helper that runs before any I/O in `createOrder.ts`. A custom `JobTimingValidationError` carries `statusCode: 400`, which maps naturally into the existing `handleEndpointError` utility via its duck-typed `statusCode` check. `createOrderAndJobs` is slimmed down — it receives pre-computed `jobTimings` instead of re-deriving them mid-loop.

A pre-flight cross-check now runs before `addOrder` to confirm every `jobConfig.jobTypeId` has a matching timing entry, making the orphan-order scenario structurally impossible. The monotonicity invariant in `computeJobReturnTimings` applies from the first job (no `index > 0` gate). Dead code (`safeCreateOrderAndJobs`) has been removed.

---

## Issues Found

No open issues. All review findings have been resolved.

---

## Resolved Issues (all rounds)

- ✅ `/* eslint-disable @typescript-eslint/no-explicit-any */` removed from `jobTimingValidationError.ts`
- ✅ Constructor parameter updated from `any` to `Record<string, unknown> | null`
- ✅ Defensive `Missing job timing` throw changed from `new Error(...)` to `new JobTimingValidationError(...)` — surfaces as 400 via `handleEndpointError`
- ✅ **Orphan order risk fixed** — pre-flight cross-check added in `createOrderAndJobs.ts` before `addOrder`; the in-loop guard was replaced with a non-null assertion backed by the pre-flight guarantee
- ✅ **First-job monotonicity fixed** — `index > 0 &&` gate removed from `computeJobReturnTimings`; invariant now applies from the first job
- ✅ **Dead code removed** — `safeCreateOrderAndJobs` deleted entirely
- ✅ **New test added** — `"throws when the first job's maxReturnWindowsMinutes is 0"` covers the newly enforced first-job invariant (9 tests total, all passing)

---

## Tests

- ✅ `computeJobReturnTimings` — 9 unit tests, all passing (including new first-job zero-window case)
- ✅ Single-job, multi-job accumulation, grouped-order start anchor, all validation error paths covered
- ✅ `statusCode: 400` on `JobTimingValidationError` verified
- ✅ All 366 customer-portal tests pass (`36 test files`)
- ⚠️ PR checklist items (manual testing, E2E, security review) still need to be completed before merge

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Correct |
| Regression risk | ✅ Low — orphan-order scenario structurally eliminated |
| Tests | ✅ All passing; new case added |
| Code quality | ✅ Clean, well-structured, follows project conventions |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions**

1. **Complete the PR checklist** — manual testing, E2E, and security review items are still unchecked.
2. **Fix the Jira spec typo** — the JSON example in §6.8 shows `returnWindowMinutes` (no `s`); update it to `returnWindowsMinutes` to match requirements text, context, and testing notes.
