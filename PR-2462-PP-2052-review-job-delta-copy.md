# PR Review: feature/PP-2052: Store the Proofed AI review delta against the review job

**PR:** https://github.com/Proofed/B2BWebserver/pull/2462
**Jira:** https://proofed.atlassian.net/browse/PP-2052
**Status:** In Progress
**Reviewed at:** `a262c1bac` (3 commits, 35 files, +2556 / −39)

---

## What this means for users (non-technical summary)

1. **Submitting a review can take a lot longer.** Submitting now waits on the AI service to fetch and save the delta copy, and there is no time limit on that wait. If the AI service is slow or hangs, the reviewer can be left watching a spinner for minutes. The browser may then show an error even though the submission eventually goes through.
2. **One unusual AI document can slow the whole portal for everyone.** Cleaning the returned document gets dramatically slower as it grows. A delta of a few hundred KB with many unusual tags can freeze the server for 10+ seconds, and nothing limits the download size.
3. **On-platform PDF/PPTX reviews may never get a delta.** On two of the three submission paths, the delta is saved before any reviewer version exists for that job. By the PR's own description of OMS rules, that save is rejected, and the failure only shows up in logs. This depends on OMS behaviour we can't check from this repo.
4. **On wysiwyg orders, the stored delta may not match what was submitted.** If the reviewer keeps editing after running AI feedback, the delta still reflects the document as it was when AI ran.
5. **Nothing is visible to customers today.** Nothing reads the stored delta yet (PP-2051/PP-2053 will). The HTML clean-up has a gap, but it can't be exploited through any current screen.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
| --- | --- | --- |
| 1.1 Generate the delta (editor vs reviewer) via Proofed AI (Huxley) | Doesn't generate a new one. It reuses the delta Huxley already produces during the reviewer-feedback run (`delta_url` on the status response). A delta only exists if the reviewer ran AI feedback. | ⚠️ Partial (conditional on an AI run; documented in PR) |
| 1.2 Store as `WorkItemContentVersion` with versionType `ReviewJobDelta` | `storeReviewJobDelta`, wired into `patchJob`, `postSubmitJob`, `postSubmitJobStream`. Wysiwyg deltas are stored as blocks JSON; file deltas as DOCX verbatim. | ⚠️ Partial (see Issue 4: PDF/PPTX on non-streaming paths) |
| 1.3 Invisible process, not exposed to the customer | `diffVersion: false`. Customer portal file pickers only select EditedCopy/CleanCopy/TrackChanges. The uuid is stripped before OMS on all paths. | ✅ Addressed (in this repo; OMS `completedWorkItemVersionId` selection unverified) |
| 2.1 On-platform orders | JSON and file-based formats are handled. | ✅ / ⚠️ (Issue 4) |
| 2.2 GDRIVE orders | Not implemented. The PR explains Huxley can't run for Drive formats, so there's nothing to store. | ❌ Missing (acknowledged; needs its own ticket) |
| 3.1 No automatic changes to the review copy | The delta is a separate version; the reviewer's copy isn't modified. | ✅ Addressed |

**Beyond Jira scope:** `packages/wysiwyg` `AiChangeReason` empty-heading guard (affects all AI change cards), `packages/shared/api/utils/fileUploadStream.ts` aborted-handler try/catch, `processWorkItemContentWithMetadata` source-error forwarding, and `.gitignore` entries for local notes.

---

## Architecture Analysis

The approach reuses Huxley's by-product delta instead of calling a new endpoint. The uuid of the completed reviewer-feedback job travels from `AiFeedbackPanel` to Formik, then through the submit payload to the server. There, `storeReviewJobDelta`:

1. shape-checks the uuid;
2. calls `getStatus`;
3. binds the uuid to the job via the echoed `job_id`/`order_id` (fails closed);
4. downloads `delta_url` with manual redirect handling and a private-address blocklist;
5. spools it through `processWorkItemContentWithMetadata`;
6. stores it: wysiwyg deltas are parsed by a hand-rolled allow-list sanitiser (`parseReviewDeltaHtml`) into the existing blocks shape (`metadata.diffed` + `metadata.changes`); file deltas are stored verbatim.

