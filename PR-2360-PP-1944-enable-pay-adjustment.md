# PR Review: feature/PP-1944: Enable Pay adjustment type in Adjust Pay or Charge modal

**PR:** https://github.com/Proofed/B2BWebserver/pull/2360
**Jira:** https://proofed.atlassian.net/browse/PP-1944
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1.1 Add "Pay" to Adjustment Type dropdown in both flows | `adjustmentTypeOptions` now `[Charge, Pay]`; selector `isDisabled` only when `options.length <= 1`, so Pay is selectable in both the order and No Order flows | ✅ Addressed |
| 1.2 Charge remains the default | No type is pre-selected — opens on the "Select Adjustment Type" placeholder (`DEFAULT.adjustmentType = ""`) | ⚠️ Intentional deviation — PR states the design's first-open placeholder supersedes req 1.2 (confirmed with team) |
| 2.1 Currency always USD, not changeable | `ChargeCurrencySync` forces `chargeCurrencyCode = "USD"` when `adjustmentType === "pay"`; no currency selector; payload omits currency (OMS defaults USD) | ✅ Addressed |
| 2.2 Amount auto-calculated, not directly editable | Auto-calc via `PayAmountCalculator` (`calculatePayAmount`) ✅, but the `FormikAmountToAdjust` input is **not disabled** for Pay, so it can be typed into | ⚠️ Partial — see Issue #1 |
| 2.3 +/- toggle sets sign; negative allowed, zero rejected | `applyAmountSign` preserves the toggle sign across recalcs; schema `valid-amount` test rejects 0 (front) + `non-zero-amount` (BFF) | ✅ Addressed |
| 2.4 Quantity written to all 3 quantity fields | `buildCompensationPayload` writes `quantity` to `quotedPayQuantity`/`userEnteredQuantity`/`approvedPayQuantity` | ✅ Addressed |
| 2.5 Description = Reason (+ details), 200 max, colon (No Order) / pipe (order) | `composeAdjustmentDescription` (colon vs pipe) + schema `description-max-length` (200) + BFF `max(200)` | ✅ Addressed |
| 3.1 Job-linked only via completed-order ellipsis | Entry point unchanged | ✅ Addressed |
| 3.2 Order ID pre-populated and disabled | `orderIdString` from order, `FormikInput ... disabled` | ✅ Addressed |
| 3.3 Job select; editor derived from job assignee; no user search | `PayAdjustmentSection` derives `proofedUserId` from `selectedJob.proofedUserId` (Job type exposes it); no user search in this flow | ✅ Addressed (editor not displayed — see Issue #4) |
| 3.4 Unit derived from job; rate/quantity follow unit | Unit select is `isDisabled`, value derived from `selectedJob.jobTasks[0].payUnit`; Words → number cell, else DurationInput | ✅ Addressed |
| 4.1 Standalone via "+" → Adjustment, No Order mode | `PayStandaloneSection` rendered when `isNoOrder && adjustmentType === "pay"` | ✅ Addressed |
| 4.2 Order ID shows "No Order ID", disabled | Default `orderIdString = "No Order ID"` | ✅ Addressed |
| 4.3 Required User field (active-user search/select) | `useUsersQuery({ searchBy: "status", userStatus: "A" })`; searchable select; schema requires `proofedUserId` for Pay | ✅ Addressed |
| 4.4 Unit selectable Hourly/Words/Fixed | `payUnitOptions = [Hourly 60, Words 1000, Fixed 1]`; rate postfix `ph`/`pkw`/`pq` | ✅ Addressed |
| 5.1 Reason dropdown per flow | `jobLinkedPayReasonOptions` / `noOrderPayReasonOptions` selected by flow | ✅ Addressed |
| 5.2 Description mandatory for certain reasons | `payReasonRequiresDetails` + schema `details.when(...)`; matches the "Y" rows of the ticket table | ✅ Addressed |
| 6.1 Pay creates a compensation record (job-linked→Job, standalone→editor) | `useCreateCompensationMutation` → `POST /api/compensations` → `addCompensation`; `jobId` only when job-linked | ✅ Addressed |
| 6.2 Success closes modal + success toast; Discard/X close without submit | Success toast "Adjustment created successfully" + `onClose()`; cancel/close unchanged | ✅ Addressed |
| Standalone "Project (optional)" deferred | Omitted intentionally | ✅ Intentional deviation (per ticket) |

---

## Architecture Analysis

The change is well-structured and follows established repo conventions closely.

- **BFF / service layer** mirrors the existing Charge stack almost 1:1: `api/compensations/{createCompensation,index,schema,types}.ts` + `api/utils/compensations/addCompensation.ts` + `pages/api/compensations/index.ts` + `services/compensations`. `addCompensation` and `createCompensation` are structural twins of `addCharge`/`createAdjustmentCharge` (same middleware, `prepareServiceAxiosConfigWithData("compensation", …)`, `requesterId` header, `handleEndpointError`). Route registration is added in both `apiRoutes` (`compensations`) and `b2bRoutesMap` (`compensation → COMPENSATION`).
- **Pure logic is extracted and unit-tested**: `calculatePayAmount`, `applyAmountSign`, `buildCompensationPayload`, `composeAdjustmentDescription`, `payReasonRequiresDetails`, and the paste/keydown guards all live in `utils.ts`/`consts.ts` and are covered by `utils.test.ts` / `consts.test.ts`. The `index.tsx` stays UI-only and delegates to `hooks.ts`, consistent with the project's component conventions.
- **Reuse-first**: `HourlyInputTextCell` and `EditableInputTextCell` are reused for the Rate/Quantity fields instead of new inputs, with their prop types widened to support form-mode usage (`FormikDurationInputProps`, `Partial<CellProps>`). Both widenings are type-permissive and the sole existing `HourlyInputTextCell` consumer (`ChargeTable/partials/QuantityCell`) passes only `name`, so there is no regression.
- **Unit encoding** is correct: `HOURLY_UNIT_VALUE = 60`, `WORDS = 1000`, `FIXED = 1`; `DurationInput` stores minutes, so `rate × (minutes / 60)` yields rate-per-hour × hours. Matches the OMS payload examples in the ticket.

The approach is sound. The findings below are refinements, not blockers.

---

## Issues Found

### 1. Amount field is directly editable in the Pay flow

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/index.tsx]**

**Function/Class:** AdjustPayOrChargeModal (FormikAmountToAdjust `disabled` prop)

**Severity:** medium

**Problem:** Req 2.2 states the Amount "is calculated automatically from Rate, Unit, and Quantity and is not directly editable." For Pay, the `disabled` expression evaluates to `false` (once charges finish loading in the job-linked flow), so the underlying `<Input type="number">` inside `FormikAmountToAdjust` remains user-editable. `PayAmountCalculator` only overwrites `amount` when `rate`/`quantity`/`unit` *change*, so a value typed directly into the Amount field persists and is submitted verbatim via `buildCompensationPayload` (`amount: parseFloat(values.amount)`).

**Impact:** An admin can submit a compensation whose `amount` does not equal `payRate × (quantity / payUnit)`. Because this record feeds the monthly Wise payout, an inconsistent amount vs. rate/quantity is a financial-integrity concern. Easily triggered (enter rate+quantity, then edit the amount), though the auto-recalc mitigates the accidental case.

Note: `FormikAmountToAdjust` hides the +/- toggle when `disabled` (`prefix={!disabled ? currencyToggle : undefined}`), so a plain `disabled` would also remove the sign control. If you disable the field, keep the toggle working by using the input's own read-only path rather than the wrapper-level `disabled`, or add a `readOnly`/`isAmountEditable` prop to `AmountToAdjust`.

### 2. New BFF route, schema, and service have no dedicated tests

**[File: apps/creative-portal/api/compensations/schema.ts]**

**Function/Class:** createCompensationSchema / createCompensation / addCompensation / postCreateCompensation

**Severity:** low

**Problem:** The 42 new tests thoroughly cover the front-end logic (`utils`, `consts`, schema rules, and modal rendering), but the BFF layer added by this PR — the Yup `createCompensationSchema` (incl. the `non-zero-amount` test), the `createCompensation` handler, `addCompensation`, and the `postCreateCompensation` service — has no direct tests. CLAUDE.md requires tests for new code.

**Impact:** Regressions in the server-side validation (e.g. the non-zero-amount rule or the required-field set) or the service-registry wiring would not be caught by unit tests. Risk is contained because the handler/service are near-exact mirrors of the existing (also lightly-tested) Charge equivalents.

**Fix:** Add a small `schema.test.ts` asserting the required-field set and the `non-zero-amount` rejection, mirroring the front-end schema tests. A handler test is optional given the 1:1 parity with `createAdjustmentCharge`.

### 3. Nested ternaries with inline eslint-disable in the JSX

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/index.tsx]**

**Function/Class:** AdjustPayOrChargeModal (reason `options` and details `placeholder`)

**Severity:** low

**Problem:** Both the Reason `options` and the details `placeholder` use nested ternaries suppressed with `// eslint-disable-next-line no-nested-ternary`. The flow-selection logic (charge vs No-Order Pay vs job-linked Pay) is duplicated inline in the render.

**Impact:** Minor readability/maintainability; two places must be kept in sync with the reason/placeholder rules.

**Fix:** Extract small helpers (e.g. `getReasonOptions({ adjustmentType, isNoOrder })` and `getDetailsPlaceholder({ adjustmentType, reason })`) into `consts.ts`/`utils.ts` — they'd also be trivially unit-testable and remove the disable comments.

### 4. Job-linked flow derives the editor but never displays it

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/partials/PayAdjustmentSection/index.tsx]**

**Function/Class:** PayAdjustmentSection

**Severity:** low

**Problem:** Req 3.3 says the editor is derived from the selected job's assignee. The code correctly sets `proofedUserId` from `selectedJob.proofedUserId`, but nothing in the UI shows *which* editor will be paid. If the selected job has no assignee (`proofedUserId` null), `proofedUserId` stays null and the form fails validation with "User is required" — but there is no User field in this flow, so the error has no visible anchor and the admin gets a dead-end with no explanation.

**Impact:** For completed orders with an assigned editor this is invisible (works fine). The dead-end only occurs on an unassigned job, which the ticket's preconditions exclude — hence low severity. Displaying the derived editor would also improve confidence for the admin.

**Fix:** Optionally render the derived editor name (read-only) once a job is selected, and/or guard the Job options to jobs that have an assignee. Confirm against the Figma whether the editor name is expected to be shown in this flow.

---

## Code-Quality Review (`/simplify`)

A separate cleanup pass (reuse / simplification / efficiency / altitude — **not** a
correctness/bug review) run across the diff via four parallel review agents. These are
observations only — no fixes/action items and nothing applied to the branch. Line numbers
are on the PR branch. Q5 overlaps with Issue #3 above and is not repeated.

### High-impact

#### Q1. The two Pay sections duplicate the whole Unit → Rate → Quantity block (~50 lines)

**[Files: partials/PayAdjustmentSection/index.tsx:84-137 vs partials/PayStandaloneSection/index.tsx:63-121]** · **reuse/simplification**

The Rate `<Box>` is verbatim-identical except the gate field (`values.jobId` ↔ `values.unit`);
the Quantity `<Box>` is the same 3-branch pattern (disabled-dash / hourly-input /
words-fixed-input) with the branches reordered. Every placeholder/prefix/postfix/guard tweak
must be made in both copies and kept in sync (the reordered Quantity branches already show
minor drift).

#### Q2. Digit/paste guards re-implement the order-create QuantityCell guards

**[File: components/organisms/modals/AdjustPayOrChargeModal/utils.ts]** · **reuse**

