# PR Review: feature/PP-1641: Dynamic Job Return Time in Creative Portal Order Creation

**PR:** https://github.com/Proofed/B2BWebserver/pull/2296
**Jira:** https://proofed.atlassian.net/browse/PP-1641
**Status:** Issues #1-#7 resolved on local branch; awaiting commit + push
**Last verified against code:** 2026-06-08 (branch `feature/PP-1641-dynamic-job-return-time`)

---

## Verification Summary (2026-06-08, post-fix)

| # | Issue | Severity | Status | Verification |
|---|---|---|---|---|
| 1 | `mergeJobOverrides` doesn't advance anchor on rejected `maxReturnTime` override | medium | ✅ RESOLVED | Anchor advanced by delivery-calc default in rejection branch + multi-job test added |
| 2 | AC 5.4 invariant `returnTime <= maxReturnTime` not asserted | low | ✅ RESOLVED | 15 parameterised invariant tests added across all three anchor sources |
| 3 | `WorkflowWindowInputRowProps.jobType` is dead | low | ✅ RESOLVED | Dropped from types, hook signature, component destructure, and call site |
| 4 | `useWorkflowWindowInputRow` exports unused values | low | ✅ RESOLVED | Dropped 5 unused returns + their backing memos (`isPreAssigned`, asides, deltas); ~42 lines deleted |
| 5 | `CreateOrderNewPayloadType.order.jobs` is `Record<string, any>` | low | ✅ RESOLVED | Replaced with explicit `CreateOrderJobPayload` interface; removed eslint-disable + 1 `@ts-ignore`; resolved 3 latent null-safety issues |
| 6 | Inner loop iterates `deliveryCalculation.jobs` not `mergedDeliveryJobs` | low | ✅ RESOLVED | Swapped iterator to `mergedDeliveryJobs` (utils.ts:534) |
| 7 | No integration test for pre-assigned payload wiring | low | ✅ RESOLVED | New `jobTimings.test.ts` — 4 cases (all-unassigned, first-pre-assigned, mixed-sequence, OFFERED) |
| 8 | `handleDateChange` only reports first order's buffer | low | ⏭️ DEFERRED | Pre-existing UX — needs design call |
| 9 | Schema's `window` field still `Yup.number().required()` | informational | ⏭️ DEFERRED | Tracked migration debt for PP-1642 follow-up |

**Verification commands run locally:**

