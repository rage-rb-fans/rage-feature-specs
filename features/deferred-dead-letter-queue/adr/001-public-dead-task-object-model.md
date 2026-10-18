---
status: accepted
---

# ADR-001: Public dead-task object model and iteration

## Context

Issue #369 selected the public name `dead_tasks`, and the framework owner requested incremental Enumerable traversal instead of an eager `all`, including mutation-safe support for the convenience flows `dead_tasks.each(&:retry)` and `dead_tasks.each(&:delete)`. Deleting or retrying records during offset-based pagination can skip later entries. The disk store is shared by workers and processes, may contain duplicate physical records for one logical task ID, and is compacted by rename.

The public model must support basic inspection even when the task class no longer exists, avoid holding the store lock while arbitrary operator code runs, and make iteration behavior observable rather than relying on unstable offsets. Rage exclusively controls marshaling and writing this versioned local store, so storage selection needs to validate physical framing and integrity, not impose a second shared schema/type-validation layer on Rage-written payloads.

## Traversal options considered

### Option A: Inert entries and collection-only mutations

`DeadTask` is a value object. Operators call `dead_tasks.retry(entry.id)` or `dead_tasks.delete(entry.id)`. Batching can remain weakly consistent, but the concise `each(&:retry)` form is unavailable and callers still can mutate the collection manually during iteration.

### Option B: Active entries over offset pagination

Entries expose `retry` and `delete`, and `each` repeatedly calls `list(limit:, offset:)`. This is simple, but deleting or retrying yielded entries shifts later offsets and silently skips records. Concurrent additions can similarly duplicate or omit results.

### Option C: Active entries over a stable, bounded-payload snapshot

The collection exposes both collection mutations and entry convenience methods. At enumeration start, the disk backend scans the whole store under its shared lock, validates records, and records the latest byte location for each logical ID in insertion order. It retains an open read descriptor as the snapshot, releases the lock before yielding, and decodes only one internal batch at a time. Rename-based compaction cannot invalidate the open descriptor; records appended after snapshot setup are excluded. The index costs memory proportional to the number of distinct logical IDs, while decoded record payloads are internally batch-bounded. The design does not bound total snapshot memory independently of store cardinality.

This option gives direct seeks to selected records, but the complete scan holds the lock for work proportional to the whole store.

### Option D: Active entries over a lazy reverse snapshot

The collection keeps the same public entry and mutation surfaces as Option C, but snapshot setup does not build an ID/offset index. While holding the permanent dead-task lock, Disk opens the live file and captures the byte boundary immediately after its last complete newline-terminated record. It releases the lock, retains the descriptor, and reads complete records backwards from that boundary as the caller requests entries.

Traversal frame-validates records lazily and adds an authoritative outer ID to a per-traversal `seen_ids` set after frame structure, operation, CRC, and non-empty outer-ID checks pass. It does not need to load or inspect the top-level payload to select an ID. Decoded entries remain batch-bounded. A complete traversal retains `seen_ids` metadata proportional to the number of distinct frame-valid IDs encountered, while an early-stopped traversal avoids processing untouched older records.

This option naturally yields newest-first. It cannot yield oldest-first while preserving newest-frame-valid duplicate selection without first examining the complete snapshot: a record seen near the beginning of a forward scan can be superseded by a frame-valid duplicate near the end.

### Option E: Active entries over a reverse-selected two-phase oldest-first snapshot

The collection keeps Option D's stable descriptor and short lock hold, but traversal performs two phases after releasing the lock. First, it scans every complete record backwards and frame-validates it without loading its top-level Marshal payload. The first frame-valid record found for an authoritative outer ID is the winner. A newer malformed frame or invalid CRC permits an older frame-valid record to win; a newer frame-valid record reserves its ID even if its payload later fails to decode or has an unexpected shape. The traversal stores only each winner's ID and serialized-payload location in newest-to-oldest discovery order.

Second, it visits those winning locations in reverse discovery order, re-reads and decodes them, and yields them oldest-first in fixed internally sized decoded-payload batches. The complete winner index costs O(distinct frame-valid outer IDs) memory. A selected winner is decoded only during this delivery phase, and a decoding or record-shape failure does not fall back to an older duplicate. The traversal does not retain every decoded payload. Even `first` and `take` require the full frame-only selection scan before the first yield; early exit saves only later winner re-reads and decodes beyond the current internal batch.

### Option F: Active entries over a forward-selected two-phase oldest-first snapshot

The collection keeps Option E's stable descriptor, short lock hold, frame-only selection phase, and complete winner index. Instead of reading the snapshot backwards, selection reads every complete physical record sequentially forwards from byte zero through the fixed boundary. The insertion-ordered winner index initially records a frame-valid authoritative outer ID at its physical position. Each later frame-valid duplicate deletes the prior index entry and reinserts the ID with the newer serialized-payload location, moving the logical task to its newest winning physical position. A frame-invalid duplicate does not change the existing winner.

After the complete scan, the index supplies winners oldest-first without reversing discovery order. Traversal attempts each winner before it advances. No later selected record is predecoded. Like Option E, this requires O(distinct frame-valid outer IDs) winner metadata. Traversal cannot yield before scanning the complete snapshot. It uses sequential forward reads and retains at most the current decoded record beyond bounded line storage. Exact lookup retains only the newest frame-valid matching payload location. It attempts that payload after the complete scan. Enumeration and lookup may share private helpers, but the decision does not require a new abstraction.

## External-iterator cleanup options

These options address a separate question from traversal Options A through F: how much lifecycle machinery the first public iterator should provide.

### Cleanup option 1: Standard Ruby Enumerator

Keep no-block `each` and return a normal Ruby Enumerator. Block traversal and normally unwound terminal Enumerable methods release their descriptor through `ensure`. Full external exhaustion does the same. Standard Ruby internal/external iteration, `rewind`, derived iterators, and lazy behavior remain unchanged.

This is the smallest and most idiomatic choice, but it cannot promise prompt cleanup after partial external or lazy traversal. A suspended execution can retain its descriptor and an old unlinked inode until it is exhausted, unwound, or garbage-collected.

### Cleanup option 2: Closeable shared-session Enumerator

Return a custom Enumerator with public `close`. Make `next`, `peek`, block `each`, Enumerable methods, and derived iterators share one traversal session, cursor, lookahead, lifecycle, and generation. This provides deterministic cancellation when the caller retains the root, but replaces standard Ruby iteration behavior and requires substantial cross-surface state and testing.

This remains a possible future design if real usage needs cancellable manual traversal. A separately named cursor API could provide the same capability without redefining normal Enumerator behavior.

### Cleanup option 3: Block-only traversal

Reject `each` without a block and require every traversal to occur through a block that Rage can unwind automatically. This makes cleanup simple and deterministic, but breaks the normal Enumerable expectation that no-block `each` returns an external iterator and prevents idiomatic manual enumeration requested for this API.

### Cleanup option 4: Eager materialization

Read and wrap all entries before returning an external iterator, then close the descriptor immediately. This uses only standard Enumerator behavior and has simple resource ownership, but defeats incremental selected-record decoding. Large stores could require memory proportional to all decoded entries.

### Cleanup option 5: Immutable generations with leases and garbage collection

Store snapshots as immutable versions. Iterators acquire leases on generations, while a separate generation collector removes versions after their leases are released; abandoned leases require an explicit crash-recovery policy. This can provide robust snapshot ownership without pinning a replaced live inode, but introduces persistent generation metadata, cross-process lease coordination, crash recovery, and storage garbage collection far beyond Task 02.

Ruby's normal IO cleanup, or a carefully implemented custom finalizer, may release unreachable iterator state as best-effort leak protection. Ruby does not guarantee prompt collection or finalizer execution, and a timeout could close a valid slow traversal. Tests and public guarantees cannot depend on cleanup timing for an abandoned partial traversal.

## Exact-lookup naming options

### `find_by_id`

Expose exact lookup as `DeadTasks#find_by_id(id)`. This states the lookup key directly, follows a Sidekiq-style separate naming path, and leaves every inherited Enumerable search method unchanged.

### `find_task`

Expose exact lookup as `DeadTasks#find_task(id)`. This avoids overriding Enumerable, but `task` is redundant on a `DeadTasks` collection and is less precise about the lookup key.

### Overload `find`

Make `find(id)` perform exact lookup while trying to preserve block-based `find`. This creates two unrelated meanings under one method, risks changing the optional fallback callable and no-block forms, and makes `detect` inconsistent with its Enumerable alias. Reject this option.


## Approved batch extension

[ADR-003](003-batch-dead-task-retry.md) defines collection retry for [task 08](../tasks/08-batch-retry-dead-tasks.md).
The entry-only statements below describe the task-05 stage.
ADR-003 extends retry ownership and bulk scope while preserving individual entry behavior and the other decisions below.
Task 08 uses one batch per invocation, without an internal batch-size limit.
Task 08 is ready for development. This extension does not change this ADR's status.

## Decision