`ALLOWED_QUANTITY_KEYS` is byte-identical to `ALLOWED_NAVIGATION_KEYS` in
`ChargeTable/partials/QuantityCell/index.tsx:22-34`, and `allowOnlyDigitsKeyDown` /
`allowOnlyDigitsPaste` match the QuantityCell inline handlers (the code comments even say
"Mirror the order-create Quantity cell"). Two copies that will drift.
`blockNegativeNumberKey` / `blockNegativeNumberPaste` (Rate) are genuinely new — no existing
equivalent.

#### Q3. `ratePostfix` is computed two different ways — and they disagree for Fixed-rate

**[Files: PayAdjustmentSection/index.tsx:71-72 and PayStandaloneSection/index.tsx:42-49]** · **reuse/simplification**

Job-linked uses `getServiceShortUnit(mapUnit(String(selectedJobUnit))) || "ph"`; standalone uses
a 4-branch nested ternary → `pkw`/`ph`/`pq`/`""`. `getServiceShortUnit` returns `""` for Fixed,
so a Fixed unit renders **`"ph"` in one section but `"pq"` in the other**. (Fixed isn't reachable
in the job-linked flow today, but the two representations will keep drifting.)

### Medium

#### Q4. `payReasonRequiresDetails` uses a parallel `Set` that can drift from the reason lists

**[File: consts.ts:50-63 (Set) vs consts.ts:196-221 (option arrays)]** · **altitude**

"Requires details" is a flat `Set<string>` of reason labels, separate from the arrays that define
those labels — rename a reason and the Set silently goes stale (no type error). Drift is already
visible in the test suite (`"Under Payment"` vs `"Underpayment"`).

#### Q5. Nested ternaries with inline `eslint-disable` in the JSX

**[File: index.tsx:180-187 (reason options) and index.tsx:194-201 (details placeholder)]** · **simplification**

_Same as **Issue #3** above — see that entry for detail._ The flow-selection logic (charge vs
No-Order Pay vs job-linked Pay) is duplicated inline in the render, each suppressed with a
`no-nested-ternary` disable.

#### Q6. Active-user list refetched on every modal open — no `staleTime`

**[File: hooks.ts:115-119]** · **efficiency**

`useUsersQuery({ searchBy: "status", userStatus: "A" }, { enabled })` passes only `enabled`; the
shared QueryClient sets no default `staleTime` (=0), so each No-Order modal open triggers a
background refetch of the entire active-user list. The sibling `useUsersHeadshotPhotoUrl`
(`services/users/index.ts:71-73`) sets a `staleTime` for the same "rarely-changes list" case.

#### Q7. Modal reaches into `tables/partials/*` cell components as form fields

**[Files: PayAdjustmentSection/index.tsx, PayStandaloneSection/index.tsx]** · **altitude**

The diff replaced `FormikInput` / `FormikDurationInput` with `EditableInputTextCell` /
`HourlyInputTextCell`, then passes `hasErrorMessageHidden={false}` on every cell to switch off the
wrapper's one distinguishing default — i.e. asking for plain-input behaviour while paying for a
table-cell import. Couples an organism modal to table internals. Note: this is a deliberate,
design-verified choice in the PR; the `Partial<CellProps>` / `FormikDurationInputProps` widenings
called out in the Architecture Analysis are permissive and safe — Q7 is about depth, not regression.

### Low / minor

- **Q8. Dead `isDisabled` on the Adjustment Type select** — `index.tsx:115`
  `isDisabled={adjustmentTypeOptions.length <= 1}` is always `false` (static 2-element array); vestige
  of the previous single-option state.
- **Q9. `toQuantityNumber` coercion re-inlined** — `PayAmountCalculator/index.tsx:19-22` repeats the
  `typeof … ? … : Number(…) || 0` that `utils.ts` already defines (currently un-exported).
