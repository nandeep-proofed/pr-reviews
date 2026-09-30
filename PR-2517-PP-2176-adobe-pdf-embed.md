# PR Review: feature/PP-2176: Mark up PDF jobs in the browser with Adobe PDF Embed

**PR:** https://github.com/Proofed/B2BWebserver/pull/2517
**Jira:** https://proofed.atlassian.net/browse/PP-2176
**Status:** In Progress

---

## What this means for users (non-technical summary)

1. **Customers' own comments come back renamed.** If a customer sends a PDF that already has their own comments, every one of those comments says "Proofed" after we submit. The ticket says original comments must keep their author.
2. **Protected PDFs can become unreadable.** If a customer sends a password-protected or permission-restricted PDF, the file we deliver can fail to open.
3. **A job can pick up another job's document.** An editor who switches between two PDF jobs quickly, or leaves the viewer while it is still loading, can have the wrong or unmarked PDF placed ready to submit.
4. **Very late marks can be missed at submit.** An editor who clicks Submit a moment after their last mark, before Adobe has finished saving, sends the previous copy. The job still reports success.
5. **Some uploads that used to work are now refused.** When a PDF our tools can't fully read also has comments in it, the submission is blocked. The editor gets a generic error instead of the intended explanation.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
| --- | --- | --- |
| 1.1–1.3 In-browser button for PDF jobs only | `PdfContentPill` in `ServiceSubmission`. Mode is gated on `WORK_ITEM_FORMAT.PDF` in `usePdfSubmissionMode` | ✅ Addressed |
| 2.1–2.2 "Trouble with the editor?" / "Prefer to edit in your browser?" links | `PdfSubmissionModeLink`, placed in both modes | ✅ Addressed |
| 3.1 Embed only for PDFs under 50 MB | `PDF_EMBED_SIZE_CEILING_BYTES` in the panel, and again in the PDF route (413) | ✅ Addressed |
| 4.1–4.4 Annotation tools and APIs on, existing marks visible, Download on, fit-width | `AUTHORING_PREVIEW_OPTIONS` | ✅ Addressed |
| 5.1.1 Pre-existing annotations keep their metadata | `normaliseAnnotationAuthors` rewrites **every** author to "Proofed", the customer's own included (Issue 1) | ❌ Missing |
| 5.1.2 New annotations labelled "Proofed" | The customer copy is stripped to "Proofed" on submit | ✅ Addressed |
| 5.1.3 Customer annotations stored at each job submission | `EditedCopy` on every submit | ✅ Addressed |
| 5.2 Internal labels per job (AI / Editor / Reviewer / QA / Admin), shown to internal users | Roles for people via `EditedCopyInternal` + Adobe author. AI labels deferred by Adam (comment 76970) | ⚠️ Partial (agreed deferral) |
| 6.1 Each submission stores the whole PDF with customer annotations | `EditedCopy` | ✅ Addressed |
| 6.2 Annotations saved after each change | `PdfAnnotations` drafts (20 s debounce, collapse, submit). The tab-close path is weak (Issue 8) | ⚠️ Partial |
| 7 Pages in correct orientation, detected once | `rotationDetect` + `applyRotationMap`, cached per version, stored file never changed | ✅ Addressed |
| 8 Editor job history unchanged | `workItemContentVersionRules` excludes drafts and internal copies from every list (callers checked) | ✅ Addressed |
| 9 Admin view unchanged | Admin submit modal keeps the upload. Viewer there deferred by Adam (76970) | ✅ Addressed |
| Validation 3: PDF never served from a public URL | The SAS URL stays server-side. Bytes are served behind the session | ✅ Addressed |
| Validation 6: unreadable file never wipes comments | Guard in `usePdfAnnotationDrafts.save` against empty reads | ✅ Addressed |
| Validation 7: client never receives an editor's real name | The customer copy is stripped to "Proofed" | ✅ Addressed |

**Beyond ticket scope (all related, all small):** annotation date fill for AI-written marks, own-mark tracking by id (Adobe/Caret workaround), unparseable-PDF fallback, `FullscreenModal` taller header variant. No unrelated refactors, except the alphabetical re-sort in `packages/shared/package.json`.

---

## Architecture Analysis

The viewer is a full-screen modal (`PdfViewerModal`) that the job panel (`PdfContentPill`) mounts. It fetches the bytes through a new session-protected route, which resolves the SAS blob on the server, corrects page rotation and fills missing annotation dates.

Marks are authored with the user's role through Adobe's profile callback. They are autosaved as small JSON `PdfAnnotations` versions holding only the current job's own marks, tracked by id.

On submit, the server stores two versions:
- `EditedCopyInternal`, with roles kept, which the next internal job opens;
- `EditedCopy`, with every author rewritten to "Proofed" and the identifier restamped, which the customer receives.

