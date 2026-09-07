# 02: CI pipeline

**What to build:** Every push and pull request runs GitHub Actions: `go vet`, `golangci-lint`, `go test ./...`, and a check that `make generate` produces no diff against the committed generated code. A second job with Docker available runs tests tagged `integration` (empty for now, so later tickets inherit a green place to put them). Lint config lives in the repo so local and CI runs agree.

**Blocked by:** 01 (Submit and fetch an order over gRPC).

**Status:** ready-for-agent

- [ ] Workflow runs on push and pull request to main
- [ ] Jobs: vet + lint + unit tests; generate-check that fails on diff; integration job with build tag `integration`
- [ ] golangci-lint config committed with a sensible default linter set
- [ ] Badge in README
