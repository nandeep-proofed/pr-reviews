# PR Review: feature/PP-1720: AI-generated feedback in Review (frontend)

**PR:** https://github.com/Proofed/B2BWebserver/pull/2291
**Jira:** https://proofed.atlassian.net/browse/PP-1720
**Status (Jira):** In Progress / Review (frontend-only, mock backend)
**Base branch:** `develop`
**Head SHA:** `cbf808ca40c48cf47cd804af327c15aba6cde106` (initial review) → `17a1d788776e2d10c139c81320e9395a26c6e8d0` (re-review, after PP-1843 merge)
**Size:** 47 files / +1,797 / −123 (initial) → **92 files / +7,590 / −305 (current, post-merge)**

> **Re-review summary (2026-05-25, head `17a1d78`):** PR #2298 (PP-1843 — real Proofed.ai backend integration) has been **merged into this PR** as merge commit `17a1d7887`. The merged code includes both PP-1843's hardening commits (`754326176` busboy/escape/word-count, `809c2a7e9` SanitizedHtml/double-click/blocklist/XSS-tests) that I reviewed previously in `PR-2298-PP-1843-proofed-ai-backend-integration.md`. This PR now contains the full PP-1720 + PP-1843 stack. Of the original 10 PR-2291 issues, **5 are fully resolved by the merge, 1 partially, 4 still apply**. The 7 of 9 PP-1843 issues that were fixed in #2298 carry forward; 2 PP-1843 issues remain deferred. **Recommendation upgraded to Approve.** See "Re-review — post PP-1843 merge" section at the bottom.

> **Companion PR (now merged in):** [#2298 — PP-1843](https://github.com/Proofed/B2BWebserver/pull/2298) — replaced `mockGenerateAiFeedback` with the real Proofed.ai backend integration + added 110 more tests. The merge brought in the panel-hook decomposition into 4 sub-hooks, the server-side word-count gate, the `SanitizedHtml` branded type, and the new `useAiFeedbackEligibility` service hook.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| **Default state** — Review Submission shows only Review Time + "No changes required" checkbox; Score/Feedback hidden | `state="hidden"` returns `null` for non-HTML reviews; checkbox added to both `ReviewSubmission` and `ReviewForm` | ✅ Addressed |
| **"No changes required" checked** — no file expected; Score pre-set to Excellent (1); Feedback optional with `"(optional)"` placeholder; no AI card | `useEffect` in both consumers sets `reviewScore=1` when checked; `feedbackPlaceholder` toggles `(optional)`; `showStandardForm = noChangesRequired \|\| isAiDisabled` skips AI path | ✅ Addressed |
| **New file + validation pass** — auto-trigger AI drafting; replace Score/Feedback with AI card | File-watch `useEffect` in `useAiFeedbackPanel` detects new `fileId`, runs size + word-count validation, calls `runAiDraft` | ✅ Addressed |
| **New file + validation fail** — show standard form + AI unavailable indicator | `setState("unavailable")`; consumer reads `isAiUnavailable` and renders `<AiUnavailableHeader />` (crossed sparkle + tooltip) | ✅ Addressed |
| **File size ≤ 25 MB** | `MAX_AI_FILE_BYTES = 25 * 1024 * 1024`; `validateAiFileSize(file)` short-circuits to `"unavailable"` | ✅ Addressed (client-side only — backend gate added in PP-1843) |
| **Word count ≤ 30,000** | `MAX_AI_WORD_COUNT = 30_000`; POST to existing `apiRoutes.wordCountsStream`; `validateAiWordCount` short-circuits | ✅ Addressed (DOCX only — HTML manual-trigger path skips it) |
| **States: Drafting / Timeout / Drafted / Edit / AI unavailable** | `AiPanelState` union has all six; each state has a dedicated partial (`DraftingState`, `TimeoutState`, `DraftedState`, `AiUnavailableHeader`) | ✅ Addressed |
| **240s timeout** — message changes, X cancel icon appears top-right | `AI_TIMEOUT_MS = 240_000`; `setTimeout` switches state to `"timeout"`; `TimeoutState` renders the X | ✅ Addressed |
| **Cancel (X)** — abort request; dismiss card; show standard form | `onCancel` aborts AbortController, restores `preDraftSnapshotRef`, sets state to `"hidden"` | ✅ Addressed |
| **Retry on new file** — auto-replace existing draft; single undoable transaction | File-watch effect re-runs; per the hook comment, Tiptap `setContent(...)` is one transaction = one undo step | ⚠️ Partial — comment claims this; not directly tested |
| **Only one generation per review** | `cancelInFlight()` called before each new run | ✅ Addressed |
| **Navigate away cancels** | Unmount cleanup `useEffect(() => () => cancelInFlight(), [cancelInFlight])` | ✅ Addressed |
| **Org gate (`aiEnabled` / `disableAi` = false)** | `useReviewAiDisabled(organizationGroupId)` reads `disableAi` from `useOrganisationGroupById`; both consumers gate render on `isAiDisabled` | ✅ Addressed |
| **HTML order — placeholder "Trigger AI" button** | `isHtmlReview = workItemFormat === "JSON"` branches to a button-only render; AC says "Design TBD (use placeholder for now)" — matches | ✅ Addressed |
| **AI HTML sanitized before reaching Formik/Tiptap** | `sanitizeHtml` called in `runAiDraft` before `setFieldValue` AND again in `DraftedState` `dangerouslySetInnerHTML` (defense-in-depth) | ✅ Addressed |
| **No secret leak via Sentry** | `captureAiError` uses `captureMessage` with only `{ status }` in `extra` — `error.config.data`, `error.message`, `error.stack` not forwarded | ✅ Addressed |
| **`noChangesRequired` not sent to backend** | `FormModal/hooks.ts` destructures `noChangesRequired` out of the payload; `Submission/hooks.ts` constructs payload by hand without it | ✅ Addressed |

**Beyond Jira scope:** Sentry scrub (security hardening from `/security` review), `?aiTimeout=long`/`?aiDelay=N` dev-only query flags (testing aid), `DotsLoader` shared atom, defense-in-depth double-sanitization. All reasonable.

**Cancel-on-navigate / one-request-at-a-time / cancel-restores-form** are flagged by Jira's own checklist as "not yet covered" — the implementation handles them, but tests are missing (see Issues #5 and Tests section below).

---

## Architecture Analysis

New `AiFeedbackPanel` organism is a clean, single-orchestrator pattern: one hook (`useAiFeedbackPanel`) owns the state machine, abort lifecycle, timeout, word-count network call, sanitization, snapshot/restore, and Formik wiring. Three small partials render each visible state. Two sidebar consumers (`ReviewSubmission` for the job sidebar, `ReviewForm` for the order modal) mount the panel and consume `isAiBusy`/`isAiUnavailable`/`isAiActive` callbacks to coordinate the surrounding form.

The mock (`mockGenerateAiFeedback`) is deliberately disposable — explicit TODO comment, 5-min default delay so reviewers can manually inspect every state in one run. PP-1843 (PR #2298) replaces it with the real Proofed.ai mutation.

Two structural concerns the CLAUDE.md convention check would flag:

1. **`index.tsx` is supposed to be UI-only** per the project convention ("Local component state, callbacks, and effects belong in a sibling `hooks.ts` exporting a single `use<ComponentName>` hook"). Both `ReviewSubmission/index.tsx` and `ReviewForm/index.tsx` now host `useState`, `useEffect`, and Formik-derived logic directly in `index.tsx`.
2. **Significant duplication between the two consumers.** Both compute `uploadedFile = cleanCopy ?? editedCopy ?? trackChanges`, `isExcellentScore`, `feedbackPlaceholder`, and run the same "force `reviewScore=1` on `noChangesRequired`" effect. Extracting `useReviewSubmissionFormState({ isReviewerFlow })` into a shared hook would remove ~40 lines and centralise the score-sync rule.

Note that the companion PR PP-1843 decomposes `useAiFeedbackPanel` into 4 sub-hooks (`useActiveUuidLifecycle`, `useAutoTriggerOnFileUpload`, `useFormikSnapshot`, plus the orchestrator), so the orchestrator-test-gap I flag below is materially mitigated in #2298 — but as PP-1720 stands today, the 288-LOC hook has no direct tests.

Security profile is strong: client-side `sanitizeHtml` at two boundaries, Sentry scrub verified (only the HTTP status code escapes), `?aiTimeout=long`/`?aiDelay=N` strictly gated on `NODE_ENV !== "production"` and strict-equality compared (never rendered), `noChangesRequired` stripped from backend payloads.

---

## Issues Found

### 1. `useAiFeedbackPanel` (288 LOC) has no direct tests — ✅ Resolved by PP-1843 merge

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/hooks/useAiFeedbackPanel.ts]**
**Function/Class:** `useAiFeedbackPanel`
**Severity:** medium
**Problem:** The orchestrator hook owns the entire state machine, abort lifecycle, word-count network call, AI dispatch, sanitization, snapshot/restore, and three Formik effects. None of these branches has a direct unit test — `AiFeedbackPanel/index.test.tsx` mocks the hook entirely (`(useAiFeedbackPanel as Mock).mockReturnValue(...)`). The branches without coverage include: new-file detection via `fileId`, size-fail → `unavailable`, word-count network call + fail → `unavailable`, abort on file change, abort on unmount, `?aiTimeout=long` wiring, pre-draft snapshot capture and restore on cancel, Sentry error capture path.
**Impact:** AC §2 ("validation runs each time a new file is uploaded"), §4.5 (cancel restores standard form), §6 ("only one generation may run at a time"), §7 (retry-after-edit), §8.2 (navigate-away cancels), and §9 (org gate) all live in this hook and are functionally untested. A regression in `cancelInFlight` (e.g. someone removes the `controller.abort()`) wouldn't be caught.
**Fix:** Add `useAiFeedbackPanel.test.tsx` using `renderHook` from `@testing-library/react` with a Formik wrapper. Cover at minimum: new-file → drafting, size-fail → unavailable, word-count-fail → unavailable, cancel restores snapshot, unmount aborts, `?aiTimeout=long` toggles timeout state, Sentry status forwarded. Note: PP-1843 (PR #2298) decomposes this hook into 4 sub-hooks each with their own tests (23 cases total), so the gap is largely closed downstream. Still worth a stub here so PP-1720 doesn't merge without orchestrator coverage.

### 2. `index.tsx` violates the project's UI-only convention; significant duplication between `ReviewSubmission` and `ReviewForm` — ⚠️ Partially mitigated (still applies)

**[File: apps/creative-portal/components/organisms/sidebars/contents/JobManagement/partials/ReviewSubmission/index.tsx, apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderJobs/partials/FormModal/partials/ReviewForm/index.tsx]**
**Function/Class:** both `ReviewSubmission` and `ReviewForm` top-level components
**Severity:** medium
**Problem:** CLAUDE.md mandates that `index.tsx` is UI-only — local state/callbacks/effects belong in a sibling `hooks.ts` exporting `use<ComponentName>`. Both consumer files now host three `useState` calls (`isAiBusy`, `isAiUnavailable`, `isAiActive`), a `useReviewAiDisabled` call, a Formik-context `useEffect` to sync `reviewScore=1`, derivations of `uploadedFile = cleanCopy ?? editedCopy ?? trackChanges`, `isExcellentScore`, `feedbackPlaceholder`. The two files are ~40 lines duplicated; the only meaningful difference is `ReviewForm`'s `isReviewerFlow` guard.
**Impact:** Future maintenance has to be done twice; drift between the two surfaces is now easy. A bug fix to the score-sync effect would need to be replicated. CLAUDE.md reference for the convention: `apps/creative-portal/components/molecules/SearchBar/`, `OrderStatusCardInfo/`, `tables/HistoryTable/`.
**Fix:** Extract `useReviewSubmissionFormState({ isReviewerFlow })` into a shared hook file (`apps/creative-portal/components/organisms/sidebars/.../hooks/useReviewSubmissionFormState.ts`) that owns the three `useState`s, the score-sync effect, `useReviewAiDisabled`, and the derived values. Both consumers then destructure the hook and stay UI-only:

```typescript
const {
  isAiBusy, isAiUnavailable, isAiActive,
  isAiDisabled, showStandardForm,
  uploadedFile, isExcellentScore, feedbackPlaceholder,
  onAiBusyChange, onAiUnavailableChange, onAiActiveChange
} = useReviewSubmissionFormState({ isReviewerFlow });
```

### 3. `workItemFormat === "JSON"` is a stringly-typed comparison; `=== 1` excellent-score sentinel duplicated 5 places — ❌ Still applies (verified on remote head `17a1d78`)

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/index.tsx + 4 others]**
**Function/Class:** `AiFeedbackPanel` + consumer components + `FormModal/hooks.ts` + `Submission/hooks.ts`
**Severity:** medium
**Problem:** `isHtmlReview = workItemFormat === "JSON"` (`AiFeedbackPanel/index.tsx:33`) — the file imports `WORK_ITEM_FORMAT` enum elsewhere (`WORK_ITEM_FORMAT.DOCX` in the hook) but uses the raw string here. Separately, the excellent-score sentinel `=== 1` is duplicated in 5 files (`AiFeedbackPanel/index.tsx`, `ReviewSubmission/index.tsx`, `ReviewForm/index.tsx`, `Submission/hooks.ts`, `FormModal/hooks.ts`).
**Impact:** Stringly-typed comparison is brittle to upstream enum changes (e.g. lowercase migration). Magic-number `1` requires a reader to trust that `1` always means "Excellent" — if the score scale ever changes, you'd touch all 5 files.
**Fix:** Use the enum + extract a constant:

```typescript
import { WORK_ITEM_FORMAT } from "@proofed/shared/api/workItemContentVersion/enums";

const isHtmlReview = workItemFormat === WORK_ITEM_FORMAT.JSON;
```

```typescript
// utils/reviewScoreOptions.ts — co-locate with REVIEW_SCORE_OPTIONS
export const REVIEW_SCORE_EXCELLENT = 1;
```

### 4. `useReviewAiDisabled` has no test file — AC §9's only enforcement — ❌ Still applies (moved to `services/aiReviewFeedback/`, still untested)

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/hooks/useReviewAiDisabled.ts]**
**Function/Class:** `useReviewAiDisabled`
**Severity:** medium
**Problem:** This 14-line hook is the sole enforcement of "Do not run the integration when the Parent Org Group of the order has `aiEnabled = False`" (Jira AC §9). No test file. Both consumer test files (`ReviewSubmission/index.test.tsx`, `ReviewForm/index.test.tsx`) mock `useReviewAiDisabled` to return `{ isAiDisabled: false }` and never exercise the `true` branch.
**Impact:** A regression that flips the return value to always-false (e.g. someone changes `!!data?.disableAi` to `false` while debugging) would silently break the org-level kill switch.
**Fix:** Add `useReviewAiDisabled.test.ts` covering: undefined org-group ID → `enabled: false` and `isAiDisabled: false`; valid ID + `disableAi: true` → `isAiDisabled: true`; valid ID + `disableAi: false` → `isAiDisabled: false`; valid ID + null/missing data → `isAiDisabled: false`. Note: in PP-1843, this hook moves to `services/aiReviewFeedback/useReviewAiDisabled.ts` — still untested there.

### 5. Pre-draft snapshot semantics surprise on retry-after-edit — ✅ Resolved by PP-1843 merge (`useFormikSnapshot` sub-hook + 3 tests)

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/hooks/useAiFeedbackPanel.ts]**
**Function/Class:** `useAiFeedbackPanel` (the file-watch effect + `triggerAi`)
**Severity:** low
**Problem:** `preDraftSnapshotRef.current` is overwritten on every retry (`runAiDraft` invocation). If the user (a) uploads a file, (b) waits for the AI to draft, (c) edits the draft, (d) uploads a new file (triggers retry), then (e) cancels the retry — the cancel-restore puts back the **edited draft**, not the original blank field that existed before step (a). AC §4.5 says "the standard feedback form must be displayed" on cancel; the wording doesn't strictly require restoring to pre-first-run state, but the snapshot semantics may surprise reviewers who expect "cancel = undo everything AI did".
**Impact:** Minor UX surprise. Not a data-loss issue (the edit is restored, not destroyed). Worth confirming with PM whether the AC author intended "cancel restores the truly-pre-AI state" or "cancel restores whatever was there immediately before this retry".
**Fix:** Either (a) document the chosen semantic clearly in a code comment, (b) only capture the snapshot on the FIRST run (skip overwrite if `preDraftSnapshotRef.current !== null && state !== "drafted"`), or (c) capture two snapshots — first-run + per-retry — and offer the user a "Undo all AI" affordance. Option (a) is the cheapest if PM confirms current behavior is intended.

### 6. `triggerAi` (HTML manual path) skips word-count validation — ✅ Resolved by PP-1843 merge (server-side word-count gate via `order.workItemSize` in `754326176`)

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/hooks/useAiFeedbackPanel.ts]**
**Function/Class:** `triggerAi`
**Severity:** low
**Problem:** The DOCX file-watch path runs `validateAiWordCount(data.workItemSize)` after the `wordCountsStream` POST. The HTML manual-trigger path calls `runAiDraft` directly. HTML reviews could exceed 30K words and the user would hit a long drafting state with no "AI unavailable" affordance.
**Impact:** HTML reviewers on large documents waste time on a request that may fail downstream. AC §2 says "validation must be evaluated each time a new file is uploaded" — HTML has no file, so the AC is technically silent. But the 30K word limit is presumably a content limit, not a file-size proxy.
**Fix:** Either (a) accept this as an HTML-flow-only limitation and document it, or (b) move the word-count gate into `triggerAi` with an HTML-equivalent source. Note: PP-1843 adds a server-side word-count gate using `order.workItemSize` that covers both paths.

### 7. CLAUDE.md violations — inline style + empty interface + button copy not in `consts.ts` — ✅ Resolved by PP-1843 merge (`GenerateFeedbackCard` replaces inline-styled placeholder button; copy now lives in card consts)

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/index.tsx, AiUnavailableHeader/types.ts]**
**Function/Class:** `AiFeedbackPanel` HTML-trigger branch, `AiUnavailableHeaderProps`
**Severity:** low
**Problem:** Three small lint-style issues:
- `<span style={{ alignSelf: "flex-start" }}>` on the HTML trigger button — CLAUDE.md mandates styled-components co-location.
- `"Trigger AI"` button copy hardcoded in JSX — should live in `consts.ts` like the other strings (`AI_DRAFTING_MESSAGE`, `AI_TIMEOUT_MESSAGE`, etc.).
- `AiUnavailableHeader/types.ts` exports an empty `interface AiUnavailableHeaderProps {}` — ESLint `@typescript-eslint/no-empty-interface` usually forbids this.

**Impact:** None functionally. Out-of-pattern code is the kind of thing that compounds; the styled-component for the trigger-button container would take ~3 lines.
**Fix:**

```typescript
// styles.ts
export const StyledAiTriggerButtonWrapper = styled.span`
  align-self: flex-start;
`;

// consts.ts
export const AI_TRIGGER_BUTTON_LABEL = "Trigger AI"; // TBD per Figma

// AiUnavailableHeader/types.ts — delete the file or:
export type AiUnavailableHeaderProps = Record<string, never>;
```

### 8. `runAiDraft` is recreated every render (no `useCallback`) — ✅ Resolved by PP-1843 merge (sub-hooks adopt ref pattern; per PP-1843 commit `b7f86c2c0`, last `exhaustive-deps` disable removed)

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/hooks/useAiFeedbackPanel.ts]**
**Function/Class:** `runAiDraft`
**Severity:** low
**Problem:** `runAiDraft` is a plain `async` function inside the hook body — not memoised. Both `triggerAi` (memoised) and the file-watch effect (deps `[fileId, noChangesRequired, workItemFormat]`) call it via closure capture. The hook author's intent (per the two `react-hooks/exhaustive-deps` disables) is to capture `values`/`setFieldValue`/`feedbackFieldName`/`scoreFieldName` lazily on each invocation, NOT to re-fire on every Formik keystroke. That intent is correct, but the chosen mechanism — letting the function be re-created each render while the call-sites are memoised against stale closures — means `triggerAi` clicked after a render captures stale values from a previous render.
**Impact:** Subtle. In practice the field names are static literals and `setFieldValue` is stable, so the staleness doesn't bite. But if anyone adds a dynamic dep, it'd silently desync.
**Fix:** Either move the captured-via-closure values into refs (`valuesRef.current = values` in a layout effect, then read `valuesRef.current` inside `runAiDraft`) for explicit "always-fresh" semantics, or accept the closure and add a comment explaining that field-name props must be static. Option 1 is preferable — it's also the pattern PP-1843 adopts in `useActiveUuidLifecycle`/`useAutoTriggerOnFileUpload`.

### 9. `useReviewAiDisabled(0)` magic-number fallback — ❌ Still applies (file moved to `services/aiReviewFeedback/`, but `?? 0` and `!!organizationGroupId` gate unchanged)

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/hooks/useReviewAiDisabled.ts]**
**Function/Class:** `useReviewAiDisabled`
**Severity:** low
**Problem:** `useOrganisationGroupById(organizationGroupId ?? 0, { enabled: !!organizationGroupId })` passes `0` as a sentinel when the ID is undefined. Safe because `enabled: false` prevents the fetch, but a magic-number fallback in a primary key field is an anti-pattern.
**Impact:** None — the fetch is gated. But a future caller dropping the `enabled` gate would silently send `GET /organisationGroups/0`.
**Fix:**

```typescript
// Prefer a sentinel that can't be a real ID, or restructure so the
// caller short-circuits before reaching this hook:
export const useReviewAiDisabled = (
  organizationGroupId: number | undefined
) => {
  const enabled = typeof organizationGroupId === "number" && organizationGroupId > 0;
  const { data } = useOrganisationGroupById(
    organizationGroupId ?? -1,
    { enabled }
  );
  return { isAiDisabled: !!data?.disableAi };
};
```

### 10. Sentry capture loses stack trace (intentional but worth noting) — ⏸ Unchanged (same scrub philosophy now applied across PP-1843 server/client paths)

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/utils.ts]**
**Function/Class:** `captureAiError`
**Severity:** low
**Problem:** The PR description explains: `Sentry.captureException` was replaced with `Sentry.captureMessage` because `error.config.data` (the multipart `FormData` containing the reviewer's filename) was leaking via axios error serialization. The chosen scrub keeps only `{ status }`, which prevents the leak but means Sentry won't have a stack trace or error message — only the HTTP status code.
**Impact:** Real backend failures will produce shallow Sentry events. Debugging an intermittent 500 from `wordCountsStream` would require reproducing locally. Acceptable security trade-off for now.
**Fix:** Long-term: add a `beforeSend` Sentry hook per-feature that scrubs `error.config.data` and `error.message` but preserves the stack and error type. Today: accept the trade-off.

---

## Tests

- ✅ `DotsLoader` atom: 3 cases (role, aria-label, custom-label) in `DotsLoader.test.tsx`
- ✅ Mock generator: 4 cases (resolve, pre-abort, post-abort, forceTimeout) in `mock.test.ts`
- ✅ Validators: 6 cases (size/word-count thresholds) in `utils.test.ts`
- ✅ Panel UI: ~9 render-by-state cases in `AiFeedbackPanel/index.test.tsx` — but hook is fully mocked
- ✅ `ReviewSubmission`: default + no-changes-required paths in `ReviewSubmission/index.test.tsx`
- ✅ `ReviewForm`: default + no-changes-required + QA + edit-mode legacy paths
- ❌ `useAiFeedbackPanel` orchestrator (288 LOC) — no direct test (Issue #1)
- ❌ `useReviewAiDisabled` — no test file (Issue #4)
- ❌ `showStandardForm` true via `isAiDisabled === true` branch — both consumer tests fix `isAiDisabled: false`
- ❌ `noChangesRequired` payload stripping in `FormModal/hooks.ts` — no assertion that the field never reaches the backend
- ❌ Pre-draft snapshot capture + restore on cancel
- ❌ AbortController unmount lifecycle
- ❌ `?aiTimeout=long` / `?aiDelay=N` query-flag wiring (dev-only, low priority)
- ❌ Sentry capture shape — no assertion that `error.config.data` is NOT in the captured `extra`
- ⚠️ Manual test plan present in PR description; covers every visible state, plus 25MB + 30K-word affordance and navigate-away
- ⚠️ Design-fidelity walkthrough still pending per PR checklist ("UI elements match designs — pending")

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ — all Jira ACs addressed; state machine complete |
| Regression risk | ⚠️ Medium — large new organism + invasive sidebar consumer changes; orchestrator hook untested directly |
| Tests | ⚠️ Strong on partials/utils/mock; gaps on orchestrator hook, org-gate hook, payload stripping |
| Code quality | ⚠️ Some CLAUDE.md violations (index.tsx housing state, inline style, duplication, stringly-typed `"JSON"`) |
| Mergeable state | ✅ Clean |
| Security | ✅ Sanitization at two boundaries, Sentry scrub verified, dev-flag gating tight, payload stripping confirmed |

---

## Recommendation

**Approve with suggestions.**

The implementation faithfully covers every Jira acceptance criterion and demonstrates solid security discipline (the `/security` review fixes for HTML sanitization and Sentry scrubbing are real and properly applied). The state machine is well-scoped, the abort/timeout lifecycle is correct, and the mock is clearly disposable.

Before merging (or as part of the PP-1843 stack landing):

1. **Add `useAiFeedbackPanel.test.tsx`** (Issue #1) — the orchestrator hook is the largest untested unit. PP-1843 decomposes it into 4 sub-hooks each with direct tests, so this is largely closed downstream — but worth a stub here.
2. **Add `useReviewAiDisabled.test.ts`** (Issue #4) — small file, but it's AC §9's only enforcement.
3. **Use `WORK_ITEM_FORMAT.JSON` instead of the `"JSON"` literal** (Issue #3) — one-line fix in `AiFeedbackPanel/index.tsx`.

Suggested follow-ups (non-blocking, candidates for `simplify`):

4. Extract `useReviewSubmissionFormState({ isReviewerFlow })` to remove ~40 lines of duplication between `ReviewSubmission` and `ReviewForm` (Issue #2).
5. Extract `REVIEW_SCORE_EXCELLENT = 1` constant — 5 magic-number call sites.
6. Resolve the inline style + empty interface + hardcoded `"Trigger AI"` copy (Issue #7).
7. Confirm cancel-restore semantics on retry-after-edit with PM (Issue #5) — document the chosen behavior.

Out of scope for this PR (handled in PP-1843):

- HTML word-count gate on the manual-trigger path (server-side gate added in PP-1843).
- Real backend integration; mock removal.
- Server-side validation mirroring the 25MB / 30K-word client gates.

---

## Re-review — post PP-1843 merge (head `17a1d78`, 2026-05-25)

PR #2298 (PP-1843 — the real Proofed.ai backend integration) was merged into this branch as merge commit `17a1d7887 feature/PP-1843: Proofed.ai backend integration for AI review feedback (#2298)`. This PR now contains the entire PP-1720 + PP-1843 stack. Both PP-1843 follow-up hardening commits (`754326176` server hardening, `809c2a7e9` quick wins) are reachable from the new head. Verification was done against the remote ref `refs/pull/2291/head` via the GitHub MCP, not against local working state.

### Original PR-2291 issues — resolution status

| # | Issue | Status | Notes |
|---|---|---|---|
| 1 | `useAiFeedbackPanel` 288 LOC untested | ✅ Resolved | Merge decomposed the hook into 4 sub-hooks each with dedicated tests: `useAiFeedbackPanel.test.tsx` (9 cases, 11.6 kB), `useActiveUuidLifecycle.test.tsx` (4 cases, 4.5 kB), `useAutoTriggerOnFileUpload.test.tsx` (7 cases, 3.7 kB), `useFormikSnapshot.test.tsx` (3 cases, 2.6 kB). |
| 2 | `index.tsx` housing state + consumer duplication | ⚠️ Partial | PP-1843's `useAiFeedbackEligibility` collapses three derivations into one destructure in both consumers — meaningful simplification. But: `ReviewSubmission/index.tsx` (verified on remote) still hosts three `useState` calls (`isAiBusy`, `isAiUnavailable`, `isAiActive`), a Formik-context `useEffect`, and derived computations directly in `index.tsx`. CLAUDE.md UI-only convention still violated; the structural duplication between `ReviewSubmission` and `ReviewForm` is reduced but not eliminated. |
| 3 | `workItemFormat === "JSON"` stringly-typed; `=== 1` magic-number | ❌ Still applies | Verified on remote: `AiFeedbackPanel/index.tsx:31` still has `const isHtmlReview = workItemFormat === "JSON";`. `WORK_ITEM_FORMAT.JSON` is now imported into `useAiFeedbackEligibility` (good) but not used in this call site. `isExcellentScore = values[scoreFieldName] === 1` also unchanged. |
| 4 | `useReviewAiDisabled` no test file | ❌ Still applies | File moved to `services/aiReviewFeedback/useReviewAiDisabled.ts` but **no test file** exists in the directory listing. Still AC §9's only enforcement. |
| 5 | Pre-draft snapshot semantics fragile | ✅ Resolved | PP-1843 extracted snapshot/restore into `useFormikSnapshot` sub-hook with 3 dedicated tests covering capture, restore-on-cancel, and overwrite-on-retry semantics. |
| 6 | HTML manual-trigger skips word-count | ✅ Resolved | PP-1843 commit `754326176` added a server-side word-count gate using `order.workItemSize` that returns 413 before `provider.submit`. Covers both DOCX and HTML paths uniformly. |
| 7 | Inline style + hardcoded "Trigger AI" + empty interface | ✅ Resolved | Verified on remote: `<span style={{ alignSelf: "flex-start" }}>` is gone, replaced by `<GenerateFeedbackCard onTrigger={triggerAi} isSubmitting={isSubmitting} />`. Card copy lives in `GenerateFeedbackCard/index.tsx`. |
| 8 | `runAiDraft` not memoised; closure staleness | ✅ Resolved | Per PP-1843 commit `b7f86c2c0`, sub-hooks "adopt the ref pattern… to remove the last `react-hooks/exhaustive-deps` disable in the panel". Closure-staleness concern designed out. |
| 9 | `useReviewAiDisabled(0)` magic-number fallback | ❌ Still applies | Verified on remote: hook moved to `services/aiReviewFeedback/useReviewAiDisabled.ts` but body unchanged — `organizationGroupId ?? 0, { enabled: !!organizationGroupId }`. Gated, but the anti-pattern persists. |
| 10 | Sentry capture loses stack trace | ⏸ Unchanged | Scrub philosophy ported into PP-1843's server-side `captureMessage` calls and `captureAiError`. Long-term `beforeSend` scrubber still a future improvement, not a regression. |

**Score:** 5 fully resolved, 1 partially resolved, 4 still apply (all low-to-medium severity). The remaining items are code-quality follow-ups, not correctness or security issues.

### Inherited PP-1843 issue status

The PR-2298 review (`PR-2298-PP-1843-proofed-ai-backend-integration.md`) flagged 9 issues; 7 were fixed in `754326176` + `809c2a7e9` before the merge. Carrying forward:

| PP-1843 # | Issue | Status |
|---|---|---|
| 1 | bb.on(finish)/bb.on(error) double-response race | ✅ Fixed in `754326176` |
| 2 | Unescaped text blocks in `blockFormatFileToHtml` | ✅ Fixed in `754326176` |
| 3 | `DraftedState` trust-contract fragility | ✅ Fixed in `809c2a7e9` (SanitizedHtml brand) |
| 4 | `BLOCKED_FORMATS` consistency | ✅ Fixed in `809c2a7e9` (shared `GOOGLE_DRIVE_FILE_FORMATS`) |
| 5 | `triggerAi` no double-click guard | ✅ Fixed in `809c2a7e9` |
| 6 | Cancel + getStatus handler-shell integration tests | ⏸ Still deferred |
| 7 | Server-side word-count gate | ✅ Fixed in `754326176` |
| 8 | `useAiFeedbackEligibility` test file | ⏸ Still deferred |
| 9 | XSS regression test for `marked` path | ✅ Fixed in `809c2a7e9` |

Two PP-1843 items remain deferred per the PR description's "Open follow-ups" section: cancel/getStatus handler-shell tests (low-severity test-coverage gap), `useAiFeedbackEligibility` test file (which would also address PR-2291 Issue #4 if added).

### Updated summary

| Aspect | Initial review (head `cbf808c`) | Re-review (head `17a1d78`) |
|---|---|---|
| Correctness | ✅ All ACs addressed | ✅ ACs + real backend integration + server-trusted gates |
| Regression risk | ⚠️ Medium — orchestrator untested | ✅ Low — orchestrator decomposed + tested; backend gates server-trusted |
| Tests | ⚠️ Hook layer untested | ✅ Strong — ~140 tests across hooks, services, strategy, handlers, UI |
| Code quality | ⚠️ CLAUDE.md violations + duplication + stringly-typed | ⚠️ Improved but: index.tsx still holds state; `"JSON"` literal + magic-number `1` persist |
| Mergeable state | ✅ Clean | ✅ Clean |
| Security | ✅ Sanitize + Sentry scrub + dev-flag gating | ✅ + server-side word-count + race-safe response + SanitizedHtml brand + auth fail-closed |

### Updated recommendation

**Approve.** _(Upgraded from "Approve with suggestions" — the PP-1843 merge resolved 5 of 10 PR-2291 findings and brought in 7 hardening fixes from its own review.)_

Suggested follow-up tickets (none blocking):

1. **`useReviewAiDisabled` test file** (Issue #4) + **`useAiFeedbackEligibility` test file** (PP-1843 #8) — together these would cover the org-gate kill switch + the composite eligibility computation. Both are small files; one PR could land both.
2. **Use `WORK_ITEM_FORMAT.JSON` constant** at `AiFeedbackPanel/index.tsx:31` (Issue #3, one-line change) — and extract `REVIEW_SCORE_EXCELLENT = 1` alongside `REVIEW_SCORE_OPTIONS`.
3. **Extract `useReviewSubmissionFormState({ isReviewerFlow })`** (Issue #2) to remove the residual state/effect duplication between `ReviewSubmission` and `ReviewForm` and restore the CLAUDE.md "index.tsx is UI-only" convention.
4. **Tighten `useReviewAiDisabled(?? 0)` fallback** (Issue #9) — small ergonomic improvement.
5. **Cancel + getStatus handler-shell integration tests** (PP-1843 #6) — already disclosed as a deferred follow-up in PR #2298's description.