- **Q10. Inline `onChange` reset handler** — `index.tsx:116-130` is a ~14-line callback
  (`setErrors`/`setTouched`/`setValues` + a cast) in the render prop, against the repo's
  "`index.tsx` is UI-only" convention. Borderline (the Formik helpers only exist in the render prop).
- **Q11. `CellProps → Partial<CellProps>`** (`EditableInputTextCell/types.d.ts:14`) — the type is
  consumed only by the cell itself, which never reads `cell/row/column/value`; `Partial<CellProps>`
  weakens nothing real but also fixes nothing. Marginal.
- **Q12. Yup schema repetition** — `consts.ts:130-157` has four near-identical
  `.nullable().typeError().when(is:"pay", required)` blocks. The single-schema-with-`.when` approach
  itself is fine (`adjustmentType` is an in-form value).

### Confirmed clean (no concern)

- Compensation BFF/service layer faithfully mirrors the existing `charges` house pattern.
- `editorOptions` user → `{value,label}` mapping — no existing shared mapper to reuse.
- `calculatePayAmount` / `applyAmountSign` correctly reuse `roundToCurrency`; no shared equivalent exists.
- The two reset `useEffect`s in `PayAdjustmentSection` should **not** be merged (different dep arrays,
  disjoint fields) — merging would wipe user input when `selectedJob` resolves.
- No cascading re-render loop in `PayAmountCalculator` (guarded + `!==` check).

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ Skipped — user opted out | PR reports pass except a pre-existing, unrelated failure in `@proofed/shared/utils/formatWordQuantity.test.ts` |
| `npx turbo run typecheck` | ⏭️ Skipped — user opted out | PR reports ✅ |
| `npx turbo run lint` | ⏭️ Skipped — user opted out | PR reports ✅ |
| `npx turbo run build` | ⏭️ Skipped — user opted out | PR defers to CI |

---

## Tests

- ✅ Strong front-end coverage: `utils.test.ts` (`calculatePayAmount` incl. Fixed, `applyAmountSign`, `buildCompensationPayload` job-linked vs standalone + quantity→3 fields + coercion + negative, paste guards) and `consts.test.ts` (options, `payReasonRequiresDetails`, `composeAdjustmentDescription`, schema rules incl. amount≠0, required fields, 200-char, conditional details).
- ✅ `index.test.tsx` covers per-flow rendering (Charge reasons, job-linked Pay reasons + Job field, standalone Pay user field + No-Order reasons) and the reason-driven details placeholder.
- ⚠️ No tests for the new BFF schema/handler/service (Issue #2).
- ⚠️ No automated coverage for the amount-not-editable behaviour (relevant to Issue #1).
- ⏭️ Full validation suite not run this review (user opted out) — must be run before merge.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ (one spec deviation — Issue #1) |
| Regression risk | ✅ Low (type widenings are permissive; sole `HourlyInputTextCell` consumer unaffected; BFF mirrors Charge) |
| Tests | ⚠️ Strong front-end, missing BFF-layer tests |
| Code quality | ✅ (minor: nested ternaries, Issue #3) |
| Validation suite | ⏭️ Skipped — user opted out |
| Mergeable state | ✅ Clean (GitHub `mergeable_state: clean`); validation not re-verified locally |

---

## Recommendation

**Approve with suggestions**

1. **Re-run the mandatory validation suite** (`test` / `typecheck` / `lint` / `build`) against the PR branch before merge — it was not run in this review. Confirm the only failing test is the pre-existing, unrelated `formatWordQuantity.test.ts` locale case and that the build is clean in CI.
2. **Decide on Issue #1 (amount editability for Pay).** Either disable direct editing of the Amount for Pay (keeping the +/- toggle functional), or confirm with the PO that overridable amounts are intended and amend req 2.2. This is the only finding with financial impact.
3. Add BFF-layer tests for `createCompensationSchema` (Issue #2) to satisfy the "tests for new code" rule.
4. Optional polish: extract the reason-options / details-placeholder helpers (Issue #3); display or guard the derived editor in the job-linked flow (Issue #4).
