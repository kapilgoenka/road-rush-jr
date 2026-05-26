# Architecture Decision Record — Road Rush Jr.

This document accumulates architectural decisions made during development. Each entry records the context, decision, and consequences at the time it was made.

---

## ADR-001 — Supabase as the auth and persistence backend

**Issue:** KGO-27 | **Date:** 2026-05-25 | **Status:** Accepted

### Context
Road Rush Jr. is a single-file vanilla HTML/Canvas game with no backend. Adding user logins requires an auth provider and a database without introducing a server. The project must stay deployable as a static file.

### Decision
Use the existing **Supabase project** (`wfatoxcygbtjbqtrhkbz`) as the auth and database layer, loaded via the `@supabase/supabase-js` v2 CDN script. A single `sb` client global is initialised at the top of `index.html` with the project URL and anon key. A `currentUser` global (`null` = guest) carries the resolved user identity throughout the game.

**Schema:**
- `public.profiles` — one row per auth user; stores `username`. RLS: public read, owner-only write.
- `public.high_scores` — append-only score rows per user. RLS: public read, owner-only insert.

### Alternatives considered
- **Firebase Auth** — viable but introduces a second vendor; Supabase was already connected.
- **Roll-your-own JWT** — requires a server; ruled out to keep the game fully static.
- **localStorage-only** — already in use for guests; doesn't support cross-device sync or identity.

### Consequences
- The anon key is visible in client-side source. RLS policies are the only access control layer — they must be correct before going to production.
- All future auth-dependent features (`currentUser` checks, score inserts) depend on this initialisation running before the game loop starts.
- Guest mode remains fully functional with no network calls; `currentUser === null` is the guest signal used everywhere.

---

## ADR-002 — HTML overlay for the auth form rather than canvas-drawn inputs

**Issue:** KGO-28 | **Date:** 2026-05-26 | **Status:** Accepted

### Context
The game renders entirely on a single `<canvas>`. Implementing a login/signup form required text inputs and buttons. Two approaches were considered: draw everything on canvas with custom input simulation, or use native HTML `<input>` elements absolutely positioned over the canvas.

### Decision
Use a **fixed `<div id="authOverlay">`** containing native HTML inputs, toggled visible/hidden via a CSS class (`visible`). The canvas continues to run and draws a dark background + title behind the overlay during `STATE.AUTH`. The overlay is removed from view (not from the DOM) on state transition, and inputs are cleared when re-shown.

### Alternatives considered
- **Full canvas input simulation** — manually track cursor position, draw text fields, handle keyboard events, caret blinking, selection. Much higher implementation complexity with no UX benefit for this use case.
- **Separate auth page** — redirect to a dedicated `/login.html`. Breaks the single-file constraint and requires a server-side session handoff.

### Consequences
- Native inputs give the browser's built-in autofill, password managers, and accessibility support for free.
- The overlay must be explicitly hidden when leaving AUTH state; forgetting this in any transition would leave inputs on screen during gameplay.
- Canvas scaling is independent of the overlay layout; the overlay uses `position: fixed; inset: 0` with flexbox centering, so it stays aligned regardless of canvas scale.
- `STATE.AUTH` is now the initial game state. The kick-off is async: `sb.auth.getSession()` resolves before the first `requestAnimationFrame`, so there is a brief blank frame before the auth screen or start screen appears.

---

## ADR-003 — Canvas-drawn logout button rather than a DOM element

**Issue:** KGO-29 | **Date:** 2026-05-26 | **Status:** Accepted

### Context
A logout action needs to be accessible from the START and GAME_OVER screens. The game UI is canvas-drawn. The auth form (KGO-28) already established that DOM elements can overlay the canvas, so a DOM `<button>` was a valid option.

### Decision
Draw the logout button **on the canvas** as a low-contrast text label ("Log out") at a fixed game-coordinate hit area (`LOUT_X1/X2`, `LOUT_Y1/Y2`). The existing canvas `click` handler converts viewport clicks to game coordinates and routes the hit. `drawLogoutBtn()` is a no-op when `currentUser` is null, so guests never see it.

### Alternatives considered
- **DOM `<button>` overlay** — consistent with the auth form, but adds another element to show/hide across three states (AUTH hidden, START/GAME_OVER visible, PLAYING hidden). More lifecycle management for a single action.
- **HUD icon during gameplay** — ruled out; logging out mid-game would discard the run and confuse kids.

### Consequences
- The hit area is defined in game-coordinate space and scales correctly with canvas resize, since the click handler already performs the viewport→game conversion.
- `logout()` is async (awaits `sb.auth.signOut()`). Clicking logout while a network call is in flight won't cause issues — state transitions happen in the `await` chain — but there is no loading indicator for the brief sign-out delay.
- `onAuthStateChange` fires `SIGNED_OUT` after `signOut()` resolves; the handler also clears `currentUser` as a safety net in case logout is triggered from outside the game (e.g. another tab).
