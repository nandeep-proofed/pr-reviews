# PR Review: PP-1857: Add Opt Out of AI Processing toggle to project Order Template

**PR:** https://github.com/Proofed/B2BWebserver/pull/2284
**Jira:** https://proofed.atlassian.net/browse/PP-1857
**Status:** Open (clean, not merged)

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Surface `disableAi` backend field (PP-1856) as a toggle in the creative-portal Admin Area | `FormikCheckboxSingle` added to `settings/general/index.tsx` | ✅ Addressed |
| Default value `false` when field absent | `orgGroup?.disableAi ?? false` in `formProps.initialValues` | ✅ Addressed |
| Full PUT only (not partial update) | Uses existing `updateOrganizationGroup` spread, no patch attempted | ✅ Addressed |
| Cache invalidated after save so persisted state refetches | `queryClient.invalidateQueries([ORGANIZATION_GROUPS_BY_ID_QUERY_KEY, projectId])` in `onSuccess` | ✅ Addressed |
| Copy-existing-partner-settings flow inherits `disableAi` | `disableAi: orgGroup?.disableAi` added to `NewProjectModal/hooks.tsx` copy payload | ✅ Addressed |
| Field must not appear in Customer Portal | No customer-portal files changed | ✅ Addressed |
| Unit tests for new surfaces | 7 new tests across 2 test files | ✅ Addressed |

---

## Architecture Analysis

The implementation correctly uses the existing `useSettingsPage` → `handleSubmit` → `updateOrganizationGroup` PUT flow. `disableAi` is included in `formProps.initialValues` and flows through Formik, then gets spread into the full PUT body alongside the existing `orgGroup` fields. No new API layer is needed.

The read-only summary display (`content.tsx`) shows the current value via a new `AiProcessingDescription` partial, following the same pattern as `GroupingDescription` and `AttachmentDescription` on the Order Template tab.

The toggle was moved from the Order Template tab (early commits) to the General tab (latest Figma) and uses `FormikCheckboxSingle` rather than `Switch`. The final placement and control type is correct per the latest design.

---

## Issues Found

### 1. PR title and description reference the wrong tab

**[File: PR metadata (not a code file)]**
**Function/Class:** N/A
**Severity:** medium
**Problem:** The PR title still reads "Add Opt Out of AI Processing toggle to project **Order Template**", and the "Areas of Change" section in the PR description lists `order-template/hooks.ts` and `order-template/index.tsx` as modified files. The actual implementation lives entirely in `settings/general/`. The description also mentions the toggle "is rendered as a Switch (matching the existing Allow attachments / Allow grouping toggles)" — but the final implementation uses `FormikCheckboxSingle`, not `Switch`.
**Impact:** Misleading for changelog entries, release notes, and future git blame readers.
**Fix:** Update the PR title to "Add Opt Out of AI Processing toggle to project General settings" and update the "Areas of Change" section. Remove the Switch/Order Template comparison sentence.

---

### 2. `AiProcessingDescription` missing `FC<Props>` annotation and `types.ts`

**[File: apps/creative-portal/components/pages/partners/[partnerId]/projects/[projectId]/settings/general/partials/AiProcessingDescription/index.tsx]**
**Function/Class:** AiProcessingDescription
**Severity:** low
**Problem:** The project conventions require every component to (a) annotate with `FC<Props>` and (b) include a `types.ts` file alongside `index.tsx`. `AiProcessingDescription` does neither — its prop type is inlined and it has no explicit `FC` annotation. Note that the sibling `GroupingDescription` has the same gap (pre-existing), so this is a pattern already present in the codebase.
**Impact:** Minor — no runtime effect, but inconsistency with the project component structure convention.
**Fix:**

```tsx
// types.ts
import { OrganizationGroup } from "api/organizationGroup/types";

export interface AiProcessingDescriptionProps {
  orgGroup?: OrganizationGroup;
}
```

```tsx
// index.tsx
import { FC } from "react";

import { AiProcessingDescriptionProps } from "./types";

const AiProcessingDescription: FC<AiProcessingDescriptionProps> = ({
  orgGroup
}) => (orgGroup?.disableAi ? <>Opted Out</> : <>Opted In</>);

export default AiProcessingDescription;
```

---

### 3. `GENERAL_INFO_FORM_DATA_SCHEMA` not updated for `disableAi`

**[File: apps/creative-portal/components/pages/partners/[partnerId]/projects/[projectId]/settings/general/consts.ts]**
**Function/Class:** GENERAL_INFO_FORM_DATA_SCHEMA
**Severity:** low
**Problem:** The Yup validation schema for the General form does not include a `disableAi` field. The field is included in `formProps.initialValues` and submitted to the API, but is invisible to schema validation.
**Impact:** Practically harmless for a boolean checkbox (it can't have an invalid value), but leaves the schema inconsistent with the actual form shape. TypeScript inference from the schema will not include `disableAi`.
**Fix:**

```ts
export const GENERAL_INFO_FORM_DATA_SCHEMA = Yup.object().shape({
  // ...existing fields...
  disableAi: Yup.boolean()
});
```

---

### 4. AI Processing summary hidden when project has no address

**[File: apps/creative-portal/components/pages/partners/[partnerId]/projects/[projectId]/settings/general/content.tsx]**
**Function/Class:** GeneralProjectInformationContent
**Severity:** low
**Problem:** The entire description list (including the new "AI Processing" row) is gated on `orgGroup?.address` being truthy. When a project has no address saved, the component renders an empty-state message instead, hiding the AI Processing status entirely.

```tsx
return !isLoadingOrgGroup && !orgGroup?.address ? (
  // empty state — AI Processing not visible here
) : (
  // full list including AI Processing
);
```

**Impact:** A project created via "Copy existing partner settings" with `disableAi: true` but no address will show no indication of opted-out status in the General summary until an address is also saved. Low-risk since admins can always navigate to the edit form to check/change the value.
**Fix:** Move the AI Processing `DescriptionList` block outside the address conditional so it always renders when data is loaded. This mirrors the intent — `disableAi` is not address-dependent.

---

## Tests

- ✅ `general/__tests__/hooks.test.ts` — 4 new tests covering `useEditGeneralInfo`: default `false` when field absent, reflects `true` from loaded orgGroup, invalidates the correct cache key on success, shows success toast and redirects.
- ✅ `AiProcessingDescription.test.tsx` — 4 tests covering `disableAi: true`, `false`, `undefined`, and `orgGroup: undefined`. Consistent with the co-location pattern used by `GroupingDescription.test.tsx`.
- ✅ `NewProjectModal/__tests__/hooks.test.ts` — 2 new tests covering copy-flow: propagates `disableAi: true` and correctly omits the field when not copying.
- ⚠️ No test for the `content.tsx` integration — the conditional that hides the AI Processing row when no address is present is untested. Low-risk but the edge case in Issue #4 would be caught by a test here.
- ✅ PR states 903/903 creative-portal tests passing.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ `disableAi` correctly flows through Formik → PUT → cache invalidation |
| Regression risk | ✅ Low — changes are additive; existing fields untouched |
| Tests | ⚠️ Good coverage on new code; `content.tsx` integration gap is minor |
| Code quality | ⚠️ Minor convention gaps (`FC<Props>`, `types.ts`, Yup schema) |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions**

1. Update the PR title and "Areas of Change" description to reflect the General tab placement and `FormikCheckboxSingle` control — avoids confusion in the changelog.
2. Add `FC<Props>` annotation and `types.ts` to `AiProcessingDescription` to conform with project conventions.
3. Consider adding `Yup.boolean()` to `GENERAL_INFO_FORM_DATA_SCHEMA` for completeness.
4. Consider moving the AI Processing `DescriptionList` outside the `orgGroup.address` gate in `content.tsx` so opted-out status is visible even before general info is filled in.
