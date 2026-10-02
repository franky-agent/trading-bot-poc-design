# Trading Bot POC — Design & Architecture

> **Status:** Design / architecture only — no code. Research-grounded; sources cited inline.
> **Audience:** A Europe-based developer building a first fully-automated trading bot as a proof of concept.
> **Scope:** Broker API selection, system architecture, and a concrete first strategy (intraday trend following).

---

## Table of Contents

1. [Background & Problem](#1-background--problem)
2. [Broker API Selection](#2-broker-api-selection)
3. [IBKR Web API — Verified Endpoint Map](#3-ibkr-web-api--verified-endpoint-map)
4. [System Architecture](#4-system-architecture)
5. [Reliability & Failure Modes](#5-reliability--failure-modes)
6. [First Strategy — Intraday Trend Following](#6-first-strategy--intraday-trend-following)
7. [POC Scope & Roadmap](#7-poc-scope--roadmap)
8. [Sources](#8-sources)

---

## 1. Background & Problem

### 1.1 Origin

The project started with a question about ING's Open Banking Developer Portal
(`developer.ing.com/openbanking`) as a data source for a portfolio report
(top-5 winners/losers of the day). Findings:

- ING exposes **only PSD2-mandated payment-account APIs**: AIS (Account
  Information Services), PIS (Payment Initiation), and CAF (Confirmation of
  Available Funds), plus a Showcase greeting endpoint.
- AIS endpoints (`GET /v3/accounts`, `…/balances`, `…/transactions`) return
  **cash balances and transaction movements** on payment accounts — not
  securities holdings. No ISIN, no quantity, no security price, no portfolio
  valuation.
- Securities/investment accounts are **outside PSD2 scope** EU-wide; no bank is
  required to expose them, and ING does not do so on this portal.
- EU neobrokers (DEGIRO, Trade Republic, Scalable Capital) offer **no official
  self-trading API** for retail automation. Trade Republic has only a PSD2 TPP
  API (account info, not order placement for self).

**Conclusion:** A portfolio/winners-losers report or a trading bot cannot be
built on ING's or EU neobrokers' open-banking APIs. The project pivoted to
brokers with real trading APIs and to designing a bot architecture.

### 1.2 Goal

Build a **fully-automated trading bot POC** that can:
1. Retrieve live account balances and portfolio positions.
2. Stream real-time market data.
3. Compute trading signals from chart/trend analysis.
4. Place, modify, and cancel orders programmatically.
5. Get order status/fills back and reconcile state.

Focus is **day trading** with **chart/trend analysis** for the first strategy.

---

## 2. Broker API Selection

### 2.1 Decision criteria that matter for *fully-automated*

1. **Headless server capable?** (no GUI required)
2. **Session autonomy** — survives without human intervention? (MFA? restarts? timeouts?)
3. **Official API** vs unofficial/scraped? (unofficial = fragile + T&C violation)
4. **EU resident can legally open account + trade desired instruments?**
5. **Paper/sandbox** for development without capital risk.

### 2.2 Broker comparison

| Broker | Official API | Transport | Headless? | Session autonomy | EU resident? | Instruments | Paper | Verdict |
|---|---|---|---|---|---|---|---|---|
| **Interactive Brokers** | Yes (Web API / Client Portal / TWS / IB Gateway) | REST + WS + socket | ✅ via IB Gateway / Client Portal Gateway | ⚠️ Weekly 2FA re-auth (Sun ~01:00 ET); `/tickle` ~60s; daily restart | ✅ IBKR Ireland (CBI-regulated) | EU + US equities, ETFs, options, futures, forex, bonds | ✅ paper accounts | **Best overall** |
| **Alpaca** | Yes | REST + WebSocket | ✅ pure cloud | ✅ API key+secret, no MFA, no GUI | ✅ individuals in 195+ countries | **US equities/options/crypto only** (intl individuals); Alpaca Europe = B2B only | ✅ | Cleanest DX, US-markets-only for solo devs |
| **TradeStation** | Yes | REST + WS | ✅ | ✅ OAuth | ⚠️ US-centric; intl access limited | US equities, options, futures | ✅ | Strong API, poor EU fit |
| **OANDA / IG** | Yes | REST + streaming | ✅ | ✅ API token | ✅ EU regulated (FCA/ESMA) | **CFDs/forex only** — no real stock ownership | ✅ demo | Wrong instrument class for equity bot |
| **DEGIRO** | ❌ no official API | unofficial/scraped | n/a | session cookies, breaks often | ✅ EU | EU + US equities, ETFs, options | ❌ | Fragile, against T&Cs |
| **Trade Republic** | PSD2 TPP API only (account info, **not** order placement for self) | REST | n/a | n/a | ✅ EU | EU + US equities, ETFs | ❌ | Cannot place orders via API |
| **Scalable Capital** | ❌ no public trading API | n/a | n/a | n/a | ✅ EU | EU + US ETFs/equities | ❌ | Not viable |
| **Schwab / Fidelity / E*TRADE** | Yes | REST + WS | ✅ | ✅ OAuth | ❌ US residents only | US equities, options | ✅ | Ruled out for EU resident |
| **Binance / Kraken / Coinbase** | Yes | REST + WS | ✅ | ✅ API key | ✅ (with local restrictions) | Crypto only | ✅ testnet | Only if POC is crypto |

### 2.3 Ranked recommendation

**#1 — Interactive Brokers (IBKR).** The only broker that simultaneously:
- Accepts EU residents via a locally regulated entity (**IBKR Ireland**, CBI-regulated).
- Gives access to **both EU and US** equities/ETFs/options/futures.
- Has a real, documented, **headless-capable** REST + WebSocket API.

The **IBKR Web API** is REST/WebSocket over HTTPS with JSON. Retail connects
through a lightweight Java **Client Portal Gateway** (binds `localhost:5000`)
or **IB Gateway** (headless TWS). No minimum funding; trading API is free;
market-data subscriptions cost extra (per exchange in Account Management).
Paper accounts are first-class.

**The one real pain point:** the retail brokerage session is not fully
unattended. Security tokens are invalidated every Sunday ~01:00 ET, requiring a
2FA re-approval. The `/tickle` endpoint must be hit roughly every 60 seconds
or the session drops in ~6 minutes. The platform does a daily restart.

**Mitigations:**
- (i) Run on paper during development; accept the weekly Sunday ping.
- (ii) Create a second IBKR user with reduced permissions and **no MFA** (IBKR
  allows this) for the automation account.
- (iii) If institutional: switch to **OAuth 2.0 `private_key_jwt`** flow — fully
  unattended.
- The open-source **IBC (IBC Alpha)** tool automates gateway startup/shutdown
  (even via crontab) and handles the daily restart, but cannot push the weekly
  2FA button.

**#2 — Alpaca (US markets only).** Cleanest developer experience: pure cloud
REST + WebSocket, API-key auth, no local process, no MFA, no GUI, paper
subdomain identical to live. Individuals in 195+ countries. Supports fractional
shares, bracket/OCO, options (US, beta). **Hard limitation for EU:** no
European equities for individual international accounts (Alpaca Europe is
B2B-only under MiFID II). US PDT rule (≥3 day trades / 5 business days on
margin account < $25k) bites for intraday. Best for a US-markets-only POC.

**#3 — OANDA (FX/CFD).** Only if the POC is specifically FX/CFD. Excellent REST
+ streaming v20 API, fully headless, EU-regulated, demo environment. But CFDs
are derivatives — no share ownership, ESMA leverage limits.

**Selected broker for this POC: IBKR (paper account).**

---

## 3. IBKR Web API — Verified Endpoint Map

> Verified against the current IBKR Campus docs
> (`interactivebrokers.com/campus/ibkr-api-page/`, 2025–2026).

### 3.1 What's new in the docs (2025–2026)

- **"IBKR Web API"** is now the unified product name, merging Client Portal Web
  API + Digital Account Management + Flex Web Service under shared **OAuth 2.0**
  (`private_key_jwt`, RFC 7521/7523). **Old endpoints are NOT deprecated** — they
  keep working and receiving updates.
- **Two auth paths:**
  - *Institutional / third-party:* OAuth 2.0 directly to `api.ibkr.com` — fully
    unattended, no GUI, no weekly 2FA. Requires onboarding by IBKR API Solutions.
  - *Retail / individual (POC):* **Client Portal Gateway** — Java program at
    `localhost:5000`. Account must be fully open, funded, **IBKR Pro** type.
    Paper account = `DU…` prefix, tied to live account. Weekly Sunday 2FA caveat
    applies.
- **TWS API Synchronous Wrapper** (Oct 27, 2025) — simplifies the callback
  model but still requires TWS/Gateway GUI. Web API remains the headless choice.
- **"Overnight + Day" Smart Orders** (Jan 2025) — new TIF (overnight + next day
  4am–8pm as SMART order). Noted for later; not used in v1.

**Net effect on architecture:** no structural change. The broker adapter targets
the Web API; the only additions are automating the **two-step order
confirmation** and the `/tickle` keep-alive.

### 3.2 Session & authentication

| Endpoint | Method | Purpose |
|---|---|---|
| `/v1/api/tickle` | GET | **Call every ~60 s.** Returns `session` id and `iserver.authStatus.authenticated`. Without it the session drops in ~6 min. |
| `/iserver/accounts` | GET | **Must call once before any order operation** (confirms account list server-side). |

### 3.3 WebSocket — `wss://api.ibkr.com/v1/api/ws`

(Replace `api.ibkr.com` with `localhost:5000` when using the Gateway.)

- **Auth:** reuse the session cookie from `set-cookie`, OR send
  `{"session":"<tickle value>"}` over the socket after opening it.
- The socket can linger behind a reverse proxy — **send a message to activate** it.
- **Subscription topics:**
  - `smd+{conid}` — streaming market data (quotes/trades).
  - `act` — **account/order updates** (fills, status changes) — this is how the OMS gets fill pushes.
  - `sor` — smart order routing.
  - `blp` — bulletins.

### 3.4 Market data (REST)

| Endpoint | Method | Notes |
|---|---|---|
| `/iserver/marketdata/snapshot?conids=265598,8314&fields=31,55,84,86` | GET | Live snapshot, comma-separated conids. Field codes: `31`=last price, `55`=symbol, `84`=bid, `86`=ask. **First call often returns only `coindex`/`conid` to "instantiate the stream"; follow-up call returns data.** Some fields (option greeks) take up to a minute. |
| `/hmds/history?conid=265598&period=1w&bar=1d&outsideRth=true&barType=midpoint` | GET | Historical OHLC. **One conid at a time.** `bar` = `{Xmin,Xh,Xd,Xw,Xm}`, `period` = `{Xmin…Xy}`, `barType` = `{last,midprice,bid,ask,inventory}` (`last` also returns volume `v`). |
| `/iserver/secdef/info?conid=…` | GET | Contract details. |
| `POST /iserver/secdef/search` | POST | Contract search by symbol. |

### 3.5 Orders (REST) — the two-step confirmation is the key automation detail

| Endpoint | Method | Notes |
|---|---|---|
| `/iserver/account/{accountId}/orders` | POST | Body `{orders:[{conid, orderType, side, tif, quantity, price}]}`. `orderType`: `MKT`/`LMT`/`STP`/`STP_LMT`/`TRAIL`. Stops use `price` for the stop; `STP_LMT` and `TRAIL` use `auxPrice` for the stop price. Returns `order_id`, `order_status` (e.g. `presubmitted`), **and for precautionary order types (stops, brackets) a `reply` id**. |
| `/iserver/reply/{replyId}` | POST | Body `{confirmed:true}`. **Required to actually transmit** stop/bracket/precautionary orders. **Can chain** (multiple reply-ids returned one after another). The execution adapter **must** automate this reply loop, or the stop-loss sits in `presubmitted` forever. |
| `/iserver/account/{accountId}/orders/whatif` | POST | Pre-flight margin/buying-power preview (use as a risk-gate check before real submission). |
| `/iserver/account/orders` | GET | Open orders (for reconciliation). |
| `/iserver/account/{accountId}/order/{orderId}` | DELETE | Cancel. |
| `/iserver/account/{accountId}/order/{orderId}` | PATCH | Modify. |

### 3.6 Account / portfolio (REST) — the data ING denied

| Endpoint | Method | Returns |
|---|---|---|
| `/portfolio/{accountId}/positions` | GET | **Real positions** (the data source for the winners/losers report). |
| `/portfolio/{accountId}/summary` | GET | `netliquidation`, `totalcashvalue`, etc. |
| `/portfolio/{accountId}/performance` | GET | P&L. |

---

## 4. System Architecture

### 4.1 Reference architecture (text diagram)

```
                       ┌─────────────────────────────────────────────────────────┐
                       │                   TRADING BOT (your process(es))          │
                       │                                                         │
   ┌──────────────┐    │  ┌───────────────┐   events   ┌──────────────────────┐   │
   │  BROKER API  │◀──▶│  │ Market Data   │───────────▶│  Signal / Strategy   │   │
   │ (IBKR/Alpaca)│    │  │  Ingest       │            │      Engine          │   │
   │              │    │  │  (WS + REST)  │            │  (pluggable strats)  │   │
   │  - quotes    │    │  └───────────────┘            └──────────┬───────────┘   │
   │  - positions │    │         │  ticks/bars                    │ orders        │
   │  - orders    │    │         ▼                                ▼               │
   │  - fills     │    │  ┌───────────────┐            ┌──────────────────────┐   │
   └──────────────┘    │  │  Indicator    │            │   Risk Manager       │   │
        ▲  reconnect   │  │  Cache        │            │  (pre-trade gate)    │   │
        │  reconcile   │  └───────────────┘            └──────────┬───────────┘   │
        │              │                                          │ approved      │
        │              │                                          ▼               │
        │              │                               ┌──────────────────────┐   │
        │              │                               │  Order Mgmt System   │   │
        │              │                               │  (OMS: state machine)│   │
        │              │                               └──────────┬───────────┘   │
        │              │                                          │ submit        │
        └──────────────│──────────────────────────────────────────┘               │
                       │   fills/rejects                                          │
                       │   ▼                                                      │
                       │  ┌────────────────────┐    snapshot    ┌──────────────┐  │
                       │  │  Portfolio / State │───────────────▶│ Persistence  │  │
                       │  │  Manager           │    events     │  (DB + WAL)  │  │
                       │  └────────────────────┘               └──────────────┘  │
                       │           │                                              │
                       │           ▼  metrics/state                                │
                       │  ┌────────────────────┐  ┌───────────────────┐           │
                       │  │  Observability     │  │  Dashboard / API  │           │
                       │  │  logs+metrics+alert│  │  (read-only view) │           │
                       │  └────────────────────┘  └───────────────────┘           │
                       └─────────────────────────────────────────────────────────┘
```

### 4.2 Components and responsibilities

**Market Data Ingest** — the single ingress for all price data. Subscribes to
the broker's WebSocket (`smd+{conid}`) and falls back to REST polling for
snapshots/history. Normalises broker-specific schemas into one internal
`Tick`/`Bar`/`Quote` format. Handles reconnection with backoff and **gap
detection** (on reconnect, fetch historical bars via `/hmds/history` to fill
gaps before resuming). Pure producer — emits events onto an internal queue,
knows nothing about strategies.

**Signal / Strategy Engine** — consumes normalised market events, produces
*intents* ("buy 10 AAPL @ limit 195"), not orders. **Critical rule: the
strategy never talks to the broker directly.** It emits an intent; the Risk
Manager decides whether it lives. Strategies are pluggable behind a small
interface contract: `on_bar`, `on_tick`, `on_fill`, `on_timer`. Indicators
computed incrementally on the streaming bar (no full-series recompute each tick).

**Risk Manager** — synchronous pre-trade gate that every order intent must pass
through. Holds its **own** copy of positions/cash (does not trust the
strategy's view). Checks: max position size, max gross exposure/leverage, daily
loss limit, single-order notional cap, banned-symbol list, kill-switch flag,
market-open/closed check, stale-data guard. A circuit breaker trips on N
consecutive rejections or drawdown breach → all strategies paused → alert.

**Order Management System (OMS)** — state machine for orders. Tracks each order
through `NEW → SUBMITTED → PARTIALLY_FILLED → FILLED` and
`… → CANCELLED / REJECTED / EXPIRED`. Assigns a **client_order_id**
(idempotency key) so a retry after a network blip doesn't double-submit. On
(re)connect, **reconciles** in-memory state against the broker's view of open
orders and recent fills. Emulates broker-side order types the broker lacks, but
prefer native bracket/OCO when available to avoid race conditions.

**Portfolio / State Manager** — maintains authoritative current positions,
cash, realised/unrealised P&L, updated from fill events (not polling — polling
is only for reconciliation). Input to the Risk Manager's exposure checks and to
the dashboard's winners/losers report.

**Persistence** — durable store for: open orders, positions, cash, trade
journal (every fill), strategy parameters, and the event log. Two patterns:
- *(a) Snapshot + changelog* (POC-pragmatic): periodic snapshot of
  positions/orders to SQLite/Postgres + append-only events table.
- *(b) Event sourcing*: store every event, derive state by replay. Perfect
  auditability, more work.

For a POC, (a) is sufficient; design the schema so upgrading to (b) is a
refactor, not a rewrite.

**Execution / Broker Adapter** — the only component that knows the broker's
wire format. Wraps REST calls, WebSocket subscriptions, auth/session keep-alive
(`/tickle` every 60s), and the **two-step reply-loop confirmation**. Translates
internal order intents ↔ broker order requests. Isolating this means swapping
brokers is one adapter, not a rewrite.

**Observability** — structured logging (every order lifecycle transition logged
with client_order_id), trade journal table, metrics (P&L, slippage, fill
latency, signal-to-order latency, rejections, API errors, drawdown), alerting
on anomalies (unexpected position, API disconnect > 60s, drawdown breach, kill
switch tripped). For a POC: structured logs to stdout + metrics table +
Telegram/email alert on kill switch.

**Dashboard / Read API** — read-only view over Portfolio/State Manager and the
trade journal. Small FastAPI endpoint serving JSON + static page, or Grafana
over the metrics DB. **Never** has write access to orders. This is the natural
home for the original winners/losers report, now backed by real position data
from `/portfolio/{accountId}/positions`.

### 4.3 Data flow & event model — event-driven, not polling

A robust bot is **event-driven**: WS tick → ingest normalises → publishes to
queue → strategy's `on_tick`/`on_bar` fires → maybe emits an order intent →
risk check (synchronous, inline) → OMS submits via adapter → fill comes back on
the `act` WS topic → OMS updates state → portfolio manager updates P&L →
dashboard sees it. Polling is reserved for reconciliation (every N seconds) and
non-streamed data.

**Design rules:**
- One internal event bus/queue with typed events (`Tick`, `Bar`, `Fill`,
  `OrderUpdate`, `Timer`).
- **Backpressure:** bounded queue; "drop oldest tick, keep latest" on the
  market-data side is fine for bar-based strategies.
- **Order placement is synchronous within the decision but async to the
  broker:** the strategy call returns after the OMS *accepts* the intent
  (passed risk + assigned client_order_id), not after the broker acknowledges.
- **Single-writer for broker calls:** the execution adapter is the only thing
  issuing orders, behind a lock/queue.

### 4.4 Strategy engine — pluggable, backtestable

The strategy contract is identical in backtest and live:

```
interface Strategy:
    on_bar(bar: Bar)        # each completed bar
    on_tick(tick: Tick)     # optional, for tick strategies
    on_fill(fill: Fill)     # own fills — update internal state
    on_timer(ts: datetime)  # scheduled callbacks (e.g. daily rebalance)
```

In backtest, `on_bar` is driven by historical bars replayed in order. In live,
`on_bar` is driven by the streaming bar aggregator. The strategy emits orders
through the same `submit_intent()` interface; in backtest that's a simulated
fill engine, in live it's the real OMS. Live and backtest differ only in the
data source and fill engine, not the strategy code.

### 4.5 OMS — order states, idempotency, reconciliation

**States:** `NEW` (intent accepted, not sent) → `SUBMITTED` (sent, awaiting
ack) → `PARTIALLY_FILLED` → `FILLED` / `CANCELLED` / `REJECTED` / `EXPIRED`.
Track: quantity submitted, quantity filled, average fill price, broker order
id, client_order_id, timestamps per transition.

**Idempotency:** every order gets a `client_order_id` (UUID). If a REST call
times out and you retry, the broker dedupes on this id. Without this, a network
timeout mid-submit is the classic cause of duplicate orders.

**Partial fills:** plan for them. The OMS accepts multiple fill events against
one order, each reducing remaining qty. Position manager updates incrementally
per fill, not per order.

**Reconciliation on reconnect:** (1) fetch all open orders from broker; (2)
diff against in-memory open orders; (3) for any order the broker shows filled
that the OMS didn't see → apply the fill; (4) for any order the broker doesn't
know about that the OMS thinks is open → mark stale and alert. The only way to
be sure of state after downtime.

**Emulating OCO/bracket:** if the broker doesn't support it natively, you watch
prices and cancel the sibling leg when one fills — this has a race (price gaps,
both legs fill before your cancel lands). IBKR supports bracket/OCO natively;
use the native version.

### 4.6 State & persistence — survive a crash

**Must be durable:** open orders, positions, cash, trade journal, strategy
parameters/flags, kill-switch state (a restart must not resume trading if the
kill switch was tripped).

**Recovery procedure on startup:**
1. Load last snapshot of positions/cash/open orders from DB.
2. Connect to broker.
3. **Reconcile:** fetch broker's current positions, open orders, today's fills;
   diff against snapshot; apply missed fills; alert on mismatches.
4. Only after reconciliation is green, set `trading_enabled = true` (keep it
   false during reconcile).
5. Begin market-data streaming.

### 4.7 Risk management — the gate that saves you

Non-negotiable pre-trade checks, evaluated **synchronously** before any order
leaves the process:

- **Max position size** per symbol (absolute qty and % of portfolio).
- **Max gross exposure** / leverage (Σ|position values| ≤ X × equity).
- **Daily loss limit:** realised+unrealised P&L today < −threshold → stop
  accepting new orders for the rest of the day.
- **Per-order notional cap** (sanity: no single order > X).
- **Symbol allow/deny list** (no penny stocks, no illiquid names).
- **Market session check:** don't submit outside trading hours unless
  extended-hours is intended.
- **Kill switch:** manual + automatic flag blocking all new orders. Tripped by:
  drawdown breach, N consecutive rejections, broker API errors > threshold,
  unexpected position detected.
- **Stale-data guard:** if last tick for a symbol is older than T seconds,
  refuse to trade it.

**Design rule:** the risk manager holds its own copy of positions/cash, updated
from fills, and does *not* trust the strategy's view. A strategy that thinks
it's flat when it's actually long 100 is the most common blowup; the risk gate
rejects the second buy.

---

## 5. Reliability & Failure Modes

| Failure | Defence |
|---|---|
| **Duplicate orders** on REST timeout+retry | `client_order_id` idempotency; never retry without it |
| **Lost fills during downtime** | reconciliation on reconnect; fetch today's executions and diff |
| **Stale quotes** (feed dies mid-trade) | stale-data guard; if no tick for T seconds, flatten/cancel and alert |
| **Partial fill ambiguity** | OMS tracks filled qty per order; position updates per fill, not per order |
| **Broker session drop** (IBKR 6-min idle) | `/tickle` every 60s in the adapter; auto-reconnect with backoff; reconcile after |
| **Weekly 2FA (IBKR retail)** | schedule Sunday restart via IBC; alert if session down; consider non-MFA sub-user |
| **IBKR stop/bracket not transmitting** | automate the `/iserver/reply/{id}` confirmation loop; test on paper |
| **Dangling stop on flat position** | OMS cancels resting STP when position closed by another exit; reconcile bracket on restart (if position but no stop → resubmit stop) |
| **Timezone/session errors** | track exchange calendar explicitly (pandas-market-calendars); never assume local time = market time |
| **Market halt / halt-resume** | listen for trading-status messages; cancel resting orders on halt |
| **Clock drift** | NTP on the host; log both your timestamp and the broker's timestamp on every event |
| **Runaway strategy** | risk gate + kill switch + daily loss limit — strategy cannot bypass these |

---

## 6. First Strategy — Intraday Trend Following

### 6.1 Why this strategy as the first one

For a *first automated* strategy:
- **Mechanically unambiguous** (no judgement) so the bot can execute unattended.
- **Exercises the full pipeline** (data → signal → risk → OMS → fill →
  dashboard) without exotic order types.
- **Fails safe** — worst case is "no position", not "stuck in a leveraged trade".
- **Well-understood and backtestable** so live behaviour can be sanity-checked
  against a backtest.

The classic fit: a **single-name intraday moving-average crossover with a hard
time stop and a hard stop-loss**. It rides short-term momentum and exits when
momentum flips or the day ends. It will lose money in choppy markets (every
trend strategy does) — that's the *point* of a POC: validating machinery, not
beating the market.

### 6.2 Strategy spec

**Universe:** one liquid US large-cap (AAPL) or ETF (SPY for calmer behaviour).
One symbol only for the POC.

**Timeframe:** 1-minute bars. Aggregate the `smd` WebSocket stream into 1-min
OHLC bars in the Market Data component (don't poll `/hmds/history` intraday —
that endpoint is for history/backfill, not live trading).

**Indicators (two EMAs):**
- Fast EMA: 9-period (≈ first ~9 minutes of momentum).
- Slow EMA: 21-period.
- **EMAs not SMAs** — faster reaction intraday, and O(1) incrementally
  updatable: `EMA_today = α·price + (1−α)·EMA_yesterday`, `α = 2/(N+1)`.

**Entry rules:**
- **Long entry:** fast EMA crosses *above* slow EMA (golden cross on 1-min).
- **Short entry:** **disabled in v1** — long-only. A death cross means *exit*,
  not go short.

**Exit rules (three independent exits, any one fires):**
1. **Trend exit:** fast EMA crosses back below slow EMA.
2. **Hard stop-loss:** price drops `-X%` from entry fill (start `-0.5%` for
   1-min on a liquid name; tune in backtest). Implemented as a **real STP order
   resting at the broker** so you're protected even if the process dies.
3. **Time stop / flat by close:** if still in a position at 15:55 ET, market-out.
   **Never carry overnight** in a day-trading POC.

**Position sizing:** fixed fractional — risk ~1% of equity per trade.
`shares = (equity × 0.01) / stop_distance`. Cap at a max absolute shares to
avoid accidents.

**Order types used:**
- Entry: **LMT** at current ask (or MKT during learning). `tif=DAY`.
- Stop-loss: **STP** submitted *immediately after the entry fills*, resting at
  `entry × (1 − stop%)`. This is a precautionary order → **the reply-loop
  automation in the adapter is mandatory.** Use a native broker-side stop, not
  "watch and sell" — protection when the bot is offline.
- Time-stop / trend-exit: **MKT** sell when signal fires or at 15:55; then
  **cancel the resting STP** (OMS must cancel the stop when the position is
  closed by another exit — otherwise a dangling sell stop on a flat position).

**Trading window:** 09:35–15:55 ET (skip first 5 min to avoid open volatility;
flat 5 min before close). Bot's clock must be exchange-time — track the US
market calendar explicitly (pandas-market-calendars), never assume local time.

### 6.3 How it maps onto the architecture

```
Market Data Ingest
  ├─ WebSocket "smd+{conid}" → 1-min bar aggregator → emits Bar events
  └─ On (re)start: GET /hmds/history to backfill ~50 1-min bars so EMAs are warm
        ↓ Bar event
Strategy Engine (EMA Cross strategy)
  ├─ Updates EMA9 / EMA21 incrementally from each Bar
  ├─ Detects crossover (prev_fast<=prev_slow AND fast>slow)
  └─ Emits OrderIntent{side=BUY, qty, type=LMT, price=ask}  OR  ExitIntent
        ↓ intent
Risk Manager (synchronous gate)
  ├─ Max 1 position (reject if already long)
  ├─ Per-trade notional cap & 1% risk sizing check
  ├─ Daily loss limit (stop trading if day P&L < -threshold)
  ├─ Kill-switch & stale-data guard (reject if last bar > 30s old)
  └─ whatIf pre-flight: POST /iserver/account/{id}/orders/whatif
        ↓ approved
OMS
  ├─ Assigns client_order_id, submits LMT via adapter
  ├─ On Fill event (from "act" WS): records fill, updates position
  ├─ Immediately submits child STP stop-loss → handles reply-loop confirmation
  └─ Watches for exit triggers (trend flip / 15:55) → MKT sell → cancel resting STP
        ↓ fills
Portfolio/State Manager → updates position, cash, realized P&L
        ↓
Persistence (SQLite) + Dashboard (winners/losers view, now real) + Alerts
```

### 6.4 IBKR-specific details this strategy forces you to handle

1. **The reply loop for the stop-loss.** The STP order returns a `reply` id;
   the adapter must `POST /iserver/reply/{id}` with `{confirmed:true}`, and
   handle a possible second reply id. If not automated, the stop never
   transmits and the "hard stop-loss" is fictional. **#1 thing to test on paper.**
2. **Child-order cancellation on alternative exit.** When the trend flips and
   you MKT-sell, `DELETE` the resting STP. The OMS treats (entry fill → stop
   submitted) and (exit fill → stop cancelled) as a linked bracket and
   reconciles on restart: if a position exists but no resting stop → **resubmit
   the stop immediately**.
3. **Bar warmup.** EMAs need history. On startup, backfill ~50 1-min bars via
   `/hmds/history` before enabling the strategy, so the first crossover is real,
   not noise from a cold EMA.
4. **Snapshot "first call instantiates the stream" behaviour.** If using the
   REST snapshot for a quick quote (e.g. entry limit price), call it twice —
   the first call primes the stream. Prefer the `smd` WebSocket for live prices.
5. **Session keep-alive.** The adapter's background `/tickle` every 60 s must
   keep running; if it stops, both WS and REST die within minutes. Monitor as a
   health metric.
6. **Paper first.** Run on the `DU…` paper account for weeks. The *expected*
   outcome on paper is a modest loss to breakeven (trend strategies on 1-min
   are noisy) — you're validating that fills, stops, exits, reconciliation, and
   the dashboard all behave, not that the strategy is profitable.

### 6.5 Backtest-before-live checklist (same strategy code)

Before connecting the broker, run the identical `on_bar` logic against
historical 1-min bars (pull via `/hmds/history` with `bar=1min&period=1y`)
through a simple simulator that fills at the next bar's open after a signal.
Confirm:
- Number of trades/day is sane (9/21 EMA on 1-min AAPL ≈ 3–10 round-trips/day;
  if it's 50+, you have a bug).
- No trade is held past 15:55.
- Every long has a corresponding stop submitted and either triggered or cancelled.
- Max drawdown on paper is within the daily-loss-limit you set.

If backtest and live paper disagree on trade count or P&L, the bug is almost
always in the data pipeline (missed bars, timezone) or the fill model — not the
strategy.

### 6.6 What to deliberately *not* do in the first POC

- **No shorting.** Long-only.
- **No options, no leverage, no margin.** Cash account, single name.
- **No multi-symbol scanning.** One symbol.
- **No "AI"/ML signals.** Two EMAs. Save complexity for v2.
- **No live money.** Paper account only until the bot runs a full week
  unattended with zero unexpected positions and zero dangling orders.

---

## 7. POC Scope & Roadmap

### 7.1 Recommended POC scope

1. **Broker:** IBKR paper account (`DU…` prefix).
2. **Strategy:** 9/21 EMA crossover on 1-min bars, one liquid US large-cap.
3. **Mode:** **scheduled-daily first** (run once at market open, reconcile,
   stream during the session, flat by 15:55, exit). Graduate to always-on
   intraday only after the daily version is boringly reliable.
4. **Risk:** max 1 position, max $X notional, daily loss limit, kill switch —
   implemented from day one, even on paper.
5. **Persistence:** SQLite with `positions`, `open_orders`, `fills`, `orders`,
   `events` tables. Snapshot-on-shutdown, reconcile-on-startup.
6. **Dashboard:** single read-only page showing current positions, today's
   fills, P&L, kill-switch status (extends the original winners/losers report
   idea, now backed by real `/portfolio/{id}/positions` data).
7. **Observability:** structured logs + metrics table + one Telegram alert on
   kill-switch / unexpected position.

### 7.2 Build order (suggested)

1. **Broker adapter skeleton** — `/tickle` keep-alive, `/iserver/accounts`,
   session health metric. Prove the session stays up.
2. **Market data ingest** — `smd+` WS subscription, 1-min bar aggregator,
   `/hmds/history` backfill on startup.
3. **Persistence + reconciliation** — SQLite schema, startup reconcile against
   `/portfolio/{id}/positions` and `/iserver/account/orders`.
4. **OMS core** — order state machine, `client_order_id`, submit LMT via
   `/iserver/account/{id}/orders`, fill handling from `act` WS.
5. **Reply-loop automation** — `/iserver/reply/{id}` for STP orders; test
   submitting and confirming a stop on paper.
6. **Risk gate** — max 1 position, notional cap, daily loss limit, kill switch,
   stale-data guard, `whatif` pre-flight.
7. **Strategy engine** — pluggable interface, EMA9/21 incremental, crossover
   detection, backtest simulator on historical bars.
8. **Dashboard** — read-only positions/fills/P&L view.
9. **Observability + alerts** — structured logs, metrics, Telegram alert.
10. **Paper week** — run unattended for a full week; gate on zero unexpected
    positions and zero dangling orders before any live consideration.

---

## 8. Sources

- **ING Open Banking Developer Portal** — `developer.ing.com/openbanking/home`,
  AIS Sandbox PDF v1.11 (June 2023). Confirms PSD2-only APIs (AIS/PIS/CAF), no
  securities endpoints.
- **IBKR API home** — `interactivebrokers.com/campus/ibkr-api-page/ibkr-api-home/`.
  Unified Web API documentation; TWS API Synchronous Wrapper (Oct 27, 2025);
  Overnight + Day Smart Orders (Jan 2025).
- **IBKR Trading Web API** — `interactivebrokers.com/campus/ibkr-api-page/web-api-trading/`.
  OAuth 2.0 `private_key_jwt` (RFC 7521/7523); Client Portal Gateway for retail;
  endpoint reference.
- **IBKR Client Portal API docs (deprecated mirror)** —
  `interactivebrokers.github.io/cpwebapi/`. REST API, headless gateway, no fees,
  paper + live.
- **IBKR WebSocket tutorial** —
  `interactivebrokers.com/campus/ibkr-quant-news/tutorial-web-api-how-to-connect-to-websocket/`.
  `wss://api.ibkr.com/v1/api/ws`, session-cookie or `{"session":...}` auth,
  `smd+`/`act`/`sor`/`blp` topics.
- **IBKR "Requesting Market Data" lesson** —
  `interactivebrokers.com/campus/trading-lessons/requesting-market-data/`.
  `/iserver/marketdata/snapshot` fields (31/55/84/86), first-call-primes-stream
  behaviour; `/hmds/history` params.
- **IBKR "Placing Orders" lesson** —
  `interactivebrokers.com/campus/trading-lessons/placing-orders/`.
  `/iserver/account/{id}/orders` (MKT/LMT/STP/STP_LMT/TRAIL), the two-step
  `/iserver/reply/{id}` confirmation loop, `whatif` pre-flight.
- **IBKR session behaviour** — `ib-gateway-docker` discussion #145 (weekly
  Sunday re-auth, IBC crontab, MFA limitation); `agentic-trading-mcp` `keeper.py`
  (`/tickle` every 60s, ~6min idle drop, ~24h max session).
- **Alpaca docs** — `docs.alpaca.markets`. WebSocket streaming
  (`paper-api.alpaca.markets/stream`, `data.alpaca.markets/stream`), order
  types, options beta; Alpaca EU docs (B2B/Fintech Partner model only);
  "non-US resident" guide (195+ countries).
- **TradeStation API docs** — `api.tradestation.com/docs/`. REST, US-centric.
- **Trade Republic PSD2 TPP Integration Guide v2.1** — TPP/account-info only.
- **OANDA v20 REST dev guide** — `developer.oanda.com/rest-live-v20/`. REST +
  streaming, CFD/forex.
- **Event-sourcing risk engine pattern** — NordVarg blog (Oct 2024). Informs
  the persistence/reconciliation design.
- **SMA crossover strategy reference** —
  `backtestingengine.com/examples/basic/sma-crossover`; `quantifiedstrategies.com/moving-average-crossover/`.

---

*This document is design and architecture only — no runnable code. Build order
and POC scope in §7 are the suggested next steps when moving to implementation.*