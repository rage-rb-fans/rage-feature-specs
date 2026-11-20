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
Q-07 selects an internal `Rage::Deferred::DeadTaskActions` collaborator for both individual and collection mutation actions. A factory selects backend-specific subclasses such as `DeadTaskActions::Disk` and `DeadTaskActions::Nil` using the identical existing backend. Public objects keep input validation, frozen inspection data, and read behavior, and delegate mutation workflows without lock or persistence-capability branches or an entry-to-collection reference.
The shallow parent shares genuinely common behavior: `delete_one(id)` calls `backend.remove_dead_tasks(id) == 1`; `delete_many(ids)` returns `0` for empty input or delegates one bulk removal. Public deletion validation and de-duplication remain unchanged. Deletion uses only the short dead-store lock and never joins the retry lock.
Backend-specific actions own separate `retry_one(entry)` and `retry_many(ids)` workflows. Individual retry checks existence but replays the original entry snapshot, preserving Boolean results, propagated errors, warning timing, and frozen inspection caches. Batch retry uses fresh multi-ID selection, per-ID errors, and shared-failure boundaries. Do not replace individual retry with a one-ID batch.
Disk actions request the invocation-spanning retry-lock scope from a Disk-specific helper, call Disk lookup/removal primitives, and call `flush_tasks` directly. Disk owns filesystem operations and cleanup, not application replay or result orchestration. Nil actions return individual `false` or an ordered all-`false` batch Hash with no enqueue, files, lock, flush, or background work. Do not impose a generic lock API or no-op lock/flush on every backend, or use `respond_to?(:flush_tasks)` in public objects or a shared retry workflow.
Actions retain only the backend; entry arguments, selected records, result/error Hashes, descriptors, and eligible-ID buffers remain invocation-local. Small internal entry helpers may provide class validation, mutable context decoding, and public enqueue without changing public inspection.
This scoped amendment supersedes collection-owned orchestration, the ban on a separate actions/service collaborator and entry-held actions, and the requirement that `DeadTask#retry` itself own the complete individual sequence. Backend-owned replay orchestration remains unnecessary: actions own the workflow and backend helpers own storage. Task 05 retains its historical implementation requirements and status; task 08 preserves public behavior while changing ownership.
The factory covers currently supported Disk and Nil configurations. Future adapters use corresponding actions subclasses, with the factory extension mechanism deferred; this adds no public registration API or new unsupported-backend error. Suggested internal files are `lib/rage/deferred/dead_task_actions.rb` and `lib/rage/deferred/dead_task_actions/{disk,nil}.rb` as detailed in task 08.
The user supersedes the earlier durable-write-after-error removal decision with Q-04's simpler retain-on-error policy: only normal public enqueue returns feed the final optional flush and bulk removal. Any enqueue exception retains its source and original per-ID error, including after a pending write or during scheduling; later IDs continue.
Add no private acknowledgement, receipts, or write-candidate tracking. No normal returns means no explicit flush/removal; failed flush remains a shared failure that removes no requested source. Q-04 through Q-07 are resolved; task 08 remains `draft`, and readiness/publication/implementation are not authorized.

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
The accepted mechanism opens the same stable retry-lock file separately per individual or batch invocation, obtaining distinct open file descriptions rather than `dup` or a shared descriptor, and uses `File::LOCK_EX | File::LOCK_NB` with Fiber-aware waiting.
Keep each descriptor through the locked sequence and release/close it through `ensure`. Do not unlink or replace the lock file while holders or contenders can exist.
Separate backend instances using the same configured path and prefix contend on the same stable lock file. Rake and application API callers use this same mechanism. No shared `@locked` flag, mutex, or backend-instance registry is needed for retry coordination. This adds no public multi-store API.

