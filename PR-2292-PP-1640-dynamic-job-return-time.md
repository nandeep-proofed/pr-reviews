# PR Review: PP-1640: dynamic job return time in customer-portal order creation

**PR:** https://github.com/Proofed/B2BWebserver/pull/2292
**Jira:** https://proofed.atlassian.net/browse/PP-1640
**Status:** Draft

> Note: The Jira URL in the PR description is broken (`https://proofed.atlassian.net/browse/PP-` — ticket number missing). Jira ticket was not fetchable for requirements cross-check; review is based on the PR description and code alone.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Replace fixed per-job returnTime accumulator with delivery-calculation-sourced windows | Old `accumulatedDuration + addMinutes` loop removed; `computeJobReturnTimings` uses `maxReturnWindowsMinutes` per job | ✅ Addressed |
| Compute `returnWindowsMinutes` + `maxReturnTime` from delivery-calculation response | `DeliveryCalculationType.jobs` extended with both fields; used in new helper | ✅ Addressed |
| Anchor grouped orders to group start time (deadline − totalOrderDuration) | `orderStartTime` computed in `createOrder.ts` using `subMinutes` for grouped orders | ✅ Addressed |
| Anchor non-grouped orders to "now" | `orderStartTime = new Date()` for non-grouped path | ✅ Addressed |
| Validation before any side-effecting call (no orphan orders) | `computeJobReturnTimings` called before `createOrderAndJobs`; throws pre-submission | ✅ Addressed |
| Each job must carry both window fields | Checked via `isMissingWindow`; throws `JobTimingValidationError` on missing fields | ✅ Addressed |
| `maxReturnTime` sequence must be strictly increasing | Enforced per-job with `<= previousMaxReturn` check (index > 0) | ✅ Addressed |
| Final `maxReturnTime` must not exceed `orderDueDateTime` | Post-loop check; throws on violation | ✅ Addressed |
| Failures throw `JobTimingValidationError` with `statusCode 400` | `handleEndpointError` checks `(error as ApiError).statusCode` — 400 surfaces correctly | ✅ Addressed |

---

## Architecture Analysis

The PR introduces a clean separation: validation logic lives in a pure, side-effect-free `computeJobReturnTimings` helper that is called in `createOrder.ts` before any I/O, so a timing failure never produces an orphan order. The `createOrderAndJobs` function is slimmed down — it now receives pre-computed `jobTimings` instead of re-deriving them mid-loop. `JobTimingValidationError` carries `statusCode = 400` which maps naturally into the existing `handleEndpointError` utility (which checks the `statusCode` duck-type property). The approach is consistent with the rest of the codebase.

---

## Issues Found

### 1. `eslint-disable @typescript-eslint/no-explicit-any` violates project convention

**[File: apps/customer-portal/api/utils/mixtures/orders/createOrder/helpers/jobTimingValidationError.ts]**
**Function/Class:** `JobTimingValidationError`
**Severity:** medium
**Problem:** The file starts with `/* eslint-disable @typescript-eslint/no-explicit-any */` and uses `any` for `details` and the constructor parameter. The project memory and CLAUDE.md explicitly prohibit using eslint-disable for `no-explicit-any` — use proper types instead.
**Impact:** Bypasses the lint rule in a new file, sets a bad precedent, and loses type safety on the `details` field.
**Fix:** Type `details` as `Record<string, unknown> | null`:

```typescript
export class JobTimingValidationError extends Error {
  details: Record<string, unknown> | null;

  readonly statusCode: number = 400;

  constructor(
    message: string,
    details: Record<string, unknown> | null = null
  ) {
    super(message);
    this.name = "JobTimingValidationError";
    this.details = details;
    Object.setPrototypeOf(this, JobTimingValidationError.prototype);
  }
}
```

The three call sites that pass `{ index, jobTypeId }` or `{ finalMaxReturnTime, orderDueDateTime }` already satisfy `Record<string, unknown>`, so no other change is needed.

---

### 2. Defensive `Missing job timing` throw uses generic `Error` (500) instead of `JobTimingValidationError` (400)

**[File: apps/customer-portal/api/utils/mixtures/orders/createOrder/helpers/createOrderAndJobs.ts]**
**Function/Class:** `createOrderAndJobs`
**Severity:** medium
**Problem:** The new defensive guard throws a plain `Error`, not a `JobTimingValidationError`:

```typescript
if (!timing) {
  throw new Error(
    `Missing job timing for jobTypeId=${jobConfig.jobTypeId}`
  );
}
```

`handleEndpointError` does not match `Error` instances as 4xx — they fall through to the `error instanceof Error` branch, which returns a 500 with `"Unhandled exception occurred: ..."`.
**Impact:** A mismatch between `jobConfigurationsWithServices` and `jobTimings` (e.g., a bug introduced by future refactoring) would return a 500 instead of a descriptive 400 to the client. The PR description states all failures should surface as 4xx.
**Fix:**

```typescript
if (!timing) {
  throw new JobTimingValidationError(
    `Missing job timing for jobTypeId=${jobConfig.jobTypeId}`,
    { jobTypeId: jobConfig.jobTypeId }
  );
}
```

---

### 3. `safeCreateOrderAndJobs` is dead code

**[File: apps/customer-portal/api/utils/mixtures/orders/createOrder/helpers/createOrderAndJobs.ts]**
**Function/Class:** `safeCreateOrderAndJobs`
**Severity:** low
**Problem:** `safeCreateOrderAndJobs` is exported but never imported anywhere else in the codebase (confirmed by codebase-wide grep). The PR updates its signature to accept `jobTimings`, which is correct work, but the function is never called — making the change a no-op at runtime.
**Impact:** Dead code that increases maintenance surface. If this function is intended for a future batch-upload path, a comment noting that intent would be helpful; otherwise it should be deleted.
**Fix:** Either delete `safeCreateOrderAndJobs` and its type-level `jobTimings` additions, or add a comment explaining why it is retained.

---

### 4. Broken Jira URL in PR description

**[File: PR description]**
**Severity:** low
**Problem:** The JIRA link reads `https://proofed.atlassian.net/browse/PP-` — the ticket number is missing.
**Impact:** Reviewers cannot navigate to the ticket from the PR.
**Fix:** Update the link to `https://proofed.atlassian.net/browse/PP-1640`.

---

## Tests

- ✅ New pure helper `computeJobReturnTimings` has dedicated test file with 8 test cases
- ✅ Single-job, multi-job accumulation, grouped-order anchor, and all validation error paths are covered
- ✅ `statusCode = 400` on `JobTimingValidationError` is verified
- ❌ No test for the `Missing job timing for jobTypeId` guard in `createOrderAndJobs`
- ❌ No test for `createOrder.ts` integration (orderStartTime computation for grouped vs non-grouped) — acceptable given the pure helper is already covered
- ⚠️ All PR checklist items are unchecked (draft PR — expected, but must be completed before merge)

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Core logic is correct |
| Regression risk | ✅ Low — validation is pre-submission; old accumulation logic fully removed |
| Tests | ⚠️ Helper well-tested; `createOrderAndJobs` guard untested |
| Code quality | ⚠️ Two fixable issues (`any` type, wrong error class) |
| Mergeable state | ❌ Draft — checklists incomplete |

---

## Recommendation

**Request changes**

1. Replace `eslint-disable @typescript-eslint/no-explicit-any` with a proper `Record<string, unknown> | null` type in `jobTimingValidationError.ts`.
2. Change the `Missing job timing` defensive throw to use `JobTimingValidationError` so it surfaces as 400, consistent with the rest of the error strategy.
3. Either delete `safeCreateOrderAndJobs` (dead code) or add a comment explaining its intended future use.
4. Fix the broken Jira URL in the PR description.
5. Complete all PR checklist items before requesting final review.
