# PR Review: feature/PP-1851: Render base64 and paragraph-wrapped images in WYSIWYG

**PR:** https://github.com/Proofed/B2BWebserver/pull/2282
**Jira:** https://proofed.atlassian.net/browse/PP-1851
**Status:** Waiting for Deployment (Jira) / Approved by `cam-getproofed` (GitHub)

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| Render `<img>` with `data:` URI sources (raster) | `sanitizeUrl` now allowlists `data:image/{png,jpe?g,gif,webp,bmp}`; `Image.configure({ allowBase64: true })` (both extensions) | ✅ Addressed |
| Keep `javascript:`, `vbscript:`, `file:` blocked | Replaced blocklist with two allowlists (`ALLOWED_DATA_IMAGE_MIME`, `SAFE_NON_DATA_SRC`); both reject these schemes; 24-case unit suite covers each | ✅ Addressed |
| Render `<img>` inside `<p>` (paragraph-wrapped) | Split into `BlockImage` (`name: "image"`, `inline: false`) and `InlineImage` (`name: "imageInline"`, `inline: true`); InlineImage matches inside `<p>` via default-priority parse rule | ✅ Addressed |
| Keep `<img>` outside `<p>` rendering (regression check) | `BlockImage` parse rule scoped to `context: "doc/"` and `context: "contentField/"` with `priority: 60`, winning at block contexts | ✅ Addressed |
| **Bonus (not in ticket)** — drop entire `<img>` when `src` sanitises to empty (review comment from cam-getproofed) | `acceptIfSafeSrc` returns `false` from `getAttrs` so ProseMirror skips the rule entirely; no orphan `src=""` node | ✅ Beyond scope, justified |
| **Bonus (not in ticket)** — sanitise `setImage` inputs (review comment) | `BlockImage.addCommands.setImage` now wraps every attr with `sanitizeUrl` / `sanitizeAttribute`; `InlineImage.addCommands` returns `{}` to prevent inheriting Tiptap's unsanitised default | ✅ Beyond scope, justified |
| **Bonus (not in ticket)** — control-char regex-bypass guard (review comment) | `CONTROL_CHARS` regex strips 0x00–0x1F + 0x7F before scheme test; tests cover TAB/LF/CR/NUL embedded in `javascript:` | ✅ Beyond scope, justified |

Scope creep audit: the inline-image split, the consumer migrations (`isAnyImageActive`, `getActiveImageAttrs`, `isImageNode`, etc.), and the `stylePreservation` IMAGE_NODE_NAMES propagation are all necessary follow-on for the schema change — not unrelated churn. Justified.

---

## Architecture Analysis

**Approach.** Two root causes, two surgical fixes:

1. `sanitizeUrl` was a coarse `startsWith("data:")` blocklist. Replaced with: strip control chars → if `data:`, test against a raster-MIME allowlist; else test against an inlined copy of DOMPurify's `ATTR_URI_SAFE_REGEXP` (so SSR doesn't crash on DOMPurify's no-window stub).
2. The single `Image` extension with `inline: false` was incompatible with `<p><img></p>` because the schema rejects block content in paragraphs. Split into `BlockImage` and `InlineImage` with the same attribute schema and renderer, but distinct ProseMirror `inline`/`group` semantics. Parse-rule priority (60 for block on `contentField/` and `doc/` contexts, default for inline) routes each `<img>` to the right type based on its HTML ancestor.

**Factoring.** `shared.ts` centralises the attribute schema, render, input rule, the `isImageNode` / `getActiveImageAttrs` family, and `acceptIfSafeSrc`. This avoids duplication across the two extensions and gives consumers a single import surface for the union semantic. Clean, minimal repetition.

**Consumer migration.** Every reachable `editor.isActive("image")`, `editor.getAttributes("image")`, and `node.type.name === "image"` site has been moved to a helper that knows about both node names:
- `components/Header/components/BaseTools/index.tsx` → `isAnyImageActive`
- `components/molecules/ImageModal/ImageModalContent.tsx` → `getActiveImageAttrs` + `isAnyImageActive`
- `components/molecules/CommentsContainer/utils.ts` → `isImageNode` / `isImageNodeName`
- `contexts/EditorContext/hooks.ts` → `getActiveImageAttrs`
- `extensions/stylePreservation/index.ts` → `...IMAGE_NODE_NAMES`

I grepped for any remaining `"image"` string-name dispatch I might have missed; the only hits are in `extensions/trackChanges/` (legacy V1 — not imported anywhere in the running app; `extensions/trackChanges-v2/` is the only one wired into `CORE_EXTENSIONS` and it does not dispatch by node name) and in `extensions/contentField/types.ts` (an unrelated `ContentType` discriminator union for content-field metadata, not a PM node name). No reachable misses.

**Security posture.** The sanitiser is now an allowlist, not a blocklist — a stronger primitive. SVG is excluded with rationale (can carry `<script>`). The previously-raised bypass class (`java\tscript:`) is closed by stripping ASCII control chars up-front. `setImage` and `changeImage` both sanitise on insert, so dangerous values can't live in the document even briefly. `acceptIfSafeSrc` prevents empty-src placeholder nodes from leaking when parse-time sanitisation strips a dangerous URL. Defense in depth is good here.

---

## Issues Found

### 1. Table cell `<img>` shape change (latent regression)

**[File: packages/wysiwyg/src/extensions/image/blockImage.ts]**
**Function/Class:** `BlockImage.parseHTML`
**Severity:** low
**Problem:** The block parse rule is restricted to `context: "doc/"` and `context: "contentField/"`. For `<img>` inside any other block-accepting parent (notably `<td>`, `<th>`, `<div>`, custom container nodes), the block rule does not apply, so the InlineImage rule wins and ProseMirror wraps the `<img>` in an auto-generated `<p>` to satisfy table-cell content. Before this PR, `<td><img></td>` ingested as `<td><image></td>` (block-level inside the cell). After this PR, it ingests as `<td><p><imageInline></p></td>`.
**Impact:** Rendered HTML is visually identical, and the round-trip is stable. But:
- Any CSS that targets `td > img` directly will miss (now `td > p > img`).
- Existing documents that already serialised a block image inside a cell still load (those nodes already exist in stored state), but newly-ingested table-cell images will follow the new shape — a quiet inconsistency.
- The ticket scenarios don't include table-with-image, and no test covers it.
**Fix:** Either (a) accept this as an intentional behavioural change and confirm with the team it doesn't break SoFi-style content, or (b) add `context: "tableCell/"`, `context: "tableHeader/"`, and any other custom block parent to BlockImage's `parseHTML`. Recommend (b) for safety, since the original behaviour was block-in-cell:

```typescript
parseHTML() {
  return [
    { tag: "img[src]", context: "contentField/", priority: 60, getAttrs: acceptIfSafeSrc },
    { tag: "img[src]", context: "doc/", priority: 60, getAttrs: acceptIfSafeSrc },
    { tag: "img[src]", context: "tableCell/", priority: 60, getAttrs: acceptIfSafeSrc },
    { tag: "img[src]", context: "tableHeader/", priority: 60, getAttrs: acceptIfSafeSrc }
  ];
}
```

Add a corresponding ingest test for `<table><tr><td><img></td></tr></table>` so the shape doesn't drift again.

### 2. Soft-hyphen / zero-width-prefix scheme bypass (defense-in-depth gap)

**[File: packages/wysiwyg/src/extensions/image/utils.ts]**
**Function/Class:** `sanitizeUrl` / `CONTROL_CHARS`
**Severity:** low
**Problem:** `CONTROL_CHARS = /[\x00-\x1f\x7f]/g` strips only ASCII control chars + DEL. Non-ASCII invisible characters (U+00AD soft hyphen, U+200B zero-width space, U+FEFF BOM, U+202E RLO, etc.) pass through. An input like `"­javascript:alert(1)"` is not stripped, falls through to `SAFE_NON_DATA_SRC`, and matches branch `[^a-z]` (U+00AD is not in `/[a-z]/i`), so it's *accepted* by the sanitiser.
**Impact:** In practice WHATWG-compliant browsers (Chrome/Firefox/Safari/Edge) reject this URL: scheme parsing requires ASCII alpha at offset 0, so the URL falls back to a relative path, fetches a 404, and never executes. So this is **not exploitable today**. But the sanitiser's contract is "reject dangerous schemes" — relying on browser URL-parser quirks for the final block is fragile, and any future browser leniency, server-side rendering target, or HTML-to-text pipeline that does its own scheme detection could shift the threat model.
**Fix:** Extend the control-char strip to cover Unicode format characters and ZWSP/BOM (or, more simply, strip all chars in the General_Category `Cf` / leading Unicode whitespace before scheme detection). Cheapest first-pass:

```typescript
// Strip ASCII control + common invisible Unicode chars that
// browsers or downstream parsers might silently elide from scheme.
// eslint-disable-next-line no-control-regex
const CONTROL_CHARS =
  /[\x00-\x1f\x7f­​-‏‪-‮⁠﻿]/g;
```

Add a test case for `"­javascript:alert(1)"` and `"​javascript:alert(1)"` expecting `""`.

### 3. Missing tests for migrated consumer call sites

**[File: packages/wysiwyg/src/components/molecules/CommentsContainer/utils.ts]**
**Function/Class:** `formatIndividualDiffs`
**Severity:** low
**Problem:** Five separate branches inside `formatIndividualDiffs` were rewritten from `node?.type?.name === "image"` to `isImageNode(node)` / `isImageNodeName(name)`. There is no test file for this 2000-line module (`ls packages/wysiwyg/src/components/molecules/CommentsContainer/` shows no `*.test.*`), so the new branches that produce `<b>Add:</b> image` messages for `imageInline` nodes are uncovered. A typo or wrong branch ordering here would silently degrade the change-history UI (image diffs falling back to the generic `default: return null` branch).
**Impact:** No regression today (the change is mechanical and small), but future edits to this large utility are unguarded. Same story for `ImageModalContent.tsx`, which now reads attributes via `getActiveImageAttrs` — no unit tests for the modal exist either.
**Fix:** Out of scope to fully cover here, but at minimum add one focused test for `formatIndividualDiffs` that asserts an inserted `imageInline` node produces `"<b>Add:</b> image"` (not the paragraph-add fallback). Pre-existing testing gap; flagging so it's visible in the merge decision.

### 4. PR body references a change that isn't in the final diff

**[File: PR description]**
**Function/Class:** —
**Severity:** low
**Problem:** The "Areas of Change" section claims `packages/wysiwyg/src/__mocks__/dompurify.ts` was extended with `isValidAttribute` mirroring DOMPurify's `ATTR_URI_SAFE` semantics. The actual file on the branch is the original 3-line pass-through stub (`const sanitize = (html, _opts) => html; export default { sanitize };`), and the runtime `utils.ts` no longer calls DOMPurify at all (the `ATTR_URI_SAFE_REGEXP` was inlined in commit `151c17c1b` to fix SSR). The mock claim is stale.
**Impact:** Cosmetic — confusing for future archaeology when somebody greps for `isValidAttribute` and wonders why it isn't there. Not a code issue.
**Fix:** Edit the PR body to remove the dompurify mock bullet, or note that it was reverted along with the DOMPurify dependency removal.

---

## Tests

- ✅ 24-case `sanitizeUrl` unit suite covers: empty/whitespace, http/https/relative, every allowed raster MIME (PNG/JPEG/JPG/GIF/WEBP/BMP), case-insensitive scheme match, javascript/vbscript/file rejection, control-char-embedded bypass attempts (TAB/LF/CR/NUL), non-image data URIs (text/html, application/javascript, SVG with and without base64), bare `data:`.
- ✅ 8 schema-ingest cases exercise the **real** `CORE_EXTENSIONS` (not stubs) for all 4 ticket "preserves" scenarios + 2 regression cases (bare `<img>` between block siblings, base64 between block siblings) + 2 round-trip shape assertions (no spurious `<p>` wrap, `<p>` wrap preserved). Each asserts both the resulting `src` and the bound node type (`image` vs `imageInline`).
- ✅ 6 schema-ingest "rejects entirely" cases cover the validation rules — `javascript:`, `vbscript:`, `file:`, `data:image/svg+xml` (XSS guard not in the ticket — added defensively), plus control-char-embedded `javascript:`.
- ✅ One `setImage` programmatic-insert test verifies the command-level sanitisation gate (addresses cam-getproofed review comment).
- ⚠️ No new tests for `ImageModalContent.tsx` or `CommentsContainer/utils.ts` consumer changes (issue #3). Pre-existing coverage gap inherited, not newly introduced.
- ⚠️ No test for `<img>` inside `<td>` / `<th>` / `<div>` (issue #1). The block-vs-inline routing in non-trivial block contexts is not exercised.
- ⚠️ No E2E test for the API-ingest flow end-to-end. Acceptable — covered by manual devtest per PR body.
- PR body reports `21 files / 225 passed` on `npx turbo run test --filter=@proofed/wysiwyg-editor`.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Both root causes addressed; consumers migrated thoroughly |
| Regression risk | ⚠️ Low — `<td><img>` shape change is the only realistic regression; `imageInline` rollout under HocusPocus collab during deploy is the usual short-lived schema-drift window |
| Tests | ✅ Strong for new code; pre-existing gaps inherited |
| Code quality | ✅ Clean shared/Block/Inline split; helper-based migration of consumers; comments on `priority`/`context`/cleared `addCommands` explain non-obvious choices |
| Mergeable state | ✅ Clean — branch is `mergeable_state: clean`, approved by `cam-getproofed` |

---

## Recommendation

**Approve with suggestions**

The PR is solid and ready to merge. The four issues above are all `low` severity, none of them blocking:

1. **Before merge:** confirm with the team whether `<td><img>` content from API integrators exists in current/expected payloads. If yes, add the `tableCell/` / `tableHeader/` parse contexts to `BlockImage`. If no, accept the shape change. (Issue #1.)
2. **Optional hardening:** extend `CONTROL_CHARS` to cover Unicode format/ZWSP chars and add two tests. Defense-in-depth, not exploitable today. (Issue #2.)
3. **Follow-up ticket:** add unit coverage for `formatIndividualDiffs` image branches and `ImageModalContent` attribute init. Pre-existing gap, but the surface is now larger. (Issue #3.)
4. **Trivial:** correct the PR body re: the dompurify mock. (Issue #4.)
