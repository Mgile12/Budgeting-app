# PRD: Envelope Budget App

Version: 1.0
Status: APPROVED for v1 build
Last updated: 2026-06-08
Owner: Mitch (n=1)

This PRD is the single canonical product requirements document. It synthesizes the supporting docs:

- **[USER-STORIES.md](USER-STORIES.md)** — epics and stories (source of product truth)
- **[DESIGN.md](DESIGN.md)** — technical contract (data model, architecture, build order)
- **[FEATURES.md](FEATURES.md)** — three-arc journey walkthrough with state and edge specs

When this PRD and a supporting doc conflict, this PRD wins for *what* v1 is; DESIGN.md wins for *how* it's built. Supporting docs go deeper on rationale.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Vision & Principles](#2-vision--principles)
3. [Problem Statement](#3-problem-statement)
4. [User & Real-Life Context](#4-user--real-life-context)
5. [Goals & Success Metrics](#5-goals--success-metrics)
6. [Scope](#6-scope)
7. [Architecture Overview](#7-architecture-overview)
8. [Functional Requirements](#8-functional-requirements)
9. [Non-Functional Requirements](#9-non-functional-requirements)
10. [Data Model](#10-data-model)
11. [API Surface](#11-api-surface)
12. [External Integrations](#12-external-integrations)
13. [Animation & Visual Spec](#13-animation--visual-spec)
14. [Build Plan & Milestones](#14-build-plan--milestones)
15. [Open Decisions](#15-open-decisions)
16. [Risks & Mitigations](#16-risks--mitigations)
17. [Done Criteria](#17-done-criteria)
18. [Glossary](#18-glossary)
19. [Source Documents](#19-source-documents)

---

## 1. Executive Summary

A personal, self-hosted envelope-style budgeting app for one user (Mitch). Manual transaction logging via a hybrid tile-tap + free-text-with-LLM-parse interface. One monthly retainer (~$6,000) is the canonical paycheck — it funds the entire next cycle (tax → bills → subs → goals → discretionary). Every other deposit is bonus and accelerates the nearest-deadline unfunded goal. A Telegram coach speaks at most once per day with runway, goal-slip, or on-pace messages — never inbound. A monthly review delivers an LLM-narrated verdict grounded in deterministic stats. A projection engine separates "underspent" from "not spent yet" and powers the coach, money-move helper, and goal pacing.

**Why this exists:** Mitch is aware of paid alternatives (YNAB, Goodbudget, Monarch, Lunch Money, Copilot) and explicitly does not want them. The motivation is craft — a tool shaped around his real life (retainer cycle, Telegram presence, monthly review cadence), not someone else's opinion of what a budget app should be.

**Why now:** AI makes the LLM-narrated coach + review economically viable for a single-user app. Cheap hosting (Hetzner / Render / DO ~$5/mo) makes self-hosted real. The frontier of "personal software" is wide open for builders willing to ship for an audience of one.

---

## 2. Vision & Principles

**Vision.** A budgeting tool that fits one person so closely it becomes part of the routine — not because it's flashy, but because the daily ritual is quiet, the moments that matter are celebrated, and the system whispers when something needs attention.

**Principles (immutable):**

1. **Manual entry is the input contract.** No bank sync ever. Sub-2-second logging is the bar for the daily ritual.
2. **LLM is an advisor, not an agent.** Every LLM-touched action stages a proposal; nothing commits without an explicit Accept.
3. **One real budget cycle: retainer-to-retainer.** The retainer funds the month. Bonuses skip obligation math and accelerate goals.
4. **Telegram is outbound only.** The bot never accepts input. Calm presence, not chatbot.
5. **Projection over current-balance.** Coach alerts, money-move suggestions, and goal pacing all read from the projection engine, not raw balances.
6. **Animations reserved for earned moments.** Retainer = full show. Bonus = light show. Goal completion = quiet glow. Daily logging is calm.
7. **Self-hosted, single-tenant, HTTPS.** Mitch's own domain, server-side Anthropic key, no third-party data residency questions.
8. **Deterministic floor under every LLM call.** Parse, allocation, coach narration, review narration — each has a fallback that ships when the LLM fails.

---

## 3. Problem Statement

**The job Mitch has.** Convert one monthly retainer into envelope allocations that fund his life; track spending against those envelopes manually; know — without checking — whether he's drifting from where he should be; and at month end, know plainly what he did well, what he did wrong, and what to change.

**Why existing tools don't fit.**

- **YNAB / Goodbudget** — generic envelope, no opinionated retainer cycle, no LLM advisor, no Telegram coach, designed for households.
- **Monarch / Copilot** — bank-sync first, defeats the awareness Mitch wants from manual logging.
- **Lunch Money** — closer in spirit but not self-hosted, not built around the LLM narrative loop Mitch wants.
- **Spreadsheet** — what he'd fall back to. No coach, no animation, no narrative. Survives ~3 weeks before the discipline lapses.

**What makes this build worth doing despite alternatives.** Craft. The tool fits *his* life (retainer cadence, Telegram already in his pocket, monthly review as the actual moment of reflection). Building it himself means he can change it whenever life changes. The motivation isn't market opportunity; it's the thing he wants to use.

---

## 4. User & Real-Life Context

**User.** Mitch. One person. No additional users planned. No sharing, no multi-user, no auth flow for others.

**Real-life cadence:**

- **Income (predictable):** one retainer of approximately $6,000 lands around the 28th–1st each month. This is the canonical paycheck.
- **Income (irregular):** bonuses from side gigs, gifts, refunds — any amount, any time. Pure upside.
- **Spend:** rent ~$2,400/mo, subscriptions ~$80/mo, tax accrual ~25% off the top, plus discretionary categories (Coffee, Groceries, Gas, Eating Out, Misc) and goal funding (Japan Trip, Emergency Fund, etc.).
- **Devices:** primarily mobile (phone) for logging, occasionally laptop. Both browsers, real PWA possible because real HTTPS.
- **Presence:** Telegram already on the phone home screen. Used daily.

**Failure mode to defeat.** Stop logging by week three because the daily ritual feels like work. The app survives or fails on whether sub-2s logging holds up in real life.

---

## 5. Goals & Success Metrics

### Primary goals (v1)

- **G1.** Make daily logging sub-2-seconds for the most common cases (tile-tap of top 5 envelopes).
- **G2.** Make the retainer allocation a moment Mitch looks forward to once a month.
- **G3.** Catch drift before it becomes overspend via the Telegram coach.
- **G4.** Deliver a monthly review that reads like it actually knows him (not horoscope).
- **G5.** Survive 8 consecutive weeks of real use without falling back to a spreadsheet.

### Success metrics

| Metric | Target | How measured |
|---|---|---|
| M1. Days with at least one transaction | ≥ 85% (24+ of 28 days/cycle) | Count of distinct `tx_date` per cycle |
| M2. Median log-to-save time | ≤ 2.0s (tile-tap), ≤ 5.0s (free-text) | Frontend instrumentation logged to backend |
| M3. Retainer allocations accepted | ≥ 2 consecutive cycles | Count of `deposits` rows with `kind=retainer` |
| M4. Coach alerts that prompted user action | ≥ 1 per cycle (qualitative) | Self-reported in monthly review reflection |
| M5. Monthly review feels "knows me" | Mitch rates 4/5 or better after first complete cycle | Self-reported |
| M6. Fallback-to-spreadsheet | 0 events across 2 cycles | Self-reported |
| M7. Uptime of self-hosted backend | ≥ 99% across the cycle | systemd `journalctl` review |

### Anti-goals (things v1 must NOT do)

- Sync with banks.
- Replace QuickBooks for business accounting.
- Notify on every transaction.
- Use the LLM as an autonomous agent that moves money without approval.
- Support multi-user, multi-currency, or shared envelopes.
- Read input on the Telegram side.

---

## 6. Scope

### In scope (v1)

- All Foundation epic (FR-0.1 through FR-0.6)
- All Daily Logging features (FR-1.1 through FR-1.8)
- Retainer allocation flow with full show (FR-2.1 through FR-2.9)
- Bonus drop flow with light show (FR-3.1 through FR-3.3)
- Telegram coach with all six message types (FR-4.1 through FR-4.7)
- Monthly review with LLM-narrated verdict + supporting charts (FR-5.1 through FR-5.5)
- Projection engine with cold/learning/learned confidence (FR-6.1 through FR-6.3)
- Money-move helper overlay on Home with cold-start guard (FR-7.1 through FR-7.4)
- Edit transaction (amount + envelope + date), overdraw display, goal glow with re-arm, LLM-down corner icon, JSON export/import, two-tab guard (FR-8.1 through FR-8.6)

### Out of scope (v1.1)

- Override-correction logging (`corrections` table for prompt tuning)
- Large-deposit sanity check (confirm chip > $10k)
- Delete transaction (v1: edit to $0.01 to neutralize)
- Most-frequent auto-rotation of top tiles
- LLM clarifying-question re-prompt loop (v1 uses tile-picker fallback)
- LLM confidence-calibration settings toggle
- Confetti on goal completion (v1: CSS glow only)

### Out of scope (v2)

- Cadence-aware "you forgot to log a deposit" detection
- Lightweight weekly spending pulse (separate from monthly review)
- Service worker queue-and-replay for offline writes
- Manual transfer modal between envelopes (v1: covered by money-move helper for negative cases only)

### Never in scope

- Bank synchronization (Plaid, Yodlee, OFX, CSV import)
- Multi-user
- Public auth flow / OAuth
- App store distribution (PWA + Add to Home Screen is the install)
- Receipt photo capture / OCR
- Native iOS or Android app

---

## 7. Architecture Overview

```
┌─────────────────┐   HTTPS    ┌─────────────────────────────────┐
│  React PWA      │ ─────────▶ │  Bun backend                    │
│  (browser /     │ ◀───────── │  ├─ HTTP API (Hono)             │
│   home-screen)  │            │  ├─ SQLite via Drizzle          │
└─────────────────┘            │  ├─ Anthropic SDK (server-side) │
                                │  ├─ Scheduled jobs              │
                                │  └─ Telegram Bot API (outbound) │
                                └────────────┬────────────────────┘
                                             │
                                       ┌─────▼──────┐
                                       │ SQLite     │
                                       │ file       │
                                       └────────────┘
```

**Frontend:** Vite + React + TypeScript + Tailwind + Framer Motion + Zod + React Query. Single-page app. State-based view switch. Default landing: Log tab. Calls backend API for all data and LLM operations.

**Backend:** Bun runtime, Hono HTTP framework, Drizzle ORM, SQLite via `bun:sqlite`, Anthropic SDK, Telegram Bot HTTP API. Three scheduled jobs (projection refresh every 4h, coach dispatcher daily at user-configured time, monthly review generator daily-check fires on cycle start). Single process; no separate worker.

**Hosting:** Caddy reverse-proxy with auto-HTTPS via Let's Encrypt. Single VM (Hetzner CX11 / DO $6 droplet) or managed (Render / Railway). Single Bun process under systemd. SQLite file at `/var/lib/budget/budget.db`. Nightly backup snapshot to Tailscale-shared folder or S3.

**Auth:** Single bearer token (`AUTH_TOKEN`) in env. Frontend pastes token into localStorage on first visit. Every API request sends `Authorization: Bearer <token>`. n=1.

---

## 8. Functional Requirements

Each FR maps to a user story in USER-STORIES.md, restated with explicit acceptance criteria (AC) and technical realization notes. ACs are testable and observable.

### 8.1 Foundation (Epic 0)

**FR-0.1 — Self-hosted on Mitch's domain over HTTPS** _(US 0.1)_

- AC1. App is reachable at `https://<mitch-domain>` with valid TLS.
- AC2. Service worker registers successfully; PWA install prompt shows on iOS Safari.
- AC3. No reliance on Tailscale or any local-network gating.

**Realization:** Caddy + Let's Encrypt; static SPA bundle served from disk; `/api/*` reverse-proxied to Bun on `:3000`.

---

**FR-0.2 — Anthropic API key server-side only** _(US 0.2)_

- AC1. `grep -ri ANTHROPIC dist/` returns no API key.
- AC2. DevTools network tab shows no `Authorization: Bearer sk-ant-...` header from the browser.
- AC3. Stopping the backend causes every LLM-dependent feature to fall back gracefully; the frontend never makes Anthropic calls directly.

**Realization:** `ANTHROPIC_API_KEY` lives in `/etc/systemd/system/budget.service.d/secrets.conf` (mode 600). All Anthropic SDK calls happen in backend handlers.

---

**FR-0.3 — Shared datastore** _(US 0.3)_

- AC1. A transaction logged via the web app is visible to the next coach dispatcher tick.
- AC2. Backend jobs and HTTP handlers operate on the same SQLite file with proper transactional isolation.

**Realization:** Single SQLite file. All access via Drizzle. Backend uses `journal_mode=WAL` for concurrent read while jobs write.

---

**FR-0.4 — `config.yml` seed-once** _(US 0.4)_

- AC1. On first backend start, `envelopes` count is 0 → `loadFromConfig()` runs and seeds envelopes + recurring + app_config + coach_state placeholder inside one Drizzle transaction.
- AC2. On subsequent starts (count > 0), seeding is skipped entirely; no overwrite of live data.
- AC3. A malformed `config.yml` causes backend startup to fail with a clear error pointing at line/col.
- AC4. A `tax_envelope: "X"` referring to a name not in `envelopes:` causes startup to fail with an orphan-reference error.
- AC5. A partial seed (mid-transaction throw) aborts atomically; count stays 0; next start retries.

**Realization:** `loadFromConfig()` in backend bootstrap; js-yaml parse + Zod schema validation + Drizzle transaction.

---

**FR-0.5 — Integer cents everywhere** _(US 0.5)_

- AC1. All money fields in SQLite schema have `INTEGER` type.
- AC2. No floating-point math in allocation or projection logic.
- AC3. Single `formatCents(n: number): string` formatter at the render boundary.

**Realization:** Drizzle schemas declare integer columns; one util module `money.ts` handles cents↔display.

---

**FR-0.6 — Atomic balance changes** _(US 0.6)_

- AC1. Every balance-changing endpoint wraps DB writes in a Drizzle transaction.
- AC2. A simulated crash mid-transaction (kill -9 during integration test) never leaves balances inconsistent with the transactions ledger.

**Realization:** All `POST /transactions`, `PATCH /transactions/:id`, and `POST /allocation/accept` wrap in `db.transaction(async tx => {...})`.

### 8.2 Daily Logging — The Calm (Epic 1)

**FR-1.1 — App opens to Log tab every time** _(US 1.1)_

- AC1. Mount of root component sets view state to `'log'` regardless of prior state.
- AC2. Browser refresh, PWA cold start, and home-screen launch all land on Log.

**Realization:** `useState<'log' | 'home' | 'review' | 'settings'>('log')` at app root.

---

**FR-1.2 — Tile-tap quick-log** _(US 1.2)_

- AC1. Top row of Log displays 4–6 tiles per `app_config.top_tiles`.
- AC2. Tile tap opens a number-pad modal with the envelope pre-selected.
- AC3. Median time from tile tap to save toast is ≤ 2.0 seconds in real use.
- AC4. Save sends `POST /api/transactions` with `source: tile_tap`.

**Realization:** React component for tile row, modal overlay, native number-pad input.

---

**FR-1.3 — Free-text LLM parse** _(US 1.3)_

- AC1. Typing in free-text bar and pressing Enter sends `POST /api/log/parse` to backend.
- AC2. Backend returns `{intent, amount_cents, envelope, description, confidence}` validated by Zod.
- AC3. Confidence ≥ 0.7 → frontend shows confirmation chip with envelope highlighted.
- AC4. Confidence < 0.7 OR network error → frontend opens tile-picker modal with amount pre-filled.
- AC5. Pinned model is `claude-haiku-4-5-20251001`; backend uses `max_tokens: 512` and 10s `AbortController`.

**Realization:** Backend endpoint calls Anthropic SDK with structured-output prompt; Zod validates response.

---

**FR-1.4 — Confirmation chip with override** _(US 1.4)_

- AC1. Confident parse shows chip "$X.YZ → Envelope [change]".
- AC2. Auto-saves after 2.5s OR immediately on tap-elsewhere OR opens tile-picker on tapping [change].
- AC3. Override changes `envelope_id`; `source` stays `text_parse`.

**Realization:** Timed promise + click-outside handler.

---

**FR-1.5 — Tile-picker fallback (universal)** _(US 1.5)_

- AC1. Low confidence, LLM error, network failure, or malformed response all open the tile-picker.
- AC2. Tile-picker pre-fills amount when parseable; empty otherwise.
- AC3. Tile-picker has all envelopes in `order_index` order, plus search bar.

**Realization:** Single `<TilePicker amount={n} onSave={...}>` component; shared by all fallback paths.

---

**FR-1.6 — Settable tx_date (default now)** _(US 1.6)_

- AC1. Number-pad modal includes a date field defaulting to today.
- AC2. Tapping date opens a date picker; backdating up to 90 days allowed.
- AC3. Backend stores `tx_date` separate from `created_at` (server-side timestamp).
- AC4. Projection engine and coach see backdated transactions on `tx_date`, not `created_at`.

**Realization:** New `tx_date` column on `transactions`; backdate window enforced at API.

---

**FR-1.7 — Calm daily logging** _(US 1.7)_

- AC1. Save of a non-allocation transaction fires a fill-bar animation (~300ms) on the tile.
- AC2. No celebration toast, no sound, no confetti.
- AC3. Goal-target crossing fires the goal glow (see FR-8.3), not a separate daily celebration.

**Realization:** Framer Motion `animate` on the tile fill bar; toast is a quiet "$X → Envelope" line that fades after 1.5s.

---

**FR-1.8 — Glance-before-buying (large purchases)** _(US 1.8)_

- AC1. Home tile displays current balance prominently (≥ 24pt or platform equivalent).
- AC2. Mitch can check an envelope's balance in ≤ 2 taps from any tab.

**Realization:** Home tile typography; tab nav constant.

### 8.3 Retainer Allocation — The Main Event (Epic 2)

**FR-2.1 — Retainer detection** _(US 2.1)_

- AC1. Parser identifies deposit-shaped inputs ("got paid", "paycheck", "payday", "retainer" + amount).
- AC2. Allocation view top chip shows "Retainer or Bonus?" with **Retainer** highlighted by default when: amount within ±10% of `app_config.retainer_amount_cents` AND `tx_date` within `retainer_cycle_start ± 5 days`.
- AC3. One tap to toggle between modes.

**Realization:** Backend parse returns `intent: deposit`; frontend computes mode default from `app_config` + `tx_date`.

---

**FR-2.2 — Full allocation proposal** _(US 2.2)_

- AC1. `POST /api/allocation/propose` with `kind: retainer` returns `[{envelope_id, amount_cents, reason}]` covering all five rule steps in order.
- AC2. Sum of proposal amount_cents equals deposit cents exactly.
- AC3. Tax envelope receives `tax_rate * deposit_cents` rounded to cents.
- AC4. Each bill due within 30 days has its envelope topped to its `amount`.
- AC5. Each subscription envelope is topped to next-30-day total.
- AC6. Each goal envelope receives its `monthly_contribution` if `current_balance < target_balance` and `deadline > today`.
- AC7. Discretionary remainder is whatever's left.

**Realization:** `runRules()` function in backend; pure, testable, no I/O.

---

**FR-2.3 — Deterministic rules engine grounds LLM** _(US 2.3)_

- AC1. Backend runs `runRules()` baseline BEFORE the LLM call.
- AC2. LLM is given the baseline and may tweak rows.
- AC3. Zod validates the LLM response; sum-equals-deposit check.
- AC4. On any LLM error (timeout, malformed, sum mismatch), the baseline is returned as the final proposal.
- AC5. LLM-down state surfaces only via the corner icon on Log; allocation flow proceeds identically.

**Realization:** Backend pipeline: baseline → LLM → validate → fallback.

---

**FR-2.4 — Reason text per row** _(US 2.4)_

- AC1. Every proposal row has a non-empty `reason` string.
- AC2. With LLM: reasons reference specifics ("$1,200 → Rent: covers next month's bill due 2026-07-01").
- AC3. Without LLM (fallback): reasons come from rule names ("Cover bills next 30 days").
- AC4. Reason text fades in under the row in the middle column during the animation.

**Realization:** `runRules()` returns `reason` per row; LLM optionally rewrites.

---

**FR-2.5 — Inline edit with Discretionary as elastic** _(US 2.5)_

- AC1. Tapping a row amount opens an inline number-pad.
- AC2. Each edit auto-balances against Discretionary.
- AC3. If an edit would force Discretionary negative or exceed deposit cents, Save is disabled and an error chip explains the violation.
- AC4. Tax row edits are allowed; a warning chip appears.
- AC5. Negative amounts blocked.

**Realization:** Component state holds the proposal; edits trigger recompute of Discretionary; sum-invariant check on every keystroke.

---

**FR-2.6 — Money-token full show animation** _(US 2.6)_

- AC1. Total animation duration is 3–5 seconds for a typical 8-row retainer.
- AC2. Tokens stagger 300ms apart.
- AC3. Token text size scales as `log(amount_cents)`.
- AC4. Token color matches destination envelope color.
- AC5. No audio plays during the show.
- AC6. Reduced-motion preference (`prefers-reduced-motion: reduce`) replaces fly-in with same-stagger fade-in.
- AC7. Animation is driven by in-memory proposal object; no live DB queries while animating.

**Realization:** Framer Motion components with custom path keyframes; CSS `@media (prefers-reduced-motion)`.

---

**FR-2.7 — Atomic accept commit** _(US 2.7)_

- AC1. `POST /api/allocation/accept` wraps deposit insert + N transactions insert + envelope balance updates in one Drizzle transaction.
- AC2. A simulated crash mid-write rolls back entirely (verified in integration test).
- AC3. All committed transactions have `source: allocation` and a `deposit_id` linking to the newly created deposit row.
- AC4. `deposits.allocation_snapshot` stores the accepted proposal as JSON; never rewritten by future edits.

**Realization:** Drizzle transaction with all inserts/updates inside.

---

**FR-2.8 — Under-paycheck loud alarm** _(US 2.8)_

- AC1. When `tax + bills + subs + goal contributions > deposit_cents`, rules engine truncates last unmet rules (tax/bills first, then subs, then goals; discretionary stays 0).
- AC2. Yellow chip renders at top of proposal: "Bills + tax covered. Goals deferred this cycle. Discretionary: $0."
- AC3. A `runway`-kind Telegram coach message fires same day.
- AC4. The chip and message use the word "deferred" not "skipped" — semantically the goals will catch up next cycle.

**Realization:** Rules engine truncation logic; coach dispatcher reads from latest deposit + projection.

---

**FR-2.9 — Reduced-motion respects** _(US 2.9)_

- AC1. `@media (prefers-reduced-motion: reduce)` is honored by:
  - Token fly-in → fade-in (same stagger timing)
  - Envelope fill bar → snap-update
  - Goal glow → static colored border (no pulsing)
- AC2. Same information density at same pacing.

**Realization:** Centralized animation config that reads the preference.

### 8.4 Bonus Drops — The Light Show (Epic 3)

**FR-3.1 — Bonus skips obligation math** _(US 3.1)_

- AC1. `POST /api/allocation/propose` with `kind: bonus` returns a proposal that does NOT touch tax, bills, or subscription envelopes.
- AC2. Bonus does NOT trigger the under-paycheck alarm.

**Realization:** `runRules()` branches on `kind`; bonus path skips obligation rules entirely.

---

**FR-3.2 — Bonus defaults to nearest-deadline unfunded goal + light show** _(US 3.2)_

- AC1. Default proposal targets the goal envelope where `kind == "goal"`, `current_balance < target_balance`, `deadline > today`, sorted by `deadline ASC` then by `(target - current) DESC`.
- AC2. Light show duration is ~1.5 seconds; single token from paycheck node to target tile.
- AC3. If no unfunded goal exists, defaults to Discretionary with reason "no unfunded goals — held in Discretionary."

**Realization:** Selector function `nearestUnfundedGoal(envelopes, today)`; light-show animation variant.

---

**FR-3.3 — Bonus override** _(US 3.3)_

- AC1. Inline edit on the bonus proposal works identically to retainer edit (Discretionary as elastic where multiple rows exist).
- AC2. Mitch can hold the entire bonus in Discretionary by editing the goal row to $0.
- AC3. Override fully replaces the default behavior; no warning shown.

**Realization:** Same edit component as retainer mode.

### 8.5 Telegram Coach — Outbound Only (Epic 4)

**FR-4.1 — Runway warning** _(US 4.1)_

- AC1. Coach fires `kind: runway` when projection aggregate slack ≤ 0 AND ≥2 days remaining until next retainer.
- AC2. Fires at most once per crossing into danger; re-arms when aggregate slack returns to ≥ 0.
- AC3. Message includes specific numbers (projected dry date, current aggregate slack, days remaining).

**Realization:** Coach dispatcher reads from `projections` cache; `coach_state.last_fire_per_kind` deduplicates.

---

**FR-4.2 — Per-envelope runway warning** _(US 4.2)_

- AC1. Coach fires `kind: envelope_runway` when a single envelope's `projected_remaining_spend > current_balance` AND projected dry-date is ≥2 days before retainer.
- AC2. One fire per envelope per crossing; re-arms when the envelope returns to positive slack.

**Realization:** Per-envelope check inside dispatcher loop.

---

**FR-4.3 — Goal-slip warning** _(US 4.3)_

- AC1. Coach fires `kind: goal_slip` when a goal's required pace exceeds current trajectory by ≥10%.
- AC2. One fire per slip; re-arms when the goal returns to on-pace.

**Realization:** Goal-pace calculation in dispatcher.

---

**FR-4.4 — Positive on-pace ping** _(US 4.4)_

- AC1. Coach fires `kind: positive` at most once per 7 days.
- AC2. Only fires when no other condition would have fired in 6+ days (silence has been earned).
- AC3. Message format: "On pace. Nothing to do." plus a brief specific (e.g., "Coffee is right where it should be").

**Realization:** Dispatcher logic checks `last_fire_per_kind` history.

---

**FR-4.5 — Review-ready ping** _(US 4.5)_

- AC1. Coach fires `kind: review_ready` exactly once per monthly review generation.
- AC2. Message: "Your <Month> review is ready" with a link back to the app.

**Realization:** Review generator job fires the coach message after writing the `monthly_reviews` row.

---

**FR-4.6 — Nothing-logged nudge** _(US 4.6)_

- AC1. Coach fires `kind: nothing_logged` when a full local day passes with zero `transactions` rows.
- AC2. Resets on any log.
- AC3. Fires at most once per quiet day at the next dispatch window.

**Realization:** Dispatcher checks count of transactions where `tx_date == yesterday`.

---

**FR-4.7 — Cadence: ≤1 message per day, outbound only** _(US 4.7)_

- AC1. Total Telegram messages from the bot ≤ 1 per local day across all kinds.
- AC2. Priority: `runway > envelope_runway > goal_slip > review_ready > nothing_logged > positive`.
- AC3. The bot does NOT register a webhook for inbound messages; outbound `POST /sendMessage` only.
- AC4. If Mitch messages the bot, the bot does not respond.

**Realization:** Dispatcher picks one candidate per day; no inbound handler registered.

### 8.6 Monthly Review (Epic 5)

**FR-5.1 — Stand-alone surface** _(US 5.1)_

- AC1. Review is a dedicated tab in the four-tab nav.
- AC2. Review can be opened any time, regardless of allocation state.
- AC3. Current cycle's review (if generated) appears at top; prior reviews collapsed below.

**Realization:** Review tab component fetches `GET /api/review/current` + `/api/review/history`.

---

**FR-5.2 — Narrative-led verdict** _(US 5.2)_

- AC1. The first thing visible in the Review tab is a paragraph of plain-language verdict.
- AC2. The verdict states what was done well, what was done poorly, and what to change.
- AC3. Charts (if any) appear below the verdict, not above.

**Realization:** Component layout: narrative `<section>` at top, charts after.

---

**FR-5.3 — Behavioral + goal-pace coverage** _(US 5.3)_

- AC1. Narrative addresses spending behavior patterns (e.g., "you overspend in the days after you skip logging").
- AC2. Narrative addresses each goal's pace and required adjustment if slipped.
- AC3. Patterns are grounded in actual stats from the cycle, not LLM speculation.

**Realization:** `deterministic_stats` blob includes pattern indicators (e.g., spend variance per log-gap, goal pace deltas); LLM narrates from these.

---

**FR-5.4 — Deterministic numbers** _(US 5.4)_

- AC1. Every numeric figure mentioned in the narrative exists in the underlying `deterministic_stats` blob.
- AC2. The LLM prompt explicitly forbids inventing figures.
- AC3. An integration test parses narrative numbers and verifies they exist in stats (best-effort regex check).

**Realization:** Strict prompt + post-generation lint that flags numbers not in stats.

---

**FR-5.5 — Small supporting charts** _(US 5.5)_

- AC1. At most 2 charts in the Review.
- AC2. Charts support the narrative (spend by envelope vs avg; goal pace over cycle).
- AC3. Charts are not the leading visual element.

**Realization:** Recharts or hand-rolled Tailwind charts; intentionally simple.

### 8.7 Projection Engine (Epic 6)

**FR-6.1 — Per-envelope spend cadence + projection** _(US 6.1)_

- AC1. For each envelope, projection calculates `avg_cycle_spend` from historical cycles up to `cycle_avg_window_cycles`.
- AC2. `projected_remaining_spend = function of avg_cycle_spend, cycle_fraction_elapsed, confidence`.
- AC3. Underspent (still has budget but spending below pace) is distinguished from on-pace (spending at cycle avg).

**Realization:** Projection job runs every 4h; results cached in `projections` table.

---

**FR-6.2 — Slack + runway** _(US 6.2)_

- AC1. Per-envelope `slack = current_balance - projected_remaining_spend`.
- AC2. Aggregate runway = sum of slack across non-goal envelopes.
- AC3. Days-to-retainer derived from `app_config.retainer_cycle_start` and today.
- AC4. Coach reads from these values, not raw balances.

**Realization:** Projection job; coach dispatcher reads `projections` cache.

---

**FR-6.3 — Cold-start window** _(US 6.3)_

- AC1. Envelope confidence is `cold` until it has at least one complete cycle of history.
- AC2. Confidence is `learning` with 1–2 cycles OR variance ≥ 25% across cycles.
- AC3. Confidence flips to `learned` at ≥ 3 cycles AND variance < 25%.
- AC4. Cold envelopes use conservative projection (recurring-amount-based) and prevent money-move auto-pulls.

**Realization:** Confidence stored as `'cold' | 'learning' | 'learned'` on each `projections` row.

### 8.8 Money-Move Helper (Epic 7)

**FR-7.1 — Quiet dismissible chip on overdraw** _(US 7.1)_

- AC1. When an envelope's balance goes negative, a chip appears on its Home tile.
- AC2. The chip is dismissible (small × button); dismissal persists until next negative crossing.
- AC3. The chip is non-blocking; Mitch can continue using the app normally.

**Realization:** Per-tile chip component; `dismissed_at` tracked in component or `coach_log`.

---

**FR-7.2 — Advisor not agent** _(US 7.2)_

- AC1. Move button stages a transfer (one debit + one credit); does NOT execute until user confirms (or in cold-start case, requires explicit confirm step).
- AC2. Decline button hides the chip without writing anything.

**Realization:** `POST /api/transfers` endpoint requires explicit body confirming source/destination/amount.

---

**FR-7.3 — Pull from genuine slack** _(US 7.3)_

- AC1. Helper proposes source envelope with highest `slack` AND `confidence: learned`.
- AC2. Helper never proposes an envelope whose projection says it's about to spend more than its balance.
- AC3. If no source has positive slack, helper proposes Discretionary with a "tight cycle" note.

**Realization:** Selector function `proposeSource(envelopes, projections)`.

---

**FR-7.4 — Cold-start guard** _(US 7.4)_

- AC1. If the envelope receiving the rebalance has `confidence: cold`, helper only proposes Discretionary as source.
- AC2. If source has `confidence: cold`, helper requires a Confirm step (not one-tap commit).

**Realization:** Branching in helper logic.

### 8.9 The Edges — Trust Layer (Epic 8)

**FR-8.1 — Edit transaction (amount, envelope, date)** _(US 8.1)_

- AC1. Tap a transaction row in an envelope's recent list to open Edit modal.
- AC2. Editable fields: amount, envelope, tx_date.
- AC3. `PATCH /api/transactions/:id` wraps reverse-delta + apply-new-delta + row-update in one Drizzle transaction.
- AC4. `deposit_id` is preserved on edit; `deposits.allocation_snapshot` is never rewritten.
- AC5. Editing through `celebrated_at` threshold updates the celebration flag (sets on crossing up, clears on crossing down; clear is silent).
- AC6. v1 does NOT support delete; edit to $0.01 is the documented workaround.

**Realization:** Backend handler computes deltas + updates atomically.

---

**FR-8.2 — Overdraw indicator** _(US 8.2)_

- AC1. Tile renders in red when `current_balance < 0`.
- AC2. Label shows "Overdrawn -$X".
- AC3. Save of new transactions is NOT blocked when an envelope is negative.

**Realization:** Conditional Tailwind class on tile component.

---

**FR-8.3 — Goal glow with re-arm** _(US 8.3)_

- AC1. Glow fires when `current_balance` first crosses `target_balance` and `celebrated_at` is null.
- AC2. Glow runs once per session per envelope; reload shows "funded" badge without re-glow.
- AC3. Balance dropping below target clears `celebrated_at`; refunding re-fires the glow on the next crossing.
- AC4. Multi-goal hit in one allocation: glows stagger 1.5s apart in `order_index` order, AFTER Accept commits.

**Realization:** Backend sets `celebrated_at` inside the crossing transaction; frontend triggers CSS keyframe on detect.

---

**FR-8.4 — LLM-down fallback + corner icon** _(US 8.4)_

- AC1. Every LLM-dependent path (parse, allocation propose, coach narrate, review narrate) has a deterministic fallback that runs end-to-end on backend failure.
- AC2. A subtle 8px corner icon on the Log input bar appears when the most recent LLM call errored.
- AC3. Icon auto-clears on the next successful LLM call.
- AC4. Icon does NOT show modal, alarm, or banner.

**Realization:** Backend tracks `last_llm_call_status` in singleton or middleware; frontend polls or reads with `GET /api/envelopes` response metadata.

---

**FR-8.5 — JSON export/import + backup chip** _(US 8.5)_

- AC1. Settings has "Export JSON" button that downloads all eight tables as a single JSON file.
- AC2. Settings has "Import JSON" flow: pick file → Zod-validate → wipe + restore inside one transaction.
- AC3. Settings displays "Last backup: N days ago" read from nightly snapshot file mtime.
- AC4. Backup chip is informational, not a nag modal.
- AC5. Corrupt JSON import is refused with a clear error.

**Realization:** Backend endpoints `POST /api/export` and `POST /api/import`; cron job runs nightly SQLite `.backup`.

---

**FR-8.6 — Two-tab guard** _(US 8.6)_

- AC1. Opening a second tab while one is already active renders "Already open in another tab. Refresh to take over."
- AC2. Second tab stays read-only until refresh.
- AC3. Cross-device (phone + laptop) is NOT covered; documented as accepted limitation.

**Realization:** `BroadcastChannel('budget-app-v1')` pings on mount; second tab listens for response.

---

## 9. Non-Functional Requirements

**NFR-1 — Performance**

- Tile-tap save latency: median ≤ 2.0s, p95 ≤ 3.5s.
- Free-text parse latency: median ≤ 5.0s, p95 ≤ 8.0s.
- Allocation proposal latency: median ≤ 5.0s, p95 ≤ 12.0s.
- Cold app start (PWA): ≤ 2s to interactive on phone over 4G.
- Backend API endpoints (non-LLM): p95 ≤ 200ms.

**NFR-2 — Security**

- HTTPS-only; HTTP requests redirect to HTTPS.
- Bearer token rotation requires editing systemd env + restarting backend.
- Anthropic API key never present in frontend bundle.
- `config.yml` is `.gitignored` (real financial numbers stay out of git).
- Telegram bot token stored only in backend env.

**NFR-3 — Reliability**

- Backend uptime ≥ 99% per cycle.
- SQLite WAL mode enabled for concurrent read.
- Nightly automated backup verified weekly by restore-to-temp test (manual or scripted).
- Every balance-changing operation atomic.

**NFR-4 — Accessibility**

- Reduced-motion preference honored on all animations.
- Touch targets ≥ 44×44 dp on phone.
- Color contrast ≥ WCAG AA on all text/background pairs.

**NFR-5 — Privacy**

- No analytics, no telemetry, no third-party scripts.
- Backup files contain financial data; storage location (Tailscale, S3) is Mitch's choice and must be encrypted-at-rest.

**NFR-6 — Maintainability**

- Backend code structured: `routes/`, `db/`, `services/` (rules, projection, coach, review), `lib/` (money, anthropic, telegram).
- Drizzle migrations checked into git.
- Integration tests cover atomic-transaction paths (allocation accept, edit transaction).

---

## 10. Data Model

Eight SQLite tables via Drizzle ORM. Full schema specifications in [DESIGN.md § Data Model](DESIGN.md#data-model). Summary:

| Table | Purpose | Key fields |
|---|---|---|
| `envelopes` | The buckets | id, name, current_balance (cents), target_balance, deadline, kind, order_index, celebrated_at |
| `recurring` | Bills + subs (informational) | id, name, amount, cadence, next_due, envelope_id, kind |
| `transactions` | Editable ledger | id, tx_date, created_at, amount, envelope_id, description, source, deposit_id |
| `deposits` | Frozen allocation snapshots | id, ts, gross_amount, kind, allocation_snapshot (JSON) |
| `app_config` | Singleton config | schema_version, allocation_rule, top_tiles, tax_rate, retainer_amount_cents, retainer_cycle_start |
| `projections` | Cached projection per envelope | envelope_id, computed_at, current_balance, avg_cycle_spend, projected_remaining_spend, slack, confidence |
| `coach_log` | Sent message history | id, ts, kind, payload, telegram_message_id |
| `coach_state` | Singleton coach config | chat_id, enabled, dispatch_time_local, last_fire_per_kind |
| `monthly_reviews` | Generated reviews | id, cycle_start, cycle_end, generated_at, llm_narrative, deterministic_stats |

**Money:** integer cents everywhere. **Timestamps:** ISO 8601 with device local offset. **Schema versioning:** Drizzle migrations; lock v1 shape.

---

## 11. API Surface

```
GET    /api/envelopes                  → current balances + projection snapshot
GET    /api/transactions?envelope=X    → recent transactions per envelope
POST   /api/transactions               → log one
PATCH  /api/transactions/:id           → edit one (amount, envelope_id, tx_date)

POST   /api/log/parse                  → free-text → structured (LLM)
POST   /api/deposits                   → log deposit + start allocation flow
POST   /api/allocation/propose         → returns proposal (LLM + rules)
POST   /api/allocation/accept          → commits N transactions atomically

GET    /api/projection                 → cached projection snapshot
POST   /api/projection/refresh         → on-demand recompute

GET    /api/review/current             → current cycle review
GET    /api/review/history             → prior reviews

POST   /api/coach/connect              → set chat_id + bot token
PATCH  /api/coach/settings             → enable/disable, dispatch_time

POST   /api/transfers                  → manual transfer (money-move helper)
POST   /api/export                     → download JSON of all tables
POST   /api/import                     → restore from JSON

GET    /api/backup-status              → "last backup N days ago" data
GET    /api/health                     → uptime check
```

All endpoints require `Authorization: Bearer <token>`. 401 on missing/invalid token.

---

## 12. External Integrations

### 12.1 Anthropic API

- **Model:** `claude-haiku-4-5-20251001`
- **SDK:** `@anthropic-ai/sdk` (server-side only)
- **Auth:** `ANTHROPIC_API_KEY` env var
- **Four call types:**
  - `text-parse`: max_tokens 512, 10s timeout
  - `allocation-propose`: max_tokens 2048, 10s timeout
  - `coach-narrate`: max_tokens 256, 10s timeout
  - `review-narrate`: max_tokens 2048, 10s timeout
- **Validation:** Zod schemas at every boundary.
- **Fallback:** every call type has a deterministic alternative that ships on failure.
- **Cost guard:** monthly spend cap set in Anthropic console; alert at 75%.

### 12.2 Telegram Bot API

- **Method:** direct HTTP via fetch (no SDK).
- **Endpoint:** `POST https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage`
- **Auth:** `TELEGRAM_BOT_TOKEN` env var; `chat_id` from `coach_state`.
- **Outbound only:** no webhook registered; bot does not receive messages.
- **Setup:** one-time via @BotFather (token), `/start` from Mitch's account (chat_id). Both saved in Settings.

---

## 13. Animation & Visual Spec

Full choreography in [FEATURES.md § Arc 2](FEATURES.md#22-loading-the-proposal). Summary:

| Animation | Duration | Trigger | Style |
|---|---|---|---|
| Envelope fill bar (daily) | ~300ms | balance change | Framer Motion `animate` on progress bar |
| Goal completion glow | ~3s | `current_balance` crosses `target_balance`, `celebrated_at` is null | CSS keyframe ring around tile |
| Retainer full show | 3–5s | Allocation Accept-eligible | Money tokens, 300ms stagger, log-scaled text, color-matched, silent |
| Bonus light show | ~1.5s | Bonus allocation Accept-eligible | Single token, same easing, silent |
| Reduced-motion variants | same timing | `prefers-reduced-motion: reduce` | Fade-in replaces fly-in; snap replaces fill bar; static border replaces glow |

**Visual identity (not yet spec'd):** colors, typography, spacing live in `reference/` images (to be added by Mitch) and will be locked during the design pass on the frontend.

---

## 14. Build Plan & Milestones

Sequential, backend-first. Hour estimates are human-solo; with Claude Code driving boilerplate they roughly halve.

| # | Milestone | Hours (solo) | Output |
|---|---|---|---|
| M0 | Write `config.yml` (real numbers) + decide hosting, domain, Telegram bot identity | 30–60 min | `config.yml`, Open Decisions resolved |
| M1 | Provision VM, domain, Caddy with HTTPS | 30–60 min | App reachable at `https://<domain>` returns 404, TLS valid |
| M2 | Backend scaffold (Bun + Hono + Drizzle, migrations, bearer auth, health endpoint) | 1–2h | Hello-world `/api/health` returns 200 with valid token |
| M3 | Backend core endpoints (transactions, envelopes, log/parse, deposits, allocation/propose, allocation/accept) + rules engine + Anthropic integration | 2–3h | Allocation flow works via curl |
| M4 | Frontend scaffold + auth-token flow + tab nav | 30 min | Empty SPA at the domain, can hit backend with token |
| M5 | Log view (tile-tap, free-text, confirmation chip, tile-picker fallback, tx_date setter) | 2–3h | Daily logging works on phone |
| M6 | Home view (envelope grid reading API, overdraw red, goal glow placeholder) | 1–2h | Home shows real balances |
| M7 | Allocation flow (retainer/bonus toggle, full show + light show, inline edit, accept) | 3–4h | End-to-end retainer allocation works |
| M8 | Projection engine + scheduled job + projection cache | 2–3h | `/api/projection` returns real numbers; cache refreshed every 4h |
| M9 | Telegram coach (connect flow + dispatcher + six message types + LLM narration + fallbacks) | 2–3h | At least one alert fires correctly in test |
| M10 | Review surface (monthly generator job + Review tab + charts) | 2–3h | Review for current cycle renders |
| M11 | Money-move helper (overlay chip + transfer endpoint + cold-start guard) | 1–2h | Negative-envelope helper appears and works |
| M12 | Animations LAST (fill bar, goal glow timing, retainer money-tokens, bonus light show, reduced motion gate) | 2–3h | All animations meet spec |
| M13 | Backup automation (nightly cron + verify) + Settings (export/import/backup chip + coach kill switch) | 1–2h | Backup happens nightly, restore tested |

**Total:** ~20–30 human-solo hours, ~10–15 with Claude Code. Two weekends, not one.

Each milestone has a single deliverable that's testable before moving on. Milestones are independent enough that M5 and M6 can swap order if you want to see Home first.

---

## 15. Open Decisions

These need Mitch's call before/during build. Numbered for cross-reference.

| # | Decision | Default if not specified | Decision deadline |
|---|---|---|---|
| OD-1 | Hosting platform: managed (Render/Railway/Fly) vs unmanaged VM (Hetzner/DO) | Hetzner CX11 + Caddy | Before M1 |
| OD-2 | Domain: subdomain of existing vs fresh | Subdomain of existing | Before M1 |
| OD-3 | Telegram bot name + handle | Pick at @BotFather time | Before M9 |
| OD-4 | `config.yml` contents: real envelopes, recurring, tax rate, goals, retainer amount, cycle start day | Cannot proceed without | Before M0 |
| OD-5 | Service worker offline strategy | Cache static assets only (fail loudly when offline) | Before M4 |
| OD-6 | Backup destination: Tailscale-shared folder vs S3 | Tailscale-shared folder | Before M13 |
| OD-7 | Coach dispatch time of day | 20:00 local | Before M9 |
| OD-8 | `retainer_cycle_start` day of month | 1st (assume retainer lands prior day) | Before M0 |
| OD-9 | `cycle_avg_window_cycles` (projection averaging window) | 3 | Before M8 |

---

## 16. Risks & Mitigations

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-1 | Daily logging discipline lapses by week 3 | Medium | Project dies | Sub-2s tile-tap UX (FR-1.2); nothing-logged coach nudge (FR-4.6); tx_date backdating (FR-1.6) |
| R-2 | LLM costs spike unexpectedly | Low | Bill shock | Monthly spend cap in Anthropic console; Haiku (cheap); 10s timeouts; per-call max_tokens caps |
| R-3 | Coach becomes annoying | Medium | Coach gets disabled, signal lost | ≤1 msg/day cap (FR-4.7); priority dedup; kill switch in Settings; positive-ping cadence ≤ weekly |
| R-4 | LLM hallucinates numbers in review | Medium | Lost trust in review | Deterministic stats are source of truth; LLM narrates only (FR-5.4); post-gen lint flags numbers not in stats |
| R-5 | Allocation accept partially commits | Low | Corrupted ledger | All accept logic in one Drizzle transaction (FR-2.7); integration test simulates crash |
| R-6 | SQLite file corruption | Low | Data loss | WAL mode; nightly snapshot; restore test weekly |
| R-7 | Backup destination compromised | Low | Financial data leak | Encryption at rest required (Tailscale + S3 SSE); never commit backup files; `.gitignore` enforces |
| R-8 | Self-hosted backend uptime fails | Medium | Logging blocked | systemd auto-restart; health endpoint; alarm if uptime drops below 99%/cycle |
| R-9 | iOS PWA quirks (storage clearing, install limits) | Medium | App "disappears" from home screen | Real PWA over HTTPS minimizes; manual reinstall accepted; data is server-side so no loss |
| R-10 | Projection engine wrong during cold-start | High | Bad coach signals early | Cold-start window with conservative defaults (FR-6.3); coach silenced for cold envelopes until learning |

---

## 17. Done Criteria

v1 is done when **all of the following** hold true for **two consecutive retainer cycles** (~8 weeks):

1. **Daily ritual stuck.** Every transaction logged on or within 48h of `tx_date`. Metric M1 ≥ 85%.
2. **Retainer show landed.** Both retainer allocations went through the full show; Mitch tapped Accept; balances reflect.
3. **At least one bonus.** A non-retainer deposit was allocated via the light show.
4. **Coach earned its keep.** At least one Telegram alert (any kind) prompted Mitch to act before payday.
5. **Review felt real.** First completed monthly review read like it knew him; he rated ≥ 4/5 mentally.
6. **No spreadsheet fallback.** Zero events of opening a spreadsheet to track money instead of the app.
7. **Animation made him smile.** Twice during the cycle, the payday animation hit.
8. **Backend stayed up.** Uptime ≥ 99% across the cycle.
9. **Backups happened.** Nightly snapshots ran without missing > 1 day; restore-test on week 4 succeeded.
10. **No LLM-down dead-end.** All LLM failures (which will happen) gracefully fell back; no feature was blocked end-to-end.

If any fail by cycle 3, v1 has failed in the way the metric describes. The failure mode itself is the brief for v1.1.

---

## 18. Glossary

- **Retainer** — Mitch's predictable ~$6,000 monthly income deposit. The canonical paycheck. Funds one full cycle.
- **Bonus** — Any non-retainer deposit. Pure upside. Defaults to nearest unfunded goal.
- **Cycle** — One retainer-to-retainer period, roughly one calendar month, anchored to `retainer_cycle_start`.
- **Envelope** — A virtual bucket holding allocated money for a category, goal, bill, or tax accrual.
- **Slack** — `current_balance - projected_remaining_spend` for an envelope. Positive = budget to spare; negative = projected overspend.
- **Runway** — Aggregate slack across non-goal envelopes. Days the budget covers at current pace before the retainer arrives.
- **Allocation** — The act of splitting a deposit across envelopes per the allocation rule + LLM proposal.
- **Show** — The retainer allocation animation (3–5s money-token flow). The main monthly event.
- **Light show** — The bonus allocation animation (~1.5s single token). Quieter ceremony.
- **Coach** — The outbound-only Telegram bot. Speaks ≤ 1×/day.
- **Projection engine** — Backend job that computes per-envelope projected spend and slack from history.
- **Cold-start window** — Period when an envelope lacks enough history for confident projection; conservative defaults used.
- **Discretionary** — The elastic envelope. All inline edits in allocation flow balance against it.
- **Advisor pattern** — LLM proposes, user accepts. Nothing commits without explicit user action.
- **n=1** — Built for one user. No multi-user, no auth flow, no sharing.

---

## 19. Source Documents

- **[USER-STORIES.md](USER-STORIES.md)** — Original workshop output. Source of product truth for user stories. PRD acceptance criteria derived from these.
- **[DESIGN.md](DESIGN.md)** — Technical contract. Source for architecture, data model, API surface, build order, hosting story.
- **[FEATURES.md](FEATURES.md)** — User-journey walkthrough in three arcs. Source for state tables, edge cases, animation specifics.
- **[README.md](README.md)** — Repository front door.

Visual references for design system (colors, typography, spacing) live in [`reference/`](reference/) — to be populated by Mitch.

---

*End of PRD v1.0.*