- `npx turbo run typecheck` — ✅ 0 errors
- `npx turbo run test` — ✅ 1,030 tests pass (including 16 new tests across 3 helper test files + 4 new integration cases)
- `npx turbo run build` — ✅ 0 errors
- `npx turbo run lint` (touched files only) — ✅ 0 errors. (Pre-existing 135 prettier errors on files unchanged from `develop` flagged; not introduced by this PR.)

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1.1 Consume `returnWindowsMinutes` + `maxReturnWindowsMinutes` from Delivery Calculation per job | `DeliveryCalculationType.jobs` extended with both fields (`api/deliveryCalculation/types.ts`); read by `getWindowsTime` and the new effect in `contexts/createOrder/provider.tsx` | ✅ Addressed |
| 2.1 First job: `maxReturnTime = orderStartTime + Job[1].maxReturnWindowsMinutes` | `computeJobReturnTimings` seeds `previousMaxReturn = orderStartTime` and adds `maxReturnWindowsMinutes` (`helpers/computeJobReturnTimings.ts:35,49`) | ✅ Addressed |
| 2.2 Subsequent job: `maxReturnTime = Job[N-1].maxReturnTime + Job[N].maxReturnWindowsMinutes` | Same forEach in `computeJobReturnTimings` chains `previousMaxReturn` | ✅ Addressed |
| 3.1 First pre-assigned: `returnTime = MIN(orderStartTime + returnWindowsMinutes, maxReturnTime)` | `computePreAssignedReturnTime` uses `orderStartTime` when no previous job set (`helpers/computePreAssignedReturnTime.ts:18-27`) and caps with `min([projected, maxReturn])` | ✅ Addressed |
| 3.2 Subsequent pre-assigned w/ previous returnTime: anchor on `previousJob.returnTime` | Helper prefers `previousJobReturnTime` when supplied; caller in `utils.ts:604-611` flips the anchor each iteration | ✅ Addressed |
| 3.3 Subsequent pre-assigned w/ only previous `maxReturnTime`: anchor on `previousJob.maxReturnTime` | Helper falls through to `previousJobMaxReturnTime` when no returnTime; caller resets the right anchor each iteration | ✅ Addressed |
| 4.1 Every job POST includes `returnWindowsMinutes` and `maxReturnTime` | `jobPayload` always sets both (`utils.ts:570-584`); `CreateJob` type allows them (`api/jobs/types.ts:107-115`) | ✅ Addressed |
| 4.2 When `proofedUserId` is provided, payload also includes `returnTime` | `computePreAssignedReturnTime` only fires when `isPreAssigned`, then spread into payload via `...(returnTime ? { returnTime } : {})` | ✅ Addressed |
| 4.3 When `proofedUserId` not provided, `returnTime` is omitted | `returnTime` is `undefined` for non-pre-assigned jobs and the spread guards against it | ✅ Addressed |
| 5.1 Pre-flight: every job has non-null `returnWindowsMinutes` and `maxReturnTime` | `computeJobReturnTimings` throws `JobTimingValidationError` when missing (`helpers/computeJobReturnTimings.ts:39-47`); additional pre-flight in `utils.ts:276-288` for missing job configurations | ✅ Addressed |
| 5.2 Sequential integrity: `Job[N].maxReturnTime > Job[N-1].maxReturnTime` | Helper rejects non-strictly-increasing maxReturnTime (`helpers/computeJobReturnTimings.ts:54-59`) | ✅ Addressed |
| 5.3 Order hard stop: `Job[final].maxReturnTime <= Order.dueDateTime` | Helper verifies final maxReturnTime against `orderDueDateTime` (`helpers/computeJobReturnTimings.ts:71-84`) | ✅ Addressed |
| 5.4 `returnTime <= maxReturnTime` when supplied | Enforced by construction via `min([projected, maxReturn])` in `computePreAssignedReturnTime`; no explicit invariant check or test | ⚠️ Partial (by construction; not asserted) |
| 5.5 Validation runs before order is created (no orphans) | Both `mergeJobOverrides` + `computeJobReturnTimings` + missing-timing check execute BEFORE `addOrder` (`utils.ts:244-288`) | ✅ Addressed |

### Beyond Jira scope (acknowledged in PR description but worth flagging)

The PR description scopes the change to API/order-creation, but ~80% of the diff is UI: the `WorkflowWindowInput` rewrite (262/156 lines), the new `WorkflowWindowInputRow` hook + component, schema additions for the form, the `useDeadlineManagement` hook rewrite (buffer math), and the cascading-chain provider effect. These belong to the spec ("workflows are flexible for unassigned jobs... commitments are locked in for pre-assigned ones") but are not separately broken out as PRs even though PP-1642 and PP-1643 carve out sidebar/job-card UI. Reviewing the form mutation paths is in scope here because they feed `createOrderData.order.jobs`.

---

## Architecture Analysis

The core API change is clean: order-creation now produces `(returnWindowsMinutes, maxReturnTime)` for every job and an effective `returnTime` only for pre-assigned ones, computed by two small pure helpers (`computeJobReturnTimings`, `computePreAssignedReturnTime`) plus a new error class. Pre-flight validation runs before `addOrder`, so a failed timing check no longer creates an orphan order. The PR ports the helpers from PP-1640 (customer portal) and re-exports them under `apps/creative-portal/utils/jobTimings/` so future consumers don't reach into `api/orders/createNew/helpers/`.

