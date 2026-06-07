# Budgeting-app

Personal envelope-style budgeting app. Manual logging, LLM as advisor (not agent), light animations on payday and goal completion. n=1 — built for one person to use.

Three docs, read in this order:

1. **[USER-STORIES.md](USER-STORIES.md)** — current product shape. Retainer-to-retainer cycle, two income flows (retainer + bonus), Telegram coach, projection engine, monthly review. Source of truth for what v1 *is*.
2. **[DESIGN.md](DESIGN.md)** — technical design. Data model, LLM contracts, animation spec, build order. **Note:** parts now diverge from USER-STORIES.md (architecture, income model, scheduled jobs) — needs reconciliation before build.
3. **[FEATURES.md](FEATURES.md)** — user-journey walkthrough of the v1 features in three arcs (Calm, Show, Edges). Also pre-dates the user-stories evolution.

## Status

Pre-build. User stories drafted; DESIGN.md and FEATURES.md need a reconciliation pass to match the retainer-cycle + Approach B architecture. Next step is resolving Open Decisions in USER-STORIES.md and writing `config.yml`.

## Stack (current direction)

- Vite + React + TypeScript, Tailwind CSS, Framer Motion (frontend unchanged)
- Self-hosted on Mitch's own domain over HTTPS
- Always-on backend (Bun or Node) — holds the Anthropic key server-side, runs scheduled jobs, hosts the Telegram coach
- Shared datastore both web app and backend read/write (SQLite via Drizzle on the backend, the web app talks to the backend instead of writing IndexedDB directly)
- Anthropic SDK (`claude-haiku-4-5-20251001`) — server-side only
