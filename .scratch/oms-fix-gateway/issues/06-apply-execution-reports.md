# 06: Apply execution reports and fill orders end to end

**What to build:** A submitted order now reaches a terminal state on its own. The OMS consumes `executions`, runs the order through a pure state machine (`PENDING_NEW` → `NEW` → `PARTIALLY_FILLED` → `FILLED`, or `REJECTED`), records each execution with a unique exec id so redelivery is ignored, moves reserved cash to spent and updates the position on fills, releases reservations on reject, and publishes an `OrderUpdated` event with the full order snapshot on the `events` fanout after every change. Illegal transitions and unknown orders are logged and dropped; a message that keeps failing is dead-lettered. This ticket also builds the end-to-end test harness: build tag `integration`, testcontainers for Postgres and RabbitMQ, the three binaries started in process with the simulator seeded and at zero latency, driven only through the gRPC client.

**Blocked by:** 05 (FIX gateway and exchange simulator).

**Status:** ready-for-agent

- [ ] State machine is a pure function: (order, execution report) → (order, events) or illegal-transition error; table tests cover every legal and illegal transition
- [ ] Fill updates filled quantity, average price, cash (reserved → spent, release remainder on full fill) and position quantity and average cost
- [ ] Reject releases reserved cash or reserved position quantity
- [ ] Duplicate exec id is ignored via unique constraint on executions
- [ ] Unknown order id or illegal transition is logged and acked, not requeued forever
- [ ] Repeated processing failure dead-letters the message after a bounded retry count
- [ ] `OrderUpdated` published on `events` fanout with full snapshot plus triggering execution
- [ ] End-to-end tests pass: full fill updates cash and position; partial fill in two executions accumulates; reject releases reserved cash; duplicate `client_order_id` returns the same order; duplicate execution report is ignored; one `OrderUpdated` per state change
- [ ] Integration job in CI runs these tests
