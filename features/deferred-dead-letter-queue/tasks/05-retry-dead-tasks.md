---
status: ready-for-development
---

# Retry dead tasks

## Goal

Let trusted operators retry dead tasks through the Ruby API with fresh state and enqueue-before-delete ordering. Document partial failures, suppression limits, and Iodine behavior.

## Context

Task 02 establishes passive entries. Task 03 adds exact lookup and safe argument decoding. Task 04 gives active entries the shared backend privately and adds direct deletion. Task 01 supplies durable pending/dead storage but no atomic transaction spans the two files.

This task depends on tasks 03 and 04. [Task 07](07-recoverable-pending-wal-appends.md) remains independent future work and does not block retry. Task 05 implements REQ-12 through REQ-16, retry-related REQ-18 through REQ-20, REQ-24 through REQ-26, and REQ-30, plus AC-05, AC-08, and the retry/concurrency portion of AC-11. Follow [ADR-001](../adr/001-public-dead-task-object-model.md) and [ADR-002](../adr/002-dead-task-retry-lifecycle.md).

## Requirements

- Add `DeadTask#retry` according to ADR-001. The entry owns the complete retry workflow through its private shared backend. Keep `DeadTasks#retry(id)` as a thin convenience method: call `find_by_id(id)`, then call the returned entry's `retry`. Return `false` when lookup returns `nil`. Preserve lookup's one-String argument contract. Do not add another retry implementation or action handler.
- Check whether the entry's exact authoritative outer ID still exists at mutation time. Use `@backend.find_dead_task(@id)` only as an existence check. Discard the returned record's class and context. Preserve task 03's `find_by_id` results, diagnostics, and error behavior. Disk uses only frame checks during selection and applies no shared top-level schema validator.

  If selected decoding raises `ArgumentError`, `NameError`, or `TypeError`, lookup prints its standard-output diagnostic and returns `nil`. Retry then returns `false` without enqueue or removal. The selected record remains stored and hides older duplicates.

  Other selected decoding failures propagate without older fallback. Public lookup entry-construction failures also propagate unchanged. Other adapters preserve the same public lookup result under their storage contracts. If the ID is absent at the check, retry returns `false` without enqueue or removal.

  Use the entry's original `@task_class` and `@__context` for class validation and argument decoding. Keep the original entry metadata and inspection cache unchanged.
- Resolve the exact stored class name without `eval`. Reject a missing constant, a non-Class constant, or a class not including `Rage::Deferred::Task` with a dedicated retry error that identifies the dead-task ID and preserves the record.
- Decode one independent replay context from the entry's serialized context bytes with ordinary `Marshal.load`, without `freeze: true`. Extract args/kwargs through `Rage::Deferred::Context` accessors. Normalize accessor-returned `nil` to new mutable empty collections. Preserve Marshal-supported shared references and cycles across both argument sets through that single decode. Normal public enqueue splats may create new outer argument containers. This contract does not preserve those outer containers' identities.
- Keep the public inspection graph frozen and its successful cache unchanged. Do not reuse or shallow-copy that graph for replay. Do not serialize the already decoded inspection graph to make a copy. Replay permits the same nested mutations as ordinary pending-WAL recovery, subject to ordinary Marshal and custom deserialization behavior.
- Marshal and context-accessor exceptions use the dedicated context-deserialization error with the dead-task ID and original `cause`. Custom deserialization failures follow the same boundary. Keep the dead record intact. Do not add context shape, minimum-length, or extracted-value type validation.
- Preserve public `Task.enqueue` without changes. Keep its unconditional `nil` return, signature, argument handling, middleware, telemetry, and multiple-yield behavior for Disk and Nil.
- Ignore the public enqueue return value. Do not add acceptance bookkeeping, retry flags, receipt objects, or context hooks.
- Accept silent middleware or telemetry suppression as a current limitation. Retry still performs optional flush, emits the applicable restart warning, removes the dead record, and returns `true`. No pending work need exist. This can lose the task. Defer acceptance detection and suppression retention to a future improvement.
- If middleware or telemetry raises, propagate the exception and retain the dead record. Pending work can already exist. A repeated retry can duplicate that work.
- Invoke the resolved task class's normal public `enqueue(*args, **kwargs)` path. Preserve fresh context construction and current enqueue middleware and telemetry. When the call reaches the queue, enqueue creates a fresh task ID and resets attempts. Do not reconstruct the old exception or preserve the old task ID, delay, scheduled time, logger context, or user context.
- `DeadTask#retry` owns the existence check, class validation, argument decoding, public enqueue, optional flush, and removal. After enqueue returns normally, the entry calls `@backend.flush` only when `@backend.respond_to?(:flush)`. The entry then calls `@backend.remove_dead_tasks(@id)` and returns `true`. The entry does not branch on backend class.
- Disk exposes optional `flush`, which delegates to its internal `TasksStorage#sync!`. An adapter may omit `flush` when successful enqueue already completes its ordinary acknowledged persistence operation. A buffered adapter must provide `flush` to complete that operation. Retry retains each adapter's existing storage limitations. Nil requires no flush method and returns `false` before enqueue.
- Class resolution, decoding, middleware, backpressure, or pending-storage failure leaves the dead record unchanged. An optional flush failure propagates unchanged and leaves the dead record intact. The pending write has an uncertain outcome. Execution can start before synchronization finishes. A repeated retry can therefore duplicate work.
- After public enqueue and optional flush return normally, remove the exact dead-task ID through task 01's backend primitive. If removal raises, raise a retry-specific error stating that work may exist in both stores. Retain the removal exception as `cause`. Never claim rollback of already-persisted work.
- If removal returns zero, return `true`. Another operator can remove the dead record before this call reaches removal. The enqueue/flush/removal sequence completed. The result does not prove that pending work exists.
- Concurrent retries are not exactly-once. Two callers racing from valid snapshots can enqueue duplicate work; a delete racing with retry cannot cancel work already enqueued. The existence check does not reserve the record or compare versions. A same-ID replacement does not change the entry's class or arguments. Later exact-ID removal can delete the replacement. Document operator coordination and do not add a lock around application middleware/user code.
- Keep `entry.retry` and `dead_tasks.each(&:retry)` as immediate, mutation-safe convenience operations for individual entries or small sets. Retrying a large set performs a public enqueue call per task. Task 01 requires individual lookup/removal and can require full dead-store compaction per completed retry. Aggregate storage work may approach O(N²). This feature adds no collection-level bulk-retry primitive.

  Operators must coordinate large retry runs. Document their cost, possible duplicates, and the suppression limitation. Optimized large-set retry is future work.
- Add an early `Iodine.running?` guard to `Queue#schedule`. When false, it must not call `Iodine.run_after`, change backlog counters, create a Fiber, or touch Iodine task counters; `Queue#enqueue` still persists the pending record before the scheduling no-op.
- Detect retry outside a running Iodine process. After enqueue and optional flush return normally, emit one stderr warning. Emit this warning even after silent suppression. Identify only the dead-task ID and state that server restart is required. Do not print args, kwargs, context, or exception text. This warning differs from task 03's selected decoding diagnostic, which can occur before enqueue.
- Do not implement orphan-WAL polling/adoption or contact a running worker. The out-of-process WAL remains unavailable to existing workers until restart, as accepted in issue #369.
- Nil dead-task retry returns `false` without class resolution, context decoding, scheduling, files, or background work. Ordinary Nil enqueue retains its unconditional `nil` return.
- Task 06's multi-ID Rake retry invokes this single-ID operation independently for every requested ID and reports partial results; it is not an atomic or optimized bulk-retry primitive.
- Document the retry sequence result, suppression limitation, mutable replay, errors, partial success, concurrency, and restart requirements with YARD.
- Update the changelog for retry. Document unchanged enqueue behavior and deferred acceptance detection. Include task 03's unsanitized selected-failure diagnostic and the resulting `false` return.

## Design

`DeadTask#retry` checks existence through `@backend.find_dead_task(@id)` and discards the returned record. A missing record returns `false`. A rescued selected failure prints the task 03 diagnostic and returns `nil`; retry then returns `false`. Other selected failures propagate without older fallback.

Private `resolve_task_class` reads `@task_class`; private `decode_task_arguments` reads `@__context`. Neither helper takes arguments. Do not construct another `DeadTask` or copy or validate the existence-check record. Public lookup and collection entry construction remain unchanged.

Validate the entry's task class. Decode a separate mutable replay context from the entry's serialized bytes. Reuse `Rage::Deferred::Context` accessors, nil normalization, and the ID-and-cause error boundary. Keep the original inspection graph and cache unchanged. Do not add a service or another validation layer.

The entry then uses this success path:

```ruby
task_class.enqueue(*args, **kwargs)
@backend.flush if @backend.respond_to?(:flush)
@backend.remove_dead_tasks(@id)
true
```

`DeadTasks#retry(id)` calls `find_by_id(id)` and returns `false` when lookup returns `nil`. Otherwise, the collection calls the returned entry's `retry`. The entry then checks existence before enqueue. An intervening removal returns `false`; a same-ID replacement leaves the first lookup's class and arguments in use. The collection does not duplicate retry logic. Entry retry does not call collection retry.

Disk implements `flush` by calling `@tasks_storage.sync!`. `respond_to?(:flush)` selects the optional operation; the check alone does not prove persistence. An adapter without `flush` completes its ordinary acknowledged persistence operation through successful enqueue. Buffered adapters must provide `flush`. Adapters with synchronous persistence need no flush method. This contract does not remove inherited backend defects.

`Task.enqueue` remains unchanged and returns `nil` after normal completion. Retry does not inspect this return value. A normal return does not prove that the intended queue call occurred. Silent middleware or telemetry suppression can therefore remove the dead record without creating pending work. Acceptance detection and suppression retention remain future improvements.

The implementation also preserves the specified removal-error wrapper and restart warning. Silent suppression still reaches removal and the applicable warning. The retry Boolean reports sequence completion, not queue acceptance. Task 07 remains deferred. Retry does not wait for pending-WAL boundary or source-cleanup repairs.

`TasksStorage#sync!` is internal to Disk. The method synchronizes the currently active WAL file, then the storage directory. A synchronization failure propagates and preserves the source dead record. The method leaves the timer's dirty state untouched. Ordinary timer batching remains unchanged.

Synchronization confirms current WAL bytes and directory metadata under the existing storage lifecycle. It does not validate or repair record framing, adopt orphan files, or fix inherited tail, rotation, or recovery defects. Those defects can still affect crash recovery after retry returns `true`. Retry adds no refusal checks for those inherited defects.

Ordinary enqueue gains no acceptance bookkeeping, retry-specific synchronization, backend type check, or shared retry state. This task changes no rotation, recovery-copying, source-cleanup, or timer-failure policy.

The shared backend contract does not require `fsync` or filesystem operations. Other adapters may omit `flush` under their ordinary acknowledged persistence semantics. Do not require `flush` on every backend. Do not add no-op flush methods. Nil returns `false` without work. Retry does not use an application callback block.

Reuse task 04's private shared backend reference. Entries retain no collection, action handler, service, iterator, descriptor, cursor, winner index, or traversal registry. Retained entries can retry after snapshot cleanup. The existence check and mutation do not change an active traversal.

Place the non-running guard at the top of `Queue#schedule`, not only in the dead-task API, so every enqueue made outside Iodine consistently persists without attempting server scheduling.

## Implementation constraints

- Never remove a dead record before public enqueue and optional flush return normally. Do not require proven queue acceptance.
- Defer WAL boundary repair, short-write handling, and source-cleanup fixes to independent task 07. Task 07 is not a prerequisite.
- Do not change rotation, recovery copying, file lifecycle, or timer policy in this task.
- Keep WAL synchronization inside Disk, behind optional `flush`. Preserve the public `Task.enqueue` and shared enqueue/`add_task` signatures.
- Keep retry orchestration only in `DeadTask#retry`. Do not add backend `retry_dead_task` or `finalize_dead_task_retry` methods.
- Do not change `Task.enqueue`. Do not add acceptance bookkeeping, retry flags, receipt objects, context hooks, or shared synchronization parameters.
- Do not promise at-least-once preservation after silent suppression. Do not promise exactly-once replay or transactional rollback.
- Do not reuse the original task ID or serialized internal context.
- Preserve the selected fresh-enqueue behavior without the original logger/user context.
- Do not add orphan-WAL polling, automatic restart, new backend configuration, or network communication.
- Do not add a collection-level bulk retry or imply that an optimized bulk-retry path exists.
- Keep backend access private. Do not introduce an action handler, collection back-reference, or public backend accessor.
- Preserve task 03's exact lookup behavior. Do not use or override Enumerable `find` for retry lookup.
- Do not hold the dead-task file lock while enqueue middleware or other application code runs.
- Do not rescue `Exception`; preserve Rage's existing exception boundaries and Ruby 3.3 behavior.
- Preserve ADR-001's active-entry and collection-mutation surfaces; do not replace either surface or otherwise change retry semantics.

## Acceptance criteria

- [ ] Missing, Nil, or rescued-selected-failure retry returns `false`. Retry returns `true` after the documented enqueue, optional flush, and removal sequence. The result does not prove queue acceptance.
- [ ] Retry checks existence through exact-ID lookup without shared top-level schema validation. Selected `ArgumentError`, `NameError`, and `TypeError` failures print the task 03 diagnostic and return `false`. Other selected decoding failures propagate. Public lookup entry-construction failures also propagate unchanged. No selected failure falls back to an older duplicate.
- [ ] `DeadTask#retry` owns the only retry implementation and reuses task 04's private shared backend. Collection retry delegates through `find_by_id` and `entry.retry`.
- [ ] Retry resolves and validates the task class without `eval` and preserves the dead record on class or context failure.
- [ ] Retry uses original positional/keyword arguments through a fresh normal enqueue with reset ID, attempts, context, middleware, and telemetry.
- [ ] Public enqueue retains unconditional `nil` returns, its signature, middleware, and telemetry for Disk and Nil. Retry ignores the return value.
- [ ] Silent middleware or telemetry suppression still permits flush, the applicable restart warning, removal, and `true` without pending work. Documentation states this risk.
- [ ] Retry performs public enqueue, optional flush selected through `respond_to?(:flush)`, and exact-ID removal in that order.
- [ ] Replay uses an independent mutable graph with Marshal-supported aliases and cycles under normal enqueue splat semantics. Immutable inspection values and their cached identities remain unchanged.
- [ ] Retry uses the entry's original class and context after the existence check, without constructing another entry. Same-ID replacement does not change replay input; later exact-ID removal can delete the replacement. The original entry metadata and inspection cache remain unchanged. Entry actions work after snapshot cleanup.
- [ ] Disk flush delegates to `TasksStorage#sync!` for the active WAL and storage directory. Other adapters retain ordinary acknowledged persistence semantics.
- [ ] An optional flush failure propagates and preserves the dead record. Documentation states the uncertain pending outcome and possible execution before synchronization.
- [ ] Documentation states that inherited WAL framing, rotation, and recovery defects remain. Successful retry does not establish complete crash-recovery safety.
- [ ] Explicit synchronization leaves timer dirty state untouched. Ordinary batching and existing timer-failure behavior remain unchanged.
- [ ] Every pre-enqueue failure leaves the dead record intact. Post-enqueue removal failure reports possible duplication and preserves its cause without claiming rollback.
- [ ] Concurrent retry/delete outcomes follow the documented duplication and suppression limits and never hold a storage lock across application code.
- [ ] Retrying during stable enumeration attempts every yielded starting-snapshot entry exactly once. Rescued enumeration failures remain skipped and stored. Documentation states per-task behavior, possible O(N²) work, and no bulk retry.
- [ ] Queue scheduling is unchanged when Iodine is running and persistence-only when it is not.
- [ ] Out-of-process retry warns without leaking stored payloads and does not claim execution before server restart.
- [ ] Public retry/error APIs have complete YARD documentation.

## Verification

- Add focused collection and entry specs for success, missing, concurrently deleted, replaced, Nil, class, and context cases. Verify exact return and error types.
- Cover fallback from a newer frame-invalid duplicate to an older frame-valid record. Cover each rescued selected failure class on a newer frame-valid duplicate.
- Verify the task 03 diagnostic, `false` result, no enqueue, no removal, stored-record preservation, and no older fallback. Verify propagation for other selected failures.
- Verify entries use the identical shared backend for the existence check, flush, and removal. Verify no action handler or collection reference remains.
- Verify collection retry delegates through exact lookup to entry retry. Cover removal or replacement between collection lookup and the existence check.
- Verify entry retry uses the original class and arguments after same-ID replacement. Preserve original metadata and cached inspection values. Verify later exact-ID removal can delete the replacement.
- Verify the existence check constructs no second entry and retry discards the returned record. Preserve public lookup and collection entry-construction behavior.
- Verify retained entries retry after `first`, `find_by_id`, and block traversal release snapshot resources. Preserve ongoing traversal state.
- Cover positional-only, keyword-only, mixed, and empty arguments under Ruby 3.3. Verify fresh ID/attempt/context state and enqueue middleware/telemetry execution.
- Verify unchanged enqueue `nil` returns and signatures for Disk and Nil. Verify retry ignores the enqueue return value.
- Cover silent middleware and telemetry suppression. Verify retry continues through optional flush, the applicable restart warning, removal, and `true` without pending work.
- Cover middleware and telemetry errors before and after pending work exists. Verify retained source records and possible pending work.
- Verify documentation identifies suppression as an accepted task-loss risk and acceptance detection as a future improvement.
- Mutate nested replay values in enqueue middleware, perform middleware, and `perform`. Compare live execution with pending-WAL recovery.
- Cover nested aliases and cycles within and across args/kwargs. Preserve normal splat behavior for outer containers. Verify replay changes cannot mutate inspection values or change cached identities.
- Cover ordinary and custom Marshal failures with the dedicated ID-and-cause boundary and retained source record.
- Verify entry-owned enqueue, optional flush, and removal order. Verify Disk flush delegates to `TasksStorage#sync!`.
- Verify adapter conformance for buffered persistence with flush and ordinary acknowledged enqueue without flush. Check semantics, not only method presence.
- Verify flush failure prevents removal and propagates unchanged. Verify Nil performs no enqueue or flush. Do not require no-op flush methods.
- Cover pending-store, middleware, backpressure, schedule, synchronization, and removal failures. Verify retained dead records, uncertain pending outcomes, and preserved removal-error causes.
- Verify active-WAL and directory synchronization before deletion, including retry without a running timer.
- Verify failed synchronization retains the dead record and propagates unchanged. Verify explicit synchronization never clears timer dirty state.
- Verify existing timer batching, rotation, recovery copying, and cleanup policy remain unchanged. Do not require task 07 coverage for retry.
- Exercise concurrent retry/retry and retry/delete interleavings to document possible duplicates without unrelated loss or storage-locking around application code.
- Exercise `each(&:retry)` across rename-based removals. Verify that each yielded starting-snapshot entry is attempted once without duplicates.
- Add a rescued enumeration failure. Verify its diagnostic, continued traversal, retained record, and attempts for all later yielded entries.
- Update queue specs for `Iodine.running?` true/false, including persistence, backlog, timer, Fiber, and Iodine task-counter assertions.
- Run focused retry specs, `bundle exec rspec spec/deferred/queue_spec.rb spec/deferred/task_spec.rb spec/deferred/deferred_spec.rb`, the broader `bundle exec rspec spec/deferred`, and RuboCop for changed files.

## Result

- Implementation PR:
