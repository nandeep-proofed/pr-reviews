# PR Review: fix/PP-1713: Show BackupCodesModal after redirect

**PR:** https://github.com/Proofed/B2BWebserver/pull/2147
**Jira:** https://proofed.atlassian.net/browse/PP-1713
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Recovery codes modal should appear with proper background context (destination page) | Modal now shows via `BackupCodesModalTrigger` in `_app.tsx` ModalQueue after redirect completes | ✅ Addressed |
| No confusing delay/redirect after closing modal | Redirect happens immediately; modal shows on the destination page via sessionStorage handoff | ✅ Addressed |
| Applies to both creative and customer portals | Both portals updated with identical ModalQueue changes | ✅ Addressed |

---

## Architecture Analysis

The PR follows a clean, well-established pattern already used by `TrustBrowserModal`:

1. **Store-and-redirect:** During TOTP enrollment (`useTotpEnrollment`), backup codes are saved to `sessionStorage` and the redirect fires immediately — no more waiting for the user to close the modal before navigating.
2. **Trigger-on-load:** A new `BackupCodesModalTrigger` component in `_app.tsx`'s `ModalQueue` reads sessionStorage on mount, displays the modal if codes exist, and cleans up on close.
3. **Modal ordering:** `BackupCodesModal` → `TrustBrowserModal` → `NewTimezoneModal`, each gated by the previous modal's `onOpen(false)` callback.

The approach correctly decouples the modal display from the enrollment flow, eliminating the original UX issue where the modal appeared over the `/authenticator-app` page instead of the destination.

---

## Issues Found

### 1. Missing `onOpen(false)` when no codes exist in sessionStorage

**[File: packages/shared/components/molecules/BackupCodesModalTrigger/index.tsx]**
**Function/Class:** BackupCodesModalTrigger (useEffect)
**Severity:** medium
**Problem:** When `sessionStorage` has no backup codes (the common case for returning users), `onOpen` is never called. The `isBackupCodesModalOpen` state in the parent `ModalQueue` stays `null`, which means `TrustBrowserModal` and `NewTimezoneModal` never render because the condition `isBackupCodesModalOpen === false` is never met.
**Impact:** On pages where there are no backup codes to show (most page loads), `TrustBrowserModal` and `NewTimezoneModal` will be blocked from appearing. This is a **regression** compared to the previous behavior where these modals rendered unconditionally.
**Fix:** Call `onOpen?.(false)` when no stored codes are found:

```typescript
useEffect(() => {
  const stored = sessionStorage.getItem(BACKUP_CODES_SESSION_KEY);

  if (stored) {
    try {
      const parsed = JSON.parse(stored) as string[];

      setCodes(parsed);
      setIsOpen(true);
      onOpen?.(true);
    } catch {
      sessionStorage.removeItem(BACKUP_CODES_SESSION_KEY);
      onOpen?.(false);
    }
  } else {
    onOpen?.(false);
  }
}, []); // eslint-disable-line react-hooks/exhaustive-deps
```

### 2. Tests only cover sessionStorage operations, not the component

**[File: packages/shared/components/molecules/BackupCodesModalTrigger/BackupCodesModalTrigger.test.ts]**
**Function/Class:** BackupCodesModalTrigger test suite
**Severity:** medium
**Problem:** The 6 tests only test `sessionStorage.getItem/setItem/removeItem` directly — they don't render or test the `BackupCodesModalTrigger` component at all. The test file name implies component tests, but no React rendering, effect triggering, or `onOpen` callback behavior is verified.
**Impact:** The critical regression from Issue #1 (missing `onOpen(false)`) would not be caught by these tests. The modal queue ordering logic is completely untested.
**Fix:** Add component-level tests using `@testing-library/react`:

```typescript
import { render } from "@testing-library/react";
import BackupCodesModalTrigger from ".";

it("calls onOpen(false) when no codes in sessionStorage", () => {
  const onOpen = vi.fn();
  render(<BackupCodesModalTrigger onOpen={onOpen} />);
  expect(onOpen).toHaveBeenCalledWith(false);
});

it("calls onOpen(true) when codes exist in sessionStorage", () => {
  sessionStorage.setItem(
    BACKUP_CODES_SESSION_KEY,
    JSON.stringify(["CODE1"])
  );
  const onOpen = vi.fn();
  render(<BackupCodesModalTrigger onOpen={onOpen} />);
  expect(onOpen).toHaveBeenCalledWith(true);
});
```

### 3. `setIsRedirecting(true)` called before `onComplete()` in creative portal

**[File: apps/creative-portal/components/pages/authenticator-app/hooks/useTotpEnrollment.ts]**
**Function/Class:** useTotpEnrollment.handleVerify
**Severity:** low
**Problem:** `setIsRedirecting(true)` is called unconditionally before checking `onComplete`. When `onComplete` is provided (e.g., from an onboarding wrapper), the component enters a "redirecting" state (likely showing a spinner) even though `onComplete` may handle navigation differently.
**Impact:** Minor UX — a brief loading/spinner state may flash before `onComplete` takes effect. The customer portal version has the same pattern.
**Fix:** Move `setIsRedirecting(true)` into the branches that actually perform `router.push()`:

```typescript
if (onComplete) {
  onComplete();
  return;
}

setIsRedirecting(true);

if (!user?.onboardingComplete) {
  // ...onboarding step logic
}

router.push(appRoutes.root);
```

### 4. Unrelated formatting changes included

**[File: apps/creative-portal/components/pages/authenticator-app/index.tsx]**
**[File: apps/customer-portal/components/pages/authenticator-app/index.tsx]**
**[File: apps/creative-portal/pages/_app.tsx]**
**Function/Class:** getServerSideProps, App component
**Severity:** low
**Problem:** The PR includes parentheses removal from ternary expressions (`? (user.totpBackupCodes ?? backupCodes)` → `? user.totpBackupCodes ?? backupCodes`) and reformatting of `onError` callbacks. These are unrelated to the bug fix.
**Impact:** No functional impact, but adds noise to the diff and makes it harder to review the actual fix.
**Fix:** Consider reverting these formatting-only changes to keep the PR focused. If they're from Prettier/ESLint auto-fix, they're acceptable but should be noted.

---

## Tests

- ⚠️ 6 unit tests added but only test sessionStorage API directly, not the component behavior
- ❌ No component render tests for `BackupCodesModalTrigger` (missing `onOpen` callback verification)
- ❌ No tests for ModalQueue ordering logic
- ✅ Existing test suite passes (1,083+ tests per PR description)
- ⚠️ Manual test plan documented but not yet executed

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Regression: missing `onOpen(false)` blocks downstream modals |
| Regression risk | ❌ High — TrustBrowserModal and NewTimezoneModal won't show on normal page loads |
| Tests | ⚠️ Tests exist but don't cover the component or the regression |
| Code quality | ✅ Clean architecture, follows existing patterns well |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Request changes**

1. **Critical:** Add `onOpen?.(false)` in the `BackupCodesModalTrigger` useEffect when no codes are found in sessionStorage (and in the catch block). Without this, `TrustBrowserModal` and `NewTimezoneModal` are blocked on every normal page load.
2. **Important:** Add component-level render tests that verify `onOpen` is called correctly in both the "codes exist" and "no codes" scenarios.
3. **Minor:** Consider moving `setIsRedirecting(true)` after the `onComplete` check to avoid a brief flash of loading state.
