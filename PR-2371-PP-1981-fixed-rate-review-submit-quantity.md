# PR Review: fix/PP-1981: Keep Fixed Rate quoted quantity on reviewer submit

**PR:** https://github.com/Proofed/B2BWebserver/pull/2371
**Jira:** https://proofed.atlassian.net/browse/PP-1981
**Status:** Ready for Refinement (Bug, High)

> **Update (2026-07-09):**
> - **Issue 1** (guard logic duplicated in three places): **resolved** — both server routes now delegate to the single `getApprovedQuantities` helper.
> - **Issue 2** (field should not be editable): a disable-the-input change was implemented on both panels and then **discarded per the client's decision** — the Work Time field stays editable for Fixed Rate jobs. The value has no billing effect (already neutralised by the Issue-1 guard), so this is a deliberate product choice, not an open defect.
> - **Issue 3** ("Approved Time" shows the quoted quantity for Fixed Rate): **resolved** — the "Approved Time" row is now hidden for Fixed Rate jobs (matching per-word), so it no longer renders a meaningless `0H 1M`.
>
> See the updated Issue 1 / Issue 2 / Issue 3 sections and Validation Checks below.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Entered minutes must **not** become `approvedChargeQuantity` / `approvedPayQuantity` for fixed-price jobs | `getApprovedQuantities` guards on `useFixedChargeQuantity` / `useFixedPayQuantity`, keeping the quoted quantity instead of the work time | ✅ Addressed |
| Charge = fixed price × 1 (not × minutes) | Fixed Rate charge now uses `quotedChargeQuantity` (1); verified order 21175 = flat £65 vs 21174 = £11,710.83 | ✅ Addressed |
| Compensation = fixed pay × 1 | Same guard applied to `approvedPayQuantity` | ✅ Addressed |
| Behaviour should match per-word | Per-word was already clamped; fix extends the same clamp to Fixed Rate | ✅ Addressed |
| Approved-quantity field "should not be editable" for fixed-price orders | Field stays editable; its value has no billing effect (neutralised by the Issue-1 guard). Disabling was implemented then **discarded per client decision** | ⚠️ Deferred by client (see Issue 2) |

**Scope note:** The PR covers the reviewer-submit billing fix and the guard de-duplication (Issue 1). The "not editable" clause (Issue 2) was intentionally left out per the client's decision. No unrelated refactors.

---

## Architecture Analysis

The bug: on reviewer submission, `handleReviewJobSubmission` set `approvedChargeQuantity` / `approvedPayQuantity` to the editor's entered work-time minutes for **every non per-word** task. For a Fixed Rate task (`chargeUnit = 1`, `useFixedChargeQuantity = true`) the .NET charge API then computed `rate × minutes` (e.g. `£65 × 180 = £11,700`), because the frontend handed it the wrong quantity.

**Why only the job-side panel was affected.** Both submission surfaces write the same two quantity fields, but via different code paths:

- **Order-side / Admin panel (OrderManagement)** → `useJobSubmitMutation` → the server helpers `api/utils/jobs/postSubmitJob.ts` / `postSubmitJobStream.ts`, which **already contained the full guard** (`isChargePerWord || useFixedChargeQuantity ? quoted : approved`). Fixed Rate billed correctly here.
- **Job-side panel (JobManagement) reviewer submit** → `Submission/hooks.ts` `handleReviewJobSubmission` → a **direct client-side `mutateJobTask` PATCH** carrying its **own inline copy** of the quantity logic that only guarded per-word (`isChargePerWord ? quoted : approved`) and omitted the `useFixed*` clause. This was the sole writer missing the guard, so the same Fixed Rate order inflated only when submitted from the job-side panel.

The fix extracts the resolution into a `getApprovedQuantities` helper that mirrors the server guard. As of the 2026-07-09 update, the two server routes were refactored to import that same helper, so **all three writers now share one implementation** (root-cause fix + single source of truth).

---

## Issues Found

### 1. Guard logic duplicated in three places — ✅ RESOLVED

**[File: apps/creative-portal/api/utils/jobs/postSubmitJob.ts, postSubmitJobStream.ts]**

**Function/Class:** getApprovedQuantities

**Severity:** low

**Problem (original):** The `isPerWord || useFixed* ? quoted : entered` guard lived in three separate implementations: the new `getApprovedQuantities` helper, `postSubmitJob.ts`, and `postSubmitJobStream.ts`. A future change to the rule would have to be made in all three.

**Resolution:** Both server routes now import and call `getApprovedQuantities(jobTask, approvedQuantity)` instead of inlining the guard. The per-task `approvedQuantity = Math.ceil(approvedWorkTime / totalJobTask)` division stays at each call site; only the charge/pay resolution moved into the shared helper. The now-unused `WORDS_UNIT_VALUE` imports were removed from both routes. Result: one guard implementation, three call sites — verified green (typecheck, lint, build, unit tests all pass).

**Remaining caveat (low, non-blocking):** The dependency direction is backwards — the server-side API utils (`api/utils/jobs/...`) now import from a deep UI-component path (`components/organisms/sidebars/contents/JobManagement/partials/Submission/utils`). It compiles and bundles cleanly, but the shared helper would ideally live in a neutral location (`packages/shared/utils` or a local `api/utils/jobs/` helper) so the API layer does not reach into a React component's internals. Optional cleanup; no functional impact.

### 2. Ticket's "field should not be editable" clause — ⚠️ Deferred by client

**[File: ServiceSubmission/index.tsx, EditingForm/index.tsx (Work Time input)]**

**Function/Class:** ServiceSubmission + EditingForm (editor's Work Time input)

**Severity:** low

**Problem:** The Jira Expected Result says the field "should not be editable" for fixed-price orders. The billing fix neutralises the field's effect on billing but still renders it as an editable input for Fixed Rate jobs.

**Status:** A disable-the-input change (job-side `disabled={isFixedRateJob}` + a `serviceWorkTime > 0` validation relaxation, and order-side `disabled={disabled || isFixedRateJob}`) was implemented on both panels and then **reverted per the client's decision** — the field remains editable. Because the value is already ignored for billing (the Issue-1 guard sends the quoted quantity), leaving it editable carries **no revenue impact**; this is a deliberate product choice.

**Impact:** Purely UX — a reviewer/editor can still type a work time that has no billing effect. No money impact.

**Fix:** None planned (deferred by client). Could be revisited in a follow-up if the "not editable" UX is later requested.

### 3. "Approved Time" showed the quoted quantity for Fixed Rate jobs — ✅ RESOLVED

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderJobs/partials/JobSubmission.tsx]**

**Function/Class:** JobSubmission (Submission accordion — "Approved Time" column)

**Severity:** low

**Problem (original):** `approvedPayQuantity` doubles as the source of `totalApprovedWorkTime`, which the Submission preview renders as "Approved Time". For a Fixed Rate job that value is the quoted billing quantity (Qty 1), so it displayed as a meaningless `0H 1M`. The show-condition excluded per-word (`!isPerWordJob`) but not Fixed Rate.

**Resolution:** Added an `isFixedRateJob` flag (`useFixedChargeQuantity && useFixedPayQuantity && !isPerWordJob`) and factored a single `hasApprovedTimeColumn` boolean (`isServiceJobOrReviewJobType && !!totalApprovedWorkTime && !isPerWordJob && !isFixedRateJob`). It now gates the "Approved Time" column **and** the Score placement (`shouldShowScore` / `shouldShowScoreInline`), so the Score stays correctly positioned when the column is hidden. The real editor work time ("Editor's Work Time", from `userEnteredQuantity`) is unaffected.

**Impact:** Fixed Rate jobs no longer show the spurious `0H 1M` "Approved Time"; behaviour now matches per-word. No money or data impact.

### 4. Undefined quoted quantity passes through unchanged (pre-existing behaviour)

**[File: apps/creative-portal/components/organisms/sidebars/contents/JobManagement/partials/Submission/utils.ts]**

**Function/Class:** getApprovedQuantities

**Severity:** low

**Problem:** When the guard is taken and `quotedChargeQuantity` / `quotedPayQuantity` is `undefined`, the returned field is `undefined` and forwarded to `patchJobTask` / `mutateJobTask`. This matches the previous inline behaviour, so it is not a regression, but the helper does not defend against it.

**Impact:** None in practice — Fixed Rate and per-word tasks always have quoted quantities set at order creation. Noted for completeness.

**Fix:** No change required. Out of scope.

---

## Validation Checks

Run on the PR branch `fix/PP-1981-fixed-rate-review-submit-quantity` (billing fix + Issue 1 de-duplication + Issue 3 "Approved Time" hide; Issue 2 reverted).

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⚠️ PR-scope pass, monorepo ❌ | creative-portal passes incl. the `getApprovedQuantities` helper tests (4/4) and the OrderJobs suite (122/122, covering `JobSubmission`'s siblings); **1 pre-existing unrelated failure** in `packages/shared/utils/formatWordQuantity.test.ts` (locale digit-grouping), present on develop, untouched here |
| `npx turbo run typecheck` | ✅ | 0 errors across all workspaces |
| `npx turbo run lint` | ⚠️ PR-scope pass, monorepo ❌ | The changed files pass eslint (exit 0); **1 pre-existing unrelated failure** — prettier in `components/molecules/JobReturnTimesTray/index.test.tsx`, present on develop, untouched here |
| `npx turbo run build` | ✅ | creative-portal build clean |

Both failing checks fail on **develop** in files this PR does not touch. Per PR scope discipline they were left unfixed; they are pre-existing infrastructure/locale issues, not introduced by this PR.

---

## Tests

- ✅ Unit tests for `getApprovedQuantities` (`Submission/utils.test.ts`, 4 cases): Fixed Rate, per-word, hourly, and mixed charge/pay tasks — also exercise the exact logic used by both server routes
- ✅ Meets the "every PR must include tests" requirement
- ✅ Manual reproduction + verification: order 21174 inflated to £11,710.83; order 21175 fixed at flat £65 (Fixed Rate, Qty 1)
- ⚠️ No dedicated unit test for `JobSubmission`'s `hasApprovedTimeColumn` gate (component has no existing test harness; change mirrors the existing untested `!isPerWordJob` guard)
- ⚠️ No test covers the `undefined` quoted-quantity edge (Issue 4) — acceptable given it matches prior behaviour

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Fixes root cause; manually verified |
| Regression risk | ✅ Low — only Fixed Rate/per-word quantity changes; hourly unchanged; all three writers now share one guard |
| Tests | ✅ Shared helper unit-tested |
| Code quality | ✅ Clean single-source guard; minor import-direction caveat noted (Issue 1) |
| Validation suite | ⚠️ PR files pass; 2 pre-existing unrelated monorepo failures (locale test + prettier in untouched files) |
| Mergeable state | ✅ Clean (no conflicts) |

---

## Recommendation

**Approve with suggestions**

1. **Merge-ready** — the fix is correct, root-caused, tested, and manually verified (charge stays flat for Fixed Rate). Issues 1 (single shared guard) and 3 ("Approved Time" hidden for Fixed Rate) are resolved. Issue 2 (Work Time "not editable") was **deferred per the client's decision** — the field stays editable but has no billing effect.
2. **Confirm the two failing gate checks are the known pre-existing develop failures** (`formatWordQuantity` locale test, `JobReturnTimesTray` prettier) and not a merge blocker — they exist independently in untouched files.
3. **Optional cleanup (Issue 1 caveat):** relocate `getApprovedQuantities` to `packages/shared/utils` so the API layer does not import from a UI-component path.
