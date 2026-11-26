---
status: rejected
---

# Retry multiple dead tasks by ID

> Rejected on 2026-11-26. The Rage maintainer decided that batch retry is not needed now. Entry retry loops and bulk delete are enough. Do not implement this task.

## Goal

Let trusted operators retry selected IDs through normal public enqueue and coordinated source removal, with fewer Disk scans, explicit flushes, and dead-store compactions.

## Context

Task 05 adds immediate entry retry. Each entry performs an exact lookup, public enqueue, optional flush, and removal.
On Disk, repeated entry retry can scan and rewrite the store once per ID.

This task adds a collection operation and a separate retry coordination lock, without a transaction between pending and dead storage.
Both individual and collection retry use the existing pending WAL and normal scheduling. Add no dedicated retry WAL, enqueue destination routing, or scheduling barrier.
Stronger WAL publication/durability protocol and orphan adoption remain separate future work, without a new task dependency or protocol assumed here.
Q-04 through Q-07 are resolved by the retain-on-error policy, retry-lock policy, operator guidance, and internal `DeadTaskActions` design below. This task remains `draft`; recording these decisions does not approve readiness, publication, or implementation.
This task depends on tasks 03 through 05.
Task 07 remains independent future work.
Task 06 retains its separate per-ID Rake workflow.
This task does not change Rake commands.

This task implements REQ-32 through REQ-36 and AC-13 through AC-15.
Follow [ADR-001](../adr/001-public-dead-task-object-model.md), [ADR-002](../adr/002-dead-task-retry-lifecycle.md), [ADR-003](../adr/003-batch-dead-task-retry.md), and [ADR-004](../adr/004-retry-batch-wal-adoption.md).
ADR-003 defines a scoped extension to the earlier entry-only retry model.
Task 05 retains its stage-specific requirements and lifecycle status.
This task moves individual and collection mutations into backend-specific actions and adds the shared retry lock. Entry replay, return values, warning, exception behavior, and both deletion contracts remain unchanged.

## Terms

- **Eligible ID** — An ID whose public enqueue call returns normally within the invocation.
- **Retry coordination lock** — A separate per-DLQ-store lock spanning retry selection, enqueue, explicit persistence, and source removal.
- **Shared failure** — A retry-coordination, dead-store snapshot, flush, bulk-removal, or cleanup failure that prevents completion of the invocation.

## Requirements

### API and results

- Add `DeadTasks#retry(id_or_ids)`.
- Accept one exact String or one flat Array of Strings, without coercion.
- Reject zero or multiple arguments with `ArgumentError`.
- Reject nested Arrays and non-String members with `TypeError`.
- Validate the complete input before storage access, decoding, enqueue, or other mutation.
- Copy the input IDs before processing can yield. Do not freeze or modify caller-owned Strings or Arrays.
- Remove duplicate IDs. Preserve first-occurrence order for processing and the result Hash.
- Return an empty Hash for an empty Array. Perform no backend operation in this case.
- Return one Hash entry per distinct requested ID after normal completion.
- Use `true` when the enqueue, optional flush, and bulk-removal sequence completes for that ID.
- Use `false` when selection finds no usable record for that ID.
- Use the original `StandardError` object for a per-ID failure, including class, context, or public enqueue failure.
- Continue with later IDs after a per-ID failure. Keep the failing ID out of bulk removal.
- Do not rescue `Exception`. A non-`StandardError` exception stops the call and unwinds resources.

```ruby
results = Rage::Deferred.dead_tasks.retry(["1735689600-12345-1", "1735689601-12345-2"])

results.each do |id, result|
  case result
  when true
    puts "#{id}: retry sequence completed"
  when false
    puts "#{id}: not retried"
  when StandardError
    puts "#{id}: failed (#{result.class})"
  end
end
```

### Selection and replay

