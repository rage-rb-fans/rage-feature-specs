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
- REQ-02: With the disk backend, Rage durably adds the dead-task record before removing the pending WAL record. A failed dead-task write leaves the pending record recoverable. A crash between those writes may leave the same logical task in both stores; the dead-tasks store presents the newest fully valid record for a duplicated task ID. If a newer duplicate is invalid, an older valid duplicate remains visible.
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

  The shared wrapper includes `Enumerable` and is stateless with respect to traversal. It may retain the shared backend, immutable collaborators or configuration, and later collection-level mutation behavior. It must not retain snapshot descriptors or boundaries, reverse-reader cursors, winner indexes, decoded batches, current Enumerators, or a traversal registry. Obtaining the wrapper may initialize the backend, including the backend's normal initialization side effects, but it does not open a DLQ snapshot descriptor or scan, decode, or preload DLQ records. Calling `each` without a block likewise performs no snapshot work and returns a new normal Ruby `Enumerator`, with standard Ruby 3.3 behavior and no Rage-specific `close` method. A `DeadTask` entry never retains the backend.
- REQ-06: Argument-free `each` yields `Rage::Deferred::DeadTask` entries oldest-first according to the physical positions of their selected winning records. Consequently, `dead_tasks.first(20)` returns the oldest 20 logical dead tasks in its snapshot. For a duplicated ID, the newest fully valid physical record remains the winner and its position determines that logical task's place in the order. Decoded payload batching is an internal implementation detail: callers cannot configure it and its numeric size is not a compatibility guarantee.

  Each block traversal and each ordinary top-level Enumerable operation uses one traversal execution. When that execution starts, the retained backend establishes its snapshot. That execution owns its snapshot, cursor, descriptor, winner index, and decoded batch. Separate calls such as `find`, `map`, `count`, or `each` start separate executions and snapshots. Overlapping, nested, or same-process Fiber-interleaved collection traversals do not share this state. An append after one traversal starts is excluded from that snapshot but may appear in a traversal that starts later.

  A no-block Enumerator follows standard Ruby internal and external iteration behavior. Rage does not synchronize `next` or `peek` with a later block `enumerator.each`, derived iterators, or other ways of consuming that same Enumerator. Those forms may use separate Ruby iteration executions, cursors, and snapshots. A normally consumed lazy pipeline uses its underlying traversal. This feature adds no broader thread-safety guarantee beyond Rage's existing Iodine/Fiber model.
- REQ-07: When traversal starts, Disk briefly holds the permanent dead-task lock, opens the current live file, and records a fixed byte boundary immediately after its last complete newline-terminated record. It then releases the lock. Before yielding, it reads and validates every complete record backwards through the retained descriptor and records the location of the first fully valid record found for each ID. This complete winner index selects newest-valid duplicates. Disk then visits the winning locations in reverse discovery order and re-reads, decodes, wraps, and yields them oldest-first in a fixed internally bounded payload batch. The index uses memory proportional to the distinct valid logical IDs; it does not cache every decoded payload. Early termination cannot avoid the complete selection scan, but it avoids re-reading and decoding later winners beyond the current internal batch. The traversal closes its descriptor through `ensure` when it exhausts or when block/internal control flow unwinds. If an external or lazy Enumerator is only partly consumed and then retained or abandoned, its descriptor, complete winner index, and any old unlinked snapshot inode may remain until Ruby exhausts, unwinds, or garbage-collects that traversal execution.
- REQ-08: `find_by_id(id)` accepts an exact String without coercion and returns the newest fully valid logical `DeadTask` with that ID, or `nil`. It follows REQ-17's invalid-newer-duplicate fallback rule and does not resolve or instantiate the task class. Its observable contract and public entry shape are backend-independent across Disk, Nil, and future adapters.
- REQ-09: A `DeadTask` exposes read-only data through `id`, `task_class`, `attempts`, `enqueued_at`, `failed_at`, `exception_class`, `exception_message`, and `backtrace`, while `retry` and `delete` are entry-level operator actions defined by REQ-11 and REQ-12. Timestamp readers return `Time` values. `task_class` and `exception_class` remain stored names (Strings), so listing and basic inspection work after a class is renamed or removed.
- REQ-10: `args` and `kwargs` lazily deserialize the stored execution context and return the original positional Array and keyword Hash, using empty collections where the context stored `nil`. Returned values must not let callers mutate the stored record. Deserialization failures raise a dedicated public error that includes the dead-task ID and keeps the record intact.

The collection and final active-entry shape, including the mutation-safe snapshot, is the accepted decision in [ADR-001](adr/001-public-dead-task-object-model.md). Task 02 initially defines `DeadTask` as a passive read-only summary value with no collection, backend, or action dependency. Task 04 introduces a narrow private action delegate with deletion, and task 05 depends on task 04 and extends that same mechanism with retry.

### Retry and deletion

- REQ-11: `DeadTask#delete` and `DeadTasks#delete(*ids)` permanently remove records. The entry method returns `true` only when its logical record was removed. The collection method accepts String IDs or nested arrays, de-duplicates them, returns the number of distinct records removed, and returns `0` for no IDs. Missing or already-removed IDs are not errors. For a large selected set, callers should collect exact IDs with `filter_map` and pass that array to one `dead_tasks.delete(ids)` call; the accumulator retains memory proportional to the selected IDs in addition to the traversal's complete winner index, while Disk performs one locked compaction rather than one per entry.
- REQ-12: `DeadTask#retry` and `DeadTasks#retry(id)` retry one record. A missing or concurrently deleted ID returns `false`; a successful retry returns `true`. This feature adds no collection-level bulk-retry primitive. Retrying a large set performs a fresh enqueue and an individual dead-store removal for every task and may approach quadratic storage work under task 01's rewrite-on-remove algorithm; an optimized large-set retry workflow is future work.
- REQ-13: Retry resolves the stored class name without `eval`, verifies that it is a class including `Rage::Deferred::Task`, deserializes the original arguments, and invokes the normal public enqueue path. This creates a fresh context and task ID, resets the attempt count, runs enqueue middleware and enqueue telemetry again, and does not recreate the old exception.
- REQ-14: Retry removes the dead-task record only after the new pending WAL write and enqueue call succeed. Class resolution, context deserialization, middleware, backpressure, or pending-storage failure leaves the dead task intact. A dead-store deletion error is raised after enqueue and can leave both a new pending task and the dead task; retry is at-least-once, not exactly-once.
- REQ-15: Concurrent retry/delete operations are serialized only for their individual storage calls. Two operators racing to retry the same snapshot can enqueue duplicate work, and a delete racing with a retry does not cancel work already enqueued. Operator documentation must require coordination for mutation commands and warn that large retry sets combine necessary per-task enqueues, repeated dead-store compactions, and at-least-once duplication risk.
- REQ-16: `Queue#schedule` is a no-op unless `Iodine.running?`. A retry from Rake, IRB, or another process without a running Iodine loop therefore persists a new pending WAL record but does not execute it. Rage emits an explicit warning that a server restart is required. Periodic adoption of out-of-band/orphan WAL files is deferred and is not part of this feature.

The fresh-enqueue and out-of-process behavior is described in [ADR-002](adr/002-dead-task-retry-lifecycle.md).

### Corruption, serialization, and security

- REQ-17: Listing and `find_by_id` return only records that pass the backend's integrity checks and the shared public-record schema. The shared schema requires the stored metadata needed by REQ-09 and REQ-10 with the types defined by tasks 02 and 03, including an opaque execution context stored as bytes of the expected String type; listing and exact lookup must not deserialize that context. Invalid physical records and an excluded incomplete tail are silently skipped as recoverable corruption, not treated as operational failures. For duplicate IDs, the newest fully valid logical record wins and an invalid newer duplicate does not hide an older valid duplicate. On Disk specifically, a record must be complete and newline-terminated; its frame must parse and pass CRC validation; its outer framed task ID must be a non-empty String; its top-level Marshal payload must load as the expected Hash; all required fields must have the expected types; and inner `record[:id]` must exactly match the authoritative outer framed ID. Disk compaction behavior remains as defined by task 01. Other adapters apply equivalent integrity checks for their own storage format while returning the same public result.

  Enumeration validates every complete record during the reverse selection scan required for oldest-first ordering and newest-valid duplicate selection. Exact lookup skips only invalid records inspected before it finds a match or reaches the boundary. Neither operation performs an additional scan merely to diagnose corruption, and neither emits logging or terminal output for skipped records or an excluded incomplete tail.
- REQ-18: A valid record whose opaque context cannot be deserialized remains listable through its metadata. Accessing `args` or `kwargs`, or retrying it, raises the dedicated deserialization error and does not delete it.
- REQ-19: Dead-task records can contain credentials, personal data, logger context, exception messages, and application object graphs. The feature adds no HTTP endpoint and performs no network I/O. Ruby and Rake interfaces are for trusted operators; documentation must warn against copying output into logs or tickets without redaction.
- REQ-20: Marshal payloads are trusted local application data, not an interchange format. Rage must not load a dead-task file from an untrusted source, and the Rake interface must never use `eval` or shell interpolation to resolve records or classes.

### Rake/operator interface

- REQ-21: Thin Rake tasks expose list, show, retry, and delete through the Ruby API. The proposed names are `deferred:dead_tasks:list`, `deferred:dead_tasks:show`, `deferred:dead_tasks:retry`, and `deferred:dead_tasks:delete`.
- REQ-22: `list` prints bounded summaries oldest-first in the collection's order and omits arguments and backtraces. `show` requires one `ID` and prints the full inspectable record, including arguments and backtrace, with a sensitive-data warning. `retry` and `delete` accept comma-separated `IDS`, process each ID independently, print a per-ID outcome and final totals, and exit non-zero after reporting all failures.
- REQ-23: Rake tasks boot the application environment, contain no storage logic, and behave consistently for Disk and Nil backends. Out-of-process retry output prominently states that the pending task requires a Rage server restart.

### Compatibility and performance

- REQ-24: The feature is additive, preserves `config.deferred.backend` and its existing options, and requires no new configuration. It supports Rage's minimum Ruby version, 3.3.0.
- REQ-25: Apps that never access `dead_tasks` and tasks that do not permanently fail incur no new public-API allocation, polling, or per-execution conditional beyond the task-01 handoff. Snapshot setup holds the permanent lock only long enough to open the live file and locate the last complete-record boundary; it does not scan or decode the whole store while locked. Enumeration then performs a complete reverse selection scan and top-level Marshal decoding synchronously in the calling Fiber before its first yield, retains a winner index proportional to the number of distinct valid logical IDs, and re-reads selected winners oldest-first in an internally bounded decoded batch. A very large snapshot can therefore delay same-worker work even for `first` or `take`; early termination saves only later winner reads and decodes. Each successful Disk removal can scan and rewrite the store, so repeatedly calling immediate entry deletion or retry across N entries can approach O(N²) aggregate storage work. These entry loops remain supported and mutation-safe for individual entries or small sets, but large deletion sets should use one collection-level bulk delete. Large-set retry optimization is a non-goal for this feature.
- REQ-26: Lock waits remain non-blocking and fiber-aware as specified by task 01. Snapshot-acquisition `Rage::Deferred::DeadTasksLockTimeout` and filesystem failures from opening, reading, seeking, or cleanup propagate unchanged from the lazy operation that performs them; Rage does not translate them into a generic listing or lookup error. Enumeration therefore raises acquisition failures on the first advancement that starts the snapshot and later I/O failures on the advancement that performs that I/O; block traversal raises from its `each` call. Exact lookup propagates them from `find_by_id`. Locks and descriptors are released through `ensure`. Cleanup must not replace an exception already being propagated. When storage cleanup is itself the only failure, its original exception propagates instead of being silently suppressed. No public iterator holds a filesystem lock while yielding user code. Multiple Iodine workers share one logical collection through the disk backend.
- REQ-27: Block-based `DeadTasks#each` is the recommended traversal form. Each block traversal closes its descriptor through `ensure` on exhaustion, `break`, another non-local exit, or exception. Ordinary terminal Enumerable operations, including early-exit operations such as `find`, `first`, and `take`, receive the same cleanup when they unwind the `each` call.

  Calling `each` without a block returns a normal Ruby Enumerator. Rage does not add `close`, a shared-cursor protocol, generation tracking, or special cross-surface `rewind` behavior. Full exhaustion gives the traversal normal `ensure` cleanup. If an external or derived/lazy Enumerator is only partly consumed, its descriptor and any old unlinked snapshot inode can remain open until that particular Ruby iteration execution is exhausted, unwound, or garbage-collected. Descriptor cleanup is therefore nondeterministic for abandoned partial iteration. Ruby or optional finalizer behavior is only a best-effort safety net; the API does not promise when it runs, and tests must not depend on it. A future feature may add a separately named closeable cursor or another deterministic cancellation mechanism if real usage requires one.
- REQ-28: `DeadTasks` does not override or overload `Enumerable#find` or its `detect` alias. Both retain Ruby's standard block-predicate behavior, optional fallback callable, and no-block Enumerator behavior. Exact-ID lookup is available only through `find_by_id`.
- REQ-29: `DeadTasks#find_by_id` delegates through a private backend exact-lookup capability and must not branch on the backend class. Disk and Nil implement the same observable contract, and shared backend-conformance examples must be reusable for future adapters. Disk may perform a worst-case reverse scan of its fixed complete-record view; Nil returns `nil`; and a future database adapter may use an indexed query. This feature guarantees behavioral consistency, not backend-independent time or I/O complexity, and does not add a database backend.

## Non-goals

- Periodically discovering or adopting WAL files created by consoles, Rake processes, or unrelated processes.
- Exactly-once replay or an atomic transaction spanning the pending WAL and dead-tasks file.
- Automatic retention, pruning, quotas, archival, or dead-task metrics.
- A web dashboard, HTTP/JSON management endpoint, authentication system, or remote administration protocol.
- Searching, filtering, or sorting by arbitrary record fields beyond oldest-first enumeration and exact-ID lookup.
- Editing arguments, retry policy, delay, or scheduled time before replay.
- An optimized bulk or large-set retry API.
- Migrating or merging older dead-task storage versions; a filename version change leaves older files untouched for an explicit future migration.
- Supporting network filesystems whose lock, rename, or `fsync` semantics do not satisfy task 01.
- Implementing a database or other new deferred backend.

## Acceptance criteria

- [ ] AC-01: Both exhausted retries and explicit `false`/`nil` retry abortion produce a dead task, while successful and still-retrying tasks do not.
- [ ] AC-02: Disk records survive restart and remain until retry or deletion; the Nil backend exposes an empty, non-persistent collection.
- [ ] AC-03: `Rage::Deferred.dead_tasks` provides backend-neutral exact lookup through `find_by_id` and stable, oldest-first, mutation-safe Enumerable traversal. `first(20)` returns the oldest 20 logical tasks by selected winning-record position. Repeated accessor calls return the same traversal-stateless wrapper. When creating the wrapper, the accessor obtains `Rage::Deferred.__backend` and passes that shared memoized backend directly to it, so the queue and collection use one identical backend regardless of initialization order. Creating the wrapper and returning an unadvanced no-block Enumerator do not open a DLQ snapshot descriptor or scan, decode, or preload DLQ records, and public `each` exposes no batch-size option. Each block traversal or ordinary top-level Enumerable operation owns an independent traversal execution and snapshot. Overlapping, nested, and same-process Fiber-interleaved collection traversals remain isolated. Calling `each` without a block returns a normal lazy Ruby Enumerator with standard Ruby 3.3 internal/external behavior and no Rage-specific `close`. Standard `Enumerable#find`/`detect` predicate, fallback, and no-block semantics remain inherited and unmodified. Disk captures a fixed complete-record boundary under lock, then scans every complete record backwards before the first yield to build the newest-valid winner index, and re-reads winners oldest-first with only an internally bounded decoded-payload batch.
- [ ] AC-04: Each public entry exposes the documented metadata, exception details, and lazily decoded positional and keyword arguments without requiring the task class for basic inspection.
- [ ] AC-05: Retry creates a fresh normally enqueued task with reset attempts and removes the dead record only after enqueue succeeds; every failure path and partial-success case follows REQ-14.
- [ ] AC-06: Delete handles single, multiple, duplicate, missing, empty, and concurrently stale IDs with the documented return values, and one collection-level bulk call delegates to one Disk compaction.
- [ ] AC-07: Corrupt, unreadable, schema-invalid, inner/outer-ID-mismatched records, and excluded torn tails follow REQ-17 and REQ-18 without deleting otherwise recoverable data or producing diagnostic output; an invalid newer duplicate falls back to the next older fully valid duplicate. Enumeration inspects the complete snapshot required by its selection algorithm, while exact lookup stops after its fully valid match. Operational lock and filesystem failures propagate unchanged rather than being treated as recoverable corruption.
- [ ] AC-08: Retry outside a running Iodine server persists work without scheduling it and clearly warns that restart is required; scheduling remains unchanged inside a running server.
- [ ] AC-09: Rake list, show, retry, and delete delegate to the Ruby API, report partial outcomes, protect list output from accidental payload disclosure, and return a failing process status when requested operations fail.
- [ ] AC-10: Public APIs have YARD documentation; operator documentation covers lifecycle, oldest-first order, shared queue/collection backend identity and traversal-lazy DLQ snapshot work, private internal batching, silent recoverable-corruption handling, unchanged lock/filesystem error propagation, at-least-once races, retention, filesystem assumptions, sensitive data, the memoized wrapper's traversal-stateless behavior, standard Enumerator behavior, the complete reverse selection scan and its O(N) winner-index memory, automatic cleanup for block and terminal Enumerable traversal, nondeterministic cleanup for abandoned partial external/lazy traversal, repeated-mutation cost, the large-set bulk-delete pattern, and the absence of bulk retry; and the changelog identifies the new API and commands.
- [ ] AC-11: Targeted RSpec coverage exercises wrapper and queue initialization in both orders with one identical backend object; direct use of that backend by the collection; no DLQ snapshot descriptor opening, record scanning, or record decoding during collection access or valid unadvanced no-block enumeration; absence of a public batch-size option; oldest-first `first(20)` behavior; silent full-snapshot corruption skipping during selection; unchanged lock-timeout and filesystem-error propagation with cleanup that preserves the original exception; shared Disk/Nil exact-lookup conformance; `find_by_id` while another enumeration is paused; inherited `find`/`detect` predicate and fallback behavior; standard lazy no-block Enumerator behavior and full-exhaustion cleanup; separate top-level Enumerable executions and snapshots; overlapping and nested traversal; same-process Fiber interleaving; standard Ruby mixed internal/external iteration without Rage synchronization; automatic cleanup on exhaustion, early terminal Enumerable exit, block exit, and exception; reverse winner selection and internally bounded oldest-first delivery; invalid-duplicate fallback and schema checks; torn-tail repair plus append; rename compaction during enumeration; retry/delete failures and races; non-running Iodine behavior; Rake output/status; and Ruby 3.3-compatible argument replay.

## Tasks

- [x] [01 — Add durable dead-letter storage](tasks/01-durable-dead-letter-storage.md)
- [ ] [02 — List dead tasks](tasks/02-list-dead-tasks.md)
- [ ] [03 — Inspect a dead task](tasks/03-inspect-a-dead-task.md)
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
