# PR Review: PP-1862: Extend Creative Portal session timeouts to 24h

**PR:** https://github.com/Proofed/B2BWebserver/pull/2290
**Jira:** https://proofed.atlassian.net/browse/PP-1862
**Status:** Code Review (Jira) · Draft PR · mergeable_state: clean

---

## Jira Requirements vs Implementation

| Jira Requirement | PR Implementation | Status |
|---|---|---|
| 1.1 Idle/inactivity auto-logout threshold = 24h | `_app.tsx` `useLogoutInactiveUser(ONE_DAY_IN_SECONDS, !!user)` (was `ONE_HOUR_IN_SECONDS * 3` = 3h) | ✅ Addressed |
| 1.2 Absolute (max) server-side session lifetime = 24h | `lib/session.ts` `COOKIE_LIFE_TIME = ONE_DAY_IN_SECONDS` (was `ONE_HOUR_IN_SECONDS * 12` = 12h); flows to `req.session.expirationDate = addSeconds(now, cookieLifeTime)` in `packages/shared/api/login`, checked in `withUserProvided` | ✅ Addressed |
| 1.3 Browser session cookie valid for full 24h window | iron-session `cookieOptions.maxAge = 86400` (24h) in `packages/shared/lib/session.ts` — already 24h, unchanged | ✅ Addressed (pre-existing) |
| 2.1 Customer Portal idle & absolute timeouts unchanged | `apps/customer-portal/lib/session.ts` (72h) and `customer-portal/_app.tsx` idle (24h) untouched | ✅ Addressed |

No scope creep — the change is limited to the two creative-portal constants plus a new test.

---

## Architecture Analysis

The Creative Portal enforces three layered session limits, and this PR aligns all to 24h:

1. **Client-side idle logout** — `useLogoutInactiveUser(maxInactiveSeconds, isEnabled)` (`packages/shared/hooks/useLogoutInactiveUser.ts`) runs a 1s `setInterval`, incrementing a `useRef` counter that resets on `mousemove`/`keyup`. When the counter exceeds `maxInactiveSeconds`, it calls `postLogout()` and redirects. Driven from `_app.tsx`; changed 3h → 24h.

2. **Absolute server-side lifetime** — `COOKIE_LIFE_TIME` is passed as `cookieLifeTime` into the shared `loginRoute`, which stamps `req.session.expirationDate = addSeconds(new Date(), cookieLifeTime)` at login. On every authenticated request, `withUserProvided`'s `getServerSideProps` compares `isAfter(new Date(), expirationDate)` and redirects to login when expired. Changed 12h → 24h.

3. **Browser cookie** — iron-session `cookieOptions.maxAge = 86400` (24h) lives in the **shared** `packages/shared/lib/session.ts`, used by both portals. It was already 24h, so no change was needed; correctly left untouched (touching it would have affected Customer Portal, violating req 2.1).

The unit choices are correct: `addSeconds` consumes seconds, iron-session `maxAge` is seconds, and `ONE_DAY_IN_SECONDS = 86400`. Import swaps (`ONE_HOUR_IN_SECONDS` → `ONE_DAY_IN_SECONDS`) are clean in both files — no leftover/unused imports, and these are the only two references to the old timeout constants in the creative portal.

The approach is sound and fixes the root cause (the two shortest-winning limits), not a symptom.

---

## Issues Found

### 1. No regression-locking test for the idle-logout change (requirement 1.1)

**[File: apps/creative-portal/pages/_app.tsx]**
**Function/Class:** App (useLogoutInactiveUser call)
**Severity:** medium
**Problem:** The new `session.test.ts` locks in `COOKIE_LIFE_TIME === 86400` (requirement 1.2), but there is no test covering the idle-logout value (requirement 1.1). The 24h idle threshold is an inline literal in `_app.tsx` with no test guarding against a silent revert, while its sibling server-side value now has one. The project convention is "every PR must include tests for new code," and the author already established the pattern for one of the two values.
**Impact:** A future refactor could change/revert the idle timeout back to 3h without any test failing. The two aligned values are guarded asymmetrically.
**Fix:** Extract the idle threshold to a named, documented constant (mirroring `COOKIE_LIFE_TIME`) and assert it. For example, in `apps/creative-portal/lib/session.ts`:

```typescript
import { ONE_DAY_IN_SECONDS } from "@proofed/shared/setup/timeUnits";

// Idle/inactivity auto-logout threshold (client-side), aligned with COOKIE_LIFE_TIME.
export const IDLE_LOGOUT_SECONDS = ONE_DAY_IN_SECONDS; // 24h
export const COOKIE_LIFE_TIME = ONE_DAY_IN_SECONDS; // 24h
```

```typescript
// _app.tsx
useLogoutInactiveUser(IDLE_LOGOUT_SECONDS, !!user);
```

```typescript
// session.test.ts
it("idle logout threshold is 24 hours", () => {
  expect(IDLE_LOGOUT_SECONDS).toBe(ONE_DAY_IN_SECONDS);
});
```

### 2. Inline magic value in `_app.tsx` reduces symmetry/testability

**[File: apps/creative-portal/pages/_app.tsx]**
**Function/Class:** App
**Severity:** low
**Problem:** `useLogoutInactiveUser(ONE_DAY_IN_SECONDS, !!user)` passes the timeout inline. The server-side counterpart is a named export (`COOKIE_LIFE_TIME`) with an explanatory comment; the idle value is not. This is the root of issue #1 — resolving #1 by extracting a named constant addresses this too.
**Impact:** Minor — the two related timeouts are documented/named inconsistently, making the "all three are 24h" invariant harder to see and to test in one place.
**Fix:** Covered by issue #1's extraction. (Standalone, this is a nice-to-have, not a blocker.)

### 3. Stale comment carried forward in `lib/session.ts`

**[File: apps/creative-portal/lib/session.ts]**
**Function/Class:** COOKIE_LIFE_TIME (preceding comment)
**Severity:** low
**Problem:** The comment the PR edited still reads "...thus we set `maxAge` to undefined (making the cookie expire when session ends)." The shared iron-session config (`packages/shared/lib/session.ts`) actually sets `cookieOptions.maxAge = 86400`, not `undefined`. The comment is misleading. Since the PR already touched this exact comment block (12h → 24h), it is a natural moment to correct it. (Same staleness exists in `customer-portal/lib/session.ts` — pre-existing, out of scope here.)
**Impact:** Documentation only — could mislead a future maintainer about cookie expiry behavior.
**Fix:** Update the comment to reflect reality, e.g.:

```typescript
// COOKIE_LIFE_TIME is the absolute server-side session lifetime: it stamps
// req.session.expirationDate at login, which is verified on every request.
// The browser cookie's own maxAge (24h) is set in packages/shared/lib/session.ts.
export const COOKIE_LIFE_TIME = ONE_DAY_IN_SECONDS; // 24h
```

### 4. Client idle counter under-counts in background tabs (pre-existing)

**[File: packages/shared/hooks/useLogoutInactiveUser.ts]**
**Function/Class:** useLogoutInactiveUser
**Severity:** low
**Problem:** The idle counter is incremented by a `setInterval(…, 1000)`. Browsers throttle timers in backgrounded/inactive tabs (often to ≥1/min), so the counter drifts behind wall-clock time when the tab isn't focused. The Jira test case "idle for more than 24 hours → automatically logged out" may therefore take noticeably longer than 24h if the tab sits in the background. This is **pre-existing behavior, not introduced by this PR.**
**Impact:** Client-side idle logout is approximate. In practice the server-side absolute `expirationDate` check (now 24h) is the reliable backstop on the next navigation/SSR, so the user is still forced to re-auth — just possibly via the absolute limit rather than the idle limit.
**Fix:** No change required for this PR. Optionally, a future hardening could base inactivity on a stored last-activity timestamp (`Date.now()` delta) rather than a tick counter. Flagging only so the reviewer understands the exact-24h idle expectation in the Jira test notes is satisfied via the server-side limit, not the client timer.

### 5. Security/process notes (informational)

**[File: PR metadata]**
**Function/Class:** —
**Severity:** low
**Problem:** (a) The PR's Security checklist is fully unchecked and the body's Jira link is broken (`.../browse/PP-` with no number). (b) Extending idle 3h → 24h and absolute 12h → 24h is a deliberate reduction in session-security posture.
**Impact:** The longer session/idle windows are an explicit product decision per the Jira ("no meaningful security gain over a single 24-hour limit"), so this is accepted — but it should be acknowledged rather than slipping through with an unchecked box. The PR is also still a **draft**.
**Fix:** Run `/security` (or confirm a manual review), tick the security boxes, fix the Jira link in the description, and mark ready-for-review before merge.

---

## Tests

- ✅ New unit test `apps/creative-portal/lib/session.test.ts` locks `COOKIE_LIFE_TIME === 86400` and `=== ONE_DAY_IN_SECONDS` (requirement 1.2).
- ⚠️ No test for the idle-logout value (requirement 1.1) — see issue #1.
- ✅ Existing `packages/shared/hooks/useLogoutInactiveUser.test.ts` continues to cover the hook's mechanics (counter, reset on activity, threshold trigger, cleanup) — behavior unchanged by this value-only change.
- ⚠️ Time-based logout scenarios (idle > 24h; absolute 24h since login) are inherently manual/E2E; a manual test plan is appropriate. PR marks "Manual testing completed" but lacks an attached scenario log for the 24h thresholds.

---

## Summary

| Aspect | Status |
|---|---|
| Correctness | ✅ Aligns all three limits to 24h, correct units, root-cause fix |
| Regression risk | ✅ Low — value-only change, scope correctly limited to creative portal |
| Tests | ⚠️ Server-side value tested; idle value untested |
| Code quality | ⚠️ Inline magic value + stale comment (both minor) |
| Mergeable state | ✅ Clean (but PR is still a Draft) |

---

## Recommendation

**Approve with suggestions**

1. **(Recommended)** Extract the idle threshold to a named constant (e.g. `IDLE_LOGOUT_SECONDS` in `lib/session.ts`) and add a test asserting it equals `ONE_DAY_IN_SECONDS` — closes issues #1 and #2 and gives requirement 1.1 the same regression guard as 1.2.
2. Correct the stale "maxAge to undefined" comment in `lib/session.ts` (issue #3).
3. Run `/security`, tick the security checklist, fix the broken Jira link in the PR body, and mark the PR ready-for-review (issue #5).
4. No code change required for the background-tab idle drift (issue #4) — confirmed the server-side absolute check is the real 24h backstop.
