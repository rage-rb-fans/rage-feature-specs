---
status: accepted
---

# ADR-001: Public dead-task object model and iteration

## Context

Issue #369 selected the public name `dead_tasks`, and the framework owner requested Enumerable traversal in small batches instead of an eager `all`, including mutation-safe support for the convenience flows `dead_tasks.each(&:retry)` and `dead_tasks.each(&:delete)`. Deleting or retrying records during offset-based pagination can skip later entries. The disk store is shared by workers and processes, may contain duplicate physical records for one logical task ID, and is compacted by rename.

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

### Option E: Active entries over a two-phase oldest-first snapshot

The collection keeps Option D's stable descriptor and short lock hold, but traversal performs two phases after releasing the lock. First, it scans every complete record backwards and frame-validates it without loading its top-level Marshal payload. The first frame-valid record found for an authoritative outer ID is the winner. A newer malformed frame or invalid CRC permits an older frame-valid record to win; a newer frame-valid record reserves its ID even if its payload later fails to decode or has an unexpected shape. The traversal stores only each winner's ID and serialized-payload location in newest-to-oldest discovery order.

Second, it visits those winning locations in reverse discovery order, re-reads and decodes them, and yields them oldest-first in fixed internally sized decoded-payload batches. The complete winner index costs O(distinct frame-valid outer IDs) memory. A selected winner is decoded only during this delivery phase, and a decoding or record-shape failure does not fall back to an older duplicate. The traversal does not retain every decoded payload. Even `first` and `take` require the full frame-only selection scan before the first yield; early exit saves only later winner re-reads and decodes beyond the current internal batch.

## External-iterator cleanup options

These options address a separate question from traversal Options A through E: how much lifecycle machinery the first public iterator should provide.

### Cleanup option 1: Standard Ruby Enumerator

Keep no-block `each` and return a normal Ruby Enumerator. Block traversal and normally unwound terminal Enumerable methods release their descriptor through `ensure`. Full external exhaustion does the same. Standard Ruby internal/external iteration, `rewind`, derived iterators, and lazy behavior remain unchanged.

This is the smallest and most idiomatic choice, but it cannot promise prompt cleanup after partial external or lazy traversal. A suspended execution can retain its descriptor and an old unlinked inode until it is exhausted, unwound, or garbage-collected.

### Cleanup option 2: Closeable shared-session Enumerator

Return a custom Enumerator with public `close`. Make `next`, `peek`, block `each`, Enumerable methods, and derived iterators share one traversal session, cursor, lookahead, lifecycle, and generation. This provides deterministic cancellation when the caller retains the root, but replaces standard Ruby iteration behavior and requires substantial cross-surface state and testing.

This remains a possible future design if real usage needs cancellable manual traversal. A separately named cursor API could provide the same capability without redefining normal Enumerator behavior.

### Cleanup option 3: Block-only traversal

Reject `each` without a block and require every traversal to occur through a block that Rage can unwind automatically. This makes cleanup simple and deterministic, but breaks the normal Enumerable expectation that no-block `each` returns an external iterator and prevents idiomatic manual enumeration requested for this API.

### Cleanup option 4: Eager materialization

Read and wrap all entries before returning an external iterator, then close the descriptor immediately. This uses only standard Enumerator behavior and has simple resource ownership, but defeats lazy retrieval and the batch-bounded payload goal. Large stores could require memory proportional to all decoded entries.

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

## Decision

Adopt traversal Option E, cleanup option 1, and `find_by_id` for public exact lookup. A standard Ruby Enumerator preserves no-block `each`, internally bounded payload decoding, and idiomatic Enumerable behavior without adding a custom stream protocol. Deterministic cancellation for partial external or lazy traversal is deliberately deferred. Cleanup option 2, a separately named closeable cursor, or a broader versioned-storage mechanism can be reconsidered if real usage requires it. `find_by_id` avoids overriding `Enumerable#find`/`detect`, whose block predicate, optional fallback callable, and no-block forms remain standard Ruby behavior.

`Rage::Deferred.dead_tasks` returns an Enumerable `DeadTasks` collection without scanning or preloading the store. Actual iteration yields `DeadTask` entries oldest-first according to their selected winning-record positions, so `dead_tasks.first(20)` returns the oldest 20 logical tasks in its snapshot. Entry data remains read-only. Task 02 introduces `DeadTask` only as a passive summary value containing stored record data; it does not retain the collection, raw backend, an operation delegate, or any other mutation dependency. Task 03 adds `find_by_id` without defining `find` or `detect`. Task 04 introduces a narrow private action delegate supporting `delete(id)` when it adds `DeadTask#delete`. Task 05 depends on task 04 and extends that same delegate with `retry(id)` when it adds `DeadTask#retry`, rather than introducing separate wiring. The delegate's concrete object may be the creating `DeadTasks` collection, but it is not the snapshot or raw backend and remains private. Equivalent collection methods support exact lookup, bulk deletion, and Rake integration.

The accessor memoizes only the shared collection wrapper: repeated `Rage::Deferred.dead_tasks` calls return the same object. When creating the wrapper, the accessor obtains the shared memoized backend through `Rage::Deferred.__backend` and passes that backend object directly to it. The wrapper never constructs a backend independently. This preserves one backend object shared with the queue whether the queue or collection reaches it first.

The wrapper may retain the shared backend, immutable collaborators or configuration, and collection-level mutation behavior, but it retains no traversal state. Each traversal execution owns only the state it needs: the snapshot descriptor and boundary, reverse-reader cursor and buffer, complete winner index, and current decoded batch. A no-block call returns a distinct normal Ruby Enumerator without opening a snapshot or scanning or decoding DLQ records. The collection does not cache current Enumerators or keep a traversal registry. `DeadTask` entries do not retain the backend.

Separate collection operations, overlapping external Enumerators, nested `each`, and same-process Fiber-interleaved traversal are reentrant because their traversal executions do not share state. Each execution establishes its snapshot independently when it starts, so an append after one execution starts is excluded from that snapshot but may appear in another execution started later. This is collection-level traversal isolation, not a promise of general thread safety beyond Rage's existing Iodine/Fiber model.

One block traversal or ordinary top-level Enumerable operation uses one execution. Separate calls such as `find`, `map`, `filter_map`, `count`, and `each` start separate executions and snapshots. A normally consumed lazy pipeline uses its underlying execution. If a caller mixes external `next` or `peek` with block `enumerator.each`, Enumerable methods, or derived consumption on the same Enumerator, standard Ruby internal/external semantics apply. Rage does not synchronize those cursors or promise that they share one snapshot or descriptor.

```ruby
dead_tasks = Rage::Deferred.dead_tasks

dead_tasks.equal?(Rage::Deferred.dead_tasks) # => true
dead_tasks.each.equal?(dead_tasks.each)       # => false
```

`find_by_id` validates an exact String and delegates through a private backend capability introduced by task 03, which may be named `find_dead_task(id)`. Task 02 removes task 01's eager lookup before task 03 adds this conforming implementation. `DeadTasks` does not branch on backend class. Disk can reverse-scan its fixed complete-record view using the frame-validity rules below, Nil returns `nil`, and a future database adapter can use an indexed query. All adapters return the same public `DeadTask` shape and newest matching logical result under their storage contract, but the API makes no cross-backend complexity guarantee. Shared backend-contract examples cover Disk and Nil now and remain reusable for later adapters; this feature does not add a database backend.

Argument-free `each` establishes a stable physical snapshot when actual traversal starts. Under the permanent lock, Disk opens the current live file and captures the last complete-record boundary; it does not scan or decode all earlier records while locked. After releasing the lock, a bounded reverse-line reader frame-validates every complete record. A Disk record is frame-valid only when it is newline-terminated inside that boundary, has valid frame structure and the expected operation, has a valid CRC, and has a non-empty authoritative outer framed ID. The outer framed ID is authoritative for selection, display, deduplication, and later exact-ID actions. An ID enters the winner index immediately after those checks; selection performs no top-level Marshal load and no Hash, key, type, backtrace, context, or inner/outer-ID validation.

`DeadTasks#each` exposes no public batch-size parameter. A no-block Enumerator remains traversal-lazy: its first traversal execution uses the retained backend and establishes the snapshot. Collection access may resolve and initialize the backend, but collection access and Enumerator construction do not open a DLQ snapshot descriptor or scan, decode, or preload DLQ records. Disk chooses a fixed internal decoded-payload batch size whose numeric value is private and can change without a public compatibility impact.

Traversal must finish the reverse selection scan and build its complete winner-location index before it yields. It then visits winners in reverse discovery order, re-reading and decoding at most one internal batch at a time, and yields them oldest-first. It never builds an eager array of every decoded entry or payload, and never holds the filesystem lock while decoding or yielding. Normal appends stay beyond the fixed boundary, task 01 tail repair changes only an excluded unfinished tail, and rename-based compaction cannot invalidate the retained descriptor. If traversal continues normally, mutating a yielded entry does not skip, duplicate, reorder, or replace entries remaining in the snapshot. The Nil backend produces an empty snapshot.

Malformed frames, invalid CRCs, and an excluded incomplete tail are silently skipped as recoverable physical corruption. Enumeration frame-validates every complete record during its required selection scan and does not perform another scan merely to diagnose corruption. Exact lookup stops at the newest frame-valid matching outer ID and decodes that selected record without inspecting older duplicates. Neither path emits logging or terminal output for skipped records or fragments. A selected top-level Marshal decoding error propagates unchanged, and neither a decoding failure nor an unexpected decoded shape triggers fallback to an older duplicate. Marshal data remains trusted local application data; frame validation does not make untrusted Marshal input safe to decode.

Corrupt records remain skipped data; lock and filesystem failures remain operational exceptions. Snapshot-acquisition `Rage::Deferred::DeadTasksLockTimeout` and open/read/seek failures propagate unchanged from the lazy operation that encounters them. Cleanup runs in `ensure` without replacing an exception already in flight; when storage cleanup is the only failure, that original exception propagates.

Block-based collection traversal closes automatically on exhaustion, `break`, non-local exit, or exception. Ordinary terminal Enumerable operations, including early exits, receive the same cleanup when they unwind `each`. Calling `each` without a block returns a normal lazy Ruby Enumerator. Full exhaustion runs normal `ensure` cleanup. A partly consumed external or derived/lazy execution may keep its descriptor and an old unlinked inode until Ruby exhausts, unwinds, or garbage-collects it. Rage adds no public `close`, shared-cursor protocol, generation, or special cross-surface `rewind` behavior in this task.

Entry mutation remains immediate. Under task 01's rewrite-on-remove Disk algorithm, each successful entry deletion or retry removal can perform another full locked compaction. `each(&:delete)` and `each(&:retry)` are therefore supported for correctness and convenience with individual entries or small sets, but repeatedly mutating N entries can approach O(N²) aggregate storage work and is not the recommended large-set maintenance path. For a large deletion set, callers should traverse/filter into an array of exact IDs and pass that array to one `dead_tasks.delete(ids)` call; this costs O(selected IDs) additional accumulator memory on top of the traversal's complete winner index and performs one Disk compaction. This feature adds no collection-level bulk-retry primitive; optimized large-set retry is future work.

## Consequences

- `dead_tasks.each(&:retry)` and `dead_tasks.each(&:delete)` have defined mutation-safe traversal semantics but are not promoted as large-set maintenance operations.
- Enumeration retains a complete winner-location index and an open file descriptor until it ends. Every traversal that reaches its first yield uses O(distinct frame-valid outer IDs) metadata and has frame-scanned the complete snapshot without loading payloads. Decoded record-payload memory remains internally batch-bounded.
- Compaction during a long enumeration can leave the prior inode consuming disk space until the snapshot descriptor closes.
- Block-based and normally unwound terminal Enumerable traversal own cleanup automatically.
- Partial external or lazy traversal has no deterministic cancellation API. Its descriptor and an old unlinked inode may remain until that execution exhausts, unwinds, or is garbage-collected.
- Unreachable iterator state may receive best-effort IO cleanup, but garbage collection and finalizer timing remain outside the public contract.
- Standard Ruby internal/external and derived Enumerator behavior is preserved. Mixed consumption of one Enumerator may create separate executions and snapshots.
- Snapshot setup holds the permanent lock only while opening the current live file and finding its last complete-record boundary. It may scan backwards across an unfinished tail, but it does not validate the complete store under lock.
- Reverse frame selection, winner-index construction, selected-record re-reading, and selected top-level Marshal decoding happen synchronously in the calling Fiber. A large traversal can delay same-worker work before its first yield even though it does not monopolize the store lock for that scan.
- On Disk, the outer framed ID is authoritative for selection, display, deduplication, and later actions. Storage selection does not compare it with an inner payload ID.
- Exact lookup is clearly separated as `find_by_id`; inherited `find` and `detect` retain standard predicate, fallback-callable, and no-block behavior.
- The collection stays backend-polymorphic. Exact-lookup behavior and entry shape are shared, while Disk may scan and future adapters may use indexes.
- Repeated immediate removals can approach quadratic aggregate storage work. Large deletion sets can trade an O(selected IDs) ID accumulator for one bulk compaction; large retry sets have no optimized API in this feature.
- The backend gains an internal snapshot/winner-index/batch traversal interface that supersedes task 01's eager `list_dead_tasks` primitive. Task 02 removes the obsolete Disk/Nil `list_dead_tasks` and `find_dead_task` facade methods and Disk's eager `DeadTasksStorage#list` and `#find`; the remove primitive remains for later mutation tasks. Task 03 introduces a conforming private exact-lookup capability from scratch.
- Task 02 entries are passive read-only values with no mutation dependency. Task 04 turns them into action objects by introducing one narrow private delegate, and task 05 extends that same mechanism; the final design does not mandate a concrete collection back-reference or expose the delegate.
- Memoizing the wrapper does not serialize separate traversals: each collection operation owns an independent snapshot and traversal state, at the cost of one descriptor and one set of execution-local state for every active Disk enumeration.
- Memoizing the wrapper may initialize the backend, including its normal initialization side effects, while DLQ snapshot opening, scanning, and decoding remain deferred until traversal starts. The queue and collection share `Rage::Deferred.__backend` as their single backend identity regardless of initialization order.
- The internal payload batch size is not a public API or configuration surface; valid Enumerators remain first-advance lazy.
- Recoverable physical corruption encountered by listing or exact lookup is skipped silently. Listing inspects the complete snapshot for frame-only winner selection, while exact lookup does not inspect records older than its newest frame-valid outer-ID match. Selected-payload failures propagate without older fallback.
- Storage and lock failures remain distinguishable by their original exception classes; cleanup adds no generic traversal error layer and does not mask a primary failure.
