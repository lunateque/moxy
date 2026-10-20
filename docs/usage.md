# Usage

This page holds the commands, backends, server, demo, test and development
notes that used to live in the README. Run all commands from the repository
root.

## 15-Second Recovery Demo

Run the local recovery demo:

```sh
go run ./cmd/moxy-recovery-demo
```

It simulates a worker fetching a task, disappearing before ACK, and the reaper
putting that task back into ready storage.

To render a terminal GIF with [VHS](https://vhs.charm.sh/):

```sh
vhs demo/recovery.tape
```

The tape writes `docs/assets/recovery.gif`.

## Internal Commands

The command layer is protocol-neutral Go code. The TCP/RESP server translates
wire input into these commands and translates command responses back to RESP.

```text
MOXY.ENQUEUE queue payload
MOXY.FETCH queue timeout_ms
MOXY.ACK lease_id
MOXY.STATS queue
```

Example:

```go
svc := service.New(func(queueName string) queue.Backend {
	return queue.NewMemoryQueue()
})
handler := command.NewHandler(svc)

enqueue, _ := handler.Handle(command.Command{
	Name: command.EnqueueName,
	Args: []string{"emails", "send welcome email"},
})

fetch, _ := handler.Handle(command.Command{
	Name: command.FetchName,
	Args: []string{"emails", "30000"},
})

_, _ = handler.Handle(command.Command{
	Name: command.AckName,
	Args: []string{fetch.LeaseID},
})

_ = enqueue
```

## Backends

### MemoryQueue

`MemoryQueue` is the default backend used by `core.NewEngine` and the demo binary.
It is deterministic, easy to test, and useful for validating lease behavior.

### RedisQueue

`RedisQueue` keeps what is waiting in a list, what is claimed in one hash, and
the lease generation fence in a second hash:

- `moxy:{queue}:ready` - list; tasks are taken with `RPOP` and returned with `LPUSH`
- `moxy:{queue}:processing` - hash keyed by task id, storing the serialized task
- `moxy:{queue}:processing:leases` - hash keyed by task id, storing the current lease id
- `moxy:{queue}:dead` - list

`Acquire(leaseID)` runs one Lua script that does `RPOP` off the ready list and
`HSET` into both processing hashes, so no crash can land between the task move
and its fence. `Complete`, `Requeue`, `DeadLetter`, and startup reclamation are
Lua scripts as well. They address a task by id in the hash and compare the
stored lease ID before mutating either hash, which is the reason processing is
not a second list. `LMOVE` would fit list-to-list and is deliberately unused.

```go
client := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
backend := queue.NewRedisQueue(client, "emails")
```

## Quickstart

```sh
go test ./...
go run ./cmd/moxy --addr 127.0.0.1:6380 --backend memory
go run ./cmd/moxy-recovery-demo
```

## Running Moxy TCP Server

Memory backend:

```sh
go run ./cmd/moxy --addr 127.0.0.1:6380 --backend memory
```

Redis backend:

```sh
go run ./cmd/moxy --addr 127.0.0.1:6380 --backend redis --redis-addr localhost:6379
```

Testing with `redis-cli`:

```sh
redis-cli -p 6380 PING
redis-cli -p 6380 MOXY.ENQUEUE jobs hello
redis-cli -p 6380 MOXY.FETCH jobs 30000
redis-cli -p 6380 MOXY.ACK <lease_id>
redis-cli -p 6380 MOXY.STATS jobs
```

`MOXY.FETCH` returns a null bulk string when the named queue has no ready task.
That makes an empty queue a normal condition instead of a server failure.

This is not full Redis proxying yet. Only `PING` and `MOXY.*` commands are
supported. Normal Redis commands such as `GET`, `SET`, and `INCR` are not passed
through to Redis in this phase.

## Redis Integration Tests

Redis tests are opt-in so normal development does not require a running Redis
server.

PowerShell:

```powershell
$env:MOXY_REDIS_INTEGRATION='1'
$env:MOXY_REDIS_ADDR='localhost:6379'
go test ./internal/queue -run Redis -count=1 -v
go test ./internal/core -run Redis -count=1 -v
```

Shell:

```sh
MOXY_REDIS_INTEGRATION=1 MOXY_REDIS_ADDR=localhost:6379 \
  go test ./internal/queue -run Redis -count=1 -v
```

## Development Checks

```sh
go mod tidy
go test ./...
go vet ./...
go test ./internal/core -count=100
go test ./internal/queue -count=100
go test ./internal/service -count=100
go test ./internal/command -count=100
go test ./internal/resp -count=100
go test ./internal/protocol -count=100
go test ./internal/server -count=100
```

Race testing is deferred until the local Windows development environment has
cgo/GCC available.
