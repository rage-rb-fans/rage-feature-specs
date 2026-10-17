---
status: draft
---

# Add Rake operations and documentation

## Goal

Give shell-only operators supported list, show, retry, and delete commands, and document the complete dead-task lifecycle and its operational risks.

## Context

Tasks 02 through 05 provide the complete Ruby API for listing, inspecting, deleting, and retrying dead tasks. Rage loads framework Rake files from `lib/rage/tasks/**/*.rake` through `Rage::Tasks`, so command wrappers can delegate without duplicating storage or replay logic.

This task follows completion of tasks 02 through 05. It implements REQ-21 through REQ-23 and the documentation portions of REQ-03, REQ-15, REQ-16, REQ-19, REQ-20, and REQ-24 through REQ-30, plus AC-09 and AC-10. Follow [ADR-001](../adr/001-public-dead-task-object-model.md) and [ADR-002](../adr/002-dead-task-retry-lifecycle.md).

## Requirements

- Add documented Rake tasks named `deferred:dead_tasks:list`, `deferred:dead_tasks:show`, `deferred:dead_tasks:retry`, and `deferred:dead_tasks:delete`.
- Boot the Rage application environment before accessing `Rage::Deferred.dead_tasks`. Keep all selection, decoding, retry, and deletion behavior in the Ruby API from tasks 02 through 05.
- `list` accepts an optional positive `LIMIT` environment variable with a documented default. Process logical winners oldest-first, matching collection order. Print ID, task class, attempts, enqueue and failure times, exception class, and one bounded single-line exception-message summary. Never print args, kwargs, backtrace, opaque context, or raw record bytes as row data. Task 02 can print its unsanitized selected decoding diagnostic between rows. A rescued winner is skipped and does not count toward `LIMIT`.
- `show` requires exactly one non-empty `ID` and delegates exact lookup to `DeadTasks#find_by_id`. Print all public entry fields, including args/kwargs and backtrace. Print a sensitive-data warning before the full entry fields. A missing record or invalid input fails without a Ruby backtrace during normal CLI use. For a rescued selected failure, lookup prints its diagnostic and returns `nil`. `show` then reports the ID as missing.
- `retry` and `delete` require `IDS` as a comma-separated list. Trim whitespace, reject an empty list, de-duplicate IDs while preserving input order, process every ID independently, and print one outcome per ID. Retry totals use succeeded/not-retried/failed. Delete totals retain succeeded/missing/failed.
- Retry/delete continue after an individual failure. After the summary, fail the Rake invocation if any ID failed. Missing IDs are not exceptions. A rescued selected failure during retry prints the task 03 diagnostic and reports that ID as not retried. Rake helpers must not add stored payloads to failure messages.
- Report retry `false` as not retried. The result can mean missing, stale, unreadable, or middleware-suppressed work. Do not claim a reason that the Boolean cannot identify. Treat not-retried results as non-exception outcomes, matching the existing missing-ID exit policy. Suppressed retry performs no flush, removal, or restart-required warning.
- Document public `Task.enqueue` acceptance Booleans and the replacement of unconditional `nil`. Explain the caller compatibility change and unchanged signature. Document independent mutable replay, preserved aliases/cycles under normal enqueue splats, immutable inspection values, and dedicated decoding errors.
- Document entry-owned enqueue, optional backend flush, and removal. Disk flush confirms WAL durability before the entry removes the dead record. An adapter omits flush only when successful enqueue guarantees durable, recoverable pending work. Flush failure preserves the record but leaves the pending outcome uncertain. Execution can start before synchronization completes. Removal failure reports possible duplication and preserves its cause.
- An accepted, persisted retry without active Iodine prints the restart-required warning prominently in addition to the final outcome. Do not claim that a currently running server will discover the out-of-band WAL.
- Nil backend list is empty. Show/delete report missing IDs; retry reports not retried. No command creates persistence files.
- Document private shared backend ownership and entry actions after snapshot cleanup. State that retry refreshes current record data without changing the original inspection snapshot. Keep collection retry as a thin lookup-and-entry convenience method. Do not describe an action handler or a second retry implementation.
- Add RSpec coverage for task loading, argument validation, output redaction/full-output boundaries, partial outcomes, final process/task failure status, and Disk/Nil delegation.
- Add or update YARD and deferred user documentation. Cover dead-letter triggers, retention, Disk and Nil behavior, collection order, and `first(20)` skip semantics. Include Ruby and Rake examples, field meanings, shared backend identity, traversal laziness, and independent traversal executions. Cover physical-corruption skipping and newest-frame-valid selection without shared schema validation. Document the exact selected `ArgumentError`, `NameError`, and `TypeError` behavior. Include the standard-output diagnostic, enumeration skip, lookup `nil`, retry `false`, stored-record preservation, and no older fallback. State that the diagnostic ID and error message are not sanitized or truncated. Document unchanged propagation for other selected failures and operational errors. Also cover incompatible contexts, fresh retry, races, restart, filesystem assumptions, storage versions, performance, adapter complexity, and sensitive-data risks.
- Document iterator ownership with examples. State that no-block `DeadTasks#each` returns a normal lazy Ruby `Enumerator` with standard Ruby 3.3 behavior and no Rage-specific `close`. Present block-based enumeration as the preferred ordinary form because exhaustion, `break`, another non-local exit, and exceptions unwind through automatic cleanup. State that normally unwound terminal Enumerable operations receive the same cleanup.
- State that `Rage::Deferred.dead_tasks` returns the same memoized collection object, but each block traversal and ordinary top-level Enumerable operation owns an independent traversal execution and snapshot. Every no-block `each` call returns a distinct Enumerator. Explain that overlapping, nested, and same-process Fiber-interleaved collection traversals do not share cursors or cleanup state, while avoiding any broader claim of thread safety.
- Explain that creating the memoized collection may initialize the configured backend through `Rage::Deferred.__backend`, including its normal file-creation side effects, and that the collection retains the same backend object used by the queue. Creating or retrieving the collection and calling the argument-free no-block `each` without advancing its Enumerator do not open a DLQ snapshot descriptor or scan, decode, or preload DLQ records. The unadvanced Enumerator starts no DLQ traversal.
- Explain that standard Ruby internal and external Enumerator behavior applies. Mixing `next` or `peek` with block `enumerator.each`, Enumerable methods, or derived iterators on the same Enumerator is not synchronized by Rage and may create separate cursors and snapshots. Recommend separate collection operations when independent traversal is intended. Do not promise shared lookahead, generation, or cross-surface exhaustion behavior.
- Explain that malformed frames, invalid CRCs, and an excluded incomplete tail are silently skipped as physical corruption. Enumeration and exact lookup perform complete frame-only winner-selection scans. Exact lookup remembers the newest frame-valid matching location and attempts it after the scan. A newer frame-invalid duplicate permits an older frame-valid record to win. A rescued selected failure prints the specified diagnostic and never permits older fallback. Enumeration skips that logical task. Lookup returns `nil`. Other selected decoding and entry-construction failures propagate unchanged. Physical-record skips do not produce output.
- Explain that Rage exclusively controls marshaling and writing this versioned local store, so listing, lookup, and retry do not impose a shared top-level payload schema/type-validation layer. Marshal payloads are trusted local application data rather than an interchange format: physical frame and CRC validation detect storage corruption but do not make an untrusted Marshal payload safe to decode.
- Document that snapshot-acquisition `Rage::Deferred::DeadTasksLockTimeout` and filesystem open/read/seek/cleanup failures propagate unchanged from the lazy traversal or lookup operation. Do not present them as skipped corruption or as the dedicated opaque-context deserialization error. State that cleanup does not mask an exception already in flight and that a sole storage cleanup failure still propagates.
- Explain the partial-iteration limitation. A retained or abandoned external, derived, or lazy Enumerator may keep its snapshot descriptor, complete winner index, and an old unlinked inode until that traversal execution exhausts, unwinds, or is garbage-collected. Garbage collection and optional finalizer behavior are best effort only, and documentation must not imply deterministic timing. If prompt cleanup matters, use block traversal or a normally unwound terminal Enumerable operation. Identify deterministic cancellation, such as a separately named closeable cursor, as possible future work rather than a current API.
- Document the performance boundary precisely. Snapshot setup holds the lock only while opening the file and finding the complete-record boundary. Enumeration then frame-validates the complete snapshot before its first yield. It retains O(distinct frame-valid outer IDs) winner-location metadata. It processes winners oldest-first, one at a time. Traversal retains at most the current decoded record beyond bounded line storage. Caller-retained entries are outside this bound. The calling Fiber performs this synchronous work. `first` and `take` cannot avoid selection. Rescued failures can make them attempt more winners than the number of returned entries. Disk exact lookup retains only its newest matching payload location. Future adapters may use indexes. The public API does not promise uniform complexity. Repeated immediate mutation can approach O(N²) storage work. Present entry actions as small-set conveniences.
- For large deletion sets, show `filter_map` collecting exact ID Strings followed by one `dead_tasks.delete(ids)` call. Pass the flat Array as the only argument. State that this retains O(selected IDs) additional accumulator memory on top of traversal's complete winner index, does not retain a collection of `DeadTask` objects, and lets Disk perform one locked compaction. State that entries can become stale between selection and deletion and retain the documented missing-ID behavior.
- State plainly that this feature has no collection-level or optimized bulk-retry primitive. The multi-ID Rake command invokes single-ID retry independently. A large retry run performs necessary per-task enqueues plus repeated dead-store removals/compactions, must be coordinated by operators, and retains at-least-once duplicate risk. Do not tell operators to use a nonexistent optimized bulk-retry path; identify optimized large-set retry as future work.
- Add a changelog entry for `dead_tasks`, Rake commands, and the public enqueue Boolean return change.

## Design

Implement thin Rake tasks plus private formatting/parsing helpers that accept plain Ruby objects and are directly specable. Use standard Rake failure mechanisms only after all requested IDs have been processed. Human-readable output is the initial interface; no stable JSON schema is introduced by this task.

## Implementation constraints

- Do not bypass or duplicate the Ruby APIs completed by tasks 02 through 05.
- Do not use `Enumerable#find` as exact-ID lookup or add backend-specific lookup logic to Rake tasks.
- Do not deserialize args/kwargs for `list`.
- Do not use `eval`, invoke a shell, or interpolate IDs into executable code.
- Do not add HTTP routes, authentication, orphan-WAL polling, automatic retention, or new backend configuration.
- Do not expose raw Marshal bytes.
- Preserve the existing `Rage::Tasks` loading contract and Ruby 3.3.0 compatibility.
- Preserve the draft's unresolved Rake namespace/input choice until the user accepts it; implement the accepted interface without changing command semantics.

## Acceptance criteria

- [ ] All four commands load through the normal Rage task loader and delegate to the completed Ruby APIs, with `show` using `find_by_id` rather than Enumerable `find`.
- [ ] List row output is bounded and excludes stored payload fields. Show output is complete and explicitly marked sensitive. Both commands preserve the unsanitized selected decoding diagnostic from the Ruby API.
- [ ] Retry/delete validate and de-duplicate input, report every outcome, and fail after partial errors. Retry false outcomes report not retried without an invented reason.
- [ ] Accepted, persisted out-of-process retry states that restart is required. Suppressed retry emits no restart-required warning.
- [ ] Disk and Nil command behavior matches the Ruby API.
- [ ] YARD, user docs, and changelog cover the complete lifecycle and public APIs. They explain traversal, lookup, cleanup, performance, compatibility, concurrency, and mutation costs. They document silent physical-corruption handling and exact selected-payload behavior. They warn that the standard-output diagnostic includes an unsanitized ID and error message. They distinguish that behavior from opaque-context and operational failures.

## Verification

- Add focused specs for the Rake definitions and formatting/parsing helpers without shelling out where direct invocation suffices.
- Add an integration-style task-loading check in the test application for command names and environment boot behavior.
- Cover oldest-first list row bounds, show warnings, malformed input, duplicate IDs, and missing IDs.
- Cover partial failures, final status, Nil behavior, and restart messaging. Verify suppressed retry is not retried, retains its record, and emits no restart-required warning.
- Cover each rescued selected failure class for list, show, and retry. Verify the exact diagnostic, enumeration skip, show missing, retry not-retried, continuation, totals, and stored-record preservation.
- Verify that the diagnostic can contain an unsanitized exception message. Do not claim full-output redaction when that diagnostic occurs.
- Verify documentation examples that use one bulk deletion call without retaining `DeadTask` entries.
- Verify `show` delegates to `find_by_id`, and documentation examples never use `find(id)` for exact lookup while accurately demonstrating standard predicate and fallback use of inherited `find`/`detect`.
- Verify documentation examples prefer block iteration and ordinary terminal Enumerable operations when prompt cleanup matters. Verify they do not call or promise a Rage-specific Enumerator `close` method.
- Verify documentation explains standard mixed internal/external Enumerator behavior, recommends separate collection operations for independent traversal, and does not promise a shared cursor, lookahead, generation, or cross-surface lifecycle.
- Verify documentation examples distinguish the memoized collection from the distinct Enumerator returned by each `each` call and do not imply general thread safety.
- Verify documentation states that collection creation may resolve and initialize `Rage::Deferred.__backend`, including its normal file-creation side effects, and that the collection retains that same object for use with the queue. Verify that collection access and an unadvanced argument-free no-block Enumerator do not open a DLQ snapshot descriptor or scan, decode, or preload DLQ records, and that the unadvanced Enumerator invokes no DLQ traversal method.
- Verify documentation states that public `each` accepts no arguments and attempts selected winners one at a time. Verify documented complete frame selection and silent physical-corruption skipping. Verify newest-frame-valid selection without shared schema validation. Verify exact rescued selected-failure behavior and unchanged propagation for other selected failures. Verify the trusted-Marshal boundary, partial-iterator retention, and operational-error propagation.
- Run the focused task specs, the broader deferred specs, applicable CLI/task-loader specs, RuboCop for changed files, and documentation generation/link checks available in the repository.
- Report any unavailable website-documentation checks honestly if the website source is maintained outside this repository.

## Result

- Implementation PR:
