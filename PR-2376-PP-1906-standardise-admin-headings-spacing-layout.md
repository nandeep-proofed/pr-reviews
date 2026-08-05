# PR Review: feature/PP-1906: Standardise admin headings, spacing & layout

**PR:** https://github.com/Proofed/B2BWebserver/pull/2376

**Jira:** https://proofed.atlassian.net/browse/PP-1906

**Status:** Code Review

**Head reviewed:** `471051bd805d96f584ca7af6d373d0cf278c8377`

**Base:** `feature/PP-1915-order-dashboard-column-widths-alignment` — **fourth in a stack**: #2368 (PP-1918) ← #2369 (PP-1910) ← #2372 (PP-1915) ← **#2376 (PP-1906)**. All three ancestors are open and all have unresolved findings.

**Scope:** 68 files, +847 / −283, 7 commits. Consolidates PP-1907, PP-1908, PP-1909, PP-1923, PP-1924.

**Method:** multi-agent review at high effort — 4 finder lenses fanned out, then an independent adversarial verifier per distinct `file:line`. 36 agents, 10 findings after verification. Two findings were re-checked by hand against the ticket and the raw diff before writing (see Issue 1 and the note under "Verified against the ticket" below).

---

## What this means for users (non-technical summary)

1. **The headline fix in this PR does not actually take effect.** The change is meant to stop the Customers and Team Members tables spilling under the fixed side navigation. The wrapper that would do it was written but never attached to anything, so those tables still slide under the side nav exactly as before — the leftmost columns (photo, name) become unreadable and unclickable.
2. **A filter-highlight from the previous PR in this chain is silently undone.** The grey "pill" that marks an applied filter on the Orders dashboard — added deliberately one PR earlier — is removed here, so an applied filter looks the same as an unapplied one unless you hover it. The PR description doesn't mention it.
3. **Dropdown options can be hidden behind the header.** On the Team Members page, opening the Roles filter near the top of the screen draws the option list *underneath* the fixed header and side panel, so the top options can't be clicked.
4. **Admin pages visibly jump on load.** Every hard load of an admin page paints with the old 64px margin and then shifts 32px sideways a moment later.
5. **A tab row in an unrelated order modal shifts 32px left,** because the admin page gutter was applied to the shared Tabs component rather than to the admin layout. The ticket explicitly says non-admin areas must stay unaffected.

The heading and spacing work the ticket actually asks for looks correctly done — see the requirements table.

---

## Verified against the ticket

One finding from the automated pass was **withdrawn** after reading PP-1906:

> The removal of the "Customers:" prefix and the "Team Members" label was flagged as a regression that leaves those pages without a title. It is not a defect — requirement 3 asks for exactly this ("Customer management — remove the 'Customers:' prefix"; "Team management — remove the 'Team Members' label above the selector"), and requirement 4 asks for no leftover gap, which the diff also handles (`ml="0.25rem"` and the wrapper offset are removed). The consequence — a page headed only by "Select a Partner ▾" when nothing is selected — is the spec's intent, not a bug. It is raised as an Open Question for the designer instead.

Also worth noting: the element removed from Team Members was `<Text variant="text4">`, not a heading variant, so there is no semantic-heading regression there.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1. All admin page headings use Headline 5 (20/28) | New `heading5` token in `textTheme`, applied across reporting, partners, availability, settings, project settings, team filters | ✅ Addressed |
| 2. Setting-card headings use Headline 5 globally (incl. Creative), padding/layout unchanged | `titleVariant` prop threaded through `Section` / `ProfilePhoto` / profile steps | ✅ Addressed — though applied by copy-paste at five call sites (Issue 9) |
| 3.1–3.3 Heading selectors show only the dropdown, no prefix/label | "Customers:" prefix and "Team Members" label removed; selectors at heading5 | ✅ Addressed |
| 4. Dropdown behaviour unchanged, no leftover gap | Wrapper offsets removed; `HeadingFilter` untouched | ✅ Addressed |
| 5. 32px above/below each heading section, applied to the whole row | `marginBottom="2rem"` on Section headings; row-level `alignItems: center` | ✅ Addressed |
| 6. Global 32px padding on admin content; side-bar-to-content gap is the content's left padding | `getAreaGutter` (2rem admin / 4rem elsewhere) on `MainLayout` | ⚠️ Partial — correct value, but keyed on state that resolves after hydration, so it paints 4rem first (Issue 5) |
| 7. Card min/max widths and centered layouts hold across side-bar states, no overflow | `isInset` variant, shared tab wrapper, `TableOverflowWrapper` | ❌ Missing — the overflow container is never mounted (Issue 1) |
| 8. Applies to all listed admin pages, including those not in Figma | Headings/spacing applied broadly across the listed pages | ✅ Addressed |
| 9. Admin area only — other creative/non-admin areas must remain unaffected (except req 2) | Gutter is area-scoped… | ❌ Missing — the shared `Tabs` padding change reaches `OrderEditDetailsModal` (Issue 6) |
| Testing note 8: Creative/non-admin areas unaffected | — | ❌ Missing — same as req 9 |

**Beyond Jira scope:** `scrollSafeMenuProps` + a shared `Select` `menuPortal` z-index (Issue 3), the `COMPUTED_HEADER_HEIGHT` → `COMPUTED_FOOTER_HEIGHT` rename, and left-aligned avatar/selection columns (Issue 7).

---

## Architecture Analysis

The direction is right and mostly well-executed. A real design token (`heading5`) with a guard test (`textTheme.test.ts` covering both the new token and the `heading3` blast radius) is the correct way to make a type change reviewable, and threading a `TextVariant` (`keyof textTheme`) through a `titleVariant` prop keeps the call sites type-safe rather than stringly-typed. Replacing per-page `min-height`/negative-margin copies with a shared `AdminTabsWrapper` is a genuine simplification, and renaming `COMPUTED_HEADER_HEIGHT` to `COMPUTED_FOOTER_HEIGHT` to match what it reserves is the kind of small honesty that pays off later.

Three problems run through the findings:

1. **A stated fix is not wired up.** `TableOverflowWrapper` is defined with an explanatory comment and exported — and referenced nowhere else in 68 files. The two tables it was written for still use their own `TableWrapper` with no `overflow-x`. Worse, the supporting work *did* land: `scrollSafeMenuProps` portals menus out of a clipping container, and a shared `Select` z-index was changed to sit above it. So the PR carries the cost and complexity of a workaround for a container that does not exist.
2. **Shared surfaces are edited to serve one page.** The admin 32px gutter went into the shared `Tabs` molecule; the active-filter pill was removed from the shared `FilterHeadingTrigger`; a shared `Select` gained a portal z-index. Each is one line, and each reaches consumers outside the admin area — which requirement 9 forbids.
3. **Layout decisions are keyed on post-hydration state.** `getAreaGutter` reads `areaType`, which starts `"creative"` and only becomes `"admin"` in an effect. `hasTabsLayout`, four lines away in the same hook, derives the equivalent flag synchronously from `pathname` — the pattern was already there to copy.

Note the stack context: this is the fourth PR in a chain, and Issue 2 is a direct (apparently unintentional) revert of its own parent's work. That is the characteristic failure mode of long stacks, and it is worth a rebase-and-diff check against `feature/PP-1915-…` before merge.

---

## Issues Found

### 1. `TableOverflowWrapper` is dead code — the scroll-containment fix never ships

**[File: apps/creative-portal/components/molecules/tables/Table/styles.ts:221]**

> **In plain terms:** The point of this change was to stop the Customers and Team Members tables sliding underneath the fixed side navigation on smaller screens. The container that does that was written and commented, but never attached to either table — so the behaviour is unchanged and the pages still break the same way. Meanwhile the supporting workaround for that container *did* ship, so the code now works around something that isn't there.

**Function/Class:** `TableOverflowWrapper` (unused export); `CustomerTable/styles.ts`, `TeamMembersTable/styles.ts`

**Severity:** High

**Confidence:** high

**Steps to reproduce:**

1. Log in as an admin and open `/team-members` (or `/customers`) on a laptop-width viewport, or with the admin side nav expanded/resized wide.
2. The table is wider than the content column.
3. **Expected:** the table scrolls horizontally *within* its own container, leaving the side nav and the rest of the page fixed.
4. **Actual:** the whole page scrolls horizontally. The leftmost avatar and Name columns slide underneath the `position: fixed` side nav, where they are unreadable and unclickable — identical to the pre-PR behaviour.

**Problem:** `TableOverflowWrapper` is added to `Table/styles.ts` with a comment describing it as the scroll-containment wrapper, but nothing imports it. `CustomerTable/styles.ts` and `TeamMembersTable/styles.ts` both still export their own `TableWrapper = styled.div` with no `overflow-x`, and both tables still render those.

**Evidence:** Across all 68 changed files, the string `TableOverflowWrapper` occurs exactly once — on its own definition line, `+export const TableOverflowWrapper = styled.div\`` in `Table/styles.ts`. There is no import, no JSX usage, and no re-export. The PR description states it is "reused by Customer/TeamMembers tables", which is not true at this head.

**Impact:** Requirement 7 ("no overflow or broken layout") is not met. Three secondary costs follow: a stale explanatory comment that will mislead the next reader, a now-false PR description, and `scrollSafeMenuProps` plus the shared `Select` z-index change (Issue 3) existing solely to escape a clipping container that was never created.

**Fix:** Wire it up — replace the local `TableWrapper` in both tables:

```typescript
// CustomerTable/index.tsx and TeamMembersTable/index.tsx
import { TableOverflowWrapper } from "components/molecules/tables/Table/styles";

<TableOverflowWrapper>
  <Table {...props} />
</TableOverflowWrapper>
```

and delete the two now-unused local `TableWrapper` exports. If the containment is being deferred, remove `TableOverflowWrapper`, `scrollSafeMenuProps` and the `Select` z-index change with it, and correct the PR description — shipping half of a mechanism is worse than shipping neither.

### 2. Silently reverts the active-filter pill added by its own base branch

**[File: apps/creative-portal/components/molecules/FilterDropdown/partials/FilterHeadingTrigger/styles.ts:74]**

> **In plain terms:** One PR earlier in this chain added a grey pill behind a filter heading to show that a filter is applied. This PR deletes that rule, so an applied filter looks identical to an unapplied one and the pill only reappears while the mouse is over it. Nothing in the PR description mentions the removal, so it reads as an accident of stacking rather than a decision.

**Function/Class:** `TriggerWrapper` — the `isActive && pillStyles` rule

**Severity:** High

**Confidence:** high

**Steps to reproduce:**

1. Open the Orders dashboard as an admin.
2. Apply a Customer or Status filter.
3. **Expected:** the filter heading keeps a navy-at-3% pill background at 40px radius, as PP-1915 (#2372) shipped, to mark it as active.
4. **Actual:** the pill is gone — the applied chip differs from an unapplied one only by label colour.
5. Hover the chip. **Actual:** the pill reappears, then disappears again on mouse-out.

**Problem:** The `isActive && pillStyles` rule added by this PR's own base branch is deleted; the accompanying edit only rewrites the surviving *hover* rule's `border-radius: 2.5rem` to `40px`, which is why the change reads as a formatting tidy-up in review.

**Evidence:** `FilterHeadingTrigger/styles.ts:72-74` at this head versus the same block at `feature/PP-1915-order-dashboard-column-widths-alignment`.

**Impact:** Merging this branch reverts a deliberate affordance from the PR directly beneath it, with no record of the decision. Note the cross-reference: in the #2372 review I flagged that the same pill leaks onto the Partners page. If the intent is to scope or remove the pill, do that deliberately in one place — do not let a stack accident decide it.

**Fix:** Restore the rule, then decide the pill's scope explicitly (a `hasPersistentActivePill` opt-in prop, per the #2372 recommendation). Before merge, diff this branch against its base for any other unintended reverts:

```bash
git diff feature/PP-1915-order-dashboard-column-widths-alignment...HEAD -- '*FilterHeadingTrigger*'
```

### 3. Portaled Select menu sits below the fixed header and side panel

**[File: packages/shared/components/atoms/Fields/Select/hooks.ts:380]**

> **In plain terms:** Filter dropdowns were changed to render at the top level of the page so they can escape their container. But they were given a stacking priority lower than the page header and the side panel — so when the filter is near the top of the screen, its options are painted behind the header and can't be clicked.

**Function/Class:** `menuPortal` style in the shared `Select` hooks

**Severity:** High

**Confidence:** high

**Steps to reproduce:**

1. Open `/team-members` as an admin.
2. Scroll the page so the Roles filter sits near the top, just under the fixed header.
3. Open the Roles filter.
4. **Expected:** the full option list is visible and clickable.
5. **Actual:** the list renders underneath the fixed header (and underneath an open side panel), so the top options are painted over and cannot be clicked.

**Problem:** The new `menuPortal` style sets `zIndex: theme.zIndices.dropdown` (9). Once the menu is portaled to `document.body` it stacks against page chrome rather than page content, and in `packages/shared/theme/theme.tsx:44` `header` is 10 and `sidebar` is 12.

**Evidence:** `Select/hooks.ts:380` (`zIndex: theme.zIndices.dropdown`); `theme.tsx:44` (`dropdown: 9`, `header: 10`, `sidebar: 12`); `TeamMembersTable/partials/consts.ts:7` (`scrollSafeMenuProps` consumer).

**Impact:** An unusable filter in a common scroll position — and the change is in `packages/shared`, so it applies to every portaled `Select` in both apps, not just the two tables this PR targets.

**Fix:** A portaled menu must out-rank the chrome it escapes. Add a dedicated token rather than reusing `dropdown`:

```typescript
// theme.tsx
zIndices: { /* … */ sidebar: 12, menuPortal: 20 }

// Select/hooks.ts
menuPortal: (base) => ({ ...base, zIndex: theme.zIndices.menuPortal })
```

Note this issue only exists because of the portal workaround, which in turn only exists for the container in Issue 1 — resolving Issue 1 one way or the other should settle this too.

### 4. A new test's name asserts the opposite of its expectation

**[File: apps/creative-portal/components/layouts/MainLayout/utils.test.ts:76]**

> **In plain terms:** A new test is called "keeps admin-only routes out of reach of non-admins", but what it actually checks is that a non-admin visiting an admin URL *is* treated as being in the admin area. Anyone who searches the tests for proof that access is restricted will find this and be reassured by a test that proves the opposite.

**Function/Class:** `getAreaType`

**Severity:** Medium

**Confidence:** high

**How to spot it:** Not user-visible. Read the test body against its name — `expect(getAreaType(appRoutes.orders, false, "creative")).toBe("admin")` under a name promising the reverse.

**Problem:** The behaviour itself is pre-existing and mostly harmless in practice (`hasAdminSidebar` correctly hides the sidebar for a non-admin, and `filterAdminNavLinks` empties the nav when all admin flags are false). The defect is the test: its name encodes the safe behaviour while its expectation encodes the unsafe one.

**Evidence:** `MainLayout/utils.test.ts:76` (name + expectation); `MainLayout/index.tsx` (`mainNavLinks = filterAdminNavLinks(adminMainNavLinks, adminFlags)`, `hasAdminSidebar`).

**Impact:** A non-admin who reaches `/orders` via a deep link, stale bookmark, or revoked admin flag gets the white admin canvas, admin header chrome, and a degenerate empty nav. More importantly, the suite will stay green if someone later widens this, and the test actively misleads the next reviewer who greps for the access check.

**Fix:** Rename the test to describe what it asserts, and — separately — decide whether the behaviour is intended:

```typescript
it("resolves an admin route to the admin area regardless of admin flags", () => {
  expect(getAreaType(appRoutes.orders, false, "creative")).toBe("admin");
});
```

If non-admins should not land in the admin area, that is its own ticket; do not fix it silently here.

### 5. The content gutter is keyed on post-hydration state, so admin pages reflow 32px on every load

**[File: apps/creative-portal/components/layouts/MainLayout/styles.ts:44]**

> **In plain terms:** Admin pages are meant to have a 32px margin. On first paint they get the non-admin 64px margin instead, then snap to 32px a moment later — so every admin page load shows the content visibly jump sideways, along with a background-colour flash.

**Function/Class:** `getAreaGutter` / `StyledMainLayoutContent`

**Severity:** Medium

**Confidence:** high

**Steps to reproduce:**

1. Hard-load `/orders` (or `/team-members`, `/customers`) as an admin — a full page load, not client-side navigation.
2. Watch the left edge of the content and the page background as it settles.
3. **Expected:** content renders at the 32px gutter immediately (requirement 6).
4. **Actual:** it paints with a 64px gutter and a tinted body, then reflows 32px horizontally after hydration.

**Problem:** `MainLayout/hooks.ts:62` declares `useState<AreaType>("creative")` and only sets it in an effect (`hooks.ts:127/164`). `getAreaGutter` returns `"4rem"` for anything that is not `"admin"`, and the `<Global>` canvas colour at `index.tsx:103` is gated on the same state, so both the gutter and the background correct post-hydration.

**Evidence:** `MainLayout/styles.ts:44` (`getAreaGutter`); `MainLayout/hooks.ts:62,127,164` (state + effect); `hooks.ts:168` (`hasTabsLayout`, derived synchronously from `pathname` — the correct pattern, four lines away).

**Impact:** Requirement 6 is visually met only after hydration, and every admin page load carries a content jump — on a PR whose stated purpose is layout consistency.

**Fix:** Derive the gutter from `pathname` during render, matching the `hasTabsLayout` pattern already in the same hook, so first paint is correct:

```typescript
const isAdminArea = getAreaType(pathname, isAdmin, areaType) === "admin";
const gutter = isAdminArea ? "2rem" : "4rem";
```

### 6. The admin gutter was hardcoded into the shared `Tabs`, shifting an unrelated modal

**[File: apps/creative-portal/components/molecules/Tabs/styles.ts:29]**

> **In plain terms:** The 32px admin page margin was applied inside the shared Tabs component rather than to the admin page layout. Tabs are used elsewhere — including a modal on the order screen — so that modal's tab labels now sit 32px further left than its own content, and no longer line up.

**Function/Class:** `TabsHeadings` (`isFullWidth` branch)

**Severity:** Medium

**Confidence:** high

**Steps to reproduce:**

1. Open an order as an admin and open the **Edit Details** modal (`OrderEditDetailsModal`, which renders `<Tabs isFullWidth>`).
2. Compare the tab labels' left edge with the modal's own content padding below them.
3. **Expected:** unchanged — requirement 9 says non-admin surfaces must not be affected.
4. **Actual:** the tab labels sit 32px further left than the modal content, misaligned.

**Problem:** `padding-left/right` on the shared `TabsHeadings` `isFullWidth` branch changed from 4rem to 2rem. `<Tabs isFullWidth>` is not admin-only — `OrderEditDetailsModal/index.tsx:160` uses it.

**Evidence:** `Tabs/styles.ts:29`; `OrderEditDetailsModal/index.tsx:160`.

**Impact:** A direct requirement-9 violation, and the gutter now lives in a molecule that has no business knowing about the admin page layout.

**Fix:** Keep the gutter in the layout. Either scope it via the new `AdminTabsWrapper` this PR already introduces, or make it a prop:

```typescript
// AdminTabsWrapper — the admin layout owns its own gutter
${TabsHeadings} { padding-left: 2rem; padding-right: 2rem; }
```

and restore `4rem` as the shared default.

### 7. Avatar columns were left-aligned but kept their 128px width

**[File: apps/creative-portal/components/molecules/tables/TeamMembersTable/consts.tsx:67]**

> **In plain terms:** The photo in each row was moved from the middle of its column to the left edge, but the column stayed the same width. So the photo now sits far from the name beside it — a wide empty gap in the middle of every row, making each row read as two disconnected halves.

**Function/Class:** avatar column definition (same change in `CustomerTable/consts.tsx:46`)

**Severity:** Medium

**Confidence:** medium — the geometry is arithmetic from the diff; the visual judgement should be confirmed against Figma

**Steps to reproduce:**

1. Open `/team-members` (and `/customers`) as an admin.
2. Look at the horizontal gap between a member's photo and their name.
3. **Expected:** photo aligned to the 32px gutter and visually connected to the name.
4. **Actual:** the ~40px avatar sits at x≈32–72 while the Name/Email column still starts at x≈144 — a ~72px gap, wider than the ~48px it replaced.

**Problem:** `align` changed from `"center"` to `"left"` while `width: 128` was left untouched, so all the slack moved to the right of the avatar instead of being removed.

**Evidence:** `TeamMembersTable/consts.tsx:67` and `CustomerTable/consts.tsx:46` (align changed, width unchanged).

**Impact:** The stated goal (avatar aligned to the 32px gutter, matching the heading above) is achieved, but row legibility gets worse rather than better.

**Fix:** Bring the column width down with the alignment — roughly `width: 72` (32px gutter + 40px avatar) — and confirm against the Figma row spec.

### 8. Dead class declarations duplicate the generic first/last-cell rules

**[File: apps/creative-portal/components/molecules/tables/TableWithFilters/styles.ts:153]**

> **In plain terms:** Two styling rules were added that do nothing — the rules directly above them already cover the same cells. They come with eight lines of comment explaining why they matter, so the next person will believe they're needed and put them back.

**Function/Class:** `.column-expand` / `.column-status` declarations

**Severity:** Low

**Confidence:** high

**How to spot it:** Not user-visible. Lines 145–150 already set `&:first-of-type { padding-left: 2rem }` and `&:last-of-type { padding-right: 2rem }` on the same `${TableHeading}, ${TableCell}` block.

**Problem:** The chevron column is always first (`table.tsx:306`) and `column-status` is always last (`tableColumns.tsx:268`), so the class-scoped padding declarations are unreachable duplicates. Only `padding-right: 0` on `.column-expand` does any work.

**Evidence:** `TableWithFilters/styles.ts:145-153`; `table.tsx:306`; `tableColumns.tsx:268`.

**Impact:** Misleading dead CSS, made stickier by its comments. Note the interaction with the #2372 review: `.column-status` is the class whose *only* real rule was deleted there — the two PRs are pulling this selector in opposite directions.

**Fix:** Delete both padding declarations, keep `padding-right: 0`, and drop the comments that describe them as load-bearing.

### 9. The admin heading preset is copy-pasted across five `Section` call sites

**[File: apps/creative-portal/components/pages/team-members/profile/[memberId]/partials/JobsTab/index.tsx:166]**

> **In plain terms:** The "admin heading" look — a specific text size plus a 32px gap — is written out by hand in five different places rather than defined once. This PR is itself an example of why that hurts: it had to change the heading size, which meant finding and editing every one of them. Miss one and that page has a visibly larger heading than its neighbours.

**Function/Class:** `Section` call sites

**Severity:** Low

**Confidence:** high

**How to spot it:** Not user-visible today. `titleVariant="heading5" marginBottom="2rem"` appears at `JobsTab/index.tsx:166` and `:201`, `HistoryTab/index.tsx:62`, `CalendarTab/index.tsx:31`, and `settings/index.tsx:21`.

**Problem:** The pairing of type token and spacing is a single design decision expressed five times, so the next token change repeats this PR's manual sweep.

**Evidence:** the five call sites listed above.

**Impact:** A missed site renders a 24px SangBleu heading beside 20px ones on the same tab — exactly the inconsistency PP-1906 exists to remove.

**Fix:** Give `Section` a preset that resolves both values, so call sites express intent rather than values:

```typescript
<Section variant="admin" title="Jobs">  // resolves heading5 + 2rem gap
```

---

## Open Questions

- **Pages with no selection now have no title.** Requirement 3 is satisfied, but a user landing on `/customers` with no partner chosen sees a header reading only "Select a Partner ▾", and `/team` reads only "Select Team ▾". Is that the intended empty state, or should the default text name the page (e.g. "All Customers")? Worth confirming with the designer rather than changing here.
- **Is there a page-level heading element on these pages at all?** The removed Customers element was `<Text variant="heading2">` and the Team Members one was `<Text variant="text4">` — neither is necessarily a semantic heading. If these pages have no `h1`/`h2` in the DOM, that predates this PR but is worth a separate a11y ticket.
- **Was the `FilterHeadingTrigger` pill removal (Issue 2) intentional?** Nothing in the PR body mentions it, and it undoes the base branch — please confirm before it is treated as a decision.
- **Does requirement 5's "two adjacent heading sections must not overlap or collapse their 32px spacing" have a test or QA case?** It is marked ❌ in the ticket's own testing notes and I could not find coverage for it in the diff.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⚠️ Not run | PR head `471051b` cannot be fetched in this environment — `git fetch` over HTTPS has no stored credentials and the available SSH key has no repo access. Unchanged from the #2368 / #2369 / #2372 reviews. |
| `npx turbo run typecheck` | ⚠️ Not run | Same. Author reports green on both affected workspaces. |
| `npx turbo run lint` | ⚠️ Not run | Same. Author reports green (`--max-warnings 0`). |
| `npx turbo run build` | ⚠️ Not run | Same. Author states full `turbo run build` was **not yet run**. |
| Mergeable state | ✅ `clean` | Against its base — but the base is `feature/PP-1915-…`, fourth in a stack of four unmerged PRs. |

**To unblock:** run `git fetch origin feature/PP-1906-standardise-admin-headings-spacing-layout` in your own shell (prefix with `!` in this session) and the suite can run against a detached worktree at the PR head.

**Scope note:** this PR touches `packages/shared` (`Select/hooks.ts`, `textTheme`), so the **full unscoped** suite is required — both by the current `CLAUDE.md` and by the revised targeted-testing policy proposed in #2372, which explicitly exempts shared packages from `--filter` scoping.

**Author's reported state (unverified here):** affected-scope tests only — 20/20 pass across four files (creative-portal ×3, shared ×1); typecheck and lint green; full test suite **not** run; full build **not** run; manual/visual QA **pending**; `yarn bump-packages` **not done**.

---

## Tests

- ✅ `textTheme.test.ts` guards both the new `heading5` token and the `heading3` blast radius — the right way to make a design-token change reviewable.
- ✅ `Section` (`titleVariant`) and `SavedViewDropdown` tests accompany the new props.
- ❌ **A new test asserts the opposite of its own name** (Issue 4) — worse than no test, because it will be trusted.
- ❌ No test covers `TableOverflowWrapper` being mounted (Issue 1). A single assertion that the tables render inside it would have caught the whole defect.
- ❌ No test or visual check covers the shared-`Tabs` change reaching `OrderEditDetailsModal` (Issue 6), which is a requirement-9 violation.
- ⚠️ 68 files changed against a 4-file affected-test scope. Even under the proposed targeted-testing policy, a `packages/shared` change requires the full suite.
- ⚠️ Manual/visual QA is explicitly pending, on a PR whose entire purpose is visual fidelity to Figma.

### Suggested manual QA script

Run as an admin, against the PR branch:

1. **Table scroll containment** (Issue 1) — narrow the viewport or widen the side nav on `/team-members` and `/customers`; confirm the table scrolls inside its own container and no column slides under the side nav.
2. **Active filter pill** (Issue 2) — apply a filter on the Orders dashboard; confirm the applied chip keeps its pill without hovering.
3. **Roles filter near the top** (Issue 3) — scroll `/team-members` so the Roles filter sits just under the header, open it, and click the topmost option.
4. **Admin page load** (Issue 5) — hard-reload `/orders`; confirm no horizontal jump or background flash as it settles.
5. **Order Edit Details modal** (Issue 6) — open it and confirm the tab labels align with the modal's content, unchanged from `develop`.
6. **Row legibility** (Issue 7) — check the gap between avatar and name on `/team-members` against Figma.
7. **Ticket acceptance** — headings at Headline 5 on all ten listed pages including Reporting / Availability / Partners (req 1, 8); card headings at Headline 5 including **Creative** account settings (req 2); selectors show only the dropdown with no leftover gap (req 3, 4); 32px above/below heading rows (req 5); 32px content padding holding as the side bar resizes and collapses (req 6); card min/max and centered layouts intact across side-bar states (req 7).
8. **Non-admin surfaces** (req 9 / testing note 8) — spot-check the creative portal for unintended changes beyond card heading sizing.
9. **Adjacent heading sections** (req 5, testing note) — find two stacked heading sections and confirm the 32px spacing does not collapse.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ❌ The PR's headline fix is dead code, it reverts its own base branch, and a shared portal z-index makes a filter unclickable |
| Regression risk | ❌ High — three shared surfaces (`Tabs`, `FilterHeadingTrigger`, `Select`) edited to serve admin pages, against requirement 9 |
| Tests | ❌ A new test asserts the opposite of its name; no coverage for the unmounted wrapper or the shared-`Tabs` leak; 68 files against a 4-file test scope |
| Accessibility | ⚠️ Page titles legitimately removed per req 3, but whether any semantic heading remains on those pages is unconfirmed (see Open Questions) |
| Error handling | ✅ n/a — no new async or failure paths |
| Security | ✅ Presentation-only; no new sinks or inputs. Author reports `/security` clean; still required per CLAUDE.md before merge |
| Code quality | ✅ Good intent — a real design token with a guard test, `TextVariant` typing, and shared wrappers replacing per-page copies |
| Validation suite | ❌ Full suite and full build not run, on a `packages/shared` change; visual QA pending on a visual-fidelity ticket |
| Mergeable state | ⚠️ `clean` against `feature/PP-1915-…` — fourth in a stack of four unmerged PRs |

---

## Recommendation

**Request changes**

1. **Wire up `TableOverflowWrapper` — or remove the whole mechanism (Issue 1).** As it stands the PR claims a fix it does not deliver, and carries a portal workaround for a container that does not exist. This is the single most important item.
2. **Restore the active-filter pill (Issue 2)** and diff this branch against `feature/PP-1915-…` for any other unintended reverts. Then decide the pill's scope deliberately, cross-referencing the #2372 review.
3. **Raise the portaled menu above the chrome it escapes (Issue 3)** with a dedicated `menuPortal` z-index token — or drop it along with Issue 1.
4. **Rename the misleading test (Issue 4)** and, separately, decide whether a non-admin should resolve to the admin area at all.
5. **Derive the gutter synchronously from `pathname` (Issue 5)** so requirement 6 holds on first paint.
6. **Move the admin gutter out of the shared `Tabs` (Issue 6)** into `AdminTabsWrapper`, restoring `4rem` as the shared default — requirement 9 forbids the current approach.
7. **Reduce the avatar column width with its alignment (Issue 7)**, confirmed against Figma.
8. **Clean up the dead class declarations and the duplicated heading preset (Issues 8, 9).**
9. **Run the full suite and the full build**, and complete the visual QA. The PR leaves four checklist items unticked, including manual testing on a ticket whose acceptance criteria are entirely visual.
10. **Sequence the stack.** This is the fourth of four dependent PRs. Land them in order, and re-verify this one against `develop` afterwards — Issue 2 shows how easily a stacked branch can undo the one below it.
