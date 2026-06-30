# PR Review: feature/PP-1944: Enable Pay adjustment type in Adjust Pay or Charge modal

**PR:** https://github.com/Proofed/B2BWebserver/pull/2360
**Jira:** https://proofed.atlassian.net/browse/PP-1944
**Status:** In Progress
**Head:** `feature/PP-1944-enable-pay-adjustment` @ `11e052da1` → `develop`
**Size:** 23 files, 12 commits, +1088 / −85 · `mergeable_state: clean`

> **Review note (v2):** Re-review after extensive UI refinement since the first report. Original Issues **1, 2, 5 are resolved**; new work added (standalone gate-on-Unit, shared-cell reuse, input guards, unit/job-change resets). New/remaining items below.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1.1 Add "Pay" to Adjustment Type, both flows | `adjustmentTypeOptions` includes Pay; enabled in both | ✅ |
| 1.2 Charge default | Placeholder "Select Adjustment Type" instead (design-confirmed deviation) | ⚠️ Superseded (deliberate) |
| 2.1 USD, not changeable | `ChargeCurrencySync` forces USD for Pay | ✅ |
| 2.2 Unit+Rate+Quantity → auto Amount, not editable | `PayAmountCalculator` → `calculatePayAmount` | ✅ |
| 2.3 +/- sign, negatives allowed, zero not | Toggle owns sign; `applyAmountSign`; Rate/Quantity blocked from negative so the toggle is the only sign source; amount≠0 (form+BFF) | ✅ |
| 2.4 Quantity → 3 quantity fields | `buildCompensationPayload` | ✅ |
| 2.5 Description ≤200, Reason(+details), colon/pipe | `composeAdjustmentDescription`; ≤200 (form+BFF) | ✅ |
| 3. Job-linked Pay: ellipsis, Job, derived Unit, Rate/Quantity | `PayAdjustmentSection`: gate on Job → derived disabled Unit → Rate/Quantity (placeholders→affixes); editor from `selectedJob.proofedUserId` | ✅ |
| 4. Standalone Pay: "+", User search, selectable Unit Hourly/Words/Fixed | `PayStandaloneSection`: User type-ahead → selectable Unit → gate Rate/Quantity on Unit | ✅ |
| 4. "Project (optional)" standalone | Deferred per ticket + no Compensation field | ✅ Correctly omitted |
| 5. Pay reasons | `payReasonOptions` | ✅ |
| 6. Pay → compensation; toast+close; Discard/X | `handleSubmit` branch; BFF | ✅ |
| API: `POST /api/Compensations`, EntryType "A", USD | BFF `pages/api/compensations` → `addCompensation` → `COMPENSATION` | ✅ |

No scope creep beyond the deliberate, confirmed deviations.

---

## Architecture Analysis

- **BFF/service** mirror the Charge pattern exactly (`withApiMiddleware` auth, Yup body schema, `prepareServiceAxiosConfigWithData("compensation", …)`); registry `COMPENSATION` confirmed.
- **Modal** keeps `index.tsx` thin; job-linked (`PayAdjustmentSection`) and standalone (`PayStandaloneSection`) now share the **same field components and behaviour** — gate Rate/Quantity (on Job vs Unit), placeholder-then-affix display, reset-on-change, keyboard guards.
- **Reuse:** both flows use the order-create cells `EditableInputTextCell` / `HourlyInputTextCell`; these were made usable outside tables (`Partial<CellProps>` + prop-forwarding) — **backward-compatible** (order-create `QuantityCell` test still green). Keyboard guards (`allowOnlyDigitsKeyDown`, `blockNegativeNumberKey`) live once in `utils.ts`, used by both.
- **Pure helpers** (`calculatePayAmount`/`applyAmountSign`/`buildCompensationPayload`/`composeAdjustmentDescription`) unit-tested (34 tests).

---

## Issues Found

### 1. Standalone editor search — restricted to editors (original Issue 1 resolved), but mechanism differs from the standard

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/hooks.ts]**

**Function/Class:** useAdjustPayOrChargeModal — `editorOptions`

**Severity:** medium

**Problem:** The search uses `useUsersQuery({ searchBy: "name" })` + client filter `active !== false && roles?.includes(UserRole.Editor)`. The role filter **resolves the original "not restricted to editors" finding**. But the codebase's standard editor search (job-card assignment) is `useSearchForAssignment({ role: Editor, organizationGroupId })` — server role-scoped, and **requires an `organizationGroupId` the No-Order Pay flow doesn't have** (Project deferred; Compensation has no project). So that flow can't be reused as-is. Eligibility also differs: the assignment BFF treats users eligible via `onboardingComplete && !deleted`, here it's `active !== false`.

**Impact:** Functional and editor-restricted. Residual risk: differing eligibility semantics could include/exclude a user the assignment flow wouldn't; and the name-search-then-client-filter relies on the server name search returning the right editors.

**Fix:** Either align eligibility (`onboardingComplete && !deleted`), or — if a project/org context is introduced for standalone Pay — adopt `useSearchForAssignment` for parity. Tracked; not a blocker.

### 2. Keyboard guards don't cover paste / programmatic input

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/utils.ts]**

**Function/Class:** `allowOnlyDigitsKeyDown`, `blockNegativeNumberKey`

**Severity:** low

**Problem:** Both guard `onKeyDown` only. A **pasted** negative (or one set programmatically) bypasses them. The order-create `QuantityCell` also guards `onPaste`; the Pay fields don't. A negative Rate/Quantity reintroduces the doubled-sign Amount these guards exist to prevent.

**Impact:** Edge (requires paste). The +/- toggle would then show a doubled sign in the Amount.

**Fix:** Add an `onPaste` guard, or keep a safety net in the calculator:

```typescript
roundToCurrency(Math.abs(rate * (quantity / unit)));
```

so a stray negative magnitude can't double the toggle sign.

### 3. Rate "$" prefix relies on a key-remount workaround

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/partials/PayAdjustmentSection/index.tsx & PayStandaloneSection/index.tsx]**

**Function/Class:** Rate field `key`

**Severity:** low (info)

**Problem:** The Rate field is remounted (`key` on job/unit presence) to force the shared `usePrefixInputIndent` hook to re-measure the `$` prefix width — that hook only measures on mount, so a conditionally-rendered prefix would otherwise overlap the value.

**Impact:** Works; it's a localized workaround for a shared-hook limitation.

**Fix (optional):** Fix `usePrefixInputIndent` to re-measure when the prefix element appears (callback ref) — removes the need for the remount, app-wide.

### 4. Non-null assertions in the payload builder

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/utils.ts]**

**Function/Class:** buildCompensationPayload

**Severity:** low

**Problem:** `proofedUserId: values.proofedUserId as number` and `payUnit: values.unit as number` cast away null. Safe — Pay validation guarantees them before submit — but bypasses TS null-safety.

**Fix:** Optional — narrow instead of asserting.

### 5. No component/render tests for the new UI behaviour

**[File: apps/creative-portal/components/organisms/modals/AdjustPayOrChargeModal/]**

**Function/Class:** modal component behaviour

**Severity:** low-medium

**Problem:** Pure logic is well covered (34 unit tests). The substantial **UI behaviour** added — gate-on-Job/Unit, reset-on-change, placeholder↔affix display, keyboard guards, type-switch reset — has no render tests; verified live only.

**Impact:** A future refactor could silently regress these without a failing test.

**Fix:** Add a `@testing-library/react` modal test (mock queries/Formik) for the section swaps, gating, and reset.

---

## Validation Checks

Recent results this session at/near head `11e052da1` (used per reviewer choice rather than a fresh full run):

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ✅ | Modal suite 34 + order-create `QuantityCell` 6 pass; only the **pre-existing, unrelated** `@proofed/shared/utils/formatWordQuantity` locale test fails (not touched by this PR). |
| `npx turbo run typecheck` | ✅ | 0 errors across the app (shared-cell type changes compile order-create too). |
| `npx turbo run lint` | ✅ | 0 errors. |
| `npx turbo run build` | ⚠️ Deferred to CI | Flaky locally this session — 3 different **post-compile, code-unrelated** failures (`MODULE_NOT_FOUND` / `@emotion jsxDEV` in untouched files) after `✓ Compiled successfully`. Confirm via CI (clean container). |

---

## Tests

- ✅ `utils.test.ts` / `consts.test.ts` — 34 tests (calculator incl. Fixed, sign handling, payload mapping per OMS spec, schema rules, description composer, null defaults, decimal/required rules).
- ✅ Order-create `QuantityCell` (6) still passes → shared-cell changes didn't regress.
- ⚠️ No render tests for the new modal UI behaviour (Issue 5).
- ✅ Live-verified both flows (gating, placeholders/affixes, reset on change, USD, negative-entry blocked, deduction via toggle).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Matches behaviour + OMS spec; Charge unchanged |
| Regression risk | ✅ Low — additive; shared-cell changes backward-compatible (QuantityCell green) |
| Tests | ⚠️ Strong unit coverage; UI render tests absent (Issue 5) |
| Code quality | ✅ Good; minor items (Issues 2–4); Issue 1 editor-restricted with a noted follow-up |
| Validation suite | ⚠️ test/typecheck/lint ✅ (recent); build deferred to CI |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions — pending a green CI build.**

1. **Build gate:** confirm a green build in CI (local failures were flaky/code-unrelated; do not treat as a code blocker).
2. **(Issue 1, medium)** Editor search is now editor-restricted; align eligibility with the assignment flow (`onboardingComplete && !deleted`) or adopt `useSearchForAssignment` if an org context is added.
3. **(Issue 2, low)** Add an `onPaste` guard (or a `Math.abs` safety net) so a pasted negative can't double the Amount sign.
4. **(Issues 3–4, low)** Optional: fix `usePrefixInputIndent` to drop the key-remount; narrow the payload assertions.
5. **(Issue 5, low-med)** Add a modal render test for the gating/reset/section-swap.
6. **Process:** run `/security` before merge (per workflow).
