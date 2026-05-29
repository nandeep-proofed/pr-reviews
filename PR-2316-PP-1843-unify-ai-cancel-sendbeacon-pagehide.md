# PR Review: feature/PP-1843: Unify AI cancel delivery on sendBeacon + handle pagehide

**PR:** https://github.com/Proofed/B2BWebserver/pull/2316
**Jira:** https://proofed.atlassian.net/browse/PP-1843
**Status:** Open · Draft=false · Mergeable=clean · Base=`fix/PP-1843-failed-toast-and-expand-fields` (depends on #2313)

> Jira ticket content could not be fetched directly — Atlassian MCP requires OAuth in this session. Requirements below are derived from the PR description (which restates the two concrete bugs from the ticket) and from inspection of the prior `useActiveUuidLifecycle` behavior on `develop`. If the live Jira ticket adds acceptance criteria not reflected here, this row mapping should be re-validated.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Explicit Discard/Cancel during AI drafting must reach the server even when the panel immediately unmounts (React Query AbortController was aborting the cancel POST). | `cancelInFlightAndClear` now calls `fireBeaconCancel` (sendBeacon, with `fetch keepalive: true` fallback). React Query is no longer in the cancel path, so unmount cannot abort. | ✅ |
| Page refresh / tab close / hard navigation during AI drafting must dispatch a cancel (React `useEffect` cleanup doesn't run on browser unload). | New `pagehide` listener calls `fireLeaveCancel` (shared helper with the unmount cleanup). | ✅ |
| Back-forward cache restores must NOT cancel work the user can come back to. | `handlePageHide` short-circuits when `event.persisted === true`. | ✅ |
| Delivery semantics for all three "I'm leaving" scenarios (explicit cancel, React unmount, browser unload) should be uniform. | All three paths funnel through `fireBeaconCancel`. | ✅ |
| Slow shutdown — pagehide arrives, then React teardown runs — must not double-fire a cancel for the same uuid. | `fireLeaveCancel` clears `activeUuidRef` after firing; `cancelInFlightAndClear` clears first then fires. Both make a follow-up call a no-op. | ✅ |
| Submit-uuid sync + terminal-status clear should share one source of truth for `activeUuidRef`. | Effects 1+2 collapsed into a single effect keyed on `[submit.data?.uuid, statusQuery.data?.status]`. | ✅ |
| Tests cover new behaviour. | 13 cases on `useActiveUuidLifecycle` (`it.each` over completed/failed/cancelled, pagehide+persisted, dedupe, listener removal). `useAiFeedbackPanel` `onCancel` test updated to assert `sendBeacon` is called with the cancel URL. | ✅ |

Scope check: nothing beyond the cancel-delivery story. No incidental refactors, no UI changes.

---

## Architecture Analysis

The hook ownership story is unchanged: `useActiveUuidLifecycle` still owns `activeUuidRef` and exposes `cancelInFlightAndClear`. What changes is the *delivery primitive*:

- **Before:** `cancelInFlightAndClear` → React Query mutation → axios POST → AbortController on unmount → request never reached the server.
- **After:** all three paths (explicit cancel, React unmount, `pagehide`) → `fireBeaconCancel` → `navigator.sendBeacon` (with `fetch keepalive: true` fallback) → fire-and-forget, immune to unmount.

The state machine for `activeUuidRef` consolidates from 3 effects to 2 — a single set/clear effect, plus the leave-listener effect. Both leave triggers (`pagehide` and unmount cleanup) share the inner `fireLeaveCancel` closure, which dedupes by clearing the ref after the beacon. `cancelInFlightAndClear` dedupes in the opposite order (clear first, then beacon), which is fine since it captures `uuid` into a local before clearing.

`useCancelAiReviewFeedbackMutation` is no longer consumed by `useAiFeedbackPanel`. The service export (in `apps/creative-portal/services/aiReviewFeedback/index.ts`) is intentionally retained per the PR description, and its unit test (`services/aiReviewFeedback/index.test.tsx`) still covers it.

`pagehide` (vs `beforeunload`) is the right choice: it fires reliably on iOS Safari and tab close, carries the bfcache signal, and is the modern recommendation for unload-time work.

`Object.defineProperty(navigator, "sendBeacon", ...)` is used in the tests in place of `vi.stubGlobal("navigator", { sendBeacon })` — a more surgical replacement that doesn't blow away the rest of the navigator object.

---

## Issues Found

### 1. Listener-removal assertion in `removes the pagehide listener on unmount` is not strictly isolated

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/hooks/useActiveUuidLifecycle.test.tsx]**
**Function/Class:** test case `removes the pagehide listener on unmount`
**Severity:** low
**Problem:** The test unmounts the hook, asserts the unmount-cleanup beacon, clears the mock, dispatches a fresh `pagehide`, and asserts `sendBeacon` was not called again. The "not called" outcome holds whether or not the listener was actually removed, because the unmount cleanup also clears `activeUuidRef`. Even if `removeEventListener` were never called, the surviving handler would hit `if (!uuid) return;` and exit silently.
**Impact:** A regression that drops `window.removeEventListener("pagehide", handlePageHide)` from the cleanup would leak a listener per mount/unmount cycle and never be caught by this test.
**Fix:** Spy on `removeEventListener` directly, or set up a state where the ref could repopulate without re-mounting (e.g., re-attach a uuid via a different test surface). A `vi.spyOn(window, "removeEventListener")` assertion is simplest:

```typescript
const removeSpy = vi.spyOn(window, "removeEventListener");
// ... unmount ...
expect(removeSpy).toHaveBeenCalledWith("pagehide", expect.any(Function));
```

### 2. `useCancelAiReviewFeedbackMutation` is now dead code in the app

**[File: apps/creative-portal/services/aiReviewFeedback/index.ts]**
**Function/Class:** `useCancelAiReviewFeedbackMutation` (and `cancelAiReviewFeedback`)
**Severity:** low
**Problem:** After this PR, no production code path imports `useCancelAiReviewFeedbackMutation`. The service unit test still exercises it, but nothing else does. The PR description acknowledges this and keeps the export "for any future caller."
**Impact:** Dead code accrues maintenance cost (renaming, type drift, hooked-into refactors) without paying for itself. The side effect it carried — `onSuccess: queryClient.removeQueries(...)` for the polling cache — is also unused.
**Fix:** Either delete the unused hook (and its service-test coverage) in a follow-up, or add a one-line comment at the export pointing to the intended future caller / ticket. If kept, no change is required here.

Note on the lost `removeQueries` side effect: in practice it doesn't matter. The status query is keyed on `submit.data?.uuid`; after `submit.reset()` the key becomes `["", ""]` and `enabled: !!uuid` flips false, so polling stops. The orphaned cache entry is inert and gets garbage-collected by React Query in due course.

### 3. Pre-existing gap: "Discard during the submit round-trip" still does not cancel

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/hooks/useActiveUuidLifecycle.ts]**
**Function/Class:** `cancelInFlightAndClear`, merged effect on `[submit.data?.uuid, statusQuery.data?.status]`
**Severity:** low (informational; pre-existing, not regressed)
**Problem:** Between `submit.mutate(...)` firing and the BFF returning a uuid, `activeUuidRef.current` is `undefined`. If the user clicks Discard, refreshes, or closes the tab during that ~submit POST round-trip, none of the three handlers fires a cancel — the upstream Proofed.ai job will be allocated server-side with no client cancel.
**Impact:** A small window where the unify-on-leave story still has a hole. The PR's own scope is the three scenarios where a uuid already exists, so this is out of scope — flagging only because the PR aims for "uniform delivery semantics across all leave scenarios," and the submit-window case is the residual asymmetry.
**Fix:** Out of scope for this PR. If addressed later: have the BFF correlate the submit POST against an idempotency key the client also retains, so a follow-up cancel can target a uuid the client never directly observed; or send the cancel signal back over a shared `AbortController` that the BFF honors on the upstream POST.

---

## Tests

- ✅ Unit tests added/updated. `useActiveUuidLifecycle.test.tsx` has 13 cases organised into four `describe` blocks (`cancelInFlightAndClear`, terminal-status clear, React unmount cleanup, pagehide handler).
- ✅ `it.each(["completed", "failed", "cancelled"])` exercises all three terminal statuses against the merged effect.
- ✅ `does not double-fire when React unmount follows a pagehide` covers the slow-shutdown dedupe path.
- ✅ `re-sets the ref when a fresh submit follows a terminal status` covers the post-completion re-submit edge.
- ✅ `useAiFeedbackPanel.test.tsx` `onCancel` test updated to assert `sendBeacon` is called with `/api/aiReviewFeedback/cancel?uuid=abc` instead of `cancel.mutate("abc")`. The mock-setup table for `cancel` is removed across all cases in the suite.
- ⚠️ `removes the pagehide listener on unmount` does not strictly assert listener removal (see Issue 1).
- ⚠️ Manual / staging smoke is checklisted but not yet run. The PR description's staging plan (8 scenarios) is the right surface — particularly scenarios 4–6 (refresh, tab close, hard nav) which were the original user-reported bug.
- N/A E2E (no Playwright change).
- CI: `get_check_runs` returned 0 runs at review time — CI may not have started or may not surface here. Verify green before merge.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ |
| Regression risk | ✅ Low |
| Tests | ⚠️ Strong coverage; one assertion (listener removal) is weaker than it looks |
| Code quality | ✅ |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions.**

1. Tighten the listener-removal assertion in `useActiveUuidLifecycle.test.tsx` by spying on `window.removeEventListener` directly so a regression that drops the `removeEventListener` line would actually fail the test (Issue 1).
2. Run the 8-scenario staging smoke listed in the PR description before declaring the user-reported bug closed — particularly the page-refresh, tab-close, and hard-nav cases that the unmount-only beacon never covered.
3. After this lands, file a quick follow-up to either delete `useCancelAiReviewFeedbackMutation` + `cancelAiReviewFeedback` from `services/aiReviewFeedback`, or leave a pointer to the intended future caller so the dead-code state is intentional and discoverable (Issue 2).
4. Merge order matters: per the PR description, #2313 must merge first; if it gets rebased/squashed onto `develop`, this branch will need a rebase before its diff is clean.
