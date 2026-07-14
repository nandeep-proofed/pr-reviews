# PR Review: fix/PP-1981: Keep Fixed Rate quoted quantity on reviewer submit

**PR:** https://github.com/Proofed/B2BWebserver/pull/2371
**Jira:** https://proofed.atlassian.net/browse/PP-1981
**Status:** Code Review — **all 3 issues triaged, verification pass complete**

---

## Review Outcomes

| # | Issue | Severity | Status | Resolution |
|---|---|---|---|---|
| 1 | API server utils import a helper from a nested UI component folder | low → **won't fix** | ❌ **Rejected** | `api/` → `components/` is the established house pattern (8 files, 3 with the identical shape). The closest precedent imports from a **`.tsx` containing JSX**; this PR's helper is a pure `.ts`. Fixing it here alone would make this PR the only non-conforming call site. Codebase-wide cleanup ticket instead. |
| 2 | Two sources of truth for the unit constants (util vs. test) | low | ✅ **Fixed** | `utils.test.ts` now imports all three constants from `@proofed/shared/config/units` (the source the util uses, and the dominant convention at 44 files vs 15). Values identical — no behaviour change. |
| 3 | No regression test at the actual bug-path (`handleReviewJobSubmission`) | low | ⏭️ **Skipped** | Gap is real, but the only *new* code in the hook is a one-line spread whose logic has 4 passing tests. The division above it is unchanged context. Reaching the internal fn costs ~7 mocks. Deferred. |

**Additional review point addressed (from PR comments, not in this report):** rename `getApprovedQuantities`' param `task` → `jobTask`. ✅ Applied — also updated the hook's map callback, aligning all three call sites with the `JobTask` type (the two API routes already used `jobTask`).

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Fixed-price job's approved **charge** quantity must default to the quoted quantity (≈1), not the entered minutes — `Total Charge = fixed price × 1` | `getApprovedQuantities` returns `quotedChargeQuantity` when `useFixedChargeQuantity` (or per-word) — entered minutes no longer applied as charge qty | ✅ Addressed |
| Fixed-price job's approved **pay** quantity must default to the quoted quantity — `compensation = fixed pay × 1` | Same helper returns `quotedPayQuantity` when `useFixedPayQuantity` (or per-word) | ✅ Addressed |
| Fix the frontend reviewer/approval path (`Submission/hooks.ts`) that special-cased per-word only, so fixed-price hourly fell through | `handleReviewJobSubmission` now calls `getApprovedQuantities(jobTask, approvedWorkTime)` instead of the per-word-only inline logic — this is the actual production bug path | ✅ Addressed (root cause) |
| Fix the API submission util (`api/utils/jobs/postSubmitJob.ts`) named in the investigation | Both `postSubmitJob.ts` **and** `postSubmitJobStream.ts` refactored to reuse the shared helper (they already had a correct inline guard; now de-duplicated) | ✅ Addressed |
| "Approved-quantity field should not be editable" for fixed-price (match per-word behaviour) | The editable approved-work-time input in `ServiceSubmission` was **already** hidden for fixed-price (`isJobPriceFixed`, pre-PR). PR additionally hides the read-only "Approved Time:" summary column for fixed-rate in `JobSubmission.tsx` | ✅ Addressed (display parity) |
| Price preview (`calculateJobTaskPrice.ts`) "reproduces the same formula" | Verified: the preview already guards with `useFixedQuantity ? quotedQuantity : approvedQuantity || …` — it is **not** buggy for fixed-rate, so no change was required | ✅ N/A (already correct) |

**Scope note:** The `JobSubmission.tsx` display change and the consolidation of the two API routes onto the shared helper go slightly beyond the literal ticket, but both are tightly coupled to the same bug (a fixed-rate job would otherwise show a nonsensical "Approved Time: 1 min") and are reasonable. No unrelated scope creep.

**Note on the PR description:** "Areas of Change" lists only 3 files, but the PR touches **7** (`JobSubmission.tsx`, `JobSubmission.test.tsx`, `postSubmitJob.ts`, `postSubmitJobStream.ts` are unlisted). Worth updating so the record matches the diff.

---

## Architecture Analysis

The fix correctly identifies and repairs the **root cause**: the frontend reviewer-submit path (`handleReviewJobSubmission`) only guarded per-word tasks, so for a Fixed Rate task the editor's entered minutes were sent as `approvedChargeQuantity`/`approvedPayQuantity`, and the .NET charge API faithfully multiplied `rate × minutes`. The two API submission routes already had the correct `useFixed*` guard; the frontend path did not. The PR extracts a single `getApprovedQuantities` helper that mirrors the API guard and wires all three call sites (frontend hook + two API routes) to it, eliminating the divergence that caused the bug.

The helper's logic is sound and matches the source-of-truth guard: keep the quoted quantity for per-word (`chargeUnit/payUnit === WORDS_UNIT_VALUE`) or fixed-quantity (`useFixedChargeQuantity/useFixedPayQuantity`) tasks; apply the entered work time only for genuinely hourly tasks. Charge and pay are resolved independently, so mixed tasks (fixed charge + hourly pay) are handled correctly.

**Parity verified between the two paths.** The API divides by `totalJobTask = jobTasks.length` and maps over the same `jobTasks`; the hook divides by `serviceJobTasks.length` and maps over the same `serviceJob.jobTasks`. Both divide by the length of the array they then iterate — the consolidation introduced no mismatch.

The display change in `JobSubmission.tsx` is a clean refactor: the previously-duplicated `isServiceJobOrReviewJobType && !!totalApprovedWorkTime && !isPerWordJob` expression is lifted into a single `hasApprovedTimeColumn` flag (now also excluding `isFixedRateJob`), and `shouldShowScore`/`shouldShowScoreInline` reuse it — reducing three copies of the condition to one.

I traced the two other surfaces the Jira investigation named and confirmed neither needs a change: the editable work-time input is already hidden for fixed-price jobs (`ServiceSubmission/index.tsx` line 128, `isJobPriceFixed`), and `calculateJobTaskPrice.ts` already branches on `useFixedQuantity`. Both `WORDS_UNIT_VALUE` sources resolve to the same value (`1000`), so the frontend's import-source change is behaviour-preserving.

---

## Issues Found

### 1. API server utils import a helper from a deeply-nested UI component folder — ❌ REJECTED (follows house convention)

**[File: apps/creative-portal/api/utils/jobs/postSubmitJob.ts]** (and `postSubmitJobStream.ts`)

**Severity:** ~~low~~ → **won't fix for this PR**

**What's true:** both API-route utilities now import `getApprovedQuantities` from `components/organisms/sidebars/contents/JobManagement/partials/Submission/utils`.

**Why this is being rejected — `api/` → `components/` is the established pattern here, not an inversion:**

```
api/orders/createNew/utils.ts:46                      → components/organisms/NewOrderForm/partials/WorkflowStep/utils
api/mixtures/jobs/addJobWithTasks/addJobWithTasks.ts  → components/organisms/NewOrderForm/partials/WorkflowStep/utils
api/mixtures/jobs/addNewJobs/addNewJobs.ts            → components/organisms/NewOrderForm/partials/WorkflowStep/utils
api/mixtures/configurations/workflow-setup/…          → components/pages/partners/[partnerId]/…/JobTemplates/utils/…
api/jobs/types.ts:4                                   → components/atoms/JobStatus/types
api/service-configuration/types.ts:2                  → components/pages/partners/[partnerId]/…/settings/types
```

8 files total; three are the *identical shape* (an API util importing a pure helper from a deeply-nested component partial's `utils`). No ESLint boundary rule (`no-restricted-paths` / `no-restricted-imports`) exists anywhere in the config.

**The closest precedent is materially worse than this PR.** `api/orders/createNew/utils.ts` imports `getJobTaskPayload` from `WorkflowStep/utils.tsx` — a **`.tsx`** file containing JSX that itself imports `getDocumentSmallIcon` from a shared component atom. This PR's `Submission/utils.ts` is a plain `.ts` whose only runtime import is `WORDS_UNIT_VALUE` from `@proofed/shared/config/units`; `JobTask` is a `type`, so that import is erased at compile time. This PR is strictly cleaner than the thing it imitates.

**Correction to the original impact claim:** it stated that relocating the `Submission/` folder would "**silently** break two server routes." It would not be silent — the import path breaks, TypeScript errors, the build fails. That's the loudest possible failure mode, and it's the same exposure the three existing precedents already carry.

**Resolution:** ❌ Rejected. Acting on this would make this PR the only call site of its kind that *doesn't* follow the house pattern, while leaving three worse instances untouched — a net loss in consistency for a low-severity smell. **Follow-up worth raising:** `getApprovedQuantities` is a *billing rule* and the API layer had it right first, so there's a genuine argument the API owns it and components should import down into it. That's a codebase-wide cleanup (move `getApprovedQuantities`, `getJobTaskPayload`, et al. to a neutral module together), not a change to this PR.

### 2. Two sources of truth for the unit constants (util vs. test import different modules) — ✅ FIXED

**[File: apps/creative-portal/components/organisms/sidebars/contents/JobManagement/partials/Submission/utils.test.ts]**

**Severity:** low

**Verified:** `utils.ts:1` imports `WORDS_UNIT_VALUE` from `@proofed/shared/config/units`; the test imported all three constants from `components/pages/partners/[partnerId]/projects/[projectId]/settings/consts`, which **redeclares** them independently at `consts.tsx:159-161` (no re-export). Both resolve to `1000` / `60` / `1`. The shared module is also the dominant convention: **44** files import from `@proofed/shared/config/units` vs **15** using the settings copy.

**Correction to the original impact claim:** it said drift would let the test "**silently** keep asserting against the wrong value." The opposite is true. The per-word case builds `chargeUnit: WORDS_UNIT_VALUE` and expects `850`; if either copy drifted, the util's `jobTask.chargeUnit === WORDS_UNIT_VALUE` goes false, it falls to the hourly branch, returns `180`, and the assertion **fails loudly**. Drift is caught, not hidden.

**Worth noting:** the util only ever references `WORDS_UNIT_VALUE` — the fixed branch keys off the `useFixed*` booleans, not the unit. So of the three constants the test imports, only `WORDS_UNIT_VALUE` is load-bearing; `FIXED_RATE_UNIT_VALUE` and `HOURLY_UNIT_VALUE` are illustrative test data.

**The original fix was a half-measure** — it suggested pointing only `WORDS_UNIT_VALUE` at shared, leaving a split import across two modules. `packages/shared/config/units.ts` already exports all three, so the settings import can be dropped entirely.

**Resolution:** ✅ Applied — all three constants now import from `@proofed/shared/config/units`; the `settings/consts` import is gone. Tests 4/4, eslint clean, typecheck green. **Follow-up (out of scope):** `settings/consts.tsx` should re-export from shared rather than redeclare — affects the other 15 files.

### 3. No regression test at the actual bug-path (`handleReviewJobSubmission`) — ⏭️ SKIPPED

**[File: apps/creative-portal/components/organisms/sidebars/contents/JobManagement/partials/Submission/hooks.ts]**

**Severity:** low

**Verified:** `Submission/` contains `consts.ts`, `hooks.ts`, `index.tsx`, `types.ts`, `utils.ts`, `utils.test.ts` — no `hooks.test.ts`. Hook tests *are* conventional in this folder tree (`useUpdateJobReturn.test.ts`, `useAddNewJob.test.ts`, `FormModal/hooks.test.ts`, `useReviewSubmissionFormState.test.tsx`), so the codebase does support this.

**Two things undercut the original "highest-value regression test" framing:**

1. **Almost nothing in the hook is new.** The diff replaces inline per-word logic with `...getApprovedQuantities(jobTask, approvedWorkTime)`. The division above it (`Math.ceil(Number(data.approvedWorkTime) / serviceJobTasks.length)`, `hooks.ts:150-152`) is **unchanged context**, not new code. The genuinely new code is a one-line spread delegating to a helper with 4 passing tests covering exactly the ticket's behaviour. Arguably `utils.test.ts` *is* the highest-value test for this ticket — and it's present.
2. **The test is more expensive than stated.** The suggested fix implies mocking `usePatchJobTaskMutation` alone. But `handleReviewJobSubmission` isn't returned from the hook — it's internal, reachable only via `onSubmit(data, true, actions)`, which first runs `validateSubmissionData`, `mutateOrder`, `prepareFiles`, and `startTracking`. A real test needs mocks for `usePatchOrder`, `usePatchJobTaskMutation`, `useUpdateJobMutation`, `useSidePanelContext`, `useWorkItemFiles`, `useUploadProgress`, and `useQueryClient`, plus an `order` fixture carrying both SERVICE and REVIEW jobs with `jobTasks` — roughly 100 lines of scaffolding to pin a one-line spread.

**Resolution:** ⏭️ **Skipped by decision.** Deferred as a follow-up. Not blocking — the billing rule is well covered, hook↔API parity was verified by inspection, and the fix is manually verified on orders 21174/21175.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx vitest run` (creative-portal, full) | ✅ **182 files, 1690 tests passed** | Includes the 4 helper tests and 3 display tests |
| `npx turbo run typecheck --filter=@proofed/creative-portal` | ✅ Pass | Re-run after the `jobTask` rename |
| `npx turbo run build --filter=@proofed/creative-portal` | ✅ Pass | |
| `npx eslint` (creative-portal) | ⚠️ 5 errors | **Unrelated / pre-existing.** All 5 are `prettier/prettier` in `components/molecules/JobReturnTimesTray/index.test.tsx` — untouched here, present on develop (from #2359). Touched files are clean. **develop is currently red on lint.** |

---

## Tests

- ✅ Helper unit tests — `utils.test.ts`, **4/4 passing**. Covers Fixed Rate, per-word, hourly, and mixed (fixed charge + hourly pay) tasks.
- ✅ Display tests — `JobSubmission.test.tsx`, **3/3 passing**. Verifies "Approved Time:" shows for hourly, is hidden for Fixed Rate (with "Editor's Work Time:" still shown), and stays hidden for per-word. Test premise verified: `totalApprovedWorkTime` derives from `approvedPayQuantity`, so the Fixed Rate case (`approvedPayQuantity: 1`) is truthy and would have shown under the old condition — the assertion is meaningful.
- ⏭️ No test at the `handleReviewJobSubmission` hook level — **skipped by decision** (Issue 3).
- ➖ E2E: not applicable (pricing-logic fix).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Fixes the root cause; helper mirrors the API guard; charge & pay resolved independently; hook↔API parity verified |
| Regression risk | ✅ Low — behaviour unchanged for per-word/hourly; only the fixed-rate path changes; import-source swap is value-equivalent |
| Tests | ✅ Helper + display covered and passing; hook-level gap knowingly deferred |
| Code quality | ✅ Good — DRY consolidation of 3 call sites; clean display refactor. The api→component import follows the house pattern (Issue 1) |
| Validation suite | ✅ Run — tests/typecheck/build green; lint red only on a pre-existing unrelated file |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve.** (Upgraded from "Approve with suggestions" — validation is now run and the actionable findings are applied.)

1. ~~Re-run the validation suite~~ → ✅ **Done.** 1690 tests ✅, typecheck ✅, build ✅. Lint's 5 errors are pre-existing and unrelated (`JobReturnTimesTray/index.test.tsx`, from #2359).
2. ~~Relocate `getApprovedQuantities`~~ (Issue 1) → ❌ **Withdrawn for this PR** — it follows the established `api/` → `components/` pattern and is cleaner than the closest precedent. Raise as a codebase-wide cleanup instead.
3. **Hook-level regression test** (Issue 3) → ⏭️ **Skipped by decision**, follow-up.
4. ~~Point `utils.test.ts` at the same `WORDS_UNIT_VALUE` source~~ (Issue 2) → ✅ **Done**, and extended to all three constants.
5. **Also applied from PR comments:** `task` → `jobTask` param rename across the helper and the hook's callback, aligning with the `JobTask` type and the two API call sites.

**Post-merge follow-ups worth raising separately:**

- **Dependency-direction cleanup (from Issue 1):** move `getApprovedQuantities`, `getJobTaskPayload`, and the other API-consumed component helpers to a neutral module so `api/` stops importing "up" into `components/`. Codebase-wide, ~8 files.
- **`settings/consts.tsx` should re-export unit constants from `@proofed/shared/config/units`** rather than redeclare them (Issue 2) — affects 15 files.
- **`Submission/hooks.ts` test coverage** (Issue 3).
- **develop lint is red** — `JobReturnTimesTray/index.test.tsx`, 5 auto-fixable prettier errors, from #2359.
- **PR description lists 3 files but the diff touches 7** — worth updating for the record.

The change is correct, targets the true root cause, and consolidates the divergent guards that let the bug exist.
