# PR Review: feature/PP-2157: Project selection and fixed-unit amount on standalone pay adjustments

**PR:** https://github.com/Proofed/B2BWebserver/pull/2487
**Jira:** https://proofed.atlassian.net/browse/PP-2157
**Status:** Code Review (blocked by PP-2156, which is **Blocked** in Jira)

Reviewed at head `632c7979f`. The PR is stacked on #2486 (base `feature/PP-2156-standalone-pay-adjustment-project`).

---

## What this means for users (non-technical summary)

1. Once an admin picks a project in the new **Project (optional)** field on a standalone Pay adjustment, they can't take it back. The field has no way to clear it. The only way out is to change the Adjustment Type, which wipes the whole form, or to discard the modal.
2. The project is sent with the adjustment, but OMS does not save it yet (see PP-2156). Until the backend accepts it, a project an admin picks is not stored, even though the adjustment succeeds.
3. Everything else works as the ticket and Figma describe: standalone Fixed Pay now asks only for the Amount and always records a quantity of 1, and Hourly and Words behave as before.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
| --- | --- | --- |
| 1.1 Standalone Pay shows the same Project selector as standalone Charge, from the same list | `PayAdjustmentSection/index.tsx:131-142` renders `FormikSelect name="organizationGroupId"` labelled "Project (optional)", fed by the same `projectOptions` as Charge. It sits between User and Unit, which matches Figma node 77058-36694. | ✅ |
| 1.2 Selecting a project attaches it to the adjustment on submit | `hooks.ts:179-198` resolves `organizationId` from `projectOptions`. `buildCompensationPayload` (`utils.ts:207-230`) sends `organizationId` + `organizationGroupId` together or not at all, which matches the PP-2156 "provided together" rule. OMS does not yet store them (PR note; PP-2156 is Blocked). | ⚠️ Partial: sent but not yet persisted by OMS |
| 2.1 Pay + Unit Fixed: Quantity hidden, always 1 | Standalone Fixed hides Rate and Quantity (`PayAdjustmentSection/index.tsx:154-187`); the payload forces quantity 1 in all three fields (`utils.ts:205`). A job-linked Pay on a Fixed-unit job still shows an editable Quantity; the PR says this is deliberate (see Open Questions). | ⚠️ Standalone ✅; job-linked not covered |
| 2.2 Only the Amount is editable for Fixed | `index.tsx:189` sets `isAmountReadOnly={isPay && !isFixedPay}`, and Rate is hidden too, which matches Figma. `payRate` is sent as the absolute value of the amount (`utils.ts:222`). | ✅ (standalone) |
| Testing note 2: "greys out Quantity and shows a locked value of 1" | Quantity is hidden instead. This follows requirement 2.1 ("hidden") and Figma. The PR description attributes "locked value of 1" to 2.1, but those words are in testing note 2. | ⚠️ Ticket contradicts itself; PR follows 2.1 and Figma |
| Testing note 3: Hourly/Words keep an editable Quantity | `isStandaloneFixedPay` is false for units 60 and 1000, so the existing markup is unchanged. | ✅ |
| Validation rule: standalone Fixed cannot send a Quantity other than 1 | Enforced by the client payload builder (a test passes `quantity: 5` and expects 1). The BFF schema does not enforce it (see Open Questions). | ✅ client-side |

**Scope:** no scope creep. The only unrelated change is renaming `opt` to `option` in the `organizationId` lookup the PR moved, which follows the naming rule in CLAUDE.md.

---

## Architecture Analysis

One predicate, `isStandaloneFixedPay({ isNoOrder, unit })` (`utils.ts:181-187`), drives all four places the Fixed behaviour touches:

- the section hides Rate and Quantity;
- the modal makes Amount editable;
- the yup schema drops the Rate and Quantity requirement (`consts.ts:80-85`, `.when(["adjustmentType","unit"])`);
- the payload builder sends quantity 1 and rate = |amount|.

This is reuse, not duplication. I traced the interaction with `PayAmountCalculator` and the unit-change reset effect in the real code, and found no leak:

- **Words → Fixed:** the reset's `setValues(fn)` overwrites the calculator's stale write, so Rate and Quantity end up null and Amount "0.00".
- **Staying on Fixed:** the calculator is guarded by `values.rate`, which stays null, so it never overwrites a typed Fixed amount.

`buildCompensationPayload` moved to an options object. Its single production caller (`hooks.ts:194`) and all 14 test calls use the new form.

The Charge path's `organizationId` lookup moved above the Pay branch unchanged, so the Charge payload is identical.

**On the PR's question about the duplicated Project `FormikSelect`:** keep both.