The next job's viewer opens the previous internal copy unless another job (e.g. an AI job) produced a version after it.

Overall the design is coherent. It reuses shared primitives well (`FullscreenModal`, `Brief`, `PopoverOnClick`, `useScript`, `reportError`) and keeps the new version types out of every existing list. The weak spots are:
- the author rewrite is too broad (Issue 1);
- no encryption guard (Issue 2);
- component lifetime across job switches and unmounts (Issue 3);
- the submit reads a snapshot taken before its own wait (Issue 4).

---

## Issues Found

### 1. The customer's own comments are relabelled "Proofed"

**[File: packages/shared/api/utils/pdf/normaliseAnnotationAuthors.ts]**

> **In plain terms:** When a customer sends a PDF that already has their own comments, every one of those comments comes back saying "Proofed" after any editor submits. The ticket requires the customer's original comments to keep their original author.

**Function/Class:** normaliseAnnotationAuthors (called by stripPdfInternalAuthors on every PDF submission)

**Severity:** high

**Confidence:** high

**Steps to reproduce:**

1. Create a PDF order whose original file contains a comment authored by "Jane Smith" (e.g. added in Acrobat).
2. As the editor, submit the job (in-browser or by upload).
3. Download the delivered file and open the comments pane.
4. **Expected:** Jane's comment still says "Jane Smith" (requirement 5.1.1).
5. **Actual:** It says "Proofed".

**Problem:** The rewrite has no way to tell pre-existing marks from ones Proofed added. It overwrites `/T` on every annotation.

**Evidence:** `normaliseAnnotationAuthors.ts:70-72`: `if (annotation?.has(AUTHOR_KEY)) { annotation.set(AUTHOR_KEY, author); }`. This runs from `stripPdfInternalAuthors.ts:57-61` for every PDF-format submission, whether it comes from embed, fallback or admin upload. The file does not exist on `origin/develop`, so this is new behaviour.

**Impact:** Breaks requirement 5.1.1, and customers lose the attribution of their own comments. The same breadth causes Issue 5.

**Fix:** Only rewrite marks that are not in the order's original version. Match by `/NM` (Adobe keeps it; verified on orders 21654/21658) plus the page number against the `Original` version. Alternatively, rewrite only authors that are internal role labels (`ROLE_LABELS` values) or not present in the original.

```ts
const originalNames = await readAnnotationNames(originalBytes); // page + /NM
if (!originalNames.has(key(pageIndex, nm))) annotation.set(AUTHOR_KEY, author);
```

### 2. Protected (encrypted) PDFs can be delivered unreadable

**[File: packages/shared/api/utils/pdf/normaliseAnnotationAuthors.ts]**

> **In plain terms:** If a customer sends a PDF with permission restrictions, which many corporate PDFs have, the file we deliver after submission can fail to open with a password error.

**Function/Class:** normaliseAnnotationAuthors

**Severity:** high

**Confidence:** high (the verifier reproduced it with a permission-encrypted fixture: pdfjs read the input, but the output failed with "No password given")

**Steps to reproduce:**

1. Create an order whose original PDF has owner-password permissions (empty user password).
2. Submit the job as the editor.
3. Open the delivered `EditedCopy`.
4. **Expected:** It opens like the original.
5. **Actual:** Readers refuse it or ask for a password.

**Problem:** The file is loaded with `ignoreEncryption: true` and always re-saved. pdf-lib keeps the `/Encrypt` trailer but writes new, unencrypted object streams, so readers decrypt plain text into garbage.

**Evidence:** `normaliseAnnotationAuthors.ts:42-44` has `PDFDocument.load(bytes, { ignoreEncryption: true })`, and line 76 has an unconditional `return document.save();`. Encrypted PDFs reach this path: `addUUID.ts:34-45` deliberately lets them through without an identifier.

**Impact:** A corrupted deliverable. Before this PR these files passed through untouched.

**Fix:** When `document.isEncrypted`, don't save. Use the pdfjs author read instead: pass through if nothing needs stripping, otherwise block with the actionable message. Also skip `save()` when no author changed. `ensureAnnotationDates` and `applyRotationMap` load the same way; the served bytes can become the submission, so apply the same guard there.

### 3. The wrong or unmarked PDF can be placed ready to submit after leaving or switching jobs

**[File: apps/creative-portal/components/organisms/modals/PdfViewerModal/hooks.ts]**

> **In plain terms:** An editor who opens the in-browser editor and then leaves it before it finishes loading (by switching to download-and-upload, or to another job) can end up with the untouched original PDF attached to the form. They may also see a false error message. In a less common case, switching between two PDF jobs keeps the first job's document in the editor, and the first job's file can be submitted for the second job.

**Function/Class:** usePdfViewerModal (start effect); JobSidebar → JobManagement

**Severity:** high

**Confidence:** high for the unmount path; medium for the cached job switch (it needs two active PDF jobs, recently viewed)

