---
status: accepted
---

# ADR-001: Public dead-task object model and iteration

## Context

Issue #369 selected the public name `dead_tasks`, and the framework owner requested Enumerable traversal in small batches instead of an eager `all`, including mutation-safe support for the convenience flows `dead_tasks.each(&:retry)` and `dead_tasks.each(&:delete)`. Deleting or retrying records during offset-based pagination can skip later entries. The disk store is shared by workers and processes, may contain duplicate physical records for one logical task ID, and is compacted by rename.

The public model must support basic inspection even when the task class no longer exists, avoid holding the store lock while arbitrary operator code runs, and make iteration behavior observable rather than relying on unstable offsets.

## Traversal options considered

### Option A: Inert entries and collection-only mutations

`DeadTask` is a value object. Operators call `dead_tasks.retry(entry.id)` or `dead_tasks.delete(entry.id)`. Batching can remain weakly consistent, but the concise `each(&:retry)` form is unavailable and callers still can mutate the collection manually during iteration.

### Option B: Active entries over offset pagination

Entries expose `retry` and `delete`, and `each` repeatedly calls `list(limit:, offset:)`. This is simple, but deleting or retrying yielded entries shifts later offsets and silently skips records. Concurrent additions can similarly duplicate or omit results.

### Option C: Active entries over a stable, bounded-payload snapshot

The collection exposes both collection mutations and entry convenience methods. At enumeration start, the disk backend scans the whole store under its shared lock, validates records, and records the latest byte location for each logical ID in insertion order. It retains an open read descriptor as the snapshot, releases the lock before yielding, and decodes only one batch at a time. Rename-based compaction cannot invalidate the open descriptor; records appended after snapshot setup are excluded. The index costs memory proportional to the number of distinct logical IDs, while decoded record payloads are bounded by the requested batch size. The design does not bound total snapshot memory independently of store cardinality.

This option gives direct seeks to selected records, but the complete scan holds the lock for work proportional to the whole store. It also chooses one offset before later payload decoding can prove that the selected record is actually readable and schema-valid.

### Option D: Active entries over a lazy reverse snapshot

The collection keeps the same public entry and mutation surfaces as Option C, but snapshot setup does not build an ID/offset index. While holding the permanent dead-task lock, Disk opens the live file and captures the byte boundary immediately after its last complete newline-terminated record. It releases the lock, retains the descriptor, and reads complete records backwards from that boundary as the caller requests entries.

Traversal validates frames, top-level payloads, schemas, and inner/outer ID consistency lazily. It adds an ID to a per-traversal `seen_ids` set only after finding a fully valid record. This naturally selects the newest fully valid duplicate and lets an invalid newer duplicate fall back to an older valid copy. Decoded entries remain batch-bounded. A complete traversal retains `seen_ids` metadata proportional to the number of distinct valid IDs encountered, while an early-stopped traversal avoids processing untouched older records.

## External-iterator cleanup options

These options address a separate question from traversal Options A through D: how much lifecycle machinery the first public iterator should provide.

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

Adopt traversal Option D, cleanup option 1, and `find_by_id` for public exact lookup. A standard Ruby Enumerator preserves no-block `each`, lazy reverse traversal, bounded payload decoding, and idiomatic Enumerable behavior without adding a custom stream protocol. Deterministic cancellation for partial external or lazy traversal is deliberately deferred. Cleanup option 2, a separately named closeable cursor, or a broader versioned-storage mechanism can be reconsidered if real usage requires it. `find_by_id` avoids overriding `Enumerable#find`/`detect`, whose block predicate, optional fallback callable, and no-block forms remain standard Ruby behavior.

`Rage::Deferred.dead_tasks` returns an Enumerable `DeadTasks` collection without scanning or preloading the store. Actual iteration yields `DeadTask` entries newest-first, and entry data remains read-only. Task 02 introduces `DeadTask` only as a passive summary value containing stored record data; it does not retain the collection, raw backend, an operation delegate, or any other mutation dependency. Task 03 adds `find_by_id` without defining `find` or `detect`. Task 04 introduces a narrow private action delegate supporting `delete(id)` when it adds `DeadTask#delete`. Task 05 depends on task 04 and extends that same delegate with `retry(id)` when it adds `DeadTask#retry`, rather than introducing separate wiring. The delegate's concrete object may be the creating `DeadTasks` collection, but it is not the snapshot or raw backend and remains private. Equivalent collection methods support exact lookup, bulk deletion, and Rake integration.

The accessor memoizes only the shared collection wrapper: repeated `Rage::Deferred.dead_tasks` calls return the same object. Creating that wrapper does not call `Rage::Deferred.__backend`. Instead, the wrapper retains an immutable lazy resolver, or an equivalent collaborator, that reaches storage only through `Rage::Deferred.__backend`. It never constructs a backend independently. This preserves one backend object shared with the queue whether the queue or collection reaches it first.

The wrapper may also retain immutable collaborators or configuration and collection-level mutation behavior, but it retains no traversal state. Each traversal execution owns only the state it needs: the snapshot descriptor and boundary, reverse-reader cursor and buffer, `seen_ids`, and current decoded batch. A no-block call returns a distinct normal Ruby Enumerator without resolving the backend. The collection does not cache current Enumerators or keep a traversal registry. `DeadTask` entries retain neither the backend nor its resolver.

Separate collection operations, overlapping external Enumerators, nested `each`, and same-process Fiber-interleaved traversal are reentrant because their traversal executions do not share state. Each execution establishes its snapshot independently when it starts, so an append after one execution starts is excluded from that snapshot but may appear in another execution started later. This is collection-level traversal isolation, not a promise of general thread safety beyond Rage's existing Iodine/Fiber model.

One block traversal or ordinary top-level Enumerable operation uses one execution. Separate calls such as `find`, `map`, `filter_map`, `count`, and `each` start separate executions and snapshots. A normally consumed lazy pipeline uses its underlying execution. If a caller mixes external `next` or `peek` with block `enumerator.each`, Enumerable methods, or derived consumption on the same Enumerator, standard Ruby internal/external semantics apply. Rage does not synchronize those cursors or promise that they share one snapshot or descriptor.

```ruby
dead_tasks = Rage::Deferred.dead_tasks

dead_tasks.equal?(Rage::Deferred.dead_tasks) # => true
dead_tasks.each.equal?(dead_tasks.each)       # => false
```

`find_by_id` validates an exact String and delegates through a private backend capability, such as the existing `find_dead_task(id)`. `DeadTasks` does not branch on backend class. Disk can reverse-scan its fixed complete-record view using the validation rules below, Nil returns `nil`, and a future database adapter can use an indexed query. All adapters return the same public `DeadTask` shape and newest-fully-valid logical result, but the API makes no cross-backend complexity guarantee. Shared backend-contract examples cover Disk and Nil now and remain reusable for later adapters; this feature does not add a database backend.

`each(batch_size: 100)` establishes a stable physical snapshot when actual traversal starts. Under the permanent lock, Disk opens the current live file and captures the last complete-record boundary; it does not scan or decode all earlier records. The descriptor remains open while a bounded reverse-line reader validates records lazily. The outer framed ID is authoritative. An ID enters `seen_ids` only after the record's frame, CRC, top-level Marshal payload, required schema and types, and inner/outer ID match are all valid. The opaque execution context must have its stored String type but is not deserialized during listing. This makes the first fully valid record encountered for an ID the logical entry while allowing fallback past invalid newer duplicates.

`DeadTasks#each` validates `batch_size` immediately, before it returns a no-block Enumerator or starts block traversal. Invalid values raise `ArgumentError` without resolving the backend or opening or inspecting the store. A valid no-block Enumerator remains lazy: its first traversal execution resolves storage through `Rage::Deferred.__backend` and establishes the snapshot. Collection access and Enumerator construction do neither.

Traversal decodes and wraps at most one requested batch at a time and never builds an eager array of every entry or a complete offset index. It never holds the filesystem lock while decoding or yielding. Normal appends stay beyond the fixed boundary, task 01 tail repair changes only an excluded unfinished tail, and rename-based compaction cannot invalidate the retained descriptor. If traversal continues normally, mutating a yielded entry does not skip, duplicate, reorder, or replace entries remaining in the snapshot. The Nil backend produces an empty snapshot.

Invalid physical records and an excluded incomplete tail are silently skipped as recoverable corruption. Lazy traversal validates invalid records only when it reaches them and does not read untouched older records merely to diagnose corruption. Exact lookup likewise skips invalid records it inspects, continues past invalid newer duplicates, and stops at the first fully valid matching record or the complete-record boundary. Neither path emits logging or terminal output for skipped records or fragments.

Corrupt records remain skipped data; lock and filesystem failures remain operational exceptions. Snapshot-acquisition `Rage::Deferred::DeadTasksLockTimeout` and open/read/seek failures propagate unchanged from the lazy operation that encounters them. Cleanup runs in `ensure` without replacing an exception already in flight; when storage cleanup is the only failure, that original exception propagates.

Block-based collection traversal closes automatically on exhaustion, `break`, non-local exit, or exception. Ordinary terminal Enumerable operations, including early exits, receive the same cleanup when they unwind `each`. Calling `each` without a block returns a normal lazy Ruby Enumerator. Full exhaustion runs normal `ensure` cleanup. A partly consumed external or derived/lazy execution may keep its descriptor and an old unlinked inode until Ruby exhausts, unwinds, or garbage-collects it. Rage adds no public `close`, shared-cursor protocol, generation, or special cross-surface `rewind` behavior in this task.

Entry mutation remains immediate. Under task 01's rewrite-on-remove Disk algorithm, each successful entry deletion or retry removal can perform another full locked compaction. `each(&:delete)` and `each(&:retry)` are therefore supported for correctness and convenience with individual entries or small sets, but repeatedly mutating N entries can approach O(N²) aggregate storage work and is not the recommended large-set maintenance path. For a large deletion set, callers should traverse/filter into an array of exact IDs and pass that array to one `dead_tasks.delete(ids)` call; this costs O(selected IDs) additional accumulator memory on top of the traversal's incremental `seen_ids` set and performs one Disk compaction. This feature adds no collection-level bulk-retry primitive; optimized large-set retry is future work.

## Consequences

- `dead_tasks.each(&:retry)` and `dead_tasks.each(&:delete)` have defined mutation-safe traversal semantics but are not promoted as large-set maintenance operations.
- Enumeration retains an incremental set of fully valid IDs encountered and an open file descriptor until it ends. A complete traversal still uses O(distinct valid IDs) metadata, but an early-stopped traversal does not build metadata for untouched older records. Decoded record-payload memory is bounded by the batch size.
- Compaction during a long enumeration can leave the prior inode consuming disk space until the snapshot descriptor closes.
- Block-based and normally unwound terminal Enumerable traversal own cleanup automatically.
- Partial external or lazy traversal has no deterministic cancellation API. Its descriptor and an old unlinked inode may remain until that execution exhausts, unwinds, or is garbage-collected.
- Unreachable iterator state may receive best-effort IO cleanup, but garbage collection and finalizer timing remain outside the public contract.
- Standard Ruby internal/external and derived Enumerator behavior is preserved. Mixed consumption of one Enumerator may create separate executions and snapshots.
- Snapshot setup holds the permanent lock only while opening the current live file and finding its last complete-record boundary. It may scan backwards across an unfinished tail, but it does not validate the complete store under lock.
- Reverse reading and top-level Marshal decoding happen synchronously in the calling Fiber as entries are requested. A large traversal can still delay same-worker work even though it no longer monopolizes the store lock for a complete setup scan.
- On Disk, the outer framed ID is authoritative for display, deduplication, and later actions. A mismatched inner ID invalidates that physical record.
- Exact lookup is clearly separated as `find_by_id`; inherited `find` and `detect` retain standard predicate, fallback-callable, and no-block behavior.
- The collection stays backend-polymorphic. Exact-lookup behavior and entry shape are shared, while Disk may scan and future adapters may use indexes.
- Repeated immediate removals can approach quadratic aggregate storage work. Large deletion sets can trade an O(selected IDs) ID accumulator for one bulk compaction; large retry sets have no optimized API in this feature.
- The backend gains an internal snapshot/batch interface in addition to the task-01 list/find/remove primitives.
- Task 02 entries are passive read-only values with no mutation dependency. Task 04 turns them into action objects by introducing one narrow private delegate, and task 05 extends that same mechanism; the final design does not mandate a concrete collection back-reference or expose the delegate.
- Memoizing the wrapper does not serialize separate traversals: each collection operation owns an independent snapshot and traversal state, at the cost of one descriptor and one set of execution-local state for every active Disk enumeration.
- Memoizing the wrapper does not eagerly initialize storage. The queue and collection share `Rage::Deferred.__backend` as their single backend identity regardless of initialization order.
- Invalid `batch_size` values fail before an Enumerator can be returned and before the backend is touched; valid Enumerators remain first-advance lazy.
- Recoverable corruption encountered by listing or exact lookup is skipped silently. Lazy traversal does not inspect untouched older records, and successful exact lookup does not inspect records older than its fully valid match.
- Storage and lock failures remain distinguishable by their original exception classes; cleanup adds no generic traversal error layer and does not mask a primary failure.
