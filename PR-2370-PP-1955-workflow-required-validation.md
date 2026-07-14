# PR Review: fix/PP-1955: Require Platform, Delivery & Workflow on Order Setup step

**PR:** https://github.com/Proofed/B2BWebserver/pull/2370
**Jira:** https://proofed.atlassian.net/browse/PP-1955
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Require a **Workflow** before letting the user advance — block progression and prompt for the missing selection | `deliveryConfigurationId` is now `Yup.number().min(1).required("Please select a Workflow")` in `ORDER_SETUP_STEP_SCHEMA`; step validation fails and the field-level prompt renders (`showErrorOnTouched` removed from the Workflow select) | ✅ Addressed |
| The user should **never be left on an endless spinner** | Blocking at Order Setup prevents reaching the submit path that produced the hang (where `deliveryConfigurationId` defaulted to `0` → invalid create request → silent `onError`) | ✅ Addressed (removes the reported cause) |
| "If a missing Workflow is only detected at submit, order creation should **fail gracefully with a clear message** and return the user to correct it" | Not addressed — the no-op `onError` in `services/orders/createOrderNew` and the `deliveryConfigurationId ?? 0` default in `api/orders/createNew/utils.ts:206/223` are unchanged. The PR relies solely on the upstream gate | ⚠️ Partial |
| (Implied) Same silent dead-end must not occur for Platform / Delivery | Platform now carries an explicit prompt message; **Delivery is newly validated** (`required`) — previously not in the schema at all | ✅ Addressed (slightly beyond ticket scope, reasonable) |

---

## Architecture Analysis

The wizard is a single Formik form (`NewOrderForm/index.tsx`) whose `validationSchema` is swapped per step via `getCreateOrderSchema(step)`. The Order Setup step uses `ORDER_SETUP_STEP_SCHEMA`. The bug was that this schema only required `organizationGroupId` + `platform`, so a blank Workflow (`deliveryConfigurationId`) left the form "valid enough" to advance, and the downstream create request then hung on a no-op error handler.

The fix is a clean, root-cause change at the right layer — the step schema — plus two supporting UI tweaks:

1. **`schemas.ts`** — adds `delivery` + `deliveryConfigurationId` as required (and a message on `platform`). This is the correct place: `getCreateOrderSchema(ORDER_SETUP_STEP)` is the only runtime consumer that gates this step, so the new rules apply exactly where intended and don't leak into later steps (`CHARGE_STEP`, `BRIEF_STEP` don't concat `ORDER_SETUP_STEP_SCHEMA`).
2. **`SetupForm/index.tsx`** — removes `showErrorOnTouched` on the three selects so the required error renders once validation populates it (these fields are never in `initialValues`, so they're never marked `touched`, which is why the old gate hid the error). It also adds `setFieldError(field, undefined)` inside the three auto-select effects so a field that auto-selects its single option clears any stale error (auto-select uses `setFieldValue`, which wouldn't otherwise clear the error the way the select's own `onChange` does).
3. **`SearchCustomerResultItem.tsx`** — adds `type="button"` to the customer-row button (it's a `theme-ui` `Button` → `<button>`, which defaults to `type="submit"` inside the `<Form>`, so a row click could accidentally submit the form). Sensible defensive fix, tangential to PP-1955.

**Behavioural mechanism (important — differs from the PR description).** The end-to-end flow still works, but *not* via the mechanism the PR body describes. See Issue 1. On arrival at Order Setup, `formik.errors` is empty (the previous step passed), so with `isInitialValid: false` + a dirty form, `isValid` is `true` and the Next button is **already enabled**. The first Next click (`type="submit"`) runs the step validation, which fails, populates the errors, marks the fields touched, and blocks progression — and the prompts render because `showErrorOnTouched` was removed. So the client's Expected Result is met, but through the existing button gating, not a change to it.

---

## Issues Found

### 1. PR description claims a change to `CreateOrderContent` that is not in the PR

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/CreateOrderContent/index.tsx]**

**Function/Class:** CreateOrderContent (Next button gating)

**Severity:** medium

**Problem:** The PR body states: *"`.../partials/CreateOrderContent/index.tsx` — Next button enabled (gated only on `isValidating`) so clicking it runs the step validation…"*. This file is **not** in the PR diff, and on the PR head commit (`626f0a3`) the button is still `disabled={formik.isValidating || !formik.isValid}`. The described "click Next → run validation" mechanism is not what actually happens.

**Impact:** Not a functional bug — the fix still works (see Architecture Analysis: the button is already enabled on arrival because `errors` is empty, so the first click triggers submit-validation; and even after a failed attempt the button disables but the error prompts remain visible, so there is no silent dead-end). The problem is that the PR record misdescribes the design. A reviewer who trusts the description will believe the button was intentionally made always-clickable, when in reality progression relies on the *pre-existing* gating plus the empty-errors-on-arrival invariant. This is a more implicit contract than stated and should be documented accurately so a future change (e.g. enabling `validateOnMount`, or a lingering error when landing on the step) is understood to only degrade to "button disabled but errors shown", never to a silent hang.

**Fix:** Correct the PR description to reflect the actual diff (4 files, no `CreateOrderContent` change) and the real mechanism. Optionally, if the team prefers the button to stay clickable so validation always fires on click (the originally-intended design), actually make that one-line change and add a test:

```tsx
// CreateOrderContent/index.tsx
<NextStepButton type="submit" disabled={formik.isValidating} />
```

Note this is optional — without it the behaviour is already correct.

### 2. Type change to `deliveryConfigurationId` / new `delivery` field — ripple not verified

**[File: apps/creative-portal/components/organisms/NewOrderForm/schemas.ts]**

**Function/Class:** ORDER_SETUP_STEP_SCHEMA / CreateOrderSchema

**Severity:** medium

**Problem:** `deliveryConfigurationId` was `Yup.number().optional()` in `REGULAR_ORDER_SCHEMA`; concatenating the new `ORDER_SETUP_STEP_SCHEMA` (`min(1).required()`) into `orderSchema` promotes it to a required `number` in the inferred `CreateOrderSchema`, and adds a newly-required `delivery: string`. `CreateOrderSchema` is consumed across the wizard (`useSelectedDeliveryConfiguration.ts`, `SetupForm`, `WorkflowStep`, the submit handler, etc.).

**Impact:** Potential typecheck/build breakage in consumers that previously relied on `deliveryConfigurationId` being optional, or that don't supply `delivery`. In practice the risk looks low — the base branch already reads `formik.values.delivery` (`SetupForm/index.tsx:39`) with no `@ts-ignore`, implying the concatenated type is already permissive, and the submit handler already uses `values.deliveryConfigurationId!`. But this was **not verified** because the validation suite was skipped (see below).

**Fix:** Run `npx turbo run typecheck --filter=@proofed/creative-portal` and `... build ...` against the PR branch before merge to confirm zero new type errors. No code change expected if it's clean.

### 3. Secondary Expected Result (graceful submit-time failure) is not addressed

**[File: apps/creative-portal/services/orders/createOrderNew/index.ts]**

**Function/Class:** useCreateOrderNewMutation `onError`

**Severity:** low

**Problem:** The ticket's Expected Result has two parts. The PR delivers part one (block + prompt). Part two — *"If a missing Workflow is only detected at submit, order creation should fail gracefully with a clear message and return the user to correct it"* — is untouched. The ticket's own technical findings call out the no-op `onError` and the `deliveryConfigurationId ?? 0` default (`api/orders/createNew/utils.ts:206,223`) as the reason the spinner hangs.

**Impact:** The reported scenario is fully fixed (you can no longer reach submit without a Workflow). However, the silent-failure safety net still doesn't exist: any *other* create failure at that submit path (network error, server 4xx/5xx, or a Workflow cleared by an edge-case back-navigation) would still spin indefinitely with no toast. This is defense-in-depth the PR chose to defer, which is defensible for a "Low priority" bug — but worth an explicit note / follow-up ticket rather than silently closing the second half of the AC.

**Fix:** Either note in the PR / ticket that the graceful-submit-failure path is intentionally out of scope (tracked separately), or add a minimal error surface in the mutation's `onError` (e.g. `showDefaultErrorToast(error)` and route to `ORDER_FAILED_STEP`) as a follow-up.

### 4. Tests cover the schema only — not the UI wiring the fix depends on

**[File: apps/creative-portal/components/organisms/NewOrderForm/schemas.test.ts]**

**Function/Class:** schemas.test.ts

**Severity:** low

**Problem:** The 10 new tests are good and correctly assert the schema rules and exact messages. But the *behavioural* correctness of the fix lives in the UI wiring — `showErrorOnTouched` removal surfacing the prompt, the auto-select `setFieldError(…, undefined)` clearing stale errors, and the Next button enabling/blocking — none of which is exercised by a schema-only test.

**Impact:** A regression in the wiring (e.g. someone re-adds `showErrorOnTouched`, or the auto-select error-clear is dropped) would keep all tests green while silently reintroducing a hidden-error or dead-end. Given the PR description already misstated the mechanism (Issue 1), a component-level test would both document and lock in the real behaviour.

**Fix:** Add an RTL test for `SetupForm` (or the Order Setup step) that renders with 2+ platform/delivery/workflow options, triggers the step submit/validation, and asserts the "Please select a …" prompt appears and progression is blocked; then asserts selecting each option clears its prompt. Not blocking, but recommended per the project's "every PR includes tests for new code" rule for the behavioural change.

### 5. Observation — `type="button"` fix is out of the stated PP-1955 scope

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/CustomerSearch/partials/SearchCustomerResultItem.tsx]**

**Function/Class:** SearchCustomerResultItem

**Severity:** low

**Problem:** Adding `type="button"` is a correct fix (prevents the customer-row `<button>` from defaulting to `type="submit"` and accidentally submitting the wizard form), but it's unrelated to Workflow validation and isn't mentioned in the Jira ticket.

**Impact:** None negative — it's a low-risk, correct hardening within the same form. Flagging only as minor scope creep so it's a conscious inclusion (a one-line note in the PR description would suffice).

**Fix:** No change needed; optionally mention it in the PR description's "Areas of Change".

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ Skipped | Skipped — user opted out |
| `npx turbo run typecheck` | ⏭️ Skipped | Skipped — user opted out (see Issue 2: the `deliveryConfigurationId` optional→required + new `delivery` type change is unverified) |
| `npx turbo run lint` | ⏭️ Skipped | Skipped — user opted out |
| `npx turbo run build` | ⏭️ Skipped | Skipped — user opted out |

---

## Tests

- ✅ New unit tests added (`schemas.test.ts`, 10 tests) covering the three required selections, the exact prompt messages, the `id === 0` invalid case, and the `getCreateOrderSchema(ORDER_SETUP_STEP)` gate.
- ⚠️ No component/behavioural test for the UI wiring the fix relies on (`showErrorOnTouched` removal, auto-select error clearing, Next-button gating) — see Issue 4.
- ⏭️ Full suite (`yarn test`) not run this review — Skipped — user opted out. PR author reports it passes locally (bar one pre-existing `en-IN` locale-only failure in the untouched `formatWordQuantity.test.ts`) and on CI.
- ⏭️ Typecheck/build not run — Skipped — user opted out; the type ripple in Issue 2 is therefore unverified.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Root cause fixed at the right layer; end result meets the primary AC |
| Regression risk | ⚠️ Medium — schema type change to a widely-consumed `CreateOrderSchema` is unverified (typecheck/build not run) |
| Tests | ⚠️ Solid schema tests; UI-behaviour test gap |
| Code quality | ✅ Clean, idiomatic, well-scoped |
| Validation suite | ⏭️ Skipped — user opted out (re-run before merge) |
| Mergeable state | ✅ Clean (GitHub `mergeable_state: clean`); validation not independently verified |

---

## Recommendation

**Approve with suggestions** — contingent on running validation before merge.

1. **Run the validation suite before merging** (it was skipped in this review). In particular `typecheck` + `build` on `@proofed/creative-portal`, to confirm the `deliveryConfigurationId` optional→required + new required `delivery` type change doesn't break any `CreateOrderSchema` consumer (Issue 2). This is a hard gate per CLAUDE.md.
2. **Fix the PR description** to match the actual 4-file diff and the real mechanism — `CreateOrderContent` was *not* changed; the Next button is still gated on `!formik.isValid` and works because errors are empty on arrival (Issue 1). Optionally make the intended one-line button change and cover it with a test.
3. **Add a component-level test** for the Order Setup prompt/block behaviour so the wiring the fix depends on is locked in (Issue 4).
4. **Note the deferred half of the AC** — graceful submit-time failure / the no-op `onError` is unchanged (Issue 3). Either state it's out of scope or open a follow-up.
5. Mention the `type="button"` customer-row fix in the PR description (Issue 5) — minor.

The core change is correct and well-targeted; the blockers to a clean approval are procedural (run validation) and documentation (accurate description), not the logic itself.
