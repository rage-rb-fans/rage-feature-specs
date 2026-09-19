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
- REQ-02: With the disk backend, Rage durably adds the dead-task record before removing the pending WAL record. A failed dead-task write leaves the pending record recoverable. A crash between those writes may leave the same logical task in both stores; the dead-tasks store presents the newest fully valid record for a duplicated task ID. If a newer duplicate is invalid, an older valid duplicate remains visible.
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

# When manually advancing an external iterator, retain and close the root
# iterator if traversal may stop before exhaustion.
iterator = dead_tasks.each(batch_size: 100)
begin
  inspect_one(iterator.next)
ensure
  iterator.close
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

- REQ-05: `Rage::Deferred.dead_tasks` returns a memoized `Rage::Deferred::DeadTasks` collection backed by the same configured backend instance as the queue, so repeated accessor calls return the same wrapper object. The shared wrapper includes `Enumerable` and is stateless with respect to traversal. It may retain the configured backend, immutable collaborators or configuration, and later collection-level mutation behavior, but it must not retain snapshot descriptors or boundaries, reverse-reader cursors, `seen_ids`, decoded batches, warning counts, closed/rewound state, current Enumerators, or a traversal registry. Obtaining the wrapper does not scan or preload the store. Every call to `each` creates its own private traversal session; calling `each` without a block opens nothing and returns a new actual Ruby `Enumerator` instance, normally implemented by subclassing or extending `Enumerator`, with an additional public `close` method for deterministic cancellation of partial external traversal. The concrete iterator class is not otherwise public API.
- REQ-06: `each(batch_size: 100)` yields `Rage::Deferred::DeadTask` entries newest-first. `batch_size` must be a positive Integer. `DeadTasks#each` validates it immediately and raises `ArgumentError` before returning a no-block Enumerator or beginning block traversal; validation opens, scans, and decodes nothing. Each traversal independently establishes a stable snapshot on its first request for an entry, does not hold the shared store lock while application code runs, and remains safe when yielded entries are retried or deleted. Overlapping, nested, or same-process Fiber-interleaved traversals must not share cursors or lifecycle state. Closing, rewinding, exhausting, breaking, or raising in one traversal must not change another traversal. An append after one traversal starts is excluded from that snapshot but may appear in another traversal whose first entry is requested later. If traversal continues normally, mutating a yielded entry does not skip, duplicate, reorder, or replace any remaining snapshot entry. This is a reentrancy and traversal-isolation guarantee, not a new general thread-safety guarantee beyond Rage's existing Iodine/Fiber model.
- REQ-07: When traversal starts, Disk briefly holds the permanent dead-task lock, opens the current live file, and records a fixed byte boundary immediately after its last complete newline-terminated record. It then releases the lock and reads complete records backwards from that boundary through the retained descriptor. Validation, duplicate selection, decoding, and wrapping happen lazily. Traversal remembers only IDs for which it has already found a fully valid newer record, so a complete traversal uses memory proportional to the distinct valid IDs it encounters; it does not build a complete offset index or cache every payload. At most one `batch_size` payload batch is decoded and wrapped at a time. Stopping early avoids parsing or decoding untouched older records. Traversal closes its descriptor automatically on exhaustion and block/internal unwinding, or immediately when the root external iterator is explicitly closed. A renamed snapshot file may continue consuming disk space only until that cleanup occurs.
- REQ-08: `find_by_id(id)` accepts an exact String without coercion and returns the newest fully valid logical `DeadTask` with that ID, or `nil`. It follows REQ-17's invalid-newer-duplicate fallback rule and does not resolve or instantiate the task class. Its observable contract and public entry shape are backend-independent across Disk, Nil, and future adapters.
- REQ-09: A `DeadTask` exposes read-only data through `id`, `task_class`, `attempts`, `enqueued_at`, `failed_at`, `exception_class`, `exception_message`, and `backtrace`, while `retry` and `delete` are entry-level operator actions defined by REQ-11 and REQ-12. Timestamp readers return `Time` values. `task_class` and `exception_class` remain stored names (Strings), so listing and basic inspection work after a class is renamed or removed.
- REQ-10: `args` and `kwargs` lazily deserialize the stored execution context and return the original positional Array and keyword Hash, using empty collections where the context stored `nil`. Returned values must not let callers mutate the stored record. Deserialization failures raise a dedicated public error that includes the dead-task ID and keeps the record intact.

