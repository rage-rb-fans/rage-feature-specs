---
status: accepted
---

# ADR-004: Future retry batch WAL publication and adoption

## Context

[Task 08](../tasks/08-batch-retry-dead-tasks.md) uses normal public enqueue, one optional flush, and one bulk dead-record removal.
[ADR-003](003-batch-dead-task-retry.md) accepts concurrent retry races and the absence of a transaction between pending and dead storage.
Pending work can execute before the invocation completes.
Periodic orphan-WAL discovery remains outside the current tasks.

The user selects a future orchestrator that discovers orphan pending WALs and merges work into active worker WALs.
Each bulk retry would publish a dedicated batch WAL for that orchestrator.
Current pending WALs have worker ownership locks.
The orchestrator needs an ownership protocol before changing active worker storage.

This ADR records an accepted future direction. The remaining protocol questions need a separate implementation specification.
Task 08's approved requirements, criteria, status, and current behavior remain unchanged.
[Task 07](../tasks/07-recoverable-pending-wal-appends.md) retains its separate boundary-repair and recovery-cleanup scope.
This ADR adds no adoption prerequisite to either task.

## Options considered

### Option A: Keep the current retry path

The current path has no batch-publication or orchestrator dependency.
Operators coordinate concurrent retries and accept the documented duplication windows.

### Option B: Coordinate retries and publish a dedicated batch WAL

A retry writer holds a separate process-shared coordination lock while preparing, publishing, and removing source dead records.
The orchestrator later adopts published work through coordinated worker ownership.
The user accepts the crash window between durable publication and source removal.
Publication, locking, and adoption need a separate protocol design.

### Option C: Add a transaction across current stores

A transaction protocol could coordinate pending creation and dead-record removal directly.
The user prefers to develop the batch-WAL direction with future orphan adoption.

## Decision

Accept Option B as the future direction. The user approved this direction on 2026-10-18.
This approval does not close the open protocol questions or authorize implementation.

The intended batch writer sequence is:

1. Acquire the DLQ store's process-shared retry coordination lock.
2. Select source dead records after acquiring the coordination lock.
3. Run normal enqueue middleware and telemetry with fresh replay contexts, routing resulting pending writes to a dedicated batch WAL.
4. Synchronize the batch data and required publication metadata to publish the batch durably.
5. Remove the exact source dead IDs after durable publication.
6. Release the coordination lock through `ensure`.

The coordination lock spans enqueue middleware and application code.
The existing dead-store lock does not span application code.
Individual and batch retry paths for the same DLQ store use the same coordination lock.
Individual retry also holds the lock from source selection through durable pending persistence and source removal.

The backend path and prefix define the DLQ store's retry lock scope.
Independent DLQ stores use independent retry locks.
Normal enqueue, task execution, failure recording, and inspection do not acquire the retry coordination lock.
Explicit deletion keeps its existing dead-store lock and does not cancel replay that a retry already selected.

The lock needs process exclusion and same-process Fiber exclusion.
An operating-system `flock` on one shared open file description does not provide a per-Fiber mutex.
Lock contention needs Fiber-aware waiting and an explicit nested-retry policy.

After successful source removal, a later cooperative retry cannot select those removed records.
The lock therefore prevents normal concurrent replay of the same source when the locked operation completes successfully.
A crash after publication but before source removal can leave both published pending work and the source dead records.
Failed source removal or uncertain publication can also permit duplicate work on a later retry.
The user accepts these duplicate outcomes as an at-least-once tradeoff.

This direction does not require pre-retry source reconciliation, a source-dead-ID transfer journal, or cross-store deduplication.
Published pending records still need ordinary pending IDs, fresh serialized contexts, and valid WAL framing.
Adoption needs safe worker ownership, ordinary at-least-once pending-WAL recovery, and durable source-file cleanup.
Those storage requirements do not imply exactly-once source transfer or execution.

The intended batch path makes work executable through adoption rather than immediate scheduling during public enqueue.
The routing and scheduling changes need a separate design that preserves normal enqueue behavior outside this path.
Current orphan recovery does not provide this coordinated protocol.
The orchestrator cannot append to or replace a worker-owned WAL without an agreed ownership handoff.

## Consequences

- Cooperative locking prevents normal concurrent duplicate retries after a successful locked operation removes the source records.
- Crashes and uncertain publication or source removal can produce duplicate work. This direction accepts this behavior without transactional source reconciliation.
- This direction does not establish an atomic transaction across the batch WAL and dead storage.
- Adoption adds execution latency and an orchestrator dependency to the future batch path.
- Long middleware calls can hold the retry coordination lock and delay other retry requests for the same DLQ store.
- Batch WALs remain trusted local application data. The future protocol needs compatible framing, retention, and cleanup rules.
- Silent enqueue suppression retains the existing task-loss risk. This direction does not promise unconditional at-least-once delivery.
- This direction does not guarantee exactly-once task execution or exactly-once external effects.
- A future task needs failure-injection coverage, Fiber-interleaving coverage, operator guidance, and compatibility criteria before implementation.

## Open questions

- Q-01: Define the batch format, publication boundary, and durability acknowledgement. Decides: the user during future task drafting.
- Q-02: Assign discovery, worker ownership handoff, merge, scheduling, and cleanup responsibilities.
  Define startup recovery and multiworker adoption under the coordination protocol.
  Decides: the user during future task drafting.
- Q-03: Define ordinary at-least-once pending-WAL recovery for interrupted publication and merge.
  Define when adoption can safely remove published source files after durable worker persistence.
  Decides: the user during future task drafting.
- Q-04: Define Fiber-local enqueue routing, ordinary nested enqueue behavior, suppression, multiple yields, and exceptions after pending writes.
  Define public retry results and restart warnings for deferred adoption.
  Decides: the user during future task drafting.
- Q-05: Define lock acquisition order, timeout, cancellation, and nested-retry behavior without Fiber deadlock.
  Define interactions with explicit deletion, same-ID replacement, and ordinary startup WAL recovery.
  Decides: the user during future task drafting.