**Steps to reproduce:**

1. As an editor with a large PDF job, click the in-browser editor pill.
2. While it is still loading, collapse and click "Download and upload instead" (or open another job).
3. Wait for the load to finish, then look at the edited-copy field.
4. **Expected:** Nothing is attached, and there is no error.
5. **Actual:** The unmarked original is attached as the edited copy, an error toast appears, and Sentry logs `pdf-viewer.open`.

**Problem:** The start effect has no cancellation. After the fetch resolves, it hands the file to the form before checking that the viewer still exists. It then throws and runs the failure handler on an unmounted tree. The submission state lives in `JobManagement`, which survives, so the stale file lands after the per-job reset has already run. Separately, `<JobManagement>` has no `key` and the start effect is guarded by `hasStartedRef`. So on a cached job switch the viewer instance, and its SAVE callback, stays bound to the first job's document.

**Evidence:**
- `PdfViewerModal/hooks.ts:97-225`: no cleanup. `:113` calls `onSubmissionFileChange(new File([buffer], ...))` before `:121-125` throws on the missing container.
- `:98` has `if (!isReady || !clientId || hasStartedRef.current) return;`.
- `JobSidebar/index.tsx:61-73` renders `<JobManagement>` without a key.
- `usePdfSubmissionMode.ts:79-86` only resets on `[isEmbedAvailable, jobId]`.

**Impact:** The wrong document can be submitted; users see a false error; Sentry gets noise.

**Fix:** Add a `cancelled` flag in the effect cleanup and bail after every `await`; pass an `AbortSignal` to the fetch. Key the viewer (or `JobManagement`) by job and seed version:

```tsx
<JobManagement key={activeJobId} ... />
// or
<PdfViewerModal key={`${attribution.jobId}-${seedVersionId}`} ... />
```

### 4. Submit can send the copy from before the save it waits for

**[File: apps/creative-portal/components/organisms/sidebars/contents/JobManagement/partials/Submission/hooks.ts]**

> **In plain terms:** If an editor collapses the editor and clicks Submit very quickly, before Adobe has finished saving their last marks, the job is submitted with the previous copy. It reports success, and the latest marks are missing. Normal use (collapse, fill in the comments, submit) worked on our test orders.

**Function/Class:** useSubmission.onSubmit

**Severity:** high

**Confidence:** high that the code is wrong; the window is narrow in practice

**Steps to reproduce:**

1. As an editor, add a mark on a large PDF.
2. Collapse the editor and click Submit immediately, with the comment fields already filled.
3. Download the submitted edited copy.
4. **Expected:** It contains the last mark.
5. **Actual:** It can contain the previous save (or no marks).

**Problem:** `onSubmit` receives Formik's `values` snapshot. It then awaits `flushPdfDraft` (which waits for Adobe's save) and uses the snapshot. The fresh file arrives through `setFieldValue` and never reaches `data`.

**Evidence:** `Submission/hooks.ts:235` has `await flushPdfDraft?.();`, and `:258` has `const files = prepareFiles(data);`. The wait is `PdfViewerModal/hooks.ts:259-289`, which polls until SAVE_API clears `hasUnsavedMarksRef`. The new file goes through `setSubmissionFile` → `PdfSubmissionFileSync` `setFieldValue`.

**Impact:** The wait cannot achieve its purpose, so the editor's latest marks can be silently lost.

**Fix:** Have the flush resolve with the latest saved file (or read it from a ref) and use it:

```ts
const latest = await flushPdfDraft?.();
const files = prepareFiles({ ...data, editedCopy: latest ?? data.editedCopy });
```

### 5. Unreadable PDFs with comments are now refused, with an unhelpful error

**[File: apps/creative-portal/api/utils/jobs/stripPdfInternalAuthors.ts]**

> **In plain terms:** Some PDFs can't be fully read by our tools. If such a PDF has any comments in it (including the customer's own), submission is now blocked, where before it went through. The editor sees a generic "failed, try again" message instead of the intended explanation. If the file is damaged enough, the submission fails with a generic error.

**Function/Class:** stripPdfInternalAuthors

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Use an order whose original PDF pdf-lib can't parse (e.g. order 20511) and that contains a customer comment.
2. Submit through download-and-upload.
3. **Expected:** The submission succeeds, as before this PR. If it has to be blocked, the editor sees why.
4. **Actual:** It is blocked with a generic job-action error toast.

**Problem:**
- Any author other than "Proofed" blocks, including the customer's originals (tied to Issue 1).
- The error is a plain `Error`. `handleEndpointError` maps it to a 500 ("Unhandled exception occurred: …") and reports it to Sentry a second time. The sidebar shows `showJobActionError`, not the message.
- The pdfjs fallback read is outside any `try`.

**Evidence:**
- `stripPdfInternalAuthors.ts:71` has `const authors = await readAnnotationAuthors(bytes);`, which is unguarded.
- `:73-87` filters for authors other than "Proofed", then calls `reportError(...)` and `throw new Error(...)`.
- `handleEndpointError.ts:49-50,59`.

**Impact:** A regression for uploads that worked before; editors don't learn what to do; there is duplicate Sentry noise.

**Fix:**
- Block only on authors that are new relative to the original (see Issue 1).
- Throw `new ApiError(422, PDF_AUTHORS_CANNOT_BE_REMOVED_MESSAGE)` and drop the pre-throw `reportError`.
- Surface the server message in the sidebar's `onError`.
- Wrap `readAnnotationAuthors` in try/catch.

### 6. Switching to download-and-upload keeps the viewer's file attached

**[File: apps/creative-portal/hooks/usePdfSubmissionMode.ts]**

> **In plain terms:** An editor who opens the in-browser editor and then switches to download-and-upload finds a file already attached (the original, or their last in-browser save). They can submit without uploading anything.

**Function/Class:** usePdfSubmissionMode

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Open the in-browser editor on a PDF job.
2. Click "Trouble with the editor? Download and upload instead".
3. **Expected:** The upload area is empty.
4. **Actual:** It shows the original PDF as already uploaded, and Submit is allowed.

**Problem:** `submissionFile` is only cleared when the job or availability changes, not when the mode changes. `PdfSubmissionFileSync` returns early when the file becomes undefined, so the Formik field keeps it.

**Evidence:**
- `usePdfSubmissionMode.ts:79-86`, effect deps `[isEmbedAvailable, jobId]`.
- `Submission/index.tsx:106-108` has `editedCopy: pdfSubmission.submissionFile ?? uploadedFiles[...]`.
- `PdfSubmissionFileSync/index.tsx:26-32` has `if (!submissionFile) { return; }`.

**Impact:** The unmarked original or an older copy can be delivered by mistake. The upload-time checks (metadata and UUID verification) are also skipped.

**Fix:** Clear `submissionFile` when the mode becomes Fallback. In `PdfSubmissionFileSync`, reset `editedCopy` to the uploaded file (or `undefined`) when `submissionFile` becomes undefined.

### 7. If Adobe's script is blocked, the editor spins forever

**[File: apps/creative-portal/hooks/useAdobeEmbedSdk.ts]**

> **In plain terms:** If an ad blocker, company proxy or network problem stops Adobe's viewer from loading, the editor sees a loading spinner that never ends. There is no message and no automatic switch to download-and-upload, and we are not alerted.

**Function/Class:** useAdobeEmbedSdk / usePdfSubmissionMode

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Block `acrobatservices.adobe.com` (ad blocker or DevTools request blocking).
2. Open a PDF job and click the in-browser editor pill.
3. **Expected:** A message, and a switch to download-and-upload.
4. **Actual:** An endless spinner. The editor has to collapse and find the fallback link themselves.

**Problem:** The script's `"error"` status is never checked, and the mode only gates on the client id being configured.

**Evidence:**
- `useAdobeEmbedSdk.ts:40-70`: `status` is used only as an effect dependency.
- `usePdfSubmissionMode.ts:40,50` gates only on `isConfigured`.
- `PdfViewerModal/hooks.ts:98` returns early while `!isReady`.

**Impact:** A confusing dead end for editors, and no visibility for us.

**Fix:** Expose `isFailed: status === "error"` (plus a ready timeout). Treat it as a new unavailable reason, and `reportError` once.

### 8. Closing or hiding the tab can lose recent marks and creates duplicate saves

**[File: apps/creative-portal/hooks/usePdfAnnotationDrafts.ts]**

> **In plain terms:** Marks made in the 20 seconds before an editor closes the tab are not saved by the "save on close" safety net. It re-sends the previous save, or nothing on a first session. Each switch to another tab also creates another identical saved version.

**Function/Class:** usePdfAnnotationDrafts (visibilitychange / pagehide beacon)

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Open the in-browser editor on a fresh job and add a mark.
2. Close the tab within 20 seconds.
3. Reopen the job.
4. **Expected:** The mark is restored.
5. **Actual:** It is gone.

**Problem:** The beacon sends `latestAnnotationsRef`, which is only updated inside a completed `save()` read. The dirty flag isn't cleared after a beacon.

**Evidence:** `usePdfAnnotationDrafts.ts:145` has `latestAnnotationsRef.current = JSON.stringify(draft);` (only in `readAnnotations`). `:358-375` has `if (... !latestAnnotationsRef.current) return;` and `sendBeacon(... buildPayload(latestAnnotationsRef.current))`.

**Impact:** The crash-safety promise is weaker than documented, and there are extra OMS rows.

**Fix:**
- On `visibilitychange` → hidden, call the async `flush()` (it works in a hidden page); keep the beacon for `pagehide` only.
- Clear the dirty flag, or skip identical payloads, after a beacon.
- Write `latestAnnotationsRef` only after the empty-list guard passes.

