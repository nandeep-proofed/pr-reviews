# PR Review: fix/PP-1647: Order Creation Notes field loses content while typing

**PR:** [https://github.com/Proofed/B2BWebserver/pull/2125](https://github.com/Proofed/B2BWebserver/pull/2125)
**Jira:** [https://proofed.atlassian.net/browse/PP-1647](https://proofed.atlassian.net/browse/PP-1647)
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Text entered in Order Creation Notes should remain stable while typing | Replaced `queryClient.invalidateQueries()` with `queryClient.setQueryData()` to prevent refetch cycle that overwrites form via `enableReinitialize` | ✅ Addressed |
| User-entered content should not be removed or altered unless explicitly edited | Same fix — eliminates the save→refetch→reinitialize cycle that reverted user input | ✅ Addressed |
| Right-side tiles should not re-render during text entry | `setQueryData` with current data avoids triggering a refetch, which was causing dependent components to re-render | ✅ Addressed |

**Scope beyond Jira:** Lodash imports changed from barrel to individual (tree-shaking improvement) — minor, acceptable. `FormDirtyTracker` removed — related to the fix but has side effects (see Issues).

---

## Architecture Analysis

The root cause was a destructive feedback loop:

1. User types → `<Form onChange={formik.submitForm}>` fires on every keystroke
2. Debounced submit (500ms) sends data to server via `usePutOrganizationGroupMutation`
3. On success, `invalidateQueries` triggers a refetch of `ORGANIZATION_GROUPS_BY_ID_QUERY_KEY`
4. Refetch returns server state (which may not include the latest keystrokes)
5. `enableReinitialize` on Formik reinitializes the form with stale server data → content lost

The fix replaces step 3: instead of invalidating (which triggers refetch), it calls `setQueryData(key, orgGroup)` with the current in-memory `orgGroup`. Since this value is the same as what's already in the cache, React Query detects no change and skips re-renders, breaking the destructive cycle.

This is a pragmatic fix that addresses the symptom effectively. The form still submits on every keystroke (debounced), which is chatty but no longer destructive.

---

## Issues Found

### 1. PR description does not match actual diff

**[File: N/A — PR metadata]**
**Function/Class:** N/A
**Severity:** medium
**Problem:** The PR description states: "Replaced `onChange` auto-submit with `onBlur`-based save" and "Removed `debounce` wrapper (unnecessary with onBlur)". Neither change is in the diff. The form still uses `<Form onChange={formik.submitForm}>` (not onBlur), and `debounce` is still imported and used. The description also claims "5 regression tests" in `hooks.test.ts`, but no test file appears in the 2 changed files.
**Impact:** Misleading for reviewers. If someone approves based on the description, they're approving changes that don't exist. The tests mentioned as passing were never committed to this PR.
**Fix:** Update the PR description to accurately reflect the actual changes: (1) replaced `invalidateQueries` with `setQueryData`, (2) removed `FormDirtyTracker`, (3) changed lodash imports. Remove the claims about onBlur, debounce removal, and test files, or add the test file to the PR.

### 2. `setQueryData` writes stale data to the cache

**[File: apps/creative-portal/components/pages/partners/[partnerId]/projects/[projectId]/settings/partials/Notes/hooks.ts]**
**Function/Class:** useNotesSetting → onSuccess
**Severity:** medium
**Problem:** `queryClient.setQueryData(key, orgGroup)` uses the `orgGroup` from the closure, which is the value from the last render — NOT the data that was just submitted. After `handleSubmit` sends `{ ...orgGroup, ...newData }` to the server, the success callback writes back the OLD `orgGroup` (without the new notes) to the query cache. While this is effectively a no-op (the cache already has this value), it means the cache never gets updated with the freshly saved data. Other consumers of `ORGANIZATION_GROUPS_BY_ID_QUERY_KEY` won't see the updated notes until a natural refetch (e.g., window focus, component mount).
**Impact:** Low in practice for the notes field specifically, but semantically incorrect. If another part of the settings page reads `orderCreationNotes` from this query, it'll show stale data.
**Fix:** Either (a) merge the submitted values into the cache update, or (b) simply remove the `onSuccess` entirely since the no-op setQueryData doesn't add value:

Option A — optimistic cache update with submitted data:
```typescript
onSuccess: () => {
  if (orgGroup) {
    queryClient.setQueryData(
      [
        QUERY_KEYS.ORGANIZATION_GROUPS_BY_ID_QUERY_KEY,
        `${orgGroup.id}`
      ],
      (oldData: typeof orgGroup) => ({
        ...oldData,
        ...formValuesRef.current
      })
    );
  }
}
```

Option B — remove onSuccess entirely (simplest):
```typescript
} = useSettingsPage({});
```

### 3. Dead code — unsaved changes tracking left behind after FormDirtyTracker removal

**[File: apps/creative-portal/components/pages/partners/[partnerId]/projects/[projectId]/settings/partials/Notes/index.tsx]**
**Function/Class:** Notes
**Severity:** low
**Problem:** `FormDirtyTracker` was removed, but the component still has `useState(false)` for `hasUnsavedChanges`, `useUnsavedChangesRegistry`, `useUnsavedChangesResetRegistry`, and the associated `useEffect` blocks. Since `hasUnsavedChanges` is never set to `true`, this is all dead code.
**Impact:** No functional impact, but misleading — future developers may think unsaved changes tracking is active when it isn't. Also, users navigating away mid-type (within the 500ms debounce window) won't get the "unsaved changes" warning.
**Fix:** Either remove the dead code entirely (if unsaved changes tracking isn't needed for auto-save forms), or keep `FormDirtyTracker` and find a different solution for the re-render issue.

### 4. Form still submits on every keystroke

**[File: apps/creative-portal/components/pages/partners/[partnerId]/projects/[projectId]/settings/partials/Notes/index.tsx]**
**Function/Class:** Notes
**Severity:** low
**Problem:** `<Form onChange={formik.submitForm}>` fires a debounced network request on every keystroke. While the fix prevents this from being destructive (no more refetch), it's still sending a PUT request to update the org group ~500ms after each pause in typing. For a notes field, this is unnecessarily chatty.
**Impact:** Extra network traffic and server load. Not a blocker, but worth noting as technical debt.
**Fix:** Consider changing to `onBlur`-based save (as the PR description originally stated) in a follow-up, which would save only when the user clicks away from the field.

---

## Tests

- ❌ **No test files in the PR** — the description claims "5 regression tests" in `hooks.test.ts` but the file is not in the diff. Only 2 files are changed (hooks.ts and index.tsx), with 9 additions and 10 deletions total.
- ❌ **Project requirement not met** — CLAUDE.md states "Every PR must include tests for new code"
- ⚠️ Manual testing checklist included in PR description but not yet confirmed

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Fixes the symptom (refetch cycle) but doesn't address root cause (onChange auto-submit) |
| Regression risk | ⚠️ Medium — stale cache data, dead unsaved-changes code |
| Tests | ❌ No tests included despite PR description claiming them |
| Code quality | ⚠️ Dead code left behind, description/diff mismatch |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Request changes**
1. **Update PR description** to match the actual diff — remove claims about onBlur, debounce removal, and test files.
2. **Add tests** — at minimum, tests for the `useNotesSetting` hook verifying that `setQueryData` is called instead of `invalidateQueries` on success.
3. **Clean up dead code** — either remove the unsaved changes tracking (`hasUnsavedChanges` state, registry hooks, effects) or keep `FormDirtyTracker`.
4. **Fix or remove the `setQueryData` call** — either do a proper optimistic update with submitted values, or remove `onSuccess` entirely since the current call is a no-op.