Adopt traversal Option F with one-at-a-time selected-record attempts, cleanup option 1, and `find_by_id` for public exact lookup. A standard Ruby Enumerator preserves no-block `each` and holds at most the current decoded record. It avoids a custom stream protocol. Deterministic cancellation for partial external or lazy traversal is deferred. A future design can reconsider a closeable cursor or versioned storage. `find_by_id` avoids overriding `Enumerable#find` and `Enumerable#detect`. Those methods retain standard Ruby behavior.

`Rage::Deferred.dead_tasks` returns an Enumerable `DeadTasks` collection without scanning or preloading the store. Iteration processes selected winning records oldest-first. `dead_tasks.first(20)` returns up to 20 oldest entries that enumeration can yield. Rescued selected failures can make it attempt more than 20 winners. Entry data remains read-only.

Task 02 introduces `DeadTask` as a passive summary value without a mutation dependency. Task 03 preserves that stage while adding detailed inspection. Task 04 gives active entries the identical shared backend privately and adds direct deletion. Task 05 adds entry-owned retry. The completed task-02 and task-03 requirements retain their historical stage-specific meaning.

The collection constructs each active entry with its `@backend` and the record data. Each entry privately retains the backend shared with `Rage::Deferred.__backend` and the queue. An entry never constructs a backend. The entry exposes no public backend accessor or setter. Entries retain no collection, action handler, service, iterator, snapshot descriptor, cursor, winner index, or traversal registry.

`DeadTask#delete` calls `@backend.remove_dead_tasks(@id) == 1` without lookup or argument decoding. `DeadTask#retry` owns an exact-ID existence check, class validation, argument decoding, enqueue, optional backend flush, and removal. `DeadTasks` has no `retry` method. ID-based callers use `find_by_id(id)` and call `entry.retry` when lookup returns an entry.

Collection deletion retains its input validation, bulk normalization, and count result. Both deletion surfaces use the same backend removal primitive. The public layer contains no compaction algorithm.

Entry retry checks existence through backend exact-ID lookup with task 03's lookup semantics. Retry discards the returned record and uses the entry's original class and serialized context. The original metadata and successful inspection cache remain unchanged. A same-ID replacement does not change replay input; later removal can delete that replacement. The existence check does not reserve the record or compare versions.

Entry actions work after traversal cleanup and do not change an active snapshot. No action calls another action recursively.

The accessor memoizes only the shared collection wrapper: repeated `Rage::Deferred.dead_tasks` calls return the same object. When creating the wrapper, the accessor obtains the shared memoized backend through `Rage::Deferred.__backend` and passes that backend object directly to it. The wrapper never constructs a backend independently. This preserves one backend object shared with the queue whether the queue or collection reaches it first.

The wrapper may retain the shared backend, immutable collaborators or configuration, and collection-level mutation behavior, but it retains no traversal state. Each traversal execution owns only the state it needs: the snapshot descriptor and boundary, forward scan cursor, complete winner index, and transient current decoded record. A no-block call returns a distinct normal Ruby Enumerator without opening a snapshot or scanning or decoding DLQ records. The collection does not cache current Enumerators or keep a traversal registry. Active entries retain only their record data, inspection cache, and private shared backend.

Separate collection operations, overlapping external Enumerators, nested `each`, and same-process Fiber-interleaved traversal are reentrant because their traversal executions do not share state. Each execution establishes its snapshot independently when it starts, so an append after one execution starts is excluded from that snapshot but may appear in another execution started later. This is collection-level traversal isolation, not a promise of general thread safety beyond Rage's existing Iodine/Fiber model.

One block traversal or ordinary top-level Enumerable operation uses one execution. Separate calls such as `find`, `map`, `filter_map`, `count`, and `each` start separate executions and snapshots. A normally consumed lazy pipeline uses its underlying execution. If a caller mixes external `next` or `peek` with block `enumerator.each`, Enumerable methods, or derived consumption on the same Enumerator, standard Ruby internal/external semantics apply. Rage does not synchronize those cursors or promise that they share one snapshot or descriptor.

```ruby
dead_tasks = Rage::Deferred.dead_tasks

dead_tasks.equal?(Rage::Deferred.dead_tasks) # => true
dead_tasks.each.equal?(dead_tasks.each)       # => false
```

`find_by_id` validates an exact String and delegates through a private backend capability from task 03. That capability may be named `find_dead_task(id)`. Task 02 removes task 01's eager lookup before task 03 adds the conforming implementation. `DeadTasks` does not branch on backend class. Disk scans its fixed complete-record view forwards. It remembers the newest frame-valid matching payload location and attempts that match after the scan. A rescued selected failure prints the specified diagnostic and returns `nil`. Nil also returns `nil`. A future database adapter can use an indexed query. All adapters preserve the public entry shape under their storage contracts. The API makes no cross-backend complexity guarantee. This feature does not add a database backend.

Detailed argument inspection treats the opaque execution context as private Rage-written data for the current storage version. `DeadTask` loads it lazily with a frozen Marshal object graph and asks `Rage::Deferred::Context` for positional and keyword arguments, normalizing accessor-returned `nil` values to frozen empty collections. It caches only a successful extraction and translates actual Marshal or accessor exceptions into the dedicated dead-task context error with the task ID and original cause. This MVP does not duplicate `Context` with Array shape, minimum-length, or extracted Array/Hash type checks. An incompatible context layout requires a future storage-version bump and compatibility policy rather than speculative validation inside the current version.

Argument-free `each` establishes a stable physical snapshot when actual traversal starts. Under the permanent lock, Disk opens the current live file and captures the last complete-record boundary; it does not scan or decode all earlier records while locked. After releasing the lock, enumeration frame-validates every complete record sequentially forwards from byte zero through that boundary. A Disk record is frame-valid only when it is newline-terminated inside that boundary, has valid frame structure and the expected operation, has a valid CRC, and has a non-empty authoritative outer framed ID. The outer framed ID is authoritative for selection, display, deduplication, and later exact-ID actions. Each frame-valid ID is deleted from and reinserted into the winner index with its current serialized-payload location; selection performs no top-level Marshal load and no Hash, key, type, backtrace, context, or inner/outer-ID validation.

`DeadTasks#each` accepts no arguments. A no-block Enumerator remains traversal-lazy: its first traversal execution uses the retained backend and establishes the snapshot. Collection access may resolve and initialize the backend, but collection access and Enumerator construction do not open a DLQ snapshot descriptor or scan, decode, or preload DLQ records. Disk has no decoded-record batch or batch-size setting.

Traversal finishes the forward selection scan before it yields. It then visits winners in insertion order and attempts one winner at a time. It never builds an eager array of decoded entries or payloads. It never decodes the next winner while the caller processes the current entry. It never holds the filesystem lock while decoding or yielding. Normal appends stay beyond the fixed boundary. Tail repair changes only an excluded unfinished tail. Rename-based compaction cannot invalidate the retained descriptor. Entry mutation cannot alter remaining snapshot winners during normal continuation. The Nil backend produces an empty snapshot.

Malformed frames, invalid CRCs, and an excluded incomplete tail are silently skipped as physical corruption. Enumeration and lookup frame-validate every complete record during selection. They do not perform a second diagnostic scan. Exact lookup attempts only the newest frame-valid matching payload after the scan. Physical-record skips produce no logging or terminal output.

If selected `String#undump` or `Marshal.load` raises `ArgumentError`, `NameError`, or `TypeError`, Disk writes this template to standard output:

```text
ERROR: Can't deserialize the dead-lettered task with id {ID}: ({ERROR_CLASS}) {ERROR_MESSAGE}
```

Disk does not sanitize or truncate `{ID}` or `{ERROR_MESSAGE}`. Enumeration skips that logical task and continues. Exact lookup returns `nil`. The record remains stored, and no older duplicate is used. Other selected decoding and entry-construction failures propagate unchanged. Marshal data remains trusted local application data. Frame validation does not make untrusted Marshal input safe.

Corrupt records remain skipped data; lock and filesystem failures remain operational exceptions. Snapshot-acquisition `Rage::Deferred::DeadTasksLockTimeout` and open/read/seek failures propagate unchanged from the lazy operation that encounters them. Cleanup runs in `ensure` without replacing an exception already in flight; when storage cleanup is the only failure, that original exception propagates.

Block-based collection traversal closes automatically on exhaustion, `break`, non-local exit, or exception. Ordinary terminal Enumerable operations, including early exits, receive the same cleanup when they unwind `each`. Calling `each` without a block returns a normal lazy Ruby Enumerator. Full exhaustion runs normal `ensure` cleanup. A partly consumed external or derived/lazy execution may keep its descriptor and an old unlinked inode until Ruby exhausts, unwinds, or garbage-collects it. Rage adds no public `close`, shared-cursor protocol, generation, or special cross-surface `rewind` behavior in this task.

Entry mutation remains immediate. Under task 01's rewrite-on-remove Disk algorithm, each successful entry deletion or retry removal can perform another full locked compaction. `each(&:delete)` and `each(&:retry)` are therefore supported for correctness and convenience with individual entries or small sets, but repeatedly mutating N entries can approach O(N²) aggregate storage work and is not the recommended large-set maintenance path. For a large deletion set, callers should traverse/filter into an array of exact IDs and pass that array to one `dead_tasks.delete(ids)` call; this costs O(selected IDs) additional accumulator memory on top of the traversal's complete winner index and performs one Disk compaction. This feature adds no collection-level bulk-retry primitive; optimized large-set retry is future work.

