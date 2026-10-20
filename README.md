<p align="center">
  <img src="docs/assets/moxy-hero.svg" alt="Moxy leased queue diagram" width="100%">
</p>

<p align="center">
  <a href="https://github.com/tempoloss/moxy/actions/workflows/go.yml"><img alt="Go" src="https://github.com/tempoloss/moxy/actions/workflows/go.yml/badge.svg"></a>
  <a href="https://goreportcard.com/report/github.com/tempoloss/moxy"><img alt="Go Report" src="https://goreportcard.com/badge/github.com/tempoloss/moxy"></a>
  <a href="LICENSE"><img alt="License" src="https://img.shields.io/github/license/tempoloss/moxy"></a>
  <img alt="Status" src="https://img.shields.io/badge/status-pre--proxy%20core-21d4a8">
</p>

# Moxy

Leases, ACKs and visibility timeouts for Redis-style queues, so a worker crash
is modeled explicitly instead of hidden inside a plain pop.

```text
READY -> PROCESSING -> ACKED
READY -> PROCESSING -> EXPIRED -> REQUEUED -> READY
```

Moxy is `v0.1.0-alpha`. It is good for learning, experiments and small
lease-model prototypes. It is not production-hardened.

## Install

```sh
git clone https://github.com/tempoloss/moxy
cd moxy
go build ./...
```

## Usage

Start the server (memory backend, or `--backend redis --redis-addr localhost:6379`):

```sh
go run ./cmd/moxy --addr 127.0.0.1:6380 --backend memory
```

Talk to it with any RESP client:

```sh
redis-cli -p 6380 MOXY.ENQUEUE jobs hello
redis-cli -p 6380 MOXY.FETCH jobs 30000     # returns a task and a lease_id
redis-cli -p 6380 MOXY.ACK <lease_id>
```

If the worker never ACKs, the task goes back to ready once the 30 s lease
expires. To watch that happen: `go run ./cmd/moxy-recovery-demo`.

Only `PING` and `MOXY.*` commands are served. This is not a Redis proxy yet.

## Design decisions

- **Leases instead of pop.** `LPOP` deletes the task before the worker finishes.
  A fetch hands out a lease with a deadline; ACK completes it, expiry requeues it.
- **Storage and ownership are separate.** Backends own task placement; the core
  engine owns lease metadata and the expiration heap.
- **Lease-fenced Redis scripts.** Acquire, complete, requeue and dead-letter are
  Lua scripts that compare the stored lease ID before touching either hash.
- **Leases survive a restart.** Every transition goes to a CRC-framed WAL. Replay
  stops at the first torn record, and recovered leases keep their original deadline.
- **Crash windows are enumerated.** Fetch journals before publishing the lease;
  Ack hits the backend before the journal. Each boundary is listed in the
  [failure matrix](docs/failure-matrix.md).
- **Orphans are reconciled at startup.** A processing task with no recovered
  lease goes back to ready without being charged an attempt.
- **At-least-once, not exactly-once.** A worker past its deadline can be joined by
  a second one, so handlers should be idempotent.
- **Not covered.** Ready-queue durability belongs to the backend, and there is no
  load test or multi-node evidence yet.

## Docs

- [ARCHITECTURE.md](ARCHITECTURE.md): package map, data flow, reliability guarantees
- [docs/lease-durability.md](docs/lease-durability.md): the problem, WAL, write ordering, limits
- [docs/failure-matrix.md](docs/failure-matrix.md): where a crash leaves a task, per boundary
- [docs/usage.md](docs/usage.md): commands, backends, server, tests, dev checks
- [docs/status.md](docs/status.md): what works today, non-goals, roadmap
- [docs/redis-production-caveats.md](docs/redis-production-caveats.md): read before using Redis
- [docs/adr/](docs/adr): decision records

## License

[MIT](LICENSE).
