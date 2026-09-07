# lastpx

Sell-side Order Management System (OMS) and FIX gateway in Go.

Client → gRPC → **oms** (pre-trade risk, Postgres, transactional outbox) → RabbitMQ → **fix-gateway** (FIX 4.4 via quickfixgo) → **exchange-sim** → execution reports back through the same path. A **trader** binary generates synthetic order flow.

Observability: Prometheus, OpenTelemetry traces, structured logs. A real-time web UI with an LLM narrating the event stream comes later.

## Layout

```
cmd/            one main per binary: oms, fix-gateway, exchange-sim, trader
internal/       application code, not importable from outside the module
proto/          protobuf definitions (gRPC API and RabbitMQ events), built with buf
migrations/     goose SQL migrations
deploy/         docker compose and observability configs
```

## Requirements

Go 1.27, Docker, buf, sqlc, goose.
