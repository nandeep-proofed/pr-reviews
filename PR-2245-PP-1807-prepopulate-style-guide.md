# PR Review: PP-1807 — Prepopulate Style Guide on Order Creation

**PR:** #2245 (feature/PP-1807-prepopulate-style-guide-on-order-creation)
**Jira:** https://proofed.atlassian.net/browse/PP-1807
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| When project has exactly 1 style guide, prepopulate the field | `useEffect` auto-populates via `changeValue("styleGuide", ...)` when `styleGuideOptions?.length === 1` | ✅ Addressed |
| Prepopulated value must remain **editable** (user can clear/change) | Field is **disabled** (`isDisabled={... styleGuideOptions?.length === 1}`) | ❌ Contradicts Jira |
| Projects with 0 or 2+ style guides behave unchanged | Guards check `styleGuideOptions?.length !== 1` — correct | ✅ Addressed |
| Only project style guides may be prepopulated | Uses `styleGuideOptions` from `useStyleQuery(orgGroupId)` — already scoped to project | ✅ Addressed |
| No additional validation rules | No new validation added | ✅ Addressed |

---

## Architecture Analysis

The approach is clean: a single `useEffect` watches `isLoading` and `styleGuideOptions`, and when loading completes with exactly one option, it calls the existing `changeValue` helper to set the style guide on all relevant orders + `sharedBrief`. The `isDisabled` prop on the `Select` prevents user interaction when there's only one choice.

A secondary change removes the redundant `validateField` call from `changeValueOrder` — Formik's `setFieldValue` triggers validation by default, making the explicit call unnecessary.

---

## Issues Found

### 1. Disabled field contradicts Jira requirement "editable"

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/BriefStep/index.tsx]**
**Function/Class:** BriefStep JSX (Select for styleGuide)
**Severity:** high
**Problem:** Jira requirement 1.1.2 states *"The pre-populated value must remain editable"* and 1.1.3 states *"Users may clear or change the style guide before submission."* The PR adds `isDisabled={isLoading || styleGuideOptions?.length === 1}`, which locks the field when there's exactly one style guide. The test file references "Per Jira discussion on PP-1807" suggesting this was agreed upon outside the ticket, but the Jira description was never updated.
**Impact:** If the Jira description is authoritative, this blocks a required user flow (clearing/changing the style guide). If the Jira discussion overrides the description, then the description is misleading for QA testers.
**Fix:** Clarify with the team which behavior is intended. If the field should be disabled, update the Jira acceptance criteria to reflect the actual decision. If it should remain editable, remove the `isDisabled` prop:

```diff
- isDisabled={isLoading || styleGuideOptions?.length === 1}
```

### 2. `validateField` removal is unrelated scope creep

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/BriefStep/index.tsx]**
**Function/Class:** changeValueOrder
**Severity:** low
**Problem:** The removal of `validateField` from `changeValueOrder` is functionally correct (Formik's `setFieldValue` validates by default), but it's unrelated to the PP-1807 style guide feature. It changes behavior for every brief field (contentType, contentSubject, language, etc.), not just styleGuide.
**Impact:** Low risk since `setFieldValue` already validates, but mixing unrelated changes into a feature PR makes it harder to bisect if a regression appears.
**Fix:** This is a minor observation — acceptable to keep, but ideally would be a separate commit clearly marked as a refactor (which it is: `PP-1807: Remove redundant validateField call in changeValueOrder`). No action required if the team is comfortable.

### 3. useEffect missing deps with eslint-disable

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/BriefStep/index.tsx]**
**Function/Class:** useEffect (auto-populate style guide)
**Severity:** low
**Problem:** The `useEffect` depends on `changeValue` and `getCurrentValue` but excludes them from the dependency array via `eslint-disable-next-line react-hooks/exhaustive-deps`. This is intentional — including them would cause the effect to re-fire on every render since both are recreated each render cycle. However, a comment explaining *why* these are excluded would help future maintainers.
**Impact:** No functional bug — the effect correctly fires only when `isLoading`/`styleGuideOptions` change. But silent eslint disables without rationale are a maintenance risk.
**Fix:** Add a brief comment above the eslint-disable:

```typescript
// changeValue and getCurrentValue are intentionally excluded —
// they change identity on every render but the effect only needs
// to fire once when style guide options finish loading.
// eslint-disable-next-line react-hooks/exhaustive-deps
```

### 4. Tests duplicate logic instead of testing the component

**[File: apps/creative-portal/components/organisms/NewOrderForm/partials/BriefStep/index.test.ts]**
**Function/Class:** getAutoPopulatedStyleGuide, getIsStyleGuideDisabled
**Severity:** medium
**Problem:** The tests re-implement the auto-population and disable logic as standalone functions (`getAutoPopulatedStyleGuide`, `getIsStyleGuideDisabled`) and then test those copies — not the actual `useEffect` or JSX in the component. If the component logic drifts from the test's reimplementation, the tests will still pass while the component is broken.
**Impact:** False confidence — tests verify the algorithm in isolation but not the integration (does the `useEffect` fire at the right time? does the `isDisabled` prop actually reach the `Select`?). A refactor that changes the component's condition would not be caught.
**Fix:** This is an acceptable tradeoff for a component that's difficult to render in tests (heavy Formik/context dependencies). However, consider adding a comment at the top of each test block noting that it mirrors component logic and must be kept in sync. Alternatively, consider a shallow render test that verifies the Select receives `isDisabled={true}` when options have length 1.

---

## Tests

- ✅ Unit tests cover the auto-population decision logic (single option, multiple options, zero options, undefined, loading, already-set value)
- ✅ Unit tests cover the disable logic (single option disabled, multiple options enabled, loading disabled)
- ✅ Field mapping test verifies `styleName` → `name` and `styleGuideUrl` → `url`
- ⚠️ Tests are logic duplicates, not component integration tests — drift risk exists
- ❌ No test for the "Multiple" sentinel case interacting with auto-population (what happens if `getCurrentValue("styleGuide")` returns `"Multiple"` — the test uses the `MULTIPLE` constant but doesn't verify the component handles this correctly end-to-end)

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Logic is correct, but disabled field contradicts Jira description |
| Regression risk | ✅ Low — changes are scoped to style guide field in BriefStep |
| Tests | ⚠️ Good coverage of logic, but tests are disconnected from component |
| Code quality | ✅ Clean, follows existing patterns |
| Mergeable state | ⚠️ Needs Jira clarification |

---

## Recommendation

**Approve with suggestions**

1. **Resolve the editable vs disabled conflict** — confirm with the team whether the field should be disabled when there's one style guide. Update the Jira acceptance criteria to match the agreed decision so QA tests the right behavior.
2. **Add a comment above the eslint-disable** explaining why `changeValue` and `getCurrentValue` are excluded from the useEffect deps.
3. **Consider** noting in the test file that the standalone functions mirror component logic and must stay in sync with the actual useEffect/JSX.
