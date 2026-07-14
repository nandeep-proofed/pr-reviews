# PR Review: fix/PP-1864: recalculate charge adjustments on word count change

**PR:** https://github.com/Proofed/B2BWebserver/pull/2294
**Jira:** https://proofed.atlassian.net/browse/PP-1864
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| `workItemSize` updates to the new value | `updateOrder({ ...order, workItemSize: updatedWorkItemSize, ...chargeAdjustments })` | ✅ Addressed |
| Per-word job tasks' `quotedChargeQuantity` / `quotedPayQuantity` update to the new value | `updatedJobTasks` maps per-word tasks (matched by `chargeUnit`/`payUnit`) to the new size, then each is sent via `updateJobTask` | ✅ Addressed |
| `minimumChargeAdjustmentAmount` / `minimumChargeRate` recalculated against new subtotal (cleared to 0 when above threshold) | `calculateOrderChargeAdjustments({ mode: "add" })` recomputes the subtotal from `updatedJobTasks`; add-mode sets a fresh adjustment when `subtotal < minimumChargeAmount`, or clears to 0 when `subtotal >= minimumChargeAmount` and a prior adjustment existed | ✅ Addressed |
| `formatPremiumAmount` / `formatPremiumRate` recalculated against new subtotal | Same utility recomputes format premium via `calculateFormatPremium` using the new subtotal + (possibly new) min-charge adjustment; result folded into the single `updateOrder` call | ✅ Addressed |

**Scope beyond Jira (all reasonable):**
- Removed the now-dead `serviceJobIds` / `serviceJobTasks` `useMemo`s and the `JobType` import — `serviceJobTasks` was only ever referenced in the dependency array, never in the callback body, so this is pure dead-code removal.
- Removed the `// eslint-disable-next-line react-hooks/exhaustive-deps` and corrected the dependency array (added `jobTasks`, `masterReferencesCharge`; removed `serviceJobTasks`), so exhaustive-deps now passes honestly.
- Added a new `hooks.test.ts` (previously none) — satisfies the "every PR must include tests" requirement.

---

## Architecture Analysis

The fix is small, surgical, and correct in approach. Rather than reimplementing the pricing math, it reuses the existing `calculateOrderChargeAdjustments` utility with `mode: "add"` — the same utility and mode already used by `useAddNewJob.ts` when a chargeable job is added to a live order. The reuse is idiomatic and matches CLAUDE.md's reuse-first convention.

Key structural improvements:
- `updatedJobTasks` is computed **once** and used for both the charge recalculation (subtotal input) and the actual `updateJobTask` persistence. This removes the prior duplication where the per-word mapping was inlined into the `updateJobTask` call, and guarantees the recalculated charge fields reflect exactly the job-task state that will be persisted.
- Charge adjustments are folded into the **single** existing `updateOrder` call (`...order, workItemSize, ...chargeAdjustments`) rather than a second round-trip.

Correctness of `mode: "add"` for a word-count change (which can move the subtotal **up or down**) was verified against the utility source:
- `computeAddModeAdjustments` handles both directions: `subtotal < minimumChargeAmount` → apply/refresh adjustment; `subtotal >= minimumChargeAmount && priorAdjustment > 0` → clear to 0. So a decrease that drops below the threshold correctly re-applies an adjustment, and an increase that crosses above correctly clears it.
- `calculateOrderSubtotalFromJobTasks` filters by `chargeable` internally, so passing all `updatedJobTasks` (per-word + hourly/non-chargeable) is safe.
- Format premium is recomputed against `subtotal + (new or existing) minimumChargeAdjustmentAmount`, matching the documented billing behaviour, and only overrides when the value actually changes.

The manual verification in the PR description (order 20907: 837 → 1200 words, min-charge adjustment clears to €0, format premium recalculates to €12.48, total €60.00 → €74.88) reconciles with this logic.

---

## Issues Found

### 1. Format premium silently keeps its stale value if submitted before the `PremiumChargeRate` query resolves

**[File: apps/creative-portal/components/pages/admin-area/orders/partials/OrderManagementSidebar/partials/ChangeWordCountModal/hooks.ts]**

**Function/Class:** useChangeWordCountModal (onSubmit)

**Severity:** low

**Problem:** `masterReferencesCharge` comes from `useMasterReferenceQuery(ReferenceName.PremiumChargeRate)` and defaults to `[]` while the query is loading. If a user submits the modal during that window, `calculateFormatPremium` finds no matching reference (`!reference?.value`) and returns `null`, so `computeAddModeAdjustments` leaves the format-premium fields unset — the order keeps its pre-change (stale) `formatPremiumAmount` / `formatPremiumRate` via the `...order` spread. The min-charge recalculation still works (it doesn't depend on the reference), so only the format-premium half of the fix silently no-ops.

**Impact:** In the narrow race where the modal is submitted before the master-reference query settles, the exact over/under-billing this ticket fixes could partially persist for format-premium orders. Low likelihood — the value is React-Query-cached and this mirrors the existing `useAddNewJob.ts` behaviour — but there is no guard preventing submission while the query is unresolved.

**Fix:** Optional hardening — disable/guard submit until the reference query is ready, e.g. surface `isLoading` from the query and block `onSubmit` (or the submit button) when references haven't loaded. Not a blocker; matches the established pattern.

### 2. Partial-failure window: order (incl. recalculated charge fields) is persisted and the modal closes before job-task updates complete

**[File: apps/creative-portal/components/pages/admin-area/orders/partials/OrderManagementSidebar/partials/ChangeWordCountModal/hooks.ts]**

**Function/Class:** useChangeWordCountModal (onSubmit)

**Severity:** low

**Problem:** `onSubmit` awaits `updateOrder(...)`, closes the modal, then awaits `Promise.all(updatedJobTasks.map(updateJobTask))`. If `updateOrder` succeeds but one of the `updateJobTask` calls fails, the order has already been written with the new `workItemSize` **and** the recalculated charge adjustments derived from job-task quantities that never persisted, leaving order-level and task-level state inconsistent. The `catch` fires `showDefaultErrorToast`, but the modal is already closed and the partial write stands.

**Impact:** Edge-case data inconsistency on a mid-flight API failure. The ordering (order first, then tasks) is **pre-existing** — the PR does not introduce it — but folding the recalculated charge fields into the same first call slightly widens the blast radius of a partial failure. Low severity given failures are rare and refresh calls follow.

**Fix:** Out of scope for this bug fix, but worth a follow-up: consider updating job tasks before (or transactionally with) the order, or only closing the modal after all writes resolve. No change required here.

### 3. Caller relies on an implementation detail of `mode: "add"`, not its documented contract

**[File: apps/creative-portal/components/pages/admin-area/orders/partials/OrderManagementSidebar/partials/ChangeWordCountModal/hooks.ts]**

**Function/Class:** useChangeWordCountModal (onSubmit) → calculateOrderChargeAdjustments

**Severity:** low

**Problem:** `calculateOrderChargeAdjustments`'s docstring states: *"Caller must ensure: for add mode, a chargeable job was added."* This caller does not add a job — it relies on add-mode performing a full subtotal recompute that happens to handle both increases and decreases. That is true of the current implementation, so it works correctly today, but the usage sits outside the utility's stated precondition.

**Impact:** No functional bug now. Risk is future maintainability: if someone later "optimizes" add-mode on the assumption that the subtotal only ever grows (per the docstring), this word-count caller would silently break for downward changes.

**Fix:** Either broaden the utility's docstring/contract to acknowledge "full recalculation" callers (word-count change), or add a brief comment at this call site noting the reliance on add-mode's full-recompute behaviour. Documentation-only.

### 4. Test carries a leftover mock for a module the hook no longer imports

**[File: apps/creative-portal/components/pages/admin-area/orders/partials/OrderManagementSidebar/partials/ChangeWordCountModal/hooks.test.ts]**

**Function/Class:** test setup

**Severity:** low

**Problem:** The test mocks `vi.mock("api/jobTypes/enums", () => ({ JobType: { SERVICE: "Service" } }))`, but the fix removes the `JobType` import from `hooks.ts`, so nothing in the unit under test consumes it. Similarly `mockJobs`' `jobType` fields are no longer read by the hook (only `jobs.map(({ id }) => ...)` is used).

**Impact:** None functionally — harmless dead mock. Minor test-hygiene nit; can mislead a future reader into thinking `JobType` still matters here.

**Fix:** Optionally drop the `api/jobTypes/enums` mock (and simplify `mockJobs` to just `{ id }`). Cosmetic.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ Skipped | Skipped — user opted out |
| `npx turbo run typecheck` | ⏭️ Skipped | Skipped — user opted out |
| `npx turbo run lint` | ⏭️ Skipped | Skipped — user opted out |
| `npx turbo run build` | ⏭️ Skipped | Skipped — user opted out |

> Validation suite was **not run** (user opted out). Re-run `test` / `typecheck` / `lint` / `build` on the PR branch before merging — the recommendation below cannot rely on validation passing.

---

## Tests

- ✅ New `hooks.test.ts` added (8 cases) — satisfies the "every PR must include tests" requirement.
- ✅ Wiring covered: args passed to `calculateOrderChargeAdjustments` (`mode: "add"`, same `order`, `masterReferencesCharge`, per-word tasks updated, non-per-word untouched).
- ✅ Threshold-cross clears min-charge adjustment; drop-below re-applies one; format-premium-only forwarding; empty-adjustments case preserves existing order fields.
- ✅ Per-word job-task updates and non-per-word pass-through; success toast + all three refresh calls; error path delegates to `showDefaultErrorToast` and does not call `updateJobTask`.
- ⚠️ Tests mock `calculateOrderChargeAdjustments` wholesale, so they verify the **hook's wiring**, not the pricing math. That is an appropriate unit boundary, **but** the utility itself has **no dedicated test file** (`calculateOrderChargeAdjustments.ts` is only exercised indirectly via `useAddNewJob.test.ts`). The real numbers in this ticket (e.g. clear-to-0 + 20% × new subtotal) are not asserted end-to-end anywhere. Pre-existing gap; consider adding direct unit tests for the utility in a follow-up.
- ⚠️ Automated validation suite not executed (see Validation Checks).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Sound — root cause fixed, both threshold directions handled |
| Regression risk | ✅ Low — dead-code removal + reuse of an already-shipped utility/pattern |
| Tests | ⚠️ Good wiring coverage; underlying utility math untested directly |
| Code quality | ✅ Clean — dedupes job-task mapping, honest deps array |
| Validation suite | ⚠️ Skipped — user opted out (must re-run before merge) |
| Mergeable state | ✅ Clean (GitHub `mergeable_state: clean`); validation not verified locally |

---

## Recommendation

**Approve with suggestions**

The fix correctly addresses the root cause described in PP-1864 by reusing `calculateOrderChargeAdjustments` in `"add"` mode, and the add-mode logic provably handles word counts moving both above and below the minimum-charge threshold. The change is tightly scoped (2 files), removes dead code, and ships with meaningful wiring tests. All findings are **low severity**.

Before merging:
1. **Re-run the validation suite** (`test` / `typecheck` / `lint` / `build`) on the PR branch — it was not run in this review.
2. (Optional) Guard submit while the `PremiumChargeRate` query is unresolved (Issue 1), so the format-premium recalculation can't silently no-op on a fast submit.
3. (Optional) Add direct unit tests for `calculateOrderChargeAdjustments` to lock in the actual pricing math (Tests note), and clarify the utility's add-mode contract for full-recompute callers (Issue 3).
4. (Optional) Drop the leftover `api/jobTypes/enums` mock from the test (Issue 4).
