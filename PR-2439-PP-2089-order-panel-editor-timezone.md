# PR Review: fix/PP-2089: show the editor profile order panel on the editor's clock

**PR:** https://github.com/Proofed/B2BWebserver/pull/2439
**Jira:** https://proofed.atlassian.net/browse/PP-2089 (origin of the regression: https://proofed.atlassian.net/browse/PP-1905)
**Status:** Code Review · Priority Highest · labels `hotfix-candidate`, `regression`, `creative-area`, `deadlines`
**Branch:** `fix/PP-2089-order-panel-editor-timezone` → `develop` · 19 files, +287 / −41

---

## What this means for users (non-technical summary)

1. **The fix itself works and is worth having.** On an editor's profile, the order panel now shows dates on the editor's clock, matching the tables behind it. Before this change an admin in the UK looking at an editor in Sydney saw the same deadline written two different ways on one screen, with nothing to explain the gap.

2. **Editing a deadline from that panel can silently move it by an hour, twice a year.** For roughly two weeks around each daylight-saving changeover, opening the deadline calendar on a live order and pressing Apply — even without picking a new date — saves a deadline one hour away from the one displayed. Nothing warns anyone; the saved value looks perfectly normal. This is not new code, but it was harmless before and this change is what makes it reachable in everyday use.

3. **Half the disagreement the ticket describes is still on screen.** On the editor's profile the job table's deadline column shows the right date but the wrong "time remaining" beside it — off by the gap between the two people's clocks, which can be nine hours. The same mismatch decides whether that row turns red. So a row can still read "overdue" while the panel it opens says there is time left.

4. **A staged deadline can vanish while you are choosing it.** If you open an order panel from a direct link and go straight for the deadline calendar, the moment the editor's details finish loading your part-made selection is wiped and the calendar jumps back to the original date.

5. **This PR does not fix the problem PP-2089 was raised for.** The ticket is about creatives seeing wrong remaining time in their own jobs queue — five hours reading as twenty-two minutes. That screen is untouched here. The PR says so plainly, and says the ticket must stay open; anyone tracking the hotfix should know this does not close it.

---

## Jira Requirements vs Implementation

PP-2089 describes a defect on the **creative jobs queue**: in-queue deadlines are held as *minutes remaining*, a timezone conversion appears to be applied to that minutes value, and a job with 5 hrs 22 mins shows as 22 mins. This PR addresses a different surface.

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| In-queue job shows the same remaining time for every user, matching the minutes held in the back end | Not addressed. `git diff origin/develop...HEAD -- **/pages/jobs/**` is empty — the creative jobs queue is untouched | ❌ Missing |
| No job shows overdue while minutes remain (creative's own queue) | Not addressed — same surface, untouched | ❌ Missing |
| Accepted job deadlines unchanged | Not applicable to this diff; no creative-area file changed | — |
| "Users Tested" row 4 — *Admin viewing profile* | The PR claims to cover this row. In the ticket that row is a **diagnostic** used to decide severity ("Critical … if a creative sees this as themselves. Low if only an admin viewing that user's profile sees it. The last row above decides which"), not a fix to deliver. The PR does change what an admin sees on a profile, but it does not answer the diagnostic question the row exists to settle | ⚠️ Partial / mis-mapped |
| Suggested test coverage: identical remaining time with browser and profile TZ varied independently | No test varies browser TZ; no test asserts a rendered remaining-time value | ❌ Missing |

**Scope assessment.** The work is a real, well-reasoned fix for a real regression introduced by PP-1905 (whose ticket never mentions timezones — confirmed). It is simply filed against the wrong ticket. The PR is honest about this in its own description and explicitly says PP-2089 must not be closed on merge. The risk is procedural: PP-2089 is `Highest` priority and labelled `hotfix-candidate`, so a merge here can easily be mistaken for the hotfix landing.

**Recommendation on ticket hygiene:** raise a separate ticket for this work (or reopen PP-1905 as a follow-up) and leave PP-2089 for the jobs-queue defect. As it stands, the Highest-priority hotfix has a merged PR against it and no fix.

---

## Architecture Analysis

The approach is sound and is the smallest change that solves the stated problem.

`OrderSidebarProvider` gains an optional `previewWithinTimeZone`; a new `useOrderSidebarTimeZone()` in `contexts/orderSidebar/hooks.ts` resolves it against the viewer's own zone and is the single read point for every date under the panel. `useZonedTime` gains an optional `overrideTimeZone` so the resolution lives in one place rather than being re-derived per component. The orders dashboard passes nothing and falls through to the viewer's zone.

Three things are done well and are worth calling out:

- **The write path moves with the display.** `useUpdateJobReturnBase` resolves one zone and hands the same one to the tray and both pickers, so display and save agree. Sourcing them separately would have been the obvious mistake and it was avoided.
- **`JobReturnTimesTray` and `DueDatePicker` are genuinely fixed, not just plumbed.** Both took a `timeZone` prop but read `todaysDate` from the viewer's zone and then un-shifted it with the prop. Identical while the zones always matched; skewed by the offset the moment they differ. Deriving `todaysDate` from the passed zone is correct and is a prerequisite here, not scope creep.
- **`orderOverdue` was rewritten to be zone-neutral**, which is right: whether a deadline has passed is a property of the order, not of who is looking at it. The old form compared a wall-clock-shifted deadline against a real instant and was wrong whenever the viewer's browser zone differed from their profile zone.

The reuse story is clean: `useOrderSidebarTimeZone` is genuinely new (nothing in `packages/shared/utils/datetime/` or `apps/creative-portal/hooks/` does this), no shared package was touched (`git diff … -- packages/` is empty), and `DeadlineDisplay` / `ZonedTime` already accepted the props the PR passes them.

The weakness is not the design. It is that the design is asserted almost entirely in prose comments and covered almost not at all by tests, and that it activates a latent defect in the shared picker-frame helpers that the PR does not address.

---

## Issues Found

### 1. Applying a deadline from the panel writes an instant an hour off across a DST boundary

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/DetailedOrderInfo.tsx]**

> **In plain terms:** For about two weeks around each daylight-saving changeover, an admin who opens the deadline calendar on an editor's profile and presses Apply saves a deadline one hour away from the one shown on screen — even if they never picked a different date. Nothing flags it, because an hour-shifted deadline is a perfectly valid deadline. It then re-applies every time someone repeats the action while that window lasts.

**Function/Class:** `DetailedOrderInfo` — `deadlineDate` (:64-67) and the picker's `onApply` (:212-248)

**Severity:** high

**Confidence:** high

**Steps to reproduce:**

1. Log in as an admin whose browser/machine timezone is `Europe/London`.
2. Open `/team/profile/<memberId>?orderId=<id>` for an editor whose profile timezone is `Australia/Sydney`, on a **Live** order.
3. Set the date so that a DST transition in either zone falls between today and the order's deadline — e.g. today 28 Sep 2026, deadline 6 Oct 2026 (Sydney DST starts 4 Oct).
4. Click the calendar icon on the Deadline row, change nothing, click **Apply**.
5. **Expected:** the order's deadline is unchanged (or set to exactly the date shown in the picker).
6. **Actual:** the saved `dueDateTime` moves by 60 minutes.

**Problem:** The value goes *into* the picker frame with one conversion family and comes *out* with a different one, and the two are not inverses.

In, at `DetailedOrderInfo.tsx:64-67`:

```ts
const deadlineDate = useMemo(
  () => toZonedTime(`${deadlineDateTime}Z`, timeZone),
  [deadlineDateTime, timeZone]
);
```

Out, at `DetailedOrderInfo.tsx:219-224`:

```ts
const newDate = getNewDateWithUserTimezonesDiff({
  date: deadline,
  systemTimeZone: timeZone
});
```

`toZonedTime` evaluates the zone offset **at the target date**. `getNewDateWithUserTimezonesDiff` evaluates it **at `new Date()`**, because `getMinutesDiffFromTimezones` samples `const currentDate = new Date()` (`packages/shared/utils/datetime/timezones.ts:8-18`). The two agree only when the relative offset between the panel zone and the browser zone is the same now as it is on the deadline's date.

**Evidence:** measured against the repo's installed `date-fns-tz@3.2.0`, feeding a UTC instant through `toZonedTime(due, panelTz)` and recovering it with `fromPickerFrame`/`getNewDateWithUserTimezonesDiff`:

```
now 28 Sep -> due  6 Oct   panel=Australia/Sydney   drift = +60 min   (Sydney DST starts 4 Oct)
now 14 Aug -> due 20 Aug   panel=Australia/Sydney   drift =   0 min
now 20 Oct -> due 28 Oct   panel=Europe/London      drift = -60 min   (London DST ends 25 Oct)
now 14 Aug -> due 20 Aug   panel=Europe/London      drift =   0 min
```

Algebraically the recovery is exact only when `offset(panelTz, dueDate) − offset(browserTz, dueDate) == offset(panelTz, now) − offset(browserTz, now)`, which is unconditionally true when `panelTz == browserTz`.

Nothing blocks the write: `disableApply` (`:259-264`) only requires the recovered instant to be in the future, and the `orderStatus !== "Live"` guard (`:240-244`) only requires the order to be live. `onChangeDeadlineDate` then PUTs `dueDateTime: newDate.toISOString()` (`OrderManagment/index.tsx:291-296`).

**Impact:** A live order's deadline is silently mutated by a no-op interaction. Downstream buffer, overdue and job-ceiling logic all key off `dueDateTime`, so the wrong value propagates. The same mismatch affects the buffer preview (`:204-207`) and the job DD picker seed (`useJobDueDatePickers.ts:220-228`, which uses `toPickerFrame` while the tray pill beside it uses `toZonedTime`).

**Why this PR owns it:** the defect is in shared helpers and predates this branch, but on `develop` the panel zone *is* the viewer's profile zone, so for any admin whose machine matches their profile the drift cancels to zero and the bug is dormant. This PR makes "panel zone ≠ browser zone" the normal case on the editor profile — which is precisely the population the change is for — and adds southern-hemisphere DST rules to the mix. It converts a dormant defect into a reachable one on the surface it introduces.

**Fix:** Make the picker-frame helpers date-relative so both families agree. `toZonedTime` / `fromZonedTime` are already exact inverses:

```ts
// packages/shared/utils/datetime/timezones.ts
export const toPickerFrame = (date: Date, systemTimeZone: string): Date =>
  toZonedTime(date, systemTimeZone);

export const fromPickerFrame = (date: Date, systemTimeZone: string): Date =>
  fromZonedTime(date, systemTimeZone);
```

That is a shared-package change and needs the full unscoped validation suite. If it is judged too broad for this PR, the minimum is to stop `DetailedOrderInfo` mixing families — but seeding with `toPickerFrame` instead only trades the write bug for a display bug, so it is not a real fix. Note `packages/shared/utils/datetime/timezones.ts` is the only file in that directory with no `.test.ts` sibling.

