# Budgeting-app

Personal envelope-style budgeting app. Self-hosted on Mitch's own domain. Manual logging, LLM as advisor (not agent). Built around one real-life cycle: a monthly retainer funds the month; bonuses accelerate the nearest goal. A Telegram coach whispers when something needs attention; a monthly review tells the truth in plain language. n=1 — built for one person to use.

## Docs (all current as of 2026-06-08)

Three docs, read in this order:

1. **[USER-STORIES.md](USER-STORIES.md)** — product source of truth. Epics, stories, open decisions for the PRD, screen map.
2. **[DESIGN.md](DESIGN.md)** — technical contract. Architecture (self-hosted Approach B), data model, LLM contracts, animation spec, backend + frontend stack, deployment, build order.
3. **[FEATURES.md](FEATURES.md)** — user-journey walkthrough in three arcs (The Calm, The Show, The Trust Layer). 10 moments deep-dived with sequences, states, edge cases.

`reference/` is for visual references (screenshots, mockups, palette refs) — drop images there as you collect them.

## Status

Pre-build. Design aligned across all three docs. Next steps in order:

1. Resolve remaining Open Questions in DESIGN.md (hosting choice, domain, Telegram bot name).
2. Write `config.yml` by hand — real envelopes, recurring, tax rate, goals, retainer amount + cycle start. This is **the assignment** from DESIGN.md.
3. Provision the VM + domain + Caddy. Backend scaffold. Then build per the Next Steps in DESIGN.md.

## Stack

- **Frontend** — Vite + React + TypeScript, Tailwind CSS, Framer Motion, Zod, React Query
- **Backend** — Bun + Hono + Drizzle ORM + SQLite, `@anthropic-ai/sdk` server-side
- **Coach** — Telegram Bot API (outbound only, via direct HTTP)
- **Hosting** — Mitch's own domain, Caddy + Let's Encrypt for HTTPS, systemd-managed Bun process
- **LLM** — Claude Haiku (`claude-haiku-4-5-20251001`), four call types: parse, allocation propose, coach narrate, review narrate
- **Backup** — nightly SQLite snapshot to Tailscale-shared folder or S3, plus manual JSON export in Settings

See DESIGN.md for the full technical breakdown.