### 9. Download-and-upload submissions also store an internal copy, and the comments say they don't

**[File: apps/creative-portal/api/utils/jobs/postAddWorkItemContentVersion.ts]**

> **In plain terms:** When an editor uploads a PDF they marked up in desktop Acrobat, we also keep an internal copy with their desktop name on each mark. The next internal person (e.g. the reviewer) sees that name. Internally the code says this never happens.

**Function/Class:** postAddWorkItemContentVersion → storePdfInternalCopy

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. As the editor, annotate the PDF in Acrobat under your own name and submit through download-and-upload.
2. As the reviewer, open the in-browser editor.
3. **Expected:** Marks are labelled by role, or at least consistently with the documented design.
4. **Actual:** Marks show the editor's desktop Acrobat name.

**Problem:** The internal copy is stored for every PDF submission; only the format is checked. `usePdfJobSeedVersion` and the design doc claim download-and-upload leaves no internal copy.

**Evidence:**
- `postAddWorkItemContentVersion.ts:99-104,141-146` calls `storePdfInternalCopy` unconditionally.
- `storePdfInternalCopy.ts:44` gates on format only.
- `usePdfJobSeedVersion.ts:64-66` says "A job that submitted through download-and-upload leaves no internal copy at all".

**Impact:** Real names are visible internally, which is a product question (not a customer leak). There is also a misleading comment, and an internal copy is stored even when the submission is then blocked (Issue 5).

**Fix:** Store the internal copy only for embed submissions (send a mode flag), or strip non-role authors from it. Also correct the comment and the doc. Confirm the intended behaviour with product.

### 10. The Brief popover's full-screen brief omits support documents

**[File: apps/creative-portal/components/organisms/sidebars/contents/PdfContentPill/hooks.ts]**

> **In plain terms:** The Brief button inside the in-browser editor shows the order brief without the customer's support documents. The same brief in the side panel shows them.

**Function/Class:** usePdfContentPill (brief)

**Severity:** low

**Confidence:** high

**Steps to reproduce:**

1. Open a PDF job whose order has support documents.
2. Open the in-browser editor and click the Brief icon.
3. **Expected:** Support documents are listed, as in the side panel's brief.
4. **Actual:** They are missing.

**Problem:** This is a third copy of the "build Brief props from the order" logic, and it has drifted from the others.

**Evidence:** `PdfContentPill/hooks.ts:34-54` has no `supportDocuments`, while `OrderManagment/index.tsx:342-360` and `JobManagement/partials/JobBrief` pass them.

**Impact:** Editors miss reference material while marking up.

**Fix:** Extract a shared `buildBriefProps(order)` and use it in all three places.

### 11. Overlapping saves can restore an older snapshot

**[File: apps/creative-portal/hooks/usePdfAnnotationDrafts.ts]**

> **In plain terms:** In a rare timing case (a mark made and the editor collapsed at the exact moment an autosave is being sent), reopening the job can bring back the slightly older set of marks.

**Function/Class:** save / flush

**Severity:** low

**Confidence:** medium (the race is real; hitting it needs the server to commit the two saves out of order)

**How to spot it:** Code health. There is no reliable UI reproduction.

**Problem:** Saves are not serialised. `restore` picks the highest id and ignores `savedAt`.

**Evidence:** `usePdfAnnotationDrafts.ts:159-221` has no in-flight guard. `:305` has `.sort((left, right) => right.id - left.id)[0]`.

**Impact:** Occasional loss of the last mark on restore.

**Fix:** Chain saves through a promise ref, or choose the draft by `savedAt`.

### 12. The Brief icon is a button inside a button

**[File: apps/creative-portal/components/organisms/modals/PdfViewerModal/partials/PdfViewerTopBar/index.tsx]**

> **In plain terms:** Keyboard and screen-reader users hit the Brief control twice when tabbing, and assistive tools report it as an invalid nested control.

**Function/Class:** PdfViewerTopBar

**Severity:** low

**Confidence:** high

**How to spot it:** Tab through the viewer's top bar: Brief takes two tab stops. axe flags `nested-interactive`.

**Problem:** The shared `PopoverOnClick` wraps its children in `div role="button" tabIndex=0`, and the child here is a real `<button>`. The WYSIWYG equivalent uses a non-interactive child.

**Evidence:** `PopoverOnClick/index.tsx:39-51`, and `PdfViewerTopBar/index.tsx:46-64` (`<Styled.ToolButton type="button" ...>`).

**Impact:** An accessibility violation that is new in this PR.

**Fix:** Render the icon as a non-interactive element inside `PopoverOnClick`, or add a controlled-trigger option to it.

### 13. Comments that describe removed behaviour (stale/contradictory)

**[File: several; see list]**

