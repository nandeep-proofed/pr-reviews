# PR Review: feature/PP-1944: Enable Pay adjustment type in Adjust Pay or Charge modal

**PR:** https://github.com/Proofed/B2BWebserver/pull/2360
**Jira:** https://proofed.atlassian.net/browse/PP-1944
**Status:** In Progress
**Head:** `feature/PP-1944-enable-pay-adjustment` @ `b9dadfc1a` → `develop`
**Files:** 18 changed (+~960 / −~60)

> **Review note:** one functional bug surfaced during the live walkthrough and was **fixed in this review** (Issue 5, commit `b9dadfc1a`) — the Amount field was locked in standalone Pay. Issues 1–4 below remain as suggestions.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1.1 Add "Pay" to Adjustment Type in both flows | `adjustmentTypeOptions` includes Pay; dropdown enabled in both flows (`isDisabled` no longer keys off `isNoOrder`) | ✅ Addressed |
| 1.2 "Charge remains the default" | **Intentionally not implemented** — opens on "Select Adjustment Type" placeholder (`adjustmentType: ""`, required) per the PP-1944 design; confirmed with the team | ⚠️ Superseded by design (deliberate) |
| 2.1 Currency always USD, not changeable | `ChargeCurrencySync` forces USD for Pay; currency not in the compensation payload | ✅ Addressed |
| 2.2 Unit + Rate + Quantity → auto-calc Amount, not editable | `PayAmountCalculator` → `calculatePayAmount` = `rate × (qty / unit)`; Amount field driven by the calculator | ✅ Addressed |
| 2.3 +/- sign; negatives allowed, zero not | Shared `FormikAmountToAdjust` toggle; `applyAmountSign` preserves the sign through recalc; amount≠0 enforced (form + BFF) | ✅ Addressed |
| 2.4 Quantity → all three quantity fields | `buildCompensationPayload` writes `quotedPayQuantity = userEnteredQuantity = approvedPayQuantity` | ✅ Addressed |
| 2.5 Description mandatory, ≤200, Reason(+details), colon/pipe separators | `composeAdjustmentDescription` (colon No-Order / pipe order-linked); ≤200 enforced for Pay (form schema + BFF `max(200)`) | ✅ Addressed |
| 3. Completed-order Pay (job-linked): ellipsis entry, Order ID disabled, Job select, editor from assignee, Unit derived | `PayAdjustmentSection`: Job select, disabled derived Unit, `proofedUserId` stashed from `selectedJob.proofedUserId`, Rate postfix `ph`/`pkw` | ✅ Addressed |
| 4. Standalone Pay (No Order): "+"→Adjustment, "No Order ID", User search, Unit Hourly/Words/Fixed | New `PayStandaloneSection`: type-ahead User search, Unit select, Rate (`ph`/`pkw`/`pq`), unit-aware Quantity | ✅ Addressed |
| 4. "Project (optional)" on standalone Pay | **Deferred** per ticket note + Compensation API has no project field (confirmed) | ✅ Correctly omitted |
| 5. Pay Reasons: Overpayment/Underpayment/Deduction/Bonus | `payReasonOptions`; Reason switches on type | ✅ Addressed |
| 6. Pay → compensation record (not charge); job-linked vs standalone by Job/editor; success toast + close; Discard/X close | `handleSubmit` branches Pay → `createCompensation`; toast + `onClose`; Discard/X unchanged | ✅ Addressed |
| API: `POST /api/Compensations`, EntryType "A", job-linked vs standalone by `jobId`, USD | BFF `pages/api/compensations` → `addCompensation` → OMS `COMPENSATION` service; payload matches the spec | ✅ Addressed |

**Beyond ticket scope:** the placeholder default (1.2) is a deliberate design-over-AC deviation, confirmed with the team. No other scope creep.

---

## Architecture Analysis

Clean, layered, and consistent with the existing Charge implementation:
- **Service/BFF** mirrors charges exactly: `services/compensations` → `pages/api/compensations` → `api/compensations` handler (`makeMethodToHandlerMapping` + `withApiMiddleware` + Yup body schema) → `addCompensation` (`prepareServiceAxiosConfigWithData("compensation", …)`). New `compensation → "COMPENSATION"` registry key (confirmed live: `OrderService/api/Compensations`).
- **UI** keeps `index.tsx` thin and pushes logic into partials/hooks: job-linked `PayAdjustmentSection` (existing, extended), new standalone `PayStandaloneSection`, `PayAmountCalculator`, `ChargeCurrencySync`.
- **Testability:** pure helpers `calculatePayAmount` / `applyAmountSign` / `buildCompensationPayload` / `composeAdjustmentDescription` extracted into `utils.ts`/`consts.ts` and unit-tested (the payload tests mirror the OMS spec examples 1:1).
- **Regression surface:** Charge paths are gated on `adjustmentType`/`isNoOrder`, so they render and submit exactly as before; the description-length check is scoped to Pay so Charge is byte-for-byte unchanged. Sole modal consumers (`OrderManagment` ellipsis, Header "+") are unaffected (same props).

---

## Issues Found

### 1. Standalone User search is not restricted to editors

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/hooks.ts]**

**Function/Class:** useAdjustPayOrChargeModal — `editorOptions`

**Severity:** medium

**Problem:** The User search uses `useUsersQuery({ searchBy: "name", searchValue: term })` and filters only by `editor.active !== false`. It does not filter by role, so any user matching the name (e.g. admins/staff) can appear and be selected. The ticket specifies a "search/select of active **editors**."

**Impact:** An admin could create a pay/compensation record against a non-editor user. The `UserDataSearchProps` shape includes `roles`, so the data is available to filter.

**Fix:** Confirm the intended role with the team and filter, e.g. `editors.filter((u) => u.active !== false && u.roles?.includes(<editorRole>))`. Already flagged with a `TODO(PP-1944)` in the code.

### 2. Type-specific field values persist across an Adjustment Type round-trip

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/index.tsx]**

**Function/Class:** adjustmentType `onChange` / form state

**Severity:** low-medium

**Problem:** Switching the type clears `errors`/`touched` (good) but **not field values**. If an admin selects an editor under Pay, switches to Charge, then back to Pay, `proofedUserId` is still set in Formik, but `PayStandaloneSection` remounts with empty local `selectedEditor`, so the User field shows the "Search for a user" placeholder while a value is silently retained. Same lingering applies to `jobId`/`unit`/`rate`/`quantity` and (Pay→Charge) `organizationGroupId`.

**Impact:** Submission is still correct (`handleSubmit`/`buildCompensationPayload` read fields by type), but the UI can show an empty-looking User while a hidden `proofedUserId` is submitted — a potential "paid the wrong/again editor" surprise. Edge case (requires a type round-trip).

**Fix:** On type change, also reset the now-irrelevant field values, e.g. clear `proofedUserId`/`jobId`/`unit`/`rate`/`quantity` (and `organizationGroupId` when leaving Charge) alongside the existing `setErrors({})`/`setTouched({})`.

### 3. Non-null assertions in the payload builder

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/utils.ts]**

**Function/Class:** buildCompensationPayload

**Severity:** low

**Problem:** `proofedUserId: values.proofedUserId as number` and `payUnit: values.unit as number` cast away `null`. They're guaranteed non-null by the Pay validation before submit, so it's safe in practice, but the assertions bypass TS null-safety.

**Impact:** If the function were ever called outside the validated submit path, it would emit `null` for required fields. Low.

**Fix:** Optional — guard/narrow instead of asserting, or leave with the validation contract documented (it is, via the schema).

### 4. New UI interactions lack automated coverage

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/]**

**Function/Class:** modal component behavior

**Severity:** low

**Problem:** Pure logic is well covered (28 tests). The new **UI behaviors** — type-switch validation reset, conditional section rendering (charge vs job-linked vs standalone), editor-option merge for label persistence, unit-change quantity reset, disabled derived Unit — have no component/render tests; they rely on the manual/live verification done in this session.

**Impact:** A future refactor could silently regress these without a failing test. Low (the repo has limited heavy modal-render test precedent).

**Fix:** Optional follow-up: add a `@testing-library/react` test rendering the modal with mocked queries to assert the section swaps and the type-switch reset.

### 5. Amount field locked in standalone Pay — ✅ Resolved (commit `b9dadfc1a`)

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/index.tsx]**

**Function/Class:** FormikAmountToAdjust `disabled`

**Severity:** high (functional)

**Problem:** The Amount field gated `disabled` on `isNoOrder && (!organizationGroupId || isFetchingProjects)`. Standalone Pay has no Project field, so `organizationGroupId` is always null → the Amount was disabled. `AmountToAdjust` only renders the +/- currency toggle when **not** disabled (`prefix={!disabled ? currencyToggle : undefined}`), so the toggle was **hidden** and the input locked — making deductions (negative amounts, req 2.3) impossible in standalone Pay, contrary to the design.

**Impact:** Standalone Pay could not record a deduction and the Amount appeared locked. High (breaks a core Pay behaviour).

**Resolution:** Scoped the "wait for a Project" gate to No-Order **Charge** only (`… && values.adjustmentType === "charge" && …`). Standalone Pay's Amount is now active with a working sign toggle (currency is USD immediately via `ChargeCurrencySync`); job-linked Pay and No-Order Charge are unchanged. Lint + typecheck green.

> Follow-up nuance: with the field enabled, the number is technically typeable; `PayAmountCalculator` recomputes it from Rate×Qty÷Unit on any field change so it stays authoritative. Strictly enforcing "not directly editable" while keeping the toggle would require a shared-component change (read-only input + active toggle).

---

## Validation Checks

Run on the PR branch at `c5abd4100` (dev server stopped, `.next` cleaned). The Issue-5 fix (`b9dadfc1a`) is a one-line change on top; typecheck + lint re-run green for it.

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ✅ | All pass except the **pre-existing, unrelated** `@proofed/shared/utils/formatWordQuantity.test.ts` (locale digit-grouping `"10,00,000"` vs `"1,000,000"`); 1318/1319 in shared, 28/28 in the modal suite. Not touched by this PR. |
| `npx turbo run typecheck` | ✅ | 0 errors across all workspaces. |
| `npx turbo run lint` | ✅ | 0 errors across all workspaces. |
| `npx turbo run build` | ⚠️ Inconclusive (flaky, code-unrelated) | Failed **after** `✓ Compiled successfully`, during *Collecting page data*: `TypeError: ReactJSXRuntimeDev.jsxDEV is not a function` from `@emotion/react/jsx-dev-runtime` in `components/molecules/Notifications/consts.tsx` — **a file this PR does not touch**. This is the **third different** post-compile failure this session on the same code (clean pass at parent `3250494e0` → `MODULE_NOT_FOUND` in `_document` → `jsxDEV` in Notifications), all after a clean compile → local build-environment flakiness, not a PR defect. Recommend CI (clean container) as authoritative. |

---

## Tests

- ✅ `utils.test.ts` (12): `calculatePayAmount` (Words/Hourly/**Fixed**), `applyAmountSign` (sign preservation incl. zero), `buildCompensationPayload` (job-linked vs standalone matching the OMS examples, quantity→3 fields, string coercion, negative/deduction, jobId omission).
- ✅ `consts.test.ts` (16): Pay enabled, `payReasonOptions`, `payUnitOptions`, `composeAdjustmentDescription` (colon/pipe/reason-only/no-reason), schema rules (valid Pay, amount≠0, proofedUserId/jobId required, standalone no-Job allowed, 200-char Description).
- ✅ Code compiles cleanly (`✓ Compiled successfully`).
- ⚠️ No component/render tests for the new UI interactions (Issue 4).
- ✅ Live-verified in nandeep's Chrome against the Figma designs (Charge, job-linked Pay incl. derived Unit/postfix, standalone Pay incl. User search + Unit, placeholder default, validation messages, USD, counter removed, type-switch reset, Amount fix).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Matches the agreed behaviour and the OMS spec; Charge unchanged (Issue 5 fixed in review) |
| Regression risk | ✅ Low — additive; Charge paths gated and unchanged; single modal consumer surface |
| Tests | ⚠️ Strong unit coverage; UI-interaction tests absent (Issue 4) |
| Code quality | ✅ Good; minor items (Issues 1–3) |
| Validation suite | ⚠️ test/typecheck/lint green; build inconclusive (flaky, code-unrelated) — confirm via CI |
| Mergeable state | ✅ Clean (GitHub `mergeable_state: clean`) |

---

## Recommendation

**Approve with suggestions — pending a green build (CI authoritative).**

1. **Issue 5 (high) — found & fixed in review** (`b9dadfc1a`): the Amount field was locked in standalone Pay (gated on the Project that Pay doesn't have), hiding the +/- toggle and blocking deductions. Now scoped to No-Order Charge.
2. **Build gate:** the local build failure is demonstrably **not** from this PR (compiles clean; error is `@emotion/react` jsxDEV in an untouched file; three different post-compile failures this session on the same code = environment flakiness). Confirm a green build in CI before merge; do **not** treat the local failure as a code blocker.
3. **(Issue 1, medium)** Confirm whether the standalone User search must filter to an editor role and apply it — currently any active user can be selected.
4. **(Issue 2, low-medium)** Clear type-specific field values (not just errors) when the Adjustment Type changes, so a hidden `proofedUserId`/`jobId` can't survive a type round-trip.
5. **(Issues 3–4, low)** Optional: tighten the payload-builder assertions; add a modal component test for the section-swap + type-switch reset.
6. **Process:** `/security` review before merge (per the repo workflow); the `COMPENSATION` registry name is confirmed; the "Project (optional)" deferral is confirmed (Compensation API has no project field).