It runs synchronously inside the submit request, before the Submitted PATCH. That's required because OMS rejects version writes once the order closes. Every failure is swallowed as `logger.warn`.

**Verified strengths:**
- The ownership binding correctly refuses cross-job uuid replay.
- The uuid never reaches OMS on any of the three paths.
- Axios errors are narrowed so the api-key isn't logged, and `delta_url` is never logged.
- The stored blocks shape round-trips through the real `generateAiChangeIdMappingForBlocks` → `convertBlocksToHtml` → `extractAiChangesFromBlocks`: every card matched its mark, checked with a harness.
- The sanitiser stops quote breakout, `<pre>` raw text, uppercase tags, comment/CDATA tricks, `javascript:`/whitespace-scheme hrefs and SVG data URIs.

Review lenses run: correctness/regressions, security, data-shape/React, resilience + tests + reuse + conventions (4 finders, then 3 adversarial verifiers). Accessibility lens not run: no UI added.

---

## Issues Found

### 1. The delta step has no timeout and blocks the reviewer's submission

**[File: apps/creative-portal/api/aiReviewFeedback/strategy/ai-review-feedback.ts]**

> **In plain terms:** When a reviewer clicks Submit after using AI feedback, the portal now waits for the AI service to hand over and save an extra hidden document before it finishes. Nothing limits how long that wait can be, so a slow AI service means a stuck submit button, a possible false error, and a confusing retry.

**Function/Class:** `AiReviewFeedbackProvider.getStatus`, `AiReviewFeedbackProvider.downloadDelta`; callers in `patchJob.ts`, `postSubmitJob.ts`, `postSubmitJobStream.ts`

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Log in as a reviewer on a review job with AI feedback enabled and run "Generate feedback" until it completes.
2. Make the AI service (or its storage) respond slowly, e.g. throttle it in a test environment.
3. Submit the review.
4. **Expected:** The submit completes promptly. The delta is a best-effort background step ("never blocks the reviewer's submission", per the docblock).
5. **Actual:** The submit request waits on `getStatus`, up to 4 download hops and the version upload before the job is marked Submitted. undici's defaults allow about 300 s per stalled hop, and a slow trickle has no overall limit.

**Problem:** `getStatus` (`ai-review-feedback.ts:379-386`) and both fetches in `downloadDelta` (`:517`, `:536`, `fetch(parsedUrl, { redirect: "manual" })`) pass no `signal`. `addWorkItemContentVersion` uses axios with no `timeout`. There's no global timeout or route `maxDuration`. All of this is awaited before `patchUpdateJob(... status: "Submitted")`.

**Evidence:** `storeReviewJobDelta/index.ts:52-53` promises "never blocks the reviewer's submission". Its callers do `await storeReviewJobDelta({...})` at `patchJob.ts:181`, `postSubmitJob.ts:136` and `postSubmitJobStream.ts:193`, all before the status patch. The repo has no `AbortSignal`, `axios.defaults.timeout` or `setGlobalDispatcher`.

**Impact:** Submission, which is the reviewer's core action, now depends on a third-party service's latency. A proxy or browser timeout can surface an error while the server finishes later, which invites a retry (see Issue 6).

**Fix:** Give the whole delta step one deadline and treat expiry like any other swallowed failure:

```typescript
const DELTA_DEADLINE_MS = 15_000;
const signal = AbortSignal.timeout(DELTA_DEADLINE_MS);
// pass `signal` into getStatus / downloadDelta fetches and the version upload
```

### 2. Parsing the delta can stall the server's event loop, and the download has no size cap

**[File: apps/creative-portal/api/utils/wysiwyg/parseReviewDeltaHtml.ts]**

> **In plain terms:** Cleaning up the AI's document gets much slower as the document grows. A modest document (about 200 KB) full of unusual tags freezes the server for around 10 seconds. While that happens, every other user of the creative portal on that server waits too.

**Function/Class:** `sanitizeTree`; `storeReviewJobDelta`; `downloadDelta`

**Severity:** medium

**Confidence:** high

**How to spot it:** Not reproducible through normal UI clicks, because it needs a specific delta document. Measured by bundling the real parser: sibling unknown tags (`<u2>x</u2>`) × 1,000 take 60 ms, × 8,000 take 647 ms, × 16,000 take 4.3 s, and × 20,000 (200 KB) take 10.1 s. `<p>a</p><hr>` × 16,000 (192 KB) takes 11.9 s. `hr` isn't on the allow-list, and StarterKit's horizontal rule is enabled in the editor. Whether Huxley echoes it back is unverified.

**Problem:** Each disallowed element is unwrapped with `child.replaceWith(child.innerHTML)` (`:189`). That re-serialises, re-parses and rebuilds the parent's child array, so N siblings cost at least O(N²). The parse runs synchronously in the submit request: `parseReviewDeltaHtml(await readFile(processed.filePath, "utf-8"))` (`storeReviewJobDelta/index.ts:195`). Neither `downloadDelta` nor `processWorkItemContentWithMetadata` caps bytes. The code comment itself calls the cap a "follow-up".

**Evidence:** As quoted above. No byte counting exists anywhere on the path: `processWorkItemContentWithMetadata.ts:176` only `statSync`s the size.

**Impact:** One large or odd delta blocks the Node event loop for all concurrent requests on that instance. There's also unbounded disk spooling on every path.

**Fix:**
- Enforce a max-bytes limit in `downloadDelta`: reject when `content-length` is over the limit, and count bytes in a Transform that destroys the stream past it. Use a much lower cap (a few MB) for JSON orders.
- Replace `replaceWith(string)` with direct child-node splicing, so unwrapping is linear.

### 3. The sanitiser can be bypassed with `/` after the tag name (latent stored XSS)

**[File: apps/creative-portal/api/utils/wysiwyg/parseReviewDeltaHtml.ts]**

> **In plain terms:** The step meant to strip dangerous code from the AI's document misses one simple trick. Nothing shows this document on screen today, so nobody is at risk yet. The admin area and job-history screens planned next (PP-2051/2053) would be exposed if they display it directly.

**Function/Class:** `sanitizeTree`

**Severity:** medium

**Confidence:** high

**How to spot it:** Not user-reproducible today, because no screen renders stored deltas. Reproduced with the real parser: input `<p>hi <img/src/onerror=alert(1)> there</p>` comes out unchanged in both `metadata.diffed` and `content`. jsdom parses it as `<img onerror="alert(1)">`. `<a/href=javascript:alert(1)>` and `<svg/onload=alert(1)>` also survive. The control `<img src=x onerror=…>` is correctly stripped.

**Problem:** node-html-parser's tag regex requires whitespace after the tag name, but browsers also accept `/`. So `<img/src/…>` becomes a TextNode. `sanitizeTree` skips non-elements (`:172`, `if (!(child instanceof HTMLElement)) return;`), and `toString()` re-emits the text verbatim, where a browser then parses it as markup.

**Evidence:** Current readers all go through the Tiptap schema, which drops unknown attributes and `javascript:` links: `getWorkItemContentVersion.ts:48`, shared `getWorkItemContentVersion.ts:90`, `createOrderDocumentInTiptapFromBlocksFormat.ts:127`. No `dangerouslySetInnerHTML` consumer of version content exists. That's why this is latent. The file's own comment declares this function "a trust boundary".

**Impact:** Any future consumer that injects `diffed`/`content` as HTML ships stored XSS. Reaching it also needs Huxley to emit that markup, whether from a compromised upstream or reviewer markup passed through unescaped (unverified).

**Fix:** Don't hand-roll the allow-list on this tokenizer. Use a DOM-backed sanitiser on the server, e.g. DOMPurify with happy-dom (already present via `@tiptap/html/server`), or `sanitize-html`. At minimum, escape `<` and `>` in every TextNode's raw text before output. Add the payloads above as regression tests.

### 4. The ordering promise in the docblock is false on two paths, so PDF/PPTX deltas may be silently rejected

**[File: apps/creative-portal/api/aiReviewFeedback/storeReviewJobDelta/index.ts]**

> **In plain terms:** For reviews of PDF or PowerPoint orders sent from the job sidebar, the hidden delta is saved before the reviewer's own file is saved. If the order system requires the reviewer's file to exist first (as the PR itself says), the delta is quietly thrown away, and later screens (PP-2051/2053) will have nothing to show for those orders.

**Function/Class:** `storeReviewJobDelta` call sites in `patchJob.ts` and `postSubmitJob.ts`; `postAddWorkItemContentVersion`

