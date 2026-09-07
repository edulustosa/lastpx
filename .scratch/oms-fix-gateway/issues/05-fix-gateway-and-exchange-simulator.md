# 05: FIX gateway and exchange simulator

**What to build:** An accepted order leaves the system over FIX and comes back as execution reports. The `fix-gateway` binary consumes `order.accepted`, keeps one FIX 4.4 initiator session (quickfixgo, file store, `ResetOnLogon=N`, heartbeat 30s) to the `exchange-sim` binary, and sends a NewOrderSingle with ClOrdID equal to the order id. The simulator acks, then fully fills, partially fills in two executions, or rejects, chosen by configurable weights, after a random latency in a configurable range, at reference price plus noise, deterministic under a fixed seed. Every ExecutionReport the gateway receives is converted to protobuf and published to the `executions` exchange. While the session is logged out the gateway stops pulling from its queue so messages wait in RabbitMQ; a redelivered order whose ClOrdID was already sent is acked and ignored. Both binaries join the compose file.

**Blocked by:** 04 (Transactional outbox publishing accepted orders).

**Status:** ready-for-agent

- [ ] Gateway sends 35=D (ClOrdID, Symbol, Side, OrdType, OrderQty, Price for limit, TimeInForce=Day, TransactTime) for each `order.accepted`
- [ ] Session config rendered from env vars; file store and file log under a data directory ignored by git
- [ ] Gateway consumer pauses while logged out and resumes on logon; restart of either side resumes with sequence numbers intact
- [ ] Duplicate `order.accepted` for an already-sent ClOrdID is not resent
- [ ] Each 35=8 is published to `executions` with order id, exec id, exec type, ord status, last qty, last px, cum qty, leaves qty, avg px, text, transact time
- [ ] Simulator honors `SIM_FILL_PCT`, `SIM_PARTIAL_PCT`, `SIM_REJECT_PCT`, `SIM_LATENCY_MIN`, `SIM_LATENCY_MAX`, `SIM_PRICE_NOISE_PCT`, `SIM_REJECT_MARKET`, `SIM_SEED`
- [ ] Simulator sends ack (ExecType=New) before fills; partial fills are two executions with correct CumQty and LeavesQty
- [ ] Running compose and submitting an order shows the matching execution reports in the `executions` queue
- [ ] ADR written for FIX 4.4 with file store and no sequence reset
