---
status: accepted
---

# ADR-003: Batch dead-task retry without a transaction

## Context

Task 05 supports immediate retry of one entry.
On Disk, each entry retry can scan the dead store, explicitly synchronize pending storage, and compact the dead store.
Operators now need retry for multiple exact IDs with fewer storage operations.
The user accepts the current absence of a transaction between pending and dead storage.

ADR-001 and ADR-002 describe the entry-only stage.
Task 08 adds a collection operation while preserving individual entry retry.
Task 07's pending-WAL repairs and enqueue-acceptance detection remain deferred.

## Options considered

### Option A: Repeat entry retry

The collection calls `find_by_id` and `entry.retry` for each ID.
This option adds convenience but keeps repeated complete scans, explicit flushes, and compactions.

### Option B: Select, enqueue, flush, and remove in one batch

The collection selects all requested IDs through one backend lookup.
The collection calls normal public enqueue for each selected ID.
The collection then performs one optional flush and one bulk removal for eligible IDs.
Per-ID errors can leave other IDs eligible.
Shared storage failures stop the invocation.

### Option C: Add an atomic storage transfer

A shared transaction log could commit pending creation and dead-record removal together.
This option needs storage recovery, idempotency, and scheduling changes.
The user considers that work excessive for the current operator API.

## Decision

Use Option B for task 08. The user approved this decision on 2026-10-18.
[ADR-004](004-retry-batch-wal-adoption.md) now adds a separate retry lock spanning selection, public enqueue, optional `flush_tasks`, and source removal for both retry surfaces.
Task-08 Q-04 is resolved: only normal public enqueue returns qualify for the final optional flush and source removal. Every enqueue exception retains the original DLQ source and original per-ID error, even if pending work was written first; later IDs continue. An invocation with no normal enqueue returns performs no explicit flush or removal.
This simpler policy supersedes the earlier durable-write-after-error removal decision. Add no private write acknowledgement, receipts, or write-candidate tracking. A failed scheduling call can leave pending work awaiting startup/recovery, currently usually restart; retaining its DLQ source lets an operator retry but accepts duplicate work. No automatic replacement enqueue or pending-to-DLQ transfer is added.
Task 08 remains `draft` with Q-05 through Q-07 open for locking, concise recovery guidance, and the internal lifecycle owner.

Add `DeadTasks#retry(id_or_ids)` with the same input forms as collection deletion.
Process all distinct requested IDs as one batch per invocation.
The user excludes internal input partitioning and batch-size limits from this task.
Return an ordered Hash with `true`, `false`, or a per-ID `StandardError` object.
Shared snapshot, I/O, and flush failures propagate unchanged.
Removal failure raises the existing `DeadTaskRetryError` with its cause.
A raised call returns no Hash. Pending work can already exist.
Task 08 defines the approved result and shared-failure contract.

The collection exposes the public API, input-validation, and result contract without knowing about locking or calling lock/coordination-scope APIs.
The earlier collection-owned orchestration and prohibition on backend-owned retry orchestration are pending task-08 Q-07: a complete invocation-spanning lifecycle owner below the collection boundary is not yet selected.
No retry service, backend orchestration interface, or replay callback is approved by this constraint amendment. Existing behavior and lock scope remain required.
Entries can share small private replay helpers without transferring individual retry ownership. Backends retain their multi-ID lookup, optional flush, and bulk-removal capabilities.

This extension replaces only the earlier prohibition on collection retry and the requirement for one sole retry workflow.
The extension also replaces the parent's bulk-retry non-goal.
Batch warning counts and per-ID error results follow task 08.
Task 05's requirements remain the stage-specific contract for individual entry retry; task 08 adds the shared retry lock around that sequence.
ADR-001's traversal, ownership, snapshot, and inspection decisions remain applicable.
ADR-002's fresh-enqueue, optional-flush, suppression, and recovery limits remain applicable.
Task 06 retains its separate per-ID Rake workflow.

Disk selects current requested winners once per invocation before application code runs.
Each selected record supplies replay input; the collection performs no later per-ID existence check.
A concurrent removal does not cancel selected work.
A concurrent same-ID replacement does not change replay input but can be removed by later exact-ID deletion.
The separate retry lock spans application code; the existing dead-store lock does not. Normal enqueue and scheduling use the existing pending WAL, without a dedicated retry WAL or destination routing.

Periodic synchronization can occur during batch processing.
The collection still requires a final explicit optional flush before bulk removal.
The collection preserves the existing timer and storage lifecycle.
This boundary does not establish complete crash-recovery safety or prove queue acceptance.

## Consequences

- One batch reduces selection scans, explicit flushes, and compactions compared with repeated entry retry.
- Selection and compaction still depend on dead-store size. The API does not guarantee a fixed speed improvement.
- Results, normalized IDs, selected records, and winner locations each require O(requested IDs) slots, plus selected payload bytes.
- One snapshot supplies replay input for the complete invocation. Callers can split IDs into separate invocations.
- Per-ID failures preserve source records. Public enqueue can still create pending work before raising.
- Shared failures stop the invocation and provide no result Hash. Pending work can already exist.
- A crash before bulk removal can leave pending copies for every eligible requested ID. The duplication window covers the complete invocation.
- Silent suppression can still remove dead records without creating pending work. Acceptance detection remains deferred.
- Entry retry gains the shared retry lock; its other behavior and normal enqueue remain compatible. The new collection API needs dedicated YARD and operator guidance.
