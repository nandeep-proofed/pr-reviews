# PR Review: fix/PP-1843: Surface AI failed-status toast and expand polling fields

**PR:** https://github.com/Proofed/B2BWebserver/pull/2313
**Jira:** https://proofed.atlassian.net/browse/PP-1843
**Status:** open (mergeable: clean)

---

## Jira Requirements vs Implementation

Jira ticket access was not available in this session, so the requirements were
derived from the PR description (which states the ticket scope as "surface
upstream `narrative` field as an error toast when AI feedback fails and expand
the polling-response mapper") and from related PP-1843 PRs already merged
(#2309, #2312).

| Jira Requirement (per PR description) | PR Implementation | Status |
|---|---|---|
| When the AI review feedback service returns `status="failed"`, surface a reviewer-visible signal (not silent manual-fallback) | `useAiFeedbackPanel` status-poll effect calls `showDefaultErrorToast(undefined, latestStatus.narrative || GENERIC_AI_FAILURE_MESSAGE)` on `failed` | ✅ |
| Use the upstream `narrative` field as the toast message; fall back to generic copy when empty | Mapper carries `narrative` (null/empty → undefined via `isNonEmptyString`); hook uses `narrative \|\| GENERIC_AI_FAILURE_MESSAGE` | ✅ |
| Fire a generic toast on submit-mutation failures (network drop, 5xx, `ApiError(502)`) | New submit-error effect branch in `useAiFeedbackPanel` calls `showDefaultErrorToast(submit.error ?? statusQuery.error, GENERIC_AI_FAILURE_MESSAGE)` | ✅ |
| Dedupe the toast across React Query replays (same uuid) and across the two failure paths | `failureToastDedupeRef` tracks `statusUuid` + `submitErrorShown`; reset on `triggerAi` / `resetAll` | ✅ |
| Expand `/status` mapper to carry new optional fields: `narrative`, `diffCount`, `reportAvailable`, `submittedAt`, `completedAt`, `createdAt` | All six fields added to `AiReviewFeedbackStatusResponse` (upstream), `AiReviewFeedbackStatusClientResponse` (BFF), service-types, and the mapper | ✅ |
| Add tests for new code | Mapper: 5 new cases; `useAiFeedbackPanel`: 9 new cases (6 toast + 3 onAiFailedChange); `AiUnavailableHeader`: 3 new cases; `useReviewSubmissionFormState`: 2 new cases | ✅ |
| (Out-of-description scope, but in PR) Suppress `AiUnavailableHeader` tooltip after runtime failure since `AI_UNAVAILABLE_TOOLTIP` copy is about file-size limits and is misleading post-failure | New `suppressTooltip` prop on `AiUnavailableHeader`, new `onAiFailedChange` callback, new `isAiFailed`/`setIsAiFailed` from `useReviewSubmissionFormState` | ✅ (scope not in description) |
| (Out-of-description scope, but in PR) Update `AI_UNAVAILABLE_TOOLTIP` copy to mention PDF/XLSX/Google Drive limitations | `consts.ts:28` updated | ⚠️ Scope creep |

**Scope creep flags:**
1. The `AI_UNAVAILABLE_TOOLTIP` copy edit is unrelated to PP-1843's "failed-status
   toast" goal and is undocumented in the PR description's "Areas of Change."
2. Five of the six new mapper fields (`completedAt`, `createdAt`, `submittedAt`,
   `diffCount`, `reportAvailable`) are not consumed by any current code path —
   they are carried "for future UI without another mapper change." Per CLAUDE.md
   ("Don't add features… beyond what the task requires") this is YAGNI, though the
   change is trivial and well-contained.

---

## Architecture Analysis

The fix is layered cleanly:

- **BFF mapper (`apiHandlers/getStatus/utils.ts`):** Extended to translate
  snake_case → camelCase and collapse `null` → `undefined` via the existing
  `isNonEmptyString` guard. Consistent with the file's existing pattern for
  `tone_notes`.
- **Type propagation:** Upstream `AiReviewFeedbackStatusResponse` (snake_case
  + nullable), client `AiReviewFeedbackStatusClientResponse` (camelCase +
  optional), and the React-Query service type are all kept in sync. No type
  drift.
- **Hook orchestration (`useAiFeedbackPanel`):** Two failure paths now toast,
  both gated by a single `failureToastDedupeRef`. The status-poll path keys on
  `latestStatus.uuid` so a window-focus refetch with the same payload can't
  double-fire; the submit-error path uses a one-shot `submitErrorShown`
  boolean. Both gates reset on `triggerAi` and `resetAll`, so a fresh
  submission re-arms cleanly. This matches the existing `hasFailed` /
  `hasCancelled` reset pattern.
- **UI suppression of misleading tooltip:** New `onAiFailedChange` callback on
  `AiFeedbackPanelProps` lets parent forms know whether the panel reached
  `state === "failed"` (a strict subset of `AI_FALLBACK_STATES`). The two
  consuming forms (`ReviewSubmission` and `ReviewForm`) thread it through
  `useReviewSubmissionFormState` and pass `suppressTooltip={isAiFailed}` to
  `AiUnavailableHeader`. The header gains a simple branch that drops the
  `Tooltip` wrapper while preserving the `aria-label` for screen readers.

Security: `narrative` is rendered as text (not via `dangerouslySetInnerHTML`),
so React's automatic escaping handles any malicious content. Confirmed by
reading `showDefaultErrorToast` → `ToastErrorContainer` (passes `message` as
text prop). No new XSS surface.

---

## Issues Found

### 1. Unused mapper fields carried for hypothetical future use

**[File: apps/creative-portal/api/aiReviewFeedback/apiHandlers/getStatus/utils.ts]**
**Function/Class:** `mapAiReviewFeedbackStatusToResponse`
**Severity:** medium
**Problem:** Five of the six new fields (`completedAt`, `createdAt`, `submittedAt`, `diffCount`, `reportAvailable`) are not consumed by any caller in this PR or in the existing codebase — only `narrative` is used. The PR description justifies this as "cheap to carry now and useful for future UI without another mapper change."
**Impact:** Violates CLAUDE.md's "Don't add features, refactor, or introduce abstractions beyond what the task requires. […] Don't design for hypothetical future requirements." Reviewers and tooling need to maintain test cases (the new "carries populated timestamps and report flags through unchanged" test exercises a dead code path) and there is no contract pressure ensuring the upstream representation remains correct over time. Low real-world risk because the implementation is trivial, but it sets a precedent for the file.
**Fix:** Either (a) drop the unused fields from this PR and add them in the follow-up that actually consumes them, or (b) link a Jira sub-ticket in the comment that names the consuming feature and a target release.

### 2. `AI_UNAVAILABLE_TOOLTIP` copy update is unrelated to PP-1843

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/consts.ts]**
**Function/Class:** `AI_UNAVAILABLE_TOOLTIP`
**Severity:** low
**Problem:** The tooltip was updated from "documents over 30k words and files over Xmb." to "documents over 30k words, files over XMB, or PDF, XLSX, and Google Drive files." This change is undocumented in the PR description's "Areas of Change" list and is unrelated to the failed-status toast feature.
**Impact:** Reviewers cannot tell whether this copy was approved by product/design or was inferred. Hard-to-audit copy changes are how the wrong noun ships to production. If the copy was approved, it should be called out so the reviewer can verify against the design source; if not, it should be split out.
**Fix:** Either link the design / product source for the new copy in the PR description, or split into a separate PR / commit with its own approval trail.

### 3. `narrative || fallback` vs `narrative ?? fallback`

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/hooks/useAiFeedbackPanel.ts]**
**Function/Class:** `useAiFeedbackPanel` (status-poll failed branch, ~line 174)
**Severity:** low
**Problem:** The hook uses `latestStatus.narrative || GENERIC_AI_FAILURE_MESSAGE`. Because the mapper already collapses null/empty strings → undefined via `isNonEmptyString`, the behavior is currently equivalent to `??`. But the *typed* contract (`narrative?: string`) only excludes `undefined`, not the empty string. If the mapper ever changes its collapse policy, `||` will silently absorb a legitimate empty string while `??` would only fall back on null/undefined.
**Impact:** Subtle behavioral coupling between the mapper and the consumer. Easy to fix.
**Fix:** Prefer `??` to make the contract explicit:

```typescript
showDefaultErrorToast(
  undefined,
  latestStatus.narrative ?? GENERIC_AI_FAILURE_MESSAGE
);
```

### 4. Comment claim "Strict subset of isAiUnavailable" is enforced only indirectly

**[File: apps/creative-portal/components/organisms/sidebars/contents/hooks/useReviewSubmissionFormState.ts]**
**Function/Class:** `useReviewSubmissionFormState`
**Severity:** low
**Problem:** The new `isAiFailed: boolean` is documented as "Strict subset of isAiUnavailable: true only when the panel reached state === 'failed'", but it's an independent `useState` driven by two separate callbacks from `useAiFeedbackPanel` (`onAiFailedChange` and `onAiUnavailableChange`). The invariant holds today because both callbacks are fired from the same effect in the same render — but nothing in this hook enforces "`isAiFailed === true ⇒ isAiUnavailable === true`". A future refactor that decouples the two callbacks could break the invariant silently.
**Impact:** Minor — only relevant if someone refactors the callback firing order. The `AiUnavailableHeader` is already guarded by `isAiUnavailable || isWordCountTooLarge` in both consumers, so a stale `isAiFailed=true` while `isAiUnavailable=false` would simply mean the badge is hidden anyway.
**Fix:** Either derive `isAiFailed` from a wider state machine, or add a unit assertion that `isAiUnavailable` is true whenever `isAiFailed` becomes true (the existing "isAiFailed is independent of isAiUnavailable" test only tests the reverse direction).

### 5. Missing test: both `submit.error` and `statusQuery.error` set simultaneously

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/hooks/useAiFeedbackPanel.test.tsx]**
**Function/Class:** `useAiFeedbackPanel — failure toast`
**Severity:** low
**Problem:** The submit-error effect uses `submit.error ?? statusQuery.error` and is gated by `submit.isError || statusQuery.isError`. There is no test asserting which error is surfaced when both are simultaneously set, even though the code's intent (`submit.error` wins) is captured.
**Impact:** Documentation gap, not a correctness gap. The behavior is unlikely to occur in production but adding a test would lock the contract in.
**Fix:** Add a single test that wires both `submit.isError + submit.error` and `statusQuery.isError + statusQuery.error`, and asserts the call lands with `submit.error` as the first arg.

### 6. `AiFeedbackPanelProps` not strictly alphabetical

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/types.ts]**
**Function/Class:** `AiFeedbackPanelProps`
**Severity:** low
**Problem:** CLAUDE.md says "Fields in alphabetical order (enforced by `eslint-plugin-perfectionist/sort-interfaces`)." The interface mixes required and optional fields and is not in alphabetical order (e.g. `feedbackFieldName, jobId, noChangesRequired, orderId, scoreFieldName, onAiActiveChange, onAiBusyChange, onAiFailedChange, onAiUnavailableChange, uploadedFile, workItemFormat`). The new `onAiFailedChange` is correctly slotted between `onAiBusyChange` and `onAiUnavailableChange`, which matches the existing local ordering of the callbacks, but the interface as a whole groups by required-then-optional rather than strict alphabetical.
**Impact:** This is a pre-existing pattern in the file and ESLint is not catching it, so the change is consistent with the surrounding code. Flagging only for awareness — no action needed for this PR unless the team wants to enforce the rule.
**Fix:** No change required for this PR.

---

## Tests

- ✅ Mapper: 5 new tests covering narrative passthrough, null/empty narrative collapse, null timestamps + diff_count collapse, populated timestamps + report_available passthrough, and the existing `processing` baseline updated for the new undefined fields
- ✅ `useAiFeedbackPanel` failure toast: fires with narrative when present; falls back to generic when absent; dedupes across same-uuid replays; fires generic + error object on submit-mutation error; does not toast in the happy path; re-arms after `triggerAi`
- ✅ `useAiFeedbackPanel` onAiFailedChange signal: fires `true` on `state === "failed"`; stays `false` on `cancelled`; fires `false` initially
- ✅ `AiUnavailableHeader`: renders Tooltip by default; suppresses Tooltip when `suppressTooltip`; preserves `aria-label` for screen readers
- ✅ `useReviewSubmissionFormState`: `setIsAiFailed` flips state; `isAiFailed` is independent of `isAiUnavailable`
- ⚠️ Missing: no test for the case where both `submit.error` and `statusQuery.error` are simultaneously set (see Issue #5)
- ⚠️ Missing: no integration assertion that `ReviewSubmission` / `ReviewForm` pass `suppressTooltip={isAiFailed}` to `AiUnavailableHeader` (covered indirectly by the unit-level wiring)
- ✅ Manual test plan documented in PR description (staging smoke); marked as planned post-deploy

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ |
| Regression risk | ✅ Low |
| Tests | ✅ |
| Code quality | ⚠️ (minor: YAGNI on unused mapper fields, undocumented copy edit, `\|\|` vs `??`) |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions**

1. **Should-fix:** Either drop the five unused mapper fields (`completedAt`, `createdAt`, `submittedAt`, `diffCount`, `reportAvailable`) from this PR or link the consuming Jira ticket in the comment so the YAGNI is bounded. (Issue #1)
2. **Should-fix:** Add the `AI_UNAVAILABLE_TOOLTIP` copy change to the PR description with the product/design source, or split it out. (Issue #2)
3. **Nice-to-have:** Switch `narrative || fallback` to `narrative ?? fallback` in `useAiFeedbackPanel.ts` to decouple the consumer from the mapper's collapse policy. (Issue #3)
4. **Nice-to-have:** Add a single test asserting `submit.error` wins over `statusQuery.error` when both are set. (Issue #5)
