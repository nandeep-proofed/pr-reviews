# PR Review: feature/PP-1694: send dob to wise on recipient creation surface wise recipient creation errors to user and sentry

**PR:** https://github.com/Proofed/B2BWebserver/pull/2114
**Jira:** https://proofed.atlassian.net/browse/PP-1694
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Include DOB in Wise "create recipient" request (onboarding + settings) | `hooks.ts` injects `userDateOfBirth` (from profile) or form-entered DOB into `transformedData.details.dateOfBirth` before calling `mutate` | ✅ Addressed |
| DOB source of truth is Proofed user profile | Fetches from `useSelfPersonalInfoQuery().profileData.birthDate`, formats via `formatBirthDateToYyyyMmDd` | ✅ Addressed |
| User-visible error on Wise recipient creation failure | Adds `showDefaultErrorToast(error, firstErrorMessage)` after field-level error handling in `onError` callback | ✅ Addressed |
| Error message sourced from Wise response where safe | Uses `data.error.errors[0]?.message` with fallback to generic message | ✅ Addressed |
| UI keeps user in editable state on error | `enableReinitialize` changed to conditional (`false` in settings, conditional in onboarding) preventing form reset on error | ✅ Addressed |
| Log Wise errors to Sentry with user ID, trigger path, Wise error ID, status, redacted payload | API handler sets `Sentry.setContext("wise_recipient_creation", {...})` with all required fields | ✅ Addressed |
| Block recipient creation when DOB required but missing | Client-side guard checks `DOB_REQUIRED_TYPES` against `transformedData.type`, shows toast + Sentry warning | ✅ Addressed |
| Missing DOB: UI shows error, Sentry event logged | `showToast` with error message + `Sentry.captureMessage` with warning level | ✅ Addressed |

**Beyond scope:** `throwSentryError.ts` — serializes `requestData`/`responseData` to JSON strings. This is a general improvement to Sentry error reporting, tangential to PP-1694.

---

## Architecture Analysis

The approach threads DOB through three layers:

1. **Data source:** `useSelfPersonalInfoQuery` fetches profile data; `formatBirthDateToYyyyMmDd` normalizes date format.
2. **Form/context layer:** `userDateOfBirth` passes through `PaymentDetailsStep2Props` → `RequirementsContext` → `WiseField` to hide the DOB field when profile DOB exists. The submission hook injects DOB into the payload.
3. **API layer:** `createRecipient.ts` strips `_triggerPath` and `_userId` metadata before forwarding to Wise, and enriches Sentry context on failure.

The metadata-stripping pattern (prefixing client-only fields with `_`) is a pragmatic approach that avoids a separate API contract, though it couples client and server via convention.

---

## Issues Found

### 1. Double error toast on Wise field-level errors

**[File: apps/creative-portal/components/pages/common/steps/PaymentDetailsStep2/hooks.ts]**
**Function/Class:** `usePaymentDetailsForm2` (onError callback, ~line 167-173 in diff)
**Severity:** medium
**Problem:** After the `forEach` loop that sets field-level errors and optionally shows toasts for path-less errors, the code unconditionally calls `showDefaultErrorToast(error, firstErrorMessage)`. When Wise returns field-level errors (with `path`), the user sees field-level validation indicators AND a toast saying "Please fix the errors below." — but when Wise returns path-less errors, the `forEach` already showed a toast via `showDefaultErrorToast(error, message)`, and then a second toast fires with `firstErrorMessage`. This results in duplicate toasts for path-less errors.
**Impact:** Users see two error toasts stacked on top of each other for the same error, which is confusing.
**Fix:** Only show the summary toast when there are field-level (path-based) errors and no generic toast was already shown:

```typescript
let hasFieldErrors = false;
data.error.errors.forEach(({ path, message }) => {
  if (path) {
    setFieldError(`details.${path}`, message);
    hasFieldErrors = true;
  } else {
    showDefaultErrorToast(error, message);
  }
});

if (hasFieldErrors) {
  const firstErrorMessage =
    data.error.errors[0]?.message ??
    "Please fix the errors below.";
  showDefaultErrorToast(error, firstErrorMessage);
}
```

### 2. `DOB_REQUIRED_TYPES` hardcoded in frontend hook

**[File: apps/creative-portal/components/pages/common/steps/PaymentDetailsStep2/hooks.ts]**
**Function/Class:** `usePaymentDetailsForm2` (handleSubmit callback)
**Severity:** low
**Problem:** `const DOB_REQUIRED_TYPES = ["southafrica"]` is defined inline as a constant inside a `useCallback`. If Wise adds more countries requiring DOB, this needs a code change. Additionally, the constant is re-created on every callback invocation.
**Impact:** Minor maintenance burden. No runtime issue since the array is small.
**Fix:** Move to a module-level constant at the top of the file:

```typescript
const DOB_REQUIRED_TYPES = ["southafrica"] as const;
```

### 3. `Sentry.setContext` called outside the Sentry event scope in API handler

**[File: apps/creative-portal/api/wise/createRecipient/createRecipient.ts]**
**Function/Class:** `createRecipient` (catch block)
**Severity:** medium
**Problem:** `Sentry.setContext("wise_recipient_creation", {...})` is called after the error is thrown but before `handleEndpointError` processes it. `setContext` sets context globally on the current scope — if `handleEndpointError` internally calls `Sentry.captureException`, the context will be attached. But if it doesn't (or if the error is handled differently), the context persists on the scope and could leak into unrelated future events on the same request.
**Impact:** Potential context leakage to unrelated Sentry events within the same request lifecycle. The context will correctly attach to the next Sentry event, but may also attach to subsequent ones.
**Fix:** Use `Sentry.withScope` to isolate the context, or call `Sentry.captureException` directly with the context before calling `handleEndpointError`:

```typescript
Sentry.captureException(error, {
  contexts: {
    wise_recipient_creation: {
      userId,
      triggerPath,
      wiseErrorId,
      wiseStatus,
      wiseErrorRedacted
    }
  }
});
return handleEndpointError(error, res, req.log);
```

### 4. `refetchInterval` change in onboarding hook may suppress polling prematurely

**[File: apps/creative-portal/components/pages/onboarding/step2/hooks.ts]**
**Function/Class:** `usePaymentDetailsStep` (refetchInterval logic)
**Severity:** medium
**Problem:** The new logic returns `false` (stop polling) when `isFormVisible` is true (i.e., user hasn't set up payment yet OR is editing). The original code polled every 1s until tax declaration and contract filenames appeared. The new code stops polling entirely when the form is visible, which means the form won't detect when contract/tax documents are signed while the user is on the payment form. This could break the existing flow where the system detects document signing in the background.
**Impact:** During onboarding, if the user is on the payment form and a document gets signed in the background (e.g., via a webhook or background process), the UI won't detect it because polling is stopped.
**Fix:** Verify this is intentional behavior. If the form being visible should still detect document changes, the polling logic should be:

```typescript
refetchInterval: (details) => {
  if (!details?.userData.taxDeclarationFilename ||
      !details?.userData.contractFilename) {
    return 1000;
  }
  return false;
}
```

### 5. `isFormReady` check uses loose equality (`==`)

**[File: apps/creative-portal/components/pages/settings/payment-information/hooks.tsx]**
**Function/Class:** `usePaymentInformation`
**Severity:** low
**Problem:** `userDetails.taxCountryCode == null` uses loose equality. While this correctly covers both `null` and `undefined`, the codebase typically uses strict equality. ESLint's `eqeqeq` rule may flag this.
**Impact:** No functional issue — this is a style concern. Loose `== null` is actually a common and intentional JS idiom for null/undefined checks.
**Fix:** This is acceptable as-is. If ESLint flags it, use `userDetails.taxCountryCode === null || userDetails.taxCountryCode === undefined`.

### 6. `toJsonString` return type is `string | unknown`

**[File: packages/shared/utils/throwSentryError.ts]**
**Function/Class:** `toJsonString`
**Severity:** low
**Problem:** The return type `string | unknown` simplifies to just `unknown`, making the type signature misleading. The function returns `null`, `undefined`, `string`, or the original value — but the type annotation doesn't communicate this.
**Impact:** No runtime issue, but the type is imprecise.
**Fix:** Use a more precise return type:

```typescript
function toJsonString(value: unknown): unknown {
```

Or if you want to be explicit:

```typescript
function toJsonString(value: unknown): string | null | undefined | unknown {
```

### 7. Missing `Sentry` import in hooks.ts (from diff)

**[File: apps/creative-portal/components/pages/common/steps/PaymentDetailsStep2/hooks.ts]**
**Function/Class:** module imports
**Severity:** N/A — false alarm
This is correctly added in the diff (`import * as Sentry from "@sentry/nextjs"`). No issue.

### 8. `updateRequirements` not called for hidden DOB field

**[File: apps/creative-portal/components/pages/common/steps/PaymentDetailsStep2/WiseField/index.tsx]**
**Function/Class:** `WiseField` (date case)
**Severity:** low
**Problem:** When `field.key === "dateOfBirth" && userDateOfBirth`, the component returns `null` without calling `updateRequirements`. For other field types like `select`, `updateRequirements` is called `onChange` to refresh Wise requirements. If the date of birth field's value affects subsequent Wise requirements, hiding it without triggering an update could cause the requirements to be incomplete.
**Impact:** Likely low — DOB probably doesn't affect subsequent Wise requirement fields. But worth verifying.
**Fix:** Confirm that DOB is not a `refreshRequirementsOnChange` field in the Wise requirements API. If it is, an effect should call `updateRequirements(userDateOfBirth, "details.dateOfBirth")` when the field is hidden.

---

## Tests

- ✅ Unit tests added for `formatBirthDateToYyyyMmDd` utility (54 lines, covers null/undefined, empty string, ISO format, dd/MM/yyyy, whitespace, invalid dates)
- ✅ Tests updated for `throwSentryError` (existing test updated for JSON stringification, new test for non-object data pass-through)
- ❌ No unit tests for `usePaymentDetailsForm2` hook changes (DOB injection, DOB-required blocking, duplicate toast behavior, Sentry logging)
- ❌ No unit tests for `WiseField` date case rendering
- ❌ No unit tests for `createRecipient` API handler Sentry context enrichment
- ⚠️ PR checklist shows unit tests not checked off

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Double toast issue on Wise errors; refetchInterval behavior change needs verification |
| Regression risk | ⚠️ Medium — `enableReinitialize` and `refetchInterval` changes affect existing form behavior |
| Tests | ⚠️ Utility tests good, but no tests for core hook behavior changes |
| Code quality | ✅ Clean, well-commented, good Sentry context structure |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Request changes**

1. **Fix double toast** (Issue #1): The unconditional `showDefaultErrorToast` after the field-error `forEach` will show duplicate toasts when Wise returns path-less errors. Gate it behind a `hasFieldErrors` check.
2. **Verify refetchInterval change** (Issue #4): Confirm that stopping polling when the payment form is visible is intentional and doesn't break contract/tax document detection during onboarding.
3. **Isolate Sentry context** (Issue #3): Use `Sentry.captureException` with contexts option or `Sentry.withScope` to prevent context leakage.
4. **Add tests for hook behavior**: The DOB injection logic, DOB-required guard, and error handling changes in `usePaymentDetailsForm2` are the core of this ticket and should have tests.
5. **Move `DOB_REQUIRED_TYPES` to module scope** (Issue #2): Minor cleanup to avoid re-creating the array in every callback invocation.
