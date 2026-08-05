# PR Review: feature/PP-1918: New Admin Area navigation — simplified top bar and left side bar

**PR:** https://github.com/Proofed/B2BWebserver/pull/2368

**Jira:** https://proofed.atlassian.net/browse/PP-1918

**Status:** Code Review

**Head reviewed:** `b7e2a06dc73aee141b9888507055da984c8c524c` (merge of `origin/develop` into `feature/PP-1918-admin-area-navigation`, 2026-08-05)

**Scope:** 55 files, +2856 / −130, 7 commits. Consolidates PP-1918–PP-1922.

**Method:** multi-agent review at high effort (4 finder angles × independent adversarial verification of every candidate location, 35 candidates → 30 verified → 10 reported). Findings were first established against `b8af708`; the SearchBar, Header, SideNav and useResponsiveSidebar sources were re-read at head `b7e2a06` and every issue below still stands there.

---

## What this means for users (non-technical summary)

The new navigation works well on a normal desktop screen. The problems show up around the edges:

1. **On a phone or a narrow window, the admin area becomes unusable.** The check that is supposed to keep the side bar off small screens doesn't actually work — it always says "this is a desktop". Tapping the menu icon slides a 255px panel over most of the screen *and* pushes the page content sideways, so the order table is squeezed into a thin strip with no way to close the panel except the button now hidden underneath it.
2. **Reload an admin sub-page and the navigation disappears.** Open a partner's project settings from within the app and everything is fine. Press F5, or open that same link from a bookmark, and the admin side bar vanishes and the top bar switches to the creative-side links — same URL, completely different navigation.
3. **The side bar animates itself shut on every page load.** If you collapsed it, each new page paints it open first and then visibly slides it closed, dragging the page content with it. The code was written to prevent exactly this; the guard doesn't fire in time.
4. **Your collapse choice is forgotten when you close the browser** (the width is remembered, so you come back to an expanded *and* wider side bar than you left).
5. **Two changes reach screens this ticket never mentioned:** the profile avatar and the logo are now smaller for everyone — including editors and reviewers who never open the Admin Area, and on the login screen — and the search box's close (X) button has disappeared, which removes the only way to dismiss the search results without a mouse.

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1.1 Remove all nav links from the top bar | `Header` renders `SearchBar` instead of `MainNav` when `hasAdminSearch`; creative branch keeps `MainNav` | ✅ Addressed |
| 1.2 / 1.2.1 Search permanently visible, centred; open-search trigger icon removed | `StyledMainSearchBarWrapper` + `SearchBar` always rendered for admins; `useUserToolsLinks` search-trigger parameter deleted (commit `77e0b44`) | ✅ Addressed |
| 1.2.2 Placeholder copy retained | `placeholder="Search for Orders or Team Members"` | ✅ Addressed |
| 1.3 Hamburger next to logo, hover background, toggles the side bar | `SidebarToggleButton` in `StyledLogoGroup` | ⚠️ Partial — gated on `useIsDesktop()`, which is a hardcoded `() => true` stub (Issue 1) |
| 1.4 Top bar 32px side padding | Unified to 32px across creative + admin per Figma 81197:36280 | ✅ Addressed |
| 2.1 Side bar order: Orders / People (Partners, Teams, Customers) / Insights (Availability, Reporting) | `adminSideNavGroups` in `SideNav/consts.ts` matches exactly, with the specified hrefs | ✅ Addressed |
| 2.2 Omit Home and My Partners | Not present in `adminSideNavGroups` | ✅ Addressed |
| 3.1–3.3 Default / hover / selected states | `SideNavItem` states + `getActiveHref` longest-prefix matcher (single-active-link rule) | ✅ Addressed |
| 4.1 Drag to resize | `useSidebarResize` pointer engine + `SidebarResizer` (also keyboard-accessible, WAI-ARIA splitter — beyond ticket scope, good) | ✅ Addressed |
| 4.2 Min 178 / max 368 / default 255 | `ADMIN_SIDEBAR_MIN_WIDTH` 178, `MAX` 368, `DEFAULT` 255 | ⚠️ Partial — the read-side `clamp` guard does not hold for non-numeric stored values (Issue 2) |
| 4.3 32px side padding at all widths | Fixed padding on the panel | ✅ Addressed |
| 4.4 Ellipsis truncation | `SideNavItem` label truncation | ✅ Addressed |
| 4.5 Width persisted across refresh | `adminSidebar.width` in **localStorage** | ✅ Addressed |
| 5.1 Menu icon collapses/expands | `useResponsiveSidebar.toggle` via `SidebarToggleButton` | ✅ Addressed |
| 5.2 Open/collapsed persisted across refresh (≥1280px) | `adminSidebar.desktop.open` in **sessionStorage** | ⚠️ Partial — survives same-tab refresh (requirement met literally) but not a new tab or browser restart, and diverges from the localStorage width (Issue 6) |
| 5.3 / 5.3.1–5.3.3 Auto-collapse below 1280px, user can still open, no repeated auto-collapse, re-apply on down-crossing and fresh narrow load | `useResponsiveSidebar` per-regime session preferences (`desktop.open` / `narrow.open`); narrow default `false` is deliberately not configurable | ✅ Addressed — the decision table is the cleanest part of the PR |
| Validation rule 4: no two items selected at once | `getActiveHref` longest-prefix, derived from visible hrefs only | ✅ Addressed |

**Beyond Jira scope (scope creep):**

- Header **avatar** `AvatarSize.Default → Small` and **logo** `LogoSize.Default → Small` applied unconditionally, so both shrink in the creative portal and on logged-out surfaces too (Issue 10).
- `SearchBar` close button made conditional on a new `onCloseClick` prop that no caller passes, removing the button everywhere (Issue 8).
- Bar padding unified across **creative + admin** areas (designer-approved per the PR body, but outside the ticket text).
- `HEADER_HEIGHT` token consolidating six literals, `getActiveHref` moved to shared, `adminNavVisibility` extracted — all reasonable, but they widen the blast radius of a nav-only ticket.

---

## Architecture Analysis

The three-layer split is the right call and is executed well: generic mechanics (`useSidebarResize`, `useResponsiveSidebar`) and a content-agnostic shell (`SidebarResizer`, `ResizableSidebar`) live in `packages/shared`, with only nav data and wiring (`SideNav`, `useAdminSidebar`) in the app. `useResponsiveSidebar`'s pure per-regime decision table is directly traceable to ticket clauses 5.2/5.3.x and is genuinely easier to reason about than a stateful "has the user overridden" flag would have been. The resize engine's rAF ghost line with a single deferred commit on release is the correct shape, and `commit f5ac3a5` already fixed two real defects in it (pending-frame leak, right-click strand).

Three structural weaknesses run through the issues below:

1. **The viewport contract is fictional.** Every gate that is supposed to keep this push-layout sidebar off narrow screens routes through `useIsDesktop()`, which is a hardcoded `() => true` stub in `packages/shared/hooks/mediaQueries.ts`. `useResponsiveSidebar` correctly uses a real `useMediaQuery`, so the *collapse* logic is responsive while the *mount* and *toggle* are not — the two halves disagree.
2. **Hydration ordering is assumed, not enforced.** `useIsClient`, `useSessionStorage` and `useLocalStorage` all resolve in passive effects, so the "hydration gate" flips in the same batched commit as the values it is meant to gate.
3. **The sidebar's mount is gated on `areaType`,** a pre-existing piece of pathname-derived React state that was never load-bearing before. It is unchanged by this PR, but this PR is what makes its exact-match route list a navigation-visibility bug.

---

## Issues Found

### 1. `useIsDesktop` is a no-op stub, so the sidebar has no real viewport gate

**[File: apps/creative-portal/components/organisms/Header/index.tsx]**

> **In plain terms:** The side bar is designed for desktop, and there is a check meant to keep it off small screens. That check is a placeholder that always answers "yes, this is a desktop". So on a phone the menu button appears, and tapping it covers most of the screen with the nav panel while also shoving the page content sideways — leaving the order table in a thin unusable strip, with the button that would close it now hidden behind the panel.

**Function/Class:** Header (toggle render) / MainLayout (SideNav mount)

**Severity:** high

**Steps to reproduce:**