- Process all distinct requested IDs as one batch per invocation.
- Do not add an internal batch-size limit, input partitioning, or new configuration.
- Select all requested records through one shared-backend lookup operation.
- `DeadTask` and `DeadTasks` delegate mutations to internal `DeadTaskActions` objects built with the identical existing backend. Public objects retain validation, read-only inspection, and frozen value data, without storage-locking or persistence-capability branches. They do not acquire/release locks, call coordination-scope APIs, or manage coordination blocks. Backend-specific actions own the complete retry lifecycle below this boundary.
- Acquire the retry coordination lock before selecting source records. Individual and collection retry of the same Disk DLQ store use the same retry lock.
- Use a separate stable retry-lock file alongside the existing dead-store lock file. Both use `flock`, without mutexes. Backend path and prefix define the store scope; independent stores use independent locks.
- Open the configured store's retry-lock file separately for each individual or batch invocation. Use distinct open file descriptions, not `dup` or a shared descriptor, so separate Fibers and backend instances for the same store compete for the same lock. Add no shared `@locked` flag, mutex, or backend-instance registry for retry coordination.
- Attempt `File::LOCK_EX | File::LOCK_NB` with Fiber-aware waiting for up to 30 seconds. On expiry, raise the existing `Rage::Deferred::DeadTasksLockTimeout` with a message identifying the retry coordination lock. Acquisition failure stops this invocation before selection or enqueue and returns no result Hash.
- The fixed 30-second deadline applies only to retry-lock acquisition, not execution after acquisition or the total invocation duration. Add no timeout configuration or new exception class; preserve the existing short dead-store lock policy.
- Acquire the retry lock first and retain it through middleware, the explicit optional `flush_tasks`, and source removal. Release it through `ensure` on every exit.
- Release an acquired lock and close each invocation's descriptor on every exit, including acquisition failure. Keep the stable lock file in place; do not unlink or replace it while holders or contenders can exist.
- Nested retry and other cyclic retry-lock waits use the same 30-second acquisition timeout. Add no Fiber-local nesting marker, special nested-retry error, reentrancy, or general cross-store lock-ordering machinery. If a nested timeout escapes middleware, apply the existing enqueue-error policy in its caller.
- Rage does not coordinate application-owned locks or guarantee that application code is deadlock-free. Callers remain responsible for avoiding application-lock cycles; the retry timeout bounds each retry-lock acquisition wait only. Add no public multi-store API.
- On Disk, capture one fixed complete-record snapshot per invocation under the existing short dead-store lock.
- Release the dead-store lock before scanning, decoding, or calling application code; retain the retry lock. Acquire the dead-store lock briefly again when needed for removal.
- Exclude other processes and other Fibers in the same process without blocking the Iodine worker while waiting. A shared-descriptor `flock` alone does not exclude competing Fibers.
- Normal enqueue, task execution, failure recording, and inspection do not acquire the retry lock. Explicit deletion uses only the existing dead-store lock and does not cancel selected replay.
- After successful locked source removal, a later cooperative retry cannot select those removed records. This does not exclude non-cooperating mutations or establish exactly-once transfer.
- Scan the snapshot once, sequentially forwards. Retain locations only for requested IDs.
- Select the newest frame-valid record for each exact authoritative outer ID without decoding candidate payloads.
- Decode only selected requested winners. Do not decode unrelated records or use an older duplicate after a selected failure.
- Apply REQ-17's diagnostic and `false` outcome for selected `ArgumentError`, `NameError`, and `TypeError` decoding failures.
- Preserve other selected-payload and entry-construction `StandardError` objects as per-ID results.
- Treat snapshot acquisition, read, seek, and descriptor-cleanup failures as shared failures. Propagate these failures unchanged.
- Finish the lookup and release its descriptor before the invocation calls public enqueue.
- Use each selected record's class and serialized context for replay. Do not perform another per-ID existence scan.
- Selection checks existence at the invocation snapshot boundary while cooperative retries are excluded. It does not reserve records against deletion or compare versions.
- A removal after selection does not cancel replay. A replacement after selection does not change replay input.
- Bulk removal can delete a same-ID replacement. Operators must coordinate concurrent mutation operations.
- Preserve task 05's class validation, independent mutable replay, Context accessors, and dedicated context error boundary.
- Preserve frozen public inspection data and successful inspection caches.
- Call the resolved class's normal public `enqueue(*args, **kwargs)` once per usable requested ID.
- Preserve fresh task IDs, reset attempts, current middleware, and current telemetry.
- Keep public `Task.enqueue` unchanged. Ignore its return value.
- Preserve silent middleware suppression and multiple-yield behavior. Telemetry cannot suppress enqueue under ADR-002. Add no private write acknowledgement, receipts, or enqueue-acceptance bookkeeping.

### Flush, removal, and failures

- Collect eligible IDs after each normal public enqueue return.
- When the invocation has no eligible IDs, perform no explicit flush or removal.
- After all per-ID enqueue attempts, perform the backend-specific persistence barrier once. Disk actions call their backend's `flush_tasks` directly. Future adapter actions preserve their own acknowledged-persistence contract; do not branch on `respond_to?(:flush_tasks)` in public objects or a shared retry workflow.
- After that flush returns normally, pass all eligible IDs to one backend `remove_dead_tasks` call.
- Mark every eligible ID `true` after normal removal return, including a zero or reduced removal count.
- Preserve failed and unselected dead records, subject to existing concurrent mutation behavior.
- If flush raises, propagate the original exception. Do not remove any requested dead record through this call.
- If removal raises, raise `DeadTaskRetryError` with the original cause. State that work can exist in both stores.
- A removal failure can occur after the filesystem change. Do not promise that requested dead records remain present.
- After any shared failure, stop the invocation. Pending work can already exist.
- If lookup raises, perform no enqueue or removal through this invocation.
- A raised call returns no result Hash. Do not claim complete partial-outcome reporting after shared failure.
- A public enqueue exception at any point, including after a pending write or during scheduling, retains the original dead record and the original per-ID error. Continue with later IDs; never include the failing source in bulk removal.
- An exception does not prove that no pending copy exists. If scheduling failed after a write, that copy can wait for startup/recovery, currently usually restart. Retaining the DLQ source lets the operator retry, accepting that recovery or a later retry can duplicate work.
- Do not automatically enqueue another copy or move the pending task back into the DLQ because scheduling failed. Scheduling recovery remains separate work.
- Flush and removal failures leave uncertain pending outcomes. Repeating requested IDs can create duplicate work.
- A crash before bulk removal, including after pending publication/persistence, can leave pending copies for every eligible requested ID. The user accepts this duplication window without source reconciliation, a source-dead-ID transfer journal, or cross-store deduplication.
- The duplication window covers the complete invocation, including work persisted by timer flushes.
- Keep the existing periodic flush active. A timer flush during enqueue does not replace the final explicit batch flush.
- Preserve timer dirty state and the existing pending-WAL storage lifecycle.
- If an interruption unwinds the invocation, release locks and descriptors and propagate the original exception without an additional WAL flush, rollback, or source removal during cleanup. Add no manual cancellation API. Ordinary per-ID `StandardError` handling remains unchanged.
- Keep normal timer flushing and the usual final explicit flush when execution reaches it. Cleanup cannot undo pending writes or source removals already performed; interruption during removal can leave some sources removed.
- Execution can start before the invocation completes. This task adds no scheduling barrier or rollback.
- Silent suppression remains eligible for removal and a `true` result. This accepted limitation can lose work.
- Preserve task 07's deferral and existing WAL recovery limitations. Do not promise complete crash-recovery safety.

### Iodine, compatibility, and documentation

- Preserve `Queue#schedule` behavior inside and outside Iodine.
- Outside Iodine, emit one stderr warning after normal enqueue attempts and optional flush, before bulk removal.
- Emit no warning for an invocation without eligible IDs or with a failed explicit flush.
- State that server restart is required. Include the eligible count, without payloads or exception text.
- Silent suppression follows the same warning rule. A warning does not prove pending work exists.
- Nil returns `false` for each distinct ID without enqueue, flush, files, or background work.
- Preserve entry `retry` behavior except for the shared retry lock; preserve collection `delete`, exact `find_by_id`, and standard Enumerable behavior.
- Preserve Ruby 3.3.0 support and existing backend configuration.
- Add YARD for the collection API and any public error documentation affected by this task.
- Document input, results, shared failures, both lock scopes, the 30-second acquisition timeout and existing timeout error, interruption cleanup, nested retry, races, suppression, storage cost, and restart requirements. Q-04 through Q-07 are resolved.
- YARD and operator guidance must explain that enqueue failure retains the original DLQ source and original error even if a pending copy was written. That copy may wait for startup/recovery after scheduling failure; an operator's later retry can create duplicates. Retry adds no automatic replacement enqueue or pending-to-DLQ transfer.
- Use this concise operator guidance: "An enqueue error leaves the original dead task available for retry. A pending copy may already exist, even though enqueue raised. If scheduling failed, that copy may require server restart to recover. Retrying the dead task can create duplicate work." Keep the original exception in the per-ID result Hash and the existing warning timing and count rules. Add no runtime warning for each enqueue error.
- Add a changelog entry for the collection API and retry coordination. Do not change Rake commands in this task.
- Preserve trusted local Marshal assumptions. Add no endpoint, network I/O, authentication mechanism, or payload diagnostic.

## Design

### Mutation actions

Q-07 selects internal `Rage::Deferred::DeadTaskActions` objects for both entry and collection mutations.
A factory selects a backend-specific actions subclass, including `DeadTaskActions::Disk` and `DeadTaskActions::Nil`, using the existing backend object. Public objects initialize or obtain their actions collaborator without constructing a backend or retaining an entry-to-collection reference. Keep backend selection in the factory, not in public mutation methods.
Actions retain only their backend. Entries, IDs, selected records, errors, eligible-ID buffers, descriptors, and other progress belong to each invocation, not to an actions instance. Sharing or retaining an actions object must not retain a traversal or prior operation. Public objects keep their existing backend identity and read/inspection responsibilities; no public actions accessor or configuration is added.

Keep the parent actions class shallow. Share only behavior that is the same for supported backends, including `delete_one(id)` as `backend.remove_dead_tasks(id) == 1`, and `delete_many(ids)` as `0` for empty IDs or one `backend.remove_dead_tasks(ids)` call otherwise. Public collection deletion retains its complete input validation and de-duplication before delegation. These paths perform no lookup, class resolution, context decoding, enqueue, or retry-lock acquisition; Disk deletion uses only the existing short dead-store lock. Preserve Boolean entry deletion, count-based collection deletion, missing-ID behavior, exceptions, and concurrent-mutation uncertainty.

Use separate internal `retry_one(entry)` and `retry_many(ids)` operations. They own the full selection/enqueue/persistence/removal sequence and its warning/error policy. Do not implement individual retry by calling a one-ID batch: their replay inputs and error contracts differ.

- Disk `retry_one(entry)` acquires the shared retry lock before the exact-ID existence check. It discards the lookup record and replays the retained entry's original class and serialized context, even after same-ID replacement. Preserve task 05's Boolean results, propagated errors, warning, removal-error cause, normal-return `true` after zero removal, and frozen inspection cache. The public `DeadTask#retry` delegates this sequence rather than owning it itself.
- Disk `retry_many(ids)` acquires the same retry lock before one fresh multi-ID selection. It replays those selected records, preserves original per-ID errors and ordered results, continues after per-ID failures, and performs at most one direct `flush_tasks` and one bulk removal. It does not loop over public `entry.retry` calls or perform extra per-ID existence scans.
- Nil actions return `false` for individual retry and an ordered Hash of `false` for the requested distinct IDs for batch retry. They perform no enqueue, lock/storage/file access, flush, or background work.

Disk actions invoke Disk-specific low-level storage and retry-lock helpers. The Disk backend owns file opening, `flock`, snapshot/scan I/O, synchronization, compaction, and resource cleanup; it does not resolve application classes, run replay, or own per-ID result orchestration. The actions layer owns the invocation-spanning workflow and requests the lock scope from Disk. Do not require a generic `with_dead_task_retry_lock` API, a no-op lock, or a no-op flush on Nil or future adapters. Persistence and coordination choices belong to each backend-specific actions subclass, without capability checks in a common retry algorithm.

This choice supersedes earlier collection-owned orchestration, the prohibition on a separate actions/service collaborator, the requirement that `DeadTask#retry` itself own the individual sequence, and the ban on entries retaining an actions collaborator. It does not move replay orchestration into the storage backend. Task 05's document and status retain their stage-specific meaning; task 08 changes internal ownership while preserving its observable behavior.

Suggested implementation targets are `lib/rage/deferred/dead_task_actions.rb` for the factory/common parent and `lib/rage/deferred/dead_task_actions/{disk,nil}.rb` for built-in actions. These are internal code locations, not new public APIs or implementation authorization.
The current public configuration supports only Disk and Nil. A future adapter supplies its backend-specific actions while preserving the public contracts; the factory's future extension mechanism is deferred. Add no new registration API, unsupported-backend error, or restriction on read/inspection behavior as part of this decision.

### Selection and replay helpers

Add a private backend multi-ID lookup capability, such as `find_dead_tasks(ids)`.
Disk returns selected records and per-ID decoding errors for all requested IDs.
Nil returns no records.
Keep storage-specific behavior inside backends and mutation orchestration inside their corresponding actions subclasses.
Future adapters follow the same results and shared-failure boundaries under their existing storage contracts.

Handle snapshot file errors separately from record-decoding errors. Snapshot file errors stop the whole retry call.
Rescue specified selected decoding failures only around `String#undump` and `Marshal.load`.
Keep natural record assembly and entry-construction failures separate from operational I/O failures.
Use the existing decoding and retry checks. Do not add a common validator that checks every decoded record's fields and types.

Reuse small private entry helpers for class validation, argument decoding, and public enqueue when practical.
Mark internal cross-object helpers `@private` and keep these helpers out of the supported public API.
Do not add an entry collection reference or public retry options.
Public entry helpers may supply class resolution, mutable context decoding, and public enqueue to actions without taking over lock or persistence orchestration. Do not change public inspection or its successful frozen caches.

### Retry coordination and failures

For Disk, the accepted lock mechanism opens the same stable retry-lock file separately per invocation, obtaining distinct open file descriptions rather than `dup` or a shared descriptor.
For example, `{prefix}dead_tasks_retry.lock` can sit alongside `{prefix}dead_tasks.lock`; the retry filename is illustrative.
Use `File::LOCK_EX | File::LOCK_NB` with Fiber-aware waiting and the fixed 30-second acquisition deadline, and retain/release/close the descriptor through the locked sequence and `ensure` cleanup.
Do not unlink or replace the lock file while holders or contenders can exist.
These Q-05 choices apply equally to application API and Rake callers using the same configured Disk path and prefix, without a process-wide coordination registry. Outside Iodine, ordinary waiting may pause the calling process; inside Iodine it must yield the Fiber.
Long or yielding middleware delays other retries for the same store. A nested retry that cannot acquire the lock fails with `DeadTasksLockTimeout` after 30 seconds; there is no special nesting detection. This bounds that acquisition wait without limiting lock-holding time or total retry duration.
Interruption cleanup releases resources and propagates the primary exception without extra persistence or removal. The existing WAL lifecycle remains unchanged.

Q-04 is resolved: retain the original DLQ source after any public enqueue exception, even when a pending write happened first. Do not add private write acknowledgement, pending/source receipts, or write-candidate tracking.
Public enqueue still returns `nil`; no per-write persistence proof is required to decide which failed calls retain their source.
Additional tasks enqueued by middleware keep their ordinary queue/WAL behavior. Normal-return eligibility and the silent-suppression limitation remain unchanged.
The retained source and a pending copy can coexist. This simpler error path accepts duplicates after recovery or an operator retry; it does not add automatic replacement work or scheduling recovery.
REQ-30's prohibition on acceptance bookkeeping remains in force.

WAL publication/durability protocol and orphan adoption need separately specified future work; this task adds neither that protocol nor a new dependency.

For a nonempty invocation, Disk performs one selection scan regardless of the requested ID count.
Disk also performs at most one explicit flush and one compaction when the invocation has eligible IDs.
The scan and compaction costs depend on dead-store size.
This design does not promise a fixed speed improvement.

For `M` distinct requested IDs, normalized IDs, result entries, selected records, and winner locations each require O(M) slots.
Payload bytes require memory proportional to the combined selected payload size, excluding application-retained objects.
Disk also uses bounded record-scan buffers.
Callers can split IDs into separate invocations to limit memory and the number of pending copies before removal.
Each separate invocation acquires the retry lock and performs its own selection scan, optional `flush_tasks`, and bulk removal. Invocation state remains local to the call; add no write-candidate tracking.

Scanning and compaction run in the calling Fiber under existing filesystem behavior.
The API does not bound request size, scan time, record size, middleware execution time, or total retry-lock hold time.
Do not add background threads or hold the dead-store lock across application code. The separate retry lock intentionally spans that code.
Use Fiber-aware waiting and `ensure` cleanup without masking an exception already in flight; a sole cleanup failure propagates.

## Implementation constraints

- Follow ADR-003 and ADR-004's scoped extension to ADR-001 and ADR-002. Do not implement until the draft decisions are resolved, readiness/publication approved, and implementation separately authorized.
- Do not implement a transaction, journal, reservation, cancellation protocol, or exactly-once guarantee.
- Do not call `entry.retry` in a collection loop; this preserves the repeated-scan and repeated-compaction cost.
- Do not add per-ID fresh lookups after batch selection.
- Do not remove sources after per-ID enqueue failures or to compensate for shared failures. A pending write before the error does not change this rule.
- Do not rewrite public enqueue or add a dedicated retry WAL, enqueue destination routing, adoption, or a scheduling barrier. Preserve ordinary enqueue persistence, storage format, timer policy, unused-feature isolation, and task 07's scope.
- Do not change task 05's individual return values, warning, exception behavior, or snapshot replay semantics.
- Keep backend access private and preserve the identical backend shared with the queue.
- Keep `DeadTaskActions` and backend-specific subclasses internal with `@private` documentation. Do not add actions creation or dispatch to ordinary enqueue or deferred-task execution paths used by applications that never access dead tasks.

## Acceptance criteria

- [ ] TC-01 (AC-13) The API validates all input before work and preserves distinct first-occurrence processing order.
- [ ] TC-02 (AC-13) Normal completion returns the specified per-ID Hash, including missing, duplicate, empty, and Nil cases.
- [ ] TC-03 (AC-13) Per-ID failures preserve source records and original errors while later IDs continue.
- [ ] TC-04 (AC-14) Each nonempty Disk invocation uses one frame-only selection scan and decodes only selected requested winners.
- [ ] TC-05 (AC-14) Each eligible invocation performs one explicit optional flush before one bulk-removal call.
- [ ] TC-06 (AC-14) Timer flushes do not replace the explicit barrier or change timer dirty state.
- [ ] TC-07 (AC-15) Shared failures stop the invocation and preserve the specified cause and uncertainty boundaries.
- [ ] TC-08 (AC-15) Selection races, partial enqueues, suppression, and execution-before-flush follow the documented behavior.
- [ ] TC-09 (AC-15) Batch state stays invocation-local; the retry lock spans application code while the dead-store lock does not.
- [ ] TC-10 (AC-15) Outside Iodine, each eligible invocation warns once after optional flush and performs no server scheduling.
- [ ] TC-11 (AC-13, AC-15) Entry retry joins the shared retry lock; its other behavior and normal enqueue, inspection, deletion, and Enumerable contracts remain compatible.
- [ ] TC-12 (AC-13, AC-14, AC-15) YARD, changelog, and operator guidance describe the new API and its limits, including possible pending copies after enqueue errors, recovery/restart needs, duplicate risk, and unchanged warning timing/count without a new per-error warning.

- [ ] TC-13 (AC-15) Individual and batch retry exclude cooperative callers across processes, Fibers, and same-store backend instances through separately opened descriptors for the stable retry-lock file, acquired before the short dead-store lock, without a shared flag, mutex, or registry. Contention that persists for the fixed 30-second acquisition wait, including nested retry, raises `Rage::Deferred::DeadTasksLockTimeout` before selection or enqueue by that invocation.
- [ ] TC-14 (AC-15) Every exit releases descriptors, coordination, and invocation state without masking the primary failure. Interruption cleanup adds no WAL flush, rollback, or source removal and preserves the normal timer/WAL lifecycle and partial-removal uncertainty.

- [ ] TC-15 (AC-15) `DeadTask` and `DeadTasks` delegate mutations through backend-specific `DeadTaskActions`, with no lock operations, coordination-scope calls/blocks, or persistence-capability checks in public objects. Disk actions own the complete retry workflow using Disk storage helpers; Nil adds no storage work. Actions retain only the identical backend, with invocation-local progress.
- [ ] TC-16 (AC-13, AC-15) Separate individual and batch actions preserve original-entry snapshot replay and individual Boolean/exception behavior versus fresh batch selection and ordered per-ID results. Inspection data and caches remain unchanged.
- [ ] TC-17 (AC-13, AC-15) The shallow shared deletion actions preserve entry Boolean and collection count results, complete-input validation, empty input, missing IDs, single-call bulk removal, and existing errors without acquiring the retry lock.

## Verification

- Derive RSpec coverage independently from these requirements when the user requests the Test Engineer stage.
- Cover complete-input validation, copied mutable IDs, duplicate order, empty input, missing IDs, and Nil.
- Cover large input Arrays, mixed per-ID errors, and invocations without eligible IDs.
- Count complete frame scans, explicit flushes, and compactions. Verify one-batch operation counts instead of elapsed-time thresholds.
- Use unrelated and duplicate records to verify frame-only selection, newest winners, and requested-only decoding.
- Cover each rescued selected decoding exception and its existing unsanitized diagnostic.
- Cover other selected decoding failures, natural record/entry failures, and no older fallback.
- Inject snapshot acquisition, read, seek, cleanup, flush, and removal failures. Verify shared-failure boundaries and cause retention.
- Fail the final flush after multiple enqueue calls. Verify pending copies can exist while all requested dead records remain.
- Fail snapshot lookup before enqueue. Verify that the invocation performs no enqueue, flush, or removal.
- Cover enqueue exceptions before and after queue writes, silent suppression, multiple yields, and mutable replay aliases and cycles.
- Interleave timer flush with enqueue. Verify the explicit final barrier precedes bulk removal.
- Interleave delete and same-ID replacement after selection. Verify recorded replay input and exact-ID removal behavior.
- Exercise nested and Fiber-interleaved calls under the resolved Q-05 policy. Verify isolated invocation state, retry-lock exclusion, and no dead-store lock across application code. A nested retry uses the ordinary timeout without a special error or Fiber-local marker; apply normal enqueue-error handling when that timeout escapes middleware.
- Cover same-store separate backend instances, distinct configured paths/prefixes, process contention including a Rake/API pair, yielding middleware, stable lock-file identity, and descriptor/lock cleanup. This requires no public multi-store support.
- Verify the fixed 30-second acquisition deadline and `Rage::Deferred::DeadTasksLockTimeout` message with a controlled clock/wait mechanism, without real 30-second sleeps. On timeout, verify no selection or enqueue by the waiting invocation and no release of the holder's lock. Verify an acquired invocation is not subject to this deadline while running.
- Interrupt a waiting invocation and an active invocation before flush, and interrupt removal after a partial change. Verify primary-exception propagation, descriptor/lock cleanup, no extra flush/rollback/removal, normal timer behavior, and no promise to restore already removed sources.
- Verify factory selection of Disk/Nil actions with the identical backend for entry and collection callers, including both queue/collection initialization orders. Verify public validation happens before mutation delegation, and actions retain only the backend across nested or Fiber-interleaved calls.
- Verify both public mutation surfaces delegate without backend-type, locking, or `respond_to?(:flush_tasks)` branches. Disk actions call `flush_tasks` directly under the shared retry lock. Nil needs no lock/flush stubs or storage work. Preserve adapter persistence-contract coverage through adapter-specific actions; no future factory registration API is selected here.
- Cover individual entry retry after snapshot cleanup and same-ID replacement: lookup checks existence while replay uses original entry data, errors propagate, zero removal returns `true`, and successful inspection caches remain unchanged. Verify one-ID collection retry still uses fresh selection and per-ID Hash/error behavior.
- Cover shared `delete_one`/`delete_many` behavior, invalid and empty collection input, one bulk compaction, no entry lookup, and deletion during an active retry. Update obsolete no-actions/instance-variable architecture assertions while preserving backend identity and traversal isolation.
- Fail enqueue before a write, after a write, and during scheduling/middleware/telemetry. Verify original per-ID errors, retained failed sources, continued later IDs, and no private acknowledgement or automatic replacement enqueue. When every enqueue raises, verify no explicit flush or removal despite possible pending copies.
- Verify YARD/operator guidance explains that failure can coexist with pending work and that recovery or a later operator retry can duplicate it.
- Verify both retry surfaces use the existing pending WAL and ordinary scheduling, without dedicated retry files or destination routing. Preserve telemetry's inability to suppress enqueue.
- Verify warning counts, suppression warnings, no-warning failures, no new per-enqueue-error warning, Iodine scheduling behavior, and no new payload output.
- Update the existing no-collection-retry assertion to cover the new collection API. Preserve existing entry behavior checks.
- Run focused collection, retry, deletion, and backend specs, then `bundle exec rspec spec/deferred`.
- Run applicable broader checks, RuboCop, and YARD. Report unavailable checks with their reasons.
- For performance evidence, compare entry loops and batch calls on identical temporary stores with identical middleware and payloads.
- Record timings and storage operation counts. Do not claim a fixed speed improvement without measurements.

## Result

- Implementation PR:
