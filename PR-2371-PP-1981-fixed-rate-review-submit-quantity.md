# PR Review: fix/PP-1981: Keep Fixed Rate quoted quantity on reviewer submit

**PR:** https://github.com/Proofed/B2BWebserver/pull/2371
**Jira:** https://proofed.atlassian.net/browse/PP-1981
**Status:** In Progress (Bug, High)

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Approved work time must **not** be applied as the approved charge/pay quantity for fixed-price jobs | `getApprovedQuantities` keeps `quotedChargeQuantity`/`quotedPayQuantity` when `useFixedChargeQuantity`/`useFixedPayQuantity` (or per-word) is set; entered minutes only for genuinely hourly tasks | ✅ Addressed |
| Total Charge = fixed price × 1 | Fixed Rate charge now uses the quoted quantity (1); verified order 21175 = flat £65 vs 21174 = £11,710.83 | ✅ Addressed |
| Compensation = fixed pay × 1 | Same guard applied to `approvedPayQuantity` | ✅ Addressed |
| Fixed price jobs should follow the **Per Word model for Approved Quantities** | The reviewer-submit path and both server routes now share one guard that treats Fixed Rate exactly like per-word (quoted quantity retained) | ✅ Addressed |
| Fixed price jobs should follow the Per Word model for **display rules** | `JobSubmission` "Approved Time" column is now hidden for Fixed Rate jobs (via `hasApprovedTimeColumn`), matching per-word | ✅ Addressed |
| Approved-quantity field **should not be editable** and should default to 1 | Billing is corrected, but the editor Work Time input **remains editable** for Fixed Rate jobs — the disable-the-input change was implemented and then reverted per the client's decision | ⚠️ Deferred by client (see Issue 2) |

**Scope note:** The PR is scoped to (a) the reviewer-submit billing fix, (b) de-duplicating the guard into one helper shared by both server routes, and (c) hiding the "Approved Time" display for Fixed Rate jobs. The ticket's literal "not editable" clause is intentionally out of scope per the client. No unrelated refactors.

---

## Architecture Analysis

The bug: on reviewer submission, `handleReviewJobSubmission` set `approvedChargeQuantity`/`approvedPayQuantity` to the editor's entered work-time minutes for **every non per-word** task. For a Fixed Rate task the .NET charge API then computed `rate × minutes` (e.g. `£65 × 180 = £11,700`), because the frontend handed it the wrong quantity.

Two submission surfaces write these quantities via different paths:

- **Order-side / Admin panel** → server helpers `postSubmitJob.ts` / `postSubmitJobStream.ts`, which already had the full `isPerWord || useFixed* ? quoted : approved` guard.
- **Job-side reviewer submit** → `Submission/hooks.ts` `handleReviewJobSubmission`, which had its **own inline copy** that only guarded per-word and omitted the `useFixed*` clause — the sole writer missing the guard.

The fix extracts the resolution into a pure `getApprovedQuantities(task, approvedWorkTime)` helper and points **all three** writers at it (the reviewer hook plus both server routes), so the rule has one implementation. Separately, `JobSubmission.tsx` gains an `isFixedRateJob` flag and a factored `hasApprovedTimeColumn` boolean so the "Approved Time" column (which reads `approvedPayQuantity` = the quoted quantity = a billing quantity, not a work time) is hidden for Fixed Rate, matching per-word — and the Score placement stays consistent when the column disappears.

Root-cause fix at the source (correct quantity sent to the API), not a symptom patch. Extraction to a pure helper enables unit testing without mocking the hook's dependencies.

---

## Issues Found

### 1. Guard helper is imported by the API layer from a UI-component path

**[File: apps/creative-portal/api/utils/jobs/postSubmitJob.ts, postSubmitJobStream.ts]**

**Function/Class:** getApprovedQuantities

**Severity:** low

**Problem:** Both server routes import `getApprovedQuantities` from `components/organisms/sidebars/contents/JobManagement/partials/Submission/utils` — the API layer reaching into a deep React-component folder. The dependency direction is backwards (server → UI component), and the helper's home implies it is job-sidebar UI code.

**Impact:** Maintainability only — it compiles, type-checks, and bundles cleanly (build is green). But the location is misleading and couples the API routes to a component folder; a future move/rename of the Submission component would ripple into server code.

**Fix:** Relocate the helper to a neutral module — `packages/shared/utils` (parameterised over the primitive fields), or a local `apps/creative-portal/api/utils/jobs/` helper — and have all three call sites import from there. Non-blocking; can be a follow-up.

### 2. Ticket's "approved-quantity field should not be editable" clause is not implemented

**[File: apps/creative-portal/components/organisms/sidebars/contents/JobManagement/partials/ServiceSubmission/index.tsx (and order-side EditingForm)]**

**Function/Class:** ServiceSubmission / EditingForm (editor Work Time input)

**Severity:** low

**Problem:** The Jira Expected Result states the approved-quantity field "should not be editable and should default to 1" for fixed-price orders. This PR corrects billing (the value is ignored) but still renders the Work Time input as editable for Fixed Rate jobs. A disable-the-input change was implemented on both panels and then reverted per the client's decision.

**Impact:** Purely UX — a reviewer/editor can type a work time that has no billing effect. No revenue impact (the Issue-1 guard already sends the quoted quantity).