1. Log in as an admin and open `/orders`.
2. Resize the browser to ~375px wide, or open the page on a phone.
3. **Expected:** no side bar and no menu toggle at this width — the ticket scopes the side bar to desktop, with auto-collapse below 1280px.
4. **Actual:** the hamburger renders. Tap it.
5. **Actual:** a 255px panel (up to 368px if a width was saved earlier) covers ~70% of the screen, the page content is pushed the same distance right, and the whole page scrolls horizontally. There is no dimmed backdrop and no way to close the panel except the toggle now underneath it.

**Problem:** The toggle render is gated on `isDesktop`, but `packages/shared/hooks/mediaQueries.ts` is, in its entirety, `export const useIsDesktop = () => true;` / `export const useIsMobile = () => false;`. The gate never evaluates false. `MainLayout` mounts `SideNav` with no viewport condition at all, so the mount is unguarded even in principle. `useResponsiveSidebar` — the one place that reads a real media query — only decides *collapsed vs open*, not *mounted vs not*.

**Impact:** The admin area is effectively unusable on a narrow viewport. The PR's own commit message for `77e0b444` acknowledges the stub ("unreachable only while `useIsDesktop()` remains a `() => true` stub"), so the team is aware — but the sidebar ships on top of it rather than around it.

**Fix:** Don't rely on the stub. Derive the desktop signal from the same media query the sidebar already trusts, and gate the mount, not just the toggle:

```typescript
// useAdminSidebar.ts — expose the regime that useResponsiveSidebar
// already computes, instead of re-deriving it from the stub
const isDesktopViewport = useMediaQuery(
  `(min-width: ${ADMIN_SIDEBAR_DESKTOP_MIN_WIDTH_PX}px)`,
  { initializeWithValue: false }
);
```

Then either gate the `SideNav` mount on it, or give the narrow regime an overlay presentation (fixed panel + backdrop, no content `margin-left`) so the push layout never applies below the breakpoint.

### 2. `clamp` does not guard against a non-numeric persisted width — NaN propagates into layout

**[File: packages/shared/hooks/useSidebarResize.ts:57]**

> **In plain terms:** The saved side bar width is read back from the browser's storage, and there's a safety check meant to force any nonsense value back into the allowed range. That check quietly fails for values that aren't numbers at all. The result is a broken layout on every admin page — the panel collapses onto the page content and sits on top of it — and dragging the handle can't fix it. The only way out is clearing browser storage by hand.

**Function/Class:** useSidebarResize

**Severity:** medium

**Steps to reproduce:**

1. Log in as an admin, open any admin page.
2. Open DevTools → Application → Local Storage and set `adminSidebar.width` to `"abc"` (simulating corrupt or tampered storage).
3. Reload the page.
4. **Expected:** the invalid value is ignored and the side bar falls back to its 255px default.
5. **Actual:** the side bar shrinks to fit its content and overlaps the page content, which is no longer offset — the left edge of every admin page is covered.
6. Try to fix it by dragging the resize handle. **Actual:** the width never recovers; only clearing the storage key does.

**Problem:** The comment reads `// Re-clamp on read so tampered storage can never escape bounds`, but lodash `clamp` does `toNumber(value)` then `baseClamp`, whose `if (number === number)` test skips NaN — so NaN passes through unchanged. `usehooks-ts` only falls back to the default when `JSON.parse` throws, and `"abc"` / `{}` parse fine. The PR's Security section claims "all localStorage values clamped to numbers before style sinks" and "stored widths re-clamped on read" — that claim does not hold.

**Impact:** `ResizableSidebar` emits `width: NaNpx` and `MainLayout` emits `margin-left: NaNpx`; the browser drops both declarations. `SidebarResizer` renders `aria-valuenow="NaN"`. `onResizePointerDown` seeds `targetWidth = width` (NaN) and `clamp(NaN + delta, …)` is still NaN, so dragging re-commits NaN.

**Fix:** Make the read-side guard actually total:

```typescript
const toWidth = (value: unknown) =>
  Number.isFinite(Number(value))
    ? clamp(Number(value), minWidth, maxWidth)
    : defaultWidth;

const width = toWidth(storedWidth);
```

### 3. The hydration gate lands in the same commit as the restored state, so the restore animates

**[File: packages/shared/components/organisms/ResizableSidebar/index.tsx:34]**

> **In plain terms:** When you load a page, the side bar first paints in its default state and then jumps to whatever you last chose. The code has a guard meant to make that correction instant and invisible — but the guard switches on at the exact moment the correction happens, so instead of hiding it, it animates it. Every page load, a collapsed side bar visibly slides shut and drags the page content with it.

**Function/Class:** ResizableSidebar (`isHydrated`), and the same pattern in `MainLayout/useAdminSidebar.ts:53`

**Severity:** medium

**Steps to reproduce:**

1. Log in as an admin on a desktop-width window and **collapse** the side bar with the menu icon.
2. Navigate to any other admin page (or press F5).
3. **Expected:** the page renders with the side bar already collapsed — no movement.
4. **Actual:** the side bar paints open at 255px, then visibly slides shut, and the page content slides left with it.
5. Repeat with the side bar **expanded to 368px** instead: the panel paints at 255px and animates outward.

**Problem:** `useIsClient`, `useSessionStorage` and `useLocalStorage` all commit in passive `useEffect`s (verified in `node_modules/usehooks-ts/dist/index.js`). React 18 batches them, so `isHydrated: true` lands in the *same* re-render as the persisted `isOpen` / `width` — enabling `transition: width 0.2s` at exactly the moment the width changes. Meanwhile `useMediaQuery` resolves in a layout effect, i.e. pre-paint, so the first paint uses the defaults.

**Impact:** The exact behaviour the code comments claim to prevent, on every admin page load.

**Fix:** Gate on a value that is known *before* the storage values are applied — e.g. flip the flag one frame after the restore:

```typescript
const [canAnimate, setCanAnimate] = useState(false);

useEffect(() => {
  const frame = requestAnimationFrame(() => setCanAnimate(true));

  return () => cancelAnimationFrame(frame);
}, []);
```

### 4. `areaType` resets to `"creative"` on reload, so the sidebar disappears on admin sub-routes

**[File: apps/creative-portal/components/layouts/MainLayout/index.tsx:71]**

> **In plain terms:** The app decides "you're in the Admin Area" by checking the current URL against a short list of exact pages. Deeper admin pages — like a partner's project settings — aren't on that list. While you click through the app it doesn't matter, because the app remembers where you came from. But reload the page, or open the link fresh from a bookmark, and it forgets: the admin side bar disappears and the top bar shows the creative-side links instead, on an admin page.

**Function/Class:** MainLayout — `hasAdminSidebar = areaType === "admin" && isAdmin`

**Severity:** high

**Steps to reproduce:**

1. Log in as an admin and open `/partners`.
2. Click through to a partner → project → **Settings → General**. The admin side bar is present. ✅
3. Press **F5** (or copy that URL, open a new tab and paste it).
4. **Expected:** the same page with the same admin navigation.
5. **Actual:** the admin side bar is gone and the top bar shows the creative navigation links.
6. Same result on `/team/profile/[memberId]/edit-personal-info`.

**Problem:** `MainLayout/hooks.ts` (unchanged by this PR) initialises `useState<AreaType>("creative")` and only force-sets `"admin"` when `[orders, teamMembers, customers, partners, reporting, availability, teamMembersProfile].includes(pathname)` — an **exact** match against the Next.js route pattern. Admin sub-routes are not in that list.

**Impact:** The same URL renders with completely different navigation depending on how it was reached. This was latent before (it only affected which top-bar links showed); making the entire primary navigation depend on it turns it into a visible defect.

**Fix:** Derive `areaType` from a route *prefix* set rather than an exact list, and derive it during render rather than in an effect so it is correct on first paint. `adminSideNavHrefs` (already exported from `SideNav/consts.ts`, currently unused) is the natural source:

```typescript
const isAdminRoute = adminSideNavHrefs.some((href) =>
  pathname.startsWith(href)
);
```

Note the PR body already flags this: "Admin route set is enumerated in nav consts and MainLayout hooks — consolidation follow-up flagged". The consolidation is what closes this bug; it should not be deferred.

### 5. `pointerup` / `pointercancel` do not filter by `pointerId`, so a stray pointer commits the drag

**[File: packages/shared/hooks/useSidebarResize.ts:152]**

> **In plain terms:** While you drag the side bar's edge to resize it, the app watches for you letting go. It doesn't check *which* finger or pointer let go — so on a touch screen, a thumb resting on the page lifting mid-drag ends the resize early and saves the half-finished width. Carrying on dragging then does nothing, and the side bar is stuck at a width you didn't choose.

**Function/Class:** useSidebarResize — drag termination listeners

**Severity:** medium

**Steps to reproduce:**

1. On a touch device (or Chrome DevTools touch emulation with multi-touch), open an admin page as an admin.
2. Rest one finger anywhere on the page.
3. With a second finger, start dragging the side bar's right-hand resize handle.
4. Mid-drag, lift the **resting** finger.
5. **Expected:** the drag continues — that finger wasn't part of it.
6. **Actual:** the guide line disappears, the drag ends, and the current half-finished width is saved. Continuing to drag with the second finger does nothing.

**Problem:** The terminating listeners are attached to `document` with a zero-argument callback, so they cannot filter by `pointerId`:

```typescript
document.addEventListener(
  "pointerup",
  () => {
    applyGuidePosition();
    endDrag();
    setStoredWidth(targetWidth);
  },
  { signal }
);
```

The `pointermove` handler tracks a specific pointer; the termination handler does not.

**Impact:** An unintended width is persisted, and the same applies to a second mouse/pen pointer or a stylus hover-out.

**Fix:** Capture the starting `pointerId` in `onResizePointerDown` and filter both terminators against it:

```typescript
const onPointerUp = (event: PointerEvent) => {
  if (event.pointerId !== activePointerId) return;
  applyGuidePosition();
  endDrag();
  setStoredWidth(targetWidth);
};
```

`setPointerCapture` on the resizer element is the alternative, and would also let the listeners live on the element rather than `document`.

### 6. Collapse state is in sessionStorage while width is in localStorage

**[File: packages/shared/hooks/useResponsiveSidebar.ts:56,61]**

> **In plain terms:** Two halves of the same preference are remembered for different lengths of time. The width you drag the side bar to is remembered permanently; the decision to collapse it is forgotten as soon as you close the browser. So a user who collapsed it and made it wider comes back the next morning to a side bar that's expanded *and* wider than before — and has to collapse it again every day.

**Function/Class:** useResponsiveSidebar

**Severity:** medium

**Steps to reproduce:**

1. Log in as an admin on a desktop-width window.
2. Drag the side bar to its maximum 368px, then collapse it with the menu icon.
3. Refresh the page — both choices are honoured. ✅
4. **Quit the browser entirely** (or open the app in a new tab) and log back in.
5. **Expected:** side bar still collapsed.
6. **Actual:** the side bar is expanded again, and now 368px wide — the width survived, the collapse did not.

**Problem:** `desktop.open` / `narrow.open` use `useSessionStorage`; `useSidebarResize` writes `${storageKey}.width` to `useLocalStorage`. Two halves of one preference, two persistence tiers, one key prefix. The session tier is a deliberate choice (the doc comment ties it to req 5.3.3 — a fresh narrow session must start collapsed), so this is a design trade-off rather than an oversight, but it is undocumented outside that hook and contradicts the PR description, which states "all preferences client-side (`adminSidebar.open` / `adminSidebar.width` in localStorage)".

**Impact:** Requirement 5.2 is satisfied for a same-tab refresh but not for a new tab or a browser restart. Note the PR body parks "Model C (sessionStorage narrow-tier)" as "pending ticket 5.3.3 amendment", which suggests the ticket wording, not the implementation, is what needs resolving.

**Fix:** Either (a) split the tiers by regime — desktop preference in localStorage, narrow preference in sessionStorage, which satisfies 5.3.3 without discarding the desktop choice; or (b) keep sessionStorage and get the ticket amended, but then move the width to sessionStorage too so the two halves cannot disagree. Correct the PR description either way.

### 7. `hasAdminSidebar` and Header's toggle gate disagree on hidden-tools pages

**[File: apps/creative-portal/components/layouts/MainLayout/index.tsx:71]**

> **In plain terms:** Some pages — like the two-factor-authentication setup screens — deliberately hide the top bar's tools. The side bar doesn't know about that rule, so it still appears on those pages, while the button that closes it does not. The nav panel sits over the setup form, pushing it sideways, and there is nothing on the page that can dismiss it.

**Function/Class:** MainLayout — `hasAdminSidebar`

**Severity:** medium

**Steps to reproduce:**

1. Log in as an admin and open `/orders` so the admin area is active.
2. Navigate **within the app** (don't reload) to the MFA setup flow — `/authentication-method` or `/authenticator-app`.
3. **Expected:** a clean setup screen with no admin navigation, matching the hidden-tools layout.
4. **Actual:** the side bar is mounted and open at 255px, the setup form is pushed 255px right, and no toggle button is rendered anywhere on the page to collapse it.

**Problem:** `hasAdminSidebar = areaType === "admin" && isAdmin`, but Header renders `SidebarToggleButton` only when `isLoggedIn && !hasHiddenToolsLayout && isDesktop && onToggleSidebar`. `hasHiddenToolsLayout` appears in the toggle condition and not in the mount condition, so the two can disagree. (Neither MFA path is in the force-creative or force-admin lists, so `areaType` keeps its previous `"admin"` value — see Issue 4.)

**Impact:** A nav panel the user cannot dismiss, on a flow that is meant to be distraction-free.

**Fix:** Make the two gates one expression and pass it down, so they cannot drift:

```typescript
const hasAdminSidebar =
  areaType === "admin" && isAdmin && !hasHiddenToolsLayout;
```

### 8. The SearchBar close button is now unreachable, removing the only keyboard dismiss path

**[File: apps/creative-portal/components/molecules/SearchBar/index.tsx:120]**

> **In plain terms:** The X button that cleared the search box no longer appears anywhere, because it now depends on something no screen passes in. For mouse users this is a minor loss — clicking elsewhere still closes the results. For anyone navigating by keyboard it's a trap: type three characters, the results panel covers the page, and there is no Escape handler and no button to close it. The only way out is deleting every character.

**Function/Class:** SearchBar

**Severity:** medium

**Steps to reproduce:**

1. Log in as an admin and put focus in the top-bar search using **Tab only** (no mouse).
2. Type three characters, e.g. `ord`.
3. **Expected:** an X button to clear the field, and/or Escape to dismiss the results.
4. **Actual:** no X button renders at all. Press **Escape** — nothing happens. The opaque results panel stays over the page content.
5. The only way to reveal the page again is to backspace all three characters.

**Problem:** The close button was unconditional (with `onCloseClick` a required prop); it is now `{onCloseClick && (…)}`. Header — the only caller — renders `<SearchBar placeholder="Search for Orders or Team Members" />` with no `onCloseClick`, so the button never renders. `SearchBar/index.tsx` and `hooks.ts` have no `keydown`/Escape handler; the only remaining reset path is a document `mousedown` listener.

**Impact:** A keyboard-only admin cannot dismiss the results overlay. It also leaves the `onCloseClick` prop, the `aria-label="Close search"` markup and the `IconCloseFilled` import as dead code in the shipped bundle.

**Fix:** Add an Escape handler in `SearchBar/hooks.ts` regardless of the button (this is the accessible dismiss path for a combobox-style overlay), and either drop the now-unused `onCloseClick` branch or have Header pass a handler:

```typescript
const onKeyDown = (event: React.KeyboardEvent) => {
  if (event.key === "Escape") setQuery("");
};
```

### 9. `SIDE_NAV_TOP_OFFSET` does not match the real header below `tabletSm`

**[File: apps/creative-portal/components/organisms/SideNav/consts.ts:24]**

> **In plain terms:** The side bar is positioned by assuming the top bar is always 60px tall. On small screens the top bar is taller than that, so the panel starts too high — its first item slides under the header and can't be clicked, with a matching strip of empty space at the bottom. You can only get there because of Issue 1, so fixing that makes this go away.

**Function/Class:** `SIDE_NAV_TOP_OFFSET = HEADER_HEIGHT`

**Severity:** low

**Steps to reproduce:**

1. Reduce the browser width below the `tabletSm` breakpoint (~768px).
2. Open the side bar (possible only because of Issue 1).
3. **Expected:** the panel starts flush beneath the header.
4. **Actual:** the top ~10–20px of the panel — including the top of the "Orders" item — sits behind the fixed header and cannot be clicked, and an equal strip of dead space appears at the bottom of the viewport.

**Problem:** `HEADER_HEIGHT` (3.75rem) is documented as the "rendered top-bar height at tabletSm+" — `StyledHeaderContent` only applies it inside `@media ${media.tabletSm}`. Below that breakpoint the header has no fixed height (just `padding-top: 1.5rem`), so the panel's `top` / `height` do not match the actual header box.

**Impact:** Unclickable nav item on narrow screens. Only reachable because of Issue 1, which is why it is rated low.

**Fix:** Fold into the Issue 1 fix. If the sidebar is ever meant to render below `tabletSm`, make the offset breakpoint-aware rather than a single token.

### 10. Avatar and logo shrunk unconditionally, changing the creative portal and logged-out pages

**[File: apps/creative-portal/components/organisms/Header/index.tsx:195,264]**

> **In plain terms:** The ticket asks for a smaller logo and avatar in the *Admin Area* top bar. The change was applied everywhere instead — so editors and reviewers who never open the Admin Area now have a profile picture 12px smaller in both directions (and a smaller click target for the logout menu), and the logo is smaller on the login and onboarding screens too. Nobody asked for that, and a designer would normally sign it off per screen.

**Function/Class:** Header — `Logo` and `Avatar` size props

**Severity:** medium

**Steps to reproduce:**

1. Log in as an **editor or reviewer** (a non-admin who only ever sees the creative portal).
2. Compare the header avatar with `develop`.
3. **Expected:** unchanged — PP-1918 covers the Admin Area only.
4. **Actual:** the avatar is 32px instead of 44px, so the account/logout click target is 12px smaller in both dimensions.
5. Also check the **login** and **onboarding** screens while logged out: the logo is smaller there too.

**Problem:** `LogoSize.Default → LogoSize.Small` at both logo call sites and `AvatarSize.Default → AvatarSize.Small` on the dropdown avatar, with no `areaType` condition on any of them.

**Impact:** A visual regression on surfaces PP-1918 does not cover, including a smaller interactive target.

**Fix:** Gate on the area, matching how the rest of the header branches:

```typescript
<Avatar
  size={areaType === "admin" ? AvatarSize.Small : AvatarSize.Default}
  …
/>
```

Or confirm with the designer that the smaller sizes are intended platform-wide and note that in the PR description.

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⚠️ Not run | PR head `b7e2a06` could not be fetched in this environment — `git fetch` over HTTPS has no credentials, and the available SSH key (`Nandeep2750`) has no access to `Proofed/B2BWebserver`. The local `origin/feature/PP-1918-admin-area-navigation` ref is stale at `62ec36a` (4 commits behind), so validating it would not reflect the PR. |
| `npx turbo run typecheck` | ⚠️ Not run | Same. |
| `npx turbo run lint` | ⚠️ Not run | Same. |
| `npx turbo run build` | ⚠️ Not run | Same. |
| GitHub checks on `b7e2a06` | ⚠️ None | `get_check_runs` returns 0 check runs; combined status is `pending` with 0 contexts — CI provides no independent signal for this PR. |

**To unblock:** run `git fetch origin feature/PP-1918-admin-area-navigation` in your own shell (prefix with `!` in this session), then the full suite can be run against a detached worktree at the PR head.

**Author's own reported state (from the PR body and commit messages — unverified here):**

- `npx turbo run test` was never completed end-to-end by the author ("vitest hangs when backgrounded on this machine"); the PR checkbox for "All existing tests pass" is **unticked**, with an explicit request to run it before merge.
- `organisms/Header` and `organisms/SideNav` test suites **do not complete** on this branch. `b7e2a06`'s message identifies the Header root cause (`vi.mock` factories hoisted above the `createIconProxy` const, so it is in its TDZ when they run and the worker dies) and states SideNav's worker dies before collection. Both are described as pre-existing and tracked separately — but they mean the three new `SideNav` filtering tests added in `77e0b444` are, in the author's words, "unproven".
- Partial runs reported by the author: shared 1419/1419, SearchBar 11/11, SideNavItem/MainNav/MainLayout 9/9, customer-portal 412/413 (one timeout under load), eslint clean, `turbo run build` 4/4.

---

## Tests

- ✅ 44 new tests added (30 shared + 14 app-component), covering the resize engine, the responsive decision table, and SearchBar behaviours.
- ✅ `useSidebarResize` regression tests for the two defects fixed in `f5ac3a5` were verified to fail without their fix — good practice.
- ❌ **`organisms/Header` and `organisms/SideNav` suites do not run at all on this branch.** Whether or not the root cause predates the PR, this PR adds `SideNav` and materially rewrites `Header`, so the two components with the largest behavioural change in the diff have zero executed coverage.
- ❌ No test covers the NaN-width path (Issue 2), the hidden-tools gate mismatch (Issue 7), or the missing keyboard dismiss (Issue 8).
- ❌ No test asserts that the sidebar stays out of the way below the 1280px breakpoint at the **mount** level (Issue 1) — the existing coverage tests `useResponsiveSidebar` in isolation, where the media query is real, not the composed layout, where the gate is the `useIsDesktop` stub.
- ⚠️ Full suite not executed by author or reviewer — see Validation Checks.

### Suggested manual QA script

Run these against the PR branch before merge — each maps to an issue above:

1. **Narrow viewport** — at ~375px, confirm the side bar and its toggle are absent, or presented as an overlay that doesn't push content (Issue 1).
2. **Deep-link reload** — open a partner project settings URL in a fresh tab; confirm the admin side bar is present (Issue 4).
3. **Collapsed reload** — collapse the side bar, navigate to another admin page; confirm no slide animation (Issue 3).
4. **Browser restart** — collapse and widen the side bar, quit the browser, reopen; confirm both choices survive (Issue 6).
5. **MFA flow** — from `/orders`, navigate to `/authenticator-app`; confirm no side bar, or a side bar with a working toggle (Issue 7).
6. **Keyboard search** — Tab into the search field, type 3 characters, press Escape; confirm the results dismiss (Issue 8).
7. **Touch resize** — with a finger resting on the page, drag the resize handle and lift the resting finger; confirm the drag continues (Issue 5).
8. **Creative portal** — log in as an editor and compare header avatar/logo sizes with `develop` (Issue 10).

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Two high-severity defects (no real viewport gate; navigation vanishes on reload of admin sub-routes) |
| Regression risk | ❌ High — unconditional avatar/logo resize and the SearchBar close-button removal reach the creative portal and logged-out pages, which the ticket does not cover |
| Tests | ❌ The two most-changed components (`Header`, `SideNav`) have suites that do not execute |
| Code quality | ✅ Good — clean three-layer split, documented decision table, self-found and self-fixed resize defects |
| Validation suite | ⚠️ Not run (PR head unfetchable here; no CI checks on the commit) |
| Mergeable state | ❌ GitHub reports `dirty` — re-check, since head `b7e2a06` is itself a fresh develop merge and the state may not have recomputed |

---

## Recommendation

**Request changes**

1. **Fix the viewport gate (Issue 1).** Either gate the `SideNav` mount on a real media query or give the narrow regime an overlay presentation. Shipping a push-layout sidebar behind a `() => true` stub makes every admin page unusable on a phone.
2. **Fix the `areaType` reload gap (Issue 4).** Switch the admin-route test to a prefix match over `adminSideNavHrefs` and derive it during render. This is the "consolidation follow-up" the PR body defers — it should land here, because this PR is what makes it user-visible.
3. **Revert or gate the avatar and logo size changes (Issue 10)** so the creative portal and logged-out pages are untouched, and confirm the intent with the designer.
4. **Restore a dismiss path for the search overlay (Issue 8)** — an Escape handler at minimum; decide whether `onCloseClick` stays or goes.
5. **Make the width guard total (Issue 2)** and correct the PR's Security section, which currently claims a guarantee the code does not provide.
6. **Unify the two sidebar gates (Issue 7)** into a single expression, and **filter drag terminators by `pointerId` (Issue 5)**.
7. **Resolve the persistence tier (Issue 6)** — split by regime, or amend the ticket — and fix the PR description, which says localStorage for both halves.
8. **Fix the hydration gate (Issue 3)** so the restore does not animate on every load.
9. **Get `organisms/Header` and `organisms/SideNav` running before merge.** The Header root cause is already diagnosed in the head commit message (TDZ on the hoisted `vi.mock` factories) — it is a small fix and it unblocks the only coverage those components have.
10. **Run the full validation suite** (`test` / `typecheck` / `lint` / `build`) against `b7e2a06` and confirm the mergeable state, per CLAUDE.md. Neither the author nor this review has a complete run.

Issue 9 needs no separate action — the Issue 1 fix makes it unreachable.