The UI side stores **deltas only** (`returnWindowsMinutes` and `maxReturnWindowsMinutes`) in Formik and derives absolute dates live from a per-render chain anchored on the current minute. The `WorkflowWindowInputRow` hook walks the chain, skipping SKIPPED jobs, so editing a Job Due Date on row N automatically cascades to N+1+, etc., without any explicit "shift downstream" code. Order-level Buffer is `Deadline − lastJob.maxReturnTime` (or falls back to `Deadline − now`).

There is an intentional cross-layer split: the UI's chain uses zoned `now` for visual accuracy; the backend's `computeJobReturnTimings` uses UTC `new Date()` at submit time. The deltas are identical, so the final stored maxReturnTime can drift by a few minutes if the admin lingers — backend rejects if this pushes Job[final].maxReturnTime past the deadline. This matches spec rule 5.3.

A new `mergeJobOverrides` helper bridges admin-edited deltas (the form) with the delivery-calc job schedule (the API), preferring `maxReturnWindowsMinutes` overrides, falling back to absolute `maxReturnTime` for legacy callers, and finally to the delivery-calc default.

---

## Issues Found

### 1. `mergeJobOverrides` does not advance the running anchor when a `maxReturnTime` override yields a non-positive delta

**[File: apps/creative-portal/api/orders/createNew/helpers/mergeJobOverrides.ts]**
**Function/Class:** mergeJobOverrides
**Severity:** medium
**Verified Status:** ✅ RESOLVED — `mergeJobOverrides.ts` now advances `runningMaxReturn` by the delivery-calc `maxReturnWindowsMinutes` default when a `maxReturnTime` override is rejected (non-positive delta). Multi-job rejection regression test added in `mergeJobOverrides.test.ts` (12 tests total, up from 11).
**Problem:** Inside the per-job loop, the `else if (override?.maxReturnTime)` branch only advances `runningMaxReturn` when `overriddenWindow > 0`. If the picked `maxReturnTime` equals or precedes the running anchor (overriddenWindow ≤ 0), the override is silently dropped, the job's `maxReturnWindowsMinutes` falls back to the delivery-calc default — but the running anchor is **not advanced** by that default either, because the branches are mutually exclusive (`if … else if … else if`).
**Impact:** Downstream chain corruption. If Job 1 has a bad `maxReturnTime` override (e.g., picker UX or stale form data) and Job 2 exists, Job 2's running anchor stays at `orderStartTime` and its merged `maxReturnWindowsMinutes` is still the delivery-calc default — but the *implicit anchor* used to chain to Job 3 is wrong. Final timings will then fail the strictly-increasing check in `computeJobReturnTimings`, masking the original cause behind a different error. The existing test `ignores a maxReturnTime override that produces a non-positive window` is single-job so it doesn't catch this. Per the in-code comment "the picker bounds normally prevent this," so this is mostly a defence-in-depth gap, but it can still happen if the form ever stores absolute `maxReturnTime` (which the type still permits).
**Fix:** When the `maxReturnTime` override is rejected, still advance `runningMaxReturn` by the delivery-calc `maxReturnWindowsMinutes` so the chain stays consistent:

```typescript
} else if (override?.maxReturnTime) {
  const overriddenMaxReturn = new Date(override.maxReturnTime);
  const overriddenWindow = differenceInMinutes(
    overriddenMaxReturn,
    runningMaxReturn
  );

  if (Number.isFinite(overriddenWindow) && overriddenWindow > 0) {
    maxReturnWindowsMinutes = overriddenWindow;
    runningMaxReturn = overriddenMaxReturn;
  } else if (typeof maxReturnWindowsMinutes === "number") {
    // Override rejected — advance anchor by the delivery-calc
    // default so downstream chain math stays correct.
    runningMaxReturn = addMinutes(
      runningMaxReturn,
      maxReturnWindowsMinutes
    );
  }
}
```

Add a multi-job test that confirms downstream chaining after a rejected `maxReturnTime` override.