The collection and final active-entry shape, including the mutation-safe snapshot, is the proposed decision in [ADR-001](adr/001-public-dead-task-object-model.md). Task 02 initially defines `DeadTask` as a passive read-only summary value with no collection, backend, or action dependency. Task 04 introduces a narrow private action delegate with deletion, and task 05 depends on task 04 and extends that same mechanism with retry.

### Retry and deletion

- REQ-11: `DeadTask#delete` and `DeadTasks#delete(*ids)` permanently remove records. The entry method returns `true` only when its logical record was removed. The collection method accepts String IDs or nested arrays, de-duplicates them, returns the number of distinct records removed, and returns `0` for no IDs. Missing or already-removed IDs are not errors. For a large selected set, callers should collect exact IDs with `filter_map` and pass that array to one `dead_tasks.delete(ids)` call; the accumulator retains memory proportional to the selected IDs in addition to the traversal's incremental set of already-seen valid IDs, while Disk performs one locked compaction rather than one per entry.
- REQ-12: `DeadTask#retry` and `DeadTasks#retry(id)` retry one record. A missing or concurrently deleted ID returns `false`; a successful retry returns `true`. This feature adds no collection-level bulk-retry primitive. Retrying a large set performs a fresh enqueue and an individual dead-store removal for every task and may approach quadratic storage work under task 01's rewrite-on-remove algorithm; an optimized large-set retry workflow is future work.
- REQ-13: Retry resolves the stored class name without `eval`, verifies that it is a class including `Rage::Deferred::Task`, deserializes the original arguments, and invokes the normal public enqueue path. This creates a fresh context and task ID, resets the attempt count, runs enqueue middleware and enqueue telemetry again, and does not recreate the old exception.
- REQ-14: Retry removes the dead-task record only after the new pending WAL write and enqueue call succeed. Class resolution, context deserialization, middleware, backpressure, or pending-storage failure leaves the dead task intact. A dead-store deletion error is raised after enqueue and can leave both a new pending task and the dead task; retry is at-least-once, not exactly-once.
- REQ-15: Concurrent retry/delete operations are serialized only for their individual storage calls. Two operators racing to retry the same snapshot can enqueue duplicate work, and a delete racing with a retry does not cancel work already enqueued. Operator documentation must require coordination for mutation commands and warn that large retry sets combine necessary per-task enqueues, repeated dead-store compactions, and at-least-once duplication risk.
- REQ-16: `Queue#schedule` is a no-op unless `Iodine.running?`. A retry from Rake, IRB, or another process without a running Iodine loop therefore persists a new pending WAL record but does not execute it. Rage emits an explicit warning that a server restart is required. Periodic adoption of out-of-band/orphan WAL files is deferred and is not part of this feature.

The fresh-enqueue and out-of-process behavior is described in [ADR-002](adr/002-dead-task-retry-lifecycle.md).

### Corruption, serialization, and security

- REQ-17: Listing and `find_by_id` return only records that pass the backend's integrity checks and the shared public-record schema. The shared schema requires the stored metadata needed by REQ-09 and REQ-10 with the types defined by tasks 02 and 03, including an opaque execution context stored as bytes of the expected String type; listing and exact lookup must not deserialize that context. Invalid physical records and an excluded incomplete tail are skipped items, not operational failures. For duplicate IDs, the newest fully valid logical record wins and an invalid newer duplicate does not hide an older valid duplicate. On Disk specifically, a record must be complete and newline-terminated; its frame must parse and pass CRC validation; its outer framed task ID must be a non-empty String; its top-level Marshal payload must load as the expected Hash; all required fields must have the expected types; and inner `record[:id]` must exactly match the authoritative outer framed ID. Disk compaction behavior remains as defined by task 01. Other adapters apply equivalent integrity checks for their own storage format while returning the same public result.

  Enumeration counts skipped items independently for each traversal and only when lazy reading actually encounters them. It must not inspect untouched older records merely to compute a count. When a traversal that encountered at least one skipped item finalizes through exhaustion, block `break` or another non-local exit, explicit root `close`, `rewind`, or an exception, Rage calls `Rage.logger.warn` exactly once for that traversal. The warning reports one aggregate count as items encountered during this traversal, never as a count of the whole snapshot or store, and includes no IDs, payloads, stored fields, raw bytes, or serialized exception details. An unadvanced traversal and a traversal with count zero emit nothing. Repeated idempotent `close` does not repeat a warning; `rewind` finalizes the old session and its next advancement starts a new count. Overlapping and nested traversals never share counts. If an iterator is abandoned without deterministic cleanup, warning timing and delivery are likewise not guaranteed; Rage does not scan or depend on garbage collection merely to produce a warning.

  Exact lookup keeps a separate operation-scoped count. When that count is nonzero, it emits exactly one aggregate `Rage.logger.warn` warning for invalid items actually inspected before it finds a match, reaches the boundary, or exits with an operational error. Its message identifies the count as encountered during that lookup, not as the complete store count, and follows the same payload-redaction rule. A lookup does not read older records solely to complete its warning count and never shares warning state with an enumeration.
- REQ-18: A valid record whose opaque context cannot be deserialized remains listable through its metadata. Accessing `args` or `kwargs`, or retrying it, raises the dedicated deserialization error and does not delete it.
- REQ-19: Dead-task records can contain credentials, personal data, logger context, exception messages, and application object graphs. The feature adds no HTTP endpoint and performs no network I/O. Ruby and Rake access is for trusted operators with filesystem/application access; documentation must warn against copying output into logs or tickets without redaction.
- REQ-20: Marshal payloads are trusted local application data, not an interchange format. Rage must not load a dead-task file from an untrusted source, and the Rake interface must never use `eval` or shell interpolation to resolve records or classes.

### Rake/operator interface

- REQ-21: Thin Rake tasks expose list, show, retry, and delete through the Ruby API. The proposed names are `deferred:dead_tasks:list`, `deferred:dead_tasks:show`, `deferred:dead_tasks:retry`, and `deferred:dead_tasks:delete`.
- REQ-22: `list` prints bounded summaries newest-first and omits arguments and backtraces. `show` requires one `ID` and prints the full inspectable record, including arguments and backtrace, with a sensitive-data warning. `retry` and `delete` accept comma-separated `IDS`, process each ID independently, print a per-ID outcome and final totals, and exit non-zero after reporting all failures.
- REQ-23: Rake tasks boot the application environment, contain no storage logic, and behave consistently for Disk and Nil backends. Out-of-process retry output prominently states that the pending task requires a Rage server restart.

### Compatibility and performance

- REQ-24: The feature is additive, preserves `config.deferred.backend` and its existing options, and requires no new configuration. It supports Rage's minimum Ruby version, 3.3.0.
- REQ-25: Apps that never access `dead_tasks` and tasks that do not permanently fail incur no new public-API allocation, polling, or per-execution conditional beyond the task-01 handoff. Snapshot setup holds the permanent lock only long enough to open the live file and locate the last complete-record boundary; it does not scan or decode the whole store while locked. Reverse traversal does synchronous file I/O and Marshal decoding in the calling Fiber as entries are requested, so a complete traversal still scans the captured store and a very large traversal can delay other work in that worker. A complete traversal retains a `seen_ids` set proportional to the number of distinct valid IDs encountered, while an early stop avoids processing untouched older records. Each successful Disk removal can scan and rewrite the store, so repeatedly calling immediate entry deletion or retry across N entries can approach O(N²) aggregate storage work. These entry loops remain supported and mutation-safe for individual entries or small sets, but large deletion sets should use one collection-level bulk delete. Large-set retry optimization is a non-goal for this feature.
- REQ-26: Lock waits remain non-blocking and fiber-aware as specified by task 01. Snapshot-acquisition `Rage::Deferred::DeadTasksLockTimeout` and filesystem failures from opening, reading, seeking, or checking permissions propagate unchanged from the lazy operation that performs them; Rage does not translate them into a generic listing or lookup error. Enumeration therefore raises acquisition failures on the first advancement that starts the snapshot and later I/O failures on the advancement that performs that I/O; block traversal raises from its `each` call. Exact lookup propagates them from `find_by_id`. Locks and descriptors are released through `ensure`. Cleanup and skipped-item warning delivery must not replace an exception already being propagated. When storage cleanup is itself the only failure, its original exception propagates instead of being silently suppressed. No public iterator holds a filesystem lock while yielding user code. Multiple Iodine workers share one logical collection through the disk backend.
- REQ-27: The directly returned external iterator's `close` is idempotent and returns `nil` after successful cleanup before first advance, during partial traversal, and after exhaustion. Closing never opens a snapshot; it releases any open descriptor, finalizes the traversal-local skipped-item warning from REQ-17, and leaves the iterator exhausted until `rewind`. After `close`, `next`, `next_values`, `peek`, and `peek_values` raise `StopIteration`, block-based `each` on that root yields no entries, and no other advancement or iteration surface may reopen or resume the closed snapshot. `rewind` first performs the same cleanup and warning finalization, resets the iterator without opening a descriptor, returns the same iterator object after successful cleanup, and makes the next advancement establish a fresh snapshot with a new skipped-item count. Storage cleanup errors propagate under REQ-26. Callers that partially consume an external iterator must retain the root iterator and call `close` in `ensure`; block-based collection traversal remains the preferred ordinary form because Rage closes it automatically. Derived iterators such as `with_index` or `lazy` are not guaranteed to expose `close`, so callers must close their retained root iterator. Unreachable iterator state may release its owned IO through Ruby's normal object cleanup or a carefully implemented finalizer as a best-effort safety net, but garbage-collection timing is not a public guarantee and tests must not rely on it. A custom finalizer, if used, must not strongly retain the iterator and should close the owned IO/state object rather than a reused raw file-descriptor integer. Idle timeouts are not deterministic cleanup and must not replace explicit `close` or block unwinding.
- REQ-28: `DeadTasks` does not override or overload `Enumerable#find` or its `detect` alias. Both retain Ruby's standard block-predicate behavior, optional fallback callable, and no-block Enumerator behavior. Exact-ID lookup is available only through `find_by_id`.
- REQ-29: `DeadTasks#find_by_id` delegates through a private backend exact-lookup capability and must not branch on the backend class. Disk and Nil implement the same observable contract, and shared backend-conformance examples must be reusable for future adapters. Disk may perform a worst-case reverse scan of its fixed complete-record view; Nil returns `nil`; and a future database adapter may use an indexed query. This feature guarantees behavioral consistency, not backend-independent time or I/O complexity, and does not add a database backend.

### Disk storage permissions

- REQ-30: When Disk creates a dead-task live data file, permanent lock file, or compaction temporary file, it requests owner-only mode `0600`. The process umask may remove permissions from that request, and Rage must not override it with broader permissions. Rage does not change the configured storage directory's mode and adds no file-mode configuration. This policy applies only to dead-task Disk files; it does not migrate pending-task WAL permissions. Nil creates no files, and future backends define their own storage permissions.
- REQ-31: Disk initialization preserves the mode of an existing live data or lock file. In particular, a legacy `0644` file remains `0644` until an operator hardens it manually. A stale or pre-existing compaction temporary file is internal scratch, not a compatibility boundary: while holding the permanent dead-task lock and before reusing that file for compaction, Disk must remove all group and other permissions without adding any owner permission that was already absent. No initialization-time operation may modify the shared temporary path without that lock.
- REQ-32: Before compaction renames its completed temporary file over the live data path, Disk applies the live file's established access mode bits to the temporary file. It then persists both content and mode metadata with the temporary-file `fsync`, performs the existing atomic rename, and `fsync`s the directory. The replacement must preserve access mode bits across compaction, but Rage does not promise to preserve or change file ownership and does not require `chown` capability.
- REQ-33: The supported default is that all Rage workers use one operating-system account. A cross-user deployment must pre-provision a trusted, non-world-writable local directory with suitable shared group ownership, setgid behavior when needed, file modes such as `0660`, and compatible process umasks. Operator documentation must recommend a `0700` directory for one account, or a controlled `0770` setgid directory for a shared group, and must warn that removing owner read or write access can make Disk storage unusable.

The dead-task Disk file policy and its compatibility tradeoff are described in [ADR-003](adr/003-dead-task-file-permissions.md).

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
- Implementing a database or other new deferred backend.

## Acceptance criteria

