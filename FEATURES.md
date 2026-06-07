# Features — by user journey

Companion to [DESIGN.md](DESIGN.md). DESIGN.md is the contract; this is the journey.

Three arcs, ordered the way the user meets them:

- **Arc 1 — The Calm.** Day 0 setup and the daily logging ritual. Survives the app or kills it by week two.
- **Arc 2 — The Show.** Payday. The reason you actually open the app on the 1st and 15th.
- **Arc 3 — The Edges.** Recovery moments — goal hit, fixing a mistake, overdraw, LLM down, backup, two-tab. The trust layer.

---

## Arc 1 — The Calm

### 1.1 First boot (Day 0)

You install Tailscale on the phone, bookmark `http://laptop-tailscale-name:5173`, open it for the first time.

**Sequence**

1. `App.tsx` mounts, queries `envelopes.count()` on Dexie.
2. Count is 0 → `loadFromConfig()` runs inside one `db.transaction('rw', ['envelopes', 'recurring', 'app_config'], ...)`.
3. `config.yml` (already in memory via `?raw` import) is parsed by js-yaml. Envelope names resolve to generated IDs. `tax_envelope` and `discretionary_envelope` strings resolve to `tax_envelope_id` and `discretionary_envelope_id`.
4. Rows write. Every envelope's `current_balance` starts at 0.
5. Home renders. **Instant. No fade-in animation.** Day 0 should feel like opening a fresh notebook.
6. A single sub-grid line below the envelope grid: **"First paycheck? Tap + to allocate."** No CTA button, no modal. Just the hint.

**States**

| State | Trigger | Behavior |
|---|---|---|
| First-boot empty | `envelopes.count() === 0` | Seed runs, Home renders with $0 tiles |
| Partial DB | Seed threw mid-transaction | rw transaction aborts atomically, count stays 0, next boot retries |
| Config malformed | js-yaml throws | Boot-time error screen: line/col error from the parser |
| Missing envelope reference | `tax_envelope: "X"` but no envelope `X` | Seed throws with a clear error pointing at the orphan name |
| Past-deadline goal | `deadline ≤ today` | Seeds fine, Home tile shows "review goal" badge, advisor skips at payday |

**Features:** YAML seed-once, Tailscale install, envelope grid render.

---

### 1.2 Logging a coffee at 8:14am

You're at a cafe. $4 coffee. Pull out your phone, open the bookmark. **App opens to Log tab every time** (default landing for the hot path).

**The happy path (tap, sub-2-second)**

```
1. Log tab open. Top row: [Coffee] [Groceries] [Gas] [Eating Out] [Misc]
2. Tap [Coffee] ────────────────────────────────────► t=0s
3. Number pad modal slides up. Amount input focused.
4. Type "4" ─────────────────────────────────────────► t=~1.2s
5. Tap Save ─────────────────────────────────────────► t=~1.5s
6. Modal dismisses.
7. Toast: "$4.00 → Coffee"
8. Coffee tile fill bar animates the deduction (~300ms)
```

**Backing transaction:** one `db.transaction('rw', ['transactions', 'envelopes'])` writes the transaction row (`source: 'tile_tap'`, `paycheck_id: null`) and decrements `envelopes[coffeeId].current_balance` by 400 cents. Atomic.

**The free-text path**

```
1. Type "12 amex dinner" + enter ───────────────────► t=0
2. "..." chip appears below the input
3. LLM round-trip (Claude Haiku, max_tokens: 512) ──► t=~400ms
4. Zod validates the JSON:
   { intent: "transaction",
     amount_cents: 1200,
     envelope: "Eating Out",
     description: "amex dinner",
     confidence: 0.85 }
5. Confidence ≥ 0.7 → save with override chip:
   "$12.00 → Eating Out [change]" ─────────────────► t=~500ms
6. Same single rw transaction commits. Fill bar animates.
```

**The low-confidence fallback (universal safety net)**

```
1. Type "amazon 40" + enter
2. LLM returns confidence 0.45
3. App opens tile-picker modal with amount pre-filled at $40.
4. Tap [Misc] → save.
```

The tile-picker is the universal fallback for every failure mode: low confidence, network error, LLM timeout, malformed JSON.

**States**

| State | Trigger | Behavior |
|---|---|---|
| Loading | LLM call in flight | "..." chip. 10s AbortController timeout. |
| Empty input | Amount 0 or unparseable | Save button disabled, no error text |
| Network down | LLM call throws | Catch → tile-picker with amount pre-filled (or empty) |
| Rate-limited | Two parses within 500ms | Debounce serializes; ~2 calls/sec ceiling |
| Save in flight | Save tapped | Spinner; modal can't re-open on same input. Idempotent. |

**Edge cases**

- `12.5` / `$12.50` / `twelve dollars` all parse to 1200 cents on free-text. Tile-tap uses native number pad, digits only.
- **Override correction.** Tap the chip to change LLM's envelope pick. Transaction's `envelope_id` reassigns; `source` stays `text_parse`. v1: not logged separately. v1.1: add a `corrections` table for prompt-tuning data.
- **Back-to-back logging.** Tile-tap path: no debounce, hammer freely. Free-text path: 500ms debounce.
- **Paycheck phrasing.** `got paid 3200 from side gig` → parser returns `intent: "paycheck"`, routes to Payday view (Arc 2).

**Features:** envelope fill bar, tile-tap quick-log, free-text + LLM parse, confirmation chip, tile-picker fallback, paycheck intent.

---

## Arc 2 — The Show

### 2.1 Paycheck logged

It's payday. Log tab. Free-text bar. Type `got paid 3200`.

```
Input  → "got paid 3200"
Output → {
  intent: "paycheck",
  amount_cents: 320000,
  envelope: null,
  description: "got paid 3200",
  confidence: 0.96
}
```

Confidence ≥ 0.7 AND intent = paycheck → app stores `{amount_cents, description}` in component state and navigates to Payday view. **No transaction written yet.** Staging only.

**Edge cases**

- `got paid` (no amount) → confidence drops, tile-picker opens in paycheck mode (single amount input, no envelope picker).
- `got 3200` (no "paid") → ambiguous, confidence ~0.5, tile-picker lets you choose paycheck or transaction.
- Suspiciously large amount (> $10,000) → v1 trusts you, no warning. v1.1 deferred.

---

### 2.2 Loading the proposal

You're on the Payday view. Paycheck node "$3,200" at top with subtle pulse. Loading skeleton in the middle.

**Two things run in parallel:**

```js
// Step 1 — Deterministic baseline ALWAYS runs
const baseline = runRules(paycheckCents, appConfig, envelopes, recurring);

// Step 2 — Optional LLM advisor on top
let final;
try {
  const llmResp = await advisorPropose({
    paycheckCents, appConfig, envelopes, recurring, baseline
  });  // 10s AbortController
  final = ProposalSchema.parse(llmResp);  // Zod throws on malformed
  validateSumEquals(final, paycheckCents); // throws on bad math
} catch (err) {
  console.warn("LLM advisor unavailable, using rules baseline:", err);
  final = baseline;
}

setProposal(final);
```

The rules engine is the floor. The LLM is the friendly narrator and optional adjuster. If the LLM goes haywire, validate-and-fallback catches it. The user never sees broken state.

**The under-paycheck case (locked decision)**

If `tax + bills + subs + goal contributions` exceeds `paycheckCents`, the rules engine **truncates the last unmet rule(s).** Bills and tax are honored first. Goals get $0 if money runs out. Discretionary stays at $0.

A yellow chip renders at the top of the proposal:

> **"Bills + tax covered. Goals deferred this paycheck. Discretionary: $0."**

Truthful, deterministic, easy to reason about. No silent proportional scaling, no "insufficient paycheck" refusal screen.

---

### 2.3 The show

Proposal loaded. Animation kicks off.

**Choreography (~4s for a typical 8-row paycheck)**

```
T+0.0s   Paycheck node "$3,200" at top. Envelope tiles at bottom.
T+0.2s   First money token launches.
         - Sized: text scales by log(amount). $40 vs $1,200 distinguishable but not absurd.
         - Colored: matches destination envelope color.
         - Path: ease-out arc from paycheck node to destination tile, ~600ms.
T+0.5s   Token arrives. Tile fill bar leaps. Token dissolves.
         Reason line fades in below the proposal row in middle column.
T+0.8s   Next token launches. (300ms stagger between tokens.)
...
T+3.5s   Last token (Discretionary remainder).
T+4.0s   Accept + Edit buttons fade in at bottom.
```

**Driven entirely by the in-memory `proposal` object.** Not by Dexie live queries. Nothing has committed yet.

**Silent by default.** No audio.

**Reduced motion gate.** `@media (prefers-reduced-motion: reduce)` → tokens don't fly, rows fade in at the same 300ms stagger. Same information, same timing, no movement.

**Visual details**

- Token text size: `log(amount_cents)` scaled.
- Token color: matches destination envelope color.
- One token at a time. Never overlap.
- Loading > 3s during proposal phase → secondary state "Falling back to rules..." Loading > 10s → AbortController fires, baseline runs. Animation runs regardless.

---

### 2.4 Edit inline

Proposal shows. You want to push $100 from Discretionary to Japan Trip.

```
1. Tap the Japan Trip row's amount field. Number pad slides up.
2. Change 400 → 500. Save.
3. Discretionary auto-decreases: 250 → 150. (Discretionary is THE elastic envelope.)
4. Affected rows pulse briefly. Total stays $3,200.
```

**Edit rules**

- **Discretionary is the only elastic envelope.** Every other edit balances against Discretionary. Simpler mental model than cascading.
- **Sum invariant.** Validation runs on every edit. Exceeding paycheckCents → error chip: "Reduce X by $Y." Save disabled until valid.
- **Editing Tax allowed.** Tax is rule-driven but overridable. v1 no warning. v1.1 small "this overrides your tax rate" chip.
- **Negative amounts blocked.** Payday only adds money to envelopes; negatives are meaningless here.

---

### 2.5 Accept and commit

```js
await db.transaction('rw',
  ['paychecks', 'transactions', 'envelopes'],
  async () => {
    const paycheckId = await db.paychecks.add({
      timestamp: nowISO(),
      gross_amount: paycheckCents,
      allocation_snapshot: JSON.stringify(finalProposal)
    });

    for (const row of finalProposal) {
      await db.transactions.add({
        timestamp: nowISO(),
        amount: +row.amount_cents,
        envelope_id: row.envelope_id,
        description: row.reason,
        source: 'payday_allocation',
        paycheck_id: paycheckId
      });
      await db.envelopes.update(row.envelope_id, env => {
        const wasUnder = env.current_balance < (env.target_balance ?? Infinity);
        env.current_balance += row.amount_cents;
        const isNowOver = env.current_balance >= (env.target_balance ?? Infinity);
        if (wasUnder && isNowOver && env.celebrated_at == null) {
          env.celebrated_at = nowISO();
        }
      });
    }
  }
);
```

**Either every row commits or none do.** No partial paycheck state.

**On accept:**

1. Accept button shows checkmark (~200ms).
2. Fade transition back to Home.
3. Home re-renders with new balances.
4. Goal envelopes that crossed target during this paycheck glow now. **Stagger 1.5s apart in `order_index` order** if multiple goals hit. Not during the payday show.

---

### 2.6 The LLM-down version

Same flow. Same animation. Same accept. Reasons read tighter ("Cover bills next 30 days" instead of "$1,200 → Rent: covers next month's bill due 2026-07-01").

**Subtle 8px corner icon** appears on the input bar when last LLM call errored. No alarm chip. Auto-clears on next successful call.

---

## Arc 3 — The Edges

### 3.1 Goal completion (the quiet celebration)

You transfer $50 into Japan Trip. Balance crosses $6,000 target.

**Trigger** — inside the `db.envelopes.update` that changes `current_balance`:

```js
const wasUnder = old.current_balance < old.target_balance;
const isNowOver = env.current_balance >= env.target_balance;
if (wasUnder && isNowOver && env.celebrated_at == null) {
  env.celebrated_at = nowISO();
}
```

**The celebration** — tile glows for ~3s (CSS keyframe ring). Small "Goal funded" label appears under the balance. **No confetti in v1, no sound.** Glow runs once per session per envelope; reload later and the tile is just a normal funded tile with a small "funded" badge.

**Crossings**

- **Dropping back below target** → clear `celebrated_at`. Label changes to "Goal unfunded" (small gray text). Future refund re-fires the glow.
- **Hit during payday allocation** → glow fires **only after Accept**, on the Home re-render. Payday belongs to the paycheck.
- **Multiple goals hit at once** → stagger glows 1.5s apart in `order_index` order. Sequenced after accept.

---

### 3.2 Edit a logged mistake

8:14am. You meant $4 coffee, typed $40. Caught it 30s later.

```
1. Tap Coffee tile on Home. Expands to recent transactions list.
2. Tap the $40 transaction.
3. Edit modal: { amount: 40.00, envelope: Coffee, description: "" }
4. Change amount to 4.00. Save.
5. db.transaction('rw', ['transactions', 'envelopes']) runs:
   - Reverse: envelopes[Coffee].balance += 36_00 cents
   - Update: transactions[id].amount = -400 cents
   - paycheck_id stays null
```

Fill bar animates the $36 difference back up (same component, reverse direction).

**Envelope reassignment** — change Coffee → Eating Out: same modal, change dropdown.

```
- Reverse Coffee:  balance += 400
- Apply Eating Out: balance -= 400
- Update transactions[id].envelope_id
- source unchanged
- paycheck_id preserved (the snapshot is frozen — see 2.5)
```

**Edit rules**

- **Amount + envelope only.** No description edit, no source-type change, no delete in v1. (Delete deferred to v1.1; for now, edit to $0.01 to neutralize.)
- **Payday transactions are editable.** `paychecks.allocation_snapshot` is NEVER rewritten — it's a frozen record of what was originally proposed. Editable ledger is `transactions`.
- **Single rw transaction.** Crash mid-edit → both writes abort.

**Edit can un-celebrate.** Edit a $400 Japan Trip transaction down to $40. Balance was $6,000.50 (funded), now $5,640.50 (unfunded). `celebrated_at` clears. Label updates. Glow does NOT fire (clearing is silent; only the UP crossing fires the glow).

---

### 3.3 Overdraw display

Coffee envelope ran from $80 to -$12 over the paycheck period.

**Display**

- Tile balance text in red. Background tint slightly muted version of envelope color.
- Label: "Overdrawn -$12".
- Tapping the tile is the same as any envelope (recent transactions list). No special overdraft UI.

**Behavior**

- Save not blocked. You can keep logging Coffee while overdrawn.
- Payday advisor sees the negative balance in `envelopes_snapshot`. The LLM may propose a rebalance: "Coffee overdrawn -$12 — propose +$12 from Eating Out."
- Rules engine fallback doesn't auto-fix overdrafts; the next paycheck allocation covers via the normal flow, and Discretionary remainder backstops it.

**No top-level "you're underwater" banner.** Per-tile signal is enough for n=1.

**Between-paycheck transfers — deferred to v1.1.** v1 forces you to learn whether mid-period transfers are actually needed. Most weeks the discretionary remainder covers small drifts.

---

### 3.4 LLM down (the holistic view)

Three LLM-critical paths, each with a deterministic fallback:

| Path | LLM-down behavior |
|---|---|
| Log free-text parse | Tile-picker modal opens with amount pre-filled (regex parse) or empty |
| Log paycheck-intent detection | Tile-picker opens in paycheck mode |
| Payday proposal | Rules engine baseline runs. Reasons read tighter. Animation identical. |

**The contract:** every LLM-dependent path has a deterministic fallback. The app's core workflow never blocks on Anthropic being up.

**Subtle 8px corner icon** on the input bar when last LLM call errored. Auto-clears on next success.

---

### 3.5 Backup and phone-wipe

The honest open question of v1.

**v1 ships with manual Export JSON** in Settings (gear icon, top-right of Home). Downloads `budget-{date}.json` — all five tables in one file, 50–500 KB typical.

**Backup chip on the Settings page.** Quiet text: "Last backup: 23 days ago." No push notifications, no modal. Just visible when you visit Settings (which you'd visit anyway to see total budget).

**Restore flow** — Settings → Import JSON → pick file → Zod validates → wipes Dexie → loads the imported snapshot inside one rw transaction.

**Corrupted JSON** → Zod rejects, shows the schema mismatch, refuses to import.

**v1.5 auto-export** — recommended path: `fs.writeFileSync` from a small Node script you run weekly, OR a one-tap "Export to Downloads" button on Settings. No BFF required.

---

### 3.6 The two-tab scenario

Laptop tab open. You open a second tab (or the phone). v1 ships a guard:

- `BroadcastChannel` listens for "I'm open" pings on app boot.
- Second tab detects an existing tab → renders "Already open in another tab. Refresh to take over."
- Second tab is read-only until refresh.
- ~20 lines of code, prevents future you from double-logging the same coffee.

**Phone + laptop is cross-device** — `BroadcastChannel` doesn't span devices. Conflict resolution is "last write wins" by Dexie timestamp. Acceptable for n=1; you're not editing both in the same minute.

---

## Feature → Arc map

| # | Feature | Arc(s) |
|---|---|---|
| 1 | YAML config + seed-once | 1.1 |
| 2 | Tailscale install | 1.1 |
| 3 | Envelope tile grid | 1.1, 3.1, 3.3 |
| 4 | Envelope fill-bar animation | 1.2, 3.2 |
| 5 | Goal completion glow | 3.1 |
| 6 | Overdraw indicator | 3.3 |
| 7 | Tile-tap quick-log | 1.2 |
| 8 | Free-text + LLM parse | 1.2, 2.1 |
| 9 | Confirmation chip | 1.2 |
| 10 | Tile-picker fallback | 1.2, 2.1, 3.4 |
| 11 | Paycheck intent detection | 2.1 |
| 12 | Edit transaction | 3.2 |
| 13 | LLM advisor proposal | 2.2, 2.3 |
| 14 | Inline payday edit | 2.4 |
| 15 | Money-token flow animation | 2.3 |
| 16 | Rules engine fallback | 2.2, 2.6, 3.4 |
| 17 | Tax allocation | 2.2 |
| 18 | Discretionary remainder | 2.2, 2.4 |
| 19 | Recurring informs advisor | 2.2 |
| 20 | Manual JSON export | 3.5 |

Plus, locked in this session:

| New | Feature | Arc |
|---|---|---|
| 21 | Under-paycheck yellow chip | 2.2 |
| 22 | Reduced-motion gate | 2.3 |
| 23 | LLM-down corner icon | 2.6, 3.4 |
| 24 | Two-tab guard (BroadcastChannel) | 3.6 |
| 25 | Settings page + backup chip | 3.5 |

---

## v1.1 deferrals

These came up across the arcs and were intentionally deferred:

- Override-correction logging (`corrections` table for prompt-tuning data)
- Large-paycheck sanity check (confirm chip when > $10,000)
- Delete transaction (v1: edit to $0.01)
- Between-paycheck transfer modal
- Auto-export schedule (Tailscale folder sync or GitHub repo via BFF)
- Confetti for goal completion
- Most-frequent tile auto-rotation
- LLM clarifying-question re-prompt loop
- LLM confidence calibration Settings toggle
- True PWA install on iOS (HTTPS, service worker)

The diagnosis brief for what becomes v1.1 comes from actually using v1 for two weeks.
