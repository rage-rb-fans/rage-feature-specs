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

Resolve the stored task class and decode args/kwargs. The entry calls normal public `enqueue`. This creates a new task ID and context, resets attempts, and runs current middleware and telemetry. The entry calls optional backend `flush` before deleting the dead record. Disk requires explicit WAL synchronization because normal enqueue uses timer-based synchronization.

A deletion failure can leave both copies. Concurrent operators can enqueue duplicates.

### Option D: Add orphan-WAL adoption before exposing retry

Workers periodically discover WALs created by Rake or console processes. This improves immediacy but introduces a separate scheduler/storage ownership feature that the issue discussion explicitly deferred.

## Decision

Use Option C with entry-owned enqueue, optional backend flush, and removal. Continue to defer Option D. This ADR remains proposed.

Retry is a fresh enqueue of the original positional and keyword arguments under the current application code and configuration. It does not preserve the original logger/user context, task ID, attempt count, delay, or exception object. The entry removes the dead record only after enqueue and persistence confirmation succeed.

`DeadTask#retry` owns exact-ID refresh, class validation, original argument decoding, enqueue, optional flush, and removal. The entry calls `task_class.enqueue(*args, **kwargs)`. If enqueue returns `false`, retry retains the dead record and returns `false`. After a `true` return, the entry calls `@backend.flush` only when `@backend.respond_to?(:flush)`. After persistence confirmation, the entry calls `@backend.remove_dead_tasks(@id)` and returns `true`. A zero removal count also returns `true`.

`DeadTask` privately retains the identical shared backend supplied by its collection. The entry refreshes its exact ID through backend lookup before validating the current class and decoding the current arguments. Refresh preserves the original entry metadata and inspection cache. Entry retry needs no collection, action handler, service, or traversal resources.

`DeadTasks#retry(id)` remains a thin `find_by_id` followed by `entry.retry`. A missing lookup returns `false`. The entry performs its own refresh before enqueue. No second retry implementation exists.

Disk exposes optional `flush`, which delegates to internal `TasksStorage#sync!`. An adapter may omit `flush` only when successful enqueue already guarantees durable, recoverable pending work. A buffered adapter must provide `flush`. The capability check alone does not prove persistence. Adapter-conformance coverage must verify the persistence guarantee.

Do not require `flush` on every backend. Do not add no-op flush methods. Dead-task retry with Nil returns `false` before enqueue or flush. The design adds no backend retry orchestrator, application callback block, retry flag, enqueue-context hook, or shared synchronization parameter.

Disk's synchronization confirms relevant WAL file and directory durability across rotation and recovery. Disk retains required descriptors and source WAL data until replacement durability is confirmed. A timer delay alone does not establish durability. The timer continues ordinary batching and retains unsynchronized writes after failure. Explicit synchronization must not erase newer dirty state.

### Enqueue acceptance

Public `Task.enqueue` returns a Boolean instead of unconditional `nil`. It returns `true` only when its intended `Queue#enqueue` call completes normally. It returns `false` when middleware prevents that call. This contract applies to ordinary enqueue with both Disk and Nil. A storage ID is unnecessary for acceptance.

Use a per-call local initialized to `false`. Set the local to `true` immediately after the intended queue call returns inside the existing middleware block. Return the local after middleware and telemetry complete. Ignore their arbitrary return values. Nested or unrelated enqueues cannot establish acceptance for the outer call. Do not add shared state, a receipt object, or a retry flag.

If middleware yields multiple times, acceptance means at least one intended queue call completed and no exception escapes. Existing multiple-enqueue effects remain unchanged. The Boolean does not promise exactly-once enqueue.

Exceptions that escape middleware or telemetry still propagate. A post-acceptance exception leaves the dead record intact even though pending work can exist. A repeated retry can duplicate that work. Suppressed retry performs no flush, removal, or restart-required warning.

The Boolean changes a public return value. Callers that expect `nil` must adapt. Public signatures, argument handling, middleware, telemetry, and ordinary timer batching remain unchanged. All enqueue calls incur local acceptance bookkeeping.

### Mutable replay arguments

Retry loads one independent context graph from the refreshed serialized bytes with ordinary `Marshal.load`, without `freeze: true`. Retry uses existing `Rage::Deferred::Context.get_args`/`.get_kwargs` accessors. Accessor-returned `nil` values normalize to new mutable empty collections. This graph is separate from public frozen inspection values and their cache.

One decode preserves Marshal-supported shared references and cycles within and across the argument sets. Normal public enqueue splats can create new outer argument containers. Their identities are not preserved. Do not shallow-copy the inspection graph or serialize that decoded graph to construct replay values.

Live retry permits the same nested mutations as ordinary pending-WAL recovery. Ordinary Marshal behavior still applies, including custom deserialization behavior that can return frozen objects. Marshal or accessor failures use the dedicated context-deserialization error with the task ID and original cause. The dead record remains intact. No new context shape or extracted-value type validation is introduced.

### Pending-WAL prerequisite

[Task 07](../tasks/07-recoverable-pending-wal-appends.md) repairs the existing pending-WAL record-boundary defect before task 05. It covers adopted unfinished tails, recovered-source boundaries, failed appends, and safe source cleanup. Task 07 applies to ordinary enqueue and recovery without depending on retry's acceptance or flush APIs. Task 05 then adds explicit Disk synchronization over those guarantees. This separates pending-WAL repair from the dead-task retry operation.

`Queue#schedule` first checks `Iodine.running?` and performs no scheduling work when false. Accepted retry outside a running server writes and synchronizes the pending WAL before dead-record removal. Retry emits the restart-required warning after accepted persistence succeeds. Periodic orphan-WAL adoption belongs to a future feature.

## Consequences

- Enqueue middleware and telemetry can observe and reject an operator retry using current code.
- Class renames, removed task classes, and undecodable arguments prevent retry but not metadata inspection.
- Retry has at-least-once failure behavior. Execution can start before explicit synchronization completes.
- An optional flush failure propagates and preserves the dead record. The pending outcome remains uncertain. Repeating retry can duplicate work.
- A crash or deletion failure after confirmed persistence can leave both records. A deletion failure raises the retry-specific error with its original cause.
- Normal enqueue keeps its signatures, middleware, telemetry, and timer batching. Its return changes from `nil` to the acceptance Boolean. Only Disk retry requests explicit WAL synchronization.
- Middleware suppression retains the dead record and returns `false`. Independent replay decoding preserves immutable inspection. The pending-WAL prerequisite protects later appends from unfinished record tails.
- Concurrent retry/delete operations require operator coordination and do not provide cancellation or exactly-once guarantees.
- Out-of-process retry is durable but not immediate. Operators must restart the Rage server before the orphaned WAL is adopted.
- The public API depends on the private serialized context layout only through `Rage::Deferred::Context` accessors used by the dedicated args/kwargs decoder; proactive compatibility validation is deferred to a future storage-version change.
