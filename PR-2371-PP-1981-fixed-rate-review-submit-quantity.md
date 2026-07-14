# PR Review: fix/PP-1981: Keep Fixed Rate quoted quantity on reviewer submit

**PR:** https://github.com/Proofed/B2BWebserver/pull/2371
**Jira:** https://proofed.atlassian.net/browse/PP-1981
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Fixed-price job's approved **charge** quantity must default to the quoted quantity (≈1), not the entered minutes — `Total Charge = fixed price × 1` | `getApprovedQuantities` returns `quotedChargeQuantity` when `useFixedChargeQuantity` (or per-word) — entered minutes no longer applied as charge qty | ✅ Addressed |
| Fixed-price job's approved **pay** quantity must default to the quoted quantity — `compensation = fixed pay × 1` | Same helper returns `quotedPayQuantity` when `useFixedPayQuantity` (or per-word) | ✅ Addressed |
| Fix the frontend reviewer/approval path (`Submission/hooks.ts`) that special-cased per-word only, so fixed-price hourly fell through | `handleReviewJobSubmission` now calls `getApprovedQuantities(task, approvedWorkTime)` instead of the per-word-only inline logic — this is the actual production bug path | ✅ Addressed (root cause) |
| Fix the API submission util (`api/utils/jobs/postSubmitJob.ts`) named in the investigation | Both `postSubmitJob.ts` **and** `postSubmitJobStream.ts` refactored to reuse the shared helper (they already had a correct inline guard; now de-duplicated) | ✅ Addressed |
| "Approved-quantity field should not be editable" for fixed-price (match per-word behaviour) | The editable approved-work-time input in `ServiceSubmission` was **already** hidden for fixed-price (`isJobPriceFixed`, pre-PR). PR additionally hides the read-only "Approved Time:" summary column for fixed-rate in `JobSubmission.tsx` | ✅ Addressed (display parity) |
| Price preview (`calculateJobTaskPrice.ts`) "reproduces the same formula" | Verified: the preview already guards with `useFixedQuantity ? quotedQuantity : approvedQuantity || …` — it is **not** buggy for fixed-rate, so no change was required | ✅ N/A (already correct) |

**Scope note:** The `JobSubmission.tsx` display change (hiding the "Approved Time:" column + rerouting `shouldShowScore`/`shouldShowScoreInline` through a new `hasApprovedTimeColumn` flag) and the consolidation of the two API routes onto the shared helper go slightly beyond the literal ticket, but both are tightly coupled to the same bug (a fixed-rate job would otherwise show a nonsensical "Approved Time: 1 min") and are reasonable. No unrelated scope creep.

---

## Architecture Analysis

The fix correctly identifies and repairs the **root cause**: the frontend reviewer-submit path (`handleReviewJobSubmission`) only guarded per-word tasks, so for a Fixed Rate task the editor's entered minutes were sent as `approvedChargeQuantity`/`approvedPayQuantity`, and the .NET charge API faithfully multiplied `rate × minutes`. The two API submission routes already had the correct `useFixed*` guard; the frontend path did not. The PR extracts a single `getApprovedQuantities` helper that mirrors the API guard and wires all three call sites (frontend hook + two API routes) to it, eliminating the divergence that caused the bug.

The helper's logic is sound and matches the source-of-truth guard: keep the quoted quantity for per-word (`chargeUnit/payUnit === WORDS_UNIT_VALUE`) or fixed-quantity (`useFixedChargeQuantity/useFixedPayQuantity`) tasks; apply the entered work time only for genuinely hourly tasks. Charge and pay are resolved independently, so mixed tasks (fixed charge + hourly pay) are handled correctly.

The display change in `JobSubmission.tsx` is a clean refactor: the previously-duplicated `isServiceJobOrReviewJobType && !!totalApprovedWorkTime && !isPerWordJob` expression is lifted into a single `hasApprovedTimeColumn` flag (now also excluding `isFixedRateJob`), and `shouldShowScore`/`shouldShowScoreInline` reuse it — reducing three copies of the condition to one.

I traced the two other surfaces the Jira investigation named and confirmed neither needs a change: the editable work-time input is already hidden for fixed-price jobs (`ServiceSubmission/index.tsx` line 128, `isJobPriceFixed`), and `calculateJobTaskPrice.ts` already branches on `useFixedQuantity`. Both `WORDS_UNIT_VALUE` sources resolve to the same value (`1000`), so the frontend's import-source change is behaviour-preserving.

---

## Issues Found

### 1. API server utils import a helper from a deeply-nested UI component folder

**[File: apps/creative-portal/api/utils/jobs/postSubmitJob.ts]** (and `postSubmitJobStream.ts`)

**Function/Class:** postSubmitJob / postSubmitJobStream

**Severity:** low

**Problem:** Both API-route utilities now import `getApprovedQuantities` from `components/organisms/sidebars/contents/JobManagement/partials/Submission/utils`. This inverts the usual dependency direction (server/API code depending on a React component partial) and hard-couples the API routes to the exact folder path of a UI partial.

**Impact:** No runtime/bundle risk today — `utils.ts` is a pure module importing only `WORDS_UNIT_VALUE` and the `JobTask` type (no React/client-only code), and ES imports pull only that file, not the sibling `index.tsx`/`hooks.ts`. But it's a fragile, backwards coupling: renaming or relocating the `Submission/` component folder (a routine UI refactor) would silently break two server routes, and it reads oddly for anyone tracing API dependencies.

**Fix:** Move the shared helper to a neutral, dependency-appropriate location that both the API and the component can import — e.g. `apps/creative-portal/api/utils/jobs/getApprovedQuantities.ts` (co-located with the routes that own the billing rule) or `@proofed/shared`. Have `Submission/hooks.ts` import from there rather than the API importing "up" into components:

```typescript
// api/utils/jobs/getApprovedQuantities.ts  (new home)
export const getApprovedQuantities = (
  task: JobTask,
  approvedWorkTime: number
): ApprovedQuantities => { /* … unchanged … */ };

// Submission/hooks.ts
import { getApprovedQuantities } from "api/utils/jobs/getApprovedQuantities";
```

### 2. Two sources of truth for the unit constants (util vs. test import different modules)

**[File: apps/creative-portal/components/organisms/sidebars/contents/JobManagement/partials/Submission/utils.test.ts]**

**Function/Class:** getApprovedQuantities test suite

**Severity:** low

**Problem:** `utils.ts` imports `WORDS_UNIT_VALUE` from `@proofed/shared/config/units`, while the new `utils.test.ts` imports `WORDS_UNIT_VALUE`, `FIXED_RATE_UNIT_VALUE`, and `HOURLY_UNIT_VALUE` from `components/pages/partners/[partnerId]/projects/[projectId]/settings/consts`. Both modules independently define the same values (`1000` / `60` / `1`), so the tests pass — but the code-under-test and its test derive the "per-word" constant from different files.

**Impact:** Duplication of the unit constants across two modules (pre-existing in the repo, not introduced here). Should the two copies ever drift, the test could silently keep asserting against the wrong value while the util uses another. Low risk given both are long-stable literals.

**Fix:** Have the test import the same `WORDS_UNIT_VALUE` the util uses (`@proofed/shared/config/units`) for the per-word case, so test and code share one source. Longer term, the partner-settings `consts.tsx` copy of `WORDS_UNIT_VALUE`/`HOURLY_UNIT_VALUE`/`FIXED_RATE_UNIT_VALUE` should re-export from `@proofed/shared/config/units` rather than redeclare — out of scope for this PR.

### 3. No regression test at the actual bug-path (`handleReviewJobSubmission`)

**[File: apps/creative-portal/components/organisms/sidebars/contents/JobManagement/partials/Submission/hooks.ts]**

**Function/Class:** handleReviewJobSubmission

**Severity:** low

**Problem:** The production bug lived in `handleReviewJobSubmission`. The PR adds strong coverage for the extracted pure helper (`utils.test.ts`, 4 cases) and for the display flag (`JobSubmission.test.tsx`, 3 cases), but there is no test exercising the hook itself — that it calls `getApprovedQuantities` with the correctly-divided `approvedWorkTime`, and that `mutateJobTask` receives `{ id, approvedChargeQuantity, approvedPayQuantity, requestType: "Approval" }` per task.

**Impact:** The wiring that actually regressed (per-word-only guard → shared helper) is verified only transitively. A future edit to the hook that dropped or misused the helper wouldn't be caught by the new tests. Low risk — the helper is well-tested and the call site is a one-line spread — but the highest-value regression test for this exact ticket is the one not present.

**Fix:** Optional but recommended: add a hook/integration test that mocks `usePatchJobTaskMutation` and asserts, for a Fixed Rate service task, that `mutateJobTask` is called with `approvedChargeQuantity`/`approvedPayQuantity` equal to the quoted quantity (not the entered minutes). This directly pins the behaviour the ticket is about.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ Skipped — user opted out | Not run. PR description claims all creative-portal tests pass (2 unrelated pre-existing failures on `develop` noted). |
| `npx turbo run typecheck` | ⏭️ Skipped — user opted out | Not run. Static review found no type-safety concerns (all helper fields exist on `JobTask`; return type is `Partial<JobTask>`-compatible). |
| `npx turbo run lint` | ⏭️ Skipped — user opted out | Not run. PR notes a pre-existing prettier failure in `JobReturnTimesTray/index.test.tsx` on `develop`, untouched here. |
| `npx turbo run build` | ⏭️ Skipped — user opted out | Not run. PR claims build green (creative-portal). |

**Validation was not run at the reviewer's request — re-run `test` / `typecheck` / `lint` / `build` on the PR branch before merging.**

---

## Tests

- ✅ New unit tests for the extracted helper — `utils.test.ts` covers Fixed Rate, per-word, hourly, and mixed (fixed charge + hourly pay) tasks; assertions match the intended behaviour.
- ✅ New display tests — `JobSubmission.test.tsx` verifies "Approved Time:" shows for hourly, is hidden for Fixed Rate (with "Editor's Work Time:" still shown), and stays hidden for per-word. Verified the test premise: `totalApprovedWorkTime` derives from `approvedPayQuantity`, so the Fixed Rate case (`approvedPayQuantity: 1`) is truthy and would have shown under the old condition — the assertion is meaningful.
- ⚠️ No test at the `handleReviewJobSubmission` hook level — the exact code path that regressed (see Issue 3).
- ⏭️ Full suite not executed (validation skipped). PR author reports creative-portal tests + typecheck + build green, with 2 unrelated pre-existing failures on `develop`.
- ➖ E2E: not applicable / not run (pricing-logic fix).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Fixes the root cause; helper mirrors the API guard; charge & pay resolved independently |
| Regression risk | ✅ Low — behaviour unchanged for per-word/hourly; only the fixed-rate path changes; import-source swap is value-equivalent (`WORDS_UNIT_VALUE === 1000` in both modules) |
| Tests | ⚠️ Good coverage of the helper + display, but the actual hook path has no direct test |
| Code quality | ✅ Good — DRY consolidation of 3 call sites; clean display refactor. One low-severity architectural smell (API → component import) |
| Validation suite | ⏭️ Skipped — user opted out (re-run before merge) |
| Mergeable state | ⏭️ Skipped — user opted out (GitHub reports `mergeable_state: clean`; no CI checks configured) |

---

## Recommendation

**Approve with suggestions.**

The change is correct, targets the true root cause, and consolidates the divergent guards that let the bug exist. The findings are all low-severity and none block merge on correctness grounds.

1. **Re-run the validation suite** (`test` / `typecheck` / `lint` / `build`) on the PR branch before merging — it was skipped in this review, so pass/fail is unverified here.
2. **Relocate `getApprovedQuantities`** out of the UI component folder into a neutral module (e.g. `api/utils/jobs/`) so the API routes don't import "up" into `components/` (Issue 1).
3. Consider adding a **hook-level regression test** for `handleReviewJobSubmission` that pins the Fixed Rate charge/pay quantities (Issue 3).
4. Minor: point `utils.test.ts` at the same `WORDS_UNIT_VALUE` source the util uses (Issue 2).
