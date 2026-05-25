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
