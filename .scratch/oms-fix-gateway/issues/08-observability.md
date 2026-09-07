# 08: Observability

**What to build:** An operator runs the `observability` compose profile, opens Grafana and sees order and FIX round-trip latency histograms, order counts by terminal status, rejections by reason, FIX session state and outbox backlog; opens Jaeger and follows one order as a single trace across gRPC, outbox, RabbitMQ, the gateway, FIX and back; queries Loki for one order id and gets every JSON log line from every service. Every binary exposes `/metrics`, exports OTLP traces to the collector, and logs JSON via `log/slog` with `service`, `trace_id`, `span_id`, and order, client order, account, symbol and exec ids when known. The core profile still runs alone on a small machine.

**Blocked by:** 07 (Cancel orders end to end).

**Status:** ready-for-agent

- [ ] Histograms: `oms_order_roundtrip_seconds` (submit → terminal), `fix_roundtrip_seconds` (35=D → first 35=8), gRPC server histograms via interceptor
- [ ] Counters: orders by terminal status, rejections by reason, execution reports by exec type, outbox published, dead-lettered messages
- [ ] Gauges: FIX session logged on, outbox unpublished count
- [ ] Spans: gRPC handler, risk, DB transaction, outbox publish, gateway consume, FIX send, FIX receive, execution apply; context propagated through AMQP headers so one order is one trace
- [ ] slog JSON handler wrapper injects trace and span ids from context
- [ ] Compose profile `observability`: Prometheus, Grafana with provisioned datasources and one dashboard, Jaeger, Loki, Promtail, OTel Collector
- [ ] README documents the profile and the URLs
