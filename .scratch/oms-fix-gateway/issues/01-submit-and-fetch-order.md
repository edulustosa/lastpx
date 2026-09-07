# 01: Submit and fetch an order over gRPC

**What to build:** A trader runs `docker compose up`, points a gRPC client at the OMS, submits a limit buy for a seeded account and symbol, gets back an order snapshot in `PENDING_NEW`, and can fetch it again by id. Submitting again with the same `client_order_id` returns the same order instead of creating a second one. This is the first tracer bullet, so it carries the whole skeleton: the buf module with the `lastpx.oms.v1` service, goose migrations for accounts, symbols and orders plus the seed data, sqlc queries over pgx, the Makefile `generate` target, the `oms` binary with env-var config and graceful shutdown, and the compose file with Postgres. No risk yet: every well-formed order is accepted.

**Blocked by:** None (can start immediately).

**Status:** ready-for-agent

- [ ] `make generate` regenerates proto and sqlc code; generated code is committed
- [ ] `docker compose up` starts Postgres, runs migrations and seeds ten symbols and five accounts
- [ ] `SubmitOrder` validates fields (returns `InvalidArgument` on bad input) and persists the order with a UUID id, status `PENDING_NEW`
- [ ] `GetOrder` returns the snapshot or `NotFound`
- [ ] Same `(account_id, client_order_id)` submitted twice returns the first order, enforced by a unique constraint
- [ ] Money and quantity use `shopspring/decimal` mapped to `NUMERIC`; ADR written for that decision
- [ ] `go build ./...` and `go vet ./...` pass; README documents how to run and call the API
