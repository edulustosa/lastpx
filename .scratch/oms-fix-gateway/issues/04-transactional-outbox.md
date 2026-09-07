# 04: Transactional outbox publishing accepted orders

**What to build:** When the OMS accepts an order, an `order.accepted` protobuf message reliably shows up on the RabbitMQ `orders` exchange, even if RabbitMQ was down at the moment of the submit. The event is written to an outbox table in the same transaction as the order; a relay goroutine publishes unpublished rows in order with publisher confirms and marks them published. The message carries a W3C `traceparent` header. RabbitMQ (with management UI) joins the compose file, and the `orders` and `executions` exchanges, the `events` fanout, and their durable queues with dead-letter exchanges are declared by the OMS at startup.

**Blocked by:** 03 (Pre-trade risk and account view).

**Status:** ready-for-agent

- [ ] Outbox table written in the accept transaction; relay publishes in id order with confirms and marks `published_at`
- [ ] Relay polls with short interval and backs off when empty; stops cleanly on shutdown
- [ ] Stopping RabbitMQ, submitting orders, and starting it again results in every accepted order being published exactly once, in order
- [ ] Topology (exchanges, queues, DLX, persistent messages) declared idempotently at startup
- [ ] `traceparent` propagated in AMQP headers
- [ ] ADRs written: transactional outbox over publish-after-commit; protobuf as the RabbitMQ body format
