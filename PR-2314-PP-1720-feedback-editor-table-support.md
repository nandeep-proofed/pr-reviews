# PR Review: feature/PP-1720: Render tables in the shared rich text Feedback editor

**PR:** https://github.com/Proofed/B2BWebserver/pull/2314
**Jira:** https://proofed.atlassian.net/browse/PP-1720
**Status:** In Progress

---

## Jira Requirements vs Implementation

PP-1720 is the umbrella story for "AI-Generated Feedback in Review (Frontend)" — a large, multi-PR feature covering form display logic, AI availability validation, drafting state UX, drafted/edit/retry states, concurrency safeguards, and HTML-order behaviour. This PR is intentionally a single infrastructure slice of that work: enabling the shared Feedback rich-text editor to render tables that arrive via paste or AI-generated drafts. The wider AI flows (validation, drafting card, score/edit, retry, cancel, concurrency) are out of scope and presumably tracked in sibling PRs (PR #2291, #2298 visible in `pr-reviews/`).

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| §5.1.3 AI Feedback card displays drafted feedback text body (which may contain tables) | Shared editor now parses, renders, and round-trips `<table>` markup with stable CSS classes | ✅ Addressed (for table rendering only) |
| §6.2.2 Standard form pre-populates the AI-drafted feedback text in the Feedback rich text editor | The same shared editor is used in the standard form (`FormikRichTextEditor`); the new extensions are wired in `richTextEditorContext/provider.tsx` so all consumers benefit | ✅ Addressed (for table rendering only) |
| §1, §2, §3, §4, §7, §8, §9, §10 (form display logic, AI availability validation, drafting / drafted / edit / retry / concurrency states, HTML-order button) | Not implemented in this PR | ❌ Out of scope (handled in sibling PRs) |
| Tests for new code | Three Vitest cases on `TableExtensions`: HTML round-trip with classes, no-op for every structural command, no insertion on empty doc | ✅ Addressed |

**Scope creep check:** None. The PR is tightly scoped to a single capability and faithfully matches the lockdown pattern already used in `packages/wysiwyg/src/extensions/table`.

---

## Architecture Analysis

The PR adds `@tiptap/extension-table` (pinned to `^3.10.7` to match the root resolution) to `@proofed/shared` and wraps the four Tiptap nodes (`Table`, `TableRow`, `TableHeader`, `TableCell`) in `StructuralReadOnly*` variants that override every structural mutation command to a no-op. Only `goToNextCell` / `goToPreviousCell` fall through to the parent so users can still tab between cells once focus lands inside one.

This is a near-verbatim copy of the existing `packages/wysiwyg/src/extensions/table/index.tsx` pattern, which has been battle-tested in the HTML review flow. Differences are confined to:
- CSS class prefix (`rich-text-table-*` vs `wysiwyg-table-*`) to avoid collision.
- Identifier renaming (`StructuralReadOnly*` vs `Custom*`) — more descriptive.
- Inline comments instead of JSDoc — matches the project's "no comments unless WHY is non-obvious" convention.
- CSS lives in Emotion `styles.ts` (Feedback editor scope) instead of `wysiwyg/src/styles.css`.

Wiring into `richTextEditorContext/provider.tsx` is one-line and additive (spread `...TableExtensions` at the tail of the extensions array). Order is correct — StarterKit doesn't ship table nodes, so there's no override conflict.

`navyBlue4` (border) is a real theme token. Header background (`rgba(0, 30, 98, 0.04)`) and even-row tint (`rgba(0, 30, 98, 0.015)`) are *not* taken from existing tokens — see issue #1.

Bundle impact: `@tiptap/extension-table` is now a direct dependency of `@proofed/shared`, so every app that pulls shared (creative-portal, customer-portal, storybook) will bundle it whether or not the surface uses `RichTextEditor`. The package is small (~10 KB minified, including `prosemirror-tables`); acceptable.

Regression risk: low. The `provider.tsx` change is purely additive, and the existing `RichTextEditor` consumers (`apps/creative-portal/.../ReviewSubmission`, `OrderEditDetailsModal`, `BriefStep`, `customer-portal/.../BriefYourEditorForm`, etc.) don't paste table markup today, so behaviour for non-table content is unchanged. The only observable surface change is that pasting an HTML table no longer strips it.

---

## Issues Found

### 1. Header / row tint colours are hard-coded rgba values instead of theme tokens

**[File: packages/shared/components/molecules/RichTextEditor/partials/EditorContainer/styles.ts]**
**Function/Class:** `tiptapOverrideStyles` — `.rich-text-table-header`, `.rich-text-table-row` rules
**Severity:** low
**Problem:** The border uses `theme.colors.navyBlue4`, but the header background (`rgba(0, 30, 98, 0.04)`) and the even-row tint (`rgba(0, 30, 98, 0.015)`) are hard-coded literals. The repo already has `navyBlue4 = rgba(0, 30, 98, 0.1)` and `navyBlue5 = rgba(0, 30, 98, 0.03)` (`packages/shared/theme/theme.tsx:31-32`) intended for exactly this kind of navy-tinted background.
**Impact:** Tokens drift from the design system. If `navyBlue4`/`navyBlue5` are ever updated for accessibility / brand reasons, these table styles won't follow. Also inconsistent with the PR description which says "table styling matches the existing border/typography tokens (`theme.colors.navyBlue4` borders, navy-tinted header)."
**Fix:** Either pull the values from theme tokens, or accept that the design called for a lighter tint than the existing tokens (in which case it's worth confirming with design and, if the tint is reusable, adding a new token rather than inlining the rgba):

```typescript
.rich-text-table-header {
  background-color: ${theme.colors.navyBlue5}; // or a new token
  font-weight: 700;
  text-align: left;
}

.rich-text-table-row {
  &:nth-of-type(even) .rich-text-table-cell {
    background-color: rgba(0, 30, 98, 0.015); // confirm with design
  }
}
```

### 2. `StructuralReadOnly*` is a near-verbatim duplicate of `packages/wysiwyg/src/extensions/table`

**[File: packages/shared/components/molecules/RichTextEditor/extensions/table.ts]**
**Function/Class:** `StructuralReadOnlyTable` / `StructuralReadOnlyTableRow` / `StructuralReadOnlyTableHeader` / `StructuralReadOnlyTableCell`
**Severity:** low
**Problem:** This file is ~95% identical to `packages/wysiwyg/src/extensions/table/index.tsx` (same options shape, same command no-ops, same parseHTML/renderHTML pattern). The only meaningful divergence is the CSS class prefix and the identifier names. Two copies means future fixes (e.g. an additional command Tiptap adds, or a `parseHTML` security hardening) have to be made in both places, and CLAUDE.md explicitly calls out "Reuse-first component convention" / "if two features need the same wrapper, it belongs in `packages/shared`, not duplicated".
**Impact:** Maintenance burden + risk of behavioural drift between the two table-locking implementations.
**Fix:** Optional follow-up — extract a factory in `@proofed/shared` that accepts a `classPrefix` ("wysiwyg" | "rich-text") and have the `wysiwyg` package import from it. This crosses a package boundary (`wysiwyg` is a separately-built Rollup package), so it's not necessarily a same-PR change, but worth tracking. Acceptable to merge as-is and follow up.

### 3. Tests don't cover the navigation-command fall-through

**[File: packages/shared/components/molecules/RichTextEditor/extensions/table.test.ts]**
**Function/Class:** describe block "TableExtensions — read-only structure"
**Severity:** low
**Problem:** The override deliberately keeps `goToNextCell` / `goToPreviousCell` working (`this.parent?.()?.goToNextCell?.() || (() => false)`). There's no test asserting they don't return `false` unconditionally — a future refactor could regress tab navigation inside a pasted table without a failing test.
**Impact:** Tab navigation between cells is a quiet UX dependency for any consumer who pastes a multi-cell table; a regression would be invisible until a user reports it.
**Fix:** Add a fourth test placing the cursor in the first cell and asserting `editor.commands.goToNextCell()` returns `true` (or, at minimum, that focus moves to the second cell). Example sketch:

```typescript
it("allows tab navigation between cells via goToNextCell", () => {
  const editor = createEditor(
    "<table><tr><td>A</td><td>B</td></tr></table>"
  );
  editor.commands.focus("start");
  // Cursor should be inside the first cell after focus.
  expect(editor.commands.goToNextCell()).toBe(true);
  editor.destroy();
});
```

### 4. CSS visual differs from the existing `wysiwyg-table` styling

**[File: packages/shared/components/molecules/RichTextEditor/partials/EditorContainer/styles.ts]**
**Function/Class:** `tiptapOverrideStyles` — `.rich-text-table`
**Severity:** low
**Problem:** Shared editor table uses `width: 100%; table-layout: fixed;` while the wysiwyg counterpart in `packages/wysiwyg/src/styles.css:82-86` uses neither. Different rendering — a table pasted into the HTML review flow vs the Feedback editor will look noticeably different (column widths, cell padding `0.375rem 0.5rem` vs `0.5rem 1rem`).
**Impact:** Inconsistency for users who see tables in both surfaces. May be intentional given the Feedback editor's tighter layout, but the PR doesn't explain the choice.
**Fix:** Either confirm the design called for distinct styling per surface (in which case leave a one-line comment pointing at the design source), or align the two so a pasted table renders identically across the wysiwyg and Feedback editors. No code change required if the divergence is intentional and approved.

---

## Tests

- ✅ Unit tests added for new code (`extensions/table.test.ts`, 3 cases)
- ✅ Existing tests not modified
- ✅ Project requirement (every PR must include tests) is met
- ⚠️ No test for `goToNextCell` / `goToPreviousCell` fall-through (issue #3)
- ⚠️ Manual staging smoke explicitly deferred ("planned post-deploy") — paste-table behaviour in Feedback editor should be eyeballed before merging into a release branch

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ |
| Regression risk | ✅ Low |
| Tests | ✅ |
| Code quality | ⚠️ (theme-token drift + duplication with wysiwyg) |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Approve with suggestions**

1. Swap the hard-coded `rgba(0, 30, 98, 0.04)` and `rgba(0, 30, 98, 0.015)` in `styles.ts` for `theme.colors.navyBlue5` (or confirm with design that a lighter, non-token tint is intentional, and add a new token if so). (Issue #1)
2. Add a unit test asserting tab navigation (`goToNextCell`) still works inside a pasted table — guards the one fall-through behaviour the lockdown deliberately preserves. (Issue #3)
3. Complete the staging smoke checklist from the PR description (paste table → render; structural shortcuts → no-op; submit + re-open round-trip) before promoting to a release branch.
4. Track the wysiwyg/shared duplication (Issue #2) as a follow-up — extracting a shared factory is out of scope for this PR but worth a ticket so the two table-lockdown copies don't diverge over time.