---

### 2. Acceptance criterion 5.4 (`returnTime <= maxReturnTime`) is enforced by construction but never asserted

**[File: apps/creative-portal/api/orders/createNew/helpers/computeJobReturnTimings.ts]**
**Function/Class:** computeJobReturnTimings / computePreAssignedReturnTime
**Severity:** low
**Verified Status:** ✅ RESOLVED — Added 15 parameterised invariant tests (3 anchor sources × 5 window/cap positions) in `computePreAssignedReturnTime.test.ts` that pin the helper's `min([...])` cap. Catches any future refactor that drops the cap without requiring a runtime check.
**Problem:** Spec rule 5.4 (validation rule, not a derivation rule) says any supplied `returnTime` must be ≤ `maxReturnTime`. The current implementation never validates this — it relies on `computePreAssignedReturnTime` always capping via `min([projected, maxReturn])`. A future refactor that bypasses that helper (e.g., when picking up overrides from the form, or for grouped orders) could violate the invariant without any check failing. No test asserts the invariant either.
**Impact:** Future regression risk. The OMS API will reject (per testing-note 4), but the failure surfaces as a network error to the admin instead of pre-flight validation feedback.
**Fix:** Either (a) add a one-line invariant check inside `computeJobReturnTimings` that runs over `jobTimings` after the loop, comparing each pre-assigned job's `returnTime` against its `maxReturnTime`; or (b) add an explicit unit test for `computePreAssignedReturnTime` asserting it never returns a value > `currentMaxReturnTime` under any input. Option (a) is more defensive; the helper already accepts `returnTime` for non-assigned jobs and could grow new branches.

---

### 3. `WorkflowWindowInputRowProps.jobType` is dead

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/partials/WorkflowWindowInput/partials/WorkflowWindowInputRow/types.ts]**
**Function/Class:** WorkflowWindowInputRowProps / useWorkflowWindowInputRow
**Severity:** low
**Verified Status:** ✅ RESOLVED — Dropped `jobType` from `WorkflowWindowInputRowProps`, the component destructure, the hook invocation, and the call site in `generateWorkflowComponents.tsx` (which also drops the unused `as JobType` cast).
**Problem:** The row component (`index.tsx:38`) and the hook signature both accept `jobType` but it is never read. The component just forwards it to the hook, and the hook only destructures `{ jobIndex, orderIndexes }`. The caller in `generateWorkflowComponents.tsx:202` passes `jobType={jobTypeName as JobType}` for no effect.
**Impact:** Slightly misleading API surface; an unused cast (`as JobType`) is being performed at the call site. Eslint may flag the unused destructure on a stricter config.
**Fix:** Drop `jobType` from `WorkflowWindowInputRowProps`, the component destructure, the hook signature, and the call site:

```typescript
// types.ts
export interface WorkflowWindowInputRowProps {
  jobIndex: number;
  orderIndexes: number[];
}

// generateWorkflowComponents.tsx
<WorkflowWindowInputRow
  jobIndex={index}
  orderIndexes={currentOrderIndexes}
/>
```

---

### 4. `useWorkflowWindowInputRow` exports values that nothing consumes

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/partials/WorkflowWindowInput/partials/WorkflowWindowInputRow/hooks.ts]**
**Function/Class:** useWorkflowWindowInputRow
**Severity:** low
**Verified Status:** ✅ RESOLVED — Removed `isPreAssigned`, `jobDueDateAside`, `userDueDateAside`, `leftDelta`, `rightDelta` from the return + their backing memos. Also pruned the now-unused `toHoursAndMinutes` import. ~42 lines deleted.
**Problem:** The hook returns `isPreAssigned`, `jobDueDateAside`, `userDueDateAside`, `leftDelta`, `rightDelta`. The row component (`index.tsx`) destructures only `leftDate, maxReturnTime, maxReturnWindowsMinutes, nextMaxReturnTimeBound, orderBufferMinutes, orderDeadline, previousAnchorForMaxReturn, previousAnchorForProjected, previousMaxReturnTimeBound, returnWindowsMinutes, userDueDateSlackMinutes`. The other returns are dead.
**Impact:** Maintenance noise — readers (and the perfectionist sort rule) have to reason about whether these are intentional. `leftDelta` / `rightDelta` are explicitly marked "kept for backwards compatibility" but nothing actually consumes them.
**Fix:** Remove the unused returns and the memos backing them (`isPreAssigned`, `userDueDateAside`, `jobDueDateAside`, `leftDelta`, `rightDelta`). If `isPreAssigned` is anticipated for PP-1642 sidebar work, leave a comment pointing to that ticket — otherwise drop it.

