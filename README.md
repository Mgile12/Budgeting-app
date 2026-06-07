# Budgeting-app

Personal envelope-style budgeting app. Self-hosted on Mitch's own domain. Manual logging, LLM as advisor (not agent). Built around one real-life cycle: a monthly retainer funds the month; bonuses accelerate the nearest goal. A Telegram coach whispers when something needs attention; a monthly review tells the truth in plain language. n=1 — built for one person to use.

## Docs (all current as of 2026-06-08)

Four docs. Start with the PRD if you want one canonical doc; read the others when you want depth on their angle.

1. **[PRD.md](PRD.md)** — single canonical product requirements doc. Executive summary, vision, scope, all functional requirements with acceptance criteria, non-functional requirements, build plan, open decisions, risks, done criteria.
2. **[USER-STORIES.md](USER-STORIES.md)** — original workshop output. Epics and stories that the PRD's acceptance criteria are derived from. Source of product truth.
3. **[DESIGN.md](DESIGN.md)** — technical contract. Architecture, data model, LLM contracts, animation spec, backend + frontend stack, deployment, build order.
4. **[FEATURES.md](FEATURES.md)** — user-journey walkthrough in three arcs (The Calm, The Show, The Trust Layer). State tables, edge cases, animation specifics.

`reference/` is for visual references (screenshots, mockups, palette refs) — drop images there as you collect them.

## Status

Pre-build. PRD approved for v1. Next steps in order:

1. Resolve Open Decisions in PRD § 15 (hosting choice, domain, Telegram bot identity, `config.yml`).
2. Write `config.yml` by hand — real envelopes, recurring, tax rate, goals, retainer amount + cycle start. **The assignment** from DESIGN.md.
3. Build per the Milestones in PRD § 14 (M0 → M13).

## Stack

- **Frontend** — Vite + React + TypeScript, Tailwind CSS, Framer Motion, Zod, React Query
- **Backend** — Bun + Hono + Drizzle ORM + SQLite, `@anthropic-ai/sdk` server-side
- **Coach** — Telegram Bot API (outbound only, via direct HTTP)
- **Hosting** — Mitch's own domain, Caddy + Let's Encrypt for HTTPS, systemd-managed Bun process
- **LLM** — Claude Haiku (`claude-haiku-4-5-20251001`), four call types: parse, allocation propose, coach narrate, review narrate
- **Backup** — nightly SQLite snapshot to Tailscale-shared folder or S3, plus manual JSON export in Settings

See PRD.md for the full breakdown.
