# PR Review: fix/PP-1842 — Populate editor profile form with existing DB values

**PR:** https://github.com/Proofed/B2BWebserver/pull/2288
**Jira:** https://proofed.atlassian.net/browse/PP-1842
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Editor profile fields display DB values on load (criterion #1) | `enableReinitialize` on `<Formik>` lets the form absorb `initialValues` once the query resolves | ✅ Addressed |
| Changes made in the UI are persisted to the DB (criterion #2) | `AutoSaveOnChange` child component calls `saveEditorProfileInformations` via a 500 ms debounce on every dirty-form value change | ✅ Addressed |
| No regression on Settings → Your profile surface | Same `YourProfile` component drives both routes; fix applies to both | ✅ Addressed |
| No regression on onboarding step 6 (PR #2214's fix) | Onboarding uses a separate Formik instance; `YourProfile` change is isolated | ✅ Addressed |

---

## Architecture Analysis

The root-cause analysis in the PR description is accurate:

1. **Missing `enableReinitialize`** — Formik ignores `initialValues` updates after initial mount by default. The query resolves asynchronously (returning `undefined` first, then data), so without `enableReinitialize` the form was always constructed from `undefined`/empty values and never updated. Adding `enableReinitialize` is the canonical Formik fix.

2. **`onChange={submitForm}` triggered validation-gated submit on every keystroke** — because `validateOnChange: true` and all 7 fields are `.required()`, every in-progress keystroke failed validation and blocked `onSubmit`. Removing this in favour of a direct call to `saveEditorProfileInformations` bypasses the validation gate while keeping the visual validation UX intact.

The `AutoSaveOnChange` pattern (a render-null Formik subscriber that fires a side effect) is clean and well-tested. `debouncedSave` is correctly memoized in `YourProfile` to avoid a fresh debounce instance per render.

---

## Issues Found

### 1. `AutoSaveOnChange` saves without client-side validation — partial/empty values reach the API

**[File: apps/creative-portal/components/pages/settings/Partials/YourProfile/Partials/AutoSaveOnChange/index.tsx]**
**Function/Class:** AutoSaveOnChange
**Severity:** medium
**Problem:** The effect fires `onChange(values)` whenever the form is dirty, with no check against Formik's `isValid` or `errors`. The hook's `debouncedSubmit` (in `hooks.ts:199`) validates via `validateForm()` before calling `saveEditorProfileInformations`, but `AutoSaveOnChange` skips that entirely. A user who clears a previously-filled required field will trigger a `PATCH` with an empty string, despite the form showing a Yup validation error.
**Impact:** Invalid data (e.g. blank required fields) can be sent to `PATCH /api/users/:id/info`. Whether this corrupts the DB depends solely on server-side validation. The PR description notes the server validates, but the client-side guard that existed in `debouncedSubmit` is now absent on this surface.
**Fix:** Thread `isValid` from `useFormikContext` and guard the effect:

```tsx
const { values, dirty, isValid } = useFormikContext<PatchUserRequestBody>();

useEffect(() => {
  if (!dirty || !isValid) return;
  onChange(values);
}, [values, dirty, isValid, onChange]);
```

---

### 2. `debouncedSave` is never cancelled on unmount — potential post-unmount API call

**[File: apps/creative-portal/components/pages/settings/Partials/YourProfile/index.tsx]**
**Function/Class:** YourProfile
**Severity:** low
**Problem:** `debouncedSave` is created with `useMemo`, which has no cleanup mechanism. If the component unmounts within the 500 ms debounce window (e.g. the user navigates away immediately after typing), the queued invocation will still fire and call `saveEditorProfileInformations` after unmount.
**Impact:** Low in practice — TanStack Query mutations do not set local React state, so no React warning is expected. However it can cause a spurious network request and a confusing success/error toast after the user has already left the page.
**Fix:** Create the debounce inside a `useEffect` with a cancel cleanup:

```tsx
const debouncedSaveRef = useRef<ReturnType<typeof debounce>>();

useEffect(() => {
  debouncedSaveRef.current = debounce(
    (values: PatchUserRequestBody) =>
      saveEditorProfileInformations(values),
    500
  );
  return () => debouncedSaveRef.current?.cancel();
}, [saveEditorProfileInformations]);
```

Then pass `debouncedSaveRef.current` to `AutoSaveOnChange`. Alternatively, keep `useMemo` and add a cleanup `useEffect`:

```tsx
useEffect(() => () => debouncedSave.cancel(), [debouncedSave]);
```

---

### 3. `formProps.onSubmit` (`debouncedSubmit`) is now dead code on this surface

**[File: apps/creative-portal/components/pages/common/steps/EditorProfileStep/hooks.ts]**
**Function/Class:** `useEditorProfileStep`
**Severity:** low
**Problem:** When `saveOnChange: true`, `formProps.onSubmit` is set to `debouncedSubmit` (line 212), which validates then saves. `YourProfile` still passes `saveOnChange: true` but never triggers Formik submit (no submit button, no `submitForm` call). `debouncedSubmit` is therefore unreachable on this surface. The `saveOnChange` flag's only active effect is now on `saveHeadshotFile` (line 124 in hooks.ts).
**Impact:** No functional bug, but future maintainers may be confused about why `saveOnChange: true` is passed or why `debouncedSubmit` exists. If someone adds a submit button or restores `onChange={submitForm}`, they'll get the original validation-blocked behaviour again.
**Fix:** A comment in `YourProfile` explaining that `saveOnChange` here only affects headshot saving, and that text-field auto-save is handled by `AutoSaveOnChange`:

```tsx
// saveOnChange: true enables auto-save for headshot changes (see hooks.ts:saveHeadshotFile).
// Text-field auto-save is handled separately via AutoSaveOnChange below,
// which bypasses the validation gate that blocked saves with the previous onChange={submitForm} approach.
const { ... } = useEditorProfileStep({ saveOnChange: true });
```

---

### 4. Test uses `fireEvent` for user interaction rather than `userEvent`

**[File: apps/creative-portal/components/pages/settings/Partials/YourProfile/index.test.tsx]**
**Function/Class:** "saves the freshly typed value, not stale closure values"
**Severity:** low
**Problem:** `fireEvent.click` is used to simulate a user typing (via a mock button that calls `setFieldValue`). This is a deliberate workaround because the real textarea is mocked, so it's acceptable here. However it's worth noting that real typing isn't exercised — the test proves the debounce wiring but not that the actual `FormikTextarea` fields feed `values` correctly.
**Impact:** Edge cases in `FormikTextarea`→`values` propagation (e.g. a trimming transform, a custom `onChange` handler in the textarea) are not caught by these tests. Not a blocker since the mock is self-contained.
**Fix:** No change required for this PR. Consider adding an integration-level test that renders the actual `FormikTextarea` fields in a future iteration.

---

## Tests

- ✅ Two unit tests added covering exactly the two PP-1842 acceptance criteria
- ✅ Test 1 — `enableReinitialize`: verifies that a `rerender` with updated `initialValues` reflects in the displayed values (criterion #1)
- ✅ Test 2 — debounced save: verifies `saveEditorProfileInformations` is called with the typed value after 500 ms, not immediately (criterion #2)
- ✅ Fake timers used correctly for debounce control (`vi.useFakeTimers` + `vi.advanceTimersByTime(500)`)
- ✅ `vi.useRealTimers()` restored in `afterEach`
- ⚠️ No test covers the validation-bypass behaviour (i.e. that a save fires even when the form is invalid) — this is the direct consequence of issue #1 and would need updating if `isValid` guard is added
- ⚠️ `AutoSaveOnChange` has no dedicated unit test (it's exercised transitively via `YourProfile` tests, which is sufficient)

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Fixes both PP-1842 criteria |
| Regression risk | ✅ Low — change is isolated to `YourProfile`; onboarding, headshot saving, and team-member routing are unaffected |
| Tests | ✅ Good coverage of the two acceptance criteria |
| Code quality | ⚠️ Minor concerns (validation bypass, no unmount cancel) |
| Mergeable state | ⚠️ Mergeable with suggestion |

---

## Recommendation

**Approve with suggestions**

1. **(Medium — consider before merge)** Add `isValid` guard to `AutoSaveOnChange` to prevent sending invalid data to the API when a user clears a required field.
2. **(Low — can follow up)** Add `debouncedSave.cancel()` cleanup on unmount to avoid post-navigation API calls.
3. **(Low — can follow up)** Add a comment in `YourProfile` clarifying why `saveOnChange: true` is passed and the role of `AutoSaveOnChange`.