---

### 5. `CreateOrderNewPayloadType.order.jobs` is `Record<string, any>`, defeating type safety

**[File: apps/creative-portal/api/orders/types.ts]**
**Function/Class:** CreateOrderNewPayloadType
**Severity:** low
**Verified Status:** ✅ RESOLVED — Introduced `CreateOrderJobPayload` interface in `api/orders/types.ts`. Dropped the `eslint-disable @typescript-eslint/no-explicit-any` and one `@ts-ignore TODO` in `utils.ts` that the new type resolves. Surfaced 3 latent null-safety issues (assignedUsers can be null/undefined) — fixed with optional chaining + a typed filter in the OFFERED branch. Other unrelated `@ts-ignore`s on `currentJob` / `jobConfigurationId` left in place (different type).
**Problem:** The job array element is declared as `Record<string, any> & { maxReturnTime?: string | Date; maxReturnWindowsMinutes?: number; returnWindowsMinutes?: number }` with an eslint-disable for `no-explicit-any`. This is precisely the workspace pattern the user's memory warns against (feedback rule against `eslint-disable` for `no-explicit-any`).
**Impact:** Inside `createOrderBusinessLogic`, fields like `jobDetails.status`, `jobDetails.assignedUsers[0]?.id`, `jobDetails.jobTypeName` are read without type checking — and there's already a `// @ts-ignore TODO` on line 550 (`jobDetails?.jobTypeName === JobType.SERVICE`). The contract between the form schema (`CreateOrderSchema['orders'][number]['jobs'][number]`) and this payload type is invisible to the compiler.
**Fix:** Replace `Record<string, any>` with an explicit interface that mirrors the form schema (status, jobId, jobTypeName, assignedUsers, maxReturnTime, maxReturnWindowsMinutes, returnWindowsMinutes). Remove the eslint-disable. This will surface the existing `// @ts-ignore` sites as real type errors to fix; some may simply go away.

---

### 6. Inner loop iterates `deliveryCalculation.jobs` instead of `mergedDeliveryJobs`

**[File: apps/creative-portal/api/orders/createNew/utils.ts]**
**Function/Class:** createOrderBusinessLogic
**Severity:** low
**Verified Status:** ✅ RESOLVED — Inner loop now iterates `mergedDeliveryJobs` so any future `mergeJobOverrides` filter/reorder stays in sync with the timings array.
**Problem:** Line 534: `for (const deliveryCalculationJob of deliveryCalculation.jobs)` — but the timings on `jobTimings` were computed from `mergedDeliveryJobs`. The loop body only uses `deliveryCalculationJob.jobTypeId` to look up `currentJob` and `timing`, so this currently works because `mergeJobOverrides` preserves order and jobTypeIds. But if `mergeJobOverrides` ever filters or reorders (e.g., to drop a job that lacks a matching configuration), the indices would diverge silently.
**Impact:** Latent fragility. A future change to `mergeJobOverrides` could break order semantics without a test failing.
**Fix:** Iterate `mergedDeliveryJobs` instead of `deliveryCalculation.jobs`:

```typescript
for (const deliveryCalculationJob of mergedDeliveryJobs) {
```

The body is unchanged — `mergedDeliveryJobs[N].jobTypeId` equals `deliveryCalculation.jobs[N].jobTypeId`.

---

### 7. `supportDocuments.test.ts` mocks `mergeJobOverrides` indirectly only through empty arrays

**[File: apps/creative-portal/api/orders/createNew/__tests__/supportDocuments.test.ts]**
**Function/Class:** describe("createOrderBusinessLogic — support documents")
**Severity:** low
**Verified Status:** ✅ RESOLVED — Added new `__tests__/jobTimings.test.ts` exercising `createOrderBusinessLogic` with real helpers (only mocks at IO boundary). Covers the four spec cases: all-unassigned, first-pre-assigned (anchor on Job1 maxReturnTime), mixed sequence (Job 3 anchors on Job 2's `returnTime` per AC 3.2), and OFFERED → IN_QUEUE rewrite with `addJobCandidate` per offered user.
**Problem:** The test fixture sets `order.jobs: []` and mocks `getDeadline → { jobs: [] }`, so `mergeJobOverrides` is invoked with empty arrays and exercises none of its branches. The other newly-added helpers (`computeJobReturnTimings`, `computePreAssignedReturnTime`, `JobTimingValidationError`) are mocked at the module boundary. None of these tests exercise the pre-assigned path or verify that `proofedUserId` and `returnTime` ride together in the payload.
**Impact:** Spec acceptance criteria for pre-assigned jobs (testing notes 2 and 3 in the Jira ticket — "all unassigned", "one pre-assigned at first position", "multiple pre-assigned in sequence") are NOT covered by integration tests against `createOrderBusinessLogic`. Unit tests for the helpers cover the math, but the wiring (`isPreAssigned` detection, `previousJobReturnTime/MaxReturnTime` flip-flop, `returnTime` spread into payload) is untested.
**Fix:** Add an integration test file (e.g., `__tests__/jobTimings.test.ts`) that mocks `addJob` to capture the payload, then asserts:
- A 3-job all-unassigned order produces 3 calls without `returnTime`, all with `maxReturnTime` + `returnWindowsMinutes`.
- A 3-job order with Job 1 ASSIGNED produces Job 1 with `returnTime` set, Jobs 2+ anchored on Job 1's `returnTime`.
- A 3-job order with Job 1 IN_QUEUE, Job 2 ASSIGNED, Job 3 ASSIGNED produces Job 2 anchored on Job 1's `maxReturnTime`, Job 3 anchored on Job 2's `returnTime`.
- An OFFERED job sends `status: IN_QUEUE` (not `proofedUserId`) and no `returnTime`.

This is the highest-value coverage gap given the spec is largely about who-gets-what-fields.

---

### 8. `useDeadlineManagement.handleDateChange` only reports the first order's buffer when multiple orders are selected

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/WorkflowStep/hooks/useDeadlineManagement.ts]**
**Function/Class:** useDeadlineManagement / handleDateChange
**Severity:** low
**Verified Status:** ⏭️ DEFERRED — `useDeadlineManagement.ts:251` still reads `setBuffer(Math.max(0, updatedBuffers[0]))`. Pre-existing behaviour carried forward; needs design call on multi-order Buffer presentation.
**Problem:** Line 251: `setBuffer(Math.max(0, updatedBuffers[0]))` — takes only `updatedBuffers[0]`. The validation block above (`isInsufficientBuffer`) correctly checks all orders, so the *error gate* is right, but the visible Buffer pill only reflects the first order. The pre-PP-1641 code had the same behavior; PP-1641 preserves it. Flagging as a minor pre-existing UX inconsistency that the rewrite carried forward.
**Impact:** If two selected orders have different buffers, the admin sees the first one's value even though both are tracked internally.
**Fix:** If the design intent is "show the tightest buffer," use `Math.min(...updatedBuffers.map((b) => Math.max(0, b)))`. Confirm with design — could be left as is.

---

### 9. Schema's `window` field is still `Yup.number().required()` despite being legacy