**Severity:** medium

**Confidence:** high for the code order; the effect depends on OMS and is unverified

**Steps to reproduce:**

1. As a reviewer, open an on-platform **PDF or PPTX** review job in the JobManagement sidebar.
2. Run AI feedback to completion, upload the edited copy, and submit.
3. Check the job's work item content versions and the server logs.
4. **Expected:** A `ReviewJobDelta` version is stored.
5. **Actual (per code order):** At delta time no version for the review job exists. If OMS enforces "latest version belongs to this job", the POST fails and only `Review delta: could not store the delta copy` is logged.

**Problem:** The docblock (`:44-48`) says the delta runs "after the reviewer's own content version is written". But `postAddWorkItemContentVersion` writes the clean/edited copy itself only when streaming (`isOnPlatform && isFileBasedWorkItemFormat && isProcessedFiles`, `:80-81`). Otherwise it returns base64 `workItemContent`, which goes to OMS on the later Submitted PATCH (`patchJob.ts:212`, `postSubmitJob.ts:155-158`).
- **DOCX:** a TrackChanges version is written first (`:143-153`, required by the form schema).
- **JSON:** a `yjs` version is written first (`:155-161`).
- **PDF/PPTX:** only upload `editedCopy` (`FormModal/utils.ts:172-173`), and AI isn't blocked for them (`api/aiReviewFeedback/consts.ts:24-27`), so nothing is written before the delta.

**Evidence:** As quoted. The PR's tests mock `postAddWorkItemContentVersion` and only assert delta-before-status-patch (`patchJob.test.ts:601-618`, `postSubmitJob.test.ts:269-293`), never version-before-delta.

**Impact:** Requirement 1.2 may silently not hold for PDF/PPTX on the sidebar and legacy submit paths.

**Fix:** Confirm the OMS rule. If it holds, move the delta write after the reviewer's version is persisted (or persist the edited copy first) on the non-streaming paths. Either way, correct the docblock and add a PDF test asserting call order.

### 5. The stale-uuid test proves nothing, and the key clears are untested

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/hooks/useAiFeedbackPanel.test.tsx]**

> **In plain terms:** The automated check meant to stop an outdated AI result being attached to a submission would still pass if that protection were deleted. The protection could break later without anyone noticing.

**Function/Class:** `useAiFeedbackPanel` tests

**Severity:** low

**Confidence:** high

**How to spot it:** Code health, not user-reproducible. The `it.each(["failed","cancelled"])` test seeds `[FIELD_UUID]: undefined` in `initialValues` (`:67`), renders once with a terminal status, and asserts `toBeUndefined()` (`:774`). Nothing ever wrote a uuid first. The clears in `resetAll` (hook `:124-133`, which the PR calls the important one) and at `triggerAi` start (`:305-318`) have no assertions.

**Problem:** It's a tautological test, and the real regression paths are uncovered.

**Impact:** A regression would let a delta computed against a replaced file be stored against the job.

**Fix:** Render with `completed` first (uuid set), then rerender to `failed`/`cancelled` and assert it's cleared. Add tests that call `resetAll` and `triggerAi` after completion.

### 6. Security guards and the client-side wiring lack meaningful tests

**[File: apps/creative-portal/api/aiReviewFeedback/strategy/ai-review-feedback.test.ts]**

> **In plain terms:** Some of the safety checks this PR adds, like refusing to follow a link to internal servers, aren't actually exercised by the automated tests. Neither is the step that passes the AI result from the screen to the server, which the PR itself says fails silently if broken.

**Function/Class:** `isBlockedRedirectHost`, `parseContentDispositionFilename`, redirect loop; `Submission/hooks.ts`, `FormModal/hooks.ts`, `pages/jobs/hooks.ts`

**Severity:** low

**Confidence:** high

**How to spot it:** Code health, not user-reproducible.
- The "private address" redirect test uses `http://169.254.169.254/latest/meta-data/` (`:625`), which is rejected by the https check (`:428`) before `isBlockedRedirectHost` (`:437`) ever runs.
- So the whole IPv4/IPv6 blocklist has no test, including the `::ffff:a9fe:a9fe` regression the code comments describe.
- Also untested:
  - the redirect-limit error (`:542`);
  - relative `Location` resolution (`:419`);
  - filename separator rejection (`:205`);
  - uuid forwarding in the three submit hooks;
  - the `AiChangeReason` guard;
  - the `fileUploadStream` aborted-handler change.

**Problem:** Security controls and cross-layer wiring have no test protecting them.

**Impact:** A future refactor can silently reopen the SSRF gap or drop the uuid, which stops deltas being stored.

**Fix:** Use `https://169.254.169.254/`, `https://[::ffff:169.254.169.254]/`, `https://[fd00::1]/` and `https://localhost/` as redirect targets, and assert the "private address" log. Add tests for the redirect limit, a relative Location and hostile filenames. Add hook tests asserting `aiReviewFeedbackUuid` reaches the mutation payload.

### 7. The SSRF redirect blocklist is hostname-only

**[File: apps/creative-portal/api/aiReviewFeedback/strategy/ai-review-feedback.ts]**

> **In plain terms:** When following the AI service's download link, the portal tries to refuse links pointing at internal machines, but some disguised addresses get through. This only matters if the AI service or its storage is compromised or misconfigured.

**Function/Class:** `isBlockedRedirectHost` / `isBlockedIpv4`

**Severity:** low

**Confidence:** high

**How to spot it:** Not user-reproducible, because it needs a malicious upstream redirect. Running the PR's function on `new URL(x).hostname`:
- **Allowed:** `169.254.169.254.nip.io`, `foo.localhost`, `metadata.google.internal`, `[::a9fe:a9fe]`, `[64:ff9b::a9fe:a9fe]`, `[fec0::1]`.
- **Correctly blocked:** `169.254.169.254`, `[::ffff:a9fe:a9fe]`, `localhost`, `[fd00::1]`.

**Problem:** There's no DNS resolution, so names that resolve to private addresses pass, as do several IPv6 transition forms. Preconditions limit this: the first hop is pinned to the provider host, redirects must be https, and there are at most 3 hops. Separately, `apps/creative-portal/package.json:8` starts with `NODE_TLS_REJECT_UNAUTHORIZED=0`. If production uses that script, the https requirement doesn't enforce valid certificates. That's unverified; see Open Questions.

**Impact:** With a compromised upstream, internal HTTPS services could be fetched, and on the DOCX path the response would be stored as a content version.

**Fix:** Validate the resolved IP at connect time with a custom undici `Agent` `connect.lookup` (e.g. `ipaddr.js` `range() !== "unicast"`), which also defeats DNS rebinding. At minimum, block `*.localhost` and trailing-dot forms.

### 8. A retry after a failed Submitted patch writes a duplicate delta

**[File: apps/creative-portal/api/aiReviewFeedback/storeReviewJobDelta/index.ts]**

> **In plain terms:** If the final "mark as submitted" step fails and the reviewer clicks Submit again, a second copy of the hidden delta is saved. Later screens may show duplicates.

**Function/Class:** `storeReviewJobDelta`

**Severity:** low

**Confidence:** high (code); OMS job state after a failed PATCH is unverified

**Steps to reproduce:**

1. As a reviewer, complete AI feedback on a review job.
2. Submit while the OMS status PATCH fails (e.g. a transient 5xx).
3. Resubmit from the same form; the error toast keeps the form and its uuid.
4. **Expected:** One `ReviewJobDelta` version.
5. **Actual:** Two, because `storeReviewJobDelta` has no existence check (`:82-226`).

**Problem:** The write isn't idempotent. Pre-existing TrackChanges and yjs writes have the same property.

**Impact:** Duplicate hidden versions that PP-2051/2053 will need to de-duplicate.

**Fix:** Skip the store when a `ReviewJobDelta` version already exists for the job, or document that consumers must pick the latest.

### 9. On wysiwyg orders, the stored delta can predate the submitted document

**[File: apps/creative-portal/components/organisms/AiFeedbackPanel/hooks/useAiFeedbackPanel.ts]**

> **In plain terms:** On in-portal (wysiwyg) orders, if the reviewer runs AI feedback and then keeps editing before submitting, the saved delta shows the document as it was when AI ran. The ticket describes the delta as the difference from the reviewer's *submitted* content.

**Function/Class:** `useAiFeedbackPanel`

**Severity:** low

**Confidence:** high (behaviour); whether it's a defect is a product call

**Steps to reproduce:**

1. As a reviewer on a JSON/wysiwyg order, click Generate Feedback and wait for "drafted".
2. Make further edits in the editor.
3. Submit.
4. **Expected (per ticket wording):** The delta reflects the submitted content.
5. **Actual:** The delta comes from the AI run's snapshot, taken by `processJsonReviewSubmission` via `getOrderDocumentFromTiptapInBlocksFormat` (`submit/utils.ts:201-203`). The uuid is cleared only by `resetAll`, `triggerAi` or failed/cancelled status, and JSON reviews have no `uploadedFile` to trigger `resetAll`.

**Problem:** Nothing ties the uuid to the document state it was computed from.

**Impact:** Admin and editor history would show a delta that doesn't match what was delivered.

**Fix:** Product decision. Either document "delta as of AI run", or clear the uuid (or prompt a re-run) when the document changes after completion.

### 10. Parser edge cases: id collisions split cards, and nested edits leave a deletion in `content`

**[File: apps/creative-portal/api/utils/wysiwyg/parseReviewDeltaHtml.ts]**

> **In plain terms:** In rare AI documents, one suggested replacement could show up as two separate cards (one without its explanation), or the "clean" copy could still contain deleted text.

**Function/Class:** `parseReviewDeltaHtml` (id assignment), `toReviewerCopy`

**Severity:** low

**Confidence:** high (executed); reachability depends on Huxley's output

**How to spot it:** Not user-reproducible today, since nothing reads the delta.
- **Collision:** raw ids `"a b"` and `"a-b"`, each as a del+ins pair, produce 3 records: `a-b`, `a-b-3` (deleted only), `a-b-4` (inserted only, empty reason). `claimedBy` only stores the suffixed key (`:345-351`), so both halves of the second pair miss.
- **Nesting:** `<del>old <ins>new <del>gone</del> kept</ins> tail</del>` produces `content` `new <del class="elizabeth-edit" data-id="3">gone</del> kept`. The hoisted inner HTML (`:246-258`) isn't re-scanned. `content` is only rendered when `diffed` is absent (`convertBlocksToHtml.ts:80-86`), and it never is here.

**Problem:** The collision guard defeats its own stated purpose (comment `:329-332`), and the reviewer-copy extraction isn't recursive.

**Impact:** Minor mis-rendering for future consumers.

**Fix:** Keep a `rawId → assignedId` map and reuse it on repeat occurrences. Loop `toReviewerCopy` until no `del.elizabeth-edit` remains.

### 11. The linked design doc is gitignored, and the `AiChangeReason` comment is wrong

**[File: .gitignore]**

> **In plain terms:** The PR description points reviewers to a design write-up that isn't in the repository. A code comment also describes behaviour that is no longer true.

**Function/Class:** n/a

**Severity:** low

**Confidence:** high

**How to spot it:** Code and PR hygiene, not user-reproducible.
- `.gitignore` adds `docs/PP-2052-REVIEW-DELTA.md` and `pp2052-samples` under "local-only working notes", so `git show HEAD:docs/PP-2052-REVIEW-DELTA.md` fails. Meanwhile the PR body links it as the "Full write-up with diagrams", and `storeReviewJobDelta/index.ts:193-194` refers to "the follow-up list" in it.
- `docs/` already commits ticket docs (`docs/delivery-plan-PP-1544.md`).
- `AiChangeReason/index.tsx:53-58` says every PP-2052 annotation carries neither list, but `parseReviewDeltaHtml.ts:366-368` fills `deleted`/`inserted`. The comment went stale within this PR.

**Problem:** Ticket-specific personal ignores sit in the shared `.gitignore`, and the design doc and the follow-ups (byte cap, storage-format decision) are unavailable to reviewers.

**Impact:** Reviewers and future maintainers lose the documented known limits.

**Fix:** Commit the doc under `docs/` (or move the ignores to `.git/info/exclude`) and reword the `AiChangeReason` comment.

---

## Open Questions