- [ ] AC-01: Both exhausted retries and explicit `false`/`nil` retry abortion produce a dead task, while successful and still-retrying tasks do not.
- [ ] AC-02: Disk records survive restart and remain until retry or deletion; the Nil backend exposes an empty, non-persistent collection.
- [ ] AC-03: `Rage::Deferred.dead_tasks` provides backend-neutral exact lookup through `find_by_id` and stable, newest-first, mutation-safe Enumerable traversal. Repeated accessor calls return the same traversal-stateless wrapper, while every no-block `each` call returns a distinct root Enumerator with its own lazy snapshot and lifecycle; block-based calls likewise own independent private sessions. `batch_size` validation is immediate and performs no storage work. Overlapping, nested, and same-process Fiber-interleaved traversals remain isolated. Standard `Enumerable#find`/`detect` predicate, fallback, and no-block semantics remain inherited and unmodified. Disk captures a fixed complete-record boundary under lock, then validates records lazily in reverse order, retains only incremental `seen_ids` metadata plus a bounded decoded batch, and does not build a full offset index. The directly returned object is a Ruby `Enumerator` with deterministic idempotent `close`, complete post-close exhaustion, and fresh-snapshot `rewind` behavior as defined by REQ-27.
- [ ] AC-04: Each public entry exposes the documented metadata, exception details, and lazily decoded positional and keyword arguments without requiring the task class for basic inspection.
- [ ] AC-05: Retry creates a fresh normally enqueued task with reset attempts and removes the dead record only after enqueue succeeds; every failure path and partial-success case follows REQ-14.
- [ ] AC-06: Delete handles single, multiple, duplicate, missing, empty, and concurrently stale IDs with the documented return values, and one collection-level bulk call delegates to one Disk compaction.
- [ ] AC-07: Corrupt, unreadable, schema-invalid, inner/outer-ID-mismatched records, and excluded torn tails follow REQ-17 and REQ-18 without deleting otherwise recoverable data or exposing payloads in warnings; an invalid newer duplicate falls back to the next older fully valid duplicate. Enumeration and exact lookup report only invalid items actually encountered, through one redacted aggregate `Rage.logger.warn` warning per completed operation or traversal session with a nonzero count. Operational lock and filesystem failures propagate unchanged and are not counted as invalid records.
- [ ] AC-08: Retry outside a running Iodine server persists work without scheduling it and clearly warns that restart is required; scheduling remains unchanged inside a running server.
- [ ] AC-09: Rake list, show, retry, and delete delegate to the Ruby API, report partial outcomes, protect list output from accidental payload disclosure, and return a failing process status when requested operations fail.
- [ ] AC-10: Public APIs have YARD documentation; operator documentation covers lifecycle, immediate side-effect-free batch validation, operation-scoped redacted corruption warnings, unchanged lock/filesystem error propagation, at-least-once races, retention, filesystem assumptions, sensitive data, the memoized wrapper's traversal-stateless behavior, lazy reverse traversal and its O(N) complete-traversal `seen_ids` memory, automatic block cleanup, external root-iterator `close`/`rewind` responsibilities, repeated-mutation cost, the large-set bulk-delete pattern, and the absence of bulk retry; and the changelog identifies the new API and commands.
- [ ] AC-11: Targeted RSpec coverage exercises immediate `batch_size` validation with no storage access; warning destination, redaction, frequency, and encountered-only count scope across exhaustion, early stop, close, rewind, and exception; independent traversal and exact-lookup counts; unchanged lock-timeout and filesystem-error propagation with cleanup that preserves the original exception; shared Disk/Nil exact-lookup conformance; `find_by_id` while another enumeration is paused; inherited `find`/`detect` predicate and fallback behavior; distinct and independently timed root Enumerators; overlapping and nested traversal; same-process Fiber interleaving; traversal-local close/rewind/exhaustion/break/exception behavior; lazy reverse batching; automatic cleanup; explicit external-iterator close/rewind lifecycle; invalid-duplicate fallback and schema checks; torn-tail repair plus append; rename compaction during enumeration; retry/delete failures and races; non-running Iodine behavior; Rake output/status; and Ruby 3.3-compatible argument replay.
- [ ] AC-12: Disk requests `0600` for newly created dead-task live, lock, and temporary files; a restrictive umask is honored; existing live and lock modes are not changed during initialization; stale temporary files are hardened under the permanent lock before reuse; and every successful compaction replacement preserves the prior live file's mode bits through the durable `fsync`/rename sequence. Nil creates no files, and operator documentation covers legacy hardening and supported directory/account layouts.

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
- [ADR-003 — Dead-task Disk file permissions](adr/003-dead-task-file-permissions.md) (proposed)

## Open questions for this draft

1. Should retry create a fresh context through `Task.enqueue`, as proposed, or preserve the original serialized logger/user context by calling the queue directly? Recommendation: use the public enqueue path, preserve only args/kwargs, and reset all execution context so middleware and telemetry observe a deliberate new enqueue.
2. Are `deferred:dead_tasks:*` and `ID`/`IDS` environment variables the desired Rake interface, or should the namespace retain the shorter `deferred:dlq:*` form from the initial issue proposal? Recommendation: use `dead_tasks` consistently with the Ruby API.

## References

- [Rage issue #369](https://github.com/rage-rb/rage/issues/369)
- [Durable storage implementation PR rage-rb/rage#383](https://github.com/rage-rb/rage/pull/383)
