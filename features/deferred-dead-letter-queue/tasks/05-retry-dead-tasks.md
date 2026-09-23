---
status: ready-for-development
---

# Retry dead tasks

## Goal

Let trusted operators retry recoverable dead tasks through the Ruby API with fresh retry state, enqueue-before-delete ordering, explicit at-least-once partial failures, and safe behavior inside or outside a running Iodine server.

## Context

Task 02 establishes the passive collection/entry retrieval model, task 03 provides exact lookup and safe argument decoding without adding mutation wiring, and task 04 introduces the private entry-action mechanism with operator-initiated deletion. Task 01 supplies durable pending/dead storage but no atomic transaction spans the two files.

This task depends on both tasks 03 and 04. It implements REQ-12 through REQ-16, retry-related REQ-18 through REQ-20 and REQ-24 through REQ-26, plus AC-05, AC-08, and the retry/concurrency portion of AC-11. Follow [ADR-001](../adr/001-public-dead-task-object-model.md) and [ADR-002](../adr/002-dead-task-retry-lifecycle.md).

## Requirements

- Add `DeadTask#retry` and `DeadTasks#retry(id)` according to ADR-001. Extend and reuse task 04's private action delegate with `retry(id)`; do not add a second delegate or pass the collection, raw backend, snapshot, or another mutation dependency separately. The concrete delegate may remain the creating `DeadTasks` collection, but callers cannot observe or replace it. The entry method delegates with its exact ID. The collection method accepts one String ID, returns `false` when no current valid record matches, and returns `true` after successful enqueue/removal handling.
- Re-read the newest fully valid logical record for the entry's exact ID at mutation time through task 03's `find_by_id` behavior. Disk applies task 02's authoritative outer-ID and validity rules inside its backend capability; other adapters apply their equivalent integrity rules. A stale entry must not replay record data that has been removed or replaced outside its snapshot.
- Resolve the exact stored class name without `eval`. Reject a missing constant, a non-Class constant, or a class not including `Rage::Deferred::Task` with a dedicated retry error that identifies the dead-task ID and preserves the record.
- Decode the original positional and keyword arguments using task 03. Context corruption, missing referenced constants, or an incompatible context layout raises the documented dead-task deserialization/retry error and leaves the record intact.
- Invoke the resolved task class's normal public `enqueue(*args, **kwargs)` path. Create a fresh task ID and context, reset attempts, and run current enqueue middleware and enqueue telemetry. Do not reconstruct the old exception or preserve the old task ID, delay, scheduled time, logger context, or user context.
- Complete the new pending WAL write/enqueue before removing the dead record. Class resolution, decoding, middleware, backpressure, or pending-storage failure leaves the dead record unchanged.
- After enqueue succeeds, remove the exact dead-task ID through task 01's backend primitive. If removal raises, raise a retry-specific error stating that work may exist in both stores and retain the removal exception as `cause`. Never claim rollback of already-persisted work.
- If removal returns zero because another operator deleted the record after this call enqueued it, return successful retry: new work exists and the dead record is absent.
- Concurrent retries are not exactly-once. Two callers racing from valid snapshots can enqueue duplicate work; a delete racing with retry cannot cancel work already enqueued. Document operator coordination and do not add a lock around application middleware/user code.
- Keep `entry.retry` and `dead_tasks.each(&:retry)` as immediate, mutation-safe convenience operations for individual entries or small sets. Retrying a large set necessarily performs a fresh enqueue per task and, under task 01, an individual lookup/removal with a possible full dead-store compaction per success; aggregate storage work may approach O(N²). This feature adds no collection-level bulk-retry primitive. Operators must coordinate large retry runs and proceed with explicit awareness of their cost and at-least-once duplicate behavior; optimized large-set retry is future work.
- Add an early `Iodine.running?` guard to `Queue#schedule`. When false, it must not call `Iodine.run_after`, change backlog counters, create a Fiber, or touch Iodine task counters; `Queue#enqueue` still persists the pending record before the scheduling no-op.
- Detect retry outside a running Iodine process. After pending persistence succeeds, emit one stderr warning identifying only the dead-task ID and stating that the Rage server must restart before execution. Do not print args, kwargs, context, or exception text.
- Do not implement orphan-WAL polling/adoption or contact a running worker. The out-of-process WAL remains unavailable to existing workers until restart, as accepted in issue #369.
- Nil retry returns `false` without class resolution, context decoding, scheduling, files, or background work.
- Task 06's multi-ID Rake retry invokes this single-ID operation independently for every requested ID and reports partial results; it is not an atomic or optimized bulk-retry primitive.
- Fully document return values, class/decode errors, fresh enqueue semantics, partial success/duplication, concurrent retry/delete behavior, and restart requirements with YARD.

## Design

Find the latest current entry through task 03's `find_by_id`, resolve/validate its task class, decode defensive args/kwargs, and call the public task enqueue method. Normal return from enqueue is the commit point for attempting backend removal under task 01's guarantees. Translate public dead-task failures while preserving their causes and the distinction between pre-enqueue failure and post-enqueue cleanup failure.

Extend the private action protocol introduced by task 04 from `delete(id)` to `delete(id)` plus `retry(id)`. Preserve the same injected delegate and private entry-construction path so inspection, deletion, and retry do not create competing entry representations.

Place the non-running guard at the top of `Queue#schedule`, not only in the dead-task API, so every enqueue made outside Iodine consistently persists without attempting server scheduling.

## Implementation constraints

- Never remove a dead record before pending persistence succeeds.
- Preserve Deferred's at-least-once model; do not promise exactly-once replay or transactional rollback.
- Do not reuse the original task ID or serialized internal context.
- Do not preserve original logger/user context unless the still-proposed ADR-002 is changed by a later user decision.
- Do not add orphan-WAL polling, automatic restart, new backend configuration, or network communication.
- Do not add a collection-level bulk retry or imply that an optimized bulk-retry path exists.
- Do not introduce a second operation delegate, inject the raw backend/snapshot, or replace task 04's private action wiring.
- Do not use or override Enumerable `find` for retry lookup; use task 03's explicit `find_by_id` path.
- Do not hold the dead-task file lock while enqueue middleware or other application code runs.
- Do not rescue `Exception`; preserve Rage's existing exception boundaries and Ruby 3.3 behavior.
- Preserve ADR-001's active-entry and collection-mutation surfaces; do not replace either surface or otherwise change retry semantics.

## Acceptance criteria

- [ ] Missing/stale/Nil retry returns `false`; a valid retry returns `true` after the documented enqueue/removal sequence.
- [ ] `DeadTask#retry` reuses task 04's private delegate and exact-ID wiring without introducing another entry dependency or changing existing deletion behavior.
- [ ] Retry resolves and validates the task class without `eval` and preserves the dead record on class or context failure.
- [ ] Retry uses original positional/keyword arguments through a fresh normal enqueue with reset ID, attempts, context, middleware, and telemetry.
- [ ] Every pre-enqueue failure leaves the dead record intact, and post-enqueue removal failure reports possible duplication without claiming rollback.
- [ ] Concurrent retry/delete outcomes follow the documented at-least-once semantics and never hold a storage lock across application code.
- [ ] Retrying entries during stable enumeration attempts every logical entry in the starting snapshot exactly once, while documentation states the per-task enqueue/removal behavior, possible O(N²) storage work, and absence of bulk retry.
- [ ] Queue scheduling is unchanged when Iodine is running and persistence-only when it is not.
- [ ] Out-of-process retry warns without leaking stored payloads and does not claim execution before server restart.
- [ ] Public retry/error APIs have complete YARD documentation.

## Verification

- Add focused collection/entry specs for success, missing/stale/Nil records, class resolution, invalid task constants, context errors, and exact return/error types.
- Verify the task-04 delegate gains `retry(id)` without replacing its identity or breaking `DeadTask#delete`, task-03 readers, or task-02 traversal.
- Cover positional-only, keyword-only, mixed, and empty arguments under Ruby 3.3, plus fresh ID/attempt/context state and enqueue middleware/telemetry execution.
- Cover pending-store, middleware, backpressure, schedule, and dead-store removal failures, including records left in one or both stores and preserved exception causes.
- Exercise concurrent retry/retry and retry/delete interleavings to document possible duplicates without unrelated loss or storage-locking around application code.
- Exercise `each(&:retry)` across rename-based removals to verify that every starting-snapshot entry is attempted once without skips or duplicates; keep this correctness coverage even though the flow is not recommended for large sets.
- Update queue specs for `Iodine.running?` true/false, including persistence, backlog, timer, Fiber, and Iodine task-counter assertions.
- Run focused retry specs, `bundle exec rspec spec/deferred/queue_spec.rb spec/deferred/task_spec.rb spec/deferred/deferred_spec.rb`, the broader `bundle exec rspec spec/deferred`, and RuboCop for changed files.

## Result

- Implementation PR:
