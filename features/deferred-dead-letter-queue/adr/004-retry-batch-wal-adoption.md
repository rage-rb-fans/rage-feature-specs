---
status: accepted
---

# ADR-004: Coordinated retry and pending-WAL adoption

## Context

[Task 08](../tasks/08-batch-retry-dead-tasks.md) uses normal public enqueue, one optional flush, and one bulk dead-record removal.
[ADR-003](003-batch-dead-task-retry.md) accepts concurrent retry races and the absence of a transaction between pending and dead storage.
Pending work can execute before the invocation completes.
Periodic orphan-WAL discovery remains outside the current tasks.

The user selects a future orchestrator that discovers orphan pending WALs and merges work into active worker WALs.
An earlier decision selected a dedicated batch WAL. The user subsequently rejected dedicated retry WALs for both individual and bulk retry: retain normal public enqueue, existing pending WALs, and current scheduling without rewriting `.enqueue`.
Current pending WALs have worker ownership locks.
The orchestrator needs an ownership protocol before changing active worker storage.

The retry coordination lock now belongs in task 08, which remains `draft`. Stronger WAL publication/durability protocol and orphan adoption remain separate future specification work, without a new task dependency assumed here.
[Task 07](../tasks/07-recoverable-pending-wal-appends.md) retains its separate boundary-repair and recovery-cleanup scope.
This ADR adds no adoption prerequisite to either task.

## Options considered

### Option A: Keep the current retry path

The current path has no batch-publication or orchestrator dependency.
Operators coordinate concurrent retries and accept the documented duplication windows.

### Option B: Coordinate retries using the existing pending WAL

A retry writer holds a separate process-shared coordination lock from selection through normal public enqueue, optional explicit `flush_tasks`, and source removal.
The user accepts duplicates after pending persistence/publication but before source removal.
Future adoption can discover ordinary orphan WALs through a separately agreed ownership protocol; it is not an execution barrier for in-process retry.

### Option C: Add a transaction across current stores

A transaction protocol could coordinate pending creation and dead-record removal directly.
The user accepts the non-transactional persistence-to-removal window and keeps future orphan adoption separate.

## Decision

Accept the revised Option B for task 08. The separate retry lock is included in this task; the earlier dedicated-WAL, destination-routing, and adoption-before-execution direction is superseded.
Both retry surfaces use existing pending WALs and normal scheduling through unchanged public enqueue. Future WAL publication/durability and orphan-adoption protocols remain separately specified work.
The collection must not know about locking, call a lock/coordination-scope API, or manage a coordination block.
The internal owner of the full invocation-spanning lifecycle is open under task-08 Q-07. Earlier collection-orchestration ownership and service/backend-orchestrator prohibitions need a scoped decision; no replacement owner or private interface is approved here.
The user supersedes the earlier durable-write-after-error removal decision with Q-04's simpler retain-on-error policy: only normal public enqueue returns feed the final optional flush and bulk removal. Any enqueue exception retains its source and original per-ID error, including after a pending write or during scheduling; later IDs continue.
Add no private acknowledgement, receipts, or write-candidate tracking. No normal returns means no explicit flush/removal; failed flush remains a shared failure that removes no requested source. Q-05 through Q-07 remain open, and readiness/implementation are not authorized.

The intended batch writer sequence is:

1. Acquire the DLQ store's process-shared retry coordination lock.
2. Select source dead records after acquiring the coordination lock.
3. Run normal public enqueue, middleware, and telemetry with fresh replay contexts into the existing pending WAL, preserving current scheduling.
4. Call optional `flush_tasks` once after per-ID attempts when eligible IDs exist.
5. Remove only normally completed enqueue IDs in one bulk call after that explicit persistence barrier. Retain every source whose enqueue raised, regardless of prior pending writes.
6. Release the coordination lock through `ensure`.

Use a separate stable retry-lock file alongside the existing dead-store lock file; both use `flock`, without mutexes.
Acquire the retry lock before briefly acquiring the dead-store lock as needed. Ordinary DLQ operations use only the existing dead-store lock.
For example, `{prefix}dead_tasks_retry.lock` can sit alongside `{prefix}dead_tasks.lock`; the retry filename is illustrative.
The coordination lock spans enqueue middleware and application code.
The existing dead-store lock does not span application code.
Individual and batch retry paths for the same DLQ store use the same coordination lock.
Individual retry also holds the lock from source selection through durable pending persistence and source removal.

