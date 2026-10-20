# Lease Durability

This page holds the motivation, the comparison with a plain Redis list
consumer, and the lease durability notes that used to live in the README.

## The Pain

Plain Redis list consumption often starts with `LPOP`. It is fast, simple, and
dangerous:

1. A worker pops a task.
2. The worker crashes before completing it.
3. Redis has already removed the task.
4. The task is gone.

Moxy replaces pop-and-pray delivery with leased delivery. A worker receives a
temporary lease; if it ACKs, the task is completed. If it vanishes, the task is
requeued after the lease expires.

## Standard Redis Lists vs Moxy

| Capability | Standard Redis List Consumer | Moxy |
| --- | --- | --- |
| Worker crash after fetch | Task can disappear | Task remains recoverable |
| Visibility timeout | Build it yourself | Native lease expiration |
| ACK semantics | Build it yourself | `MOXY.ACK lease_id` |
| Expired task recovery | Manual scripts or side tables | Built into the engine |
| Testable core logic | Usually buried in worker code | Dedicated core package |
| Backend choices | Redis list only | MemoryQueue and RedisQueue |
| Protocol status | Redis commands today | RESP command path for Moxy commands |
| Restart with leases held | Claims are lost, tasks stay stuck | WAL replay restores open leases; see Lease Durability |

## Lease Durability

A lease is a claim: this worker holds this task until this moment. That claim
used to live only in the engine's memory, so a restart lost it - and every task
already moved into processing storage was stranded, because the reaper walks the
in-memory expiration heap and cannot expire a lease it never saw.

Every transition is now journalled to an append-only file, and the journal is
replayed on boot:

```text
[uint32 length][uint32 CRC32][payload]
```

A crash can tear the final frame, so replay stops at the first record that is
short, fails its checksum, or does not decode, and truncates the file back to
that boundary. A half-written record is discarded rather than poisoning
recovery. Compaction rewrites the journal down to the still-open leases through
a temporary file and a rename, so an interrupted compaction leaves either the
old journal or the new one.

Recovered leases keep their **original** deadline. One that lapsed while the
process was down is reaped on the first pass instead of being handed a fresh
window it never earned.

Write ordering differs per operation, and the asymmetry is deliberate:

| Operation | Order | Why |
| --- | --- | --- |
| Fetch | generate lease ID, backend acquire with that lease ID, then journal before publishing the lease | The task has already left ready storage under a generation fence; a failed write hands that exact claim back rather than leaving it held by nobody |
| Ack | backend, then journal | A crash in the gap leaves a stale lease the next recovery, retry, or reap closes with `wal.OpStale`. The reverse order redelivers finished work: recovery folds the lease closed, startup reconciliation returns the still-processing task to ready, and no attempt is charged to bound it |

**What this does not cover.** The backend owns ready-queue durability, so a
Redis configured to lose writes still loses tasks. An append that has been
written but not yet fsynced promises nothing: recovery may see the record, a
torn record, or none, and only the surviving-record branch restores the original
deadline. And the guarantee is at-least-once, so a worker that runs past its
deadline can be joined by a second one holding the requeued task.

The WAL mechanism is implemented in `internal/wal` and exercised by
`internal/wal/wal_test.go` tests such as `TestRecordsSurviveReopen`,
`TestTornTailIsDroppedAndTruncated`, and `TestCorruptChecksumStopsReplay`.
Lease restoration is exercised in `internal/core/recovery_test.go` by
`TestLeaseSurvivesRestart` and `TestRestoredLeaseKeepsItsOriginalDeadline`.

Where a crash leaves a task is enumerated per boundary in
[`failure-matrix.md`](failure-matrix.md): for each window between the
backend operation, the journal append, the fsync, the lease publication and the
reply, it states where the task ends up after restart, whether it can be lost or
delivered twice, who reconciles it, and which test proves it.

Read it before relying on Moxy for anything. The window that used to be its named
weakness is now closed: a crash after the backend acquire but before any fetch
WAL bytes are durable (`F1`, and the torn-record case `F2`) left the task in
backend `processing` with no recovered task-plus-lease fence, so nothing reaped
it. Startup recovery now reconciles the backend against the task and lease IDs
the journal restored: a processing task without an exact recovered fence cannot
be held by anyone in the new process, so it returns to `ready` without being
charged an attempt. `RecoverOrphanedProcessing` does the move atomically inside
each backend, including a Lua script for Redis.

The guarantee across those windows is at-least-once, not exactly-once: a task can
be delivered twice if a recovered lease passes its preserved deadline while work
continues elsewhere. Reconciliation runs only during recovery, never
periodically, so it cannot requeue a task a live worker still holds under a valid
task-plus-lease fence. `internal/core/crash_matrix_test.go` and
`internal/core/recovery_test.go` cover recovered lease and stale-generation
boundaries.

## Limitations / Not Yet Proven

Moxy has no in-repo load test, benchmark, failover drill, uptime history, or
multi-node evidence. Treat the WAL and Redis Lua paths as implemented mechanisms,
not as proof of operational maturity, availability guarantees, lossless
delivery, or specific latency/throughput behavior.