**[File: apps/creative-portal/components/organisms/NewOrderForm/schemas.ts]**
**Function/Class:** JOBS_SCHEMA
**Severity:** informational
**Verified Status:** ⏭️ DEFERRED — `schemas.ts:21` still has `window: Yup.number().required()`. Tracked migration debt; address in PP-1642 / follow-up.
**Problem:** Line 21: `window: Yup.number().required()` is marked legacy by the PR's own comment ("Remove `window` once all consumers have migrated"). It is still required, while `returnWindowsMinutes` (the new replacement) is `optional()`. The provider effect that writes `window` (lines 491-536) and the new effect that writes `returnWindowsMinutes` (lines 543-608) are both gated independently — there's no transactional guarantee that both fields land before schema validation runs.
**Impact:** Mostly OK because the same `useEffect` cascade writes both — but if a future refactor drops the legacy `window` writer first, schema validation will fail until callers stop expecting `window`. Acknowledged technical debt, not a blocker.
**Fix:** No action this PR; track in PP-1642 / migration follow-up.

---

## Tests

- ✅ `computeJobReturnTimings.test.ts` — 10 cases covering single-job, multi-job chaining, missing windows, zero windows, non-monotonic maxReturnTime, deadline overshoot, empty list, custom start time, and `statusCode` propagation. Good coverage.
- ✅ `computePreAssignedReturnTime.test.ts` — 7 cases covering all three anchor sources, max-cap behavior, and the "previous returnTime wins over maxReturnTime" precedence rule.
- ✅ `mergeJobOverrides.test.ts` — 11 cases covering no-overrides, `returnWindowsMinutes`-only, `maxReturnTime`-only chain propagation, `maxReturnWindowsMinutes` direct override, rejection of non-positive overrides, sub-minute precision, unknown-jobId skip, and combined `returnWindowsMinutes + maxReturnTime`.
- ✅ `useDeadlineManagement.test.ts` — 4 cases for the new buffer-from-chain-end logic.
- ❌ No integration test covering `createOrderBusinessLogic` with non-empty job payloads (see issue 7).
- ⚠️ No test asserting `returnTime <= maxReturnTime` invariant (see issue 2).
- ⚠️ Multi-job chain integrity after a rejected `maxReturnTime` override is untested (see issue 1).

PR description notes "Manual testing completed" and "E2E tests pass" are unchecked; per the CLAUDE.md design-fidelity rule and the size of the UI change (the `WorkflowWindowInput` rewrite + new modal info captions + chain-cascade behaviour), this should be visually verified in `yarn dev:creative-portal` before merge.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Spec ACs all addressed; one subtle chain-anchor bug under an edge case |
| Regression risk | ⚠️ Medium — large UI rewrite, no integration test on the API wiring |
| Tests | ⚠️ Strong helper-level coverage, missing integration + invariant tests |
| Code quality | ⚠️ Clean overall; small dead code, one `Record<string, any>`, several `// @ts-ignore` sites preserved |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions**

The core implementation is sound and the spec is correctly translated into the helpers. Before merging:

1. Fix issue 1 — advance the running anchor when `mergeJobOverrides` rejects a `maxReturnTime` override (and add a 2-job test).
2. Add an integration test (issue 7) covering the all-unassigned, first-pre-assigned, mixed-sequence, and OFFERED paths through `createOrderBusinessLogic`.
3. Drop the dead `jobType` prop and unused hook returns (issues 3 + 4) — small, safe cleanup.
4. Replace `Record<string, any>` in `CreateOrderNewPayloadType` with an explicit shape (issue 5) — surfaces the existing `@ts-ignore` debt for a follow-up.
5. Complete the manual + E2E test boxes in the PR description, and confirm Figma fidelity for the new chip captions and cascading chain UX per CLAUDE.md.

Issues 2, 6, 8, 9 are non-blocking; pick up in PP-1642 / PP-1643 or as a follow-up.
