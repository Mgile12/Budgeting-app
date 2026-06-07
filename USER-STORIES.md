# User Stories — Envelope Budget App

Companion to DESIGN.md and FEATURES.md. This is the workshop output: the product as it stands after the design session, expressed as user stories for the PRD.

Audience of one. The user is **Mitch** throughout, so stories drop the "As a user" boilerplate and name him directly.

## What changed in this session (read first)

The workshop moved the product meaningfully off the original "weekend wedge":

1. **One real budget cycle: retainer-to-retainer.** The fixed ~$6k end-of-month retainer is *the* paycheck — it funds the entire month (tax, all bills, all subs, goal contributions, discretionary). It is sized to cover everything until the next retainer lands. Every other deposit is **bonus / upside**.
2. **Two income flows, two ceremonies.** Retainer = the full payday show + obligation allocation. Bonus = a light show that defaults the money toward the nearest-deadline goal.
3. **Architecture is now Approach B.** Self-hosted on Mitch's own domain → HTTPS, an always-on backend, a shared datastore, the Anthropic key server-side, and a place for scheduled jobs and the Telegram bot to run.
4. **Telegram is an outbound coach only.** It never accepts input. It speaks when something is worth acting on: runway warnings, goal-slip warnings, occasional on-pace reassurance, and "review ready." Logging stays exclusively in the web app.
5. **A projection engine is the new backbone.** Per-envelope spend cadence → projected remaining spend this cycle → genuine slack and runway. It powers the coach alerts, the money-move helper, and goal pace.
6. **A stand-alone monthly review.** Narrative-first verdict from the advisor LLM (behavioural patterns + goal pace), grounded in deterministic stats. Not a chart dashboard.

Invariant that survives intact: **the LLM is an advisor, not an agent.** Every story below that involves the LLM proposes; nothing commits without an explicit Accept.

---

## Epic 0 — Foundation

- **0.1** As Mitch, I want the app self-hosted on my own domain over HTTPS, so that I get a real PWA, an always-on backend, and no Tailscale/webclip limitations. `[v1]`
- **0.2** As Mitch, I want the Anthropic API key held server-side in the backend, so that the key is never shipped in the browser bundle and I don't depend on network-level gating to keep it safe. `[v1]`
- **0.3** As Mitch, I want a shared datastore the web app and backend jobs both read and write, so that the Telegram coach and the web app see the same truth. `[v1]`
- **0.4** As Mitch, I want my envelopes, bills, subs, tax rate, and goals defined in `config.yml` and seeded once on first boot, so that I do the real budgeting thinking before any code runs and never accidentally re-seed over live data. `[v1]`
- **0.5** As Mitch, I want all money stored as integer cents with one formatter at the render boundary, so that I never hit floating-point drift. `[v1]`
- **0.6** As Mitch, I want every balance change to run as an atomic transaction, so that a crash mid-write never leaves a half-applied state. `[v1]`

## Epic 1 — The Calm (daily logging)

- **1.1** As Mitch, I want the app to open to the Log tab every time, so that the thing I do 5–15× a day is zero taps from the start. `[v1]`
- **1.2** As Mitch, I want a top row of pinned quick-tap envelope tiles, so that logging a coffee is tap-number-save in under two seconds. `[v1]`
- **1.3** As Mitch, I want a free-text bar that an LLM parses into `{intent, amount, envelope, description, confidence}`, so that I can type "amex 240 dinner" without picking an envelope by hand. `[v1]`
- **1.4** As Mitch, I want a confirmation chip on a confident parse with a one-tap "change" override, so that I can correct a wrong envelope guess before it saves. `[v1]`
- **1.5** As Mitch, I want a low-confidence or failed parse to fall back to a tile-picker with the amount pre-filled, so that logging never dead-ends when the LLM is unsure or down. `[v1]`
- **1.6** As Mitch, I want to set the date/time on a logged transaction, defaulting to now, so that the coffee I forgot until tonight or Tuesday's lunch I enter Thursday land on the correct day. `[v1]`
- **1.7** As Mitch, I want daily logging to be visually calm — a quiet fill-bar tick, no celebration — so that the payday moment stands out by contrast. `[v1]`
- **1.8** As Mitch, I want to glance at an envelope's current balance before a *large* purchase, so that I can decide whether I can afford it (small purchases need no check). `[v1]`