## Consequences

- `dead_tasks.each(&:retry)` and `dead_tasks.each(&:delete)` have defined mutation-safe traversal semantics but are not promoted as large-set maintenance operations.
- Enumeration retains a complete winner-location index and an open file descriptor until it ends. Every traversal that reaches its first yield uses O(distinct frame-valid outer IDs) metadata and has frame-scanned the complete snapshot without loading payloads. Beyond bounded physical-line scan storage, traversal retains at most the one decoded record being prepared for or suspended at the current yield; caller-retained yielded entries are outside this bound.
- Compaction during a long enumeration can leave the prior inode consuming disk space until the snapshot descriptor closes.
- Block-based and normally unwound terminal Enumerable traversal own cleanup automatically.
- Partial external or lazy traversal has no deterministic cancellation API. Its descriptor and an old unlinked inode may remain until that execution exhausts, unwinds, or is garbage-collected.
- Unreachable iterator state may receive best-effort IO cleanup, but garbage collection and finalizer timing remain outside the public contract.
- Standard Ruby internal/external and derived Enumerator behavior is preserved. Mixed consumption of one Enumerator may create separate executions and snapshots.
- Snapshot setup holds the permanent lock only while opening the current live file and finding its last complete-record boundary. It may scan backwards across an unfinished tail, but it does not validate the complete store under lock.
- Sequential forward frame selection, winner-index construction, selected-record re-reading, and selected top-level Marshal decoding happen synchronously in the calling Fiber. A large traversal can delay same-worker work before its first yield even though it does not monopolize the store lock for that scan. Exact lookup also scans its complete fixed-boundary view forwards before decoding its newest remembered match.
- On Disk, the outer framed ID is authoritative for selection, display, deduplication, and later actions. Storage selection does not compare it with an inner payload ID.
- Exact lookup is clearly separated as `find_by_id`; inherited `find` and `detect` retain standard predicate, fallback-callable, and no-block behavior.
- The collection stays backend-polymorphic. Exact-lookup behavior and entry shape are shared, while Disk may scan and future adapters may use indexes.
- Repeated immediate removals can approach quadratic aggregate storage work. Large deletion sets can trade an O(selected IDs) ID accumulator for one bulk compaction; large retry sets have no optimized API in this feature.
- The backend gains an internal snapshot/winner-index traversal interface that supersedes task 01's eager `list_dead_tasks` primitive. Task 02 removes the obsolete Disk/Nil `list_dead_tasks` and `find_dead_task` facade methods and Disk's eager `DeadTasksStorage#list` and `#find`; the remove primitive remains for later mutation tasks. Task 03 introduces a conforming private exact-lookup capability from scratch.
- Task 02 and task 03 describe passive entries at their completed stages. Task 04 adds the private shared backend; task 05 adds entry-owned retry. The final entry has no action handler or collection back-reference.
- Memoizing the wrapper does not serialize separate traversals: each collection operation owns an independent snapshot and traversal state, at the cost of one descriptor and one set of execution-local state for every active Disk enumeration.
- Memoizing the wrapper may initialize the backend, including its normal initialization side effects, while DLQ snapshot opening, scanning, and decoding remain deferred until traversal starts. The queue and collection share `Rage::Deferred.__backend` as their single backend identity regardless of initialization order.
- Disk retains no `RECORD_BATCH_SIZE` or decoded-record batch. Valid Enumerators remain first-advance lazy. Rescued failures can make terminal operations attempt more winners than they yield.
- Recoverable physical corruption is skipped silently. Listing and lookup inspect complete fixed-boundary snapshots before attempting selected payloads.
- Selected `ArgumentError`, `NameError`, and `TypeError` failures print the unsanitized standard-output diagnostic. Enumeration skips; lookup returns `nil`; retry returns `false`; and the record remains stored.
- Other selected decoding and entry-construction failures propagate unchanged. No selected failure falls back to an older duplicate.
- The final Disk read design retains no reverse-record traversal helper: enumeration and exact lookup both use sequential forward frame scans. Backward reading remains only a rejected traversal alternative described above, not an implementation requirement.
- Storage and lock failures remain distinguishable by their original exception classes; cleanup adds no generic traversal error layer and does not mask a primary failure.
- Current-version argument decoding relies on Rage's own context writer and `Rage::Deferred::Context` accessors. Proactive context-layout and extracted-value type validation is deferred until a storage-version change requires compatibility handling.
