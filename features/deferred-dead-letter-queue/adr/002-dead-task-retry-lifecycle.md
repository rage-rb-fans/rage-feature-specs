---
status: proposed
---

# ADR-002: Dead-task retry lifecycle

## Context

A dead-task record contains the original serialized execution context and final failure metadata. Retrying must choose whether to reuse that internal context or represent a fresh public enqueue. It must also order the new pending WAL write and dead-record removal without a transaction across the two files.

Issue #369 explicitly deferred periodic adoption of WAL files created out of band. A Rake or console process can therefore persist work that a running worker will not discover until restart. The issue discussion also requested that `Queue#schedule` do nothing unless Iodine is running.

## Options considered

### Option A: Remove the dead record, then enqueue

This prevents two visible copies, but an enqueue failure or crash permanently loses the only recoverable copy. It violates the feature's at-least-once priority.

### Option B: Replay the serialized internal context, then remove

Calling the queue directly can preserve logger and user context and reuse internal values, but bypasses the public enqueue API, enqueue middleware, and enqueue telemetry. It couples the public feature to the private context layout.

### Option C: Perform a fresh public enqueue, confirm persistence, then remove

Resolve the stored task class and decode args/kwargs. The collection calls normal public `enqueue`. This creates a new task ID and context, resets attempts, and runs current middleware and telemetry. The collection calls optional backend `flush` before deleting the dead record. Disk requires explicit WAL synchronization because normal enqueue uses timer-based synchronization.

A deletion failure can leave both copies. Concurrent operators can enqueue duplicates.

### Option D: Add orphan-WAL adoption before exposing retry

Workers periodically discover WALs created by Rake or console processes. This improves immediacy but introduces a separate scheduler/storage ownership feature that the issue discussion explicitly deferred.

## Decision

Use Option C with collection-owned enqueue, optional backend flush, and removal. Continue to defer Option D. This ADR remains proposed.

Retry is a fresh enqueue of the original positional and keyword arguments under the current application code and configuration. It does not preserve the original logger/user context, task ID, attempt count, delay, or exception object. The collection removes the dead record only after enqueue and persistence confirmation succeed.

`DeadTasks#retry` owns exact-ID lookup, class validation, original argument decoding, enqueue, optional flush, and removal. The collection calls `task_class.enqueue(*args, **kwargs)`. The collection then calls `@backend.flush` only when `@backend.respond_to?(:flush)`. After persistence confirmation, the collection calls `@backend.remove_dead_tasks(id)` and returns `true`. A zero removal count also returns `true`.

Disk exposes optional `flush`, which delegates to internal `TasksStorage#sync!`. An adapter may omit `flush` only when successful enqueue already guarantees durable, recoverable pending work. A buffered adapter must provide `flush`. The capability check alone does not prove persistence. Adapter-conformance coverage must verify the persistence guarantee.

Do not require `flush` on every backend. Do not add no-op flush methods. Nil returns `false` before enqueue or flush. The design adds no backend retry orchestrator, application callback block, retry flag, enqueue-context hook, or shared synchronization parameter.

Disk's synchronization confirms relevant WAL file and directory durability across rotation and recovery. Disk retains required descriptors and source WAL data until replacement durability is confirmed. A timer delay alone does not establish durability. The timer continues ordinary batching and retains unsynchronized writes after failure. Explicit synchronization must not erase newer dirty state.

For the current storage version, retry obtains those arguments through task 03's lazy frozen Marshal decode and `Rage::Deferred::Context.get_args`/`.get_kwargs`. Because Rage is the sole writer of this version's opaque contexts, the MVP does not add duplicate Array shape, minimum-length, or extracted Array/Hash type validation. Actual Marshal or accessor exceptions use task 03's dedicated ID-and-cause error and leave the dead record intact. A future incompatible context layout requires a storage-version bump and compatibility guardrails.

`Queue#schedule` first checks `Iodine.running?` and performs no scheduling work when false. Retry outside a running server writes and synchronizes the pending WAL before dead-record removal. Retry emits the restart-required warning after persistence succeeds. Periodic orphan-WAL adoption belongs to a future feature.

## Consequences

- Enqueue middleware and telemetry can observe and reject an operator retry using current code.
- Class renames, removed task classes, and undecodable arguments prevent retry but not metadata inspection.
- Retry has at-least-once failure behavior. Execution can start before explicit synchronization completes.
- An optional flush failure propagates and preserves the dead record. The pending outcome remains uncertain. Repeating retry can duplicate work.
- A crash or deletion failure after confirmed persistence can leave both records. A deletion failure raises the retry-specific error with its original cause.
- Normal enqueue keeps its signatures, middleware, telemetry, and timer batching. Only Disk retry requests explicit WAL synchronization.
- Middleware can return without yielding. Public `Task.enqueue` still returns `nil`, so the selected sequence cannot detect that case. Task 05 records the unresolved outcome. Interrupted WAL tails and replay argument mutability also remain unresolved in task 05.
- Concurrent retry/delete operations require operator coordination and do not provide cancellation or exactly-once guarantees.
- Out-of-process retry is durable but not immediate. Operators must restart the Rage server before the orphaned WAL is adopted.
- The public API depends on the private serialized context layout only through `Rage::Deferred::Context` accessors used by the dedicated args/kwargs decoder; proactive compatibility validation is deferred to a future storage-version change.
