# PR Review: fix/PP-1864: recalculate charge adjustments on word count change

**PR:** [https://github.com/Proofed/B2BWebserver/pull/2294](https://github.com/Proofed/B2BWebserver/pull/2294)
**Jira:** [https://proofed.atlassian.net/browse/PP-1864](https://proofed.atlassian.net/browse/PP-1864)
**Status:** In Progress

---

## Jira Requirements vs Implementation


| Jira Requirement                                                                                                              | PR Implementation                                                                                                                  | Status      |
| ----------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| `workItemSize` updates to the new value                                                                                       | Passed as before to `updateOrder` alongside the new charge fields                                                                  | ✅ Addressed |
| Per-word job tasks' `quotedChargeQuantity` and `quotedPayQuantity` update                                                     | `updatedJobTasks` computed locally and passed to `updateJobTask` per task                                                          | ✅ Addressed |
| `minimumChargeAdjustmentAmount` and `minimumChargeRate` recalculated against new subtotal (cleared to 0 when above threshold) | `calculateOrderChargeAdjustments` in `add` mode handles both the clear and the recalculate cases; result spread into `updateOrder` | ✅ Addressed |
| `formatPremiumAmount` and `formatPremiumRate` recalculated against new subtotal                                               | Same `calculateOrderChargeAdjustments` call handles format premium recalculation                                                   | ✅ Addressed |


No out-of-scope changes detected.

---

## Architecture Analysis

The fix reuses the existing `calculateOrderChargeAdjustments` utility already used in `useAddNewJob` — a clean, consistent approach. The hook computes `updatedJobTasks` locally (without a round-trip) before calling the utility, so the charge recalculation is based on the exact quantities that will be written to the DB. The result is spread into the single `updateOrder` call, keeping the mutation count the same as before. The ordering of operations (recalculate → updateOrder → close modal → updateJobTasks → refresh) is correct.

---

## Issues Found

### 1. Stale `serviceJobTasks` in `useCallback` dep array

**[File: apps/creative-portal/components/pages/admin-area/orders/partials/OrderManagementSidebar/partials/ChangeWordCountModal/hooks.ts]**
**Function/Class:** `onSubmit` (`useCallback`)
**Severity:** low
**Problem:** `serviceJobTasks` is listed in the `useCallback` dependency array at line 109 but is never referenced inside the `onSubmit` callback body. It was already a phantom dependency before this PR. The PR adds the correct new deps (`jobTasks`, `masterReferencesCharge`) but leaves this stale entry in place.
**Impact:** No runtime bug — extra deps only cause unnecessary re-creation of the callback, but it is misleading and keeps the `eslint-disable-next-line react-hooks/exhaustive-deps` comment alive unnecessarily.
**Fix:** Remove `serviceJobTasks` from the dep array, then check whether the `eslint-disable-next-line` comment can also be removed (it likely can, since all actual deps are now listed correctly).

```typescript
// Remove serviceJobTasks from the array:
[
  order,
  jobs,
  jobTasks,
  masterReferencesCharge,
  updateOrder,
  updateJobTask,
  chargePerWordUnit,
  payPerWordUnit,
  orderManagementModalProps,
  refreshOrderSidePanel,
  refreshJobs,
  refreshJobTasks
]
```

### 2. Refresh calls not covered by tests

**[File: apps/creative-portal/components/pages/admin-area/orders/partials/OrderManagementSidebar/partials/ChangeWordCountModal/hooks.test.ts]**
**Function/Class:** `useChangeWordCountModal` test suite
**Severity:** low
**Problem:** `refreshOrderSidePanel`, `refreshJobs`, and `refreshJobTasks` are important side effects of the success path but none of the eight tests assert they are called. The "closes modal and shows success toast" test is the natural home for these assertions.
**Impact:** A future refactor that accidentally removes the refresh calls would go undetected by the test suite.
**Fix:** Extend the success-path test:

```typescript
it("closes the modal and shows a success toast after a successful update", async () => {
  mockCalculateOrderChargeAdjustments.mockReturnValue({});

  await submit("1,500");

  expect(mockSetIsChangeWordCountModalOpen).toHaveBeenCalledWith(false);
  expect(mockShowToast).toHaveBeenCalledWith(
    expect.objectContaining({ type: "success", title: "Updated Word Count" })
  );
  expect(mockRefreshOrderSidePanel).toHaveBeenCalledWith({ orderId: mockOrder.id });
  expect(mockRefreshJobs).toHaveBeenCalledWith(mockOrder.id);
  expect(mockRefreshJobTasks).toHaveBeenCalledWith(
    mockJobs.map(({ id }) => id.toString())
  );
});
```

### 3. Incomplete Jira ticket URL in PR description

**[File: PR description]**
**Severity:** low
**Problem:** The Jira link in the PR body reads `https://proofed.atlassian.net/browse/PP-` — the ticket number is missing.
**Impact:** Administrative; does not affect the code. Makes it harder to trace from GitHub to Jira without reading the branch name.
**Fix:** Update to `https://proofed.atlassian.net/browse/PP-1864`.

---

## Tests

- ✅ New test file `hooks.test.ts` added with 8 focused unit tests
- ✅ Tests cover the add-mode charge recalculation path (correct args to `calculateOrderChargeAdjustments`)
- ✅ Tests cover minimum charge threshold crossed in both directions
- ✅ Tests cover format-premium-only return from utility
- ✅ Tests cover no-adjustment return (existing order values preserved via spread)
- ✅ Tests cover per-word vs non-per-word job task update behaviour
- ✅ Tests cover error path (updateOrder failure → showDefaultErrorToast, job tasks not updated)
- ⚠️ Refresh side-effects (`refreshOrderSidePanel`, `refreshJobs`, `refreshJobTasks`) not asserted in any test

---

## Summary


| Aspect          | Status                                     |
| --------------- | ------------------------------------------ |
| Correctness     | ✅                                          |
| Regression risk | ✅ Low                                      |
| Tests           | ⚠️ Partial (refresh side-effects untested) |
| Code quality    | ⚠️ Stale dep + eslint-disable comment      |
| Mergeable state | ✅ Clean                                    |


---

## Recommendation

**Approve with suggestions**

1. Remove `serviceJobTasks` from the `useCallback` dep array and drop the `eslint-disable-next-line react-hooks/exhaustive-deps` comment if all deps are now correctly listed.
2. Add assertions for `refreshOrderSidePanel`, `refreshJobs`, and `refreshJobTasks` in the success-path test.
3. Fix the Jira ticket URL in the PR description (`PP-` → `PP-1864`) and mark the PR as ready for review when the draft work is complete.

