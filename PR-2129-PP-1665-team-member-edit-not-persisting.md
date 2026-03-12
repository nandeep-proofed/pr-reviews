# PR Review: PP-1665: Fix team member edit not persisting from search popup

**PR:** [https://github.com/Proofed/B2BWebserver/pull/2129](https://github.com/Proofed/B2BWebserver/pull/2129)
**Jira:** [https://proofed.atlassian.net/browse/PP-1665](https://proofed.atlassian.net/browse/PP-1665)
**Status:** Waiting for Deployment

---

## Jira Requirements vs Implementation


| Jira Requirement                                               | PR Implementation                                                                                                                                                                                            | Status     |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- |
| Edit Team Member from search popup should persist role changes | PR removes duplicate TeamMemberModal from MainLayout (Header already renders it). However, the hooks.ts fix described in the PR description (moving `onClose` after `invalidateQueries`) is NOT in the diff. | ⚠️ Partial |
| Re-opening team member via search should show updated role     | Removing the duplicate modal may help, but the core hooks.ts ordering issue remains unfixed in this PR                                                                                                       | ⚠️ Partial |


---

## Architecture Analysis

The Jira ticket describes edits made via the search popup not persisting. The PR description claims two fixes:

1. Moving `onClose()` after `await Promise.all(invalidateQueries)` in hooks.ts
2. Wrapping the handler in `try/catch`

**Neither of these changes exists in the actual diff.** The PR only has 4 changed files: `TASKS_COMPLETED.md`, `MainLayout/index.tsx`, `hooks.test.ts` (new), and `vitest.config.ts`.

What the PR actually does:

- **Removes duplicate TeamMemberModal from MainLayout** — The Header component (`Header/index.tsx:357`) already renders `<TeamMemberModal>` using the same `editedUserId` from `TeamMemberModalContext`. MainLayout was rendering a second instance, meaning two modals opened simultaneously. Removing the duplicate is a valid fix.
- **Adds test file** — Tests behavior (invalidateQueries before onClose, try/catch error handling) that doesn't exist in the actual hooks.ts code.
- **Adds vitest path aliases** — Enables bare-path imports in creative-portal tests.

The current hooks.ts on develop (line 198) still calls `onClose()` BEFORE `invalidateQueries`, which is the root cause described in the PR. This fix is missing from the diff.

---

## Issues Found

### 1. Core fix is missing from the diff — hooks.ts unchanged

**[File: apps/creative-portal/components/organisms/modals/TeamMemberModal/hooks.ts]**
**Function/Class:** onEditTeamMember
**Severity:** high
**Problem:** The PR description says `onClose()` was moved after `await Promise.all(invalidateQueries)` and the handler was wrapped in `try/catch`. Neither change appears in the diff. The current hooks.ts (line 198) still calls `onClose()` before query invalidation. The TASKS_COMPLETED.md references "Stabilize initialValues" and "Reverted setQueryData approach" — suggesting hooks.ts changes were made and then reverted, leaving the PR in an inconsistent state.
**Impact:** The primary fix for PP-1665 is not present. While removing the duplicate modal from MainLayout may partially help (by eliminating conflicting query observers), the core issue — `onClose()` unmounting the modal before `invalidateQueries` completes — remains in the Header's rendering of TeamMemberModal.
**Fix:** Reapply the hooks.ts changes: move `onClose()` after `await Promise.all(invalidateQueries)` and wrap in `try/catch` as described in the PR description.

### 2. Tests will fail — they test non-existent behavior

**[File: apps/creative-portal/components/organisms/modals/TeamMemberModal/tests/hooks.test.ts]**
**Function/Class:** useTeamMemberModal - onEditTeamMember
**Severity:** high
**Problem:** The test file tests that: (a) `invalidateQueries` is called before `onClose`, (b) errors in mutations prevent `onClose` and `showToast` from firing. But the actual hooks.ts code calls `onClose()` before `invalidateQueries` (line 198), and has no `try/catch` around the mutation calls in `onEditTeamMember`. These tests will fail when run against the actual code.
**Impact:** CI will fail. Tests assert behavior that doesn't exist in the codebase.
**Fix:** Either reapply the hooks.ts changes so the tests pass, or update the tests to match the actual code behavior.

### 3. Removing TeamMemberModal from MainLayout — verify no side effects

**[File: apps/creative-portal/components/layouts/MainLayout/index.tsx]**
**Function/Class:** MainLayout
**Severity:** medium
**Problem:** The PR removes the `TeamMemberModal` rendering from MainLayout, relying solely on the Header's rendering. The Header renders the modal inside `<Suspense>` with `isHydrated` guard. If the Header's modal rendering has different lifecycle behavior (e.g., SSR vs CSR, Suspense boundaries), the edit flow could behave differently.
**Impact:** The duplicate was clearly a bug (two modals opening simultaneously), so removing it is correct. But since the Header's `onClose` is `() => onTeamMemberClick(undefined)` while MainLayout used `closeTeamMemberModal`, verify these behave identically.
**Fix:** Confirm that `onTeamMemberClick(undefined)` (Header) and `closeTeamMemberModal` (MainLayout) both result in the same state change. Manual testing is critical for this change.

### 4. TASKS_COMPLETED.md references non-existent changes

**[File: TASKS_COMPLETED.md]**
**Function/Class:** N/A
**Severity:** low
**Problem:** The TASKS_COMPLETED.md entry for PP-1665 describes "Extracted `initialValues` into a separate `useMemo`" and "Reverted `setQueryData` approach — restored fire-and-forget `invalidateQueries`". Neither change exists in the diff. This suggests work was done and then reverted.
**Impact:** Misleading documentation. Future developers reading TASKS_COMPLETED.md will think these changes were made.
**Fix:** Update or remove the TASKS_COMPLETED.md entry to reflect what's actually in the PR.

---

## Tests

- ❌ **Test file added but tests will fail** — the 6 tests assert behavior (invalidateQueries before onClose, try/catch error handling) that doesn't exist in the actual hooks.ts code
- ✅ Test infrastructure improvement — vitest path aliases added for creative-portal, enabling tests with bare-path imports
- ⚠️ Manual testing checklist included but not yet confirmed

---

## Summary


| Aspect          | Status                                                                 |
| --------------- | ---------------------------------------------------------------------- |
| Correctness     | ❌ Core hooks.ts fix is missing from the diff                           |
| Regression risk | ⚠️ Medium — removing duplicate modal is correct but needs verification |
| Tests           | ❌ Tests will fail against actual code                                  |
| Code quality    | ❌ PR in inconsistent state (description, tests, docs don't match diff) |
| Mergeable state | ❌ Dirty (merge conflicts)                                              |


---

## Recommendation

**Request changes**

1. **Reapply the hooks.ts fix** — the core fix (moving `onClose()` after query invalidation, adding try/catch) is missing from the diff. This is the primary fix for PP-1665.
2. **Verify tests pass** — the test file currently tests behavior that doesn't exist. After reapplying hooks.ts changes, confirm all 6 tests pass.
3. **Update TASKS_COMPLETED.md** — align the entry with the actual changes in the PR.
4. **Resolve merge conflicts** — mergeable state is dirty.
5. **Manual testing** — verify the search popup edit flow works end-to-end with only the Header rendering the modal.

