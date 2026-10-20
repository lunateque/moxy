# Status

This page holds the project status, scope and roadmap notes that used to live
in the README.

Moxy is an early-stage alpha project being built from the inside out: first the
correctness model, then the network/protocol layer, and only later transparent
Redis proxying and persistence. The current repo is already useful as a compact
Go reference for leased queue internals.

Moxy is `v0.1.0-alpha` software. It is useful for learning, experiments, and
small lease-model prototypes, but it is not production-hardened yet.

## What Works Today

- In-memory queue backend with `READY` and `PROCESSING` storage.
- Redis queue backend using `go-redis/v9`.
- Atomic Redis `Complete`, `Requeue`, and `DeadLetter` operations with lease-fenced Lua scripts.
- Single-queue lease coordinator in `internal/core`.
- Multi-queue service layer in `internal/service`.
- Protocol-neutral command handler in `internal/command`.
- RESP2 reader/writer in `internal/resp`.
- Redis-compatible TCP command server for Moxy commands and `PING`.
- Background expiration reaper.
- Shared backend contract tests for MemoryQueue and RedisQueue.
- Opt-in Redis integration tests.

## Non-Goals For This Alpha

Moxy is still single-node and backend-adapter based. These are intentionally not
implemented yet:

- Transparent Redis proxying
- Full Redis command pass-through
- snapshots of backend contents
- Redis Streams
- distributed coordination
- additional Redis-compatible commands beyond `PING` and `MOXY.*`

## Roadmap

- Keep hardening the command/service boundary.
- Add observability-friendly stats and structured errors.
- Add transparent Redis pass-through after the Moxy command path stays boring.
- Clear the stale lease on ack retry instead of surfacing `ErrTaskNotProcessing` and waiting for the reaper.
- Run the engine under `go test -race`, and simulate a crash during WAL compaction.

The boring part is the point: a leased queue coordinator should be legible,
testable, and conservative before it becomes networked.
