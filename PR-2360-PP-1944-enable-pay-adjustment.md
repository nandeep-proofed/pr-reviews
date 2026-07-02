# PR Review: feature/PP-1944: Enable Pay adjustment type in Adjust Pay or Charge modal

**PR:** https://github.com/Proofed/B2BWebserver/pull/2360
**Jira:** https://proofed.atlassian.net/browse/PP-1944
**Status:** Code Review

> **Re-verified against PR head `b34cc678`.** Issue #1 has since been **fixed**; Issues #2–#4 are **Skipped** (accepted low-priority). Two changes were added to the PR after the initial review — the **negative-Pay-adjustment cap** (job pay + existing adjustments, via a new Compensation search GET route) and the **Rate `type="text"` decimal-display** fix — both verified green (see the note at the end of Issues Found).

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1.1 Add "Pay" to Adjustment Type dropdown in both flows | `adjustmentTypeOptions` now `[Charge, Pay]`; selectable in both the order and No Order flows | ✅ Addressed |
| 1.2 Charge remains the default | No type pre-selected — opens on the "Select Adjustment Type" placeholder | ⚠️ Intentional deviation — design's first-open placeholder supersedes req 1.2 (confirmed with team) |
| 2.1 Currency always USD, not changeable | `ChargeCurrencySync` forces `chargeCurrencyCode = "USD"` for Pay; no currency selector | ✅ Addressed |
| 2.2 Amount auto-calculated, not directly editable | Auto-calc via `PayAmountCalculator`; the Amount input is now **read-only** for Pay (`isAmountReadOnly`) | ✅ Addressed (Issue #1 fixed) |
| 2.3 +/- toggle sets sign; negative allowed, zero rejected | `applyAmountSign` preserves the toggle sign; schema rejects 0 | ✅ Addressed |
| 2.4 Quantity written to all 3 quantity fields | `buildCompensationPayload` writes Quoted/UserEntered/Approved | ✅ Addressed |
| 2.5 Description = Reason (+ details), 200 max, colon/pipe per flow | `composeAdjustmentDescription` + schema `description-max-length` | ✅ Addressed |
| 3.1–3.4 Job-linked: ellipsis entry, OrderID disabled, Job → derived editor, derived Unit | `PayAdjustmentSection`; `proofedUserId` from `selectedJob`; Unit disabled/derived | ✅ Addressed (editor not displayed — Issue #4) |
| 4.1–4.4 Standalone: "+" menu, "No Order ID", active-user search, Unit H/W/Fixed | `PayStandaloneSection`; active-user query; `payUnitOptions` | ✅ Addressed |
| 5.1–5.2 Reason per flow + conditional Description | `jobLinkedPayReasonOptions` / `noOrderPayReasonOptions`; `payReasonRequiresDetails` | ✅ Addressed |
| 6.1–6.2 Pay creates a compensation record; success/discard | `useCreateCompensationMutation` → `POST /api/compensations`; success toast + close | ✅ Addressed |
| Standalone "Project (optional)" deferred | Omitted intentionally | ✅ Intentional deviation (per ticket) |
| **Negative cap = job pay + existing adjustments** *(added post-review, from a PR comment — not a ticket AC)* | New Compensation **GET** search + `sumJobCompensation` → `jobPayCap`; schema caps Pay deductions; Charge cap unchanged | ✅ Added |

---

## Architecture Analysis

The change is well-structured and follows established repo conventions.

- **BFF / service layer** mirrors the existing Charge stack: `api/compensations/*` + `addCompensation` + `services/compensations` are structural twins of the Charge equivalents. The post-review cap work adds a **GET** `getCompensations` handler (mirroring `getCharges`) + `useCompensationsQuery` + `sumJobCompensation`.
- **Pure logic is extracted and unit-tested** (`calculatePayAmount`, `applyAmountSign`, `buildCompensationPayload`, `composeAdjustmentDescription`, `payReasonRequiresDetails`, guards, `sumJobCompensation`). `index.tsx` stays UI-only.
- **Reuse-first**: `HourlyInputTextCell` / `EditableInputTextCell` reused for Rate/Quantity; sole existing `HourlyInputTextCell` consumer unaffected.
- **Unit encoding** correct: Hourly 60 / Words 1000 / Fixed 1; matches OMS payload examples.

The approach is sound. The findings below are refinements, not blockers.

---

## Issues Found

### 1. Amount field is directly editable in the Pay flow

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/index.tsx]**

**Function/Class:** AdjustPayOrChargeModal (FormikAmountToAdjust)

**Severity:** medium

**Problem:** Req 2.2 states the Amount "is calculated automatically … and is not directly editable." Originally the `<Input type="number">` inside `FormikAmountToAdjust` stayed user-editable for Pay, so a value typed into it persisted and was submitted verbatim — a financial-integrity concern for the Wise payout.

**Impact:** An admin could submit a compensation whose `amount` ≠ `payRate × (quantity / payUnit)`.

**Fix:** Make the numeric Amount read-only for Pay while keeping the +/- currency toggle interactive.

**Resolution:** ✅ **Fixed** (commit `e3799de34`). Added an opt-in `isAmountReadOnly` prop to `AmountToAdjust` (numeric input `readOnly`, value muted `navyBlue3`, toggle still active) and set `isAmountReadOnly={values.adjustmentType === "pay"}` in the modal. Verified live.

### 2. New BFF route, schema, and service have no dedicated tests

**[File: apps/creative-portal/api/compensations/schema.ts]**

**Function/Class:** createCompensationSchema / createCompensation / addCompensation / postCreateCompensation

**Severity:** low

**Problem:** Front-end logic is thoroughly tested, but the BFF **create** layer (Yup `createCompensationSchema` incl. `non-zero-amount`, the `createCompensation` handler, `addCompensation`, `postCreateCompensation`) has no direct tests. CLAUDE.md requires tests for new code.

**Impact:** Regressions in server-side validation or registry wiring wouldn't be caught by unit tests. Contained because the handler/service are near-exact mirrors of the (also lightly-tested) Charge equivalents.

**Fix:** Add a small `schema.test.ts` for the required-field set + `non-zero-amount` rejection.

**Resolution:** ⏭️ **Skipped** — accepted for this PR (low; 1:1 parity with the existing Charge stack). Note: the post-review cap work did add `services/compensations/index.test.ts` (`sumJobCompensation`), but the **create** schema/handler remain untested. Optional follow-up.

### 3. Nested ternaries with inline eslint-disable in the JSX

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/index.tsx]**