Q-05 uses a fixed 30-second retry-lock acquisition deadline. On expiry, raise the existing `Rage::Deferred::DeadTasksLockTimeout` with a message identifying the retry coordination lock, before selection or enqueue by that invocation. Add no timeout configuration or new exception class. This deadline does not limit execution after acquisition or the total invocation duration, and it does not change the short dead-store lock's existing policy.
Nested retry and other cyclic retry-lock waits use that same timeout. Add no Fiber-local nesting detection, special nested-retry error, reentrancy, or general cross-store ordering machinery. If a nested timeout escapes middleware, its caller uses the existing enqueue-error handling. The retry-before-dead-store order remains required.
Rage does not coordinate application-owned locks or promise deadlock-free application code. Callers remain responsible for application-lock cycles; this timeout bounds each retry-lock acquisition wait only. In Iodine, waits yield the Fiber; outside Iodine, ordinary waiting may pause the process.

An interruption that unwinds the invocation releases locks and descriptors and propagates the primary exception. Cleanup performs no extra WAL flush, rollback, or source removal, and adds no manual cancellation API. Ordinary per-ID `StandardError` handling remains unchanged. Preserve timer flushing and the usual final explicit flush when execution reaches it. Pending writes and removals already performed cannot be undone; interruption during removal can leave some sources removed. Cleanup does not mask an exception already in flight; a sole cleanup failure propagates.

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
Q-06 records this concise YARD/operator guidance: "An enqueue error leaves the original dead task available for retry. A pending copy may already exist, even though enqueue raised. If scheduling failed, that copy may require server restart to recover. Retrying the dead task can create duplicate work."
Keep the original per-ID exception and existing warning timing and count rules. Add no runtime warning for each enqueue error. Do not automatically enqueue another copy or move pending work back into the DLQ because scheduling failed; scheduling recovery is separate work.
REQ-30 retains its prohibition on acknowledgement/acceptance bookkeeping and public enqueue changes. Silent-suppression compatibility remains unchanged.
Current orphan recovery does not provide this coordinated protocol.
The orchestrator cannot append to or replace a worker-owned WAL without an agreed ownership handoff.

## Consequences

- Cooperative locking prevents normal concurrent duplicate retries after a successful locked operation removes the source records.
- Crashes and uncertain publication or source removal can produce duplicate work. This direction accepts this behavior without transactional source reconciliation.
- This direction does not establish an atomic transaction across pending WALs and dead storage.
- Future adoption needs its own ownership, durability, discovery, and cleanup protocol; retry adds no orchestrator prerequisite or scheduling barrier.
- Long middleware calls can hold the retry coordination lock and delay other retry requests for the same DLQ store.
- A contender, including an accidental nested retry, fails after 30 seconds of retry-lock acquisition waiting. Middleware and application locks can still delay the outer invocation; the timeout is not a total duration or general deadlock guarantee.
- Interruption cleanup releases resources without forcing additional persistence or removal. Pending copies and partial source removal remain possible.
- Pending WALs remain trusted local application data. Future publication/adoption work needs compatible framing, retention, and cleanup rules.
- Silent enqueue suppression retains the existing task-loss risk. This direction does not promise unconditional at-least-once delivery.
- This direction does not guarantee exactly-once task execution or exactly-once external effects.
- Task 08 needs lock/Fiber/failure coverage and operator guidance. Separate future WAL/adoption work needs its own protocol and verification before implementation.
- Public entries and collections share mutation action contracts while backend-specific actions keep Disk coordination and persistence out of public objects. Actions add no progress state or work to normal enqueue/execution paths.

## Open questions

- Q-01: Define any stronger existing-WAL publication boundary and complete-write/durability acknowledgement. Decides: the user during separate future WAL drafting; no dedicated retry WAL is used.
- Q-02: Assign discovery, worker ownership handoff, merge, scheduling, and cleanup responsibilities.
  Define startup recovery and multiworker adoption under the coordination protocol.
  Decides: the user during future task drafting.
- Q-03: Define ordinary at-least-once pending-WAL recovery for interrupted publication and merge.
  Define when adoption can safely remove published source files after durable worker persistence.
  Decides: the user during future task drafting.
