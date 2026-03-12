# PR Review: fix/PP-1717: prevent browser being saved when TrustBrowserModal is dismissed

**PR:** [https://github.com/Proofed/B2BWebserver/pull/2127](https://github.com/Proofed/B2BWebserver/pull/2127)
**Jira:** [https://proofed.atlassian.net/browse/PP-1717](https://proofed.atlassian.net/browse/PP-1717)
**Status:** Code Review

---

## Jira Requirements vs Implementation


| Jira Requirement                                                        | PR Implementation                                                                                                             | Status      |
| ----------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ----------- |
| Closing modal via X should NOT save/trust browser                       | Added `onDismiss` handler that only calls `setIsTrustBrowserModalOpen(false)` — no API call, no localStorage, no trustBrowser | ✅ Addressed |
| Browser saved only on explicit accept CTA                               | `onSubmit` still handles "Proceed" button click, calls API + trustBrowser when "Save browser" selected                        | ✅ Addressed |
| Dismissing modal discards the action                                    | `onClose` prop changed from `onSubmit` to `onDismiss`                                                                         | ✅ Addressed |
| (Secondary fix) "Don't save" + Proceed should prevent modal reappearing | `localStorage.setItem` moved before the `shouldSaveBrowser` check so both choices mark the modal as seen                      | ✅ Addressed |


No scope creep — all changes directly relate to the bug.

---

## Architecture Analysis

The fix is clean and minimal. The root cause was that `ModalWithConfirmation`'s `onClose` (triggered by the X button) was bound to the same `onSubmit` handler as "Proceed", and since `shouldSaveBrowser` defaults to `"true"`, dismissing the modal was equivalent to accepting it.

The fix introduces a separate `onDismiss` handler for the X button that only closes the modal state. It also correctly repositions the `localStorage.setItem` call so that both "Save browser" and "Don't save" + Proceed prevent the modal from reappearing on subsequent logins — this was a secondary bug where choosing "Don't save" would cause the modal to reappear because `localStorage.setItem` was after the early return.

Both creative-portal and customer-portal consume the shared `TrustBrowserModal` as thin wrappers, so the fix propagates to both apps automatically.

---

## Issues Found

### 1. localStorage set before API call — potential inconsistency on API failure

**[File: packages/shared/components/molecules/TrustBrowserModal/index.tsx]**
**Function/Class:** onSubmit
**Severity:** low
**Problem:** `localStorage.setItem` now runs before `await axios.post(...)`. If the API call fails (network error, server error), localStorage is set but the backend doesn't know the user has seen the modal. On next login, the backend may still return `userSeenTrustBrowserModal: false`, but localStorage prevents the modal from showing.
**Impact:** Minimal — the modal was already designed with localStorage as the primary gate (checked in the `useEffect`). The backend flag `userSeenTrustBrowserModal` is a secondary check. The user experience is arguably better this way (no repeated modal on transient API failures). This is an observation, not a required change.
**Fix:** If strict backend consistency is desired, wrap in try/catch and only set localStorage on success. However, the current approach is pragmatically fine — the modal reappearing on API failure would be more annoying than the minor inconsistency.

### 2. Pre-existing: localStorage key uses potentially undefined user ID

**[File: packages/shared/components/molecules/TrustBrowserModal/index.tsx]**
**Function/Class:** onSubmit, onDismiss
**Severity:** low
**Problem:** `user?.id` could theoretically be undefined, producing key `hasUserSeenTrustBrowserModal_undefined`. This is pre-existing and not introduced by this PR.
**Impact:** Negligible — the component already guards against `!user?.mfaVerified` in the useEffect, so the modal won't open if user is undefined.
**Fix:** No action required for this PR.

---

## Tests

- ✅ 6 new unit tests added in `TrustBrowserModal.test.tsx` (new file)
- ✅ Dismiss via X does not call trustBrowser or set localStorage
- ✅ "Save browser" + Proceed calls trustBrowser and sets localStorage
- ✅ "Don't save" + Proceed does NOT call trustBrowser but DOES set localStorage
- ✅ localStorage set on submit regardless of save choice
- ✅ Modal hidden when user has already seen it
- ✅ Modal hidden when MFA not verified
- ✅ Mocks are well-structured and test the actual component behavior
- ✅ `vitest.config.ts` updated with `esbuild: { jsx: "automatic" }` to support JSX tests in shared package
- ⚠️ No test for error handling when `axios.post` fails (acceptable given low severity)

---

## Summary


| Aspect          | Status                                                   |
| --------------- | -------------------------------------------------------- |
| Correctness     | ✅ Fixes root cause correctly                             |
| Regression risk | ✅ Low — onDismiss is additive, localStorage move is safe |
| Tests           | ✅ Comprehensive coverage of all key scenarios            |
| Code quality    | ✅ Clean, minimal, follows existing patterns              |
| Mergeable state | ❌ Dirty (merge conflicts with develop)                   |


---

## Recommendation

**Approve with suggestions**

1. Resolve merge conflicts with `develop` branch to get mergeable state to clean.
2. (Optional) Consider whether localStorage-before-API is the desired ordering for error scenarios — current approach is pragmatically fine but worth a conscious decision.

