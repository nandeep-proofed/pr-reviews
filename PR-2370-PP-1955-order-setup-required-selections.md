# PR Review: fix/PP-1955: Require Platform, Delivery & Workflow on Order Setup step

**PR:** https://github.com/Proofed/B2BWebserver/pull/2370
**Jira:** https://proofed.atlassian.net/browse/PP-1955
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Require a Workflow before letting the user advance — **block progression** | `deliveryConfigurationId` added to `ORDER_SETUP_STEP_SCHEMA` as `.min(1).required()`; Next button enabled but `type="submit"` so a click runs validation and Formik refuses to advance when invalid | ✅ Addressed |
| **Prompt for the missing selection** | `showErrorOnTouched` removed from the Workflow select so the field-level error renders on the Next-click validation (previously hidden because the field is never `touched`) | ✅ Addressed |
| If a missing Workflow is only detected at submit, fail gracefully with a clear message and return the user to correct it | Made unreachable — the required rule blocks at Order Setup, so no workflow-less order ever reaches submit. Client explicitly agreed Fix #1 (prevention) is sufficient | ⚠️ Addressed by prevention (submit-time path intentionally not built) |
| The user should never be left on an endless spinner | The Charge-step spinner (disabled `useSelectedDeliveryConfiguration` query when `deliveryConfigurationId` is falsy) is now unreachable | ✅ Addressed |
| *(Scope addition)* Platform & Delivery also silently un-prompted | `platform` given a clear message; `delivery` added as required (was not validated at all); `showErrorOnTouched` dropped on both | ✅ Addressed (requested by client during implementation) |

**Scope note:** The PR goes slightly beyond the literal ticket (Workflow only) by also making **Platform** and **Delivery** required with prompts. This was explicitly requested during implementation and is a consistent, in-spirit extension of the same Order Setup step. Not unmanaged scope creep.

---

## Architecture Analysis

The fix targets the real root cause identified in the ticket: `ORDER_SETUP_STEP_SCHEMA` under-validated the step, so the wizard stayed "valid" without a Workflow and let the user reach a Charge step whose delivery-config query never resolves.

Three coordinated changes:
1. **Schema** (`schemas.ts`) — the three Order Setup selections are now required with human-readable messages.
2. **Gate** (`CreateOrderContent`) — the shared `NextStepButton` no longer disables on `!formik.isValid`; it stays enabled (except while validating) so a click triggers validation-on-submit, which both blocks the advance and populates errors.
3. **Display** (`SetupForm`) — `showErrorOnTouched` removed so the populated errors actually render (the fields are never in Formik `initialValues`, so submit never marks them `touched`, which was silently suppressing the message).

This is the idiomatic Formik "validate on submit" pattern and fits the existing per-step `getCreateOrderSchema(step)` design. `ORDER_SETUP_STEP_SCHEMA` is consumed only inside `schemas.ts` (step validation + inferred type); `NextStepButton` is used only by `CreateOrderContent`. Blast radius is confined to the New Order wizard.

**Regression-critical check:** the final **submit** happens on `WORKFLOW_STEP` (step 6), whose schema is base-only (`step`/`dataSourceType`/`chargeCurrency`) — so `formik.isValid` was already `true` there and the button was **already enabled** before this PR. Enabling it therefore does **not** create any new path to submit an incomplete order; it only changes the earlier gated steps from "silently disabled" to "click-to-validate."

---

## Issues Found

### 1. Enabled Next button gives no feedback on the Customer Search step

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/CreateOrderContent/index.tsx]**

**Function/Class:** CreateOrderContent

**Severity:** low

**Problem:** `NextStepButton` is shared across every wizard step with `showNext`. On the Customer Search step the required field is `organizationGroupId`, which has no visible input (selection happens via the results table/row click). Previously the button was disabled until a customer was picked; now it is always enabled, so clicking it runs validation that fails with no on-screen field to attach the error to — the click appears to do nothing.

**Impact:** Minor UX oddity on one step: a button that looks actionable but produces no visible result until a customer row is selected. No functional regression (advancing still correctly blocked).

**Fix:** Optional. If desired, keep the disabled-on-`!isValid` behavior specifically for `CUSTOMER_SEARCH` (which has no inline error surface), or surface a toast/inline message when submit fails on that step. Given customers advance by clicking a row rather than the Next button, this is acceptable to leave as-is for this ticket.

### 2. Errors can appear on blur of a sibling field, not strictly on the Next click

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/OrderSetupStep/partials/SetupForm/index.tsx]**

**Function/Class:** SetupForm

**Severity:** low

**Problem:** Removing `showErrorOnTouched` means the error renders whenever `meta.error` is populated. With `validateOnChange: false` but the default `validateOnBlur: true`, blurring any one field validates the whole form and can populate `platform`/`delivery`/`deliveryConfigurationId` errors before the user has interacted with those specific selects.

**Impact:** Prompts may show a touch earlier than "only on Next click." In practice the dominant trigger is still the Next click (which blurs + submits), and each field auto-selects when it has a single option, so the common single-config path shows nothing. Acceptable and arguably better feedback; noting for awareness.

**Fix:** None required. If strict "only on Next" is desired later, add `deliveryConfigurationId`/`platform`/`delivery` to Formik `initialValues` so submit marks them `touched`, and restore `showErrorOnTouched`.

### 3. Misconfigured platform (zero delivery configs) yields an unsatisfiable prompt

**[File: apps/creative-portal/components/organisms/NewOrderForm/schemas.ts]**

**Function/Class:** ORDER_SETUP_STEP_SCHEMA

**Severity:** low

**Problem:** If a selected platform has zero delivery configurations, the Delivery and Workflow selects are disabled and empty, yet both are now required. The user is correctly blocked (an order genuinely cannot be created), but the prompt asks them to select something they cannot.

**Impact:** Edge case (data misconfiguration). Behavior is arguably correct (block the impossible order) but the messaging is a dead-end. Pre-existing data condition, not introduced by this PR.

**Fix:** Optional/out-of-scope. Could show a distinct "No workflows configured for this platform" message when `workflowOptions.length === 0` rather than the generic required prompt.

---

## Validation Checks

Reused from the pre-commit run on this exact commit (`b237f36ff`); working tree unchanged since. User opted to reuse rather than re-run.

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ✅ | All pass except 1 pre-existing, **locale-only** failure in the untouched `packages/shared/utils/formatWordQuantity.test.ts` (`new Intl.NumberFormat()` with no locale → dev machine `en-IN` → "10,00,000"). Proven green under `LANG=en_US.UTF-8` (9/9) and on CI. New `schemas.test.ts` = 10/10 pass. |
| `npx turbo run typecheck` | ✅ | `yarn app:creative-portal typecheck` clean (only creative-portal touched). Full-monorepo turbo typecheck not separately re-run. |
| `npx turbo run lint` | ✅ | `eslint` clean on all 4 touched files; Husky `lint-staged` (eslint --fix + prettier) passed at commit. Full-monorepo turbo lint not separately re-run. |
| `npx turbo run build` | ✅ | `turbo run build` 4/4 successful (creative-portal compiled). |

---

## Tests

- ✅ New unit tests added (`schemas.test.ts`, 10 tests) covering: all-valid, each of Platform/Delivery/Workflow missing, Workflow id `0`, the three exact prompt messages (`it.each`), and the `getCreateOrderSchema(ORDER_SETUP_STEP)` gate.
- ✅ Tests assert behavior (schema validity + messages), not implementation.
- ⚠️ No component-level test for the render behavior (button enabled + error text appears on Next). This is hard to unit test (react-select + several React Query hooks) and was covered by manual browser verification instead. Acceptable given the schema is the source of truth and is unit-tested.
- ✅ Manual testing completed and screenshotted on `b2btest.proofed.com` for all three fields (blocks Next, shows prompt, clears on selection).
- ✅ Meets the "every PR includes tests for new code" requirement.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Fixes root cause (under-validated step); spinner path now unreachable |
| Regression risk | ✅ Low — schema/button changes confined to the New Order wizard; final-step submit behavior unchanged |
| Tests | ✅ 10 new unit tests; manual UI verification |
| Code quality | ✅ Minimal, idiomatic Formik validate-on-submit; clear messages |
| Validation suite | ✅ All pass (sole failure is an unrelated env-locale artifact, green on CI) |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve**

1. Ship as-is — it correctly and minimally resolves PP-1955 (block progression + prompt + no endless spinner), with tests and manual verification.
2. Optional, non-blocking follow-ups: (a) suppress or explain the enabled Next button's no-op click on the Customer Search step (Issue 1); (b) consider a distinct message for the zero-workflow-config edge case (Issue 3). Neither needs to hold this PR.
3. Reviewer note: the sole red test (`formatWordQuantity`) is a machine-locale artifact in an untouched file and passes on CI — not a blocker.
