# PR Review: fix/PP-1981: Keep Fixed Rate quoted quantity on reviewer submit

**PR:** https://github.com/Proofed/B2BWebserver/pull/2371
**Jira:** https://proofed.atlassian.net/browse/PP-1981
**Status:** Ready for Refinement (Bug, High)

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Entered minutes must **not** become `approvedChargeQuantity` / `approvedPayQuantity` for fixed-price jobs | `getApprovedQuantities` guards on `useFixedChargeQuantity` / `useFixedPayQuantity`, keeping the quoted quantity instead of the work time | ✅ Addressed |
| Charge = fixed price × 1 (not × minutes) | Fixed Rate charge now uses `quotedChargeQuantity` (1); verified order 21175 = flat £65 vs 21174 = £11,710.83 | ✅ Addressed |
| Compensation = fixed pay × 1 | Same guard applied to `approvedPayQuantity` | ✅ Addressed |
| Behaviour should match per-word | Per-word was already clamped; fix extends the same clamp to Fixed Rate | ✅ Addressed |
| Approved-quantity field "should not be editable" for fixed-price orders | Fix **ignores** the field's value for billing rather than **hiding** the input; the editable field is still rendered in the review form | ⚠️ Partial |

**Scope note:** The PR is tightly scoped to the reviewer-submit billing path. It does **not** hide the approved-work-time input for fixed-price jobs (the ticket's "not editable" clause) — see Issue 2. No scope creep / bonus refactors beyond the extraction of the guard into a helper.

---

## Architecture Analysis

The bug: on reviewer submission, `handleReviewJobSubmission` set `approvedChargeQuantity` / `approvedPayQuantity` to the editor's entered work-time minutes for **every non per-word** task. For a Fixed Rate task (`chargeUnit = 1`, `useFixedChargeQuantity = true`) the .NET charge API then computed `rate × minutes` (e.g. `£65 × 180 = £11,700`), because the frontend handed it the wrong quantity.

The two server-side submission routes — `api/utils/jobs/postSubmitJob.ts` and `postSubmitJobStream.ts` — already contain the correct guard (`isChargePerWord || jobTask.useFixedChargeQuantity ? quoted : approved`). This parallel **frontend** path was the only writer missing it. The fix extracts the resolution into a `getApprovedQuantities` helper that mirrors the server guard, so the client sends the quoted quantity for Fixed Rate (and per-word) tasks and the entered work time only for genuinely hourly tasks.

Root-cause fix (correct quantity at source), not a symptom patch. Extraction to a pure helper enables unit testing without mocking the 6+ hook dependencies.

---

## Issues Found

### 1. Guard logic is now duplicated in three places

**[File: apps/creative-portal/components/organisms/sidebars/contents/JobManagement/partials/Submission/utils.ts]**

**Function/Class:** getApprovedQuantities

**Severity:** low

**Problem:** The `isPerWord || useFixed* ? quoted : entered` guard now lives in three separate implementations: this new helper, `api/utils/jobs/postSubmitJob.ts`, and `postSubmitJobStream.ts`. A future change to the rule must be made in all three.

**Impact:** Maintenance / drift risk only — no functional defect. The three copies are currently consistent.

**Fix:** Acceptable for this PR (the API copies operate on a different data shape and layer). If the team wants to converge, a shared `packages/shared/utils` helper parameterised over `{ chargeUnit, payUnit, useFixedChargeQuantity, useFixedPayQuantity, quotedChargeQuantity, quotedPayQuantity }` could back all three. Not a blocker.

### 2. Ticket's "field should not be editable" clause is not implemented

**[File: apps/creative-portal/components/organisms/sidebars/contents/JobManagement/partials/Submission/*]**

**Function/Class:** ReviewSubmission / ReviewForm (approved-work-time input)

**Severity:** low

**Problem:** The Jira Expected Result says the approved-quantity field "should not be editable and should default to 1" for fixed-price orders. This PR neutralises the field's effect on billing but still renders it as an editable input for Fixed Rate jobs.

**Impact:** No revenue impact (value is ignored for billing). Purely UX — a reviewer can still type a work time that has no effect, which could be confusing.

**Fix:** Follow-up — hide/disable the approved-work-time input for fixed-quantity jobs (mirror the `isFixedQuantityJob` guard already used in the OrderManagement `ReviewForm`). Reasonable to defer since the revenue-impacting behaviour is resolved.

### 3. "Approved Time" informational displays now show quoted quantity for Fixed Rate jobs

**[File: apps/creative-portal/utils/calculateJobTasksWorkAndPayTotals.ts (consumer, unchanged)]**

**Function/Class:** consumers of `totalApprovedWorkTime` (ServiceHistoryPreview, ReviewHistoryPreview, OrderJobs/JobSubmission)

**Severity:** low

**Problem:** `approvedPayQuantity` doubles as the source of `totalApprovedWorkTime`, which several surfaces render as the editor's "Approved Time". For Fixed Rate jobs this value changes from the entered minutes (e.g. `3h 0m`) to the quoted quantity (e.g. `0h 1m`).

**Impact:** Cosmetic only — the real editor work time is preserved separately in `userEnteredQuantity` ("Service time" / "Editor's Work Time" rows are unaffected). No money or data loss.

**Fix:** Optional follow-up — exclude fixed-quantity tasks from `totalApprovedWorkTime` (they carry a billing quantity, not a work time) so the "Approved Time" row hides for Fixed Rate, matching per-word. A one-line change to the totals reducer was explored and intentionally deferred to keep this PR minimal.

### 4. Undefined quoted quantity passes through unchanged (pre-existing behaviour)

**[File: apps/creative-portal/components/organisms/sidebars/contents/JobManagement/partials/Submission/utils.ts]**

**Function/Class:** getApprovedQuantities

**Severity:** low

**Problem:** When the guard is taken and `quotedChargeQuantity` / `quotedPayQuantity` is `undefined`, the returned field is `undefined` and forwarded to `mutateJobTask`. This matches the previous inline behaviour, so it is not a regression, but the helper does not defend against it.

**Impact:** None in practice — Fixed Rate and per-word tasks always have quoted quantities set at order creation. Noted for completeness.

**Fix:** No change required. Could fall back to `approvedWorkTime` if desired, but that would alter established behaviour and is out of scope.

---

## Validation Checks

Run this session on the PR branch `fix/PP-1981-fixed-rate-review-submit-quantity` (not re-run for this review — identical branch state).

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⚠️ PR-scope pass, monorepo ❌ | creative-portal passes incl. the 4 new tests; **1 pre-existing unrelated failure** in `packages/shared/utils/formatWordQuantity.test.ts` (locale digit-grouping `10,00,000` vs `1,000,000`), present on develop, untouched here |
| `npx turbo run typecheck` | ✅ | 0 errors across all 5 workspaces |
| `npx turbo run lint` | ⚠️ PR-scope pass, monorepo ❌ | The 3 changed files pass eslint (exit 0); **1 pre-existing unrelated failure** — prettier in `components/molecules/JobReturnTimesTray/index.test.tsx`, present on develop, untouched here |
| `npx turbo run build` | ✅ | creative-portal build clean |

Both failing checks fail on **develop** in files this PR does not touch (verified by extracting develop's copies). Per PR scope discipline they were left unfixed; they are pre-existing infrastructure/locale issues, not introduced by this PR.

---

## Tests

- ✅ New unit tests added (`Submission/utils.test.ts`, 4 cases): Fixed Rate, per-word, hourly, and mixed charge/pay tasks
- ✅ Meets the "every PR must include tests" requirement
- ✅ Manual reproduction + verification: order 21174 inflated to £11,710.83; order 21175 fixed at flat £65 (Fixed Rate, Qty 1)
- ⚠️ No test covers the `undefined` quoted-quantity edge (Issue 4) — acceptable given it matches prior behaviour
- ⚠️ No automated test for the reviewer-submit hook end-to-end (logic isolated into the tested helper instead)

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Fixes root cause; manually verified |
| Regression risk | ✅ Low — only Fixed Rate/per-word quantity changes; hourly unchanged; all price readers already guarded |
| Tests | ✅ New helper unit-tested |
| Code quality | ✅ Clean extraction, mirrors existing server guard |
| Validation suite | ⚠️ PR files pass; 2 pre-existing unrelated monorepo failures (locale test + prettier in untouched files) |
| Mergeable state | ✅ Clean (no conflicts) |

---

## Recommendation

**Approve with suggestions**

1. **Merge-ready for the revenue bug** — the fix is correct, minimal, root-caused, tested, and manually verified (charge stays flat for Fixed Rate).
2. **Confirm the two failing gate checks are the known pre-existing develop failures** (`formatWordQuantity` locale test, `JobReturnTimesTray` prettier) and not a merge blocker for this PR — they exist independently in untouched files.
3. **Consider a follow-up ticket** for the ticket's "field should not be editable" clause (Issue 2) and the cosmetic "Approved Time" display for Fixed Rate jobs (Issue 3) — both low severity, no revenue impact.