- Figma places the field differently in each flow: after Adjustment Type for Charge, between User and Unit for Pay.
- The two selects differ only in `label`.
- Under the repo's folder rule, a `ProjectSelect` partial would need `index.tsx` + `types.ts` + a label prop. That is more code than the 12 lines it removes, and no creative-portal partial wraps a single labelled `FormikSelect`.

---

## Issues Found

### 1. The optional Project cannot be cleared once picked

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/partials/PayAdjustmentSection/index.tsx]**

> **In plain terms:** On a standalone Pay adjustment, the Project field is labelled optional, but once an admin picks a project there is no way to go back to "no project". To undo a mistaken pick they have to start the form again, and could easily attribute pay to the wrong project instead.

**Function/Class:** PayAdjustmentSection (Project `FormikSelect`)

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Log in to OMS as an admin and open **Adjust Pay or Charge** with no order (standalone).
2. Set Adjustment Type = Pay, pick a User, then pick any project in **Project (optional)**.
3. Try to remove it: click into the select and press Backspace, or look for a clear (×) control.
4. **Expected:** an optional field can be returned to "Select Project", so the adjustment is sent with no project.
5. **Actual:** the project stays selected. The only escape is changing Adjustment Type, which resets User, Unit, Amount and Reason, or Discard.

**Problem:** The new select at `PayAdjustmentSection/index.tsx:133-140` passes no clear option. The shared single select hard-codes `isClearable={false}` (`packages/shared/components/atoms/Fields/Select/index.tsx:353`), and react-select only clears a single value on Backspace when `isClearable` is set. The Charge Project select has the same limitation, but there the field is required, so it does not matter.

**Evidence:** `PayAdjustmentSection/index.tsx:133-140`:

```tsx
<FormikSelect
  name="organizationGroupId"
  label="Project (optional)"
  placeholder="Select Project"
  options={projectOptions}
  isSearchable
  isLoading={isFetchingProjects}
/>
```

`Select/index.tsx:353` is `isClearable={false}`, and `Select/index.tsx:443` is `: (newValue as SelectOption).value`.

**Impact:** Admins cannot correct a mis-picked project. Once OMS persists the field (PP-2156), the adjustment is attributed to a project the admin did not intend.

**Fix:** Passing `isClearable` alone is not enough. On clear, react-select calls `onChange(null)`, and `FormikSelect` would throw at `(newValue as SelectOption).value`. Two options:

- **Local and lowest risk:** prepend a "No project" option to the Pay list (value `null`) in `PayAdjustmentSection`.
- **Shared:** make `FormikSelect` null-safe and pass `isClearable` on this select. This touches `packages/shared` (used by both apps), but it is backward-compatible because no single select is clearable today.

```tsx
// packages/shared/components/atoms/Fields/Select/index.tsx
: ((newValue as SelectOption | null)?.value ?? null)
```

### 2. Two comments no longer describe the code next to them

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/partials/PayAdjustmentSection/index.tsx]**

> **In plain terms:** Two explanatory notes in the code now describe the screen as it was before this change. Nothing a user sees is affected. It only misleads the next developer who reads them.

**Function/Class:** PayAdjustmentSection docblock; `getAdjustPayOrChargeFormSchema`

**Severity:** low

**Confidence:** high

**How to spot it:** code health only, not reproducible by a user.

- Read `PayAdjustmentSection/index.tsx:25-39`.
- Read `consts.ts:62-87`.

**Problem:**

- **PayAdjustmentSection docblock (`PayAdjustmentSection/index.tsx:25-39`):**
  - It still says the standalone flow's "first field is the User select and the Unit is picked by the admin", and that "The Unit / Rate / Quantity markup below is identical for both flows".
  - The PR adds a standalone-only Project select and hides Rate and Quantity for standalone Fixed, so both statements are now false. The diff never touches this docblock.
- **Schema comment (`consts.ts:62-87`):**
  - The new comment and `isQuantityDrivenPay` (lines 77-85) sit between the PP-1947 comment that explains `amountSchema` (lines 62-76) and `amountSchema` itself (line 87).
  - With no blank line between line 76 and line 77, the two comments read as one block introducing `isQuantityDrivenPay`, and `amountSchema` loses its explanation.

**Impact:** Misleading documentation in a money-handling form. CLAUDE.md and team feedback require comments to describe the current state.

**Fix:**

- Update the docblock's standalone bullet to mention the Project select and the Fixed exception, and qualify the "identical" sentence.
- Move `isQuantityDrivenPay` and its comment above line 62, or next to the `rate`/`quantity` rules at around line 172.
- Optional: `isQuantityDrivenPay` is also true for a job-linked Fixed job, so `requiresRateAndQuantity` would describe it more accurately.

### 3. Two tests don't test what their titles say

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/utils.test.ts]**

> **In plain terms:** Two automated checks look like they protect behaviour that they don't actually check. If that behaviour broke later, these checks would still pass.

**Function/Class:** `buildCompensationPayload` tests; modal `it.each` for Hourly/Words

**Severity:** low

**Confidence:** high

**How to spot it:** test health only, not reproducible by a user.

- Read `utils.test.ts:155-180`.
- Read `index.test.tsx:498-516`.

**Problem:**

- **`utils.test.ts:155-180` ("maps standalone Pay (omits jobId, reason-only description)"):**
  - It uses `unit: 1, rate: 50, quantity: 1, amount: "50.00"` with `isNoOrder: true`. Since this PR, unit 1 is Fixed, so the test now runs the Fixed branch.
  - Its `payRate: 50` comes from the amount and its quantity 1 is forced, not taken from the form; the fixture's rate and quantity happen to equal those values.
  - It no longer checks that a standalone payload maps Rate and Quantity.
- **`index.test.tsx:498-516` (the Hourly/Words `it.each`):**
  - It is titled "keeps Rate and Quantity and a calculated Amount", but it only asserts `input-rate` and the Amount's read-only flag.
  - Quantity (`cell-quantity` for Hourly, `input-quantity` for Words) is never asserted, so hiding Quantity for every unit would still pass.

**Impact:** Weaker regression protection on the exact paths this PR changes.

**Fix:**

- Change the first fixture to `unit: WORDS_UNIT_VALUE` (or Hourly), or rename it to a Fixed case.
- Add the Quantity assertion per unit:

```ts
it.each([
  ["Hourly", HOURLY_UNIT_VALUE, "cell-quantity"],
  ["Words", WORDS_UNIT_VALUE, "input-quantity"]
])(
  "Standalone Pay (%s): keeps Rate and Quantity and a calculated Amount",
  (_unitLabel, unitValue, quantityTestId) => {
    // ...
    expect(screen.getByTestId(quantityTestId)).toBeInTheDocument();
  }
);
```

---

## Open Questions

- **Job-linked Pay on a Fixed-unit job.** Requirements 2.1 and 2.2 say "When Adjustment Type is Pay and Unit is Fixed", with no standalone qualifier. A Pay adjustment linked to a job whose task is Fixed still shows editable Rate and Quantity and can send a quantity other than 1. The PR scopes this out on purpose, and `consts.test.ts` "still requires Rate and Quantity for a job-linked Pay on a Fixed job" pins it. Can the PO confirm job-linked is out of scope? (`utils.ts:181-187`, `PayAdjustmentSection/index.tsx:154`)
- **Hidden or locked "1"?** Requirement 2.1 and Figma say hidden, but testing note 2 says "greys out Quantity and shows a locked value of 1". Can the PO update the testing notes so QA doesn't fail the hidden field?
- **Can the list offer deactivated projects?** The picker loads with `useAllPartnersAndProjectsQuery(undefined)`, which sends no status. The PP-2156 route only accepts an active project (`findActiveOrganisationGroup`, status `"A"`) and otherwise returns a 400, "The selected project does not exist or is not active", shown as a toast. If OMS returns deactivated groups when no status is sent, an admin can pick a project that can never be submitted. Does it? The Charge picker shares the list. (`hooks.ts:76-79`, `api/mixtures/partnerProjects/utils.ts`)
- **Server-side quantity rule.** The ticket rule "cannot send a Quantity other than 1" holds in the UI only. `createCompensationSchema` (PP-2156, `api/compensations/schema.ts:41-45`) accepts any quantity for `payUnit: 1` without a `jobId`. Is client-side enforcement enough, or should #2486 add a schema test?

---

## Validation Checks

| Check | Result | Notes |
| --- | --- | --- |
| `npx turbo run test` | ⏭️ Skipped | User opted out. The PR reports 161 affected tests passing; not re-run here. |
| `npx turbo run typecheck` | ⏭️ Skipped | User opted out. The PR reports creative-portal typecheck clean. |
| `npx turbo run lint` | ⏭️ Skipped | User opted out. The PR reports lint clean. Prettier (70 cols) was checked on all 10 files at PR head and passes. |
| `npx turbo run build` | ⏭️ Skipped | User opted out. The PR's "Build successful" checkbox is unchecked. |

No CI check runs are attached to the PR.

---

## Tests

- ✅ Payload: standalone Fixed forces quantity 1 in all three fields, even when the form holds `quantity: 5`, and `payRate` = |amount| (−16 → 16).
- ✅ Payload: the project is sent both-or-neither (none selected, organization unresolved, job-linked).
- ✅ Schema: standalone Fixed does not require Rate/Quantity; standalone Words and job-linked Fixed still do.
- ✅ Modal: the Project select appears for standalone Pay only; Fixed hides Rate and Quantity and makes Amount editable; the end-to-end Fixed submit sends 143/407, quantity 1, payRate 16. The mocks emit the queried test ids, so the `toBeNull()` checks would fail if the change were reverted.
- ⚠️ Issue 3: one old payload test now runs the Fixed branch by accident, and the Hourly/Words test never asserts Quantity.
- ⚠️ No test covers the standalone **Charge** submit. `createCharge`'s `mutateAsync` is an inline `vi.fn()` that no test captures (`index.test.tsx:63-69`), so the `organizationId` lookup this PR moved is only tested on the Pay side. The gap existed before this PR, but this PR edited that code.
- ⚠️ Not covered: job-linked Pay on a Fixed job (the fixture job is Words), the Words → Fixed switch (`PayAmountCalculator` is mocked), and a negative Fixed amount through the modal (covered in `utils.test.ts` only).
- ⚠️ `createCompensation.mockClear()` runs only inside the last test (`index.test.tsx:519`). Move it to a `beforeEach` so a future test cannot see a leftover call.

### Suggested manual QA script

1. **Req 1.1:** Standalone → Pay → pick a User. **Project (optional)** appears between User and Unit and lists the same projects as the Charge flow.
2. **Req 1.2, validation rule:** pick a project, set Unit = Fixed, type 16.00, choose a Reason, click Adjust. In DevTools, `POST /api/compensations` should carry `organizationId`, `organizationGroupId`, `payRate: 16`, and all three quantities = 1. The project is not persisted until OMS accepts it (PP-2156).
3. **Req 2.2, deduction:** repeat with the amount toggled to minus. The payload should have `amount: -16` and `payRate: 16`.
4. **Unit switching:** on Words, enter a Rate and a Quantity, then switch to Fixed. Rate and Quantity disappear, and Amount resets to 0.00 and is editable. Switch to Hourly: Rate and Quantity return empty, and Amount is read-only at 0.00.
5. **Issue 1:** pick a project, then try to clear it (Backspace or a × control). It cannot be cleared today.
6. **Testing note 3:** Hourly and Words still show an editable Quantity.
7. **Job-linked:** open the modal from an order → Pay → the Project field is absent. On a Fixed-unit job, Quantity is still shown; confirm with the PO that this is expected (Open Questions).
8. **Charge regression, no automated test:** Standalone → Charge → pick a project and submit. The request still carries `organizationId` + `organizationGroupId`.

---

## Summary

| Aspect | Status |
| --- | --- |
| Correctness | ✅ The standalone flow is correct. Project persistence is blocked on OMS (PP-2156). |
| Regression risk | ✅ Low. The single caller of the changed signature is updated, and the Charge path is unchanged. |
| Tests | ⚠️ Good core coverage; the gaps are listed in Issue 3 and Tests. |
| Accessibility | ✅ Reuses the labelled shared `FormikSelect`. |
| Error handling | ✅ A server 400 for an invalid project reaches the admin as a toast via `showDefaultErrorToast`. |
| Security | ✅ The project pair is validated server-side by PP-2156. `/security` is still required; the PR checkbox is unchecked. |
| Code quality | ⚠️ Two outdated comments (Issue 2). |
| Validation suite | ⏭️ Skipped (user opted out). Run before merge. |
| Mergeable state | ✅ GitHub reports clean. Stacked on #2486: retarget to `develop` once it merges, and expect a conflict with PP-2175 in `PayAdjustmentSection/index.tsx` and `utils.ts`. |

---

## Recommendation

**Approve.** Only medium and low findings. Fix Issue 1 before merge if possible.

1. Make the optional Project clearable (Issue 1). A "No project" option is the local, low-risk route.
2. Update the PayAdjustmentSection docblock, and move `isQuantityDrivenPay` off the PP-1947 amount comment (Issue 2).
3. Fix the two misleading tests (Issue 3), and consider adding a standalone Charge submit test for the moved `organizationId` lookup.
4. Get PO answers on the Open Questions: job-linked Fixed scope, and hidden vs locked "1".
5. Before merge, run `/security` and the validation suite (`test`/`typecheck`/`lint`/`build` for `@proofed/creative-portal`); neither was run in this review. Merge only after #2486 lands, and retarget to `develop`.
