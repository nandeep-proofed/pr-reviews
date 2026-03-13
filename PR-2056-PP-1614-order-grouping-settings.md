# PR Review: feature/PP-1614: Add order grouping functionality to organization settings

**PR:** [https://github.com/Proofed/B2BWebserver/pull/2056](https://github.com/Proofed/B2BWebserver/pull/2056)
**Jira:** [https://proofed.atlassian.net/browse/PP-1614](https://proofed.atlassian.net/browse/PP-1614)
**Status:** Code Review

---

## Jira Requirements vs Implementation


| Jira Requirement                                                              | PR Implementation                                                                                    | Status                     |
| ----------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | -------------------------- |
| Settings tile shows allowOrderGrouping state (Allowed/Not Allowed with icons) | GroupingDescription component shows icon + text based on `orgGroup?.allowOrderGrouping`              | ✅ Addressed                |
| Edit Template Settings page: toggle for enabling/disabling order grouping     | Switch component added with `allowOrderGrouping` field in Formik form                                | ✅ Addressed                |
| Save triggers PATCH to update organization group                              | handleSubmit spreads all form values into orgGroup update — allowOrderGrouping is included           | ✅ Addressed                |
| Cancel discards changes and reverts toggle                                    | Formik `enableReinitialize` + Cancel navigates back to settings page                                 | ✅ Addressed                |
| Copy setting when creating new project from existing                          | `createOrganizationGroup.ts` already handles `allowOrderGrouping` with default `false`               | ✅ Addressed (pre-existing) |
| Checkbox replaced with Switch for UI consistency                              | Checkbox replaced with Switch for `allowSupportDocuments`, new Switch added for `allowOrderGrouping` | ✅ Addressed                |


**Scope creep:** The PR also replaces the existing `allowSupportDocuments` Checkbox with a Switch component and adds `StyledSwitchLabel`. This is a reasonable UI consistency change but goes slightly beyond the ticket scope.

---

## Architecture Analysis

The approach follows the existing pattern established by `allowSupportDocuments` — adding a boolean field to the `OrganizationGroup` type, a Switch toggle in the edit form, and a read-only description component in the content view. The form submission flows through the existing `useSettingsPage` hook which spreads all form data into the API update, so no additional submit logic was needed. The API endpoints (`putOrganizationGroup`, `createOrganizationGroup`) already handle `allowOrderGrouping` with a default `false` value, indicating backend work was done prior to this PR.

---

## Issues Found

### 1. Suspicious margin logic in GroupingDescription

**[File: apps/creative-portal/components/pages/partners/[partnerId]/projects/[projectId]/settings/order-template/partials/GroupingDescription/index.tsx]**
**Function/Class:** GroupingDescription
**Severity:** low
**Problem:** `m: isAllowed && 0` evaluates to `false` when `isAllowed` is falsy (which is not a valid theme-ui value) and `0` when truthy. Setting margin to `0` only when allowed is confusing — it's unclear what margin problem this solves, and `false` as a style value may produce unexpected behavior.
**Impact:** Possible inconsistent spacing between the Allowed/Not Allowed states. This is copied from AttachmentDescription which has the same pattern, so it's a pre-existing issue.
**Fix:** If this is intentional to match AttachmentDescription, it's acceptable as-is. If the intent is to always have `m: 0`, simplify to just `m: 0`. Otherwise, clarify the intent with a comment.

### 2. GroupingDescription always renders (unlike conditional fields above it)

**[File: apps/creative-portal/components/pages/partners/[partnerId]/projects/[projectId]/settings/order-template/content.tsx]**
**Function/Class:** OrderTemplateInfoContent
**Severity:** low
**Problem:** Content Type, Industry, and Language are conditionally rendered (only when they have values), but Grouping is always rendered regardless. This matches the behavior of Attachments, so it's consistent with the existing pattern for boolean toggles vs. optional text fields.
**Impact:** None — this is correct behavior. The Grouping row always shows either "Allowed" or "Not Allowed", which is the expected UX.
**Fix:** No fix needed — this is the correct approach.

### 3. No unit tests included

**[File: N/A]**
**Function/Class:** N/A
**Severity:** medium
**Problem:** The PR adds a new GroupingDescription component and modifies form behavior (new field, Checkbox-to-Switch change) but includes no unit tests.
**Impact:** Violates the project requirement that every PR must include tests for new code. The GroupingDescription component and the form's new `allowOrderGrouping` field are untested.
**Fix:** Add tests for:

- GroupingDescription renders "Allowed" with check icon when `allowOrderGrouping` is true
- GroupingDescription renders "Not Allowed" with cancel icon when `allowOrderGrouping` is false/undefined
- OrderTemplateInfoPage form includes the `allowOrderGrouping` switch
- The switch correctly calls `setFieldValue` on change

---

## Tests

- ❌ No unit tests for GroupingDescription component
- ❌ No unit tests for the new allowOrderGrouping form field
- ❌ No tests for the Checkbox-to-Switch migration on allowSupportDocuments
- ✅ Existing tests should not be broken (type additions are additive/optional)

---

## Summary


| Aspect          | Status                                                         |
| --------------- | -------------------------------------------------------------- |
| Correctness     | ✅ Implementation matches Jira requirements                     |
| Regression risk | ✅ Low — additive changes, existing Checkbox→Switch is cosmetic |
| Tests           | ❌ Missing — no tests included                                  |
| Code quality    | ✅ Follows existing patterns (mirrors AttachmentDescription)    |
| Mergeable state | ✅ Clean                                                        |


---

## Recommendation

**Approve with suggestions**

1. **Add unit tests** for GroupingDescription and the new form field — this is a project requirement.
2. Consider clarifying the `m: isAllowed && 0` pattern in GroupingDescription (low priority, pre-existing pattern from AttachmentDescription).