- Does OMS `POST /workItemContentVersions` actually reject when the order's latest version belongs to another job, and does a TrackChanges-only version satisfy it? That decides whether Issue 4 is a live defect. — `storeReviewJobDelta/index.ts:44-48`
- Does Huxley's status response really echo `job_id` / `order_id`? If not, `status.job_id !== String(jobId)` always fails and no delta is ever stored. Tests mock it, so they wouldn't catch this. — `storeReviewJobDelta/index.ts:117-120`
- How does OMS choose `completedWorkItemVersionId` at CloseOrder? If it's "newest version by id" without a type filter, the stream path (delta written after CleanCopy) could surface the delta as the customer's final copy. — `postSubmitJobStream.ts:193`
- Does production start the creative portal via the `start` script (`NODE_TLS_REJECT_UNAUTHORIZED=0`)? That raises the practicality of Issue 7. — `apps/creative-portal/package.json:8`
- The submit branch of `patchJob` has no `job.proofedUserId === requesterId` check before the delta work (the start branch does). This predates the PR, and the uuid binding limits the damage. Does OMS reject version writes from non-assignees? — `patchJob.ts:159-193`
- Could a delta contain `<ins>`/`<del>` without the `elizabeth-edit` class? Those survive sanitisation and would become highlights with no card. The reviewer HTML sent upstream is cleaned by `cleanTrackChangesHtml`, so this would have to come from Huxley itself. — `parseReviewDeltaHtml.ts:95-106`
- The PR states that submitting mid-generation (status `processing`) stores nothing and never retries. Is that loss acceptable for PP-2051/2053?
- `downloadDelta`'s error path reads the entire response body before slicing to 2,000 characters (`ai-review-feedback.ts:553`). Is this worth bounding alongside Issue 2's byte cap?

---

## Validation Checks

Full monorepo scope, because `packages/shared` and `packages/wysiwyg` changed. Ran in place on a detached checkout of `a262c1bac`, then restored to `develop`.

| Check | Result | Notes |
| --- | --- | --- |
| `turbo run typecheck` | ✅ | 5/5 workspaces, 0 errors |
| `turbo run lint` | ✅ | 5/5 workspaces, 0 ESLint errors/warnings (only yarn `package.json` dependency-range notices) |
| `turbo run test` | ⚠️ Partial | See the breakdown below |
| `turbo run build` | ⏭️ Not run | Skipped after the test run hung; typecheck already covers types. Re-run in CI before merge. |

**Test run breakdown:**
- **wysiwyg:** 275 passed / 4 skipped.
- **creative-portal:** the full run hung (0% CPU for 30+ min after ~250 files) and was stopped. A scoped re-run of every PR-affected area passed: 238/238 tests across 19 files. That covers `storeReviewJobDelta` 14, `ai-review-feedback` 26, `parseReviewDeltaHtml` 21, `patchJob` 25, `postSubmitJob` 10, `postSubmitJobStream` 9, `useAiFeedbackPanel` 25, FormModal hooks and pages/jobs hooks.
- **Local-environment failures, not caused by the PR:**
  - shared: 1,445 passed, 1 failed, 18 files failed to load. Causes: `muhammara` native binary needs `GLIBCXX_3.4.29`; ESM `require()` of `@mdx-js/react`; locale formatting `10,00,000`.
  - customer-portal: 374 passed, 6 failed, due to the same ESM issue plus network 400s.
  - `ReviewForm/index.test.tsx`: same `@mdx-js/react` load error.
- **PR-touched shared test:** `extractFileInfo` 17/17 passed. `processWorkItemContentWithMetadata.test.ts` couldn't load locally (muhammara native binary), so **it is unverified here**.

---

## Tests

- ✅ New unit tests for `storeReviewJobDelta` (ownership mismatch, missing refs, malformed uuid, both storage paths, swallowed failures)
- ✅ `parseReviewDeltaHtml` tests (script/iframe/style, `<pre>`, quote re-quoting, svg data URIs, nesting)
- ✅ Submit-path tests assert delta-before-status-patch on all three paths
- ✅ `patchJob` test asserts the uuid is stripped before OMS
- ❌ Stale-uuid clear test is tautological; `resetAll`/`triggerAi` clears untested (Issue 5)
- ❌ SSRF blocklist, redirect limit, relative Location, hostile filenames untested (Issue 6)
- ❌ No test for the `/`-after-tag-name sanitiser bypass (Issue 3)
- ❌ No version-before-delta ordering test; PDF/PPTX path uncovered (Issue 4)
- ❌ No tests for uuid forwarding in the three submit hooks, the `AiChangeReason` guard, or the `fileUploadStream` aborted handler (Issue 6)
- ⚠️ `processWorkItemContentWithMetadata.test.ts` not runnable locally (native dependency); confirm in CI