The backend path and prefix define the DLQ store's retry lock scope.
Independent DLQ stores use independent retry locks.
Normal enqueue, task execution, failure recording, and inspection do not acquire the retry coordination lock.
Explicit deletion keeps its existing dead-store lock and does not cancel replay that a retry already selected.

The lock needs process exclusion and same-process Fiber exclusion without blocking the Iodine worker on contention.
The proposed mechanism opens the same stable retry-lock file separately per invocation, obtaining distinct open file descriptions rather than `dup` or a shared descriptor, and uses `File::LOCK_EX | File::LOCK_NB` with Fiber-aware waiting.
Keep each descriptor through the locked sequence and release/close it through `ensure`. Do not unlink or replace the lock file while holders or contenders can exist.
The separate lock, retry-before-dead-store order, `flock`, and no-mutex choices are accepted; descriptor/wait details remain a proposed mechanism.
Timeout, cancellation, same-store backend instances, and nested retry need an explicit policy; a nested caller can deadlock while its parent holds the lock.

After successful source removal, a later cooperative retry cannot select those removed records.
The lock therefore prevents normal concurrent replay of the same source when the locked operation completes successfully.
A crash after pending publication/persistence but before source removal can leave both pending work and the source dead records.
Failed source removal or uncertain publication can also permit duplicate work on a later retry.
The user accepts these duplicate outcomes as an at-least-once tradeoff.

This direction does not require pre-retry source reconciliation, a source-dead-ID transfer journal, or cross-store deduplication.
Pending records still use ordinary fresh pending IDs, fresh serialized contexts, and existing WAL framing.
Adoption needs safe worker ownership, ordinary at-least-once pending-WAL recovery, and durable source-file cleanup.
Those storage requirements do not imply exactly-once source transfer or execution.

In-process retry keeps normal scheduling; additional tasks enqueued by middleware keep their ordinary queue/WAL destination.
An enqueue exception does not prove that no pending copy exists. After a scheduling failure, that copy may wait for startup/recovery, currently usually restart. The retained DLQ source allows an operator retry, accepting that recovery or a later retry can duplicate work.
Document this tradeoff in YARD and operator guidance. Do not automatically enqueue another copy or move pending work back into the DLQ because scheduling failed; scheduling recovery is separate work.
REQ-30 retains its prohibition on acknowledgement/acceptance bookkeeping and public enqueue changes. Silent-suppression compatibility remains unchanged.
Current orphan recovery does not provide this coordinated protocol.
The orchestrator cannot append to or replace a worker-owned WAL without an agreed ownership handoff.

## Consequences

- Cooperative locking prevents normal concurrent duplicate retries after a successful locked operation removes the source records.
- Crashes and uncertain publication or source removal can produce duplicate work. This direction accepts this behavior without transactional source reconciliation.
- This direction does not establish an atomic transaction across pending WALs and dead storage.
- Future adoption needs its own ownership, durability, discovery, and cleanup protocol; retry adds no orchestrator prerequisite or scheduling barrier.
- Long middleware calls can hold the retry coordination lock and delay other retry requests for the same DLQ store.
- Pending WALs remain trusted local application data. Future publication/adoption work needs compatible framing, retention, and cleanup rules.
- Silent enqueue suppression retains the existing task-loss risk. This direction does not promise unconditional at-least-once delivery.
- This direction does not guarantee exactly-once task execution or exactly-once external effects.
- Task 08 needs lock/Fiber/failure coverage and operator guidance. Separate future WAL/adoption work needs its own protocol and verification before implementation.

## Open questions

- Q-01: Define any stronger existing-WAL publication boundary and complete-write/durability acknowledgement. Decides: the user during separate future WAL drafting; no dedicated retry WAL is used.
- Q-02: Assign discovery, worker ownership handoff, merge, scheduling, and cleanup responsibilities.
  Define startup recovery and multiworker adoption under the coordination protocol.
  Decides: the user during future task drafting.
- Q-03: Define ordinary at-least-once pending-WAL recovery for interrupted publication and merge.
  Define when adoption can safely remove published source files after durable worker persistence.
  Decides: the user during future task drafting.
- Q-05: Define timeout/error, cancellation, same-store backend-instance handling, nested retry, and ordering with additional locks without Fiber deadlock. The retry lock precedes the short dead-store lock; deletion remains independent.
  Decides: the user during task 08 drafting.

- Q-07: Select an internal full-invocation lifecycle owner below the collection boundary, with any scoped amendment to the earlier orchestration/service prohibitions. The collection must remain unaware of locking; accepted lock scope and behavior remain unchanged. Decides: the user during task 08 drafting.
