# OMS + FIX gateway

Status: ready-for-agent

## Problem Statement

I want a portfolio project that proves, to a foreign hiring manager, that I can build a production-shaped trading backend in Go: idiomatic code and project layout, real concurrency, synchronous gRPC, asynchronous RabbitMQ, the FIX protocol, an investments domain, and first-class observability. Nothing I have today shows that. Later, a web UI with an LLM narrating the system in real time will sit on top of it, so the backend must emit the events and logs that UI will need from day one.

## Solution

A sell-side brokerage Order Management System (OMS) and a FIX gateway, in one Go monorepo, runnable end to end with docker compose:

1. A client submits an order over gRPC. The OMS runs pre-trade risk (cash, position, max notional, kill switch), persists the order and an outbox event in one transaction, and acknowledges synchronously.
2. An outbox relay publishes the accepted order to RabbitMQ. The FIX gateway consumes it and sends a NewOrderSingle (35=D) over a FIX 4.4 session.
3. A simulated exchange accepts the session, fills, partially fills, or rejects the order with configurable probabilities and latency, and answers with ExecutionReports (35=8).
4. The gateway publishes each ExecutionReport to RabbitMQ. The OMS consumes it, advances the order state machine, updates cash and position, and publishes an `order.updated` event on a fanout exchange for future consumers (the web UI).
5. A synthetic trader generates continuous order flow so the system is alive when someone looks at it.
6. Every service exposes Prometheus metrics, emits OpenTelemetry traces, and logs structured JSON with trace ids. Grafana, Jaeger, Loki and an OTel Collector ship in the compose file under a separate profile.

## User Stories

### Submitting and tracking orders

1. As a trader, I want to submit a limit buy order for a US equity over gRPC, so that it is routed to the exchange.
2. As a trader, I want to submit a market order, so that it fills at the current reference price.
3. As a trader, I want to submit a sell order, so that I can reduce a position I hold.
4. As a trader, I want the submit call to return synchronously with the order id and status, so that I know immediately whether the OMS accepted it.
5. As a trader, I want a rejected order to come back with a machine-readable reason (insufficient cash, insufficient position, notional too large, trading halted, unknown symbol, invalid quantity), so that I can act on it.
6. As a trader, I want to fetch an order by id and see its current status, filled quantity, leaves quantity, average fill price and last update time, so that I can follow its life.
7. As a trader, I want to cancel a working order, so that I stop an unfilled remainder.
8. As a trader, I want a cancel of an already filled or already canceled order to be refused with a clear reason, so that I do not misread the state.
9. As a trader, I want a cancel to be routed to the exchange as an OrderCancelRequest (35=F) and confirmed by an ExecutionReport, so that the cancel is real and not just local.
10. As a trader, I want to fetch my account and see cash balance, reserved cash, and positions per symbol, so that I understand my buying power.
11. As a trader, I want to supply my own client order id, so that I can retry a submit safely and get the same order back instead of a duplicate.
12. As a trader, I want a day order that is still working when the session ends to be expired, so that no stale order lives forever. (Deferred to Out of Scope; kept here as the intended behavior.)

### Pre-trade risk

13. As a risk officer, I want a buy to be rejected when the account lacks cash for quantity times reference price, so that clients cannot overspend.
14. As a risk officer, I want cash to be reserved on accept and released on reject, cancel or expiry, so that two concurrent buys cannot both pass on the same cash.
15. As a risk officer, I want a sell to be rejected when the account holds less than the quantity, so that there is no short selling.
16. As a risk officer, I want a per-order notional cap, so that a fat-finger order is stopped before it reaches the exchange.
17. As a risk officer, I want a global kill switch that rejects every new order while on, so that I can halt trading in one action.
18. As a risk officer, I want the kill switch to still allow cancels, so that halting does not strand working orders.
19. As a risk officer, I want risk decisions logged and counted per reason, so that I can see why orders are being rejected.

### Routing and FIX

20. As an operator, I want the gateway to keep one FIX 4.4 initiator session with the exchange, so that orders have a path out.
21. As an operator, I want the session to use a file store and not reset sequence numbers on logon, so that a restart resends what the counterparty missed and gap fills are handled by the engine.
22. As an operator, I want the gateway to buffer accepted orders while the FIX session is down and send them when it logs back on, so that a disconnect does not lose orders.
23. As an operator, I want an accepted order that the gateway has already sent to not be sent again when the same message is redelivered, so that at-least-once delivery does not create duplicate exchange orders.
24. As an operator, I want every ExecutionReport to be published to the OMS exactly as received (order status, exec type, last quantity, last price, cumulative quantity, leaves, exec id), so that the OMS owns state and the gateway stays dumb.
25. As an operator, I want a duplicate ExecutionReport (same exec id) to be ignored by the OMS, so that redelivery does not double-fill.
26. As an operator, I want an ExecutionReport for an unknown order to be logged and dropped, not crash the consumer, so that one bad message does not stop the flow.
27. As an operator, I want the OMS to fail a message to a dead-letter queue after repeated processing errors, so that poison messages do not block the queue.

### Exchange simulator

28. As a developer, I want a simulated exchange that accepts a FIX 4.4 session, so that the gateway has a real counterparty.
29. As a developer, I want the simulator to fully fill, partially fill in two executions, or reject an order with probabilities I set by environment variables, so that the system shows every order state.
30. As a developer, I want the simulator to add random latency within a configurable range before responding, so that the latency histograms show a distribution.
31. As a developer, I want the simulator to fill at the reference price plus small random noise, so that fills look like fills.
32. As a developer, I want the simulator to honor a cancel while leaves quantity remains and reject it otherwise, so that cancel paths are exercised.
33. As a developer, I want the simulator to be deterministic under a fixed seed and zero latency, so that the end-to-end test is stable.
34. As a developer, I want the simulator to reject a market order when market orders are disabled by configuration, so that the reject path can be forced.

### Synthetic flow

35. As a demo viewer, I want a trader binary that submits random orders across seeded accounts and symbols at a configurable rate, so that the system is alive without manual input.
36. As a demo viewer, I want a share of those orders to be intentionally invalid (too much cash, selling what is not held, over the notional cap), so that rejections appear in the stream.
37. As a demo viewer, I want the trader to occasionally cancel a working order, so that cancels appear in the stream.

### Observability

38. As an operator, I want a Prometheus histogram of order round-trip latency from submit to terminal state, so that I can see tail latency.
39. As an operator, I want a Prometheus histogram of FIX round-trip latency from 35=D sent to first 35=8 received, so that I can separate exchange latency from OMS latency.
40. As an operator, I want counters of orders by terminal state and rejections by reason, so that the dashboard tells a story.
41. As an operator, I want gauges for FIX session state, outbox backlog and queue consumer lag, so that I can see the system stalling before users do.
42. As an operator, I want one OpenTelemetry trace to follow an order across gRPC, the outbox, RabbitMQ, the FIX gateway and back, so that I can open Jaeger and see the whole life of one order.
43. As an operator, I want every log line as JSON with service, trace id, order id and client order id when known, so that Loki queries and a future LLM can correlate them.
44. As an operator, I want a provisioned Grafana dashboard with those panels, so that docker compose up shows something.
45. As an operator, I want the observability stack behind a compose profile, so that I can run the core system alone on a small machine.

### Future web UI

46. As a UI developer, I want every order state change published as a protobuf event on a fanout exchange, so that a web gateway can subscribe without touching the OMS.
47. As a UI developer, I want events to carry the full order snapshot, not a delta, so that a late subscriber can render from the latest event.
48. As a UI developer, I want the event proto to live next to the gRPC proto in one buf module, so that the frontend can generate types from it.

### Developer experience

49. As a developer, I want one make target that generates protos and sqlc code, so that I never run tools by hand.
50. As a developer, I want docker compose up to bring the core system to a working state with seeded accounts and symbols, so that a first run just works.
51. As a developer, I want go test to run unit tests fast without infrastructure, so that the feedback loop is tight.
52. As a developer, I want the end-to-end test to run under a build tag with testcontainers, so that it is opt-in.
53. As a developer, I want CI to run vet, golangci-lint and tests on every push, so that main stays green.
54. As a reader of the repo, I want the layout to follow Go community conventions (cmd, internal, flat packages named by what they provide), so that it reads as native Go.

## Implementation Decisions

### Monorepo and binaries

- Single Go module `github.com/edulustosa/lastpx`, Go 1.27. Directories: `cmd` (one main per binary), `internal` (all application code), `proto` (buf module), `migrations` (goose), `deploy` (compose and observability config). The web app is added later as a sibling directory with its own bun toolchain.
- Four binaries: `oms`, `fix-gateway`, `exchange-sim`, `trader`. Each main wires dependencies explicitly, no DI framework. Configuration by environment variables read at startup into a plain struct, with defaults that work under compose.
- Every binary handles SIGINT/SIGTERM with context cancellation and drains in-flight work before exit.
- Language: English for code, comments, commits, docs.

### Domain

- Money and quantity use `shopspring/decimal`. Postgres columns are `NUMERIC`. quickfixgo already uses this type for price and quantity fields.
- Entities: Account (cash, reserved cash), Position (account, symbol, quantity, average cost), Symbol (ticker, reference price), Order.
- Order fields: id (UUID, also the FIX ClOrdID), client order id (caller-supplied idempotency key, unique per account), account, symbol, side, type (market, limit), limit price, quantity, filled quantity, average price, status, reject reason, timestamps.
- Order status mirrors FIX OrdStatus: `PENDING_NEW` (accepted by OMS, not yet acked by exchange), `NEW` (acked), `PARTIALLY_FILLED`, `FILLED`, `PENDING_CANCEL`, `CANCELED`, `REJECTED`. Terminal states: `FILLED`, `CANCELED`, `REJECTED`.
- The state machine is a pure function: current order plus execution report in, new order plus list of domain events out, or an error for an illegal transition. Illegal transitions are logged and dropped, never applied.
- Pre-trade risk is a pure function over (account, position, symbol, order request, kill switch) returning accept or a typed reject reason. Reasons: insufficient cash, insufficient position, notional exceeds limit, trading halted, unknown symbol, invalid quantity, invalid price.
- Cash reservation: on accept of a buy, reserve quantity times reference price (limit orders use limit price). On fill, move the filled notional from reserved to spent and release the difference. On reject, cancel or expiry, release the remainder. Sells reserve nothing but decrement a `reserved quantity` on the position so concurrent sells cannot oversell.
- Concurrency control for accept: one Postgres transaction that locks the account row (`SELECT ... FOR UPDATE`), runs risk, writes order and outbox event, commits. The database is the serialization point; there is no in-process lock.
- Kill switch: a single row in a `settings` table, read inside the same transaction. Toggled by SQL or an admin gRPC call (see API).
- Seed data: ten US symbols with fixed reference prices, five accounts with starting cash, applied by the last goose migration so compose up is enough.

### gRPC API (proto package `lastpx.oms.v1`)

- `SubmitOrder(account_id, client_order_id, symbol, side, type, quantity, limit_price)` returns the order snapshot. On a duplicate `client_order_id` for the same account, returns the existing order with the same response shape (idempotent). Risk rejects return a normal response with status `REJECTED` and a reason, not a gRPC error. Validation errors (missing fields, bad enum) return `InvalidArgument`.
- `CancelOrder(order_id)` returns the order snapshot with status `PENDING_CANCEL`, or `FailedPrecondition` if the order is terminal.
- `GetOrder(order_id)` returns the snapshot or `NotFound`.
- `GetAccount(account_id)` returns cash, reserved cash and positions.
- `SetKillSwitch(enabled)` toggles the global halt. No auth in v1; `account_id` is trusted from the request.
- Interceptors: OpenTelemetry tracing, Prometheus request metrics, structured request logging, panic recovery.

### Messaging (RabbitMQ, protobuf bodies)

- Direct exchange `orders` with routing keys `order.accepted` and `order.cancel_requested`, consumed by the gateway from queue `fix-gateway.orders`.
- Direct exchange `executions` with routing key `execution.report`, consumed by the OMS from queue `oms.executions`.
- Fanout exchange `events` receiving `OrderUpdated` (full snapshot plus the triggering execution) for future consumers. No queue is bound in v1; a consumer binds its own.
- All queues are durable with a dead-letter exchange; messages are persistent; consumers use manual ack with a bounded prefetch. On processing error the message is nacked without requeue after the broker-side retry count is exhausted (dead-letter with TTL requeue loop, capped by a header count).
- Message envelopes carry the W3C `traceparent` in AMQP headers so traces continue across the broker.

### Transactional outbox

- Table `outbox` (id, aggregate id, routing key, payload bytes, created at, published at). Written in the same transaction as the order change.
- A relay goroutine in the OMS polls unpublished rows in batches ordered by id, publishes with publisher confirms, marks them published. Poll interval short (tens of milliseconds) with backoff when empty. Single relay per process; multiple OMS replicas are out of scope.
- The gateway deduplicates `order.accepted` by order id using its own small table or the FIX message store: if a 35=D was already sent for that ClOrdID, the redelivery is acked and ignored.

### FIX gateway

- quickfixgo initiator, FIX 4.4, one session, file message store and file log under a data directory, `ResetOnLogon=N`, heartbeat 30 seconds. Session config is a template rendered from environment variables.
- Outbound: consumes `order.accepted` and builds `NewOrderSingle` (ClOrdID = order id, Symbol, Side, OrdType, OrderQty, Price for limit, TimeInForce = Day, TransactTime). Consumes `order.cancel_requested` and builds `OrderCancelRequest` (OrigClOrdID = order id, new ClOrdID = order id plus suffix).
- While the session is logged out, the consumer stops pulling from the queue (does not ack) so messages wait in RabbitMQ; it resumes on logon.
- Inbound: `FromApp` for `ExecutionReport` converts to an `ExecutionReport` protobuf (order id from ClOrdID, exec id, exec type, ord status, last qty, last px, cum qty, leaves qty, avg px, text, transact time) and publishes to `executions`. `OrderCancelReject` is mapped to an execution-like message with a reject flag so the OMS can revert `PENDING_CANCEL`.
- Records the FIX round-trip histogram keyed by the first ExecutionReport per ClOrdID.

### Exchange simulator

- quickfixgo acceptor, FIX 4.4, in-memory store (the simulator is allowed to forget). Same monorepo, same proto-free contract: only FIX in and out.
- On `NewOrderSingle`: after a random delay within `[SIM_LATENCY_MIN, SIM_LATENCY_MAX]`, choose an outcome by weights `SIM_FILL_PCT`, `SIM_PARTIAL_PCT`, `SIM_REJECT_PCT`. Always sends an ack (`ExecType=New`) first, then either one fill, two partial fills separated by another delay, or a reject. Fill price is reference price (from a static symbol map in the simulator) plus noise within `SIM_PRICE_NOISE_PCT`. Market orders rejected when `SIM_REJECT_MARKET=true`.
- On `OrderCancelRequest`: if leaves remain, respond `ExecType=Canceled`; else `OrderCancelReject`.
- `SIM_SEED` fixes the random source for deterministic tests.
- Keeps in-memory order state per ClOrdID guarded by a mutex; one goroutine per order lifecycle.

### Trader

- Loops at `TRADER_RATE` orders per second across the seeded accounts and symbols. Order mix: mostly valid limit orders near reference price, some market orders, `TRADER_INVALID_PCT` deliberately invalid, and `TRADER_CANCEL_PCT` followed by a cancel after a short delay. Uses the public gRPC client only.

### Observability

- Metrics: `promhttp` on each binary's `/metrics`. Histograms: `oms_order_roundtrip_seconds` (submit to terminal), `fix_roundtrip_seconds` (35=D to first 35=8), gRPC server histograms via the standard interceptor. Counters: orders by terminal status, rejections by reason, execution reports by exec type, outbox published, messages dead-lettered. Gauges: FIX session logged on, outbox unpublished count.
- Tracing: OpenTelemetry SDK with OTLP exporter to the collector. Spans: gRPC handler, risk, DB transaction, outbox publish, gateway consume, FIX send, FIX receive, OMS execution apply. Context propagated through AMQP headers.
- Logging: `log/slog` JSON handler to stdout. A handler wrapper injects `trace_id` and `span_id` from context. Common fields: `service`, `order_id`, `client_order_id`, `account_id`, `symbol`, `exec_id`.
- Compose profile `observability`: Prometheus, Grafana (provisioned datasources and one dashboard), Jaeger all-in-one, Loki, Promtail, OTel Collector. Core profile: Postgres, RabbitMQ (management UI), the four binaries.

### Database

- Postgres 16. Migrations with goose in SQL. Queries in SQL files compiled by sqlc into typed Go using pgx v5.
- Tables: `accounts`, `symbols`, `positions`, `orders`, `executions` (unique on exec id), `outbox`, `settings`.
- Unique constraints: `orders(account_id, client_order_id)`, `executions(exec_id)`.

### CI

- GitHub Actions on push and pull request: `go vet`, `golangci-lint`, `go test ./...`. The end-to-end test runs in a second job that has Docker available. Generated code is committed so CI does not need buf or sqlc installed, but a job checks that regeneration produces no diff.

## Testing Decisions

- A good test drives the system through a seam the product actually exposes and asserts on observable outcomes: a gRPC response, a row visible through `GetAccount`, an event on the fanout. It never asserts on which function was called or on a repository mock.
- Seam 1, end to end, through the OMS gRPC API. Build tag `integration`. Testcontainers start Postgres and RabbitMQ. The test starts `oms`, `fix-gateway` and `exchange-sim` in process as goroutines with a shared context, with the simulator seeded and at zero latency. Scenarios: full fill updates cash and position; partial fill in two executions accumulates correctly; reject releases reserved cash; cancel of a working order ends `CANCELED` with leaves released; cancel of a filled order is refused; duplicate `client_order_id` returns the same order; kill switch rejects submits but allows cancels; duplicate execution report is ignored; an `OrderUpdated` event is published per state change and carries the full snapshot.
- Seam 2, the pure domain package: table tests for the order state machine (every legal transition, every illegal one) and for risk (each reject reason, boundary values such as cash exactly equal to notional, zero and negative quantity).
- No tests for the FIX gateway or simulator in isolation; both are covered by seam 1. No handler tests, no mocks.
- Prior art: none in this repo yet. Follow the standard library `testing` package with table-driven tests and `t.Run` subtests; use `testify/require` only if assertion noise becomes a problem.

## Out of Scope

- The web UI, the web gateway (SSE/WebSocket), and the LLM narrator. Only the fanout exchange and event proto are built now.
- Order replace (35=G), good-till-cancel, stop orders, end-of-day expiry of day orders.
- Live market data, order books, matching between orders.
- Short selling, margin, per-symbol limits, per-account limits beyond notional cap.
- Authentication and multi-tenancy on the gRPC API.
- Multiple OMS replicas or multiple FIX sessions.
- Kubernetes; docker compose only.

## Further Notes

- The machine running this is resource constrained. Heavy commands (build, test, lint) run sequentially, never in parallel. The observability profile exists so the core system can run alone.
- Decisions that are hard to reverse and deserve an ADR under `docs/adr`: transactional outbox over publish-after-commit; `shopspring/decimal` over integer cents; FIX 4.4 with file store and no sequence reset; protobuf as the RabbitMQ body format; Postgres row lock as the risk serialization point.
- Delivery is in phases, each reviewed by the user before the next: skeleton and compose and proto; OMS gRPC with database and risk; outbox and RabbitMQ; exchange simulator and FIX gateway; observability; trader and CI.
- Research reference: quickfixgo v0.9.10 (Aug 2025) is active and supports FIX 4.4 with typed packages under `github.com/quickfixgo/fix44`; the code generator `generate-fix` is maintained. The official `qf executor` and `qf ordermatch` examples were considered and rejected as the exchange because the first cannot partially fill or cancel and the second is FIX 4.2 only.