> **In plain terms:** Code health only. Several code comments still describe an earlier design (a "who marked what" toggle and a "reconcile" step that were removed). Future developers will be misled.

**Function/Class:** see list

**Severity:** medium

**Confidence:** high (each one was checked against current code)

**How to spot it:** Not user-reproducible. Read the comments at the lines below.

**Problem:** The current design writes the role into the file (Adobe author) and keeps it in `EditedCopyInternal`. These comments say the opposite or reference removed code:
- `usePdfAnnotationDrafts.ts:398-411`: an orphan JSDoc about reconciling, sitting on the hook's `return`.
- `config/pdfAnnotationDraft.ts:12-17`: says the role "cannot live in the PDF".
- `config/pdfAnnotationDraft.ts:109-115`: says labels are "applied from our own record".
- `normaliseAnnotationAuthors.ts:12-14`: says the role "lives in our own annotation record".
- `ServiceSubmission/index.tsx:110-112`: "It never reaches the PDF… only place the role is kept".
- `PdfViewerTopBar/styles.ts:48-52`: ToolButton doc about "the mode each one puts the viewer into".
- `config/adobeEmbed.ts:103-108`: mentions the removed "internal-labels view".
- `config/adobeEmbed.ts:75-83`: the `deleteAnnotations` doc justifies a removed relabel path.
- `config/adobeEmbed.ts:12`: a full-window-mode comment on the SDK URL constant.
- `PdfViewerModal/hooks.ts:46-47`: "what the record labels a mark with" (the record stores the raw role).
- `postAddWorkItemContentVersion.ts:91-98`: the restamp paragraph sits over the `storePdfInternalCopy` call.
- `config/pdfSubmission.ts:6-7`: "those live in the draft record".
- `usePdfJobSeedVersion.ts:64-66`: see Issue 9.
- `Submission/index.tsx:104` and `PdfViewerModal/hooks.ts:112`: cite "requirement 5" for zero-mark submit. That was client Q3, not requirement 5.
- `docs/PP-2176-adobe-pdf-embed.md:204-226`: present-tense reconcile text under "read this first".

**Impact:** Maintainers are misled about where roles live and what protects the customer copy.

**Fix:** Delete the orphan block and rewrite each comment to the current design. In the doc, mark the reconcile section as historical.

### 14. Mandatory convention violations (CLAUDE.md Code Style)

**[File: several; see list]**

> **In plain terms:** Code health only. Some new code doesn't follow the repo's agreed structure rules, which makes it harder to maintain consistently.

**Function/Class:** see list

**Severity:** medium

**Confidence:** high

**How to spot it:** Not user-reproducible. Compare with CLAUDE.md "Code Style".

**Problem:**
- `PdfSubmissionFileSync/index.tsx:26-32`: `useEffect` in `index.tsx` with no `hooks.ts` ("index.tsx is UI-only").
- `ServiceSubmission/index.tsx:108-120`: adds `useUserContext` + a fifth `useMemo` to an `index.tsx` that already broke the rule.
- `PdfViewerTopBar/types.ts:4`: `onCollapse: () => void | Promise<void>` instead of `VoidFunction`.
- `PdfContentPill/styles.ts:8-10`: `styled.div\` display: block; \``, a no-op styled component ("use the plain HTML element").
- `PdfContentPill/index.tsx:8-13`: imports another component's `styles.ts` (`../HTMLContentPill/styles`) ("styles co-location").
- `packages/shared/api/utils/pdf/types.ts:17-24`: a runtime function (`resolveRotation`) in `types.ts`.
- `PdfSubmissionModeLink/index.stories.tsx:14`: `"Molecules/Pdf submission mode link"`. The initialism must stay uppercase: `"Molecules/PDF submission mode link"`.
- Multiple separate JSX spreads instead of one object: `PdfViewerModal/index.tsx:55-56`, `PdfContentPill/index.tsx:72-75`, `ServiceSubmission/index.tsx:141-143`.
- Destructuring order doesn't match the interface: `PdfViewerModal/hooks.ts:26-33`.
- `PdfContentPill/hooks.ts:27`: `setMode("fallback")` where `PdfSubmissionMode.Fallback` is used elsewhere.
- `usePdfSubmissionMode.ts:121`: `as PdfSubmissionState & { bytes?: number }`. The cast contradicts `config/pdfSubmission.ts:40-45`, and `bytes` is never read.
- `PdfViewerModal/index.tsx:16`: a local `noop` where the repo uses `lodash/noop`.
- `workItemContentVersionRules.ts:44-49`: `excludeDraftContentVersions` also excludes `EditedCopyInternal`, so the name is misleading (`excludeInternalContentVersions`).

**Impact:** Convention drift and review friction.

**Fix:** Apply each rule as listed. Most are one-line changes.

### 15. Dead code

**[File: several; see list]**

