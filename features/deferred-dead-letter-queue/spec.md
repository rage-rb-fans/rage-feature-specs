---
title: Dead letter queue for deferred tasks
status: draft
---

# Dead letter queue for deferred tasks

## Motivation

Deferred tasks already have an at-least-once pending-task WAL, but operators also need a durable and supported way to investigate and recover work that Rage has stopped retrying. The dead-tasks feature records the final failure, exposes it through Ruby and command-line operator interfaces, and lets an operator retry or permanently delete the work.

Durable storage and the pending-to-dead handoff are already implemented by task 01. The remaining work turns that storage into a public operator feature without adding overhead to successful deferred-task execution.

## Proposed behavior

### Failure lifecycle and retention

- REQ-01: A deferred task becomes a dead task when its retry budget is exhausted or when `retry_interval` returns `false` or `nil`. A task that succeeds or still has a retry interval never enters the dead-tasks store.
- REQ-02: With the disk backend, Rage durably adds the dead-task record before removing the pending WAL record. A failed dead-task write leaves the pending record recoverable. A crash between those writes may leave the same logical task in both stores; the dead-tasks store presents only the latest record for a duplicated task ID.
- REQ-03: Dead tasks have no automatic retention period, maximum age, or drop-oldest policy. They remain until an operator retries or deletes them. Documentation must warn that this makes disk use unbounded when the store is not maintained.
- REQ-04: With `config.deferred.backend = nil`, failures remain non-persistent. The public collection is empty, lookup returns `nil`, and mutation of a missing ID reports that no record was changed; Rage does not create dead-task files.

Task 01 defines the canonical disk format and crash guarantees. Its version remains in the filename (`{prefix}dead_tasks-{STORAGE_VERSION}`), not in each record, and its permanent lock file coordinates workers and processes.

### Ruby API

The proposed public name is `dead_tasks`, as selected in the issue discussion:

```ruby
dead_tasks = Rage::Deferred.dead_tasks

dead_tasks.each(batch_size: 100) do |entry|
  puts "#{entry.id}: #{entry.task_class} (#{entry.exception_class})"
end

entry = dead_tasks.find("1735689600-12345-1")
entry.retry
entry.delete

# Entry actions remain mutation-safe during traversal and are convenient for
# individual entries or small sets. They are not the large-set maintenance path.
dead_tasks.each(&:retry)
dead_tasks.each(&:delete)

# Prefer one bulk delete for a large selected set. This accumulates IDs, not
# DeadTask entries, and Disk performs one locked compaction.
ids = dead_tasks.filter_map { |dead_task| dead_task.id if removable?(dead_task) }
dead_tasks.delete(ids)
```

- REQ-05: `Rage::Deferred.dead_tasks` returns a memoized `Rage::Deferred::DeadTasks` collection backed by the same configured backend instance as the queue. The collection includes `Enumerable`; obtaining it does not scan or preload the store, and `each` returns an enumerator when no block is given.
- REQ-06: `each(batch_size: 100)` yields `Rage::Deferred::DeadTask` entries newest-first. `batch_size` must be a positive Integer; invalid values raise `ArgumentError`. Actual traversal establishes a stable snapshot of the logical entries present when traversal begins, does not hold the shared store lock while application code runs, and remains safe when yielded entries are retried or deleted. If traversal continues normally, mutating a yielded entry does not skip, duplicate, reorder, or replace any remaining snapshot entry. Records added after the snapshot begins are left for a later enumeration.
- REQ-07: Snapshot setup performs a full locked store scan and retains ID/offset metadata proportional to the number of distinct logical task IDs. The batching bound applies only to decoded and wrapped record payloads: subsequent traversal keeps at most one `batch_size` payload batch at a time. Total iterator memory is therefore not bounded independently of store cardinality. The collection and iterator do not build or cache an eager array of all entries; callers may still explicitly choose materialization through standard `Enumerable` operations such as `to_a`. Traversal closes its snapshot file descriptor when exhausted, when the block exits non-locally, or when an exception is raised. A renamed snapshot file may continue consuming disk space until that descriptor closes.
- REQ-08: `find(id)` returns a `DeadTask` for the latest valid record with that exact String ID, or `nil`. It does not resolve or instantiate the task class.
- REQ-09: A `DeadTask` exposes read-only data through `id`, `task_class`, `attempts`, `enqueued_at`, `failed_at`, `exception_class`, `exception_message`, and `backtrace`, while `retry` and `delete` are entry-level operator actions defined by REQ-11 and REQ-12. Timestamp readers return `Time` values. `task_class` and `exception_class` remain stored names (Strings), so listing and basic inspection work after a class is renamed or removed.
- REQ-10: `args` and `kwargs` lazily deserialize the stored execution context and return the original positional Array and keyword Hash, using empty collections where the context stored `nil`. Returned values must not let callers mutate the stored record. Deserialization failures raise a dedicated public error that includes the dead-task ID and keeps the record intact.

The collection and final active-entry shape, including the mutation-safe snapshot, is the proposed decision in [ADR-001](adr/001-public-dead-task-object-model.md). Task 02 initially defines `DeadTask` as a passive read-only summary value with no collection, backend, or action dependency. Task 04 introduces a narrow private action delegate with deletion, and task 05 depends on task 04 and extends that same mechanism with retry.

### Retry and deletion

- REQ-11: `DeadTask#delete` and `DeadTasks#delete(*ids)` permanently remove records. The entry method returns `true` only when its logical record was removed. The collection method accepts String IDs or nested arrays, de-duplicates them, returns the number of distinct records removed, and returns `0` for no IDs. Missing or already-removed IDs are not errors. For a large selected set, callers should collect exact IDs with `filter_map` and pass that array to one `dead_tasks.delete(ids)` call; the ID accumulator costs memory proportional to the selected IDs, in addition to the snapshot's proportional-to-logical-IDs index, while Disk performs one locked compaction rather than one per entry.
- REQ-12: `DeadTask#retry` and `DeadTasks#retry(id)` retry one record. A missing or concurrently deleted ID returns `false`; a successful retry returns `true`. This feature adds no collection-level bulk-retry primitive. Retrying a large set performs a fresh enqueue and an individual dead-store removal for every task and may approach quadratic storage work under task 01's rewrite-on-remove algorithm; an optimized large-set retry workflow is future work.
- REQ-13: Retry resolves the stored class name without `eval`, verifies that it is a class including `Rage::Deferred::Task`, deserializes the original arguments, and invokes the normal public enqueue path. This creates a fresh context and task ID, resets the attempt count, runs enqueue middleware and enqueue telemetry again, and does not recreate the old exception.
- REQ-14: Retry removes the dead-task record only after the new pending WAL write and enqueue call succeed. Class resolution, context deserialization, middleware, backpressure, or pending-storage failure leaves the dead task intact. A dead-store deletion error is raised after enqueue and can leave both a new pending task and the dead task; retry is at-least-once, not exactly-once.
- REQ-15: Concurrent retry/delete operations are serialized only for their individual storage calls. Two operators racing to retry the same snapshot can enqueue duplicate work, and a delete racing with a retry does not cancel work already enqueued. Operator documentation must require coordination for mutation commands and warn that large retry sets combine necessary per-task enqueues, repeated dead-store compactions, and at-least-once duplication risk.
- REQ-16: `Queue#schedule` is a no-op unless `Iodine.running?`. A retry from Rake, IRB, or another process without a running Iodine loop therefore persists a new pending WAL record but does not execute it. Rage emits an explicit warning that a server restart is required. Periodic adoption of out-of-band/orphan WAL files is deferred and is not part of this feature.

The fresh-enqueue and out-of-process behavior is described in [ADR-002](adr/002-dead-task-retry-lifecycle.md).

### Corruption, serialization, and security

- REQ-17: Listing and finding skip malformed, CRC-invalid, or unreadable top-level records and report a warning with a count without printing record payloads. A corrupt record is never presented as retryable. Compaction behavior remains as defined by task 01.
- REQ-18: A valid record whose opaque context cannot be deserialized remains listable through its metadata. Accessing `args` or `kwargs`, or retrying it, raises the dedicated deserialization error and does not delete it.
- REQ-19: Dead-task records can contain credentials, personal data, logger context, exception messages, and application object graphs. The feature adds no HTTP endpoint and performs no network I/O. Ruby and Rake access is for trusted operators with filesystem/application access; documentation must warn against copying output into logs or tickets without redaction.
- REQ-20: Marshal payloads are trusted local application data, not an interchange format. Rage must not load a dead-task file from an untrusted source, and the Rake interface must never use `eval` or shell interpolation to resolve records or classes.

### Rake/operator interface

- REQ-21: Thin Rake tasks expose list, show, retry, and delete through the Ruby API. The proposed names are `deferred:dead_tasks:list`, `deferred:dead_tasks:show`, `deferred:dead_tasks:retry`, and `deferred:dead_tasks:delete`.
- REQ-22: `list` prints bounded summaries newest-first and omits arguments and backtraces. `show` requires one `ID` and prints the full inspectable record, including arguments and backtrace, with a sensitive-data warning. `retry` and `delete` accept comma-separated `IDS`, process each ID independently, print a per-ID outcome and final totals, and exit non-zero after reporting all failures.
- REQ-23: Rake tasks boot the application environment, contain no storage logic, and behave consistently for Disk and Nil backends. Out-of-process retry output prominently states that the pending task requires a Rage server restart.

### Compatibility and performance

- REQ-24: The feature is additive, preserves `config.deferred.backend` and its existing options, and requires no new configuration. It supports Rage's minimum Ruby version, 3.3.0.
- REQ-25: Apps that never access `dead_tasks` and tasks that do not permanently fail incur no new public-API allocation, polling, or per-execution conditional beyond the task-01 handoff. Snapshot setup synchronously scans the whole dead-tasks file under the shared lock and retains metadata proportional to the number of distinct logical IDs; on a large store, the scan can delay other Fibers in that worker and make competing operations exhaust task 01's finite lock-retry budget. Each successful Disk removal can scan and rewrite the store, so repeatedly calling immediate entry deletion or retry across N entries can approach O(N²) aggregate storage work. These entry loops remain supported and mutation-safe for individual entries or small sets, but large deletion sets should use one collection-level bulk delete. Large-set retry optimization is a non-goal for this feature.
- REQ-26: Lock waits remain non-blocking and fiber-aware as specified by task 01. No public iterator holds a filesystem lock while yielding user code. Multiple Iodine workers share one logical collection through the disk backend.

## Non-goals

- Periodically discovering or adopting WAL files created by consoles, Rake processes, or unrelated processes.
- Exactly-once replay or an atomic transaction spanning the pending WAL and dead-tasks file.
- Automatic retention, pruning, quotas, archival, or dead-task metrics.
- A web dashboard, HTTP/JSON management endpoint, authentication system, or remote administration protocol.
- Searching, filtering, or sorting by arbitrary record fields beyond newest-first enumeration and exact-ID lookup.
- Editing arguments, retry policy, delay, or scheduled time before replay.
- An optimized bulk or large-set retry API.
- Migrating or merging older dead-task storage versions; a filename version change leaves older files untouched for an explicit future migration.
- Supporting network filesystems whose lock, rename, or `fsync` semantics do not satisfy task 01.

## Acceptance criteria

- [ ] AC-01: Both exhausted retries and explicit `false`/`nil` retry abortion produce a dead task, while successful and still-retrying tasks do not.
- [ ] AC-02: Disk records survive restart and remain until retry or deletion; the Nil backend exposes an empty, non-persistent collection.
- [ ] AC-03: `Rage::Deferred.dead_tasks` provides exact lookup and stable, newest-first, mutation-safe Enumerable traversal whose payload materialization is batch-bounded while snapshot metadata remains proportional to distinct logical IDs.
- [ ] AC-04: Each public entry exposes the documented metadata, exception details, and lazily decoded positional and keyword arguments without requiring the task class for basic inspection.
- [ ] AC-05: Retry creates a fresh normally enqueued task with reset attempts and removes the dead record only after enqueue succeeds; every failure path and partial-success case follows REQ-14.
- [ ] AC-06: Delete handles single, multiple, duplicate, missing, empty, and concurrently stale IDs with the documented return values, and one collection-level bulk call delegates to one Disk compaction.
- [ ] AC-07: Corrupt records and undecodable contexts follow REQ-17 and REQ-18 without deleting otherwise recoverable data or exposing payloads in warnings.
- [ ] AC-08: Retry outside a running Iodine server persists work without scheduling it and clearly warns that restart is required; scheduling remains unchanged inside a running server.
- [ ] AC-09: Rake list, show, retry, and delete delegate to the Ruby API, report partial outcomes, protect list output from accidental payload disclosure, and return a failing process status when requested operations fail.
- [ ] AC-10: Public APIs have YARD documentation; operator documentation covers lifecycle, at-least-once races, retention, filesystem assumptions, sensitive data, O(N) snapshot metadata, repeated-mutation cost, the large-set bulk-delete pattern, and the absence of bulk retry; and the changelog identifies the new API and commands.
- [ ] AC-11: Targeted RSpec coverage exercises Disk and Nil behavior, snapshot batching and cleanup, mutation during enumeration, retry/delete failures and races, non-running Iodine behavior, Rake output/status, and Ruby 3.3-compatible argument replay.

## Tasks

- [x] [01 — Add durable dead-letter storage](tasks/01-durable-dead-letter-storage.md)
- [ ] [02 — List dead tasks](tasks/02-list-dead-tasks.md)
- [ ] [03 — Inspect a dead task](tasks/03-inspect-a-dead-task.md)
- [ ] [04 — Delete dead tasks](tasks/04-delete-dead-tasks.md)
- [ ] [05 — Retry dead tasks](tasks/05-retry-dead-tasks.md)
- [ ] [06 — Add Rake operations and documentation](tasks/06-rake-operations-and-documentation.md)

Task dependencies: task 01 precedes task 02; task 02 precedes tasks 03 and 04; both tasks 03 and 04 precede task 05; task 06 follows completion of tasks 02 through 05.

## Decisions

- [ADR-001 — Public dead-task object model and iteration](adr/001-public-dead-task-object-model.md) (proposed)
- [ADR-002 — Dead-task retry lifecycle](adr/002-dead-task-retry-lifecycle.md) (proposed)

## Open questions for this draft

1. Should retry create a fresh context through `Task.enqueue`, as proposed, or preserve the original serialized logger/user context by calling the queue directly? Recommendation: use the public enqueue path, preserve only args/kwargs, and reset all execution context so middleware and telemetry observe a deliberate new enqueue.
2. Are `deferred:dead_tasks:*` and `ID`/`IDS` environment variables the desired Rake interface, or should the namespace retain the shorter `deferred:dlq:*` form from the initial issue proposal? Recommendation: use `dead_tasks` consistently with the Ruby API.
3. Should new dead-task data and temp files be created with owner-only mode `0600` instead of task 01's current `0644` mode? Recommendation: use `0600` for newly created data/temp files while preserving existing file modes, and document that deployments needing cross-user access must provision the directory accordingly. This security-sensitive compatibility choice must be resolved before implementation.

## References

- [Rage issue #369](https://github.com/rage-rb/rage/issues/369)
- [Durable storage implementation PR rage-rb/rage#383](https://github.com/rage-rb/rage/pull/383)
