# Budgeting-app

Personal envelope-style budgeting app. Manual logging, LLM as advisor (not agent), light animations on payday and goal completion. n=1 — built for one person to use.

See [DESIGN.md](DESIGN.md) for the full design — problem statement, premises, data model, LLM contracts, animation spec, build order.

## Status

Pre-build. Design approved. Next step is writing `config.yml` by hand (real envelopes, real bills, real goals) before any code lands.

## Stack (planned)

- Vite + React + TypeScript
- Tailwind CSS
- Framer Motion
- Dexie (IndexedDB)
- Zod
- js-yaml
- Anthropic SDK (`claude-haiku-4-5-20251001`)

No backend. Local-first. Phone access via Tailscale.