## Epic 2 — The Main Event (retainer allocation)

- **2.1** As Mitch, I want to log the ~$6k retainer and have it recognised as the monthly paycheck, so that it triggers the full allocation, not a bonus drop. `[v1]` *(detection rule: see Open Decisions)*
- **2.2** As Mitch, I want the advisor to propose an allocation of the full retainer across tax → bills → subs → goal contributions → discretionary remainder, so that the one deposit funds my entire month. `[v1]`
- **2.3** As Mitch, I want a deterministic rules engine to ground every proposal, with the LLM adding the narrative and one-off tweaks on top, so that the allocation still works when the LLM is down. `[v1]`
- **2.4** As Mitch, I want each proposed row to carry a plain-language reason, so that I understand *why* money went where before I accept. `[v1]`
- **2.5** As Mitch, I want to edit any row inline before accepting, with Discretionary as the single elastic envelope that absorbs the difference, so that rebalancing the proposal has a simple mental model. `[v1]`
- **2.6** As Mitch, I want the full money-token show (3–5s, silent) when the retainer is allocated, so that the main event of the month feels like an event. `[v1]`
- **2.7** As Mitch, I want nothing written to disk until I tap Accept, and then all rows committed atomically under one paycheck, so that I never get a partial allocation. `[v1]`
- **2.8** As Mitch, I want a rare, *loud* alarm if the retainer genuinely can't cover the month's obligations, so that a real shortfall is unmissable (this is an exception, not routine UI, because the retainer is sized to cover everything). `[v1]`
- **2.9** As Mitch, I want all animations to respect reduced-motion (fade-in instead of fly-in, same timing), so that the show degrades gracefully. `[v1]`

## Epic 3 — Bonus Drops (irregular income)

- **3.1** As Mitch, I want any non-retainer deposit treated as bonus, so that it skips obligation math (life is already funded by the retainer). `[v1]`
- **3.2** As Mitch, I want bonus money to default toward the nearest-deadline unfunded goal, with a light show rather than the full ceremony, so that extra income accelerates what's closest without ceremony fatigue. `[v1]`
- **3.3** As Mitch, I want to override the bonus default to hold it in Discretionary or send it elsewhere, so that I keep final say on upside. `[v1]`

## Epic 4 — The Coach (Telegram, outbound only)

- **4.1** As Mitch, I want a runway warning when total money-left-vs-days-to-retainer says I'll run out before the next $6k, fired once when I cross into danger, so that I learn I'm overspending while I can still react. `[v1]`
- **4.2** As Mitch, I want a per-envelope warning when a single envelope is on pace to run dry before the retainer, so that I catch a specific overspend (e.g. Eating Out) early. `[v1]`
- **4.3** As Mitch, I want a goal-slip warning when a goal falls behind its required pace, fired once on the slip, so that I know to add to it. `[v1]`
- **4.4** As Mitch, I want occasional positive "on pace, nothing to do" reassurance, so that silence isn't the only signal and I trust the coach is watching. `[v1]`
- **4.5** As Mitch, I want a "your month-end review is ready" ping when the review generates, so that I'm pulled to read it rather than having to remember. `[v1]`
- **4.6** As Mitch, I want a gentle "nothing logged today" nudge, so that the habit survives past week three (my #1 failure mode). `[v1]` *(cadence: see Open Decisions)*
- **4.7** As Mitch, I want all coach messages batched to at most one per day and never accept input, so that the bot is a calm tap on the shoulder, not noise. `[v1]`

## Epic 5 — The Review (monthly, stand-alone)

- **5.1** As Mitch, I want a stand-alone Review surface I can open any time, so that I can read it without being mid-allocation. `[v1]`
- **5.2** As Mitch, I want the review led by a plain-language verdict — what I did wrong, what to improve, what to keep doing — so that it moves the needle instead of being a chart wall. `[v1]`
- **5.3** As Mitch, I want the verdict to cover behavioural patterns (e.g. "you overspend in the days after you skip logging") and goal pace (e.g. "Japan Trip is 3 weeks behind; add $X/month to recover"), so that it's diagnostic *and* prescriptive. `[v1]`
- **5.4** As Mitch, I want the review's numbers computed deterministically and only *narrated* by the LLM, so that it can never invent a figure. `[v1]`
- **5.5** As Mitch, I want at most one or two small charts *supporting* the verdict, not leading it, so that visuals clarify without becoming the noise I want to avoid. `[v1]`

## Epic 6 — Projection Engine (enabling capability)

- **6.1** As the system, I need to compute each envelope's spend cadence and a projected remaining spend for the current cycle, so that "underspent" can be distinguished from "not spent *yet*." `[v1]`
- **6.2** As the system, I need to derive genuine slack (`balance − projected remaining spend`) per envelope and an aggregate runway (money-left vs days-to-retainer), so that coach alerts and money-move suggestions are accurate. `[v1]`
- **6.3** As the system, I need a defined cold-start window where history is insufficient, so that downstream features behave conservatively until patterns are learned. `[v1]` *(threshold: see Open Decisions)*

## Epic 7 — Money-Move Helper (proactive rebalance)

- **7.1** As Mitch, I want a quiet, dismissible suggestion to surface the moment an envelope goes negative, so that I'm prompted to fix it without a blocking modal. `[v1]`
- **7.2** As Mitch, I want the helper to propose where to pull from and let me accept or decline, so that money only moves on my say-so (advisor, not agent). `[v1]`
- **7.3** As Mitch, I want learned-state suggestions to pull from envelopes with genuine slack (per the projection), never from one I'm about to spend from, so that a rebalance doesn't create the next shortfall. `[v1]`
- **7.4** As Mitch, I want cold-start suggestions to pull only from Discretionary and always ask permission, so that the helper is safe before it has learned my patterns. `[v1]`

## Epic 8 — The Edges (trust layer)

- **8.1** As Mitch, I want to edit a logged transaction's amount, envelope, and date, so that I can fix the $40-meant-$4 mistake cleanly. `[v1]`
- **8.2** As Mitch, I want an overdrawn envelope shown in red with the amount, no blocking, so that overdraft is honest signal I can keep logging against. `[v1]`
- **8.3** As Mitch, I want a goal tile to glow once when it first crosses its target (and re-arm if it drops back), so that funding a goal gets a quiet celebration without repeat noise. `[v1]`
- **8.4** As Mitch, I want every LLM-dependent path to have a deterministic fallback and a subtle "last call errored" icon, so that the core workflow never blocks on Anthropic being up. `[v1]`
- **8.5** As Mitch, I want manual JSON export/import in Settings with a "last backup N days ago" chip, so that I can recover from a wipe. `[v1]`
- **8.6** As Mitch, I want a guard against two simultaneous editing sessions, so that I don't double-log the same coffee from phone and laptop. `[v1]`

---

## Deferred (v1.1 / v2)

Carried from the design docs and this session, intentionally out of v1:

- Override-correction logging (`corrections` table for prompt tuning) `[v1.1]`
- Large-deposit sanity check (confirm chip > $10k) `[v1.1]`
- Delete transaction (v1: edit to $0.01) `[v1.1]`
- Most-frequent auto-rotation of top tiles `[v1.1]`
- LLM clarifying-question re-prompt loop `[v1.1]`
- LLM confidence-calibration settings toggle `[v1.1]`
- Confetti on goal completion `[v1.1]`
- Cadence-aware "you forgot to log a deposit" detection `[v2]`
- Lightweight weekly spending pulse (separate from the monthly review) `[v2]`

---

## Open decisions for the PRD

These were raised but not fully nailed; the PRD should resolve them:

1. **Retainer vs bonus detection.** How does the app know a deposit is *the retainer*? Options: amount heuristic (~$6k), an explicit "this is my retainer" toggle at log time, or end-of-month timing. Recommend an explicit toggle (cheapest, unambiguous), defaulted on for an ~$6k end-of-month deposit.
2. **"Nothing logged today" cadence.** Daily, weekdays only, or after N quiet days? Recommend: fire only after a full day with zero logs, never more than one nudge before a log resets it.
3. **Positive-ping frequency.** "Occasional" needs a number so it stays meaningful. Recommend: at most weekly, and only when there's genuinely nothing to warn about.
4. **Projection cold-start threshold.** How much history before the engine "knows" a pattern? Recommend: per-envelope, switch from conservative to learned once an envelope has ~3–4 cycles or a stable cadence; until then it's cold-start for that envelope specifically.
5. **Runway danger threshold.** What margin counts as "into danger" for 4.1/4.2? Recommend: projected to run dry with ≥2 days of cycle remaining, so the warning is actionable rather than too-late.
6. **Goal definition.** Which real goals, targets, and deadlines go in `config.yml` — Mitch's to fill before build (this is "the assignment" from DESIGN.md).

---

## Screen map

Screens needed to deliver the stories above, split by behaviour: primary views (tab-level), modals & overlays, system screens, and the external Telegram channel. The purpose column makes each row self-explanatory.

### Primary views

| Screen | Purpose | User stories |
|---|---|---|
| **Log** | Default landing; daily logging hot path (tile row + free-text bar + confirmation chip) | 1.1, 1.2, 1.3, 1.4, 8.4* |
| **Home** | Envelope grid; glance-before-buying, overdraw, goal glow, calm fill-bar | 1.7, 1.8, 8.2, 8.3 |
| **Deposit Allocation** (the show) | Proposal + edit + accept for a deposit. *Retainer mode* = full show; *Bonus mode* = light show | 2.1–2.9, 3.1–3.3 |
| **Review** | Stand-alone monthly narrative verdict + supporting charts | 5.1–5.5 |
| **Settings** | Export/import, backup chip, config overview, Telegram connection | 8.5 |

### Modals & overlays

| Screen | Purpose | User stories |
|---|---|---|
| **Number Pad Modal** | Amount (and date) entry after a tile tap | 1.2, 1.6 |
| **Tile-Picker Modal** | Universal fallback: low confidence, error, or paycheck-no-amount | 1.5 |
| **Envelope Detail** | Expanded tile: recent transactions list + entry point to edit | 8.1, 8.2 |
| **Edit Transaction Modal** | Change amount / envelope / date of a logged transaction | 8.1, 1.6 |
| **Money-Move Suggestion** | Quiet, dismissible rebalance prompt with source options | 7.1–7.4 |
| **Import / Restore Flow** | Pick file → validate → wipe → restore snapshot | 8.5 |

### System & utility screens

| Screen | Purpose | User stories |
|---|---|---|
| **First-Boot / Empty State** | Day-0 instant render, "First paycheck? Tap + to allocate" hint | 0.4 |
| **Boot Error Screen** | Malformed/orphaned config — show parser line/col error | 0.4 |
| **Two-Tab Guard Screen** | "Already open in another tab. Refresh to take over." | 8.6 |

### External channel (not an app screen)

| Surface | Purpose | User stories |
|---|---|---|
| **Telegram** | Outbound coach: runway/goal warnings, on-pace pings, "review ready" | 4.1–4.7 |

\* **Cross-cutting states**, not owned by one screen: 8.4 (LLM-error icon + fallbacks) lives mainly on Log but also Allocation; 2.9 (reduced-motion) governs the fill bar, the goal glow, and the Allocation show.

### Notes

- **Navigation.** Four tabs: **Log, Home, Review, Settings.** Allocation is not a tab — it's entered from Log on deposit detection or a "+ deposit" button, and returns Home on accept. Money-Move is an overlay on Home.
- **Telegram connection has no story yet.** Setup (link account, store chat ID) is parked in Settings above; add a story such as "As Mitch, I want to connect my Telegram once in Settings, so the coach can reach me." See Open Decisions.
- **No-screen stories.** Epic 0 infrastructure (0.1, 0.2, 0.3, 0.5, 0.6) and the projection engine (6.1–6.3) are backend/data with no UI of their own; they belong in the PRD's technical section.
