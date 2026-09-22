---
status: todo
---

# Add Rake operations and documentation

## Goal

Give shell-only operators supported list, show, retry, and delete commands, and document the complete dead-task lifecycle and its operational risks.

## Context

Tasks 02 through 05 provide the complete Ruby API for listing, inspecting, deleting, and retrying dead tasks. Rage loads framework Rake files from `lib/rage/tasks/**/*.rake` through `Rage::Tasks`, so command wrappers can delegate without duplicating storage or replay logic.

This task follows completion of tasks 02 through 05. It implements REQ-21 through REQ-23 and the documentation portions of REQ-03, REQ-15, REQ-16, REQ-19, REQ-20, and REQ-24 through REQ-29, plus AC-09 and AC-10. Follow [ADR-001](../adr/001-public-dead-task-object-model.md) and [ADR-002](../adr/002-dead-task-retry-lifecycle.md).

## Requirements

- Add documented Rake tasks named `deferred:dead_tasks:list`, `deferred:dead_tasks:show`, `deferred:dead_tasks:retry`, and `deferred:dead_tasks:delete`.
- Boot the Rage application environment before accessing `Rage::Deferred.dead_tasks`. Keep all selection, decoding, retry, and deletion behavior in the Ruby API from tasks 02 through 05.
- `list` accepts an optional positive `LIMIT` environment variable with a documented default. Print newest-first rows containing only ID, task class, attempts, enqueue/failure times, exception class, and a single-line bounded exception-message summary. Never print args, kwargs, backtrace, opaque context, or raw record bytes.
- `show` requires exactly one non-empty `ID` and delegates exact lookup to `DeadTasks#find_by_id`. Print all public entry fields, including args/kwargs and backtrace, preceded by a warning that the output may contain sensitive application data. A missing record or invalid input fails the task without a Ruby backtrace during normal CLI use.
- `retry` and `delete` require `IDS` as a comma-separated list. Trim whitespace, reject an empty list, de-duplicate IDs while preserving input order, process every ID independently, and print one outcome per ID plus final succeeded/missing/failed totals.
- Retry/delete continue after an individual failure, then fail the Rake invocation after the summary if any ID failed. Missing IDs are reported as missing and are not exceptions. Do not print stored payloads in failure messages.
- A retry run without active Iodine prints the restart-required warning prominently in addition to the final outcome. Do not claim that a currently running server will discover the out-of-band WAL.
- Nil backend list prints an empty result; show/retry/delete report IDs as missing and do not create persistence files.
- Add RSpec coverage for task loading, argument validation, output redaction/full-output boundaries, partial outcomes, final process/task failure status, and Disk/Nil delegation.
- Add or update YARD/deferred user documentation with: dead-letter triggers; no retention; Disk versus Nil behavior; Ruby examples using `find_by_id`; unchanged Enumerable `find`/`detect` predicate, fallback, and no-block semantics; lazy backend resolution and shared queue/collection backend identity; immediate side-effect-free `batch_size` validation; memoized collection identity with independent traversal executions; Rake examples; field meanings; silent skipping of corrupt or schema-invalid top-level records and an incomplete tail; incompatible-context behavior; invalid-newer-duplicate fallback; unchanged lock/filesystem error propagation; fresh retry semantics; at-least-once duplicate/race risks; restart requirement; local-filesystem assumptions; storage-version behavior; fixed-boundary lazy reverse traversal and batching behavior; backend-neutral result shape with adapter-specific lookup complexity; and sensitive-data warnings.
- Document iterator ownership with examples. State that no-block `DeadTasks#each` returns a normal lazy Ruby `Enumerator` with standard Ruby 3.3 behavior and no Rage-specific `close`. Present block-based enumeration as the preferred ordinary form because exhaustion, `break`, another non-local exit, and exceptions unwind through automatic cleanup. State that normally unwound terminal Enumerable operations receive the same cleanup.
- State that `Rage::Deferred.dead_tasks` returns the same memoized collection object, but each block traversal and ordinary top-level Enumerable operation owns an independent traversal execution and snapshot. Every no-block `each` call returns a distinct Enumerator. Explain that overlapping, nested, and same-process Fiber-interleaved collection traversals do not share cursors or cleanup state, while avoiding any broader claim of thread safety.
- Explain that creating or retrieving the memoized collection does not initialize the configured backend. Invalid `batch_size` and valid unadvanced no-block `each` calls also leave it unresolved and do not create Disk files. The first operation that needs storage resolves through `Rage::Deferred.__backend`, so the collection and queue share the same backend object regardless of which initializes it first.
- Explain that standard Ruby internal and external Enumerator behavior applies. Mixing `next` or `peek` with block `enumerator.each`, Enumerable methods, or derived iterators on the same Enumerator is not synchronized by Rage and may create separate cursors and snapshots. Recommend separate collection operations when independent traversal is intended. Do not promise shared lookahead, generation, or cross-surface exhaustion behavior.
- Explain that invalid physical records and an excluded incomplete tail are silently skipped as recoverable corruption rather than treated as operational failures. Document that lazy traversal inspects invalid records only when it reaches them and that exact lookup does not scan older records after finding a fully valid match. State that neither operation emits logging or terminal output for skipped records or fragments.
- Document that snapshot-acquisition `Rage::Deferred::DeadTasksLockTimeout` and filesystem open/read/seek/cleanup failures propagate unchanged from the lazy traversal or lookup operation. Do not present them as skipped corruption or as the dedicated opaque-context deserialization error. State that cleanup does not mask an exception already in flight and that a sole storage cleanup failure still propagates.
- Explain the partial-iteration limitation. A retained or abandoned external, derived, or lazy Enumerator may keep its snapshot descriptor and an old unlinked inode until that traversal execution exhausts, unwinds, or is garbage-collected. Garbage collection and optional finalizer behavior are best effort only, and documentation must not imply deterministic timing. If prompt cleanup matters, use block traversal or a normally unwound terminal Enumerable operation. Identify deterministic cancellation, such as a separately named closeable cursor, as possible future work rather than a current API.
- Document the performance boundary precisely: snapshot setup holds the permanent lock only while opening the live file and finding the last complete-record boundary; reverse reading and top-level decoding then happen lazily in the caller's Fiber without the lock. A complete traversal still reads the captured store and retains O(distinct valid IDs encountered) `seen_ids` metadata even though decoded payloads are batch-bounded, while stopping early avoids processing untouched older batches. Disk `find_by_id` may require a worst-case reverse scan; future adapters may use indexed lookup, and the public API does not promise uniform complexity. Repeated immediate `entry.delete`/`entry.retry` operations can each trigger a full Disk removal compaction and approach O(N²) aggregate storage work. Present entry actions and `each(&:delete)`/`each(&:retry)` as supported, mutation-safe conveniences for individual entries or small sets, not as the recommended large-set workflow.
- For large deletion sets, show `filter_map` collecting exact ID Strings followed by one `dead_tasks.delete(ids)` call that passes the array as a nested argument. State that this retains O(selected IDs) additional accumulator memory on top of traversal's incremental `seen_ids` set, does not retain a collection of `DeadTask` objects, and lets Disk perform one locked compaction. State that entries can become stale between selection and deletion and retain the documented missing-ID behavior.
- State plainly that this feature has no collection-level or optimized bulk-retry primitive. The multi-ID Rake command invokes single-ID retry independently. A large retry run performs necessary per-task enqueues plus repeated dead-store removals/compactions, must be coordinated by operators, and retains at-least-once duplicate risk. Do not tell operators to use a nonexistent optimized bulk-retry path; identify optimized large-set retry as future work.
- Add a changelog entry for the public `dead_tasks` API and Rake commands.

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
- [ ] List output is bounded and payload-redacted; show output is complete and explicitly marked sensitive.
- [ ] Retry/delete validate and de-duplicate input, report every outcome, and fail after partial errors.
- [ ] Out-of-process retry unambiguously tells the operator that restart is required.
- [ ] Disk and Nil command behavior matches the Ruby API.
- [ ] YARD/user docs and changelog cover lifecycle, `find_by_id`, preserved Enumerable `find`/`detect`, lazy backend resolution with one shared queue/collection backend object, immediate batch validation, memoized wrapper identity with independent traversal executions, standard Ruby Enumerator behavior without a custom `close`, backend-neutral lookup results versus adapter-specific complexity, fixed-boundary lazy reverse traversal, batching versus complete-traversal `seen_ids` memory, automatic cleanup for block and normally unwound terminal operations, partial external/lazy resource-retention limitations, silent recoverable-corruption handling, unchanged operational-error propagation, invalid-duplicate fallback, security, compatibility, concurrency, repeated-mutation complexity, the large-set bulk-delete pattern, the absence of bulk retry, and other operational constraints.

## Verification

- Add focused specs for the Rake definitions and formatting/parsing helpers without shelling out where direct invocation suffices.
- Add an integration-style task-loading check in the test application for command names and environment boot behavior.
- Cover list output bounds/redaction, show warnings/full details, malformed input, duplicate/missing IDs, partial mutation failures, final status, Nil behavior, restart messaging, and documentation examples that use one bulk deletion call without retaining `DeadTask` entries.
- Verify `show` delegates to `find_by_id`, and documentation examples never use `find(id)` for exact lookup while accurately demonstrating standard predicate and fallback use of inherited `find`/`detect`.
- Verify documentation examples prefer block iteration and ordinary terminal Enumerable operations when prompt cleanup matters. Verify they do not call or promise a Rage-specific Enumerator `close` method.
- Verify documentation explains standard mixed internal/external Enumerator behavior, recommends separate collection operations for independent traversal, and does not promise a shared cursor, lookahead, generation, or cross-surface lifecycle.
- Verify documentation examples distinguish the memoized collection from the distinct Enumerator returned by each `each` call and do not imply general thread safety.
- Verify documentation states that collection access, invalid `batch_size`, and a valid unadvanced no-block Enumerator do not resolve the backend or create Disk files, while the first storage operation resolves through `Rage::Deferred.__backend` and shares that object with the queue.
- Verify documentation states that invalid `batch_size` raises directly from `each` without store access; invalid records and incomplete tails are skipped silently without eager diagnostic scans; abandoned partial iterators have no deterministic descriptor cleanup guarantee; and lock/filesystem failures propagate rather than being treated as recoverable corruption.
- Run the focused task specs, the broader deferred specs, applicable CLI/task-loader specs, RuboCop for changed files, and documentation generation/link checks available in the repository.
- Report any unavailable website-documentation checks honestly if the website source is maintained outside this repository.

## Result

- Implementation PR:
