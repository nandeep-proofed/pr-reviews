# PR Review: feature/PP-1843: Proofed.ai backend integration for AI review feedback

**PR:** https://github.com/Proofed/B2BWebserver/pull/2298
**Jira:** https://proofed.atlassian.net/browse/PP-1843
**Status (Jira):** Code Review
**Base branch:** `feature/PP-1720-ai-feedback-review` (stacked PR — not `develop`)
**Head SHA:** `8902a2613eb98a255f3bfd95b9e5ef76bbb115e4`
**Size:** 67 files changed, +6,273 / −794, 43 commits, ~110 new tests

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| **1.1** Only run for non-PDF/XLSX/GDoc Review jobs | `AI_REVIEW_FEEDBACK_BLOCKED_FORMATS` Set (`PDF`, `XLSX`, `GDOC`, `GSLIDES`, `GSHEETS`) gated server-side in `submit/index.ts:148` against `order.workItemFormat` | ✅ Addressed |
| **1.2** Only run for Proofed-platform orders | `if (order.platformName !== PlatformType.Proofed)` 400 in `submit/index.ts:132` | ✅ Addressed |
| **1.3** Environment-level toggle | Service-metadata registry entry `aiReviewFeedback: "PROOFEDAISERVICE-FE"` in `b2bRoutesMap`; `b2bRoutes.PROOFEDAI` env gate; `useReviewAiDisabled` client hook | ✅ Addressed |
| **1.4** Editor + Reviewer sourced from `workItemContentVersions` | `processFileReviewSubmission` fetches via `fetchWorkItemContentVersions` by SERVICE jobId; reviewer file uploaded via multipart, not converted server-side | ✅ Addressed |
| **1.5** Skip if >25MB or >30K words | 25MB via busboy `fileSize` + `limit` → 413; 30K words via `MAX_AI_REVIEW_FEEDBACK_WORD_COUNT` in `useAiFeedbackEligibility` (client gate) | ⚠️ Partial — word-count gate is client-side only. Server doesn't validate word count on submit; a crafted request could bypass it. The PR description mentions "document > 30k words" as a test step but the gate is purely in the eligibility hook. |
| **2.1** Bearer token in service-metadata registry | `getServiceMetadata("aiReviewFeedback", false)` resolved per-request in `createAiReviewFeedbackProvider` | ✅ Addressed |
| **2.2** Token resolved server-side at request time | Confirmed — provider factory does the lookup; no module-level constant | ✅ Addressed |
| **2.3** Token never sent to browser | Bearer attached server-side only in `AiReviewFeedbackProvider.submit/getStatus/cancel`; not in any browser-facing response | ✅ Addressed |
| **2.4** All Proofed.ai calls server-side | Three `pages/api/aiReviewFeedback/{submit,status,cancel}.ts` wrappers; client only calls portal endpoints | ✅ Addressed |
| **2.5** `withApiMiddleware` with ironSession/auth/maintenance | All three handlers wrap with `{ ironSession: true, authNeeded: true, maintenanceMode: true }` | ✅ Addressed |
| **2.6** Reviewer-to-job assignment check | `verifyReviewerJobAssignment` (new shared helper) called on submit, getStatus, cancel; `isAdministrative(roles)` bypass | ✅ Addressed |
| **2.7** Per-UUID poll state on iron-session | `IronSessionData.aiReviewFeedbackPolls: Record<uuid, {jobId, orderId, createdAt}>` with TTL sweep on submit | ✅ Addressed |
| **2.8** Token rotation via service-metadata update | Inherited from `getServiceMetadata` pattern | ✅ Addressed |
| **3.1** Browser calls submit proxy with content-version refs | `useAiFeedbackPanel` + service-layer `useSubmitAiReviewFeedbackMutation` post multipart FormData; server resolves the editor version | ✅ Addressed |
| **3.2** Webserver retrieves versions and forwards with Bearer | `processFileReviewSubmission` / `processJsonReviewSubmission` + `provider.submit` with multipart stream | ✅ Addressed |
| **3.3** Forwarded request includes order_id + job_id | Form fields in multipart body; logged | ✅ Addressed |
| **3.4** UUID returned to browser, held for page lifetime | 201 response carries `uuid`; held in `useActiveUuidLifecycle` via `activeUuidRef` | ✅ Addressed |
| **4.1** Poll until terminal | `useGetAiReviewFeedbackStatusQuery` with `refetchInterval` driven by server `pollIntervalMs` (3000ms) | ✅ Addressed |
| **4.2** Stop polling on navigate-away | Unmount path in `useActiveUuidLifecycle` fires `sendBeacon` cancel + clears query | ✅ Addressed |
| **4.3** `summary` populates Feedback field | `useAiFeedbackPanel` status effect calls `setFieldValue(feedbackKey, sanitizeHtml(summary))` on `completed` | ✅ Addressed |
| **4.4** `failed` → fallback to manual form | Drives `hasFailed=true` → `state="hidden"` → standard form renders | ✅ Addressed |
| **5.1.1** X cancel → cancel API | `cancelInFlightAndClear` calls `cancel.mutate(uuid)` | ✅ Addressed |
| **5.1.2** Navigate-away → cancel | `fireBeaconCancel` on unmount uses `navigator.sendBeacon` with `fetch` fallback | ✅ Addressed |
| **5.2** Ignore poll responses for cancelled UUID | `activeUuidRef` gate + react-query removal of the query on cancel.onSuccess | ✅ Addressed |
| **6.1** Non-2xx upstream → graceful fallback | `buildUpstreamApiError` → standardized error → client `hasFailed` branch | ✅ Addressed |
| **6.2** No retry | Confirmed — no retry logic in provider or hooks | ✅ Addressed |

**Beyond Jira scope (called out in PR):** the HTML/JSON branch (`processJsonReviewSubmission`) for non-file orders, `formatAiSummaryAsHtml`/marked-based renderer for structured findings, defense-in-depth XSS sanitization, in-flight upstream cancel on file swap, busboy `limit`-event handler. These are listed in the PR description as security/correctness hardening prompted by review. All look reasonable; none extend scope unfairly.

---

## Architecture Analysis

The implementation faithfully mirrors the OCR integration pattern Jira asked for: shared handlers + strategy class + thin route wrappers + service-metadata registry + ironSession per-poll state. Good structural fit.

The panel hook decomposition (`useAiFeedbackPanel` orchestrator + `useActiveUuidLifecycle` + `useAutoTriggerOnFileUpload` + `useFormikSnapshot`) is a sound refactor — the previous monolithic hook would have been hard to test. Each sub-hook has its own test file (4+7+3+9 = 23 cases).

XSS defense is **defense-in-depth** as advertised: server escapes raw fields → `marked.parse` markdown → client `sanitizeHtml` at the trust boundary in `useAiFeedbackPanel`'s `draft` useMemo. The `DraftedState` component then renders the sanitized HTML via `dangerouslySetInnerHTML` and explicitly does *not* re-sanitize (documented in a comment). The trust-boundary contract is correct but fragile (see issue #3 below).

`workItemFormat` source-of-truth split is well done — server reads `order.workItemFormat`; client form field is silently ignored. The PR description correctly cites this as a security review fix.

Iron-session sweep is bounded and self-healing — 1hr TTL, well past `AI_TIMEOUT_MS` (4 min), so live submissions are never affected.

The PR is stacked on `feature/PP-1720-ai-feedback-review`. **Merge target should remain PP-1720, not develop** — the PP-1720 PR will fold this work in when it lands. Reviewers should verify the base branch state on the GitHub UI before approving.

---

## Issues Found

### 1. Double-response race between `bb.on("finish")` and `bb.on("error")` after `fileStream.on("limit")`

**[File: apps/creative-portal/api/aiReviewFeedback/apiHandlers/submit/index.ts]**
**Function/Class:** `getAiReviewFeedbackSubmitHandler`
**Severity:** medium
**Problem:** When the reviewer uploads a file larger than 25 MB, the `fileStream.on("limit")` listener calls `bb.emit("error", new ApiError(413, ...))`. The `bb.on("error")` handler then writes a 413 response. However, busboy does not abort the `finish` event after an error emitted on its own instance — `bb.on("finish")` still fires once parsing completes (with the truncated bytes). The `finish` handler awaits `processedFilePromise` (which resolves on the truncated stream), runs validation/submission, and may call `res.status(201).json(...)` — at that point headers are already sent, throwing `ERR_HTTP_HEADERS_SENT` in Node and a 500 in the access log.
**Impact:** On a 25 MB+ upload, the reviewer correctly gets a 413, but the server logs a noisy `Cannot set headers after they are sent` (or, worse, a partial 201 race wins and the truncated upload silently completes and ships to Proofed.ai). Either way, the error path isn't clean. Low-probability in practice but worth a guard.
**Fix:** Track a sentinel and short-circuit either handler. The simplest fix:

```typescript
let responseAlreadySent = false;

bb.on("finish", async () => {
  if (responseAlreadySent) return;
  // ... existing body
});

bb.on("error", (err: Error) => {
  if (responseAlreadySent) return;
  responseAlreadySent = true;
  // ... existing body
});
```

Set `responseAlreadySent = true` immediately before each `res.status(...).json(...)` call (or simply check `res.headersSent` at the top of each handler).

### 2. `processJsonReviewSubmission` ships unescaped text blocks as HTML to Proofed.ai

**[File: apps/creative-portal/api/aiReviewFeedback/apiHandlers/submit/utils.ts]**
**Function/Class:** `blockFormatFileToHtml`
**Severity:** medium
**Problem:** For `type === "text"` blocks, the content is wrapped in `<p>${block.content}</p>` without HTML-escaping. If a reviewer/editor's document contains `<`, `>`, or `&` in a text block, the AI provider receives malformed HTML instead of escaped text. For example, a paragraph reading `2 < 3 and x > y` becomes `<p>2 < 3 and x > y</p>` — the upstream parser will treat `< 3 and x >` as a tag.
**Impact:** Not a security issue (the data flows out to a trusted upstream, not back into our DOM), but the AI sees corrupted/missing content and returns lower-quality feedback. Reviewer-side mathematical, code-flavored, or copy-editing-of-HTML documents would hit this most often.
**Fix:** Apply the same `escapeHtml` helper used in `getStatus/utils.ts`:

```typescript
const escapeHtml = (input: string): string =>
  input
    .replaceAll("&", "&amp;")
    .replaceAll("<", "&lt;")
    .replaceAll(">", "&gt;")
    .replaceAll('"', "&quot;")
    .replaceAll("'", "&#39;");

const blockFormatFileToHtml = (file: BlockFormatFile): string =>
  file.blocks
    .filter((block) => block.type === "html" || block.type === "text")
    .map((block) =>
      block.type === "text"
        ? `<p>${escapeHtml(block.content)}</p>`
        : block.content
    )
    .join("\n");
```

For `type === "html"` blocks, content is already HTML (from the Tiptap export) and shouldn't be re-escaped.

### 3. `DraftedState` trusts caller to pre-sanitize — no defense in depth at the render boundary

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/partials/AiFeedbackCard/partials/DraftedState/index.tsx]**
**Function/Class:** `DraftedState`
**Severity:** medium
**Problem:** `DraftedState` renders `draft.summary` and `draft.toneNotes` via `dangerouslySetInnerHTML` and explicitly documents that sanitization happens in `useAiFeedbackPanel`. The pattern works today, but any future caller — a Storybook story, a unit test mock, a new consumer pulling the partial into another panel — could pass unsanitized HTML and quietly introduce a stored-XSS vector. Defense-in-depth ("sanitize at every render boundary") would cost ~µs per render and remove a class of regressions.
**Impact:** Today: zero — single caller, well-documented. Tomorrow: a future ticket reuses `DraftedState` and assumes the prop is plain text, or a stored summary is hand-edited bypassing the `draft` useMemo path. Low probability, but the cost of the defense is also low.
**Fix:** Either re-sanitize in the component (idempotent, cheap):

```typescript
<StyledDraftedBody
  data-testid="ai-drafted-body"
  // eslint-disable-next-line react/no-danger
  dangerouslySetInnerHTML={{ __html: sanitizeHtml(draft.summary) }}
/>
```

Or accept a branded type at the prop level so the compiler enforces the contract:

```typescript
type SanitizedHtml = string & { readonly __brand: "SanitizedHtml" };

// in useAiFeedbackPanel:
summary: sanitizeHtml(latestStatus.summary ?? "") as SanitizedHtml
```

The branded-type approach is preferable — it pushes the contract into the type system without paying runtime cost.

### 4. `BLOCKED_FORMATS` case-sensitivity is inconsistent with the JSON branch

**[File: apps/creative-portal/api/aiReviewFeedback/apiHandlers/submit/index.ts]**
**Function/Class:** `getAiReviewFeedbackSubmitHandler` (the gate at line 148)
**Severity:** low
**Problem:** The blocked-formats check is `AI_REVIEW_FEEDBACK_BLOCKED_FORMATS.has(workItemFormat)` — case-sensitive, expects `"PDF"`, `"XLSX"`, etc. Two lines later the JSON branch uses `workItemFormat.toUpperCase() === "JSON"`. If `order.workItemFormat` ever comes back lowercased from a different code path or a database migration, the JSON branch matches but the gate doesn't.
**Impact:** A lowercase `"pdf"` would bypass the gate and call Proofed.ai with a PDF file — which the upstream would reject, falling back cleanly. Not exploitable; minor consistency issue.
**Fix:** Either normalize once at the top:

```typescript
const workItemFormatUpper = workItemFormat.toUpperCase();
if (AI_REVIEW_FEEDBACK_BLOCKED_FORMATS.has(workItemFormatUpper)) { ... }
const isJsonFormat = workItemFormatUpper === "JSON";
```

Or use the existing `WORK_ITEM_FORMAT` enum on both sides.

### 5. `triggerAi` has no double-invocation guard

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/hooks/useAiFeedbackPanel.ts]**
**Function/Class:** `triggerAi` (called from `GenerateFeedbackCard` button)
**Severity:** low
**Problem:** A double-click on the "Generate Feedback" button (HTML/JSON flow) could call `submit.mutate(...)` twice before the first promise resolves. React Query does not coalesce these — both submits hit the server.
**Impact:** Two upstream Proofed.ai UUIDs created, one orphaned, billable. Already mitigated by `useActiveUuidLifecycle`'s auto-cancel-on-new-uuid behaviour, but a defensive guard at the button source is cheaper than detecting + cancelling.
**Fix:**

```typescript
const triggerAi = useCallback(() => {
  if (submit.isLoading) return;
  submit.mutate(/* … */);
}, [submit]);
```

Alternatively, disable the button while `submit.isLoading` in `GenerateFeedbackCard`.

### 6. Cancel and getStatus handler shells lack integration tests

**[File: apps/creative-portal/api/aiReviewFeedback/apiHandlers/cancel/index.ts, getStatus/index.ts]**
**Function/Class:** handler shells
**Severity:** medium
**Problem:** Only `submit/index.test.ts` (557 lines, thorough) and `getStatus/utils.test.ts` (mapper-only) exist. The cancel and getStatus *handlers* (method gates, assignment check call, terminal-status cleanup, error handling) have no direct coverage — only transitive coverage via the service-layer test.
**Impact:** A regression in either handler (e.g. someone weakens the assignment check, or skips the session cleanup on terminal status) won't be caught. The PR description acknowledges this as "Open follow-ups" but the cancel handler in particular is a security-sensitive path (calls the upstream cancel) and deserves direct tests.
**Fix:** At minimum, add tests for:
- `cancel/index.test.ts`: missing uuid → 400; poll not in session → 404; assignment check failure → 403; success deletes poll and saves session.
- `getStatus/index.test.ts`: assignment check failure → 403; terminal status (`completed`/`failed`/`cancelled`) removes poll from session; non-terminal status leaves it.

The PR mentions ~400-800 LOC of busboy/middleware mock scaffolding required — that's a fair reason to defer for submit, but cancel and getStatus don't use busboy and are simpler to mock.

### 7. Server doesn't enforce the 30K word-count gate on submit

**[File: apps/creative-portal/api/aiReviewFeedback/apiHandlers/submit/index.ts]**
**Function/Class:** `getAiReviewFeedbackSubmitHandler`
**Severity:** low
**Problem:** Jira requirement 1.5 says "should not run for documents above 25MB or when the document exceeds 30K words". The 25MB gate is enforced server-side (busboy `fileSize`); the 30K word gate is enforced only on the client (`useAiFeedbackEligibility.ts`). A direct API call bypassing the UI could submit a 200K-word file.
**Impact:** Bypassing the gate would either succeed (Proofed.ai accepts it and charges us for the larger document) or fail upstream (still wastes capacity). Not a security issue but defeats a cost-control gate. The client gate is sufficient for normal use; this is hardening.
**Fix:** Mirror the 25MB pattern — compute reviewer word count after `processWorkItemContentWithMetadata` resolves and reject with 413 if it exceeds `MAX_AI_REVIEW_FEEDBACK_WORD_COUNT`. Or defer with a follow-up ticket if the cost of the word-count calc is significant.

### 8. `useAiFeedbackEligibility` has no test file in the diff

**[File: apps/creative-portal/services/aiReviewFeedback/useAiFeedbackEligibility.ts]**
**Function/Class:** `useAiFeedbackEligibility`
**Severity:** low
**Problem:** This hook is the gate that decides whether the AI panel renders in both `ReviewSubmission` and `ReviewForm`. It combines `useReviewAiDisabled`, blocked-format check, word-count check, and assignment context. No dedicated test file was added.
**Impact:** A regression that flips `isEligible` to `true` when it shouldn't be (or vice versa) would silently degrade UX in two surfaces. Tests at the consumer level (`ReviewSubmission`, `ReviewForm`) may exercise some paths but not the hook contract directly.
**Fix:** Add `useAiFeedbackEligibility.test.tsx` covering the matrix: disabled → false; blocked format → false; over word-count → false; happy path → true; admin role bypass cases if any.

### 9. `tone_notes` XSS regression test missing

**[File: apps/creative-portal/api/aiReviewFeedback/apiHandlers/getStatus/utils.test.ts]**
**Function/Class:** `formatAiSummaryAsHtml`
**Severity:** low
**Problem:** The escape-then-parse pattern (`marked.parse(escapeHtml(markdown))`) is the only thing preventing raw HTML in upstream markdown from rendering. There's no explicit test like:

```typescript
it("escapes raw HTML in tone_notes before passing to marked", () => {
  const html = formatAiSummaryAsHtml({
    summary: "ok",
    tone_notes: "<script>alert(1)</script>**bold**"
  });
  expect(html).not.toContain("<script>");
  expect(html).toContain("&lt;script&gt;");
  expect(html).toContain("<strong>bold</strong>");
});
```

**Impact:** Regression-test gap. Defense-in-depth in `sanitizeHtml` on the client would catch it, but a server-side test pins the contract at the source.
**Fix:** Add the test above to `utils.test.ts`.

---

## Tests

- ✅ Server-side strategy: 4 cases in `strategy/ai-review-feedback.test.ts` (covers happy path + 502/auth/network paths)
- ✅ Submit handler: 7 cases in `submit/index.test.ts` (auth, format gate, JSON branch, file branch, session save, 413, missing reviewer)
- ✅ `formatAiSummaryAsHtml` + helpers: 22 cases in `getStatus/utils.test.ts` (escape, markdown roundtrip, findings/examples shaping)
- ✅ Service-layer hooks: 4 cases in `services/aiReviewFeedback/index.test.tsx` (submit/getStatus/cancel + cache eviction)
- ✅ Panel hook + sub-hooks: 23 cases across 4 test files
- ✅ UI components: 28 cases (`AiFeedbackPanel`, `DraftedState`, `ReviewSubmission`, `ReviewForm`)
- ✅ Shared utils: `getMimeTypeFromFileName` extended (+30 lines); `verifyReviewerJobAssignment` 10 cases including strict-equality / fail-closed / error paths
- ❌ Cancel handler integration tests — none direct
- ❌ getStatus handler integration tests — utils only
- ❌ `useAiFeedbackEligibility` — no test file
- ❌ XSS regression test for `marked` markdown path
- ⚠️ Manual test plan present in PR description; covers DOCX, HTML/JSON, admin, disableAi, 25MB, >30K words

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ |
| Regression risk | ⚠️ Medium — stacked on PP-1720; deletes mock provider; significant refactor of the panel hook |
| Tests | ⚠️ Strong on hooks/services/strategy; gap on cancel + getStatus handler shells and eligibility hook |
| Code quality | ✅ |
| Mergeable state | ✅ Clean |
| Security | ✅ — defense-in-depth XSS, fail-closed auth, server-trusted format, busboy 413, session sweep |

---

## Recommendation

**Approve with suggestions.**

The PR is well-engineered, faithfully mirrors the OCR reference architecture Jira asked for, and demonstrates real security discipline (five hardening commits prompted by review, all addressing real concerns). The acceptance-criteria mapping is complete or near-complete on every Jira requirement.

Suggested before merging the eventual PP-1720 stack to `develop`:

1. **Address the bb.on("finish") / bb.on("error") double-response race (issue #1)** — small fix, prevents a noisy log on every 25MB+ upload.
2. **HTML-escape text blocks in `blockFormatFileToHtml` (issue #2)** — one-line fix using the existing `escapeHtml` helper; improves AI input quality.
3. **Either re-sanitize in `DraftedState` or introduce a `SanitizedHtml` branded type (issue #3)** — pushes the trust contract into the type system.

Deferrable to follow-up tickets (acknowledged in the PR description):

4. Cancel + getStatus handler shell tests (issue #6) — at least for the two non-busboy handlers.
5. Server-side 30K word-count gate (issue #7).
6. `useAiFeedbackEligibility` test file (issue #8).
7. XSS regression test for the marked path (issue #9).
8. `BLOCKED_FORMATS` case-sensitivity normalization (issue #4).
9. `triggerAi` double-click guard (issue #5).