> **In plain terms:** Code health only. Leftover pieces from removed features are still in the code.

**Function/Class:** see list

**Severity:** low

**Confidence:** high (each was grepped for usage)

**How to spot it:** Not user-reproducible.

**Problem:**
- `PdfViewerTopBar/styles.ts:59-62`: `LabelsToggle` is unused.
- `config/adobeEmbed.ts`: `deleteAnnotations`, `AdobeUserProfile` and `ADOBE_CALLBACK_TYPE.EVENT_LISTENER` are unused in production.
- `FullscreenModal/styles.ts:76-80` + `index.tsx:88`: the `MainWrapper` `isHeaderToolsHidden` prop no longer does anything.
- `PdfViewerModal/hooks.ts:298`: `annotationManagerRef` is returned but unused.
- `workItemContentVersionRules.ts:100-103`: `pickSeedContentVersion` has no production caller. `usePdfJobSeedVersion.ts:48` re-implements it with a raw `minBy` that skips the exclusion.

**Impact:** Clutter, and two definitions of "the seed version" that can diverge.

**Fix:** Delete the unused code, and use `pickSeedContentVersion` in `usePdfJobSeedVersion`.

### 16. Duplicated logic that should be shared

**[File: several; see list]**

> **In plain terms:** Code health only. The same small pieces of logic are written more than once, and some copies have already started to differ.

**Function/Class:** see list

**Severity:** medium

**Confidence:** high

**How to spot it:** Not user-reproducible.

**Problem:**
- The pdf-lib "visit every page's annotations" loop is duplicated in `normaliseAnnotationAuthors.ts:48-74` and `ensureAnnotationDates.ts:96-115`, and it has drifted. One uses a typed `lookupMaybe`; the other's comment says a typed lookup throws on dangling refs.
- The pdfjs open/loop/destroy scaffold is duplicated in `readAnnotationAuthors.ts:16-50` and `rotationDetect.ts:170-201`.
- `parseOmsTimestamp` (`getWorkItemContentVersionPdf.ts:27-34`) overlaps the shared `parseUtcDateString`. Extend the shared helper rather than forking it.
- `SwitchButton` (`PdfSubmissionModeLink/styles.ts`) duplicates `TryAgainButton` (`FileSubmission/styles.ts:11-14`).
- `roleLabel` is computed twice (`PdfViewerModal/hooks.ts:48`, `usePdfAnnotationDrafts.ts:92`). `ROLE_LABELS` is `Record<string, string>`, and its `"Proofed"` duplicates `PDF_CUSTOMER_FACING_AUTHOR`.
- `decodeDraft` could use `convertToArrayBuffer`. `storePdfInternalCopy` re-implements `fileToBase64`.
- The rotation normalisation `((x % 360) + 360) % 360` is repeated three times.

**Impact:** Future fixes land in one copy and miss the other (already visible in the lookup drift).

**Fix:** Add `forEachAnnotation(document, fn)` / `loadPdf(bytes)` and `withPdfjsPages(bytes, fn)` helpers in `packages/shared/api/utils/pdf/`, and a `getRoleLabel(role)` helper. Reuse the shared date and base64 helpers.

---

## Open Questions

- After a crash-restore, the viewer marks the document "unsaved" (the restore raises annotation events). If Adobe does not save programmatically added marks via focus polling, can an editor who restores and makes no new mark still submit? Please test: restore → no mark → Submit. — `PdfViewerModal/hooks.ts:196-199`
- The streaming submit branch (admin order-jobs modal) doesn't call `restampPdfIdentifier`. pdf-lib keeps the Info `/UUID`, so it appears harmless. Add it anyway for symmetry? — `postAddWorkItemContentVersion.ts:141-153`
- The client create route now accepts `versionType: "EditedCopyInternal"` (the enum grew). Since `EditedCopy` was already postable, should this route be restricted to `PdfAnnotations`? — `createWorkItemContentVersion/schema.ts:9-11`
- Customer portal by-id handler: does the OMS client API refuse `EditedCopyInternal` / `PdfAnnotations` versions to customers? If not, a 404 guard using `isInternalContentVersion` would enforce "never accessible to the customer" (Adam 76831). This was flagged below the confidence bar by the security review. — `packages/shared/api/workItemContentVersion/[id]/getWorkItemContentVersion`
- The full-screen viewer has no dialog role, focus trap or Escape handling, and focus isn't returned to the pill on collapse. `FullscreenModal` and the WYSIWYG modal have the same gap already. Fix in `FullscreenModal` as a follow-up? — `FullscreenModal/index.tsx:44-122`
- `usePdfSubmissionMode` returns a new object each render, which defeats downstream memos. It is cheap today; wrap it in `useMemo`? — `usePdfSubmissionMode.ts:109-121`
- Stale-draft detection matches the OMS error text "not associated with this job". Is that message contractually stable? — `usePdfAnnotationDrafts.ts:207`

