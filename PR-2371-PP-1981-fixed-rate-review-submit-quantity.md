# PR Review: fix/PP-1981: Keep Fixed Rate quoted quantity on reviewer submit

**PR:** https://github.com/Proofed/B2BWebserver/pull/2371
**Jira:** https://proofed.atlassian.net/browse/PP-1981
**Status:** Ready for Refinement (Bug, High)

> **Update (2026-07-09):** Issue 1 (guard logic duplicated in three places) has been **resolved** on the branch — both server routes now delegate to the single `getApprovedQuantities` helper. See the updated Issue 1 section and Validation Checks below.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Entered minutes must **not** become `approvedChargeQuantity` / `approvedPayQuantity` for fixed-price jobs | `getApprovedQuantities` guards on `useFixedChargeQuantity` / `useFixedPayQuantity`, keeping the quoted quantity instead of the work time | ✅ Addressed |
| Charge = fixed price × 1 (not × minutes) | Fixed Rate charge now uses `quotedChargeQuantity` (1); verified order 21175 = flat £65 vs 21174 = £11,710.83 | ✅ Addressed |
| Compensation = fixed pay × 1 | Same guard applied to `approvedPayQuantity` | ✅ Addressed |
| Behaviour should match per-word | Per-word was already clamped; fix extends the same clamp to Fixed Rate | ✅ Addressed |
| Approved-quantity field "should not be editable" for fixed-price orders | Fix **ignores** the field's value for billing rather than **disabling** the input; the editable field is still rendered | ⚠️ Partial — deferred to follow-up (see Issue 2) |

**Scope note:** The PR is tightly scoped to the reviewer-submit billing path plus the guard de-duplication. It does **not** disable the approved-work-time input for fixed-price jobs (the ticket's "not editable" clause) — see Issue 2.

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

### 2. Ticket's "field should not be editable" clause is not implemented

**[File: apps/creative-portal/components/organisms/sidebars/contents/JobManagement/partials/Submission/*]**

**Function/Class:** ServiceSubmission (editor's work-time input)

**Severity:** low

**Problem:** The Jira Expected Result says the approved-quantity field "should not be editable and should default to 1" for fixed-price orders. This PR neutralises the field's effect on billing but still renders it as an editable input for Fixed Rate jobs. A disable-the-input change was prototyped and then reverted to keep this PR scoped to the revenue bug.

**Impact:** No revenue impact (value is ignored for billing). Purely UX — a reviewer/editor can still type a work time that has no effect, which could be confusing.

**Fix:** Follow-up — disable the work-time input for fixed-quantity jobs on the job-side panel (excluding per-word, which keeps its informational time entry), relaxing the `serviceWorkTime > 0` validation for that case so a disabled 0 field does not silently block submit. The order-side panel already handles Fixed Rate gracefully (field marked "(optional)" + informational note, validation already relaxed). Reasonable to defer since the revenue-impacting behaviour is resolved.

### 3. "Approved Time" informational displays now show quoted quantity for Fixed Rate jobs

**[File: apps/creative-portal/utils/calculateJobTasksWorkAndPayTotals.ts (consumer, unchanged)]**

**Function/Class:** consumers of `totalApprovedWorkTime` (ServiceHistoryPreview, ReviewHistoryPreview, OrderJobs/JobSubmission)

**Severity:** low

**Problem:** `approvedPayQuantity` doubles as the source of `totalApprovedWorkTime`, which several surfaces render as the editor's "Approved Time". For Fixed Rate jobs this value changes from the entered minutes (e.g. `3h 0m`) to the quoted quantity (e.g. `0h 1m`).

**Impact:** Cosmetic only — the real editor work time is preserved separately in `userEnteredQuantity` ("Service time" / "Editor's Work Time" rows are unaffected). No money or data loss.

**Fix:** Optional follow-up — exclude fixed-quantity tasks from `totalApprovedWorkTime` so the "Approved Time" row hides for Fixed Rate, matching per-word. Intentionally deferred to keep this PR minimal.

### 4. Undefined quoted quantity passes through unchanged (pre-existing behaviour)

**[File: apps/creative-portal/components/organisms/sidebars/contents/JobManagement/partials/Submission/utils.ts]**

**Function/Class:** getApprovedQuantities

**Severity:** low

**Problem:** When the guard is taken and `quotedChargeQuantity` / `quotedPayQuantity` is `undefined`, the returned field is `undefined` and forwarded to `patchJobTask` / `mutateJobTask`. This matches the previous inline behaviour, so it is not a regression, but the helper does not defend against it.

**Impact:** None in practice — Fixed Rate and per-word tasks always have quoted quantities set at order creation. Noted for completeness.

**Fix:** No change required. Out of scope.

---

## Validation Checks

Re-run on the PR branch `fix/PP-1981-fixed-rate-review-submit-quantity` after the Issue 1 de-duplication.

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⚠️ PR-scope pass, monorepo ❌ | creative-portal passes incl. the helper unit tests (4/4); **1 pre-existing unrelated failure** in `packages/shared/utils/formatWordQuantity.test.ts` (locale digit-grouping `10,00,000` vs `1,000,000`), present on develop, untouched here |
| `npx turbo run typecheck` | ✅ | 0 errors across all workspaces |
| `npx turbo run lint` | ⚠️ PR-scope pass, monorepo ❌ | The changed files pass eslint (exit 0, after `--fix` removed the now-unused `WORDS_UNIT_VALUE` import, corrected import order, and dropped a trailing comma); **1 pre-existing unrelated failure** — prettier in `components/molecules/JobReturnTimesTray/index.test.tsx`, present on develop, untouched here |
| `npx turbo run build` | ✅ | creative-portal build clean — the cross-layer helper import resolves and bundles without issue |

Both failing checks fail on **develop** in files this PR does not touch. Per PR scope discipline they were left unfixed; they are pre-existing infrastructure/locale issues, not introduced by this PR.

---

## Tests

- ✅ Unit tests for `getApprovedQuantities` (`Submission/utils.test.ts`, 4 cases): Fixed Rate, per-word, hourly, and mixed charge/pay tasks — now also exercise the exact logic used by both server routes
- ✅ Meets the "every PR must include tests" requirement
- ✅ Manual reproduction + verification: order 21174 inflated to £11,710.83; order 21175 fixed at flat £65 (Fixed Rate, Qty 1)
- ⚠️ No dedicated integration test for the two server routes end-to-end (logic isolated into the tested helper instead)
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

1. **Merge-ready for the revenue bug** — the fix is correct, minimal, root-caused, tested, and manually verified (charge stays flat for Fixed Rate). Issue 1 is now resolved with a single shared guard.
2. **Confirm the two failing gate checks are the known pre-existing develop failures** (`formatWordQuantity` locale test, `JobReturnTimesTray` prettier) and not a merge blocker — they exist independently in untouched files.
3. **Optional cleanup (Issue 1 caveat):** relocate `getApprovedQuantities` to `packages/shared/utils` so the API layer does not import from a UI-component path.
4. **Consider a follow-up ticket** for the ticket's "field should not be editable" clause (Issue 2) and the cosmetic "Approved Time" display for Fixed Rate jobs (Issue 3) — both low severity, no revenue impact.
