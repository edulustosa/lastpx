# 09: Synthetic trader

**What to build:** A demo viewer runs `docker compose up` and, without touching anything, sees a continuous stream of orders being accepted, filled, partially filled, rejected and canceled. The `trader` binary loops at a configurable rate across the seeded accounts and symbols using only the public gRPC client: mostly valid limit orders near reference price, some market orders, a configurable share deliberately invalid (too much cash, selling what is not held, over the notional cap), and a configurable share cancelled after a short delay.

**Blocked by:** 07 (Cancel orders end to end).

**Status:** ready-for-agent

- [ ] `TRADER_RATE`, `TRADER_INVALID_PCT`, `TRADER_CANCEL_PCT` env vars with compose defaults
- [ ] Uses only the generated gRPC client; no database or broker access
- [ ] Graceful shutdown on signal
- [ ] Trader joins the core compose profile; after `docker compose up` the `events` fanout shows a steady mix of every terminal status