---

## Validation Checks

| Check | Result | Notes |
| --- | --- | --- |
| `npx turbo run test` | ⚠️ | shared 1997/1998. The one failure is `formatWordQuantity` (locale en-IN); unrelated, and it passes with en-US. customer-portal 500/500. creative-portal 372/375 files pass. `organisms/Header/index`, `Header/hooks` and `SideNav/index` hang, identically on `origin/develop` (verified in a worktree); this branch doesn't touch them |
| `npx turbo run typecheck` | ✅ | 0 errors, all workspaces |
| `npx turbo run lint` | ✅ | 0 errors, all workspaces |
| `npx turbo run build` | ✅ | creative-portal and customer-portal compiled successfully |

Validation was run unscoped for typecheck/lint (the PR touches `packages/shared`), with full test suites for shared, creative-portal and customer-portal, and builds for both portals.

---

## Tests

- ✅ Unit tests added for all new utils, hooks, the PDF route and the submit helpers (rotation, author rewrite and read, date fill, rules, seed resolver, drafts, viewer hooks, env).
- ✅ Edge cases covered: unparseable PDFs, AI-in-between seeding, Caret documents read by id, undated marks, oversize and unknown sizes, drafts refused as documents.
- ❌ Missing: encrypted PDF through `normaliseAnnotationAuthors` (Issue 2); the original customer comments' authors being preserved (Issue 1); an unmount mid-load and a job switch (Issue 3); a late SAVE during submit (Issue 4); a mode switch clearing the file (Issue 6); an SDK load error (Issue 7).
- ✅ Manual end-to-end checks on devtest orders 21658 (Service → Review) and 21654 (AI Pre-edit → Service → AI Post-edit → Review).

### Suggested manual QA script

1. (Issue 1) Upload a PDF with an Acrobat comment by "Jane Smith", then submit as the editor. The delivered comment should still say "Jane Smith".
2. (Issue 2) Use a permission-restricted PDF as the original, then submit. The delivered file must open.
3. (Issue 3) Open the in-browser editor on a large PDF. While it loads, collapse and switch to download-and-upload. There should be no error toast and no file attached.
4. (Issue 4) Mark, collapse and submit immediately. The delivered file should contain the last mark.
5. (Issue 6) Open the in-browser editor, then switch to download-and-upload. The upload area should be empty.
6. (Issue 7) Block `acrobatservices.adobe.com` and open the in-browser editor. Expect a message and a fallback, not an endless spinner.
7. (Issue 8) On a fresh job, mark, then close the tab within 20 s and reopen. The mark should be restored.
8. (Issue 10) Open Brief inside the editor on an order with support documents. They should be listed.
9. (Open question 1) Restore after a crash, make no new mark, and Submit. It should submit without "not saved yet".

---

## Summary

| Aspect | Status |
| --- | --- |
| Correctness | ❌ Issues 1–4 (customer-comment authors, encrypted PDFs, stale load / job switch, late save at submit) |
| Regression risk | ⚠️ Medium: every PDF submission now passes through a pdf-lib rewrite (Issues 1, 2, 5) |
| Tests | ⚠️ Good coverage of new logic, but the defect paths above are untested |
| Accessibility | ⚠️ Nested interactive Brief control (Issue 12); dialog semantics gap is pre-existing |
| Error handling | ⚠️ SDK load failure, blocked-submit messaging, unguarded pdfjs read |
| Security | ✅ `/security-review` found nothing above the confidence bar. One open question on customer-portal by-id access |
| Code quality | ⚠️ Stale comments, convention drift, dead code, duplication (Issues 13–16) |
| Validation suite | ✅ Typecheck, lint and build all pass. Tests pass apart from pre-existing, unrelated locale and hang issues |
| Mergeable state | ❌ Dirty: conflict with `develop` in `packages/shared/api/workItemContentVersion/enums.ts` |

---

## Recommendation

**Request changes**

1. Limit the author rewrite to marks Proofed added, and keep the customer's original authors (Issue 1, requirement 5.1.1). Also re-scope the unparseable-file block to the same rule (Issue 5).
2. Don't re-save encrypted PDFs (Issue 2). Add a test with a permission-encrypted fixture.
3. Add cancellation to the viewer start effect, and key the viewer or `JobManagement` by job (Issue 3).
4. Use the file returned by the pre-submit wait, not the Formik snapshot (Issue 4).
5. Clear the viewer's file when switching to download-and-upload (Issue 6), and handle the SDK load error (Issue 7).
6. Resolve the `enums.ts` merge conflict with `develop`.
7. Should-fix before merge: the beacon (Issue 8); internal-copy scope and its comments (Issue 9); the Brief support documents (Issue 10); stale comments (Issue 13); convention violations (Issue 14).
8. Can follow later: Issues 11, 12, 15 and 16, and the open questions.
