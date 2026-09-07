# 03: Pre-trade risk and account view

**What to build:** An order that the account cannot afford comes back `REJECTED` with a typed reason instead of being accepted. Buys need cash for quantity times price (limit price for limit orders, reference price for market orders); sells need enough unreserved position; every order must be under the per-order notional cap; a global kill switch rejects all submits while on. Accepting a buy reserves cash on the account, accepting a sell reserves quantity on the position, and a reject reserves nothing. The decision runs inside one transaction that locks the account row, so two concurrent buys cannot both pass on the same cash. `GetAccount` shows cash, reserved cash and positions. `SetKillSwitch` toggles the halt.

**Blocked by:** 01 (Submit and fetch an order over gRPC).

**Status:** ready-for-agent

- [ ] Risk is a pure function over account, position, symbol, request and kill switch, returning accept or a typed reason: insufficient cash, insufficient position, notional exceeds limit, trading halted, unknown symbol, invalid quantity, invalid price
- [ ] Table tests cover every reason and boundary values (cash exactly equal to notional, zero and negative quantity)
- [ ] Risk rejects return a normal `SubmitOrder` response with status `REJECTED` and reason, not a gRPC error
- [ ] Accept and reservation happen in one transaction with `SELECT ... FOR UPDATE` on the account; ADR written for the row lock as serialization point
- [ ] `GetAccount` returns cash, reserved cash and positions with reserved quantity
- [ ] `SetKillSwitch` persists to a settings row read inside the accept transaction
- [ ] Notional cap configurable by env var