**Function/Class:** AdjustPayOrChargeModal (reason `options` and details `placeholder`)

**Severity:** low

**Problem:** The Reason `options` and details `placeholder` use nested ternaries suppressed with `// eslint-disable-next-line no-nested-ternary`; the flow-selection logic is duplicated inline.

**Impact:** Minor readability/maintainability; two places to keep in sync.

**Fix:** Extract `getReasonOptions(...)` / `getDetailsPlaceholder(...)` helpers into `consts.ts`/`utils.ts`.

**Resolution:** ⏭️ **Skipped** — cosmetic; no functional impact. Deferred as optional polish.

### 4. Job-linked flow derives the editor but never displays it

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/partials/PayAdjustmentSection/index.tsx]**

**Function/Class:** PayAdjustmentSection

**Severity:** low

**Problem:** `proofedUserId` is correctly derived from `selectedJob.proofedUserId`, but the UI never shows which editor will be paid; an unassigned job (`proofedUserId` null) fails validation with "User is required" that has no visible anchor in this flow.

**Impact:** Invisible for completed orders with an assigned editor (works fine). The dead-end only occurs on an unassigned job, which the ticket's preconditions exclude — hence low.

**Fix:** Optionally render the derived editor name (read-only) once a job is selected, and/or guard the Job options to assigned jobs.

**Resolution:** ⏭️ **Skipped** — low; the failing path is excluded by the ticket preconditions. Deferred (nice-to-have UI polish; confirm against Figma).

> **Post-review additions (verified green on `b34cc678`).** Two changes were made after the initial review:
> - **Negative-Pay cap = job pay + existing adjustments** — new BFF **GET** `/api/compensations` (mirrors `getCharges`) → `useCompensationsQuery` → `sumJobCompensation` (nets the job's `T` total + `A` adjustments, skips the redundant `D` detail to avoid double-counting) → derived `jobPayCap` the schema caps against. Charge keeps its order-total cap **unchanged**. Verified against real live data (job 23929: cap 14.96 → a −16.50 deduction is correctly blocked). Note: enforcement is **client-side only** — the OrderService does not enforce the cap (a real over-deduction record exists), so a backend ticket is recommended.
> - **Rate `type="text"` + `inputMode="decimal"`** (both Pay sections) so trailing-zero decimals like `13.60`/`155.00` stay in sync with the `pkw` postfix (a number input reported normalized values through `onChange`, desyncing the measuring ruller). Verified live (`$ 155.00 pkw` renders cleanly).

---

## Validation Checks

*This session's runs on `b34cc678` (reviewer opted to use session results; full `turbo run build` not run).*

| Check | Result | Notes |
|---|---|---|
| `test` (modal + `services/compensations`) | ✅ | 64/64 pass |
| `typecheck` (`@proofed/creative-portal`, `@proofed/shared`) | ✅ | 0 errors |
| `lint` (`@proofed/creative-portal`, `@proofed/shared`) | ✅ | 0 errors |
| `build` (all workspaces) | ⚠️ Not run | Confirm via CI |

Pre-existing unrelated failure on `develop`: `@proofed/shared/utils/formatWordQuantity.test.ts` (locale digit-grouping) — not touched by this PR.

---

## Tests

- ✅ Strong front-end coverage: `utils.test.ts`, `consts.test.ts` (options, `payReasonRequiresDetails`, `composeAdjustmentDescription`, schema rules), `index.test.tsx` (per-flow rendering, "only Unit clears" on job select).
- ✅ Post-review: cap tests (job-pay cap; Charge cap unchanged; exact full-pay), `sumJobCompensation` (D/T double-count, fallback), Rate `allowOnlyDecimalKeyDown`.
- ⚠️ No tests for the BFF **create** schema/handler/service (Issue #2 — skipped).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Req 2.2 now met (Amount read-only); cap verified vs live data |
| Regression risk | ✅ Low — Charge/standalone flows untouched; additive BFF route |
| Tests | ⚠️ Strong front-end; BFF create-layer tests still missing (Issue #2, skipped) |
| Code quality | ✅ (minor: nested ternaries, Issue #3, skipped) |
| Validation suite | ✅ test/typecheck/lint pass (session); ⚠️ build not run |
| Mergeable state | ✅ Clean (GitHub `mergeable_state: clean`) |

---

## Recommendation

**Approve** — the one finding with financial impact (Issue #1) is **fixed**; Issues #2–#4 are **Skipped** (accepted low-priority).

Follow-ups (outside this PR / optional):

1. **File a backend ticket** to enforce the negative-Pay cap server-side — the frontend rule is a UX guard, not authoritative (a real over-deduction exists in the data).
2. Run the full `turbo run build` in CI before merge (not run locally in this review).
3. Optional: BFF create-schema test (Issue #2); extract reason/placeholder helpers (Issue #3); display/guard the derived editor (Issue #4).
