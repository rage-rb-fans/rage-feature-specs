---
status: ready-for-development
---

# Retry multiple dead tasks by ID

## Goal

Let trusted operators retry selected IDs with fewer Disk scans, explicit flushes, and dead-store compactions.

## Context

Task 05 adds immediate entry retry. Each entry performs an exact lookup, public enqueue, optional flush, and removal.
On Disk, repeated entry retry can scan and rewrite the store once per ID.

This task adds a collection operation without a transaction between pending and dead storage.
This task depends on tasks 03 through 05.
Task 07 remains independent future work.
Task 06 retains its separate per-ID Rake workflow.
This task does not change Rake commands.

This task implements REQ-32 through REQ-36 and AC-13 through AC-15.
Follow [ADR-001](../adr/001-public-dead-task-object-model.md), [ADR-002](../adr/002-dead-task-retry-lifecycle.md), and [ADR-003](../adr/003-batch-dead-task-retry.md).
ADR-003 defines a scoped extension to the earlier entry-only retry model.
Task 05 retains its original requirements and lifecycle status.

## Terms

- **Eligible ID** — An ID whose public enqueue call returns normally within the invocation.
- **Shared failure** — A dead-store snapshot, flush, bulk-removal, or cleanup failure that prevents completion of the invocation.

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
- On Disk, capture one fixed complete-record snapshot per invocation under the existing short lock.
- Release the lock before scanning, decoding, or calling application code.
- Scan the snapshot once, sequentially forwards. Retain locations only for requested IDs.
- Select the newest frame-valid record for each exact authoritative outer ID without decoding candidate payloads.
- Decode only selected requested winners. Do not decode unrelated records or use an older duplicate after a selected failure.
- Apply REQ-17's diagnostic and `false` outcome for selected `ArgumentError`, `NameError`, and `TypeError` decoding failures.
- Preserve other selected-payload and entry-construction `StandardError` objects as per-ID results.
- Treat snapshot acquisition, read, seek, and descriptor-cleanup failures as shared failures. Propagate these failures unchanged.
- Finish the lookup and release its descriptor before the invocation calls public enqueue.
- Use each selected record's class and serialized context for replay. Do not perform another per-ID existence scan.
- Selection checks existence at the invocation snapshot boundary. Selection does not reserve records or compare versions.
- A removal after selection does not cancel replay. A replacement after selection does not change replay input.
- Bulk removal can delete a same-ID replacement. Operators must coordinate concurrent mutation operations.
- Preserve task 05's class validation, independent mutable replay, Context accessors, and dedicated context error boundary.
- Preserve frozen public inspection data and successful inspection caches.
- Call the resolved class's normal public `enqueue(*args, **kwargs)` once per usable requested ID.
- Preserve fresh task IDs, reset attempts, current middleware, and current telemetry.
- Keep public `Task.enqueue` unchanged. Ignore its return value.
- Preserve silent suppression and multiple-yield behavior. This task does not detect enqueue acceptance.

### Flush, removal, and failures

- Collect eligible IDs after each normal public enqueue return.
- When the invocation has no eligible IDs, perform no explicit flush or removal.
- After all per-ID enqueue attempts, call optional backend `flush` once when the backend provides that capability.
- After that flush returns normally, pass all eligible IDs to one backend `remove_dead_tasks` call.
- Mark every eligible ID `true` after normal removal return, including a zero or reduced removal count.
- Preserve failed and unselected dead records, subject to existing concurrent mutation behavior.
- If flush raises, propagate the original exception. Do not remove any requested dead record through this call.
- If removal raises, raise `DeadTaskRetryError` with the original cause. State that work can exist in both stores.
- A removal failure can occur after the filesystem change. Do not promise that requested dead records remain present.
- After any shared failure, stop the invocation. Pending work can already exist.
- If lookup raises, perform no enqueue or removal through this invocation.
- A raised call returns no result Hash. Do not claim complete partial-outcome reporting after shared failure.
- A public enqueue exception can occur after pending work exists. Preserve the dead record and report that exception for the ID.
- Flush and removal failures leave uncertain pending outcomes. Repeating requested IDs can create duplicate work.
- A crash before bulk removal can leave pending copies for every eligible requested ID.
- The duplication window covers the complete invocation, including work persisted by timer flushes.
- Keep the existing periodic flush active. A timer flush during enqueue does not replace the final explicit batch flush.
- Preserve timer dirty state and the existing pending-WAL storage lifecycle.
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
- Preserve entry `retry`, collection `delete`, exact `find_by_id`, and standard Enumerable behavior.
- Preserve Ruby 3.3.0 support and existing backend configuration.
- Add YARD for the collection API and any public error documentation affected by this task.
- Document input, results, shared failures, races, suppression, storage cost, and restart requirements.
- Add a changelog entry for the collection API. Do not change Rake commands in this task.
- Preserve trusted local Marshal assumptions. Add no endpoint, network I/O, authentication mechanism, or payload diagnostic.

## Design

The collection owns batch orchestration. Each invocation retains its own input, results, and selected records.
The memoized collection retains no invocation state after the call.
Overlapping or nested invocations do not share progress or eligible-ID buffers.

Add a private backend multi-ID lookup capability, such as `find_dead_tasks(ids)`.
Disk returns selected records and per-ID decoding errors for all requested IDs.
Nil returns no records.
The collection must not branch on the backend class.
Future adapters follow the same results and shared-failure boundaries under their existing storage contracts.

Keep snapshot I/O outside the per-record decoding rescue boundary.
Rescue specified selected decoding failures only around `String#undump` and `Marshal.load`.
Keep natural record assembly and entry-construction failures separate from operational I/O failures.
Do not add a shared payload schema validator.

Reuse small private entry helpers for class validation, argument decoding, and public enqueue when practical.
Mark internal cross-object helpers `@private` and keep these helpers out of the supported public API.
Do not add a retry service, entry collection reference, backend retry orchestrator, or public retry options.
Keep `DeadTask#retry` responsible for its existing individual sequence and error behavior.

For a nonempty invocation, Disk performs one selection scan regardless of the requested ID count.
Disk also performs at most one explicit flush and one compaction when the invocation has eligible IDs.
The scan and compaction costs depend on dead-store size.
This design does not promise a fixed speed improvement.

For `M` distinct requested IDs, normalized IDs, result entries, selected records, and winner locations each require O(M) slots.
Payload bytes require memory proportional to the combined selected payload size, excluding application-retained objects.
Disk also uses bounded record-scan buffers.
Callers can split IDs into separate invocations to limit memory and the number of pending copies before removal.
Each separate invocation performs its own selection scan, optional flush, and bulk removal.

Scanning and compaction run in the calling Fiber under existing filesystem behavior.
The API does not bound request size, scan time, record size, or middleware execution time.
Do not add background threads or hold a storage lock across application code.
Use the existing Fiber-aware lock and `ensure` cleanup rules.

## Implementation constraints

- Follow ADR-003's scoped extension to ADR-001 and ADR-002.
- Do not implement a transaction, journal, reservation, cancellation protocol, or exactly-once guarantee.
- Do not call `entry.retry` in a collection loop; this preserves the repeated-scan and repeated-compaction cost.
- Do not add per-ID fresh lookups after batch selection.
- Do not remove failed IDs to compensate for shared failures.
- Do not change public enqueue, ordinary enqueue persistence, storage format, timer policy, or task 07's scope.
- Do not change task 05's individual return values, warning, exception behavior, or snapshot replay semantics.
- Keep backend access private and preserve the identical backend shared with the queue.

## Acceptance criteria

- [ ] TC-01 (AC-13) The API validates all input before work and preserves distinct first-occurrence processing order.
- [ ] TC-02 (AC-13) Normal completion returns the specified per-ID Hash, including missing, duplicate, empty, and Nil cases.
- [ ] TC-03 (AC-13) Per-ID failures preserve source records and original errors while later IDs continue.
- [ ] TC-04 (AC-14) Each nonempty Disk invocation uses one frame-only selection scan and decodes only selected requested winners.
- [ ] TC-05 (AC-14) Each eligible invocation performs one explicit optional flush before one bulk-removal call.
- [ ] TC-06 (AC-14) Timer flushes do not replace the explicit barrier or change timer dirty state.
- [ ] TC-07 (AC-15) Shared failures stop the invocation and preserve the specified cause and uncertainty boundaries.
- [ ] TC-08 (AC-15) Selection races, partial enqueues, suppression, and execution-before-flush follow the documented behavior.
- [ ] TC-09 (AC-15) Batch state stays invocation-local, and application code runs without the dead-store lock.
- [ ] TC-10 (AC-15) Outside Iodine, each eligible invocation warns once after optional flush and performs no server scheduling.
- [ ] TC-11 (AC-13, AC-15) Entry retry, normal enqueue, inspection, deletion, and Enumerable contracts remain compatible.
- [ ] TC-12 (AC-13, AC-14, AC-15) YARD, changelog, and operator guidance describe the new API and its limits.

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
- Exercise nested and Fiber-interleaved collection calls. Verify independent progress and no lock across application code.
- Verify warning counts, suppression warnings, no-warning failures, Iodine scheduling behavior, and no new payload output.
- Update the existing no-collection-retry assertion to cover the new collection API. Preserve existing entry behavior checks.
- Run focused collection, retry, deletion, and backend specs, then `bundle exec rspec spec/deferred`.
- Run applicable broader checks, RuboCop, and YARD. Report unavailable checks with their reasons.
- For performance evidence, compare entry loops and batch calls on identical temporary stores with identical middleware and payloads.
- Record timings and storage operation counts. Do not claim a fixed speed improvement without measurements.

## Result

- Implementation PR:
