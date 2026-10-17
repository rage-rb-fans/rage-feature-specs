---
status: proposed
---

# ADR-002: Dead-task retry lifecycle

## Context

A dead-task record contains the original serialized execution context and final failure metadata. Retrying must choose whether to reuse that internal context or represent a fresh public enqueue. It must also order the new pending WAL write and dead-record removal without a transaction across the two files.

Issue #369 explicitly deferred periodic adoption of WAL files created out of band. A Rake or console process can therefore persist work that a running worker will not discover until restart. The issue discussion also requested that `Queue#schedule` do nothing unless Iodine is running.

## Options considered

### Option A: Remove the dead record, then enqueue

This prevents two visible copies, but an enqueue failure or crash permanently loses the only recoverable copy. It removes the source before the enqueue attempt.

### Option B: Replay the serialized internal context, then remove

Calling the queue directly can preserve logger and user context and reuse internal values, but bypasses the public enqueue API, enqueue middleware, and enqueue telemetry. It couples the public feature to the private context layout.

### Option C: Perform a fresh public enqueue, flush if available, then remove

Resolve the stored task class and decode args/kwargs. The entry calls normal public `enqueue`. Public enqueue builds a fresh context and runs current middleware and telemetry. When the call reaches the queue, enqueue creates a new task ID and resets attempts. The entry calls optional backend `flush` before deleting the dead record. Disk requires explicit WAL synchronization because normal enqueue uses timer-based synchronization.

A deletion failure can leave both copies. Concurrent operators can enqueue duplicates.

### Option D: Add orphan-WAL adoption before exposing retry

Workers periodically discover WALs created by Rake or console processes. This improves immediacy but introduces a separate scheduler/storage ownership feature that the issue discussion explicitly deferred.

## Decision

Use Option C with entry-owned enqueue, optional backend flush, and removal. Continue to defer Option D. This ADR remains proposed.

Retry is a fresh enqueue of the original positional and keyword arguments under the current application code and configuration. It does not preserve the original logger/user context, task ID, attempt count, delay, or exception object. The entry removes the dead record after public enqueue and optional flush return normally. This sequence does not prove queue acceptance.

`DeadTask#retry` owns an exact-ID existence check, class validation, original argument decoding, enqueue, optional flush, and removal. The entry calls `task_class.enqueue(*args, **kwargs)`. The entry ignores the enqueue return value. After enqueue returns normally, the entry calls `@backend.flush` only when `@backend.respond_to?(:flush)`. The entry then calls `@backend.remove_dead_tasks(@id)` and returns `true`. A zero removal count also returns `true`.

`DeadTask` privately retains the identical shared backend supplied by its collection. The entry calls `@backend.find_dead_task(@id)` only to check existence before enqueue. A missing record returns `false`; existing selected decoding and diagnostic rules still apply. Retry discards the returned record. Retry validates the entry's `@task_class` and decodes the entry's `@__context`.

Retry constructs no second entry. The original metadata and inspection cache remain unchanged. Entry retry needs no collection, action handler, service, or traversal resources.

`DeadTasks` has no `retry` method. ID-based callers use `find_by_id(id)` and call `entry.retry` when lookup returns an entry. The caller returns `false` when lookup returns `nil`. The entry checks existence before enqueue without replacing the first lookup's class or arguments. No second retry implementation exists.

Disk exposes optional `flush`, which delegates to internal `TasksStorage#sync!`. An adapter may omit `flush` when successful enqueue already completes its ordinary acknowledged persistence operation. A buffered adapter must provide `flush`. The capability check alone does not prove persistence. Adapter-conformance coverage must verify ordinary acknowledged persistence semantics.

Do not require `flush` on every backend. Do not add no-op flush methods. Dead-task retry with Nil returns `false` before enqueue or flush. The design adds no backend retry orchestrator, application callback block, retry flag, enqueue-context hook, or shared synchronization parameter.

Disk synchronizes the currently active WAL file, then the storage directory. Failed synchronization propagates and preserves the source dead record. Explicit synchronization leaves timer dirty state untouched. Existing timer batching remains unchanged.

Synchronization confirms current bytes and directory metadata under the existing storage lifecycle. It does not validate or repair framing or remove inherited tail, rotation, and recovery defects. Task 05 accepts those limits. A successful retry therefore does not establish complete crash-recovery safety. Retry adds no refusal checks for inherited defects.

### Deferred enqueue-acceptance detection

Keep public `Task.enqueue` unchanged. Preserve its unconditional `nil` return, signature, argument handling, middleware, telemetry, and multiple-yield behavior for Disk and Nil. Retry ignores the return value. Do not add acceptance bookkeeping, retry flags, receipt objects, or context hooks.

A normal enqueue return does not prove that the intended queue call occurred. Middleware or telemetry can silently suppress that call. Retry still performs optional flush, emits the applicable restart warning, removes the dead record, and returns `true`. No pending work need exist. This accepted limitation can lose the task and has no at-least-once guarantee.

Defer enqueue-acceptance detection and suppression retention to a future improvement. The retry Boolean reports completion of the enqueue/flush/removal sequence, not queue acceptance. This decision avoids a public enqueue change and ordinary enqueue bookkeeping.

Exceptions that escape middleware or telemetry still propagate and preserve the dead record. Pending work can already exist. A repeated retry can duplicate that work.

### Mutable replay arguments

Retry loads one independent context graph from the entry's serialized bytes with ordinary `Marshal.load`, without `freeze: true`. Retry uses existing `Rage::Deferred::Context.get_args`/`.get_kwargs` accessors. Accessor-returned `nil` values normalize to new mutable empty collections. This graph is separate from public frozen inspection values and their cache.

One decode preserves Marshal-supported shared references and cycles within and across the argument sets. Normal public enqueue splats can create new outer argument containers. Their identities are not preserved. Do not shallow-copy the inspection graph or serialize that decoded graph to construct replay values.

Live retry permits the same nested mutations as ordinary pending-WAL recovery. Ordinary Marshal behavior still applies, including custom deserialization behavior that can return frozen objects. Marshal or accessor failures use the dedicated context-deserialization error with the task ID and original cause. The dead record remains intact. No new context shape or extracted-value type validation is introduced.

### Deferred pending-WAL repairs

[Task 07](../tasks/07-recoverable-pending-wal-appends.md) remains independent future work. It does not precede or block task 05. Task 05 adds only the explicit active-WAL and directory synchronization required by retry. It does not change tail repair, short-write handling, rotation, recovery copying, source cleanup, or timer-failure policy. Existing WAL defects can still affect crash recovery after the dead record is removed. This is an accepted scope limitation, not an unresolved implementation gate.

`Queue#schedule` first checks `Iodine.running?` and performs no scheduling work when false. When enqueue reaches the queue outside Iodine, retry writes and synchronizes pending work without scheduling it. Retry emits the restart-required warning after normal enqueue and optional flush, including silent suppression. Periodic orphan-WAL adoption belongs to a future feature.

## Consequences

- Enqueue middleware and telemetry can observe and reject an operator retry using current code.
- Class renames, removed task classes, and undecodable arguments prevent retry but not metadata inspection.
- Pending work can execute before explicit synchronization completes. Silent suppression can delete the source without creating pending work.
- An optional flush failure propagates and preserves the dead record. The pending outcome remains uncertain. Repeating retry can duplicate work.
- A crash or deletion failure after pending persistence can leave both records. A deletion failure raises the retry-specific error with its original cause.
- Normal enqueue keeps its signature, unconditional `nil` return, middleware, telemetry, and timer batching. Only Disk retry requests explicit WAL synchronization.
- Silent middleware or telemetry suppression does not retain the dead record. Acceptance detection remains future work. Independent replay decoding preserves immutable inspection. Existing pending-WAL defects remain outside task 05.
- Concurrent retry/delete operations require operator coordination and do not provide cancellation or exactly-once guarantees. The existence check does not reserve the record or compare versions. Retry uses the entry's original class and arguments even after same-ID replacement. Later exact-ID removal can delete that replacement.
- Out-of-process work that reaches the queue requires server restart for adoption. Silent suppression can produce the same warning without pending work.
- The public API depends on the private serialized context layout only through `Rage::Deferred::Context` accessors used by the dedicated args/kwargs decoder; proactive compatibility validation is deferred to a future storage-version change.
