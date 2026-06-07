# Features — by user journey

Companion to [DESIGN.md](DESIGN.md) and [USER-STORIES.md](USER-STORIES.md). DESIGN.md is the technical contract; USER-STORIES.md is the product spec; this is the journey.

Three arcs, ordered the way Mitch meets the product:

- **Arc 1 — The Calm.** Day 0 setup and the daily logging ritual. If this arc fails, nothing else gets the chance to land.
- **Arc 2 — The Show.** The retainer (full show) and the bonus (light show). The reason the app earns being opened.
- **Arc 3 — The Trust Layer.** The coach that whispers, the review that tells the truth, the projection engine under the hood, the money-move helper, plus the recovery moments (edit, overdraw, goal glow, LLM down, backup, two-tab).

---

## Arc 1 — The Calm

### 1.1 First boot (Day 0)

You've provisioned the VM, pointed the domain, the backend is running behind Caddy with HTTPS, and you visit `https://budget.yourdomain.example` for the first time on phone.

**The sequence:**

1. Browser loads the static SPA bundle from Caddy.
2. SPA boots, looks for `Authorization: Bearer <token>` in localStorage. None → renders the **one-time auth screen**: a single password field "Paste your auth token to continue."
3. You paste the token (set in the systemd env file when you provisioned). Stored in localStorage. SPA reloads.
4. Frontend hits `GET /api/envelopes`. Backend has run its own first-boot dance: on backend startup it queries `SELECT COUNT(*) FROM envelopes`; count=0 → backend reads bundled `config.yml`, runs `loadFromConfig()` inside a Drizzle transaction (envelopes, recurring, app_config, coach_state placeholder). Atomic — abort on error leaves count=0 and next start retries.
5. Backend returns envelopes with $0 balances + projection placeholder (confidence: `cold`).
6. Frontend renders Home — envelope grid, every tile at $0.00. **Instant render. No fade-in animation.** Day 0 feels like opening a fresh notebook.
7. Single sub-grid line below the envelope grid: **"First retainer? Tap + to allocate."** No CTA button, no modal — just the hint.

**Settings → Connect Telegram** is the second Day-0 task. Walks you through `@BotFather` to create a bot, paste the token, send `/start` to it from your account, paste the chat_id. Saves to `coach_state`. Coach is now wired.

**States that actually matter:**

| State | Trigger | Behavior |
|---|---|---|
| Auth missing | localStorage has no token | One-time paste screen |
| Auth invalid | Backend returns 401 on first API call | Clear localStorage, re-prompt |
| Backend down | API call throws / times out | "Backend unreachable" splash with retry. PWA shell still loads. |
| First-boot empty | `envelopes` count = 0 on backend | Backend seeds inside a transaction, returns empty grid |
| Partial seed | Seed transaction threw mid-write | Aborts atomically, count stays 0, next request retries |
| Malformed config.yml | js-yaml throws on backend boot | Backend logs the line/col error and refuses to start. You ssh in, fix, restart. |
| Missing envelope reference | `tax_envelope: "X"` but no envelope `X` in `envelopes:` | Backend boot throws with clear error pointing at orphan name |
| Past-deadline goal | `deadline ≤ today` | Seeds fine. Home tile gets "review goal" badge. Allocation advisor skips it. |

**Edge case: Telegram not connected.** Coach silently no-ops. No errors. The system logs internally but doesn't bother you. Connect it whenever you're ready; coach kicks in next dispatch window.

**Features touched:** 0.1 (HTTPS hosting), 0.2 (server-side key), 0.3 (shared datastore), 0.4 (YAML seed), 0.5 (integer cents), 0.6 (atomic writes), 4.x (Telegram connect setup).

---

### 1.2 Logging a coffee at 8:14am

You're at a cafe. $4 coffee. Pull out your phone, open the bookmark (now installable as a real PWA because HTTPS — story 0.1). **App opens to Log tab every time** (story 1.1, the hot path).

**The happy path (tap, sub-2-second):**

```
1. Log tab open. Top row: [Coffee] [Groceries] [Gas] [Eating Out] [Misc]
2. Tap [Coffee] ─────────────────────────────────► t=0s
3. Number pad modal slides up. Amount input focused.
   Date field below shows "Today, 8:14am" — tap to change.
4. Type "4" ──────────────────────────────────────► t=~1.2s
5. Tap Save ─────────────────────────────────────► t=~1.5s
6. Modal dismisses.
7. Toast: "$4.00 → Coffee"
8. Coffee tile fill bar animates the deduction (~300ms)
```

**Behind the scenes:** frontend sends `POST /api/transactions` with `{amount_cents: 400, envelope_id: <coffee>, tx_date: "2026-06-08", source: "tile_tap"}`. Backend wraps the insert + balance update in a Drizzle transaction. Returns the updated envelope balance. Frontend reflects.

**The backdated path (story 1.6):** It's Thursday and you remember Tuesday's lunch.

```
1. Tap [Eating Out]. Number pad modal.
2. Tap the date field — date picker opens. Select Tuesday.
3. Type "15". Save.
4. Backend stores tx_date as Tuesday, created_at as Thursday.
5. Home tile updates as if it landed Tuesday — the projection engine and the coach see it on Tuesday.
```

**The free-text path:** tap the free-text bar below the tiles and type `12 amex dinner`.

```
1. Type "12 amex dinner" + enter ───────────────► t=0
2. "..." chip appears
3. Frontend POSTs /api/log/parse with the text.
4. Backend calls Claude Haiku (max_tokens: 512, 10s timeout).
   Zod-validates: {intent, amount_cents, envelope, description, confidence}.
5. Returns: { intent: "transaction", amount_cents: 1200,
              envelope: "Eating Out", description: "amex dinner",
              confidence: 0.85 }                ───► t=~500ms
6. Frontend shows confirmation chip:
   "$12.00 → Eating Out [change]"
7. Auto-save after 2.5s, OR immediate save on tap-elsewhere, OR
   tap [change] to open tile-picker for override.
```

**The low-confidence fallback (universal safety net):** parse confidence < 0.7 → backend returns the parse with `should_fallback: true` → frontend opens tile-picker modal with amount pre-filled. Same path on network error or malformed response. The tile-picker is the catch-all.

**The deposit detection path:** `got paid 6000` → parser returns `intent: "deposit"` → frontend doesn't write a transaction; it stores the amount and routes to the Allocation flow (Arc 2). The retainer-vs-bonus chip there defaults based on the heuristic.

**States that actually matter:**

| State | Trigger | Behavior |
|---|---|---|
| Loading | LLM in flight | "..." chip. 10s timeout. |
| Empty input | Amount 0 / unparseable | Save disabled, no error text |
| Network down | API throws | Catch → tile-picker with amount pre-filled (regex parse) or empty |
| Rate-limited | Two parses within 500ms | Frontend debounces; ~2 calls/sec ceiling |
| Save in flight | Save tapped | Spinner; modal can't re-open. Idempotent. |
| LLM errored on last call | Backend returns parse with `should_fallback: true` or 5xx | Subtle 8px corner icon on the input bar. Auto-clears on next success. |

**Edge cases:**

- `12.5` / `$12.50` / `twelve dollars` all parse to 1200 cents on free-text. Tile-tap uses native number pad, digits only.
- Override correction (tap the chip) reassigns `envelope_id` but `source` stays `text_parse`. Useful signal data; v1.1 may log corrections to a dedicated table.
- Backdating a transaction by N days **updates projections** within the affected envelope's cycle — the coach sees it the same way it would have if you'd logged it on time.

**Features touched:** 1.1 (default landing), 1.2 (tile-tap), 1.3 (free-text parse), 1.4 (confirmation chip), 1.5 (tile-picker fallback), 1.6 (tx_date), 1.7 (calm fill bar), 8.4 (LLM-down icon).

---

## Arc 2 — The Show

### 2.1 The retainer lands (the full show)

It's the 28th of the month. Your retainer hits your bank. You open the app, Log tab, tap the free-text bar, type `got paid 6000`.

**The detection:** parser returns `intent: "deposit", amount_cents: 600000, confidence: 0.97`. Frontend stores `{amount_cents: 600000, tx_date: today}` in component state and navigates to the Allocation view. **Nothing has been written yet.** Staging only.

**The retainer/bonus chip.** Allocation view top shows: **"Retainer or Bonus?"** with **Retainer** highlighted by default because:
- Amount within ±10% of `app_config.retainer_amount_cents` ($5,400–$6,600), AND
- tx_date within `retainer_cycle_start ± 5 days` (story 2.1 + open decision #1)

One tap to toggle. You leave it on Retainer.

### 2.2 Loading the proposal

Paycheck node "$6,000" renders at top with a subtle pulse. Loading skeleton in the middle. Envelope tiles at bottom in their resting state.

```
Frontend → POST /api/allocation/propose
  body: { deposit_cents: 600000, kind: "retainer",
          tx_date: "2026-06-28" }

Backend:
  1. Loads app_config, envelopes, recurring, projection snapshot from DB.
  2. Runs runRules(deposit_cents, app_config, envelopes, recurring,
                   projection_snapshot, mode: "retainer").
     - tax_percentage              → $1,500 → Tax Savings
     - cover_bills_next_30_days    → $2,400 → Rent + others
     - subscriptions               → $80 → Subscriptions
     - goal_contributions          → $400 → Japan Trip + $200 → Emergency
     - discretionary_remainder     → $1,420 → Discretionary
  3. Calls Claude Haiku with the baseline + projection state.
     LLM may tweak rows (e.g., "Coffee is overdrawn -$15 this cycle,
     propose +$15 from Eating Out").
  4. Zod-validates response. Sum-equals-deposit check.
     Fallback to baseline on any failure.
  5. Returns { proposal: [...], narrative: "...", mode: "retainer" }.
```

The rules engine is the floor. The LLM is the friendly narrator + optional adjuster. If the LLM goes haywire, validate-and-fallback catches it. **The user never sees broken state.**

**Under-paycheck behavior (the rare loud alarm).** If `tax + bills + subs + goal contributions` exceeds `deposit_cents` (story 2.8), the engine truncates the last unmet rules. Bills and tax honored first; goals get $0; discretionary stays $0. A yellow chip at the top: **"Bills + tax covered. Goals deferred this cycle. Discretionary: $0."** AND a `runway`-kind Telegram message fires same day. This should be **rare** — the retainer is sized to cover everything.

### 2.3 The show itself

Proposal loaded. Animation kicks off.

**Choreography (~4s for a typical 8-row retainer):**

```
T+0.0s   Paycheck node "$6,000" at top. Envelope tiles at bottom.
T+0.2s   First money token launches.
         - Sized: text scales by log(amount). $80 vs $1,500 visually distinguishable.
         - Colored: matches destination envelope.
         - Path: ease-out arc from paycheck node to destination tile, ~600ms.
T+0.5s   Token arrives. Tile fill bar leaps. Token dissolves.
         Reason line fades in below the row in middle column.
T+0.8s   Next token launches. (300ms stagger.)
...
T+3.5s   Last token: Discretionary remainder.
T+4.0s   Accept + Edit buttons fade in at bottom.
```

**Driven entirely by the in-memory proposal object.** No live DB queries while animating. Nothing committed.

**Silent by default.** No audio. **Reduced-motion gate.** `@media (prefers-reduced-motion: reduce)` → tokens don't fly; rows fade in at the same 300ms stagger. Same information, same rhythm, no movement.

### 2.4 Inline edit (Discretionary as elastic)

Maybe you want to push $200 from Discretionary to Japan Trip this cycle.

```
1. Tap Japan Trip row's amount field. Number pad slides up.
2. Change 400 → 600. Save.
3. Discretionary auto-decreases: 1420 → 1220. (Discretionary is THE elastic envelope.)
4. Affected rows pulse briefly. Total stays $6,000.
```

**Rules of the edit UX:**

- Discretionary is the **single elastic envelope.** Every other edit balances against Discretionary. Simpler mental model than cascading.
- Sum invariant runs on every keystroke. Exceeding `deposit_cents` → error chip "Reduce X by $Y." Save disabled until valid.
- Editing Tax is allowed; small warning chip "this overrides your tax rate — remember at year-end."
- Negative amounts blocked.

### 2.5 Accept and commit

```
Frontend → POST /api/allocation/accept
  body: { proposal: [...], deposit_cents: 600000,
          kind: "retainer", tx_date: "2026-06-28" }

Backend:
  await db.transaction(async tx => {
    const deposit = await tx.insert(deposits).values({
      ts: nowISO(),
      gross_amount: depositCents,
      kind: "retainer",
      allocation_snapshot: JSON.stringify(proposal)
    }).returning();

    for (const row of proposal) {
      await tx.insert(transactions).values({
        tx_date: txDate,
        created_at: nowISO(),
        amount: row.amount_cents,
        envelope_id: row.envelope_id,
        description: row.reason,
        source: "allocation",
        deposit_id: deposit.id
      });
      // Update envelope balance + check for goal crossing
      const env = await tx.select().from(envelopes).where(eq(id, row.envelope_id));
      const wasUnder = env.current_balance < (env.target_balance ?? Infinity);
      const newBalance = env.current_balance + row.amount_cents;
      const isNowOver = newBalance >= (env.target_balance ?? Infinity);
      await tx.update(envelopes).set({
        current_balance: newBalance,
        celebrated_at: (wasUnder && isNowOver && env.celebrated_at == null)
          ? nowISO()
          : env.celebrated_at
      }).where(eq(id, row.envelope_id));
    }
  });

  // After commit: kick projection refresh (non-blocking)
  void refreshProjections();
```

**Either every row commits or none do.** No partial allocation.

**On accept:**

1. Accept button → checkmark (200ms).
2. Fade transition back to Home.
3. Home re-renders with new balances.
4. Goal envelopes that crossed during this allocation glow now — **stagger 1.5s in `order_index` order.** Not during the show. The retainer show belonged to the paycheck; goals get their own moment.
5. Backend triggers a projection refresh in the background.

**The LLM-down version of the retainer show.** Same flow. Same animation. Same accept. Reasons read tighter ("Cover bills next 30 days" instead of "$1,200 → Rent: covers next month's bill due 2026-07-01"). Subtle 8px corner icon on the input bar surfaces the error. You wouldn't notice unless you looked.

**Features touched:** 2.1–2.9 (retainer allocation flow), 8.3 (goal glow after accept).

---

### 2.6 A bonus drops (the light show)

It's the 14th. Side gig pays you $400. You log `got paid 400`.

**Detection:** parser returns `intent: "deposit"`. Allocation view loads. Top chip shows **"Retainer or Bonus?"** with **Bonus** highlighted by default (amount not within ±10% of retainer, date not near cycle start). One tap to switch if you genuinely got a $400 retainer (you won't).

**The proposal:**

```
Backend runRules(400_00, kind: "bonus", ...):
  - Find the nearest-deadline UNFUNDED goal envelope:
    e = envelopes
        .filter(kind === "goal" &&
                current_balance < target_balance &&
                deadline > today)
        .sort by (deadline ASC, target - current DESC)
        .first()
  - propose: [{ envelope_id: e.id, amount_cents: 40000,
                reason: "$400 → Japan Trip (nearest unfunded goal)" }]
  - Optional LLM narration on top.

Returns { proposal: [...], narrative: "...", mode: "bonus" }
```

**The light show:**

```
T+0.0s   Paycheck node "$400" at top. Japan Trip tile highlighted at bottom.
T+0.2s   Single money token launches.
T+0.8s   Token arrives. Japan Trip tile fill bar leaps.
         Token dissolves. Reason line fades in.
T+1.5s   Accept + Edit buttons fade in.
```

**One token. ~1.5 seconds. Same easing curve as the retainer show, no audio.** Light by design — bonus deposits don't deserve the same ceremony as the monthly retainer (story 3.2: "extra income accelerates what's closest without ceremony fatigue").

**Override.** Edit inline to send the bonus to Discretionary, split across goals, or anywhere else. Same edit UX as retainer mode.

**Accept** commits one transaction (or N if you split) linked via `deposit_id` of kind `bonus`. No re-allocation of obligations; bonus money is pure upside.

**Edge case: no unfunded goals exist.** Every goal is at target. Bonus defaults to Discretionary. The light show flies the token to Discretionary instead; "no goals to fund — held in Discretionary" reason.

**Features touched:** 3.1 (bonus skips obligations), 3.2 (nearest-deadline default + light show), 3.3 (override).

---

## Arc 3 — The Trust Layer

### 3.1 The coach whispers (Telegram, outbound only)

It's the 22nd. You've been eating out more than usual. The projection engine refreshes every 4h and sees Eating Out is on pace to run dry by the 25th — 3 days before the next retainer.

**What happens (no screen — Telegram):**

At 20:00 local (your configured `dispatch_time_local`), the coach dispatcher job runs:

```
for envelope in envelopes:
  if projection.confidence in (learning, learned)
     and projection.projected_remaining_spend > current_balance
     and days_until_zero(envelope) + 2 < days_to_retainer:
       fire_candidate("envelope_runway", envelope)

# Priority sort: runway > envelope_runway > goal_slip > review_ready > nothing_logged > positive
# Pick highest-priority candidate
# Check last_fire_per_kind to dedupe same-condition repeats
# Fire at most one per day total
```

**The Telegram message:**

> Eating Out is on pace to run dry on the 25th, 3 days before your retainer. You're $42 over the cycle average so far. Pull from Discretionary or slow down — your call.

Plain text. ≤280 chars. LLM-generated narrative grounded in the projection numbers. Deterministic fallback template if the LLM call errored: `"Eating Out projected to run dry 2026-06-25; current $-2.40; avg cycle spend $180; this cycle so far $222. Action recommended."`

**Cadence rules (story 4.7):**

- At most ONE message per day total across all kinds.
- Each `kind` is deduped via `coach_state.last_fire_per_kind` — won't re-fire `envelope_runway` for the same envelope until the condition clears and re-arms.
- The `nothing_logged` nudge waits for a full local-day with zero `transactions` rows; resets on any log.
- The `positive` ping fires at most weekly AND only when no other condition would have fired in 6+ days.

**You can't reply.** The bot is read-only. If you act on the advice, you act in the web app — open it, optionally tap the money-move suggestion that's now sitting on the Eating Out tile (because the envelope went negative), pull from Discretionary, done.

**Features touched:** 4.1 (runway warning), 4.2 (envelope runway), 4.3 (goal slip), 4.4 (positive), 4.5 (review ready), 4.6 (nothing logged), 4.7 (cadence).

---

### 3.2 The projection engine (invisible, foundational)

You never look at this directly. It's the math under the coach, the money-move helper, and the goal-pace number on goal tiles.

**Refresh.** Every 4 hours, a scheduled job runs:

```
for envelope in envelopes:
  txs = transactions where envelope_id and tx_date in current cycle
  avg_cycle_spend = mean(historical_cycle_spend(envelope, N_cycles))
  cycle_fraction_elapsed = days_into_cycle / cycle_length_days
  spend_so_far_this_cycle = sum(txs.amount where amount < 0)

  if confidence == learned:
    projected_remaining = avg_cycle_spend * (1 - cycle_fraction_elapsed)
  elif confidence == cold:
    # Conservative — assume full ratable spend
    projected_remaining = recurring_amount * (1 - cycle_fraction_elapsed)
  elif confidence == learning:
    # 50/50 blend
    projected_remaining = (avg_cycle_spend + recurring_amount) / 2 * (1 - cycle_fraction_elapsed)

  slack = current_balance - projected_remaining
  on_pace = (slack >= 0)

  upsert into projections (envelope_id, computed_at, ...)
```

**Cold-start window** (open decision #4 resolved): an envelope earns `learned` confidence once it has **≥3 completed cycles** of data AND the cycle-to-cycle variance is < 25%. Between 1 and 3 cycles, or above 25% variance, it's `learning` — the 50/50 blend. Until then `cold`.

**Inputs to LLM.** Every LLM call (text-parse excepted) receives the projection snapshot. The advisor sees state, not just balances. The coach speaks from numbers it didn't invent.

**Features touched:** 6.1 (spend cadence + projection), 6.2 (slack + runway), 6.3 (cold-start window).

---

### 3.3 The monthly review

It's the 28th. The retainer just hit, you allocated it. At 09:00 the next morning a scheduled job runs and notices the prior cycle is now complete.

```
Backend job (daily, fires on tx_date == retainer_cycle_start):
  1. Compute deterministic stats for the just-completed cycle:
     - spend per envelope vs avg
     - days with zero logs
     - goals that funded / slipped / stayed flat
     - retainer underspend or overspend
     - over-pace and under-pace envelopes
  2. POST /api/review/narrate (internal) with the stats blob.
     LLM (Claude Haiku, max_tokens: 2048) returns narrative.
  3. Persist in monthly_reviews.
  4. Fire Telegram coach kind: "review_ready".
```

**You get a Telegram nudge.** "Your June review is ready." Tap to open the app (or open it whenever). The **Review tab** shows the current month's review at the top, prior months collapsed below.

**Review surface anatomy:**

```
┌────────────────────────────────────────────┐
│ June 2026 — Review                         │
│                                            │
│ The narrative verdict (paragraph):         │
│ "You did better on bills than the prior   │
│ two cycles. Eating Out is the pattern to  │
│ watch — you overspend by ~$40 in the     │
│ four days after you skip logging. Japan   │
│ Trip is on pace; Emergency Fund slipped   │
│ 2 weeks. Add $35/month to recover by      │
│ year-end. Keep doing daily logging."       │
│                                            │
│ [Small chart: spend by envelope vs avg]   │
│ [Small chart: goal pace over the cycle]   │
│                                            │
│ ▾ Underlying stats                        │
│   - days_with_zero_logs: 3                │
│   - retainer_overspend_cents: -8400       │
│   - envelopes_overspent: ["Eating Out"]   │
│   - goals_funded: []                      │
│   - goals_slipped: ["Emergency Fund"]     │
└────────────────────────────────────────────┘
```

**The verdict is words first. Charts support, they don't lead.** Story 5.5. Story 5.4: the LLM can NEVER invent numbers — it sees the deterministic stats blob and narrates from it. If a number isn't in the stats blob, the narrative can't mention it.

**Edge case: LLM failed when narrating.** Fallback narrative is a simple template ("This cycle: $X spent across N transactions, M envelopes ran over, K goals progressed"). Review still ships. You can tap "regenerate narrative" later if Anthropic comes back up.

**Features touched:** 5.1 (stand-alone surface), 5.2 (narrative-led verdict), 5.3 (behavioral + goal pace), 5.4 (deterministic numbers), 5.5 (small supporting charts), 4.5 (review-ready coach ping).

---

### 3.4 The money-move helper

It's 8:14am. You logged a $4 coffee but you'd been overdrawn on Coffee from yesterday's $80 dinner. Coffee balance: -$4.

**On Home, a quiet chip appears under the Coffee tile.** Not a modal. Not a notification. Dismissible.

> "Coffee is -$4. Move from Discretionary?  [Move]  [Dismiss]"

The helper picked Discretionary because: projection says Discretionary has $400 of slack, more than any other envelope, and confidence on Discretionary is `learned`.

**Two paths:**

- **Move** → frontend POSTs a transfer transaction (one in, one out, both linked). Coffee balance: $0. Discretionary balance: -$4 from current. Animation: fill bars on both tiles update.
- **Dismiss** → chip hides for this envelope until next negative crossing.

**Cold-start guard (story 7.4).** If Coffee or the proposed source envelope has `confidence: cold`, the helper:
- Only proposes pulling from Discretionary (never another learned envelope).
- Phrases the chip as a question, not a one-tap commit: "Pull $4 from Discretionary?" with an explicit Confirm step.

**Why this is safe.** Story 7.3 enforces "never pull from an envelope you're about to spend from." The projection engine knows projected remaining spend per envelope, so the helper actually has signal.

**Features touched:** 7.1 (quiet dismissible suggestion), 7.2 (advisor not agent), 7.3 (pull from genuine slack), 7.4 (cold-start guard).

---

### 3.5 Edit a logged mistake

8:14am. You meant $4 coffee, typed $40. Caught it 30s later.

```
1. Tap Coffee tile on Home. Expands to recent transactions list.
2. Tap the $40 transaction.
3. Edit modal: { amount: 40.00, envelope: Coffee,
                 description: "", tx_date: today }
4. Change amount to 4.00. Save.
5. PATCH /api/transactions/:id
6. Backend wraps in a transaction:
   - Reverse old delta: Coffee balance += 36_00
   - Update transactions[id].amount = -400
   - deposit_id stays null (this wasn't from an allocation)
7. Fill bar animates the $36 difference back up.
```

**Story 8.1 also covers date editing.** Same modal, change tx_date. Both deltas apply to the same envelope.

**Envelope reassignment.** Same modal, change envelope dropdown. Coffee → Eating Out:

```
- Reverse Coffee:    balance += 400
- Apply Eating Out:  balance -= 400
- Update envelope_id, source unchanged
- deposit_id preserved (the snapshot is frozen)
```

**Edit rules (v1):**

- Amount + envelope + date only. No description edit, no source-type change, no delete. (Delete deferred to v1.1; for now, edit to $0.01 to neutralize.)
- Allocation-sourced transactions are editable. `deposits.allocation_snapshot` is NEVER rewritten — frozen record of what was originally proposed. Editable ledger is `transactions`.
- Single backend transaction. Crash mid-edit → both writes abort.

**Edit can un-celebrate.** Edit a $400 Japan Trip transaction down to $40 — `current_balance` drops below `target_balance` → `celebrated_at` clears. Label updates to "Goal unfunded." Glow does NOT fire (clearing is silent; only UP crossings fire glow).

**Features touched:** 8.1 (edit transaction including date).

---

### 3.6 Overdraw display

Coffee envelope ran from $80 to -$12 across the cycle.

**Display (story 8.2):** tile balance text in red. Background tint slightly muted version of envelope color. Label: "Overdrawn -$12". Tapping the tile is the same as any envelope (recent transactions list). No special overdraft UI.

**Behavior:**

- Save not blocked. You can keep logging Coffee while overdrawn.
- Money-move helper auto-suggests pulling from Discretionary (Arc 3.4).
- Coach `envelope_runway` would have fired before the envelope went negative — overdraw is the situation that warning was trying to prevent.
- Next retainer allocation sees the negative in `envelopes` snapshot. LLM may propose a rebalance line; rules engine fallback just allocates per the normal rule order.

**No top-level "you're underwater" banner.** Per-tile signal + coach signal are enough.

**Features touched:** 8.2 (overdraw indicator).

---

### 3.7 Goal completion (the quiet celebration)

You log a $50 transaction into Japan Trip. The envelope tips over the target ($6,000 → $6,050).

**Trigger** is inside the `db.envelopes.update` call:

```js
const wasUnder = old.current_balance < old.target_balance;
const isNowOver = env.current_balance >= env.target_balance;
if (wasUnder && isNowOver && env.celebrated_at == null) {
  env.celebrated_at = nowISO();
}
```

**The celebration** — tile glows for ~3s (CSS keyframe ring). Small "Goal funded" label appears under the balance. **No confetti in v1, no sound** (deferred to v1.1). Glow runs once per session per envelope; reload later and the tile is a normal funded tile with a small "funded" badge.

**Crossings:**

- **Drop back below target** → clear `celebrated_at`. Label becomes "Goal unfunded" (small gray text). Future refund re-fires the glow.
- **Hit during retainer/bonus allocation** → glow fires only after Accept, on the Home re-render. Not during the show.
- **Multiple goals hit at once** → stagger 1.5s apart in `order_index` order. Sequenced after accept.

**Features touched:** 8.3 (goal glow with re-arm).

---

### 3.8 LLM down (the holistic view)

Three LLM-critical paths, each with a deterministic fallback:

| Path | LLM-down behavior |
|---|---|
| `POST /api/log/parse` | Backend returns `should_fallback: true`. Frontend opens tile-picker with amount pre-filled (regex parse) or empty. |
| `POST /api/allocation/propose` | Rules engine baseline becomes the proposal. Reasons read tighter. Animation identical. |
| `POST /api/coach/narrate` (internal) | Template-based fallback message body. Coach still fires, just less colorful. |
| `POST /api/review/narrate` (internal) | Template-based fallback narrative ("This cycle: $X spent..."). Review still ships. |

**Contract:** every LLM-dependent path has a deterministic fallback. The app's core workflow never blocks on Anthropic being up.

**Subtle 8px corner icon** on the Log input bar when the last LLM call errored. Auto-clears on next success. The only UI surface that exposes LLM-down state.

**Features touched:** 8.4 (LLM-dependent paths + fallback + corner icon).

---

### 3.9 Backup and recovery

**Primary v1 backup is automated.** Nightly cron on the VM:

```
sqlite3 /var/lib/budget/budget.db ".backup '/backup/budget-$(date +\%Y\%m\%d).db'"
rsync /backup/budget-*.db user@tailscale-host:/path/to/backup/
# OR
aws s3 sync /backup/ s3://your-bucket/budget-backups/
```

**Settings page shows "Last backup: N days ago"** read from the backup file mtime via a `GET /api/backup-status` endpoint. Quiet chip, no nag modal.

**Manual JSON export still exists in Settings** as paranoia. Downloads a single `budget-{date}.json` of all eight tables. Roughly 50–500 KB.

**Restore flow** — Settings → Import JSON. Picks a file, validates with Zod, wipes the DB inside a transaction, restores from the snapshot. If the JSON is corrupt, Zod rejects, the import refuses.

**Phone-wipe scenario.** Your laptop dies. You restore the SQLite file from the backup (S3 or Tailscale-shared folder), redeploy the backend pointing at it, and the frontend just works — the data was never on the device.

**Features touched:** 8.5 (export/import/backup chip).

---

### 3.10 The two-tab scenario

Laptop tab open. You open a second tab (or the phone). v1 ships a guard:

- `BroadcastChannel('budget-app-v1')` pings "I'm open" on mount.
- Second tab detects an existing tab → renders **"Already open in another tab. Refresh to take over."**
- Second tab is read-only until refresh.

**Cross-device note.** Phone + laptop is NOT covered by BroadcastChannel — it doesn't span devices. Acceptable for n=1; you're rarely editing both within the same minute. The backend is the source of truth; both tabs see the same data on next API call.

**Features touched:** 8.6 (two-tab guard).

---

## Feature → Arc map

USER-STORIES.md is organized by epic; this is the by-journey lens. Cross-reference:

| US epic | Arc | Coverage |
|---|---|---|
| 0 Foundation (0.1–0.6) | 1.1, threaded through all | Hosting, server-side key, datastore, seed-once, integer cents, atomic writes |
| 1 The Calm (1.1–1.8) | 1.2 | Default landing, tile-tap, free-text parse, confirmation chip, tile-picker fallback, tx_date, calm fill bar, glance-before-buying |
| 2 The Main Event (2.1–2.9) | 2.1–2.5 | Retainer detection, advisor proposal, rules-engine ground, reasons, inline edit, money-token show, atomic accept, under-paycheck alarm, reduced motion |
| 3 Bonus Drops (3.1–3.3) | 2.6 | Bonus skips obligations, nearest-goal default, light show, override |
| 4 The Coach (4.1–4.7) | 3.1 | Runway, envelope runway, goal slip, positive, review ready, nothing logged, batched ≤1/day |
| 5 The Review (5.1–5.5) | 3.3 | Stand-alone surface, narrative-led, behavioral + goal pace, deterministic numbers, supporting charts |
| 6 Projection Engine (6.1–6.3) | 3.2 | Spend cadence, slack + runway, cold-start window |
| 7 Money-Move Helper (7.1–7.4) | 3.4 | Quiet dismissible chip, advisor not agent, pull from genuine slack, cold-start guard |
| 8 The Edges (8.1–8.6) | 3.5–3.10 | Edit (incl. date), overdraw, goal glow + re-arm, LLM-down fallbacks, export/import/chip, two-tab guard |

---

## v1.1 deferrals

Carried from USER-STORIES.md:

- Override-correction logging (`corrections` table for prompt tuning)
- Large-deposit sanity check (confirm chip > $10k)
- Delete transaction (v1: edit to $0.01)
- Most-frequent auto-rotation of top tiles
- LLM clarifying-question re-prompt loop
- LLM confidence-calibration settings toggle
- Confetti on goal completion
- Cadence-aware "you forgot to log a deposit" detection [v2]
- Lightweight weekly spending pulse separate from monthly review [v2]
- Service worker queue-and-replay for offline writes

The diagnosis brief for v1.1 comes from actually using v1 across two complete retainer cycles.
