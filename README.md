# trading-bot-poc-design

Design & architecture for an automated trading bot proof of concept, built on
the **Interactive Brokers (IBKR) Web API** with an intraday trend-following
first strategy.

> **This repository is design-only — no runnable code.** It captures broker
> research, system architecture, and a concrete first strategy spec.

## Contents

- [`DESIGN.md`](./DESIGN.md) — the full design document.

## What's inside

1. **Background** — why ING's / EU neobrokers' open-banking APIs can't power a
   trading bot (PSD2 scope = payment accounts only; no securities holdings).
2. **Broker API selection** — comparison of IBKR, Alpaca, TradeStation, OANDA/IG,
   DEGIRO, Trade Republic, Scalable Capital, US brokers, and crypto exchanges.
   Ranked recommendation for an EU-based developer: **IBKR** (paper account).
3. **IBKR Web API — verified endpoint map** — session (`/tickle`), WebSocket
   (`wss://api.ibkr.com/v1/api/ws`, `smd+`/`act` topics), market data
   (`/iserver/marketdata/snapshot`, `/hmds/history`), orders (including the
   critical two-step `/iserver/reply/{id}` confirmation loop for stop/bracket
   orders), and portfolio (`/portfolio/{id}/positions`).
4. **System architecture** — reference diagram + component breakdown (Market
   Data Ingest, Strategy Engine, Risk Manager, OMS, Portfolio/State Manager,
   Persistence, Broker Adapter, Observability, Dashboard). Event-driven data
   flow, idempotency, reconciliation, crash recovery, risk gate.
5. **Reliability & failure modes** — defences for duplicate orders, lost fills,
   stale quotes, session drops, weekly 2FA, dangling stops, timezone, halts,
   clock drift, runaway strategies.
6. **First strategy — intraday trend following** — long-only 9/21 EMA
   crossover on 1-min bars, three exits (trend flip, broker-side STP stop-loss,
   15:55 ET time stop), 1% risk sizing, 09:35–15:55 ET window. Mapped onto the
   architecture with IBKR-specific gotchas and a backtest-before-live checklist.
7. **POC scope & build order** — suggested 10-step implementation sequence.

## Status

Research and design complete. Implementation is the next phase.

## Sources

See §8 of [`DESIGN.md`](./DESIGN.md) for the full source list.