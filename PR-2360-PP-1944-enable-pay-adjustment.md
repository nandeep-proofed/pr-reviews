# PR Review: feature/PP-1944: Enable Pay adjustment type in Adjust Pay or Charge modal

**PR:** https://github.com/Proofed/B2BWebserver/pull/2360
**Jira:** https://proofed.atlassian.net/browse/PP-1944
**Status:** In Progress
**Head:** `feature/PP-1944-enable-pay-adjustment` @ `31982fbd5` → `develop`
**Size:** 23 files, 16 commits, +1250 / −89 · `mergeable_state: clean`

> **Review note (v3.1 — full re-review, supersedes v1/v2):** Since v2, two ticket updates from the PO (Orlin) landed and are now implemented: **(point 1)** the finalised Reason table — flow-specific Pay reasons + a conditional "Additional details" requirement; and **(point 3)** the standalone **User** field changed from "active editors" to **active users**, fetched up front (OMS User Search status filter) to match the job-assignment dropdown UX. The "no negative quantity" instruction is covered by keyboard **and paste** guards. **Update (v3.1):** review Issues **2 (paste guards)** and **6 (PR description)** are now resolved — see below.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1.1 Add "Pay" to Adjustment Type, both flows | `adjustmentTypeOptions` includes Pay; enabled in both | ✅ |
| 1.2 Charge default | Opens on "Select Adjustment Type" placeholder (design-confirmed deviation) | ⚠️ Superseded (deliberate) |
| 2.1 USD, not changeable | `ChargeCurrencySync` forces `chargeCurrencyCode = "USD"` for Pay | ✅ |
| 2.2 Unit+Rate+Quantity → auto Amount, not editable | `PayAmountCalculator` → `calculatePayAmount` = `rate × (qty / unit)` | ✅ |
| 2.3 +/- sign, negatives allowed, zero not | `applyAmountSign` preserves sign; Rate/Quantity blocked from negative (keyboard + paste) so the toggle owns the sign; amount≠0 (form + BFF) | ✅ |
| 2.4 Quantity → 3 quantity fields | `buildCompensationPayload` writes quoted/userEntered/approved | ✅ |
| 2.5 Description ≤200, Reason(+details), colon/pipe | `composeAdjustmentDescription` (colon No-Order / pipe order-linked); ≤200 (schema + BFF) | ✅ |
| 3. Job-linked Pay: ellipsis, Job, derived Unit, Rate/Quantity | `PayAdjustmentSection`: gate on Job → derived disabled Unit → Rate/Quantity; editor from `selectedJob.proofedUserId` | ✅ |
| 4. Standalone Pay: "+", **active-user** search, selectable Unit Hourly/Words/Fixed | `PayStandaloneSection` + `hooks.ts`: active-user list fetched up front (`searchBy: "status"`, `userStatus: "A"`), react-select filters by name; Unit gates Rate/Quantity | ✅ |
| 4. "Project (optional)" standalone | Deferred per ticket; no Compensation field | ✅ Correctly omitted |
| 5. **Pay reasons per flow + conditional Description (table)** | `jobLinkedPayReasonOptions` (6) / `noOrderPayReasonOptions` (12); `payReasonRequiresDetails` + schema `.when([adjustmentType, reason])` make Details required for the "Y" rows | ✅ |
| 6. Pay → compensation; toast+close; Discard/X | `handleSubmit` Pay branch → `createCompensation`; success toast + `onClose` | ✅ |
| API: `POST /api/Compensations`, EntryType "A", USD, jobId only when job-linked | BFF `pages/api/compensations` → `addCompensation` → `COMPENSATION`; payload per OMS spec | ✅ |

**Reason table cross-check (req 5):** verified row-by-row against the ticket. Job-linked = Over Payment / Under Payment / Deduction / Unsupported Service / Bonus / Other (all require Details). No-Order = Order Assignment, Order Processing, Client Comms, Client Meeting, Style Guide Management, Team Management, Reviewer Meeting, Training, Other, Internal Training, Internal Meeting, 1:1 — only **Other / Internal Meeting / 1:1** require Details. The `payReasonRequiresDetails` set matches exactly. ✅

No scope creep beyond the deliberate, confirmed deviations.

---

## Architecture Analysis

