# PR Review: feature/PP-2176: Mark up PDF jobs in the browser with Adobe PDF Embed

**PR:** [https://github.com/Proofed/B2BWebserver/pull/2517](https://github.com/Proofed/B2BWebserver/pull/2517)
**Jira:** [https://proofed.atlassian.net/browse/PP-2176](https://proofed.atlassian.net/browse/PP-2176)
**Status:** Follow-up fixes done in [PP-2223](https://proofed.atlassian.net/browse/PP-2223) on `feature/PP-2223-pdf-markup-follow-up` (8 Oct 2026). Issue 5 waits for a product decision.
**Reviewed at:** `335a9c9a6` (105 files, +9,688 / −178, 36 commits). `yarn.lock` excluded from line review.
**Validation suite:** Skipped (user opted out).

---

## Fix status (PP-2223, updated 8 Oct 2026)

All fixes are on `feature/PP-2223-pdf-markup-follow-up`, pushed. Each was checked with unit tests, typecheck, lint, a production build, and in the browser on localhost (test orders 21767 to 21775).

| # | Issue | Status | Commit(s) |
|---|---|---|---|
| 1 | Submit before restore loses marks | Fixed: the seed file is handed over only after restore; submit refuses while the document is still opening | `1cc25488c` |
| 2 | Text-less scans turned sideways | Fixed: pages are compared by the turn they need, own `/Rotate` first; under 20 characters nothing turns | `be543a90a` |
| 3 | Order identifier not restamped on the main path | Fixed: one `processPdfSubmission` helper for both branches; restamp names the file a PDF | `19f0cac3c` |
| 4 | SAS link reaches the browser | Not changed, by decision: other screens on develop already give signed-in users signed links. The false comments are corrected | `95768e37b` |
| 5 | Encrypted PDFs refused at submit | **Open:** waiting for Adam (viewer skipped for encrypted files, and how to handle names on upload) | |
| 6 | Viewer can load forever | Fixed: after 60 s a "taking longer than usual" message with the download-and-upload link; no auto-cancel | `1cc25488c` |
| 7 | Restored replies lose their parent | Fixed: parents restored first, reply sources remapped, one refusal no longer sinks the rest | `1cc25488c` |
| 8 | Stale-job guard never matches | Fixed: reads the AxiosError response body | `1cc25488c` |
| 9 | Leaving without collapsing drops marks | Fixed: unsaved marks are saved on unmount | `1cc25488c` |
| 10 | Quick saves dropped or out of date | Fixed: save on tab hide; on page close a recent snapshot is sent | `1cc25488c` |
| 11 | Valid customer dates rewritten, links dated | Fixed: hex dates read, short dates written out in full with value and zone kept, links/popups/widgets skipped | `1c9c5a31f` |
| 12 | PDF route performance | Partly fixed: failed orientation checks reported and cached, `responseLimit: false`. The memory part (dates cache, views not copies) is deferred | `c4736d57f` |
| 13 | Extra PDF copies in the browser, download not cancelled | Fixed: dead refs removed, Adobe gets the buffer (submit file made first, since Adobe empties it), download aborted on leave | `3fd728d66` |
| 14 | Raw server text in submit toasts | Resolved: no caller passes server text any more; the unused `getJobActionErrorMessage` is removed | `49cc7901a` |
| 15 | Off-platform PDFs get an internal copy and can be refused | Fixed in part: no internal copy off platform, and the copy is stored only after the strip passes. The strip still runs off platform, on purpose: the job panel's PATCH submit sends that content to OMS (`patchJob.ts:215`) | `b171fc94e` |
| 16 | Browser can store `EditedCopyInternal` | Fixed: the create route refuses it with 400 | `7a8fdedc9` |
| 17 | Adobe script loads on every job | Fixed: loaded only for an active PDF job | `285de5e18` |
| 18 | Missing tests | Fixed: tests added with every fix above, each shown failing on the old code where it guards a bug | all |
| 19 | Conventions and code health | Fixed: rename, draft helpers moved to `utils/`, `apiRoutes` entry, shared helpers reused, one pdf-lib loader, `X-Rotations` honest, viewer brief matches the job panel, comments cleaned, made-up test ids, bounded caches. Skipped: duplicate `brief-eye.svg` (the wysiwyg package does not export its assets), `yarn bump-packages` and the `package.json` re-sort (merge-time process) | `df1c8c8a2` `c1f12ba87` `49cc7901a` `070688ab8` `ceb917eac` `c72a4f5f7` `95768e37b` `0bfa682f8` `a9e19b4c7` |

**Open questions, answered:**
- pdfjs in the production build: it did fail. The worker file was not traced into the standalone build, so every server read failed silently and customer comment authors were relabelled. Fixed in `25171d256`; needs one check on b2btest after deploy.
- Restored marks blocking submit: confirmed once on b2btest (order 21756), not on 8 Oct (order 21773, where Adobe re-saved the restored mark). Kept as a known limitation; the submit message now says to make a small change and click Save (`795c0cb27`).
- Adobe keeping `/UUID`: kept in the test files; restamped anyway (Issue 3). Encrypted files stay encrypted, and Adobe blocks all mark-up on them.
- Customer portal access by id: guarded. Internal copies and drafts answer 404 on the client API (`5b178bf14`).
- Unstripped files when both parsers fail, admin submit-on-behalf, the `/pdf` route's session-only check (backend ticket), and the design doc location: unchanged in PP-2223. The design doc moved to this repo.

**Also found and fixed during testing:** a `patchJob` test that timed out under load and leaked a call into the next case (`78b157fd9`). Not fixed here, pre-existing on develop: `formatWordQuantity` fails on a machine with an Indian number locale, and the Header and SideNav tests hang on this branch (fixed on develop by #2530).

---

## What this means for users (non-technical summary)

1. **An editor can lose their restored marks for good.** Say an editor reopens a PDF job that has autosaved marks, closes the viewer while it is still loading, and submits. The job goes in without those marks, and the marks cannot be recovered afterwards.
2. **Some scanned PDFs open sideways.** A scanned document with no text layer, which normally displays upright, is turned the wrong way in the viewer.
3. **Password- or permission-protected PDFs with any marks on them can no longer be submitted.** The error tells the editor to use the in-browser editor, and that route hits the same refusal.
4. **The viewer can spin forever.** If Adobe's viewer fails to start, or the download stalls, the editor sees a loading screen with no error and is never switched to download-and-upload.
5. **The claim that the PDF is never exposed by a public link does not hold.** Opening the viewer still sends the browser a temporary direct-download link to the file. Other screens on develop already do this, so the exposure is not new, but this PR's security claim and ticket rule 3 are not met.

---

## Jira Requirements vs Implementation


| Jira Requirement                                                                                               | PR Implementation                                                                                                                                       | Status          |
| -------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- |
| 1. PDF jobs open in an embedded Adobe viewer, via a button, for PDF jobs only                                  | `PdfContentPill` + `PdfViewerModal`, gated by `usePdfSubmissionMode` (`isPdf`)                                                                          | ✅               |
| 2. "Trouble with the editor?" / "Prefer to edit in your browser?" toggle; marks kept on switch (agreed 28 Sep) | `PdfSubmissionModeLink`, mode state in `usePdfSubmissionMode`, viewer kept mounted                                                                      | ✅               |
| 3. Embed only for PDFs under 50 MB                                                                             | `PDF_EMBED_SIZE_CEILING_BYTES`; route returns 413 and the client falls back                                                                             | ✅               |
| 4. Annotation tools on, existing annotations editable, download available, fit-width                           | `config/adobeEmbed.ts` viewer options                                                                                                                   | ✅               |
| 5.1.1 Pre-existing (original) annotations keep their metadata                                                  | Authors kept via `readOriginalPdfAuthors`. But `ensureAnnotationDates` rewrites legal-but-short or hex dates and adds dates to links/widgets (Issue 11) | ⚠️ Partial      |
| 5.1.2 New annotations labelled "Proofed" for the customer                                                      | `stripPdfInternalAuthors` on every server submit path                                                                                                   | ✅               |
| 5.2 Internal labels by role for internal users                                                                 | `EditedCopyInternal` + role labels. AI labels deferred by agreement (comment 76970)                                                                     | ✅ (scoped)      |
| 6.1 Each submission stores the whole PDF with customer annotations                                             | EditedCopy written on submit                                                                                                                            | ✅               |
| 6.2 Autosave after each annotation so progress is never lost                                                   | `PdfAnnotations` drafts, 20 s debounce, minimise/beacon/flush. Gaps in Issues 1, 9, 10                                                                  | ⚠️ Partial      |
| 7. Pages shown in correct orientation (≥20 chars / 60% rule)                                                   | `rotationDetect.ts` implements the rule correctly for text pages, but undoes /Rotate on text-less scans (Issue 2)                                       | ⚠️ Partial      |
| 8. Editor job history unchanged                                                                                | Internal and draft versions excluded via `workItemContentVersionRules`. Verified unchanged                                                              | ✅               |
| 9. Admin view unchanged                                                                                        | Admin UI unchanged (Fallback state). Server-side, admin submit-on-behalf now also stores an internal copy and rewrites authors (see Open Questions)     | ⚠️ Partial      |
| Rule: over-ceiling PDF shows the fallback                                                                      | 413 → fallback                                                                                                                                          | ✅               |
| Rule: PDF never served from a public URL                                                                       | `/pdf` route streams bytes, but the seed lookup still returns the SAS URL (Issue 4)                                                                     | ❌               |
| Rule: a save built on a stale version is never accepted                                                        | Client guard exists but can never match (Issue 8). OMS enforces "latest version belongs to job" on submit                                               | ⚠️ Partial      |
| Rule: a file is never rotated twice                                                                            | Correction subtracts `page.rotate`; verified                                                                                                            | ✅               |
| Rule: an unreadable file never wipes existing comments                                                         | pdf-lib failure falls back without rewriting                                                                                                            | ✅               |
| Rule: the client never receives a file showing an editor's real name                                           | Strip runs on all paths. Deliberate pass-through when both parsers fail (Open Questions)                                                                | ✅ (with caveat) |


**Scope beyond the ticket:**

- `getJobActionErrorMessage` changes the jobs-page submit error toast for all submits (Issue 14).
- `RawButton` gains `isUnderlined` (shared atom).
- `FullscreenModal` layout change (`isTallHeader`).
- `HTMLContentPill` refactor.
- A 678-line `docs/PP-2176-pdf-in-browser-markup.md`.

All are related to the feature, but they widen the review surface.

---

## Architecture Analysis

- **Client flow:**
  - `JobManagement` → `usePdfSubmissionMode` decides between Embed and Fallback.
  - `PdfContentPill` mounts `PdfViewerModal` once per job turn. It hides the viewer on collapse rather than unmounting it, so Adobe's in-memory marks survive.
  - The viewer fetches bytes from a new BFF route, `/api/workItemContentVersion/[id]/pdf`. That route resolves the SAS URL server-side, enforces the 50 MB ceiling, applies orientation, fills missing annotation dates and refuses drafts.
  - `usePdfAnnotationDrafts` saves the job's own marks as `PdfAnnotations` versions and restores them on reopen.
  - On submit, Adobe's `SAVE_API` output becomes the form's `editedCopy`.
- **Server submit** (`postAddWorkItemContentVersion`) runs three steps for PDFs:
  1. Write an `EditedCopyInternal` with role labels.
  2. Rewrite authors to "Proofed" in the customer file.
  3. Re-stamp the order UUID. This step is on the non-streaming path only; see Issue 3.
- **Shared code:** `packages/shared/api/utils/pdf/`* holds rotation detect/apply (pdfjs + pdf-lib), author normalise/read and date fill. `workItemContentVersionRules` hides internal and draft versions from every list.

The design fits the codebase well. It reuses `useScript`, `addUUID` and the shared rules module, parameterises one `normaliseAnnotationAuthors` for both copies, and keeps SDK absence non-fatal.

The problems are concentrated in three areas:

- the timing between viewer readiness, restore and submit;
- the duplicated streaming vs non-streaming submit branch;
- edge cases in the new PDF utilities.

---

## Issues Found

### 1. Submitting before the viewer finishes restoring sends the unmarked file and loses the draft

> **PP-2223 status:** Fixed in PP-2223 (`1cc25488c`).

**[File: apps/creative-portal/components/organisms/modals/PdfViewerModal/hooks.ts]**

> **In plain terms:** An editor comes back to a PDF job that has autosaved marks. They open the viewer, close it again before it has finished loading, and press Submit. The job is submitted without any of their saved marks. Once the job is submitted, those marks can never be put back.

**Function/Class:** `usePdfViewerModal` (`start`, `prepareForSubmit`)

**Severity:** high

**Confidence:** high

**Steps to reproduce:**

1. As an editor with an Active PDF job, open the viewer, add a few marks, wait more than 20 s for the autosave, then reload the page.
2. Click the PDF pill to open the viewer. While the loading overlay is still showing, click the collapse button in the top bar.
3. Click Submit in the side panel.
4. **Expected:** the submit waits for the document and restored marks, or refuses with "document still opening".
5. **Actual:** the unmarked starting file is submitted. The draft is never applied, and after submit it cannot be recovered.

**Problem:** The unmarked starting file is handed to the form as soon as its bytes arrive, before the viewer has loaded, attached its listeners or restored the draft. Submit has no check that the viewer is ready.

**Evidence:**

- **Seed handed over first:** `hooks.ts:148` `handOverFile(new File([buffer], fileName, { type: "application/pdf" }))` runs before `previewFile` (:218), `registerEventListener` (:235) and `await restore()` (:247).
- **Seed written into the form:** `PdfSubmissionFileSync/hooks.ts:17` `setFieldValue("editedCopy", submissionFile, false)`.
- **Collapse is never disabled:** the collapse button (`PdfViewerTopBar/index.tsx:66-73`) and `handleCollapse` (:279-287) run `onCollapse()` in a `finally`.
- **Submit only awaits the flush:** `Submission/hooks.ts:245` awaits `flushPdfDraft?.()`.
- **Flush returns the seed:** `prepareForSubmit` (:324-325) returns `latestFileRef.current` when `!hasUnsavedMarksRef.current`. That flag is only set by the listener that is attached at :235.

**Impact:** Silent, permanent loss of an editor's work. This breaks "restored on reopen" (Req 6.2).

**Fix:** Track readiness and make `prepareForSubmit` wait for it (or refuse), instead of handing over the seed early.

```typescript
const readyRef = useRef<Promise<void>>();
// in start(): readyRef.current = (async () => { ...previewFile...; await restore(); })();
const prepareForSubmit = async () => {
  if (!readyRef.current) throw new Error("The document is still opening.");
  await readyRef.current;
  // existing logic
};
```

### 2. Text-less scanned PDFs with an existing page rotation are turned sideways

> **PP-2223 status:** Fixed in PP-2223 (`be543a90a`).

**[File: packages/shared/api/utils/pdf/rotationDetect.ts]**

> **In plain terms:** Many scanners save pages the way they were fed in and add an instruction to display them upright. When a scan like that has no selectable text, the viewer undoes that instruction and shows the pages sideways. The editor then marks up a sideways document.

**Function/Class:** `resolvePageAngles` / `topAngle` / `detectRotationMap`

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Take an image-only PDF (a scan, no OCR text) whose pages carry `/Rotate 90`. It opens upright in Acrobat or Chrome.
2. Create a PDF order with it and open the job in the in-browser viewer.
3. **Expected:** the pages display upright, as in every other viewer.
4. **Actual:** the pages display rotated 90°.

**Problem:** With no text, the document angle falls back to 0. The per-page correction `0 − page.rotate` then cancels the page's own rotation.

**Evidence:** `rotationDetect.ts:61` `let best = 0;` and :73 `share: total > 0 ? … : 0`. For pages under the character threshold, :119-120 returns `document.angle`, which is 0. Then :190 `normaliseDegrees(angle - pageRotations[index])` gives 270. `applyRotationMap.ts:51` sets `(90 + 270) % 360 = 0`. No test covers a non-zero `page.rotate` or a text-less document.

**Impact:** Breaks Req 7 ("Pages should be displayed in their correct orientation") for a common class of documents.

**Fix:** Return no correction when the document has no reliable text, and add tests for both cases.

```typescript
if (documentWeights.size === 0 || totalChars < MIN_CHARS) return {};
```

### 3. The normal submit path never re-stamps the order identifier on the PDF

> **PP-2223 status:** Fixed in PP-2223 (`19f0cac3c`).

**[File: apps/creative-portal/api/utils/jobs/postAddWorkItemContentVersion.ts]**

> **In plain terms:** Each order's PDF carries a hidden identifier that the platform checks when someone uploads that file again. For in-browser submits, the step that restores the identifier is skipped. If Adobe's save drops it, a later reviewer who downloads the file and uploads it back may be told it belongs to the wrong order.

**Function/Class:** `postAddWorkItemContentVersion` (streaming branch)

**Severity:** medium

**Confidence:** high that the step is skipped; medium on how often it matters, which depends on whether Adobe's save keeps the identifier.

**Steps to reproduce:**

1. Submit an on-platform PDF job from the in-browser viewer.
2. As the next assignee, download the file, mark it up in a desktop tool, and upload it with the fallback.
3. **Expected:** the upload is accepted, because the identifier was restored at submit.
4. **Actual:** if Adobe's save or the author rewrite dropped the identifier, the upload fails the identifier check.

**Problem:** The PDF steps are copied into both branches, and the copies have drifted. Only the non-streaming branch calls `restampPdfIdentifier`, yet on-platform submits always stream.

**Evidence:**

- **Non-streaming branch restamps:** `postAddWorkItemContentVersion.ts:112` `await restampPdfIdentifier({ file: finalVersionFile as FormidableFile, order });`.
- **Streaming branch does not:** :142-165 runs `storePdfInternalCopy` → `stripPdfInternalAuthors` → `addWorkItemContentVersionStream` with no restamp.
- **On-platform submits always stream:** `submitJob.ts` sets `useStreaming = isOnPlatform && isFileBasedWorkItemFormat`.
- **Nothing else re-adds it:** the stream conversion passes `skipUuidInjection: true`.
- **The code says it's needed:** the PR's own docblock (:93-97) says neither Adobe's save nor the rewrite "can be relied on to keep it".

**Impact:** The main path skips a safeguard the PR itself says is required. `restampPdfIdentifier` has no test, and the streaming branch is never exercised by tests.

**Fix:** Extract one helper and call it from both branches, then add a branch-level test.

```typescript
const processPdfSubmission = async ({ filePath, job, order, requesterId }) => {
  await storePdfInternalCopy({ filePath, job, order, requesterId });
  await stripPdfInternalAuthors({ file: { filepath: filePath }, order, requesterId });
  await restampPdfIdentifier({
    file: { filepath: filePath, mimetype: "application/pdf" } as FormidableFile,
    order
  });
};
```

### 4. The viewer still exposes the PDF's direct SAS download link to the browser

> **PP-2223 status:** Not changed, by decision; comments corrected (`95768e37b`).

**[File: apps/creative-portal/hooks/usePdfJobSeedVersion.ts]**

> **In plain terms:** The PR says the PDF is only ever streamed through our server, and the ticket requires it is never served from a public link. But opening the viewer still sends the browser a temporary direct-download link to the exact file being viewed. For a reviewer, that file is the internal copy with role labels. Anyone who copies the link can download the file until it expires.

**Function/Class:** `usePdfJobSeedVersion`

**Severity:** medium (develop already exposes such links elsewhere; this adds one more, and makes the PR's guarantee and ticket rule 3 untrue)

**Confidence:** high

**Steps to reproduce:**

1. Open DevTools → Network. As an editor, open a PDF job's side panel.
2. Find the request to `/api/workItemContentVersion/{seedVersionId}?returnContent=false`.
3. **Expected:** no blob URL in any response. Only the `/pdf` route returns the file.
4. **Actual:** the JSON response contains `versionPath` with a SAS-tokened Azure blob URL.

**Problem:** The seed hook calls the generic version endpoint only to read `bytes`, and that endpoint passes OMS's `VersionPath` (a SAS URL) straight through.

**Evidence:**

- **Seed hook calls the generic endpoint:** `usePdfJobSeedVersion.ts:91-98` `useGetWorkItemContentVersion({ convertContentToTiptap: false, id: seedVersionId as number, returnContent: false })`.
- **Handler passes the response through:** the shared handler `getWorkItemContentVersion.ts:114` does `res.status(200).json(data)`.
- **OMS puts the SAS URL in it:** OMS `WorkItemContentVersionsController.cs:386-393` sets `VersionPath = await AzureBlobHelper.RetrievePathWithSASToken(...)` when `returnContent` is false.
- **The PR's claim contradicts this:** `getWorkItemContentVersionPdf.ts:41-44` says "It never leaves the server".
- **Already exposed on develop:** develop already exposes such links via `FilesSection` and `UploadedFileView`.

**Impact:** Ticket rule 3 is not met, and the PR description's security claim is inaccurate.

**Fix:** Get the size without the generic endpoint. For example, have the `/pdf` route answer `HEAD` with `Content-Length`, or add a size-only BFF route. Then remove `useGetWorkItemContentVersion` from this hook, and correct the comment and the PR text.

### 5. Encrypted PDFs with any marks are now refused at submit, and the error points at a route that fails the same way

> **PP-2223 status:** Open: waiting for Adam.

**[File: apps/creative-portal/api/utils/jobs/stripPdfInternalAuthors.ts]**

> **In plain terms:** Some customers send permission-protected PDFs. Before this PR, editors could mark these up and submit them. Now any marks on such a file block the submission. The message says to use the in-browser editor, but marks made there are labelled "Editor", and those are refused the same way.

**Function/Class:** `stripPdfInternalAuthors`

**Severity:** medium

**Confidence:** high for desktop-tool uploads; medium for the in-browser case, which depends on whether Adobe's save keeps the file encrypted.

**Steps to reproduce:**

1. Create a PDF order with a permission-restricted (encrypted) PDF.
2. As the editor, download it, add a comment in Acrobat, and upload it with the fallback. Or mark it in the in-browser viewer.
3. Submit.
4. **Expected:** the submission succeeds, as before this PR, with the authors relabelled or safely handled.
5. **Actual:** 422 "This PDF names its authors and could not be processed. Use the in-browser editor, or remove the author names before uploading."

**Problem:** For encrypted files, `normaliseAnnotationAuthors` throws. The fallback then treats every author not in `kept` as foreign, and `kept` never contains our own role labels.

**Evidence:**

- **Role labels never kept:** `stripPdfInternalAuthors.ts:59-66` calls `pickKeptPdfAuthors` with `keepInternalLabels: false`, and `readOriginalPdfAuthors.ts:134` builds `new Set(keepInternalLabels ? labels : [])`.
- **Refusal:** :107-119 `foreign = authors.filter((author) => !kept.has(author))` → `throw new ApiError(422, PDF_AUTHORS_CANNOT_BE_REMOVED_MESSAGE)`.
- **Misleading message:** :28 text recommends the in-browser editor.
- **Encrypted PDFs do reach this path:** they are accepted into orders (`addUUID.ts:34-45` skips them rather than rejecting).
- **No test for this case:** the only encrypted test (`stripPdfInternalAuthors.test.ts:185-197`) uses a kept customer author.

**Impact:** A regression for encrypted PDF orders, on both editor submit and admin submit-on-behalf. The ticket says admin submit-on-behalf is unchanged.

**Fix:** Treat role labels as ours in the encrypted check, and give a message that offers a way out that actually works. Add tests for encrypted + role label and encrypted + desktop author.

### 6. The viewer can stay on its loading screen forever, with no error and no fallback

> **PP-2223 status:** Fixed in PP-2223 (`1cc25488c`).

**[File: apps/creative-portal/components/organisms/modals/PdfViewerModal/hooks.ts]**

> **In plain terms:** If Adobe's viewer script loads but never starts, the editor sees a spinner indefinitely. The same happens when the environment's Adobe ID doesn't match the site, or a large download stalls. Nothing tells them to switch to download-and-upload, and nothing is reported to us.

**Function/Class:** `usePdfViewerModal` (`start`)

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Set `NEXT_PUBLIC_ADOBE_EMBED_CLIENT_ID` to an ID registered for a different domain, or throttle the network so the `/pdf` request stalls.
2. Open a PDF job and click the pill.
3. **Expected:** after a reasonable time, an error and an automatic switch to download-and-upload.
4. **Actual:** the white loading overlay stays forever.

**Problem:** Nothing puts a deadline on the start-up steps. `hasFailed` only reflects a failed script tag.

**Evidence:**

- **Start-up effect waits silently:** `hooks.ts:128` returns early until `isReady`. If Adobe's ready event never fires, nothing moves on.
- **No deadline on any step:** the awaited `fetchWorkItemContentVersionPdf`, `previewFile`, `getAnnotationManager` and `restore()` (:135-247) have none. The only timer in these files is the submit poll at :333.
- **Failure means script-load failure only:** `useAdobeEmbedSdk.ts:82` sets `hasFailed` only from `status === "error"`.
- **Loader stays up:** `index.tsx:67` shows it while `status === "loading"`.

**Impact:** The editor is stuck, with no clear route to the fallback the ticket requires ("fallback to manual download").

**Fix:** Race `start()` against a timeout (e.g. 60 s) that calls `handleFailure(error, "pdf-viewer.open-timeout")`. Pass an `AbortSignal` to the fetch so it is cancelled on timeout or unmount (see Issue 13).

### 7. Restored replies lose their parent comment, and one bad entry can sink the whole restore

> **PP-2223 status:** Fixed in PP-2223 (`1cc25488c`).

**[File: apps/creative-portal/hooks/usePdfAnnotationDrafts.ts]**

> **In plain terms:** Say a reviewer replies to a comment, then reloads the page or comes back later. The restored reply no longer points at the comment it answered. It may be dropped, or it may stop every restored mark from coming back, with no message to the reviewer.

**Function/Class:** `readdressToThisDocument` / `restore`

**Severity:** medium

**Confidence:** high on the re-addressing; unconfirmed whether Adobe rejects the whole batch or drops only the bad entry.

**Steps to reproduce:**

1. As a reviewer on a PDF job, reply to an existing comment in the viewer. Wait more than 20 s for the autosave.
2. Reload the page and reopen the viewer.
3. **Expected:** the reply shows under its parent comment.
4. **Actual:** the reply is detached or missing. If Adobe rejects the batch, none of the restored marks appear.

**Problem:** Restore gives every entry a new id and points every `target.source` at the document id. For replies, `target.source` is the parent annotation's id.

**Evidence:**

- **Every entry re-addressed:** `usePdfAnnotationDrafts.ts:318-326` sets `id: crypto.randomUUID()` and `target: { ...annotation.target, source: String(seedVersionId) }`.
- **Replies are saved in the draft:** only `creator.name === roleLabel` is filtered (:143-145, :284-290), with no motivation filter.
- **One call for the whole batch:** :383 `await manager.addAnnotations(restored)`. The catch at :389 only reports.

**Impact:** Crash-recovery data is lost for reviewers and QA, who are the people most likely to reply.

**Fix:** Keep an old→new id map, rewrite only `motivation !== "replying"` entries to the document, and remap each reply's `source` through the map. Add replies after their parents, or add entries individually so one failure can't sink the rest.

### 8. The stale-job guard never matches in production, so a stale viewer keeps retrying and reporting

> **PP-2223 status:** Fixed in PP-2223 (`1cc25488c`).

**[File: apps/creative-portal/hooks/usePdfAnnotationDrafts.ts]**

> **In plain terms:** When another job takes over the order while the viewer is open, the autosave is meant to notice and stop. It never notices, so it keeps failing in the background on every change and filling our error tracker. No marks are lost.

**Function/Class:** `save` (stale detection)

**Severity:** low

**Confidence:** high

**Steps to reproduce:**

1. Open a PDF job's viewer as the editor.
2. In another session, reassign or submit so this job is no longer the order's latest.
3. Add marks in the first session.
4. **Expected:** autosave stops after the first rejection.
5. **Actual:** every change or flush POSTs again, fails, and sends a Sentry event. The minimise and tab-close beacons keep firing too.

**Problem:** The check reads `error.message`. Axios always sets that to "Request failed with status code 4xx"; the OMS text is in `error.response.data.error`.

**Evidence:**

- **Check:** `usePdfAnnotationDrafts.ts:236-240` `const message = (error as Error)?.message ?? ""; if (message.includes("not associated with this job"))`.
- **Plain axios, nothing rewrites the error:** `createWorkItemContentVersion` is a plain `axios.post`, and there are no response interceptors in the repo.
- **Test hides it:** `usePdfAnnotationDrafts.test.ts:542-546` rejects with a plain `new Error(...)`.

**Impact:** Sentry noise and wasted requests. The test doesn't check what it claims to.

**Fix:** Read `axios.isAxiosError(error) ? error.response?.data?.error : error.message`, as `getJobActionErrorMessage` already does, and reject with an AxiosError-shaped object in the test.

### 9. Leaving the viewer without collapsing it drops the last unsaved marks from the recovery draft

> **PP-2223 status:** Fixed in PP-2223 (`1cc25488c`).

**[File: apps/creative-portal/hooks/usePdfAnnotationDrafts.ts]**

> **In plain terms:** Autosave waits 20 seconds after the last change. If the editor leaves the viewer any way other than the collapse button, those most recent marks are not saved for recovery. Examples: navigating in the app, the browser Back button, or switching to download-and-upload.

**Function/Class:** listener effect cleanup

**Severity:** low

**Confidence:** high

**Steps to reproduce:**

1. Open the viewer and add a mark.
2. Within 20 s, click "Trouble with the editor? Download and upload instead", or navigate to another page in the app.
3. Return to the job and switch back to the in-browser editor.
4. **Expected:** the mark is restored.
5. **Actual:** the mark is missing.

**Problem:** The unmount cleanup cancels the pending save without flushing.

**Evidence:**

- **Cleanup only cancels:** `usePdfAnnotationDrafts.ts:471-478` cleanup is `clearTimeout(timerRef.current)`.
- **Mode switch doesn't flush:** `onSwitchToFallback` (`ServiceSubmission/hooks.ts:48-51`) only calls `setMode`.
- **No route-change flush:** the diff adds no `routeChangeStart` handler.
- **Contradicts its comment:** the comment at :64-66 claims "every way out of the viewer flushes".

**Impact:** A narrower version of the data-loss risk in Req 6.2.

**Fix:** In a `[]`-dependency unmount effect, fire-and-forget `save()` when `isDirty()`. Flush explicitly in `onSwitchToFallback`.

### 10. The quick saves on minimise and tab close can be silently dropped or out of date

> **PP-2223 status:** Fixed in PP-2223 (`1cc25488c`).

**[File: apps/creative-portal/hooks/usePdfAnnotationDrafts.ts]**

> **In plain terms:** When the editor switches tabs or closes the browser, the page tries a last-second save. For a draft with many marks, the browser refuses that save and nothing is retried. On tab close, the save sends whatever was read last, which may be from much earlier in the session.

**Function/Class:** `sendBeacon`, `onVisibilityChange`, `onPageHide`

**Severity:** low (the marks remain "dirty" and are saved at the next change, collapse or submit, so only the crash-recovery safety net is affected)

**Confidence:** high

**Steps to reproduce:**

1. Make enough marks (e.g. freehand ink) that the draft passes about 48 KB of JSON.
2. Within 20 s of the last mark, close the tab.
3. Reopen the job.
4. **Expected:** all marks are restored.
5. **Actual:** the latest marks are missing.

**Problem:**

- **Size limit:** `sendBeacon` has an approximately 64 KB quota. The payload is the full draft, base64-encoded, and a `false` return is ignored after the debounce timer has already been cleared.
- **Out-of-date snapshot:** `pagehide` sends `latestAnnotationsRef`. That is only refreshed by a full read, and steady marking keeps pushing that read back.

**Evidence:**

- **Timer cleared before the beacon:** `usePdfAnnotationDrafts.ts:441` clears it, then the beacon is sent.
- **Result ignored:** :416-425 `navigator.sendBeacon?.(…)`, with no retry and no re-arm on `false`.
- **Snapshot sent as-is:** :453-466 sends `latestAnnotationsRef.current`, which is only set at :190. The comment at :406-408 acknowledges "the last read is sent as it stands."

**Impact:** The "on tab close (beacon)" guarantee in the PR description is weaker than stated.

**Fix:**

- On `visibilitychange` → hidden, the page is still alive, so call the normal awaited `save()` there.
- Keep the beacon for `pagehide` only, send it without base64, and re-arm the timer when it returns `false`.
- Keep `latestAnnotationsRef` fresh with a short-debounced read.

### 11. Opening a PDF rewrites valid dates on the customer's existing comments and adds dates to links

> **PP-2223 status:** Fixed in PP-2223 (`1c9c5a31f`).

**[File: packages/shared/api/utils/pdf/ensureAnnotationDates.ts]**

> **In plain terms:** When the viewer opens a PDF, it fills in missing comment dates so Adobe doesn't crash. It also replaces dates on the customer's own comments that are valid but written in a shorter form, and stamps dates onto hyperlinks. Those changed dates then end up in the file delivered to the customer.

**Function/Class:** `fillDates` / `ensureAnnotationDates`

**Severity:** low

**Confidence:** high on the code path. That hex strings are not `PDFString` comes from pdf-lib's documented class split; its source is not on disk.

**How to spot it:** Open a PDF whose original comments carry a short date such as `D:20230101`, or which contains hyperlinks, submit it, and inspect `/M` and `/CreationDate` in the EditedCopy. The originals are replaced with the version timestamp.

**Problem:** Any date that isn't a 14-digit `PDFString` is replaced, and every annotation subtype is processed.

**Evidence:**

- **Pattern too strict:** `ensureAnnotationDates.ts:7` requires all 14 digits.
- **Replacement condition:** :84-93 replaces unless `current instanceof PDFString && isReadablePdfDate(...)`.
- **No subtype filter:** `collectAnnotations.ts:26-31` keeps every `PDFDict`.
- **Runs on every open:** `getWorkItemContentVersionPdf.ts:172-178`.

**Impact:** Conflicts with Req 5.1.1 ("pre-existing annotations should retain their metadata"). It also forces a pdf-lib save of any PDF containing a link on every open (see Issue 12).

**Fix:** Decode `PDFHexString` too. Only replace dates that are missing or that `new Date` cannot parse. Skip `Link`, `Widget` and `Popup` subtypes.

### 12. The PDF route re-parses and copies up to 50 MB on every open, and doesn't report or cache a failed orientation check

> **PP-2223 status:** Partly fixed in PP-2223 (`c4736d57f`); memory part deferred.

**[File: apps/creative-portal/api/workItemContentVersion/[id]/pdf/getWorkItemContentVersionPdf.ts]**

> **In plain terms:** Every time anyone opens a large PDF, the server re-reads the whole document and holds several copies in memory. A few editors opening big files at once could slow down or crash the portal's server. If a document can't be checked for orientation, the server retries the expensive check on every open and never alerts us.

**Function/Class:** default handler

**Severity:** medium

**Confidence:** high

**How to spot it:** Not user-reproducible in normal use. Under load, open a ~45 MB PDF in several tabs and watch server memory. Rough totals:

- normal open: about 150 MB of buffers plus pdf-lib's object graph per request;
- a rotated cache miss: roughly 300 MB or more.

**Problem:** `ensureAnnotationDates` always runs a full `PDFDocument.load`, with no caching. The route copies the buffer in and out of it even when nothing changed. Orientation detection that fails is neither cached nor reported to Sentry, while its sibling steps are.

**Evidence:**

- **Copies around the date step:** `getWorkItemContentVersionPdf.ts:173-177` `Buffer.from(await ensureAnnotationDates(new Uint8Array(buffer), …))` copies on the way in and again on the way out, even when unchanged.
- **Always a full load:** `ensureAnnotationDates.ts:67` always calls `PDFDocument.load`.
- **Only successes are cached:** `setCachedRotationMap` sits inside the `try` (:138).
- **Failure is only logged:** :139-145 `logger.warn(...)` only, while :158-163 and :179-184 call `reportError`.
- **Size warning on every open:** there is no `responseLimit` config, so Next logs "API response … exceeds 4 MB" on every large open (`pages/api/workItemContentVersion/[id]/pdf.ts`).

**Impact:** Memory and CPU pressure on the shared Next server. A silent orientation outage would cost a pdfjs parse on every open.

**Fix:**

- Cache "dates OK / needs dates" per version id.
- Pass views, not copies: `new Uint8Array(buf.buffer, buf.byteOffset, buf.length)`. Skip `Buffer.from` when the result `===` the input.
- Cache failed detections as `{}`.
- Add `reportError(error, { operation: "pdf-viewer.rotation-detect", … })`.
- Export `config = { api: { responseLimit: false } }`.

### 13. The viewer keeps several copies of the PDF in browser memory and never cancels its download

> **PP-2223 status:** Fixed in PP-2223 (`3fd728d66`).

**[File: apps/creative-portal/components/organisms/modals/PdfViewerModal/hooks.ts]**

> **In plain terms:** For a large PDF, the browser holds two or three extra copies of the file for as long as the viewer is open. If the editor leaves before it has finished loading, the whole download carries on in the background anyway.

**Function/Class:** `usePdfViewerModal`; `fetchWorkItemContentVersionPdf`

**Severity:** low

**Confidence:** high

**How to spot it:** Code health, not user-reproducible.

- `documentRef` (:66, written at :142) and `viewRef` (:70, written at :167) are written but never read anywhere in the repo.
- The buffer is also copied at :149 (`new File([buffer], …)`) and :220 (`buffer.slice(0)`).
- `services/workItemContentVersion/index.tsx` `fetchWorkItemContentVersionPdf` calls `fetch` with no `AbortSignal`, and unmount only discards the result (:138).

**Problem:** Dead references keep up to 50 MB alive, and the download cannot be cancelled.

**Impact:** Unnecessary memory use on editors' machines, and wasted bandwidth.

**Fix:** Delete `documentRef` and `viewRef`, and their misleading comment at :59-65. Hand Adobe the original buffer. Accept an `AbortSignal` in `fetchWorkItemContentVersionPdf` and abort it on unmount or timeout.

### 14. Submit failures on the jobs page now show raw server error text

> **PP-2223 status:** Resolved in PP-2223 (`49cc7901a`).

**[File: apps/creative-portal/components/pages/jobs/utils.ts]**

> **In plain terms:** When any submit fails with a client-side error, the editor used to see "Please try again". Now they see whatever text the server returned, which may be a technical message.

**Function/Class:** `getJobActionErrorMessage`, `showJobActionError`

**Severity:** low

**Confidence:** high on the path; unverified whether OMS returns internal-looking strings.

**How to spot it:** `utils.ts:524-537` returns `error.response.data.error` for any 4xx whose body is a string. `pages/jobs/hooks.ts:136` passes it into the `Submitted` toast branch (`utils.ts:314` `text: message ?? "Please try again."`). Neither function has a test.

**Problem:** The change was made to show the new 422 message, but it applies to every 4xx on submit.

**Impact:** Possible confusing or technical copy in a user-facing toast.

**Fix:** Mark user-facing errors explicitly (e.g. a `code` or `userMessage` field on the 422) and show only those. Add tests: non-axios, 5xx, 4xx without a message, 4xx with a message.

### 15. Off-platform PDF orders now get an internal copy and can be refused

> **PP-2223 status:** Fixed in part in PP-2223 (`b171fc94e`); strip kept off platform on purpose.

**[File: apps/creative-portal/api/utils/jobs/postAddWorkItemContentVersion.ts]**

> **In plain terms:** For orders whose file isn't stored on the platform, the new PDF steps still run on submit. They save an extra internal file that is never used, and in rare cases they block a submission that used to go through.

**Function/Class:** `postAddWorkItemContentVersion` (non-streaming branch)

**Severity:** low

**Confidence:** high on the gating; the count of off-platform PDF orders is unknown.

**How to spot it:** `postAddWorkItemContentVersion.ts:92` gates only on `!useStreaming && finalVersionFile`, and both helpers check only `workItemFormat`. `postSubmitJob.ts:121` then discards the content for off-platform orders (`...(isOnPlatform ? { workItemContent } : {})`).

**Problem:** The PDF steps aren't gated on `isOnPlatform`.

**Impact:** An orphan `EditedCopyInternal`, and a possible 422 on a file this path never delivers.

**Fix:** Guard the PDF steps with `isOnPlatform`. Consider also writing `EditedCopyInternal` only after the strip succeeds, so a rejected submit doesn't leave an internal copy behind (`storePdfInternalCopy` currently runs before a strip that can throw 422).

### 16. The public create-version endpoint now accepts `EditedCopyInternal` from the browser

> **PP-2223 status:** Fixed in PP-2223 (`7a8fdedc9`).

**[File: apps/creative-portal/api/workItemContentVersion/createWorkItemContentVersion/schema.ts]**

> **In plain terms:** Someone assigned to a job who hand-crafts a request could store a fake internal copy. The next internal person would then open it, bypassing the author clean-up. This needs deliberate misuse by an insider.

**Function/Class:** create schema

**Severity:** low

**Confidence:** high

**How to spot it:** `schema.ts:9-11` `versionType: Yup.mixed().oneOf(Object.values(WorkItemContentVersionType)).optional()`. The enum gained `EDITED_COPY_INTERNAL` in this PR, and the handler (`createWorkItemContentVersion.ts:24-25`) forwards the body unchecked. The only new client use sends `PdfAnnotations`.

**Problem:** Server-only version types can be written from the browser.

**Impact:** Integrity risk for the internal copy that `usePdfJobSeedVersion` prefers.

**Fix:** Refuse `EditedCopyInternal` on this route; allow `PdfAnnotations`. The server writes internal copies itself.

### 17. The Adobe viewer script is downloaded on every job panel, including non-PDF jobs

> **PP-2223 status:** Fixed in PP-2223 (`285de5e18`).

**[File: apps/creative-portal/hooks/usePdfSubmissionMode.ts]**

> **In plain terms:** Once the Adobe ID is configured, opening any job loads Adobe's third-party script, even for Word or text jobs that can never use it. If a browser blocks it, we report a viewer error for users who never saw a PDF.

**Function/Class:** `usePdfSubmissionMode` → `useAdobeEmbedSdk`

**Severity:** low

**Confidence:** high

**How to spot it:**

- `usePdfSubmissionMode.ts:38` computes `isPdf`, then :40-41 calls `useAdobeEmbedSdk()` unconditionally.
- `useAdobeEmbedSdk.ts:50-53` runs `useScript(isConfigured ? ADOBE_EMBED_SDK_URL : null, …)`.
- `JobManagement/index.tsx:75` calls the hook for every job.
- The script loads once per page, deduplicated by id.

**Problem:** The script load isn't gated on the job being a PDF.

**Impact:** A needless third-party download and telemetry on non-PDF pages, plus misleading Sentry events.

**Fix:** `useAdobeEmbedSdk({ enabled: isEnabled && isPdf })`.

### 18. Missing tests for new code (project rule)

> **PP-2223 status:** Fixed in PP-2223: tests added with each fix.

**[File: multiple]**

> **In plain terms:** Several new pieces have no automated tests. These include the identifier re-stamp from Issue 3 and the new submit error message. Future changes to them won't be caught.

**Function/Class:** see list

**Severity:** medium (DEVELOPMENT_WORKFLOW.md:116-121 and CLAUDE.md: "Every PR must include tests for new code")

**Confidence:** high

**How to spot it:** Code health. Grep the test files for each symbol:

- `restampPdfIdentifier.ts`: only `vi.mock`-ed (`patchJob.test.ts:114`). Its three siblings each got a test.
- `postAddWorkItemContentVersion.ts` streaming branch: never exercised. The non-streaming branch is partly covered via `vi.importActual` (`patchJob.test.ts:720`), but the strip → restamp order isn't asserted.
- `getJobActionErrorMessage` and the new `message` fallback in `showJobActionError` (`components/pages/jobs/utils.ts:314, 524`): `utils.test.ts` is untouched.
- `fetchWorkItemContentVersionPdf`: filename parsing and the `"document.pdf"` fallback are only mocked.
- `rotationMapCache.ts`: the eviction at 200 entries is never reached.
- `detectRotationMap`: no case with a non-zero `page.rotate` or a text-less document (would have caught Issue 2).
- `stripPdfInternalAuthors`: no encrypted + role-label case (would have caught Issue 5).
- `usePdfAnnotationDrafts`: the stale-guard test uses a plain `Error`, not an AxiosError (masks Issue 8).

**Problem:** These gaps sit right where the confirmed bugs are.

**Impact:** The confirmed issues above went uncaught.

**Fix:** Add the listed tests, and make each fail on the current code before fixing.

### 19. Conventions and code-health notes (CLAUDE.md, Cursor rules, naming, comments)

> **PP-2223 status:** Fixed in PP-2223, with the skips listed in Fix status.

**[File: multiple]**

> **In plain terms:** None of these change what users see. They make the code harder to maintain: explanatory comments that will go stale, small duplications that have already drifted apart, and a few process steps skipped.

**Function/Class:** various

**Severity:** low

**Confidence:** high (every item re-checked against the written rules; refuted items are listed below)

**How to spot it:** Code health, not user-reproducible.

- **Comments.** Comment lines are ~21.5% of new source lines (798 of 3,718; 24.8% of non-blank lines). The team's usual norm is 2 to 8%, though recent PP-2051/2053 files already run 19 to 41%. No written rule caps density, but the content is the real issue: comments narrate history rather than describe the code.
  - **Real order numbers (customer data):**
    - `stripPdfInternalAuthors.ts:40`, `getWorkItemContentVersionPdf.ts:150` and `readAnnotationAuthors.ts:8` (order 20511);
    - `usePdfAnnotationDrafts.ts:125, 204` and `ensureAnnotationDates.ts:55` (order 21654);
    - several test files.
  - **Meeting history:**
    - `PdfViewerModal/hooks.ts:147`: "agreed on PP-2176, question 3 of 28 Sep";
    - `Submission/index.tsx:104`;
    - `stripPdfInternalAuthors.ts:98`: "worked before PP-2176".
  - **Ticket clause numbers and Figma node ids:** about 15 across source files.
  - **Shared enum points at a PR doc:** `packages/shared/api/workItemContentVersion/enums.ts:21` references `docs/PP-2176-pdf-in-browser-markup.md`.
  - **Examples of pure restatement:** `packages/shared/api/utils/pdf/consts.ts` (10 comment lines for 1 constant) and `config/pdfSubmission.ts` (50%).
  - **Fix:** keep the "why" JSDoc; describe the property instead of the order ("some PDFs have a page tree pdf-lib cannot walk"); drop dates, "agreed" and "used to"; leave ticket references to the PR.
- **Duplication.**
  - Streaming vs non-streaming PDF steps (root cause of Issue 3).
  - `brief-eye.svg` duplicates `packages/wysiwyg/src/assets/svg/brief-eye.svg` (same glyph, 48 vs 24 viewBox).
  - `usePdfJobSeedVersion.ts:69-75` inlines the `EDITED_COPY_INTERNAL` comparison that this PR adds as `isInternalCopyContentVersion` in `workItemContentVersionRules.ts:30-35`.
  - The pdf-lib load + `isEncrypted` guard is copied in `applyRotationMap.ts` and `ensureAnnotationDates.ts:67-76`, with identical comments.
  - `rotationDetect.ts:39-44` re-implements `normaliseDegrees`, which it imports on line 3.
  - `finalVersionFile as FormidableFile` is cast three more times in `postAddWorkItemContentVersion.ts` (~99, 107, 113); use one local const.
- **Structure (advisory; no rule is broken).**
  - `ContentPillButton` is now used by two organisms but lives in `HTMLContentPill/partials/` and still uses `HTMLContent*` names. Promote it to `components/molecules/ContentPillButton`.
  - `config/` previously held only `appRoutes.ts` and `swrKeys.ts`. It now holds Adobe SDK typings (`config/adobeEmbed.ts`, imported into `@types/global.d.ts`), draft logic with tests (`config/pdfAnnotationDraft.ts`) and enums (`config/pdfSubmission.ts`). `@types/` and `utils/` fit better.
  - `services/workItemContentVersion/index.tsx:133` hand-appends `/pdf` to `apiRoutes.workItemContentVersionById(id)`. It is the only such case; add an `apiRoutes` entry.
  - The `PdfContentPill/hooks.ts:43` comment says the Brief is built "as `JobBrief` builds it", but it adds `workItemStyleIdentifier`, so Stylus-ID orders show the style guide in the viewer and not in the sidebar.
- **Process.**
  - No `yarn bump-packages` commit (PR template line 33, `DEVELOPMENT_WORKFLOW.md:226`, `docs/CONTRIBUTING.md:55`), although recent develop merges also skip it.
  - Commit types `fix/`, `chore/` and `docs/` on a `feature/` branch vs CLAUDE.md "`<type>` matches the branch prefix" (cosmetic under squash-merge).
  - `packages/shared/package.json` has unrelated re-sort churn.
  - `patchJob.test.ts:717`: the title "writes the delta as the %s reviewer's first version" contradicts its PDF assertion that the internal copy is written first.
  - `ServiceSubmission/index.tsx:43-44` appends `pdfSubmission` to an already-unordered destructuring.
- **Small hygiene.**
  - `readOriginalPdfAuthors.ts:19`: module-level `Map` with no eviction, unlike `rotationMapCache`'s 200-entry cap.
  - The `X-Rotations` header (`getWorkItemContentVersionPdf.ts:196`) reports rotations even when they weren't applied; there is no consumer.
  - Accessibility: the full-screen viewer doesn't move focus in on open or back to the pill on collapse. `FullscreenModal`'s missing dialog semantics predate this PR.

**Checked and compliant:**

- **Naming:** no single-letter or shorthand variable names anywhere.
- **Types:** `VoidFunction` used; props interfaces sorted required-first; `FC<Props>` everywhere; folder structure correct; no `any`, `@ts-ignore`, non-null `!`, `console.log` or TODO without a ticket.
- **Styling:** styled-component hygiene is clean, with theme tokens throughout.
- **Env var:** follows PP-2119 fully (`env.js` `.notRequired()`, `KEYS`, `.env.example`).
- `**RawButton isUnderlined`:** has a story, a test and `stopForwarding`.
- `**PdfSubmissionModeLink` story:** title follows the sentence-case rule.

**Refuted during verification (not issues):**

- Custom-hook calls in `index.tsx`: CLAUDE.md bans React primitives, and existing precedent calls custom hooks there.
- `MDASH` constant in `index.tsx`: `consts.ts` is optional, and there is precedent.
- `normalise` vs `normalize` spelling: no rule.
- The Brief icon's accessible name: resolved through `PopoverOnClick`'s `role="button"`.
- Importing a sibling's partial as a rule violation: common precedent; kept above as advice only.

**Problem / Impact / Fix:** see each bullet.

---

## Open Questions

- **Does pdfjs work in the production standalone build?** `forEachPdfjsPage.ts:17` imports `pdfjs-dist/legacy/build/pdf.mjs` with no `GlobalWorkerOptions.workerSrc`. `@proofed/shared` is transpiled, the output is `standalone`, and there are no externals. If the fake worker's `pdf.worker.mjs` isn't traced into the build, three things fail silently: rotation is skipped, original authors come back empty so customer comments get relabelled "Proofed", and the "cannot read" branch lets files through. Please run `next build` + `next start` and do one real PDF open and submit.
- **Do restored marks block submit?** `PdfViewerModal/hooks.ts:236-238` sets `hasUnsavedMarksRef = true` before `onAnnotationEvent` checks `isRestoringRef`, so restore's `addAnnotations` events arm the 8 s submit wait. If Adobe doesn't fire `SAVE_API` for marks added through the API, a returning editor who opens and collapses without clicking inside the document gets "not been saved yet" on submit. Is this covered by real-Adobe testing?
- **Does Adobe's save keep the order's `/UUID`, and does it keep an encrypted file encrypted?** The first determines how severe Issue 3 is in practice; the second determines the in-browser half of Issue 5.
- **Unstripped files when both parsers fail:** `stripPdfInternalAuthors.ts:93-105` deliberately stores the file unstripped when both pdf-lib and pdfjs fail to read it. Has product accepted that trade-off against the rule "the client never receives a file showing an editor's real name"?
- **Admin submit-on-behalf, server side:** it now stores an `EditedCopyInternal` and rewrites authors (`patchJob.ts:162` → `postAddWorkItemContentVersion`). Does "Admin view is currently unchanged" cover only the UI, or should these steps skip admin submits?
- **Customer portal access by id:** the customer portal's `/api/workItemContentVersion/[id]` and `searchAndDownloadFileBasedWorkItemContentVersion` fetch any id without a `versionType` guard. Can a customer reach the `EditedCopyInternal` on their own order, or does OMS's client-API check block it? A defensive `isInternalContentVersion` check there would be cheap.
- **Pre-existing, not introduced here:** the `/pdf` route, like the existing `[id]` route, only checks for a session. The OMS job-assignment check on `GET {id}` is commented out (`WorkItemContentVersionsController.cs:~282`). The new route adds another way to reach that gap, so it is worth a backend ticket.
- **Where the design doc should live:** should the 678-line `docs/PP-2176-pdf-in-browser-markup.md` be in the repo, or on the ticket or Confluence?
- **Test order numbers in committed files:** are internal order numbers acceptable in committed tests and comments?

---

## Validation Checks


| Check                     | Result     | Notes                                                                                                                                                                     |
| ------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `npx turbo run test`      | ⏭️ Skipped | Skipped: user opted out. The PR description reports shared 1997/1998 (locale-dependent `formatWordQuantity`) and creative 372/375 files (3 known hangs, also on develop). |
| `npx turbo run typecheck` | ⏭️ Skipped | Skipped: user opted out.                                                                                                                                                  |
| `npx turbo run lint`      | ⏭️ Skipped | Skipped: user opted out.                                                                                                                                                  |
| `npx turbo run build`     | ⏭️ Skipped | Skipped: user opted out. The PR says the creative-portal build passed. The pdfjs question above needs a production build either way.                                      |


---

## Tests

- ✅ Substantial new unit tests: `usePdfAnnotationDrafts` (868 lines), `PdfViewerModal/hooks`, `getWorkItemContentVersionPdf`, `strip`/`store`/`readOriginal*`, shared pdf utils, `workItemContentVersionRules`, `env.test.ts`.
- ❌ No tests for `restampPdfIdentifier`, the streaming submit branch, `getJobActionErrorMessage`, `fetchWorkItemContentVersionPdf` or `rotationMapCache` eviction (Issue 18).
- ❌ Missing edge cases, each of which would have caught a confirmed bug: text-less or pre-rotated PDFs (Issue 2), encrypted + role label (Issue 5), AxiosError-shaped rejection (Issue 8), replies in the draft (Issue 7), submit before restore (Issue 1).
- ⚠️ pdfjs is mocked in the route tests, and nothing exercises the production bundle (Open Question 1).
- ⏭️ Validation suite not run (user opted out).

### Suggested manual QA script

1. **(Issue 1)** Autosave some marks, reload, open the viewer, collapse it immediately, and submit. The submitted file must contain the marks, or submit must refuse until the document is ready.
2. **(Issue 2)** Open an image-only scan whose pages carry `/Rotate 90`. It must display upright.
3. **(Issue 3)** After an in-browser submit, the next assignee downloads the EditedCopy, adds a comment in Acrobat and uploads it with the fallback. It must be accepted.
4. **(Issue 4)** Open the viewer with DevTools open. No response may contain a `blob.core.windows.net` SAS URL.
5. **(Issue 5)** Submit an encrypted PDF with one comment, both via the fallback and in the browser. It must not be refused, or the message must offer a working alternative.
6. **(Issue 6)** Set a wrong-domain Adobe client ID. The viewer must fail over to download-and-upload within a reasonable time.
7. **(Issue 7)** As a reviewer, reply to a comment, wait 25 s and reload. The reply must be restored under its parent.
8. **(Issues 9, 10)** Add a mark, then within 20 s either switch to download-and-upload or close the tab. On return, the mark must be restored.
9. **(Issue 11)** A PDF with hyperlinks and short-form comment dates keeps its original dates in the delivered file.
10. **(Issue 17)** Open a DOCX job with the client ID set. The Network tab must show no `acrobatservices.adobe.com` request.
11. **(Req 3)** A PDF over 50 MB shows download-and-upload only.
12. **(Req 5.1.2)** The customer download shows "Proofed" on every Proofed mark and keeps the original customer authors.

---

## Summary


| Aspect           | Status                                                                                                                                                             |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Correctness      | ⚠️ One high (Issue 1) and several medium defects (Issues 2, 3, 5, 6, 7)                                                                                            |
| Regression risk  | ⚠️ Medium: encrypted PDFs (Issue 5), all-submit error text (Issue 14), off-platform orders (Issue 15)                                                              |
| Tests            | ⚠️ Extensive, but gaps line up with the confirmed bugs (Issue 18)                                                                                                  |
| Accessibility    | ⚠️ Focus is not managed for the full-screen viewer (minor; the modal's dialog gaps predate this PR)                                                                |
| Error handling   | ⚠️ No open timeout (Issue 6); orientation failure not reported (Issue 12)                                                                                          |
| Security         | ⚠️ SAS URL still reaches the browser (Issue 4); client can write `EditedCopyInternal` (Issue 16). `/security` not yet run per PR checklist; required before merge. |
| Code quality     | ⚠️ Comment volume and history narration; duplicated submit branch                                                                                                  |
| Validation suite | ⏭️ Skipped (user opted out)                                                                                                                                        |
| Mergeable state  | ✅ GitHub reports clean (validation not run)                                                                                                                        |


---

## Recommendation

**Approve with suggestions.** No blocker-class finding was confirmed, but the High and the listed Mediums should be fixed before merge.

Pre-merge asks:

1. **Fix Issue 1 (high):** make submit wait for, or refuse before, the viewer's restore, and add a test.
2. **Fix Issue 3:** one shared PDF-processing helper used by both submit branches, with the restamp and a streaming-branch test.
3. **Fix Issue 2:** skip rotation correction for text-less documents; add tests for pre-rotated and text-less PDFs.
4. **Fix Issue 5:** don't refuse encrypted PDFs over our own role labels, and make the message offer a working way out.
5. **Fix Issue 4:** stop reading `bytes` through the SAS-returning endpoint, and correct the PR description's security claim.
6. **Fix Issue 6:** add an open timeout that falls back to download-and-upload.
7. **Fix Issue 7:** keep reply parents when restoring.
8. **Run `/security`** (unchecked in the PR) and a production build smoke test with one real PDF open and submit (pdfjs worker, Open Question 1).
9. **Re-run validation** (`test` / `typecheck` / `lint` / `build`); it was skipped in this review.

Follow-ups, can be separate PRs: Issues 8 to 19, especially Issue 12's server memory, Issue 18's test gaps and trimming the history-narrating comments.