# PR Review: fix/PP-1883: lock toggle discard popup

**PR:** https://github.com/Proofed/B2BWebserver/pull/2311
**Jira:** https://proofed.atlassian.net/browse/PP-1883
**Status:** Code Review

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Toggling the lock with **no** unsaved changes should switch the lock state directly, with no "Discard unsaved changes?" popup | `onClick={(event) => event.stopPropagation()}` on `Styled.ChipWrapper` stops the click from bubbling to the `Dropdown`'s internal trigger span (`onClick={toggle}`), so the popup is no longer self-opened. The popup is now driven solely by `isOpened={isLockConfirmationOpen}` | ✅ Addressed |
| Popup should appear **only** when there are genuine unsaved changes | `isLockConfirmationOpen` is set `true` only inside `handleLockToggle` when `!isReadyForOrderCreation && hasUnsavedChanges`. The spurious internal-toggle path that bypassed that condition is removed | ✅ Addressed |
| Drag/sloppy click must not trigger the popup (clean click already worked) | `stopPropagation` neutralizes the bubbling-click self-toggle regardless of click style (clean vs. dragged), so the intermittent behavior is eliminated | ✅ Addressed |
| Lock state must not end up reverted/double-toggled after confirming "Discard and Lock" | The reversal came from `handleChange()` running once via the Switch's `onChange` and again via `handleDiscardAndLock` after a spuriously-opened popup. Suppressing the spurious popup removes the second `handleChange` path | ✅ Addressed |
| Behavior of this and other dropdowns' dismissal must be unchanged | `click` `stopPropagation` does not affect `mousedown` — and `useOnClickOutside` defaults to `mousedown` (verified in `usehooks-ts` `index.cjs:533`). The Switch's `onChange` (a `change` event) is also unaffected. Shared `Dropdown`/`ConfirmationDropdown` are untouched | ✅ Addressed |
| Every PR must include tests for new code | Regression test added at `packages/shared/components/molecules/Dropdown/index.test.tsx` guarding the self-toggle contract | ⚠️ Partial — see Issue #1 |

No scope creep: the production change is a single JSX attribute on one local page. The only other file is a new test.

---

## Architecture Analysis

The root cause analysis in the PR description is accurate and matches the code:

- The shared `Dropdown`'s wrapper visibility is `isOpen={isOpened || isOpen}` (`index.tsx:336`), where `isOpened` is the parent-controlled prop and `isOpen` is an internal `useToggle` flipped by `onClick={toggle}` on the trigger span `StyledDropdownButtonAsSpan` (`index.tsx:342`).
- `ConfirmationDropdown` passes the lock `trigger` through as the Dropdown `label`, so it renders **inside** that always-on click-to-toggle span.
- A click anywhere in the `Switch`/`Chip` therefore bubbled to the span and flipped the internal `isOpen`, opening the confirmation popup independently of the parent's `isLockConfirmationOpen`. A clean click on the label generated an even number of bubbled clicks (label + synthesized input click) that cancelled out; a dragged click did not — hence the intermittent reproduction.

The fix stops the click at `ChipWrapper`, leaving `isOpened` (= `isLockConfirmationOpen`) as the sole driver. This is correct, minimal, and avoids touching the shared component — appropriate given the blast radius (`ConfirmationDropdown` has 6+ consumers).

The fix targets the symptom location (the page) rather than the underlying design coupling in `Dropdown` (an internal click-toggle that is always active even when the component is used as a fully-controlled popover via `isOpened`). Given the constraints, the local fix is the pragmatic choice, and the added test pins the `Dropdown` contract it relies on so a future shared-component change can't silently break it.

The other `ConfirmationDropdown` consumers (`JobTemplates`, `SingleOrderTemplate`, `ScheduleOverrides`, `DropdownFilterWithConfirmationPanel`) use the default icon/button trigger, where the internal click-to-open **is** the intended behavior — so they correctly should not receive this change. The fix is properly scoped to the only place that uses an interactive `Switch` as a controlled trigger.

---

## Issues Found

### 1. No direct test for the actual page-level fix

**[File: packages/shared/components/molecules/Dropdown/index.test.tsx]**
**Function/Class:** Dropdown test — "does not self-toggle when the trigger content stops click propagation"
**Severity:** low
**Problem:** The added test verifies the *generic* `Dropdown` contract (a trigger whose content stops click propagation does not self-toggle) using a synthetic `<button onClick={stopPropagation}>` label. It does not exercise the actual change in `settings/index.tsx` — i.e. that `ChipWrapper` carries the `onClick`, that the `Switch`'s `onChange` (`handleLockToggle`) still fires while propagation is stopped, and that the popup opens **only** when `hasUnsavedChanges` is true. There is no test file for `settings/index.tsx` (only sub-partials are covered).
**Impact:** A future edit to `settings/index.tsx` (e.g. removing the `onClick`, or moving it off `ChipWrapper`) would not be caught by this suite; the regression could silently reappear. The guard protects the mechanism but not the wiring.
**Fix:** The Dropdown-contract test is a good and valid addition — keep it. Consider also adding a focused test (or extending an existing settings harness) asserting the integration behavior, e.g.:

```tsx
// pseudocode — render SettingsPageInner with controllable unsaved-changes context
it("does not open the discard popup when toggling the lock with no unsaved changes", () => {
  // no unsaved changes
  fireEvent.click(screen.getByText("Lock to mark as ready"));
  expect(handleChange).toHaveBeenCalledTimes(1);      // lock toggled directly, once
  expect(screen.queryByText("Discard unsaved changes?")).not.toBeInTheDocument();
});

it("opens the discard popup when toggling the lock with unsaved changes", () => {
  // unsavedChangesByKey.settings = true
  fireEvent.click(screen.getByText("Lock to mark as ready"));
  expect(screen.getByText("Discard unsaved changes?")).toBeInTheDocument();
  expect(handleChange).not.toHaveBeenCalled();
});
```

If a full page render is impractical, at minimum confirm the manual test plan in the PR (drag-click with and without unsaved changes) is recorded — the Before/After videos already cover this, which mitigates the gap.

### 2. Confirmation popup is not dismissible by clicking outside (pre-existing, now the only open path)

**[File: apps/creative-portal/components/pages/partners/[partnerId]/projects/[projectId]/settings/index.tsx]**
**Function/Class:** SettingsPageInner / ConfirmationDropdown wiring
**Severity:** low
**Problem:** After this fix the popup is driven exclusively by `isLockConfirmationOpen` (`isOpened`). The shared `Dropdown`'s outside-click handler (`handleClickOutside` → `toggleOff`) only flips the **internal** `isOpen`; it never clears the parent's `isLockConfirmationOpen`. So an outside click does not dismiss the popup — it can only be closed via "Discard and Lock" or "Cancel".
**Impact:** Minor UX inconsistency vs. typical dropdown/popover behavior. This is **not a regression introduced by this PR** — when `isOpened` was set `true` (legitimate unsaved-changes path) the popup already stayed open on outside click, because `isOpened` survives `toggleOff`. The fix just removes the other (buggy) open path, making this the only path. Flagging for awareness, not as a blocker.
**Fix:** Optional. If outside-click dismissal is desired, pass an `onClickOutsideHandler` to the underlying `Dropdown` (via `ConfirmationDropdown`) that calls `closeLockConfirmation`. Out of scope for PP-1883; only pursue if product wants it.

---

## Tests

- ✅ Regression test added (`Dropdown/index.test.tsx`) — control case (trigger self-toggles) + the stop-propagation case, mirroring the repo's existing styles-mock pattern (`FilterHeadingTrigger`).
- ✅ Mock of `./styles` is complete for the exports `Dropdown/index.tsx` imports; only `children` is rendered so `DropdownMenu`/icon paths aren't exercised — appropriate.
- ✅ Test assertions are sound: initial `onOpenChange(false)` on mount, `waitFor` for the `true` toggle, `not.toHaveBeenCalledWith(true)` for the stop-propagation case.
- ⚠️ No test exercises `settings/index.tsx` directly (the file the fix lives in) — see Issue #1.
- ✅ Before/After demonstration videos attached to the PR cover the manual repro.
- ⬜ Unable to execute the suite in this review session — recommend confirming `npx turbo run test` is green locally/CI before merge.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Fixes root cause of the symptom; addresses both the spurious popup and the lock-reversal |
| Regression risk | ✅ Low — single local JSX attribute; shared components untouched; `change`/`mousedown` paths unaffected |
| Tests | ⚠️ Mechanism guarded; page-level integration untested |
| Code quality | ✅ Minimal, well-reasoned, correctly scoped, follows conventions |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions**

1. (Nice-to-have) Add a page-level test asserting the popup opens only with unsaved changes and that `handleChange` runs exactly once on a no-changes toggle — or explicitly record the manual test plan to cover the integration gap (Issue #1).
2. Confirm `npx turbo run test`, `typecheck`, and `lint` are green before merge (per CLAUDE.md pre-commit gates).
3. (Optional, out of scope) Decide whether outside-click should dismiss the confirmation popup (Issue #2); if so, wire `onClickOutsideHandler` → `closeLockConfirmation`.