**Fix:** None planned (client-deferred). If the "not editable" UX is later requested, re-apply the disable on the Work Time input plus the `serviceWorkTime > 0` validation relaxation for the job-side (the order-side validation already skips the rule for fixed-quantity jobs).

### 3. New `JobSubmission` display logic has no unit test

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderJobs/partials/JobSubmission.tsx]**

**Function/Class:** JobSubmission — `hasApprovedTimeColumn` / `isFixedRateJob`

**Severity:** low

**Problem:** The reviewer-submit helper (`getApprovedQuantities`) is well covered (4 cases), but the `JobSubmission` change — hiding "Approved Time" for Fixed Rate and the derived Score-placement booleans — has no test. The component has no existing test harness, so the branch is unverified by automation.

**Impact:** Low — the change mirrors the existing untested `!isPerWordJob` guard, and the Score-layout booleans were refactored to a single source (`hasApprovedTimeColumn`) which reduces the risk of divergence. But a regression that re-shows "Approved Time" for Fixed Rate (or misplaces the Score) would not be caught.

**Fix:** Optional — add a focused render test asserting the "Approved Time" column is absent for a Fixed Rate job and present for an hourly one, plus the Score inline/new-line placement. Requires mocking `DescriptionList`/`Accordion`; reasonable to defer given the low blast radius.

### 4. `getApprovedQuantities` forwards `undefined` when the quoted quantity is missing

**[File: apps/creative-portal/components/organisms/sidebars/contents/JobManagement/partials/Submission/utils.ts]**

**Function/Class:** getApprovedQuantities

**Severity:** low

**Problem:** When the guard is taken and `quotedChargeQuantity`/`quotedPayQuantity` is `undefined`, the returned field is `undefined` and forwarded to `mutateJobTask` / `patchJobTask`. The helper does not defend against it.

**Impact:** None in practice — Fixed Rate and per-word tasks always have quoted quantities set at order creation. This matches the previous inline behaviour, so it is not a regression. Noted for completeness.

**Fix:** No change required.

---

## Validation Checks

Run in place on the PR branch `fix/PP-1981-fixed-rate-review-submit-quantity` (HEAD `4da4d3a75`), clean working tree.

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⚠️ PR-scope pass, monorepo ❌ | creative-portal passes incl. `getApprovedQuantities` tests (4/4); shared 1318/1319 — **1 pre-existing unrelated failure** in `packages/shared/utils/formatWordQuantity.test.ts` (locale digit-grouping `1,000,000`), present on develop, untouched here |
| `npx turbo run typecheck` | ✅ | 0 errors across all 5 workspaces |
| `npx turbo run lint` | ⚠️ PR-scope pass, monorepo ❌ | The PR's 6 files pass eslint; **5 pre-existing prettier errors** all in `components/molecules/JobReturnTimesTray/index.test.tsx`, present on develop, untouched here |
| `npx turbo run build` | ✅ | creative-portal build clean (the cross-layer helper import resolves and bundles) |

Both failing checks fail on **develop** in files this PR does not touch. Per PR scope discipline they were left unfixed; they are pre-existing locale/formatting issues, not introduced by this PR.

---

## Tests

- ✅ Unit tests for `getApprovedQuantities` (`Submission/utils.test.ts`, 4 cases): Fixed Rate, per-word, hourly, and mixed charge/pay — also exercise the exact logic now used by both server routes
- ✅ Meets the "every PR must include tests" requirement for the core fix
- ✅ Manual reproduction + verification: order 21174 inflated to £11,710.83; order 21175 fixed at flat £65 (Fixed Rate, Qty 1)
- ⚠️ No test for the `JobSubmission` "Approved Time" display change (Issue 3) — component has no existing harness
- ⚠️ No test covers the `undefined` quoted-quantity edge (Issue 4) — acceptable, matches prior behaviour

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Fixes root cause; manually verified (flat £65) |
| Regression risk | ✅ Low — only Fixed Rate/per-word quantity changes; hourly unchanged; all three writers now share one guard |
| Tests | ⚠️ Core helper well tested; `JobSubmission` display change untested |
| Code quality | ✅ Clean single-source guard; low import-direction caveat (Issue 1) |
| Validation suite | ⚠️ PR files pass; 2 pre-existing unrelated monorepo failures (locale test + prettier, untouched files) |
| Mergeable state | ✅ Clean (GitHub `mergeable_state: clean`, no conflicts) |

---

## Recommendation

**Approve with suggestions**

1. **Merge-ready for the revenue bug** — the fix is correct, root-caused, tested, and manually verified (charge stays flat for Fixed Rate). Guard is now single-sourced across all three writers, and the "Approved Time" display matches per-word.
2. **Confirm the two failing gate checks are the known pre-existing develop failures** (`formatWordQuantity` locale test, `JobReturnTimesTray` prettier) and not a merge blocker — they exist independently in untouched files.
3. **Note the deferred Jira clause (Issue 2)** — the ticket's literal "field should not be editable / default to 1" is not implemented (client decision). Consider recording this on the ticket so QA doesn't flag it as a regression, or track it as a follow-up.
4. **Optional follow-ups:** relocate `getApprovedQuantities` out of the UI-component folder (Issue 1); add a `JobSubmission` display test (Issue 3).