---

### 2. The profile's job table still shows the countdown and the overdue colour on the viewer's clock

**[File: apps/creative-portal/components/atoms/DeadlineDisplay/index.tsx]**

> **In plain terms:** On an editor's profile the job table shows the right deadline date but the wrong "time remaining" next to it — out by the difference between the admin's clock and the editor's, which for the UK and Australia is about nine hours. The same wrong number decides whether the row turns red. So a row can read as overdue while the panel it opens says there is time left, which is the exact confusion this change set out to remove.

**Function/Class:** `DeadlineDisplay`

**Severity:** high

**Confidence:** high

**Steps to reproduce:**

1. Log in as an admin whose profile timezone is `Europe/London`.
2. Open the profile of an editor whose profile timezone is `Australia/Sydney`.
3. Find an assigned job in the Assigned/Available table with a deadline a few hours out. Note the "(Xh Ym)" suffix in the Deadline column.
4. Click the row to open the order panel and read the Deadline row there.
5. **Expected:** both read the same remaining time (the PR's stated outcome: "the panel's dates now match the Deadline column of the table behind it").
6. **Actual:** the formatted date matches, but the remaining-time suffix differs by the offset between the two zones — and the table's red/not-red state follows the wrong one.

**Problem:** `DeadlineDisplay` falls the two halves of the calculation back **independently**, at `index.tsx:20-24`:

```tsx
const { timeZone: zonedTimeZone, todaysDate: zonedTodaysDate } =
  useZonedTime();                       // always the VIEWER's zone

const currentTimeZone = timeZone ?? zonedTimeZone;
const currentTodaysDate = todaysDate ?? zonedTodaysDate;
```

`getRemainingTime` un-shifts *both* operands with the **same** `timeZone` (`packages/shared/utils/datetime/getRemainingTime.ts:24-29`), so the shifts only cancel when `todaysDate` was produced in that same zone. Pass `timeZone` without `todaysDate` and the error equals the offset between the zones.

**Evidence:** measured with the installed `date-fns-tz` — now `2026-08-14T12:00Z`, deadline `18:00Z`, panel zone `Australia/Sydney`, viewer `Europe/London`:

```
both props passed (correct) : 360 min
timeZone only (JobTable)    : 900 min   <- 9h error, rendered
```

The PR gets this right at its own call site — `DetailedOrderInfo.tsx:270-274` passes `{...{ timeZone, todaysDate }}` — but the sibling call sites on the very page it targets pass only `timeZone`:

- `components/molecules/tables/JobTable/consts.tsx:181-183`
- `components/molecules/tables/JobTable/consts.tsx:677-679`
- `components/molecules/tables/JobTable/consts.tsx:619-624` — explicitly mixed: `todaysDate` from `useZonedTime()` (viewer, line 599-600) combined with `timeZone: effectiveTimeZone` (previewed)
- `components/organisms/sidebars/contents/JobManagement/partials/JobTasks/DetailedJobInfo.tsx:41-44`

**Impact:** The PR's headline claim is only half delivered. The date string agrees; the countdown and the red threshold — which is what a creative or an admin actually triages on — still disagree between the row and the panel it opens.

**Why this PR owns it:** pre-existing on `develop`, and `JobTable` has no diff here. But it sits squarely inside the stated goal, and this PR's own change to `useZonedTime` is what makes the one-line fix possible.

**Fix:** one line in `DeadlineDisplay`, which corrects every call site at once and makes the PR's `todaysDate` prop-drill unnecessary:

```tsx
const { timeZone: zonedTimeZone, todaysDate: zonedTodaysDate } =
  useZonedTime(timeZone);
```

---

### 3. The wiring that makes the feature work is covered by neither a test nor the type system

**[File: apps/creative-portal/contexts/orderSidebar/provider.tsx]**

> **In plain terms:** The change works, but nothing would notice if it stopped working. Removing one line would put every date on the panel back on the admin's clock — undoing the whole fix — and the test suite would still pass, the build would still succeed, and no reviewer would see a red flag. Given nobody has manually tested this yet either, there is currently no mechanism that would catch a silent regression.

**Function/Class:** `OrderSidebarProvider` — context value memo

**Severity:** high

**Confidence:** high

**How to spot it:** code health, not user-reproducible on this branch. Delete `previewWithinTimeZone` from the memo object at `provider.tsx:437` and run the affected tests — everything still passes.

**Problem:** The PR's six tests cover the two ends of the chain and skip the middle.

- `MemberOrderSidebar.test.tsx:34-45` replaces `OrderSidebarProvider` with a prop-recording `div`. It proves the value is *handed to* the provider.
- `contexts/orderSidebar/hooks.test.tsx:39-45` builds a raw `<OrderSidebarContext.Provider value={{ previewWithinTimeZone } as OrderSidebarContextProps}>`, bypassing `OrderSidebarProvider` entirely. It proves the hook *reads from* the context.

Nothing covers `provider.tsx:437` (into the memo value) or `:478` (into the deps). There is no test file for `provider.tsx` at all — the folder has `utils.test.ts` and the new `hooks.test.tsx`. And the type system does not cover it either: the field is declared optional at `types.ts:90`, and the provider hands the memo over with a cast at `provider.tsx:506` (`value={value as OrderSidebarContextProps}`), so a missing property is neither an error nor a warning.

Two further gaps compound it:

- **Every `useZonedTime(timeZone)` call-site change is invisible to its own test.** All six affected test files mock the hook with a zero-parameter arrow that discards the argument — `useJobDueDatePickers.test.ts:23-32`, `useUpdateJobReturnBase.test.ts:26-31`, `useUpdateJobReturn.test.ts:32-37`, `DueDatePicker/hooks.test.ts:21-24`, `JobReturnTimesTray/index.test.tsx:94-103`, `AssignmentModalView/hooks.test.ts:30-35`. Reverting any of those edits leaves the suite green.
- **No test asserts a rendered date.** The ticket is "wrong times are displayed" and all six tests are hook plumbing or prop forwarding. The suite would pass on a build where every panel date still renders in the viewer's zone.

`orderOverdue` (`provider.tsx:374-382`) and everything in `DetailedOrderInfo.tsx` (`:81-88`, `:107-109`, `:204-207`) are likewise uncovered — neither file has a test.

**Impact:** The feature has no regression barrier. Combined with the unchecked "Manual testing completed" box, nothing between this branch and production would catch the fix being undone.

**Fix:** The cheapest test with real mutation resistance is a render-level one, and the infrastructure already exists — `DeadlineDisplay/index.stories`/`index.test.tsx:27` already drives an explicit zone. Render the panel inside a real `OrderSidebarProvider` with `previewWithinTimeZone` set and assert the visible deadline text is the editor's wall clock:

```tsx
it("renders the panel's deadline on the previewed zone", () => {
  render(
    <OrderSidebarProvider
      activeOrderId="900"
      previewWithinTimeZone="Australia/Sydney"
    >
      <OrderManagement {...props} />
    </OrderSidebarProvider>
  );

  expect(screen.getByText(/6th Oct 26 8:00pm/)).toBeInTheDocument();
});
```

That single test covers `provider.tsx:437`, the hook, and the render, and fails on the deletion above.

---

### 4. The PR description contradicts the code on `orderOverdue`, so QA will not know to retest the dashboard

**[File: apps/creative-portal/contexts/orderSidebar/provider.tsx]**

> **In plain terms:** The change notes say the overdue flag and the orders dashboard are untouched. They are not — the overdue calculation changed for everyone, and an admin whose computer clock is set to a different region from their profile will see orders flip between "overdue" and "on time" differently from before. The change is a genuine improvement, but nobody reading the description would think to test it.

**Function/Class:** `OrderSidebarProvider` — `orderOverdue`

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Log in as an admin whose **profile** timezone is `Europe/London` while their **machine** is set to `UTC`, during British Summer Time.
2. Open `/orders` and click an order whose deadline passed within the last hour.
3. **Expected (per the PR description, "nothing changes on `/orders`"):** identical behaviour to `develop`.
4. **Actual:** the panel header and its overdue warning bar now flag the order as overdue immediately, where `develop` waited an hour. (Travelling further afield inverts it — a `Europe/London` profile on a Sydney machine previously flagged orders overdue nine hours early.)

**Problem:** The description states:

> **`orderOverdue` deliberately stays on the viewer's zone.**
> … the orders dashboard passes nothing and is **behaviourally unchanged**. … On `/orders?orderId=<x>` nothing changes.

The code removes the zone from the calculation entirely rather than keeping it on the viewer's — `provider.tsx:374-382`:

```ts
return isAfter(new Date(), parseUtcDateString(order.dueDateTime));
```

against `develop`'s:

```ts
return isAfter(
  new Date(),
  new Date(convertDatetimeFromUTC(order.dueDateTime, timeZone))
);
```

`convertDatetimeFromUTC` returns a naive wall-clock string with no offset (`packages/shared/setup/timezones.ts:552-570`), so `new Date(...)` re-parsed it as browser-local and compared it against a real instant. The old predicate flipped at `dueUTC + (profileOffset − browserOffset)`.

The rewrite is correct and better — and it is documented in an inline comment at `provider.tsx:367-373`. Consumers are `OrderSidebarHeader/index.tsx:83-90` (header colour and border) and `:194-200` (the overdue warning bar). The dashboard **table** genuinely is unchanged; it computes its own overdue state from `orderTimeDiff` (`TableWithFilters/utils.ts:294`).

**Impact:** Not a code defect — a documentation defect with QA consequences. A reviewer or tester reading only the PR body would not exercise `/orders` with a mismatched browser timezone, which is the one configuration where behaviour moved.

**Fix:** Correct the description to say that `orderOverdue` becomes zone-neutral, that this is a deliberate fix to a pre-existing offset bug, and that the dashboard's side-panel header changes for admins whose browser and profile zones differ. Add that configuration to the manual test plan.

---

### 5. A part-made deadline selection is wiped when the editor's timezone arrives

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/DetailedOrderInfo.tsx]**

> **In plain terms:** Open an order panel from a direct link on an editor's profile and go straight to the deadline calendar. A moment later, when the editor's details finish loading in the background, whatever date you had picked is discarded and the calendar snaps back to the original — with no message and nothing to undo.

**Function/Class:** `DetailedOrderInfo` — the new resync effect

**Severity:** medium

**Confidence:** high

**Steps to reproduce:**

1. Paste a deep link `/team/profile/<memberId>?orderId=<id>&jobId=<id>` for a Live order and an editor whose timezone differs from yours, into a fresh tab.
2. As soon as the panel paints, click the calendar icon on the Deadline row and pick a different date. Do **not** press Apply.
3. **Expected:** the selection stays until you Apply or dismiss it.
4. **Actual:** when the member query resolves and the zone changes from the viewer's to the editor's, the selection is replaced by the order's stored deadline.

**Problem:** `DetailedOrderInfo.tsx:107-109` resyncs unconditionally, with no guard for the dropdown being open:

```ts
useEffect(() => {
  setDeadline(deadlineDate);
}, [deadlineDate]);
```

`deadlineDate` is memoised on `[deadlineDateTime, timeZone]` (`:64-67`), so it is stable across renders and across a refetch returning the same `dueDateTime` — it will not fire spuriously. But `timeZone` transitions from the viewer's zone to the member's on every deep link, because `previewWithinTimeZone` is `member?.timeZone` and is `undefined` until the member query resolves (`components/pages/team-members/profile/[memberId]/index.tsx:55-56`). The zone also traverses a second effect-sync hop in `contexts/teamProfileContext/provider.tsx:27-29`, so the panel lands two commits behind the prop. A refetch that returns a genuinely different `dueDateTime` (another admin saving in the meantime) produces the same clobber.

**Impact:** Lost work in the picker, and one committed render showing the previous zone's deadline before the effect forces a second.

**Fix:** The effect is only needed for one reader. `DetailedOrderInfo` has exactly one call site (`OrderManagment/index.tsx:268`) and it always passes `onChangeDeadlineDate`, so the read-only branch at `:279` — `format(deadline.getTime(), dateTimeFormat)` — renders only for Canceled/Complete orders. Every other reader of `deadline` (`:190`, `:217`, `:220`, `:261`) sits inside the editable branch, which already reseeds on the trigger's `onClick` at `:179`. So the simplest correct change is to read the derived value in that one row and delete the effect:

```tsx
<Text color={deadlineIsOverdue ? "red" : ""}>
  {format(deadlineDate.getTime(), dateTimeFormat)}
</Text>
```

If the effect is kept for other reasons, the staged-override shape avoids both the clobber and the extra render:

```ts
const [stagedDeadline, setStagedDeadline] = useState<Date | null>(null);
const deadline = stagedDeadline ?? deadlineDate;
```

---

### 6. Reuse-first: a validated-timezone helper already exists, and the hand-rolled guard is weaker

**[File: apps/creative-portal/hooks/useZonedTime.ts]**

> **In plain terms:** The new code checks that an editor's stored timezone is "not blank" before trusting it. A blank one is handled, but a stale or misspelled one is not — and that value is now used not just to display dates but to save them. The project already has a shared, tested check that covers both cases.

**Function/Class:** `useZonedTime`

**Severity:** medium

**Confidence:** high

**How to spot it:** code health with a latent correctness edge; not reproducible without a bad zone value on a member record. Compare `useZonedTime.ts:31` against `packages/shared/utils/isValidTimezone.ts`.

**Problem:** `useZonedTime.ts:31` guards a value the code itself describes as "an unvalidated cast over another user's record" with bare truthiness:

```ts
const timeZone = overrideTimeZone || user?.timeZone;
```

`packages/shared/utils/isValidTimezone.ts` already exists for exactly this, is tested (`isValidTimezone.test.ts`), and is already the guard used by `packages/shared/utils/datetime/getUserTime.ts:6`:

```ts
export const isValidTimezone = (timezone: string) => {
  try { return !!Intl.DateTimeFormat(undefined, { timeZone: timezone }); }
  catch { return false; }
};
```

CLAUDE.md's reuse-first convention requires searching `packages/shared/utils` before rolling a new guard. The gap is concrete, not stylistic: `||` rejects `""`, but a non-empty invalid IANA id — a renamed zone such as `"US/Pacific-New"`, or a trailing-space value — passes straight through, and `toZonedTime` then yields `Invalid Date` with no throw. Before this PR that value only fed read-only displays; it now flows into the apply path (`DetailedOrderInfo.tsx:219-224` → `onChangeDeadlineDate` → `dueDateTime: newDate.toISOString()`).

**Impact:** A malformed zone on a member record renders `Invalid Date` across the panel instead of falling back, and can reach a write.

**Fix:** Validate once at the boundary where the cast happens, so every downstream consumer benefits rather than each guarding differently — `components/pages/team-members/profile/[memberId]/index.tsx:55-56`:

```ts
const previewWithinTimeZone =
  member?.timeZone && isValidTimezone(member.timeZone)
    ? (member.timeZone as (typeof TIMEZONES)[number])
    : undefined;
```

---

### 7. `useUpdateJobReturnBase` is documented as context-free but now reads context, and is used outside the provider

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderManagment/partials/OrderJobs/hooks/useUpdateJobReturnBase.ts]**

> **In plain terms:** A shared piece of the deadline logic carries a note saying it deliberately depends on nothing outside itself, so it can be reused on the main orders list as well as in the side panel. That is no longer true. Nothing breaks today, but the note will mislead whoever touches it next.

**Function/Class:** `useUpdateJobReturnBase`

**Severity:** low

**Confidence:** high

**How to spot it:** code health, not user-reproducible. Read the docblock at `:54-58` against line 84.

**Problem:** The docblock still reads:

```
 * Context-free: takes its side effects as `handlers` so the same guards
 * can be driven from the sidebar (context-bound, via `useUpdateJobReturn`)
 * or inline from the dashboard table.
```

Line 84 now reads React context: `const { timeZone, todaysDate } = useOrderSidebarTimeZone();`.

The "inline from the dashboard table" path is real and does run outside `OrderSidebarProvider`: `TableWithFilters/partials/DeadlineCellContent/partials/InlineJobDueDate/index.tsx:43` → `useJobDueDatePickers` → `useUpdateJobReturnBase`, and `TableWithFilters` is rendered as a **sibling** of the provider in `components/pages/admin-area/orders/index.tsx` (the provider closes at ~:130, the table renders at ~:132).

**Impact:** No behaviour change today — `OrderSidebarContext` has a real default (`contexts/orderSidebar/consts.ts:14`) with no `previewWithinTimeZone`, so the out-of-provider path resolves the viewer's zone exactly as before. The risk is future: there is no "must be inside the provider" guard, so a genuine mis-mount degrades to a silent wrong-zone fallback rather than an error, and the docblock actively points the next developer the wrong way.

**Fix:** At minimum correct the docblock. Better, keep the base hook context-free by accepting `timeZone` as a parameter and letting `useUpdateJobReturn` (the sidebar binding) supply it from `useOrderSidebarTimeZone()`.

---

### 8. Three new comments state things that are not true

**[File: apps/creative-portal/hooks/useZonedTime.ts]**

> **In plain terms:** The change is heavily commented, which is good — but three of the explanations are wrong. Comments that confidently state a false fact are worse than none, because the next person will trust them instead of checking.

**Function/Class:** `useZonedTime`, `useOrderSidebarTimeZone`

**Severity:** low

**Confidence:** high

**How to spot it:** code health, not user-reproducible.

**Problem:** Three separate claims, each checkable and each wrong:

1. `useZonedTime.ts:27-28` — "`toZonedTime(x, "")` yields Invalid Date". Measured against the installed `date-fns-tz@3.2.0`, `toZonedTime(new Date("2026-08-14T12:00:00Z"), "")` returns a **valid** Date reading the UTC wall clock. The `||` is still the right operator — `new Intl.DateTimeFormat("en-US", { timeZone: "" })` genuinely throws — so an empty zone produces *silently wrong* dates rather than a visible `Invalid Date`. That is a stronger argument for the guard, not a weaker one, but the stated reason is false. The same claim is repeated at `contexts/orderSidebar/hooks.test.tsx:83-87`, and the assertion it justifies (`:95-97`, `expect(Number.isNaN(...)).toBe(false)`) is vacuous — it passes under `??` too. The two real assertions at `:93-94` do catch that mutation, so the test is not worthless, but that line should go.

2. `useZonedTime.ts:29-30` — "Every other consumer of this value guards the same way." Two do not: `components/molecules/tables/JobTable/consts.tsx:601` (`previewWithinTimeZone ?? zonedTimeZone ?? DEFAULT_TIME_ZONE`) and `components/pages/jobs/hooks.ts:275` (`previewWithinTimeZone ?? DEFAULT_TIME_ZONE`). `DeadlineDisplay/index.tsx:23` also uses `??`. This matters beyond pedantry: it means an empty zone falls back in the panel but passes through in the table, so the row and the panel disagree again — in the other direction.

3. `contexts/orderSidebar/hooks.ts:12` — "Use this rather than `useZonedTime()` anywhere under the panel." Three sites under the panel still call `useZonedTime` directly after this PR: `useJobDueDatePickers.ts`, `DueDatePicker/hooks.ts`, `JobReturnTimesTray/hooks.ts`. All three are correct (they pass the panel zone explicitly), but the directive as written reads as violated.

**Impact:** Maintenance traps in the one area of the codebase where reasoning from the comment rather than the code is most tempting.

**Fix:** Reword (1) to "an empty zone silently formats in the browser's frame, and `Intl` throws on it downstream"; delete the last sentence of (2) or make it accurate; reword (3) to "rather than the zero-argument `useZonedTime()`". Drop `hooks.test.tsx:95-97`.

---

### 9. `OrderHistory/index.tsx` regresses the UI-only convention

**[File: apps/creative-portal/components/organisms/sidebars/contents/OrderHistory/index.tsx]**

> **In plain terms:** Housekeeping only — no user impact. A file that followed the project's structure rule now breaks it.

**Function/Class:** `OrderHistory`

**Severity:** low

**Confidence:** high

**How to spot it:** code health, not user-reproducible. Compare `index.tsx` on `develop` (one hook) with this branch (two).

**Problem:** CLAUDE.md requires `index.tsx` to be UI-only, with state and hooks in a sibling `hooks.ts` exporting a single `use<ComponentName>`. Before this PR the file called exactly one hook — `useOrderHistoryContext()` at `:26` — and was compliant. `:31` adds a second:

```tsx
const { timeZone } = useOrderSidebarTimeZone();
```

A sibling `hooks.tsx` already exists in the same folder and already consumes `useOrderSidebarContext`.

**Impact:** Convention drift only.

**Fix:** Better than moving the line: `hooks.tsx:56-64` already builds the event objects via `getEvents(...)`. Attach `timeZone` there so `SingleEvent`'s props arrive complete — which also removes the double-spread at `index.tsx:129-132` and `:139` (`{...{ ...eventGroupItem, timeZone }}`), a form CLAUDE.md's spread convention does not cover (it is for forwarding several locals, not injecting one into an existing spread).

---

### 10. Provider destructure order does not match the type's field order

**[File: apps/creative-portal/contexts/orderSidebar/types.ts]**

> **In plain terms:** Housekeeping only — no user impact.

**Function/Class:** `OrderSidebarProviderProps` / `OrderSidebarProvider`

**Severity:** low

**Confidence:** high

**How to spot it:** code health, not user-reproducible.

**Problem:** `types.ts:25` places the new field before `orderGroupCountForActiveOrder`; `provider.tsx:52-53` places it after. CLAUDE.md requires the destructure to follow the interface order. Separately, all six fields are optional, so the alphabetical rule puts `previewWithinTimeZone` last, not third — though note this is a `type` alias, not an `interface`, and `packages/eslint-config/index.js:79-86` configures only `perfectionist/sort-interfaces`, so lint will not catch it either way. (`SingleLogEventProps` in `OrderHistory/types.ts` *is* an interface and *is* correctly ordered.)

**Fix:** One line, either direction.

---

## Open Questions

- The Deadline buffer preview at `DetailedOrderInfo.tsx:194-210` calls `getBufferMinutes(..., lastJobReturnTime)`, and `lastJobReturnTime` is `lastJob?.returnTime ?? ""` (`OrderManagment/index.tsx:273`). `getBufferMinutes` returns `0` when either argument is empty and clamps at `Math.max(0, …)`. Does that mean the buffer readout drops to "0h 0m" on the first calendar click when the order's last job is unassigned, and can it ever show the negative buffer the row below it renders in red? Both behaviours are pre-existing, but this PR edits that exact expression — `DetailedOrderInfo.tsx:194-210`
- Is fixing the picker-frame helpers (issue 1) in scope for this PR, or a follow-up? It is a `packages/shared` change requiring the full unscoped validation suite — `packages/shared/utils/datetime/timezones.ts:8-18`
- Product semantics of the write path: when an admin previewing Sydney picks "3pm", this PR writes the instant that is 3pm *in Sydney*. Is that intended, or should edits always be authored on the admin's own clock and only *displayed* on the editor's? The ticket does not say, and the PR's own "Note for reviewers" raises the same question about a future dual user/admin view — `DetailedOrderInfo.tsx:212-248`
- Can `member.timeZone` actually be `""` or a malformed IANA id in practice? The new test and several comments assert it can, but the OMS user-lookup contract is not verifiable from the repo. Issue 6's impact depends on this — `components/pages/team-members/profile/[memberId]/index.tsx:55-56`
- `provider.tsx:414-417` still does `isAfter(new Date(job.returnTime), new Date(order.dueDateTime))` — naive strings parsed as browser-local — thirty lines below the `parseUtcDateString` fix. Both operands shift by the same rule so the result is normally preserved; is it worth aligning for consistency, or deliberate? — `contexts/orderSidebar/provider.tsx:414-417`
- `DetailedOrderInfo.tsx:65` and `:70` build dates with the raw template `` `${deadlineDateTime}Z` `` while `:86` uses `parseUtcDateString`. If OMS ever returns a Z-suffixed value the overdue flag stays correct while every rendered date becomes `Invalid Date`. Is a Z-suffixed `dueDateTime` reachable? — `DetailedOrderInfo.tsx:64-72`
- Should `AssignmentModalView` follow the previewed zone when opened from the profile page's table? It currently falls back to the viewer's zone there (no provider above it) — unchanged by this PR, but it is now the only assignment surface on that page not on the editor's clock
- `apps/creative-portal/vitest.config.ts` pins no `TZ`, and `hooks.test.tsx:71,80,94` asserts on `Date.getHours()`. The assertions should hold for any runner zone, but pinning `TZ=UTC` would remove the environment dependency

---

## Validation Checks

| Check | Result | Notes |
|---|---|---|
| `npx turbo run test` | ⏭️ | Skipped — user opted out |
| `npx turbo run typecheck` | ⏭️ | Skipped — user opted out |
| `npx turbo run lint` | ⏭️ | Skipped — user opted out |
| `npx turbo run build` | ⏭️ | Skipped — user opted out |

Validation was not run. The working tree is dirty (26 entries) on an unrelated branch, so in-place checkout was unsafe and a fresh worktree plus full `yarn install` was judged not worth the cost for this review.

Two things are worth carrying forward regardless. The PR itself reports that `build --filter=@proofed/creative-portal` **already fails on `develop`** — 14 `Cannot find module 'msw'` errors in creative-portal `.stories.tsx` files, introduced by #2423, `msw` being a dependency of `apps/storybook` only. The PR states it verified this is identical with its own changes stashed. That is a pre-existing blocker for everyone and deserves its own ticket. Separately, this repo runs **no CI checks on PRs** (workflows fire only on `develop` and tags), so "CI is green" is never evidence here — the four gates must be run locally before merge.

Static review of formatting and hygiene was done and is clean: every changed file round-trips byte-identically through the repo's own Prettier config, imports are correctly sorted, and there are no `console.*`, `any`, `@ts-ignore`, commented-out additions, or unticketed TODOs in the added lines. No unused imports or variables were left behind — `useUserContext` was correctly removed from `OrderManagment/index.tsx` and correctly kept in `JobCard.tsx` (still needed at `:196`).

---

## Tests

- ❌ The link that makes the feature work (`provider.tsx:437`) is covered by neither test nor type system — deleting it type-checks clean and leaves all six tests green
- ❌ No test asserts a rendered date changes zone; the ticket is about wrong times being displayed
- ❌ All six existing test files for the changed hooks mock `useZonedTime` with an argument-blind arrow, so every `useZonedTime(timeZone)` edit can be reverted with the suite still green
- ❌ `orderOverdue` was rewritten with no coverage; the only test that mentions it passes it in as a prop
- ❌ `DetailedOrderInfo.tsx` has no test file, and the PR puts three behavioural changes in it — a sibling (`GeneralOrderInfo`) is tested, so the pattern exists
- ⚠️ `contexts/orderSidebar/hooks.test.tsx` — 4 tests, meaningful assertions, deterministic (verified: Vitest 2.1.9 fakes `Date` by default, and the London/Sydney hour constants are correct and system-TZ-independent). But it tests against a hand-built provider, not the real one
- ⚠️ `hooks.test.tsx:95-97` is vacuous — passes under the mutation it exists to catch; `:93-94` do the real work
- ✅ The `MemberOrderSidebar.test.tsx` `beforeEach` change does **not** weaken the four pre-existing tests (they assert router payloads and use `expect.objectContaining`)
- ⚠️ The two new `MemberOrderSidebar` tests are near-tautological — they assert a prop reaches a mocked provider, which is one line of production code
- ⚠️ Manual testing is unchecked in the PR's own checklist and needs a reviewer with an editor whose timezone differs from theirs. Given the coverage gaps above, this is the largest residual risk

**Net: 3 of 15 changed production files have any assertion touching the changed lines.**

### Suggested manual QA script

Requires two accounts: an admin, and an editor whose profile timezone is several hours away (e.g. `Australia/Sydney` vs `Europe/London`).

1. **(Verifies the fix works.)** Open the editor's profile, note a job's Deadline in the table, click the row to open the order panel. The panel's Deadline, Created on, Submission Date and History timestamps should all read the editor's clock and match the table.
2. **(Verifies issue 2.)** On the same row, compare the "(Xh Ym)" remaining time in the table against the panel's. Expect them to differ today — report the gap.
3. **(Verifies issue 2.)** Find a job the table shows in red and check whether the panel agrees it is overdue.
4. **(Verifies issue 1.)** Pick a Live order whose deadline is on the far side of the next daylight-saving changeover in either zone. Open the Deadline calendar, change nothing, press Apply. Re-open and check the saved deadline is unchanged. Expect it to move by one hour.
5. **(Verifies issue 5.)** Paste a `/team/profile/<id>?orderId=<id>` link into a fresh tab and go straight for the Deadline calendar. Pick a date but do not Apply. Expect the selection to be discarded a moment later.
6. **(Verifies issue 4.)** Set your machine clock to a different region from your Proofed profile timezone. Open `/orders`, click an order whose deadline has just passed, and check whether the panel header and warning bar agree with the table row. Compare against `develop`.
7. **(Ticket AC not covered by this PR.)** Confirm the creative jobs queue still shows wrong remaining time — PP-2089's actual defect is untouched here and must stay open.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ⚠️ Core change is right; activates a latent DST write bug and leaves half the stated goal undelivered |
| Regression risk | ⚠️ Medium — every call site verified inert on the dashboard, but `orderOverdue` genuinely changes there and is undocumented |
| Tests | ❌ The load-bearing link is covered by neither test nor types; no rendered-date assertion |
| Accessibility | n/a — no markup or interaction changes |
| Error handling | ✅ No new async or failure paths |
| Security | ✅ No new inputs, no API surface, no secrets. `/security` still required per CLAUDE.md, but this diff gives it little to chew on |
| Code quality | ✅ Clean formatting, imports, naming and reuse; three inaccurate comments and two minor convention slips |
| Validation suite | ⏭️ Skipped — user opted out. Note `build` already fails on `develop` (pre-existing `msw`), and this repo runs no CI on PRs |
| Mergeable state | ✅ Clean |

---

## Recommendation

**Request changes** — for issue 3 primarily, and a decision on issues 1 and 2.

The design is right and the reasoning behind it is better than most. The problem is that almost none of it is enforced by anything except prose, on a change whose failure mode is invisible: wrong times that look like right times.

Before merge:

1. **Add one render-level test** that mounts the panel inside a real `OrderSidebarProvider` with `previewWithinTimeZone` set and asserts a visible date reads the editor's clock. This closes issue 3's main gap — it covers `provider.tsx:437`, the hook and the render in one test, and it fails on the silent revert. Without it there is no barrier at all.
2. **Correct the PR description** on `orderOverdue` (issue 4) and add the mismatched-browser-timezone configuration to the manual test plan. Cheap, and it is what determines whether the dashboard gets retested.
3. **Decide on issue 1 (the DST write) explicitly** rather than by omission. It mutates order data. If the shared-helper fix is too broad for this PR, say so in the description, raise the follow-up ticket, and note the exposure window — do not let it merge unremarked.
4. **Decide on issue 2** — a one-line change in `DeadlineDisplay` fixes every call site, is enabled by this PR's own `useZonedTime` change, and is the difference between the stated outcome being true and half-true.
5. **Do the manual test.** The checklist is honest that it has not happened, and it needs an editor in a genuinely different zone. Given the coverage gaps, this is the only thing currently standing between the change and production.

Then, before or alongside merge:

6. Fix the three inaccurate comments (issue 8) and drop `hooks.test.tsx:95-97` — trivial, but they are in the file most likely to be reasoned about rather than read.
7. Take the reuse-first fix (issue 6) and the two convention slips (issues 9, 10) if the branch is being touched anyway.

Separately, and independent of this PR's merit:

8. **Raise a ticket for the work itself** and leave PP-2089 for the jobs-queue defect. PP-2089 is `Highest` and `hotfix-candidate`; right now it has a PR against it that does not address it. The PR says so, but a ticket board does not read PR descriptions.
9. **Raise a ticket for the `develop` build failure** (14 `Cannot find module 'msw'` errors from #2423). It blocks the build gate for everyone.