### Suggested manual QA script

1. **(Req 1.2, DOCX)** Reviewer on an on-platform DOCX order: run AI feedback to completion, upload the clean copy and track changes, submit. Confirm one `ReviewJobDelta` version exists on the job.
2. **(Issue 4)** Repeat with an on-platform **PDF** order from the JobManagement sidebar. Check whether a `ReviewJobDelta` version exists, and check logs for "could not store the delta copy".
3. **(Req 1.2, wysiwyg)** Repeat on a JSON/wysiwyg order. Confirm the stored version is blocks JSON with `metadata.diffed` and `metadata.changes`.
4. **(Issue 9)** On a wysiwyg order, run AI feedback, then edit the document and submit. Compare the stored delta with the submitted content.
5. **(Req 1.3)** As the customer on each order above, confirm the final copy and the "Track Changes" download are the reviewer's documents, not the delta.
6. **(Issue 1)** With the AI service throttled or unavailable, submit a review after AI feedback. Measure submit time, and confirm the submission still succeeds.
7. **(Uuid lifecycle)** Run AI feedback, replace the uploaded file, then submit without re-running. Confirm no delta is stored.
8. **(Admin route)** Submit a review via the admin OrderJobs form modal after AI feedback. Confirm the delta is stored.

---

## Summary

| Aspect | Status |
| --- | --- |
| Correctness | ⚠️ Works on the main paths; PDF/PPTX ordering is unverified against OMS (Issue 4); GDRIVE not deliverable (acknowledged) |
| Regression risk | ⚠️ Medium: submit latency now depends on the AI service (Issue 1); shared/wysiwyg changes are additive |
| Tests | ⚠️ Good volume; key guards and wiring untested or tautological (Issues 5, 6) |
| Accessibility | n/a (no UI) |
| Error handling | ⚠️ Failures swallowed correctly, but no timeout or size cap (Issues 1, 2) |
| Security | ⚠️ Latent sanitiser bypass, SSRF blocklist gaps (Issues 3, 7); run `/security` after fixes |
| Code quality | ⚠️ Very heavy comments vs surrounding code; stale comments; doc gitignored (Issue 11) |
| Validation suite | ⚠️ typecheck ✅ lint ✅; tests: PR-scoped ✅, full run hit local env failures; build not run |
| Mergeable state | ✅ GitHub `clean` (validation incomplete locally; confirm CI) |

---

## Recommendation

**Approve with suggestions.** No blockers. The rubric allows a plain Approve for medium/low findings, but Issues 1–4 affect submit reliability and a security trust boundary, so they should be treated as pre-merge asks.

1. Add an overall deadline (`AbortSignal.timeout`) to `getStatus`, the delta download and the version upload, and fix the "never blocks" docblock (Issue 1).
2. Add a byte cap to `downloadDelta` and make the unwrap step linear (Issue 2).
3. Replace the hand-rolled sanitiser with a DOM-backed one (DOMPurify + happy-dom, or `sanitize-html`), and add the `/`-separator payloads as regression tests (Issue 3). This must land before PP-2051/2053 render deltas.
4. Confirm the OMS "latest version belongs to this job" rule, then fix ordering for PDF/PPTX on the non-streaming paths, or correct the docblock and add a test (Issue 4).
5. Fix the tautological stale-uuid test and add SSRF-blocklist, redirect and uuid-forwarding tests (Issues 5, 6).
6. Commit `docs/PP-2052-REVIEW-DELTA.md` (or drop the dead link) and remove the personal `.gitignore` entries (Issue 11).
7. Get a product decision on Issues 8 and 9, and on the wysiwyg storage format flagged in the PR description.
8. Run `/security` on the revised branch, and confirm `test` (including `processWorkItemContentWithMetadata.test.ts`) and `build` pass in CI.
