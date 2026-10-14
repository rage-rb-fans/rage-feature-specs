---
title: Dead letter queue for deferred tasks
---

# Dead letter queue for deferred tasks

## Motivation

Deferred tasks already have an at-least-once pending-task WAL, but operators also need a durable and supported way to investigate and recover work that Rage has stopped retrying. The dead-tasks feature records the final failure, exposes it through Ruby and command-line operator interfaces, and lets an operator retry or permanently delete the work.

Durable storage and the pending-to-dead handoff are already implemented by task 01. The remaining work turns that storage into a public operator feature without adding overhead to successful deferred-task execution.

## Proposed behavior

### Failure lifecycle and retention

- REQ-01: A deferred task becomes a dead task when its retry budget is exhausted or when `retry_interval` returns `false` or `nil`. A task that succeeds or still has a retry interval never enters the dead-tasks store.
- REQ-02: With the disk backend, Rage durably adds the dead-task record before removing the pending WAL record. A failed dead-task write leaves the pending record recoverable. A crash between those writes may leave the same logical task in both stores; the dead-tasks store selects the newest frame-valid record for a duplicated authoritative outer task ID. A newer frame-valid record hides every older record with that outer ID even if its selected Marshal payload later cannot be decoded or used as a dead-task record.
- REQ-03: Dead tasks have no automatic retention period, maximum age, or drop-oldest policy. They remain until an operator retries or deletes them. Documentation must warn that this makes disk use unbounded when the store is not maintained.
- REQ-04: With `config.deferred.backend = nil`, failures remain non-persistent. The public collection is empty, lookup returns `nil`, and mutation of a missing ID reports that no record was changed; Rage does not create dead-task files.

Task 01 defines the canonical disk format and crash guarantees. Its version remains in the filename (`{prefix}dead_tasks-{STORAGE_VERSION}`), not in each record, and its permanent lock file coordinates workers and processes.

### Ruby API

The proposed public name is `dead_tasks`, as selected in the issue discussion:

```ruby
dead_tasks = Rage::Deferred.dead_tasks

dead_tasks.each do |entry|
  puts "#{entry.id}: #{entry.task_class} (#{entry.exception_class})"
end

entry = dead_tasks.find_by_id("1735689600-12345-1")
entry.retry
entry.delete

# Enumerable search keeps its standard Ruby meaning.
mailer_failure = dead_tasks.find { |dead_task| dead_task.task_class == "MailerTask" }
fallback = -> { :not_found }
result = dead_tasks.detect(fallback) { |dead_task| dead_task.attempts > 10 }

# Entry actions remain mutation-safe during traversal and are convenient for
# individual entries or small sets. They are not the large-set maintenance path.
dead_tasks.each(&:retry)
dead_tasks.each(&:delete)

# Prefer one bulk delete for a large selected set. This accumulates IDs, not
# DeadTask entries, and Disk performs one locked compaction.
ids = dead_tasks.filter_map { |dead_task| dead_task.id if removable?(dead_task) }
dead_tasks.delete(ids)
```

- REQ-05: `Rage::Deferred.dead_tasks` returns a memoized `Rage::Deferred::DeadTasks` collection, so repeated accessor calls return the same wrapper object. When creating the wrapper, the accessor obtains the shared memoized backend through `Rage::Deferred.__backend` and passes that backend object directly to the collection; the collection must never construct a backend independently. This makes the collection and queue use the identical backend object whether the queue or collection reaches the backend first.

  The shared wrapper includes `Enumerable` and is stateless with respect to traversal. It may retain the shared backend, immutable collaborators or configuration, and later collection-level mutation behavior. It must not retain snapshot descriptors or boundaries, scan cursors, winner indexes, decoded records, current Enumerators, or a traversal registry. Obtaining the wrapper may initialize the backend, including the backend's normal initialization side effects, but it does not open a DLQ snapshot descriptor or scan, decode, or preload DLQ records. Calling `each` without a block likewise performs no snapshot work and returns a new normal Ruby `Enumerator`, with standard Ruby 3.3 behavior and no Rage-specific `close` method. A `DeadTask` entry never retains the backend.
- REQ-06: Argument-free `each` processes selected winning records oldest-first according to their physical positions. For a duplicated authoritative outer ID, the newest frame-valid physical record is the winner. Its position determines that logical task's place in the order. After winner selection, Disk decodes selected records one at a time immediately before possible yields.

  If selected top-level decoding raises `ArgumentError`, `NameError`, or `TypeError`, Disk prints the diagnostic defined by REQ-17. Enumeration then skips that logical task and continues with the next winner. Consequently, `dead_tasks.first(20)` returns up to 20 oldest entries that enumeration can yield. Enumeration can attempt more than 20 winners to yield 20 entries. Other selected-payload or entry-construction failures propagate and stop that traversal.

  Each block traversal and each ordinary top-level Enumerable operation uses one traversal execution. When that execution starts, the retained backend establishes its snapshot. That execution owns its snapshot, cursor, descriptor, winner index, and transient current decoded record. Separate calls such as `find`, `map`, `count`, or `each` start separate executions and snapshots. Overlapping, nested, or same-process Fiber-interleaved collection traversals do not share this state. An append after one traversal starts is excluded from that snapshot but may appear in a traversal that starts later.

  A no-block Enumerator follows standard Ruby internal and external iteration behavior. Rage does not synchronize `next` or `peek` with a later block `enumerator.each`, derived iterators, or other ways of consuming that same Enumerator. Those forms may use separate Ruby iteration executions, cursors, and snapshots. A normally consumed lazy pipeline uses its underlying traversal. This feature adds no broader thread-safety guarantee beyond Rage's existing Iodine/Fiber model.
- REQ-07: When traversal starts, Disk briefly holds the permanent dead-task lock. Disk opens the live file and records its last complete-record boundary. Disk then releases the lock.

  Before yielding, Disk reads and frame-validates each complete record through that boundary. Disk scans sequentially forwards from byte zero. Disk builds an insertion-ordered winner index without inspecting the top-level payload. The scan does not call `Marshal.load`.

  For a repeated frame-valid authoritative outer ID, Disk deletes the existing index entry. Disk then reinserts that ID with the newer payload location. The reinserted ID moves to its newest winning physical position. A frame-invalid duplicate does not change the existing winner.

  Disk visits winning locations in insertion order. Disk seeks, reads, and attempts each winner before proceeding. Disk wraps and yields the record unless REQ-17 requires a skip. An error can also stop traversal.

  Index memory is proportional to the distinct frame-valid outer IDs. Beyond bounded line storage, traversal retains at most the current decoded record. Caller-retained entries are outside this internal bound.

  Early termination cannot avoid the complete selection scan. Early termination prevents later winners from being read or decoded. `first(n)` and `take(n)` can attempt more than `n` winners when rescued failures are skipped. Disk processes winners in physical winner order.

  The traversal closes its descriptor through `ensure` after exhaustion or control-flow unwinding. A partial external or lazy Enumerator can retain its descriptor and complete winner index. It can also retain the current entry and an old unlinked snapshot inode. These resources can remain until Ruby exhausts, unwinds, or garbage-collects that traversal execution.
- REQ-08: `find_by_id(id)` accepts an exact String without coercion. It returns the newest frame-valid logical `DeadTask` with that authoritative outer ID when selected decoding and entry construction succeed. It returns `nil` when no frame-valid record matches. It also returns `nil` when selected top-level decoding raises `ArgumentError`, `NameError`, or `TypeError`, after printing the REQ-17 diagnostic. Disk scans its fixed complete-record snapshot sequentially forwards, replacing the remembered serialized-payload location for every frame-valid matching authoritative outer ID. It decodes only the newest remembered match after the complete scan. A newer frame-invalid match does not replace an older frame-valid match. A rescued selected match hides every older duplicate and remains stored. A decoded payload that cannot produce a dead-task record does not cause fallback to an older duplicate. That natural construction failure propagates unchanged. Lookup does not resolve or instantiate the task class. Its observable contract and public entry shape are backend-independent across Disk, Nil, and future adapters.
- REQ-09: A `DeadTask` exposes read-only data through `id`, `task_class`, `attempts`, `enqueued_at`, `failed_at`, `exception_class`, `exception_message`, and `backtrace`, while `retry` and `delete` are entry-level operator actions defined by REQ-11 and REQ-12. Timestamp readers return `Time` values. `task_class` and `exception_class` remain stored names (Strings), so listing and basic inspection work after a class is renamed or removed.
- REQ-10: `args` and `kwargs` lazily deserialize the stored execution context with `Marshal.load(..., freeze: true)`, extract values through `Rage::Deferred::Context.get_args` and `.get_kwargs`, and return the original positional Array and keyword Hash, using frozen empty collections where the accessors return `nil`. The successfully decoded values are cached with stable identities, while a failed decode is not cached. The frozen decoded object graph must not let callers mutate the stored replay input. Exceptions from Marshal deserialization or context accessor calls raise a dedicated public error that includes the dead-task ID, preserves the original exception as `cause`, and keeps the record intact.

  The current storage version contains contexts written only by Rage, so this MVP does not duplicate `Rage::Deferred::Context` with proactive checks that the decoded context is an Array of a minimum length or that the extracted values are an Array and Hash. Missing slots therefore follow the current accessors' normal `nil` behavior and normalize to empty collections. Manually written or schema-drifted contexts are outside the current-version contract; an incompatible context-layout change must use a future storage-version bump and add its compatibility guardrails there.

The collection and final active-entry shape, including the mutation-safe snapshot, is the accepted decision in [ADR-001](adr/001-public-dead-task-object-model.md). Task 02 initially defines `DeadTask` as a passive read-only summary value with no collection, backend, or action dependency. Task 04 introduces a narrow private action delegate with deletion, and task 05 depends on task 04 and extends that same mechanism with retry.

### Retry and deletion

- REQ-11: `DeadTask#delete` and `DeadTasks#delete(*ids)` permanently remove records. The entry method returns `true` only when its logical record was removed. The collection method accepts String IDs or nested arrays, de-duplicates them, returns the number of distinct records removed, and returns `0` for no IDs. Missing or already-removed IDs are not errors. For a large selected set, callers should collect exact IDs with `filter_map` and pass that array to one `dead_tasks.delete(ids)` call; the accumulator retains memory proportional to the selected IDs in addition to the traversal's complete winner index, while Disk performs one locked compaction rather than one per entry.
- REQ-12: `DeadTask#retry` and `DeadTasks#retry(id)` retry one record. A missing or concurrently deleted ID returns `false`; a successful retry returns `true`. A rescued selected lookup failure also returns `false` after the REQ-17 diagnostic. It does not enqueue or remove the stored record. This feature adds no collection-level bulk-retry primitive. Retrying a large set performs a fresh enqueue and an individual dead-store removal for every task. This work may approach quadratic storage cost under task 01's rewrite algorithm. An optimized large-set retry workflow is future work.
- REQ-13: Retry resolves the stored class name without `eval`, verifies that it is a class including `Rage::Deferred::Task`, deserializes the original arguments, and invokes the normal public enqueue path. This creates a fresh context and task ID, resets the attempt count, runs enqueue middleware and enqueue telemetry again, and does not recreate the old exception.
- REQ-14: Retry removes the dead-task record only after the new pending WAL write and enqueue call succeed. Class resolution, context deserialization, middleware, backpressure, or pending-storage failure leaves the dead task intact. A dead-store deletion error is raised after enqueue and can leave both a new pending task and the dead task; retry is at-least-once, not exactly-once.
- REQ-15: Concurrent retry/delete operations are serialized only for their individual storage calls. Two operators racing to retry the same snapshot can enqueue duplicate work, and a delete racing with a retry does not cancel work already enqueued. Operator documentation must require coordination for mutation commands and warn that large retry sets combine necessary per-task enqueues, repeated dead-store compactions, and at-least-once duplication risk.
- REQ-16: `Queue#schedule` is a no-op unless `Iodine.running?`. A retry from Rake, IRB, or another process without a running Iodine loop therefore persists a new pending WAL record but does not execute it. Rage emits an explicit warning that a server restart is required. Periodic adoption of out-of-band/orphan WAL files is deferred and is not part of this feature.

The fresh-enqueue and out-of-process behavior is described in [ADR-002](adr/002-dead-task-retry-lifecycle.md).

### Corruption, serialization, and security

- REQ-17: Rage exclusively controls marshaling and writing the versioned dead-task store. Listing and `find_by_id` therefore do not provide a shared public-record schema/type-validation layer for semantically invalid but correctly stored payloads, manually constructed records, schema drift without a storage-version bump, or inner/outer-ID mismatches. The outer framed ID is authoritative for selection, display, deduplication, and later exact-ID actions.

  On Disk, a record is frame-valid when and only when it is newline-terminated inside the captured snapshot boundary, has valid frame structure and the expected operation, has a valid CRC, and has a non-empty authoritative outer framed ID. Winner selection does not call `Marshal.load` and does not require the decoded payload to be a Hash, contain any required keys or value types, contain only String backtrace entries, store context as a String, or have an inner ID matching the outer ID. A malformed frame, invalid CRC, or excluded incomplete tail is silently skipped as recoverable physical corruption without logging or terminal output.

  Enumeration and exact lookup complete their sequential forward frame-validation scans before decoding selected records. Enumeration builds its insertion-ordered winner index and attempts selected records one at a time immediately before possible yields. Exact lookup retains only the newest frame-valid matching payload location and attempts that match after the scan. Once either path selects a winner, it never falls back to an older duplicate.

  Disk writes this template when selected top-level decoding raises `ArgumentError`, `NameError`, or `TypeError`:

  ```text
  ERROR: Can't deserialize the dead-lettered task with id {ID}: ({ERROR_CLASS}) {ERROR_MESSAGE}
  ```

  Disk does not sanitize or truncate `{ID}` or `{ERROR_MESSAGE}`. Enumeration skips that selected logical task and continues. Exact lookup returns `nil`. Both operations leave the record stored. The selected frame-valid record still hides older duplicates.

  Other selected top-level decoding failures propagate unchanged. Natural public-model construction failures also propagate. These failures do not trigger storage-level fallback. Disk compaction behavior remains as defined by task 01. Other adapters return the same public model. They need not emulate Disk framing or a shared schema-validation layer.
- REQ-18: A valid record whose opaque context raises during Marshal deserialization or access through `Rage::Deferred::Context` remains listable through its metadata. Accessing `args` or `kwargs`, or retrying it, raises the dedicated context-deserialization error and does not delete it. The error boundary does not add the proactive context shape or extracted-value type validation deferred by REQ-10.
- REQ-19: Dead-task records can contain credentials, personal data, logger context, exception messages, and application object graphs. The feature adds no HTTP endpoint and performs no network I/O. Ruby and Rake interfaces are for trusted operators; documentation must warn against copying output into logs or tickets without redaction. The REQ-17 decoding diagnostic writes the ID, exception class, and unsanitized exception message to standard output. Rage does not inspect the message for sensitive data.
- REQ-20: Marshal payloads are trusted local application data, not an interchange format. Rage must not load a dead-task file from an untrusted source, and the Rake interface must never use `eval` or shell interpolation to resolve records or classes. Disk frame validation provides corruption detection for Rage's local storage framing; it does not make an untrusted Marshal payload safe to decode.

### Rake/operator interface

- REQ-21: Thin Rake tasks expose list, show, retry, and delete through the Ruby API. The proposed names are `deferred:dead_tasks:list`, `deferred:dead_tasks:show`, `deferred:dead_tasks:retry`, and `deferred:dead_tasks:delete`.
- REQ-22: `list` prints bounded summaries oldest-first in the collection's order and omits arguments and backtraces. `show` requires one `ID` and prints the full inspectable record, including arguments and backtrace, with a sensitive-data warning. `retry` and `delete` accept comma-separated `IDS`, process each ID independently, print a per-ID outcome and final totals, and exit non-zero after reporting all failures.
- REQ-23: Rake tasks boot the application environment, contain no storage logic, and behave consistently for Disk and Nil backends. Out-of-process retry output prominently states that the pending task requires a Rage server restart.

### Compatibility and performance

- REQ-24: The feature is additive, preserves `config.deferred.backend` and its existing options, and requires no new configuration. It supports Rage's minimum Ruby version, 3.3.0.
- REQ-25: Apps that never access `dead_tasks` incur no new public-API allocation or polling. Tasks that do not permanently fail incur no new per-execution conditional beyond the task-01 handoff.

  Snapshot setup holds the permanent lock while Disk opens the live file and locates the last complete-record boundary. Disk does not scan or decode the complete store while locked.

  The calling Fiber performs the complete frame-validation scan before the first yield. The scan proceeds sequentially forwards without calling `Marshal.load`. Enumeration retains a winner index proportional to the distinct frame-valid outer IDs. Enumeration then processes selected winners oldest-first and one at a time.

  A very large snapshot can delay same-worker work for `first` or `take`. After selection, those operations attempt winners in winner order. They stop when they return, raise, or exhaust the snapshot. Rescued failures can increase the number of attempted winners.

  Each successful Disk removal can scan and rewrite the store. Repeated immediate deletion or retry across N entries can approach O(N²) aggregate storage work. Entry loops remain supported and mutation-safe for small sets. Large deletion sets should use one collection-level bulk delete. Large-set retry optimization is a non-goal.
- REQ-26: Lock waits remain non-blocking and fiber-aware as specified by task 01. Snapshot-acquisition `Rage::Deferred::DeadTasksLockTimeout` and filesystem failures from opening, reading, seeking, or cleanup propagate unchanged from the lazy operation that performs them; Rage does not translate them into a generic listing or lookup error. Enumeration therefore raises acquisition failures on the first advancement that starts the snapshot and later I/O failures on the advancement that performs that I/O; block traversal raises from its `each` call. Exact lookup propagates them from `find_by_id`. Locks and descriptors are released through `ensure`. Cleanup must not replace an exception already being propagated. When storage cleanup is itself the only failure, its original exception propagates instead of being silently suppressed. No public iterator holds a filesystem lock while yielding user code. Multiple Iodine workers share one logical collection through the disk backend.
- REQ-27: Block-based `DeadTasks#each` is the recommended traversal form. Each block traversal closes its descriptor through `ensure` on exhaustion, `break`, another non-local exit, or exception. Ordinary terminal Enumerable operations, including early-exit operations such as `find`, `first`, and `take`, receive the same cleanup when they unwind the `each` call.

  Calling `each` without a block returns a normal Ruby Enumerator. Rage does not add `close`, a shared-cursor protocol, generation tracking, or special cross-surface `rewind` behavior. Full exhaustion gives the traversal normal `ensure` cleanup. If an external or derived/lazy Enumerator is only partly consumed, its descriptor and any old unlinked snapshot inode can remain open until that particular Ruby iteration execution is exhausted, unwound, or garbage-collected. Descriptor cleanup is therefore nondeterministic for abandoned partial iteration. Ruby or optional finalizer behavior is only a best-effort safety net; the API does not promise when it runs, and tests must not depend on it. A future feature may add a separately named closeable cursor or another deterministic cancellation mechanism if real usage requires one.
- REQ-28: `DeadTasks` does not override or overload `Enumerable#find` or its `detect` alias. Both retain Ruby's standard block-predicate behavior, optional fallback callable, and no-block Enumerator behavior. Exact-ID lookup is available only through `find_by_id`.
- REQ-29: `DeadTasks#find_by_id` delegates through a private backend exact-lookup capability and must not branch on the backend class. Disk and Nil implement the same observable contract, and shared backend-conformance examples must be reusable for future adapters. Disk performs a complete sequential forward scan of its fixed complete-record view; Nil returns `nil`; and a future database adapter may use an indexed query. This feature guarantees behavioral consistency, not backend-independent time or I/O complexity, and does not add a database backend.

## Non-goals

- Periodically discovering or adopting WAL files created by consoles, Rake processes, or unrelated processes.
- Exactly-once replay or an atomic transaction spanning the pending WAL and dead-tasks file.
- Automatic retention, pruning, quotas, archival, or dead-task metrics.
- A web dashboard, HTTP/JSON management endpoint, authentication system, or remote administration protocol.
- Searching, filtering, or sorting by arbitrary record fields beyond oldest-first enumeration and exact-ID lookup.
- Editing arguments, retry policy, delay, or scheduled time before replay.
- An optimized bulk or large-set retry API.
- Migrating or merging older dead-task storage versions, or proactively validating alternate opaque-context layouts inside the current version. A context-layout compatibility change requires a storage-version bump; a filename version change leaves older files untouched for an explicit future migration.
- Supporting network filesystems whose lock, rename, or `fsync` semantics do not satisfy task 01.
- Implementing a database or other new deferred backend.

## Acceptance criteria

- [ ] AC-01: Both exhausted retries and explicit `false`/`nil` retry abortion produce a dead task, while successful and still-retrying tasks do not.
- [ ] AC-02: Disk records survive restart and remain until retry or deletion; the Nil backend exposes an empty, non-persistent collection.
- [ ] AC-03: `Rage::Deferred.dead_tasks` provides backend-neutral exact lookup through `find_by_id`. It also provides stable, oldest-first, mutation-safe Enumerable traversal. `first(20)` processes winners oldest-first. It returns up to 20 entries that enumeration can yield. REQ-17 skips can cause more than 20 attempts.

  Repeated accessor calls return the same traversal-stateless wrapper. The accessor obtains the shared memoized backend through `Rage::Deferred.__backend`. The accessor passes that backend directly to the wrapper. The queue and collection use the identical backend for both initialization orders.

  Wrapper creation does not open a DLQ snapshot or process DLQ records. An unadvanced no-block Enumerator also performs no DLQ snapshot work. Public `each` accepts no arguments.

  Each block traversal owns an independent traversal execution and snapshot. Each ordinary top-level Enumerable operation also owns an independent execution and snapshot. Overlapping, nested, and same-process Fiber-interleaved traversals remain isolated.

  Calling `each` without a block returns a normal lazy Ruby Enumerator. Standard Ruby 3.3 internal and external behavior applies. Rage adds no `close` method. Standard `Enumerable#find` and `Enumerable#detect` behavior remains unchanged.

  Disk captures a fixed complete-record boundary under lock. Enumeration scans each complete record sequentially forwards before the first yield. It builds the newest-frame-valid insertion-ordered winner index without calling `Marshal.load`. Enumeration then processes winners oldest-first and one at a time.

  Exact lookup uses the same fixed-boundary forward frame scan. It remembers only the newest matching payload location. It attempts that payload after the scan.
- [ ] AC-04: Each public entry exposes the documented metadata, exception details, and lazily decoded positional and keyword arguments without requiring the task class for basic inspection. Argument decoding uses frozen Marshal loading plus `Rage::Deferred::Context` accessors, nil normalization, successful-only caching, and the dedicated ID-and-cause error boundary, without duplicating the current storage version's context shape or extracted-value type rules.
- [ ] AC-05: A recoverable retry creates a fresh normally enqueued task with reset attempts. It removes the dead record only after enqueue succeeds. A rescued selected lookup failure prints the REQ-17 diagnostic, returns `false`, and keeps the record stored. Every other failure and partial-success case follows REQ-14.
- [ ] AC-06: Delete handles single, multiple, duplicate, missing, empty, and concurrently stale IDs with the documented return values, and one collection-level bulk call delegates to one Disk compaction.
- [ ] AC-07: Malformed frames, invalid CRCs, and excluded torn tails produce no diagnostic output. These failures do not delete recoverable data. A newer frame-valid duplicate remains the winner after a payload problem. Covered problems include non-Hash payloads, missing or mistyped fields, and invalid backtrace or context values. Coverage includes inner/outer-ID mismatches. No selected payload problem causes fallback to an older duplicate.

  Enumeration and exact lookup inspect their complete fixed-boundary snapshots before decoding selected payloads. A selected `ArgumentError`, `NameError`, or `TypeError` produces the exact standard-output diagnostic. Enumeration then skips that logical task. Exact lookup returns `nil`.

  Other selected decoding failures propagate unchanged. Natural entry-construction failures also propagate unchanged. Operational lock and filesystem failures propagate unchanged. All selected failures leave the record stored.
- [ ] AC-08: Retry outside a running Iodine server persists work without scheduling it and clearly warns that restart is required; scheduling remains unchanged inside a running server.
- [ ] AC-09: Rake list, show, retry, and delete delegate to the Ruby API, report partial outcomes, protect list output from accidental payload disclosure, and return a failing process status when requested operations fail.
- [ ] AC-10: Public APIs have YARD documentation. Operator documentation covers the lifecycle, oldest-first processing, shared backend identity, and traversal-lazy snapshot work. It explains newest-frame-valid selection, one-at-a-time attempts, and silent physical-corruption handling. It specifies exact rescued selected-payload behavior and the unsanitized standard-output diagnostic. It distinguishes other selected failures, opaque-context failures, and operational failures.

  Operator documentation covers trusted local Marshal data, races, retention, filesystem assumptions, and sensitive data. It explains wrapper identity, traversal isolation, and standard Enumerator behavior. It describes forward selection, winner-index memory, decoded-record retention, and cleanup. It covers repeated-mutation cost, large-set bulk deletion, and the absence of bulk retry. The changelog identifies the new API and commands.
- [ ] AC-11: Targeted RSpec coverage exercises both wrapper and queue initialization orders. Both orders use one identical backend object. Coverage verifies direct collection use of that backend. Collection access and valid unadvanced enumeration perform no DLQ snapshot work. Coverage verifies argument-free public `each` and oldest-first `first(20)` behavior. It verifies extra winner attempts only for rescued selected failures.

  Coverage verifies complete forward frame-only selection without `Marshal.load`. It verifies that no reverse-record reader remains. It covers silent malformed-frame, invalid-CRC, and torn-tail skipping. It covers newest-frame-valid duplicate selection for enumeration and exact lookup. Selected winners decode only after selection.

  Coverage verifies the exact diagnostic for each rescued exception class. It verifies enumeration skip, lookup `nil`, stored-record preservation, and no older fallback. Other selected decoding and entry-construction failures propagate unchanged. Lock timeout and filesystem failures also propagate unchanged. Cleanup preserves the original exception.

  Coverage verifies shared Disk and Nil exact-lookup behavior. It exercises forward `find_by_id` while another enumeration is paused. It verifies inherited `find` and `detect` behavior. It covers lazy no-block Enumerators and full-exhaustion cleanup.

  Coverage exercises separate top-level executions and snapshots. It includes overlapping traversal, nested traversal, and same-process Fiber interleaving. Standard mixed Ruby iteration remains unsynchronized by Rage. Coverage verifies cleanup after exhaustion, early terminal exit, block exit, and exceptions.

  Coverage verifies forward winner selection and one-at-a-time oldest-first delivery. It includes torn-tail repair with append and rename compaction during enumeration. It covers lazy frozen context decoding through `Rage::Deferred::Context`. Context coverage includes nil normalization, immutable graphs, successful-only caching, and ID-and-cause errors.

  Coverage includes retry and delete failures, races, non-running Iodine behavior, Rake output, and Rake status. Argument replay remains Ruby 3.3 compatible.

## Tasks

- [x] [01 — Add durable dead-letter storage](tasks/01-durable-dead-letter-storage.md)
- [x] [02 — List dead tasks](tasks/02-list-dead-tasks.md)
- [x] [03 — Inspect a dead task](tasks/03-inspect-a-dead-task.md)
- [ ] [04 — Delete dead tasks](tasks/04-delete-dead-tasks.md)
- [ ] [05 — Retry dead tasks](tasks/05-retry-dead-tasks.md)
- [ ] [06 — Add Rake operations and documentation](tasks/06-rake-operations-and-documentation.md)

Task dependencies: task 01 precedes task 02; task 02 precedes tasks 03 and 04; both tasks 03 and 04 precede task 05; task 06 follows completion of tasks 02 through 05.

## Decisions

- [ADR-001 — Public dead-task object model and iteration](adr/001-public-dead-task-object-model.md) (accepted)
- [ADR-002 — Dead-task retry lifecycle](adr/002-dead-task-retry-lifecycle.md) (proposed)

## Open questions for this draft

1. Should retry create a fresh context through `Task.enqueue`, as proposed, or preserve the original serialized logger/user context by calling the queue directly? Recommendation: use the public enqueue path, preserve only args/kwargs, and reset all execution context so middleware and telemetry observe a deliberate new enqueue.
2. Are `deferred:dead_tasks:*` and `ID`/`IDS` environment variables the desired Rake interface, or should the namespace retain the shorter `deferred:dlq:*` form from the initial issue proposal? Recommendation: use `dead_tasks` consistently with the Ruby API.

## References

- [Rage issue #369](https://github.com/rage-rb/rage/issues/369)
- [Durable storage implementation PR rage-rb/rage#383](https://github.com/rage-rb/rage/pull/383)
