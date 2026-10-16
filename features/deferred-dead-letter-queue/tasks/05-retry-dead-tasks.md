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

- Add `DeadTask#retry` and `DeadTasks#retry(id)` according to ADR-001. Extend and reuse task 04's private action delegate with `retry(id)`. Do not add a second delegate or another mutation dependency. The concrete delegate may remain the creating `DeadTasks` collection, but callers cannot observe or replace it. The entry method delegates with its exact ID. The collection method accepts one String ID. It returns `false` when task 03 lookup returns `nil`. It returns `true` after successful enqueue and removal handling.
- Re-read the newest frame-valid record for the entry's exact authoritative outer ID at mutation time. Use task 03's `find_by_id` behavior. Disk uses only frame checks during selection and applies no shared top-level schema validator. If selected decoding raises `ArgumentError`, `NameError`, or `TypeError`, lookup prints its standard-output diagnostic and returns `nil`. Retry then returns `false` without enqueue or removal. The selected record remains stored and hides older duplicates. Other selected decoding and entry-construction failures propagate without older fallback. Other adapters preserve the same public lookup result under their storage contracts. A stale entry must not replay removed or replaced record data from its old snapshot.
- Resolve the exact stored class name without `eval`. Reject a missing constant, a non-Class constant, or a class not including `Rage::Deferred::Task` with a dedicated retry error that identifies the dead-task ID and preserves the record.
- Decode the original positional and keyword arguments using task 03's frozen Marshal load, `Rage::Deferred::Context` accessors, nil normalization, and successful-only cache. Marshal deserialization failures, missing referenced constants, or raised accessor failures use the documented dead-task context-deserialization/retry error and leave the record intact. Retry does not add context shape, minimum-length, or extracted-value type validation that task 03 deliberately defers to a future storage-version change.
- Invoke the resolved task class's normal public `enqueue(*args, **kwargs)` path. Create a fresh task ID and context, reset attempts, and run current enqueue middleware and enqueue telemetry. Do not reconstruct the old exception or preserve the old task ID, delay, scheduled time, logger context, or user context.
- `DeadTasks#retry` owns lookup, class validation, argument decoding, public enqueue, optional flush, and removal. After enqueue returns, the collection calls `@backend.flush` only when `@backend.respond_to?(:flush)`. The collection then calls `@backend.remove_dead_tasks(id)` and returns `true`. The collection does not branch on backend class.
- Disk exposes optional `flush`, which delegates to its internal `TasksStorage#sync!`. An adapter may omit `flush` only when successful enqueue already guarantees durable, recoverable pending work. A buffered adapter must implement `flush` and establish that guarantee before returning. Nil requires no flush method and returns `false` before enqueue.
- Class resolution, decoding, middleware, backpressure, or pending-storage failure leaves the dead record unchanged. An optional flush failure propagates unchanged and leaves the dead record intact. The pending write has an uncertain outcome. Execution can start before synchronization finishes. A repeated retry can therefore duplicate work.
- After enqueue and backend persistence confirmation succeed, remove the exact dead-task ID through task 01's backend primitive. If removal raises, raise a retry-specific error stating that work may exist in both stores. Retain the removal exception as `cause`. Never claim rollback of already-persisted work.
- If removal returns zero because another operator deleted the record after this call enqueued it, return successful retry: new work exists and the dead record is absent.
- Concurrent retries are not exactly-once. Two callers racing from valid snapshots can enqueue duplicate work; a delete racing with retry cannot cancel work already enqueued. Document operator coordination and do not add a lock around application middleware/user code.
- Keep `entry.retry` and `dead_tasks.each(&:retry)` as immediate, mutation-safe convenience operations for individual entries or small sets. Retrying a large set necessarily performs a fresh enqueue per task and, under task 01, an individual lookup/removal with a possible full dead-store compaction per success; aggregate storage work may approach O(N²). This feature adds no collection-level bulk-retry primitive. Operators must coordinate large retry runs and proceed with explicit awareness of their cost and at-least-once duplicate behavior; optimized large-set retry is future work.
- Add an early `Iodine.running?` guard to `Queue#schedule`. When false, it must not call `Iodine.run_after`, change backlog counters, create a Fiber, or touch Iodine task counters; `Queue#enqueue` still persists the pending record before the scheduling no-op.
- Detect retry outside a running Iodine process. After pending persistence succeeds, emit one stderr warning. Identify only the dead-task ID and state that server restart is required. Do not print args, kwargs, context, or exception text. This warning differs from task 03's selected decoding diagnostic, which can occur before enqueue.
- Do not implement orphan-WAL polling/adoption or contact a running worker. The out-of-process WAL remains unavailable to existing workers until restart, as accepted in issue #369.
- Nil retry returns `false` without class resolution, context decoding, scheduling, files, or background work.
- Task 06's multi-ID Rake retry invokes this single-ID operation independently for every requested ID and reports partial results; it is not an atomic or optimized bulk-retry primitive.
- Fully document return values, class and decode errors, fresh enqueue semantics, partial success, concurrency, and restart requirements with YARD. Include task 03's unsanitized selected-failure diagnostic and the resulting `false` return.

## Design

Find the newest frame-valid current entry through task 03's `find_by_id`. A rescued selected failure prints the task 03 diagnostic and returns `nil`. Retry then returns `false` without enqueue or removal. Other selected failures propagate without older fallback.

`DeadTasks#retry` resolves the task class and decodes the original arguments. The collection then uses this success path:

```ruby
task_class.enqueue(*args, **kwargs)
@backend.flush if @backend.respond_to?(:flush)
@backend.remove_dead_tasks(id)
true
```

Disk implements `flush` by calling `@tasks_storage.sync!`. `respond_to?(:flush)` selects the optional operation; the check alone does not prove persistence. An adapter without `flush` must confirm durable, recoverable persistence through successful enqueue. Buffered adapters must provide `flush`. Database adapters with that enqueue guarantee need no flush method.

The implementation also preserves the specified removal-error wrapper and restart warning. The success path does not detect middleware that returns without yielding. See Q-01. Interrupted WAL tails and replay argument mutability also remain open. See Q-03 and Q-04.

`TasksStorage#sync!` is internal to Disk. The method confirms file and directory durability for the retry's pending work across WAL rotation and recovery. The method must cover the relevant WAL files, even if enqueue changes the active WAL. Disk must keep each required file open until synchronization finishes. Disk must retain source WAL data until its replacement is confirmed durable. Elapsed `fsync_frequency` alone must not authorize source-file cleanup.

The timer continues to batch ordinary WAL synchronization. Explicit synchronization must not clear the dirty state of newer writes. Failed timer synchronization must leave unsynchronized writes eligible for another attempt. Ordinary enqueue gains no retry-specific synchronization or retry-state checks.

The shared backend contract does not require `fsync` or filesystem operations. Other adapters may omit `flush` under the enqueue persistence guarantee above. Do not require `flush` on every backend. Do not add no-op flush methods. Nil returns `false` without work. Retry does not use an application callback block.

Extend the private action protocol introduced by task 04 from `delete(id)` to `delete(id)` plus `retry(id)`. Preserve the same injected delegate and private entry-construction path so inspection, deletion, and retry do not create competing entry representations.

Place the non-running guard at the top of `Queue#schedule`, not only in the dead-task API, so every enqueue made outside Iodine consistently persists without attempting server scheduling.

## Implementation constraints

- Never remove a dead record before enqueue and backend persistence confirmation succeed.
- Keep WAL synchronization inside Disk, behind optional `flush`. Preserve the public `Task.enqueue` and shared enqueue/`add_task` signatures.
- Keep retry orchestration in `DeadTasks#retry`. Do not add backend `retry_dead_task` or `finalize_dead_task_retry` methods.
- Do not add retry flags, `RetryAttempt`, enqueue-context hooks, or shared `sync:`/`durable:` enqueue parameters.
- Preserve Deferred's at-least-once model; do not promise exactly-once replay or transactional rollback.
- Do not reuse the original task ID or serialized internal context.
- Preserve the selected fresh-enqueue behavior without the original logger/user context.
- Do not add orphan-WAL polling, automatic restart, new backend configuration, or network communication.
- Do not add a collection-level bulk retry or imply that an optimized bulk-retry path exists.
- Do not introduce a second operation delegate, inject the raw backend/snapshot, or replace task 04's private action wiring.
- Do not use or override Enumerable `find` for retry lookup; use task 03's explicit `find_by_id` path.
- Do not hold the dead-task file lock while enqueue middleware or other application code runs.
- Do not rescue `Exception`; preserve Rage's existing exception boundaries and Ruby 3.3 behavior.
- Preserve ADR-001's active-entry and collection-mutation surfaces; do not replace either surface or otherwise change retry semantics.

## Acceptance criteria

- [ ] Missing, stale, Nil, or rescued-selected-failure retry returns `false`. A recoverable retry returns `true` after the documented enqueue and removal sequence.
- [ ] Retry re-reads the newest frame-valid record for the exact authoritative outer ID without shared top-level schema validation. Selected `ArgumentError`, `NameError`, and `TypeError` failures print the task 03 diagnostic and return `false`. Other selected decoding and entry-construction failures propagate. No selected failure falls back to an older duplicate.
- [ ] `DeadTask#retry` reuses task 04's private delegate and exact-ID wiring without introducing another entry dependency or changing existing deletion behavior.
- [ ] Retry resolves and validates the task class without `eval` and preserves the dead record on class or context failure.
- [ ] Retry uses original positional/keyword arguments through a fresh normal enqueue with reset ID, attempts, context, middleware, and telemetry.
- [ ] The collection performs public enqueue, optional flush selected through `respond_to?(:flush)`, and exact-ID removal in that order.
- [ ] Disk flush delegates to `TasksStorage#sync!`. An adapter omits flush only when successful enqueue guarantees durable, recoverable pending work.
- [ ] An optional flush failure propagates and preserves the dead record. Documentation states the uncertain pending outcome and possible execution before synchronization.
- [ ] Rotation and recovery preserve durable retry data. Source WAL cleanup requires confirmed replacement durability, including directory durability.
- [ ] Ordinary enqueue retains timer batching without retry flags or signature changes. Failed timer synchronization remains eligible for another attempt.
- [ ] Every pre-enqueue failure leaves the dead record intact. Post-enqueue removal failure reports possible duplication and preserves its cause without claiming rollback.
- [ ] Concurrent retry/delete outcomes follow the documented at-least-once semantics and never hold a storage lock across application code.
- [ ] Retrying during stable enumeration attempts every yielded starting-snapshot entry exactly once. Rescued enumeration failures remain skipped and stored. Documentation states per-task behavior, possible O(N²) work, and no bulk retry.
- [ ] Queue scheduling is unchanged when Iodine is running and persistence-only when it is not.
- [ ] Out-of-process retry warns without leaking stored payloads and does not claim execution before server restart.
- [ ] Public retry/error APIs have complete YARD documentation.

## Open questions

- Q-01: If enqueue middleware returns without yielding, should retry retain the dead record or treat the middleware decision as success? Decides: the user. Recommendation: retain the dead record. Public `Task.enqueue` returns `nil` in both cases. The selected sequence cannot detect the difference. Detection remains undecided.

- Q-03: How must Disk establish recoverable pending records after interrupted appends or recovery copying? Decides: the user. Recommendation: retain the dead record until recoverability is established. A successful sync alone cannot establish a valid record boundary. The tail policy remains undecided.
- Q-04: What mutability must retry arguments provide during live execution and after WAL recovery? Decides: the user. Recommendation: preserve immutable inspection values and require equivalent replay behavior. The replay policy remains undecided. Frozen nested values can cause `FrozenError` in middleware or task execution.

## Verification

- Add focused collection and entry specs for success, missing, stale, Nil, class, and context cases. Verify exact return and error types.
- Cover fallback from a newer frame-invalid duplicate to an older frame-valid record. Cover each rescued selected failure class on a newer frame-valid duplicate.
- Verify the task 03 diagnostic, `false` result, no enqueue, no removal, stored-record preservation, and no older fallback. Verify propagation for other selected failures.
- Verify the task-04 delegate gains `retry(id)` without replacing its identity or breaking `DeadTask#delete`, task-03 readers, or task-02 traversal.
- Cover positional-only, keyword-only, mixed, and empty arguments under Ruby 3.3, plus fresh ID/attempt/context state and enqueue middleware/telemetry execution.
- Verify collection-owned enqueue, optional flush, and removal order. Verify Disk flush delegates to `TasksStorage#sync!`.
- Add adapter-conformance coverage for buffered persistence with flush and durable enqueue without flush. Verify persistence semantics, not only method presence.
- Verify flush failure prevents removal and propagates unchanged. Verify Nil performs no enqueue or flush. Do not require no-op flush methods.
- Cover pending-store, middleware, backpressure, schedule, synchronization, and removal failures. Verify retained dead records, uncertain pending outcomes, and preserved removal-error causes.
- Verify file and directory synchronization before deletion, including retry without a running timer. Cover rotation during enqueue and recovery-source cleanup after synchronization failure.
- Verify timer batching after explicit synchronization. Cover failed timer synchronization and newer writes during synchronization without lost dirty state.
- Exercise concurrent retry/retry and retry/delete interleavings to document possible duplicates without unrelated loss or storage-locking around application code.
- Exercise `each(&:retry)` across rename-based removals. Verify that each yielded starting-snapshot entry is attempted once without duplicates.
- Add a rescued enumeration failure. Verify its diagnostic, continued traversal, retained record, and attempts for all later yielded entries.
- Update queue specs for `Iodine.running?` true/false, including persistence, backlog, timer, Fiber, and Iodine task-counter assertions.
- Run focused retry specs, `bundle exec rspec spec/deferred/queue_spec.rb spec/deferred/task_spec.rb spec/deferred/deferred_spec.rb`, the broader `bundle exec rspec spec/deferred`, and RuboCop for changed files.

## Result

- Implementation PR:
