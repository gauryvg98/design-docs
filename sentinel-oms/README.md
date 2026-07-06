# Sentinel — Execution-Integrity-First OMS

Architecture for **Sentinel**, an order management system for automated crypto trading that
I built end-to-end: **Python 3.11 / asyncio** over **PostgreSQL**, a **FastAPI + WebSocket**
terminal, trading Binance / Bybit perpetuals. It runs a fleet of independent per-symbol bots
on one exchange account.

The design has one organising idea: **the OMS refuses to lie about your position.** It is an
**append-only event ledger** with a **pure state machine**; orders, positions and fills are
**derived projections** you can rebuild from the log at any time. When the ledger and the
exchange disagree it **reconciles against broker truth and halts on genuine disagreement** —
"halt, don't absorb." A timeout is not a rejection; an unknown is not a zero; we never invent
exposure the exchange doesn't hold.

— **Yashvardhan Gaur** · [github.com/gauryvg98](https://github.com/gauryvg98)

> **Low-level design** — DB schema, component breakdown and glossary: **[lld.md](lld.md)**.

---

## Diagrams

### 1. System overview
One account-scoped `SentinelApp` owns the durable core — the event ledger, command gateway,
reconciler, task supervisor and single-writer coordinator. An `InstrumentManager` runs a
fleet of independent bots (each: market feed, bar clock, exchange rules, strategy, order
terminal). A `Venue` abstraction hides Binance-futures / Bybit / spot. The browser talks to
it all over a single WebSocket that pushes only what changed.

![System overview](diagrams/01-system-overview.svg)

### 2. Event-sourced write path
Commands pass the guards, then **append an immutable event**. State moves only through one
pure, total function — `transition(order, event) → order` — and orders / positions / fills
are folded from the log. Drop the projections and you rebuild them by replay: the log is the
source of truth.

![Event-sourced write path](diagrams/02-event-sourced-write-path.svg)

### 3. Order state machine
The subtlety that separates a real OMS from a toy is what it does when it *doesn't know*. A
submission that times out is not rejected — its outcome is unprovable, so it parks in
`UNKNOWN` and only reconciliation may move it. A fill can race ahead of the ack; a cancel can
be confirmed by the venue that we never requested (self-trade prevention). Each is an
explicit, legal, tested edge — not an exception that crashes the loop.

![Order state machine](diagrams/03-order-state-machine.svg)

### 4. Reconciliation — halt, don't absorb
On restart (and on any live divergence) the OMS rebuilds projections, back-fills missed fills
idempotently (exec-id dedup → exactly-once), then compares booked quantity to broker truth.
**Booked > broker** is invented exposure — halt at once. **Booked < broker** may be the fill
feed lagging `executedQty` mid-fill — retry a few times, then halt only if the gap survives.
A pure connectivity **timeout** means we learned nothing — retry, never halt. Only a genuine
disagreement about exposure stops the account.

![Reconciliation decision](diagrams/04-reconciliation-decision.svg)

### 5. Data model
An append-only log is the truth; `orders` / `positions` / `fills` are projections folded from
it and rebuildable by replay. The safety properties live in **DB constraints** — `command_id
UNIQUE` (idempotency), `exec_id` PRIMARY KEY (exactly-once exposure), `filled_qty <= qty`
(no overfill). Full per-table docs in **[lld.md](lld.md)**.

![Database schema](diagrams/05-db-schema.svg)

---

## Strategies — a plug-in interface

A strategy is one **narrow, pure contract**: evaluate a closed bar → `Decision(stance, detail)`,
where `stance` ∈ {`LONG`, `FLAT`, `SHORT`, `None`-while-warming} and `detail` is an open dict of
risk hooks. Two entry points exist — `on_bar(close)` and the richer `on_bar_ohlcv(Bar{high,
low, close})` — and the runner prefers whichever the strategy exposes. Strategies are **pure
and deterministic** (no I/O, clock, or RNG): that buys unit-testing, replay-safety, and the
guarantee that a strategy — however sophisticated — reconciles through the same gateway a human
does and can never corrupt the ledger.

The strategies that ship (`sma`, `sma-ls`, `regime`) are deliberately trivial — the point is
execution integrity, not alpha. What matters is that the *interface* extends to arbitrarily
rich signals **without touching the OMS**:

- **Widen the input, not the interface.** A bar today is OHLC — price only, not even volume yet.
  The runner already "prefers the richer method if the strategy exposes it"; the same pattern
  adds a `FeatureBar` (volume, VWAP, realized vol, funding, open interest, order-book imbalance,
  cross-asset features) behind an `on_features(…)`. A strategy accumulates its own window and
  derives whatever it needs — `regime` already computes ATR, ADX, Donchian channels, z-scores
  and realized vol from the bar stream.
- **The output already fits ML.** `Decision(stance, detail)` maps onto a model's outputs:
  direction → `stance`; edge / probability → `target_weight` (conviction, [0,1]); predicted
  vol / stop → `stop_dist`. An ML model plugs in as a **pure predictor** — weights loaded once
  at construction, deterministic inference, features from the fed inputs — so `MLStrategy(
  model_path, feature_spec)` satisfies the same contract as an SMA; its "many parameters" are
  just construction config. Live external data is fed *in* as features; training is offline.
- **Risk stays orthogonal to alpha.** However the signal is derived, a strategy only ever emits
  two risk hooks — **`stop_dist`** (*where the thesis breaks*; the risk layer sizes **and**
  enforces the SL off the same distance) and **`target_weight`** (*how convinced*; scales the
  size). The risk layer turns those into size, brackets and margin —
  `qty = risk_pct · equity_share · conviction / stop_distance`, leverage-capped. Swap an SMA for
  a neural net and the sizing / execution / reconciliation machinery is untouched. The
  pluggability is the capability.

| Strategy | Signal | Risk hook |
|---|---|---|
| `sma` / `sma-ls` | fast/slow SMA crossover (long-only / stop-and-reverse) | `stop_dist` = distance to the slow SMA, floored |
| `regime` | Donchian breakout, ADX-gated, with a z-score MR overlay in range regimes | `target_weight` — vol-targeted conviction |

---

## Engineering properties worth calling out

- **Integrity over cleverness.** The exchange is the source of truth for *exposure*; the
  ledger for *intent and history*; any contradiction is a first-class, halt-worthy event
  rather than something to paper over. The strategy is replaceable — the integrity is the
  product.
- **The core is pure.** The domain has no I/O and no clock — the entire state machine and
  every guard are unit-tested before a broker or feed exists (315 tests). A bad strategy can
  trade badly but can never corrupt the ledger, because `transition()` is the only way state
  moves.
- **Rebuildable by construction.** Projections are a fold of the event log — drop them and
  replay. Continuously-checked invariants (`positions = fills`, `no_overfill`,
  `exits_bounded`, `audit_traced`) prove the fold stayed honest.
- **Fail toward safety, not brittleness.** Timeouts, unknown broker statuses,
  self-trade-prevention cancels and lagging fill feeds are all *legal, tested transitions* —
  each was a real production halt before it was handled. Transient (retry) is separated
  explicitly from genuine divergence (halt), so the system is neither brittle (halting on
  every network blip) nor unsafe (trading through real divergence).
- **A fleet on one account.** Tens of bots share one ledger, one user-data stream and one
  margin pool. A single-writer coordinator serialises writes per instrument; instrument rules
  (lot / tick / min-notional / settlement asset) are fetched from the venue, never hardcoded;
  and each bot sizes off its *slice* of the settlement-asset pool it actually draws on — so N
  bots can never each size against the whole account.
- **Risk tied to the stop.** Size is `risk_pct · equity_share · conviction / stop_distance`,
  leverage-capped — the *same* stop feeds both the position size and the protective stop-loss,
  so they can never disagree. A strategy can supply its own stop geometry (e.g. distance back
  to the trend line); liquidation distance is derived live from the streamed mark.
- **The browser polls nothing.** A topic-based change signal pushes one bot card, or the
  account, over a single WebSocket, with a client liveness watchdog that reconnects a
  silently-dead socket instead of freezing.

---

*Content describes engineering patterns in a personal project. Diagrams are hand-authored SVG
(committed directly), themed to match the Sentinel terminal.*