- **Service / BFF** mirror the Charge pattern exactly: `services/compensations` (`useCreateCompensationMutation`) → `pages/api/compensations` → `api/compensations` handler (`makeMethodToHandlerMapping` + `withApiMiddleware` + Yup body schema) → `addCompensation` (`prepareServiceAxiosConfigWithData`). Registry `compensation → "COMPENSATION"` confirmed.
- **Modal** keeps `index.tsx` UI-only; logic in `hooks.ts` + partials. Job-linked (`PayAdjustmentSection`) and standalone (`PayStandaloneSection`) share the same field components and behaviour — gate Rate/Quantity (on Job vs Unit), placeholder→affix display, reset-on-change, keyboard + paste guards.
- **Reuse:** both flows use the order-create cells `EditableInputTextCell` / `HourlyInputTextCell`, made usable outside react-table (`Partial<CellProps>` + prop-forwarding) — **backward-compatible** (order-create `QuantityCell` test still green). Keyboard/paste guards (`allowOnlyDigitsKeyDown`/`Paste`, `blockNegativeNumberKey`/`Paste`) live once in `utils.ts`.
- **Standalone User search (point 3)** uses the same OMS **User Search** API as the assignment flow, parameterised for "all active users" (`searchBy: "status"`, `userStatus: "A"`) since No-Order Pay has no org group and req 4.3 wants any active user. The list is fetched on modal open; react-select filters client-side — so the dropdown is pre-populated like the assignment UI.
- **Pure helpers** (`calculatePayAmount` / `applyAmountSign` / `buildCompensationPayload` / `composeAdjustmentDescription` / `payReasonRequiresDetails`) extracted for unit testing (42 tests).

---

## Issues Found

### 1. Standalone Pay loads the full active-user list up front (no scope)

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/hooks.ts]**

**Function/Class:** useAdjustPayOrChargeModal — `editors` query

**Severity:** low-medium

**Problem:** `useUsersQuery({ searchBy: "status", userStatus: "A" })` fetches **every active user** when the No-Order modal opens, then react-select filters client-side. This matches the assignment-dropdown UX (pre-populated), but the job-assignment flow bounds its list by `organizationGroupId` + role; standalone Pay has neither, so the payload is the entire active-user set.

**Impact:** Fine for a modest user count; if the active-user population is large, this is a heavier initial fetch + a large option list for react-select to render/filter. No correctness issue.

**Fix:** Acceptable as-is per the requirement (active users, assignment-style list). If scale becomes a concern, switch to a debounced server search (`searchBy: "name"` + `userStatus: "A"`) with a low character threshold, or virtualise the option list.

### 2. Keyboard guards don't cover paste / programmatic input — ✅ Resolved (commit `31982fbd5`)

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/utils.ts]**

**Function/Class:** `allowOnlyDigitsPaste`, `blockNegativeNumberPaste`

**Severity:** low

**Problem:** The `onKeyDown` guards blocked *typing* a negative Rate / non-digit Quantity but not a **pasted** value — a pasted `-16` could reintroduce a negative magnitude and double the Amount sign (the +/- toggle already owns the sign).

**Resolution:** Added `onPaste` guards mirroring the order-create `QuantityCell` — `allowOnlyDigitsPaste` (Quantity: digits only) and `blockNegativeNumberPaste` (Rate: digits + one optional decimal; blocks sign/exponent/symbol) — wired to Rate/Quantity in both `PayAdjustmentSection` and `PayStandaloneSection`. The calculator was intentionally left unchanged (no `Math.abs` — the earlier decision was to block negative *entry* instead). 4 unit tests added (modal suite now 42).

### 3. Rate "$" prefix relies on a key-remount workaround

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/partials/PayAdjustmentSection/index.tsx & PayStandaloneSection/index.tsx]**

**Function/Class:** Rate field `key`

**Severity:** low (info)

**Problem:** The Rate field is remounted (`key` keyed on job/unit presence) to force the shared `usePrefixInputIndent` hook to re-measure the `$` prefix width — that hook only measures on mount, so a conditionally-rendered prefix would otherwise overlap the value.

**Impact:** Works; localized workaround for a shared-hook limitation.

**Fix (optional):** Make `usePrefixInputIndent` re-measure when the prefix element appears (callback ref) — removes the remount, app-wide.

### 4. Non-null assertions in the payload builder

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/utils.ts]**

**Function/Class:** buildCompensationPayload

**Severity:** low

**Problem:** `proofedUserId: values.proofedUserId as number` and `payUnit: values.unit as number` cast away `null`. Safe — Pay validation guarantees them before submit — but bypasses TS null-safety.

**Fix:** Optional — narrow instead of asserting. (Note: `Number(null)` → `0` is a valid-looking id, so a naive narrow would be a subtle regression; keep the validation contract or add an explicit guard.)

### 5. No component/render tests for the new UI behaviour

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/]**

**Function/Class:** modal component behaviour

**Severity:** low-medium

**Problem:** Pure logic is well covered (42 unit tests, incl. the new per-flow reasons + conditional-details rules). The **UI behaviour** — gate-on-Job/Unit, reset-on-change, placeholder↔affix display, keyboard/paste guards, type-switch reset, the pre-populated active-user select, and per-flow Reason rendering — has no render tests; verified live only.

**Fix:** Add a `@testing-library/react` modal test (mock queries/Formik) covering the section swaps, gating, reason-list-by-flow, and details-required.

### 6. PR description is stale — ✅ Resolved

**[File: PR #2360 description]**

**Severity:** low (housekeeping)

**Problem:** The PR body still described the pre-update state — Pay reasons "Overpayment/Underpayment/Deduction/Bonus", a name type-ahead User search, and an open "filter to editor role?" follow-up. Points 1 & 3 changed all three.

**Resolution:** PR description refreshed to reflect the per-flow reasons + conditional details, the active-user list (fetched up front), and the keyboard + paste guards; the resolved follow-up was removed.

---

## Validation Checks

Run via `npx turbo run test typecheck lint` against the PR branch (base `8294fca68`; the Issue-2 paste fix `31982fbd5` re-ran green for typecheck/lint + the modal suite, now 42).

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ✅ | Modal suite **42** + order-create `QuantityCell` **6** pass. Only failure is the **pre-existing, unrelated** `@proofed/shared/utils/formatWordQuantity` locale test (1308/1309 in shared) — not touched by this PR. |
| `npx turbo run typecheck` | ✅ | 0 errors across all workspaces (shared-cell type changes compile order-create too). |
| `npx turbo run lint` | ✅ | 0 errors. |
| `npx turbo run build` | ⚠️ Deferred to CI | Not run this pass; the local build has been flaky all session with post-compile, code-unrelated failures (`@emotion jsxDEV` / `MODULE_NOT_FOUND` in untouched files) after `✓ Compiled successfully`. Confirm via CI (clean container). |

---

## Tests

- ✅ `utils.test.ts` / `consts.test.ts` — **42 tests**: calculator (Words/Hourly/Fixed), `applyAmountSign`, `buildCompensationPayload` (job-linked vs standalone per OMS spec, quantity→3 fields, string coercion, negative deduction, jobId omission), schema rules (amount≠0, proofedUserId/jobId/unit/rate/quantity required, 200-char Description), `composeAdjustmentDescription`, **per-flow reason options**, **`payReasonRequiresDetails`**, **conditional Details-required** validation, and the **keyboard + paste guards**.
- ✅ Order-create `QuantityCell` (6) still passes → the shared-cell changes didn't regress.
- ⚠️ No render tests for the modal UI behaviour (Issue 5).
- ✅ Live-verified both flows (gating, placeholders/affixes, reset-on-change, USD, negative-entry blocked, deduction via toggle, per-flow reasons, pre-populated active-user select).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Matches the updated ticket (reasons table + active users) and OMS spec; Charge unchanged |
| Regression risk | ✅ Low — additive; shared-cell changes backward-compatible (QuantityCell green) |
| Tests | ⚠️ Strong unit coverage (incl. new reason/details + paste-guard rules); UI render tests absent (Issue 5) |
| Code quality | ✅ Good; remaining items are low / low-medium (Issues 1, 3–5) |
| Validation suite | ✅ test / typecheck / lint pass; build deferred to CI |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions — pending a green CI build.**

1. **Build gate:** confirm a green build in CI (local failures were flaky/code-unrelated; not a code blocker).
2. ✅ **(Issue 2 — done, `31982fbd5`)** Paste guards added to Rate/Quantity.
3. ✅ **(Issue 6 — done)** PR description refreshed.
4. **(Issue 1, low-med)** Standalone Pay loads the full active-user list — fine per the requirement; revisit (debounced server search / virtualised list) only if the active-user count grows large.
5. **(Issues 3–4, low)** Optional: fix `usePrefixInputIndent` to drop the key-remount; narrow the payload assertions.
6. **(Issue 5, low-med)** Add a modal render test for the gating/reset/per-flow reasons/details-required.
7. **Process:** run `/security` before merge (per workflow).
