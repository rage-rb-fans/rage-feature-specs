---
status: todo
---

# List dead tasks

## Goal

Expose the durable dead-task store as a newest-first Enumerable. The collection yields passive, read-only summaries. It decodes record payloads in bounded batches instead of loading every entry at once.

## Context

Task 01 added the Disk/Nil backend facade and durable dead-task storage. Each disk record contains summary failure metadata and an opaque marshaled execution context.

The disk file can contain more than one physical record with the same task ID. The public collection treats those physical records as one **logical dead task** and yields the newest record that is fully valid. A newer damaged record must not hide an older usable copy.

This is the first public-API task and depends only on task 01.

Task 02 owns:

- REQ-05 through REQ-07;
- only these listing fields from REQ-09: `id`, `task_class`, `attempts`, `enqueued_at`, and `failed_at`;
- only the enumeration parts of REQ-17: schema validation, silent corruption handling, and invalid-newer-duplicate fallback;
- the listing-related parts of REQ-19 and REQ-20;
- REQ-24 and REQ-25;
- only the enumeration and cleanup parts of REQ-26;
- REQ-27; and
- the public-collection parts of AC-02, AC-03, AC-07, and AC-11.

Task 02 does not own REQ-08 or REQ-10. It also does not own the exact-lookup parts of REQ-17 and REQ-26, or Task 03's detailed exception and context inspection.

Task 02 defines the stable traversal behavior that later entry actions rely on. It does not add `delete`, `retry`, or their private mutation wiring.

Task 03 adds exact-ID inspection through `find_by_id` and lazy context decoding. It keeps the inherited Enumerable `find`. Task 04 adds deletion and the first private entry-action wiring. Task 05 extends that wiring for retry.

Follow [ADR-001](../adr/001-public-dead-task-object-model.md).

### Terms used below

- A **stable snapshot** is the sequence of complete physical records ending at a fixed byte boundary captured when traversal starts. Later changes to the live path do not change that sequence.
- A **logical dead task** is the newest fully valid record for one task ID within that snapshot.
- A **payload** is the top-level stored record data that Rage decodes and wraps as a `DeadTask`. Its opaque execution context remains serialized in this task.
- A **read descriptor** is the open file handle used to read the snapshot.
- A **traversal execution** is one run of the collection's `each` body. It owns its snapshot descriptor and boundary, reverse-reader cursor and buffer, `seen_ids`, and decoded batch.
- The **snapshot boundary** is the byte position immediately after the last complete newline-terminated record that exists when traversal starts. An unfinished tail after that position is not part of the snapshot.
- **Rename-based compaction** is task 01's removal process: write surviving records to a temporary file, then rename that file over the live data path.

## Requirements

### Public collection and entries

- Add the memoized `Rage::Deferred.dead_tasks` accessor.
- The accessor returns a `Rage::Deferred::DeadTasks` collection. Creating or memoizing this collection must not call `Rage::Deferred.__backend`.
- Repeated accessor calls return the same collection object. The wrapper may retain:
  - an immutable lazy backend resolver or an equivalent collaborator;
  - immutable collaborators or configuration; and
  - the collection-level mutation behavior added by later tasks.
- When a collection operation needs storage, the resolver must go through `Rage::Deferred.__backend`. The collection must not construct a backend separately. This guarantees that the collection and queue use the identical backend object whether the queue or collection initializes it first.
- Obtaining the collection must not resolve the backend or open, create, scan, or preload the store.
- `DeadTasks` includes `Enumerable`.
- The shared collection wrapper must not retain state that belongs to one traversal. In particular, it must not retain a snapshot descriptor or boundary, reverse-reader cursor or buffer, `seen_ids`, decoded batch, current Enumerator, or traversal registry.
- Define `Rage::Deferred::DeadTask` as a passive, read-only summary value. At this stage, an entry contains only stored record data.
- A `DeadTask` must not retain the `DeadTasks` collection, the raw backend, the backend resolver, an operation delegate, or any other mutation dependency.
- Entry construction and raw backend records remain private.

### Enumeration and snapshot boundary

- Block-based `each(batch_size: 100)` is the primary and recommended traversal form.
- Calling `each` without a block returns a new normal Ruby `Enumerator`, as an `enum_for`-style implementation would. It has standard Ruby 3.3 behavior. Returning this unadvanced Enumerator must not resolve the backend. This task does not add `close` or another Rage-specific lifecycle method to that Enumerator.
- Each block traversal and each ordinary top-level Enumerable operation invokes `DeadTasks#each` and owns one traversal execution. Separate calls such as `find`, `map`, `filter_map`, `count`, or `each` use separate cursors and snapshots. Repeated calls on the memoized collection do not share traversal state.
- `batch_size` must be a positive Integer. `DeadTasks#each` validates it immediately, before returning a no-block Enumerator or starting block traversal. Any other value raises `ArgumentError`. This validation must not call the backend resolver or open, create, scan, or decode the store.
- Returning a valid no-block Enumerator must not establish or decode a snapshot. Its first traversal execution establishes the snapshot.
- Traversal yields logical dead tasks newest-first.
- When a valid traversal execution starts, resolve the backend through the wrapper's lazy resolver. For Disk, then acquire task 01's permanent dead-task lock, open the current live data file, and locate the snapshot boundary. Release the lock after capturing the descriptor and boundary. Setup must not scan or validate every complete record.
- If the file ends with an incomplete tail that has no terminating newline, silently exclude the entire tail. Setup may read backwards in fixed-size chunks to find the preceding newline while holding the lock. It must not parse or decode earlier complete records.
- Never read a snapshot record past the captured boundary. An append after setup, including an append that first repairs an incomplete tail through task 01's real `add` path, is not part of that traversal.
- Keep the snapshot descriptor open. If compaction renames a replacement over the live path, the descriptor still reads the original inode up to the fixed boundary.
- Appending, repairing a torn tail, or rename-compacting the live store between yields must not skip, duplicate, reorder, or replace remaining logical entries when traversal continues normally.
- Every traversal execution created by a separate collection operation owns its own snapshot descriptor, boundary, cursor, reverse-reader buffer, `seen_ids`, and decoded batch. This includes overlapping external Enumerators, nested block traversal, and same-process Fiber-interleaved traversal. Exhausting, breaking, or raising in one collection traversal must not alter another.
- Each traversal establishes its snapshot independently. For example, first advance one Enumerator, append a new record, and then first advance a second Enumerator. The first snapshot excludes the new record, while the second may include it.
- This task guarantees reentrant use of the memoized wrapper and isolation between traversals. It does not introduce a broader thread-safety promise beyond Rage's existing Iodine/Fiber execution model.
- Tests must start and pause an enumeration, change the live store through task 01's internal backend methods, and then resume enumeration. They must not depend on the public `delete` or `retry` methods added by later tasks.

### Standard Enumerator behavior

- A normally consumed lazy pipeline uses its underlying traversal execution. One ordinary top-level Enumerable method call likewise uses one traversal execution.
- Ruby may keep internal and external iteration state separately on one Enumerator. If a caller mixes `next` or `peek` with block `enumerator.each`, Enumerable methods, or derived iterators, standard Ruby behavior applies. Rage does not synchronize those cursors or promise that they use one snapshot or descriptor.
- Standard `next_values`, `peek_values`, `rewind`, `with_index`, `lazy`, and other Enumerator behavior remains unchanged. This task adds no shared lookahead, generation tracking, or cross-surface exhaustion rules.
- Callers that need independent complete traversals should start separate collection operations. Mixed manual consumption of one returned Enumerator is allowed but receives no Rage-specific lifecycle guarantee.
- Separate collection traversals are reentrant under Rage's single-threaded Fiber model. This task adds no broader thread-safety guarantee.

### Lazy reverse traversal and memory

- Read complete physical records backwards from the snapshot boundary. Validate a record only when traversal reaches it.
- Maintain a separate `seen_ids` set for each traversal. Add an ID only after its physical record passes every validity check below. Once an ID is in the set, an older physical record with the same ID cannot replace it.
- A newer invalid duplicate does not add its ID to `seen_ids`. Continue backwards so an older, fully valid record with that ID can become the logical entry.
- Decode and wrap no more than `batch_size` valid payloads at one time. Do not build or cache an eager array of every entry, every record offset, or every payload.
- Traversal memory includes the incremental `seen_ids` set and at most one decoded payload batch. It also includes the reverse reader's bounded byte buffer and the bytes for the physical record currently being checked. A complete traversal can therefore retain O(distinct valid IDs encountered) metadata. Its total memory use is not bounded independently of store size.
- Stopping early must stop further reverse traversal. Do not parse or decode records that belong only to older batches that have not been requested. Work already performed for the current batch remains bounded by `batch_size`.
- A caller may still deliberately materialize all public entries with a standard Enumerable method such as `to_a`; caller-retained memory is outside the iterator guarantee.

### Record validity and duplicate selection

A physical record is fully valid for Task 02 only when all of these checks pass:

1. The record is complete and ends with a newline inside the snapshot boundary.
2. Its frame has the expected operation, separators, and checksum width, and its CRC matches.
3. The outer framed task ID is a non-empty String. This outer ID is authoritative for deduplication and all later exact-ID actions.
4. Decoding the top-level dumped payload with `Marshal.load` succeeds and returns a Hash.
5. The Hash contains the symbol keys `:id`, `:task_class`, `:attempts`, `:enqueued_at`, `:failed_at`, `:exception_class`, `:exception_message`, `:backtrace`, and `:context`.
6. Every required value has the expected type:
   - `:id`, `:task_class`, `:exception_class`, and `:exception_message` are Strings.
   - `:attempts`, `:enqueued_at`, and `:failed_at` are Integers.
   - `:backtrace` is an Array that contains only Strings.
   - `:context` is a String that contains the opaque stored bytes.
7. The inner `record[:id]` exactly equals the authoritative outer framed ID.

Extra Hash keys do not invalidate a record. Task 02 verifies that `:context` has the expected stored-byte type. It never deserializes those bytes. Task 03 defines what happens when a valid top-level record contains an execution context that cannot be decoded.

When any required check fails, silently skip that physical record and continue reading. Snapshot setup likewise silently excludes an incomplete tail.

Skipped records and fragments represent recoverable corruption. They are not operational failures. Do not treat an invalid record as a valid duplicate.

### Recoverable corruption and operational failures

- Invalid physical records and an excluded incomplete tail are skipped without logging or terminal output. Traversal validates invalid records only as lazy reverse reading reaches them. Do not parse untouched older records merely to diagnose corruption.
- `Rage::Deferred::DeadTasksLockTimeout` during snapshot acquisition is an operational failure, not a skipped item. Filesystem errors from open, read, seek, or descriptor cleanup are also operational failures. Propagate these errors unchanged from the lazy advancement that performs the operation. Do not translate them into a generic listing error. For block traversal, they propagate from the `each` call.
- Release every acquired lock and each normally unwound traversal descriptor through `ensure`. If traversal, application code, or storage work is already raising an exception, cleanup must not replace it. If storage cleanup is the only failing operation, propagate that cleanup exception instead of suppressing it.

### Locking and cleanup

- Hold the dead-task lock only while opening the live file and capturing the snapshot boundary. Release the lock before frame validation, `Marshal.load`, entry wrapping, or yielding.
- Do not hold the lock while calling application code. A caller's block can be slow or can start another store operation that needs the same lock.
- Close the read descriptor automatically when traversal is exhausted. Also close it when block or internal traversal unwinds through `break`, another non-local exit, or an exception. This includes ordinary terminal Enumerable methods and their early-exit paths when they unwind `each`.
- After compaction, a long-lived descriptor can keep the old, unlinked snapshot file on disk. The file's space is released when the descriptor closes.

For no-block and lazy iteration:

- Full exhaustion runs the traversal's normal `ensure` cleanup.
- A partially consumed external or derived/lazy Enumerator can keep its descriptor open if the traversal execution remains suspended. After rename-based compaction, that descriptor can also keep the old unlinked inode on disk.
- The descriptor is released only when that execution later exhausts, Ruby unwinds it, or the unreachable execution is garbage-collected. The public API does not promise when garbage collection or an optional finalizer runs.
- This task adds no public `close` method, idle timeout, or other deterministic cancellation mechanism. Prefer block-based traversal or a terminal Enumerable operation when prompt cleanup matters.

### Summary data, backend behavior, and compatibility

- In this task, `DeadTask` exposes these readers:
  - `id` returns the logical task ID as a String. On Disk this is the authoritative outer framed ID.
  - `task_class` returns a String.
  - `attempts` returns an Integer.
  - `enqueued_at` returns a `Time` created from the stored Integer.
  - `failed_at` returns a `Time` created from the stored Integer.
- Returned mutable values must be defensive copies or frozen values. Callers must not receive the backend record itself.
- Listing and summary readers must not resolve or instantiate the stored task class.
- Listing and summary readers must not deserialize the opaque execution context.
- With the Nil backend, the accessor returns the same public collection type and enumeration yields nothing.
- Nil enumeration must not create storage files or background work.
- Keep storage and snapshot helpers private. Use YARD `@private` where Ruby visibility cannot express that boundary.
- Document all of the following:
  - the accessor, collection, and passive summary readers;
  - lazy backend resolution and the shared backend identity used by the collection and queue;
  - immediate `batch_size` validation and the fact that validation has no storage side effects;
  - stable snapshots, lazy reverse traversal, complete-traversal `seen_ids` memory, and decoded-payload batching;
  - silent skipping of invalid records and an incomplete tail;
  - unchanged propagation of operational errors;
  - automatic cleanup for block traversal and normally unwound terminal Enumerable operations;
  - standard Ruby Enumerator behavior and the absence of a custom `close` API;
  - nondeterministic descriptor cleanup for abandoned partial external or lazy traversal;
  - the lifetime of an old, unlinked inode; and
  - the sensitive-data risk.
- Reverse file reading and top-level Marshal decoding are synchronous work in the calling Fiber. Document this behavior. Snapshot setup no longer scans the complete store under lock, but a large traversal can still delay other work in that worker.
- Preserve Ruby 3.3.0 compatibility and the existing `config.deferred.backend` contract.
- Do not add polling or work to successful deferred-task execution.

## Design

### How traversal works

1. `Rage::Deferred.dead_tasks` creates or returns the collection. It keeps a lazy resolver but does not call `Rage::Deferred.__backend` or touch the store.
2. `DeadTasks#each` validates `batch_size` without resolving the backend or touching the store. A no-block call returns a normal lazy Enumerator. The memoized collection does not register executions or copy their state onto itself.
3. When a block, Enumerable operation, or external Enumerator actually starts traversal, the collection resolves storage through `Rage::Deferred.__backend`. Disk then takes the permanent lock and opens the current live data file. If the queue already initialized the backend, the resolver returns that same object. If traversal initializes it first, the queue later receives that same object.
4. Disk finds the last newline at or before the current end of file. The fixed snapshot boundary is the position immediately after that newline. If the file already ends in a newline, the boundary is its current size. If the file contains no newline, the boundary is zero. An incomplete tail is silently excluded.
5. Disk releases the lock but keeps the read descriptor. All later snapshot reads are limited to bytes before the captured boundary.
6. A bounded reverse-line reader walks complete records from the boundary toward byte zero. It frame-validates and top-level-decodes records lazily.
7. For each record, Disk checks the complete schema and the inner/outer ID match. An invalid record is silently skipped and does not reserve its ID. If a fully valid record's ID has not been seen, Disk adds the ID to `seen_ids`, wraps the record, and includes it in the next bounded batch. Older records with that ID cannot be yielded.
8. The traversal execution yields entries newest-first. It performs no further storage work while suspended.
9. When traversal exhausts or block/internal control flow unwinds, `ensure` closes that execution's descriptor without masking an exception already in flight.

This design makes the snapshot stable in three different storage cases:

- A normal append writes after the fixed boundary, so the iterator never reads it.
- Task 01's torn-tail repair truncates only the excluded bytes after the last complete newline. It then appends at or after the same boundary. Earlier snapshot bytes do not change.
- Rename-based compaction changes the live path, but the open descriptor still refers to the original snapshot inode.

Nil provides an empty version of the same internal traversal interface.

At the public boundary, wrap backend record hashes in passive `DeadTask` values. Keep opaque context bytes as private record data. This allows task 03 to add detailed inspection later. Do not decode or expose those bytes in this task.

Block-based traversal needs no caller cleanup:

```ruby
dead_tasks.each do |entry|
  break if selected?(entry)
end
```

No-block `each` returns a normal Ruby Enumerator:

```ruby
iterator = dead_tasks.each(batch_size: 100)
inspect_one(iterator.next)
```

If this external traversal remains only partly consumed, its descriptor may stay open until Ruby exhausts, unwinds, or garbage-collects that execution. The same limitation applies to partly consumed lazy and derived Enumerators. Use block traversal or a terminal Enumerable operation when prompt cleanup matters.

Do not rely on mixed consumption of one Enumerator sharing a cursor. Ruby may treat external `next`/`peek` and internal block or derived consumption as separate executions. Start separate collection operations when independent traversal is intended.

The memoized wrapper and its Enumerators have different identities and responsibilities:

```ruby
dead_tasks = Rage::Deferred.dead_tasks

dead_tasks.equal?(Rage::Deferred.dead_tasks) # => true
dead_tasks.each.equal?(dead_tasks.each)       # => false
```

Snapshots also begin independently on first advancement:

```ruby
first = dead_tasks.each
first.next # snapshot A starts

append_new_task

second = dead_tasks.each
second.next # snapshot B starts later and may include the appended task
```

## Implementation constraints

- Do not change task 01's record format, filename version, write ordering, retention, or crash guarantees.
- Keep task 01 marked done and do not rewrite its specification or result.
- Do not add public exact-ID lookup or context/argument readers. Those belong to task 03.
- Do not add `delete`, `retry`, or mutation wiring. Those belong to tasks 04 and 05.
- Do not inject or retain any mutation-capable object in a `DeadTask`.
- Do not build the former complete ID/offset snapshot index.
- Do not store or cache traversal state on the memoized `DeadTasks` collection.
- Do not call `Rage::Deferred.__backend` while creating or returning the memoized collection, validating `batch_size`, or returning a valid unadvanced no-block Enumerator.
- Do not construct a backend inside `DeadTasks`. Resolve it through `Rage::Deferred.__backend` when an operation actually needs storage so the queue and collection always share one backend instance.
- Do not replace standard Ruby Enumerator behavior with a shared-cursor, lookahead, generation, or mixed-consumer protocol.
- Do not defer `batch_size` validation until first advancement or touch the backend while validating it.
- Do not mark an ID as seen until one of its records has passed frame, top-level decode, schema, and ID-consistency validation.
- Do not read or frame-parse beyond the captured complete-record boundary.
- Do not hold a filesystem lock while decoding a record or calling a user block.
- Do not constantize application classes.
- Do not deserialize or expose raw opaque context bytes.
- Do not describe `batch_size` as a bound on `seen_ids`, caller-retained entries, or the size of one stored physical record.
- Do not add a public `close` method to the returned Enumerator or its derivatives. Do not claim that abandoning a partial external or lazy traversal releases resources deterministically.
- Garbage collection or a carefully implemented finalizer may provide best-effort cleanup, but tests and public guarantees must not depend on its timing. Do not add an idle timeout.
- Do not scan extra records or emit logging or terminal output merely to report recoverable corruption.
- Do not rescue and translate `Rage::Deferred::DeadTasksLockTimeout` or filesystem `SystemCallError` failures into a generic public traversal error.

## Acceptance criteria

- [ ] `Rage::Deferred.dead_tasks` is memoized. Repeated calls return the same shared public collection object without calling `Rage::Deferred.__backend`.
- [ ] The collection retains a lazy resolver or equivalent collaborator. Storage operations resolve through `Rage::Deferred.__backend`, so the collection and queue use one identical memoized backend object regardless of which one initializes it first.
- [ ] The shared collection retains no traversal state. Every no-block `each` call returns a distinct normal Ruby Enumerator. Each block traversal and top-level Enumerable operation owns a private traversal execution.
- [ ] Obtaining the collection or a valid unadvanced Enumerator does not resolve the backend or open, create, scan, or decode the store.
- [ ] `DeadTasks#each` validates `batch_size` immediately. An invalid value raises `ArgumentError` before a no-block Enumerator is returned or block traversal begins. Validation does not resolve the backend or access storage.
- [ ] The no-block return value is a normal lazy Ruby Enumerator. It has standard Ruby 3.3 behavior and no Rage-specific `close`, shared-cursor, lookahead, generation, or mixed-consumer lifecycle.
- [ ] Separate top-level Enumerable operations and separate collection `each` calls use separate traversal executions and snapshots. Mixed internal and external consumption of one returned Enumerator follows standard Ruby behavior without Rage synchronization.
- [ ] Overlapping collection Enumerators, nested traversal, and same-process Fiber-interleaved traversal remain independent. Exhaustion, `break`, or an exception in one collection traversal does not affect another.
- [ ] Snapshot boundaries are established independently on first advancement, so a later-starting Enumerator may include an append that an already-started Enumerator excludes.
- [ ] Under the permanent lock, Disk setup captures an open descriptor and the last complete-record boundary. It does not scan or decode all earlier complete records. It releases the lock before validation or yielding.
- [ ] Traversal reads complete records backwards and lazily, then yields them newest-first. It retains an incremental `seen_ids` set instead of a full offset index and holds at most one decoded payload batch.
- [ ] For duplicate IDs, the newest fully valid record is yielded. A newer malformed, CRC-invalid, Marshal-unreadable, non-Hash, schema-invalid, or inner/outer-ID-mismatched record does not hide an older fully valid record.
- [ ] Missing or wrongly typed required fields and an incomplete tail are silently skipped without payload disclosure or eager inspection of untouched older records. On Disk, the outer framed ID is consistently used as the public and logical ID.
- [ ] A normally continuing snapshot is unchanged by append, task 01 torn-tail repair plus append, or rename-based compaction performed through backend or storage hooks between yields.
- [ ] Stopping before another batch prevents parsing or decoding records belonging only to untouched older batches.
- [ ] Enumeration closes snapshot resources on exhaustion, block `break`, non-local exit, exception, and normally unwound terminal Enumerable early exit. It never holds the dead-task lock while decoding or running user code.
- [ ] A partially consumed external or lazy traversal may retain its descriptor and an old unlinked inode until that execution exhausts, unwinds, or is garbage-collected. Rage offers no deterministic public cancellation API in this task.
- [ ] Snapshot-acquisition lock timeout and open/read/seek/cleanup errors propagate unchanged from the lazy operation that encounters them. Cleanup runs through `ensure` and does not mask an existing exception. When cleanup is the only failure, its original storage exception propagates.
- [ ] Summary entries are passive, read-only values and expose the documented typed metadata.
- [ ] Summary entries do not resolve task classes, deserialize contexts, or retain a collection, backend, backend resolver, operation delegate, or other mutation dependency.
- [ ] Nil enumeration is empty and creates no persistence or background activity.
- [ ] Public APIs and private helpers have the required YARD documentation.

## Verification

- Add focused specs for the accessor, collection, passive summary entries, Disk snapshot primitive, reverse-line reader, and Nil traversal behavior.
- Start with `Rage::Deferred`'s `@__backend` unset. Cover lazy collection creation, memoized wrapper identity, and lazy no-block Enumerator creation. Verify that collection access and each valid unadvanced Enumerator leave `@__backend` unset, do not construct Disk, and do not open or create Disk files. Every no-block `each` call returns a distinct normal Ruby Enumerator, while block-based traversal executions are independent.
- Cover both initialization orders. If the queue resolves `Rage::Deferred.__backend` first, the collection traversal must use that exact object. If collection traversal resolves it first, the queue must later use that exact object. Verify that only one configured backend instance is constructed in either order.
- Verify that every invalid `batch_size` raises `ArgumentError` immediately from `each`. The call must return no Enumerator, leave `@__backend` unset, invoke no backend resolver or storage method, and open no descriptor.
- Cover newest-first order and batch boundaries. Instrument decoding to prove that stopping before a later batch does not decode records that belong only to that older batch.
- Cover multiple physical records for one ID. The newest valid record wins. Verify fallback to the next older fully valid record when a newer record has a malformed frame, invalid CRC, unreadable top-level Marshal payload, decoded non-Hash, missing field, wrong field type, or inner/outer ID mismatch.
- Verify that the opaque context is type-checked as a String but not deserialized. Cover invalid backtrace contents and all summary-field types.
- Create earlier complete records and an incomplete tail. Start the snapshot, then call task 01's real add path so it repairs the tail and appends. Verify that the traversal yields only the original complete logical records, in unchanged order.
- Through backend or storage hooks, append and run rename-based compaction between yields. Verify that remaining snapshot entries are not skipped, duplicated, reordered, or replaced.
- Use a representative many-batch fixture. Verify incremental `seen_ids` growth, the separate decoded-payload bound, and the absence of an all-offset or all-payload cache.
- Verify that Disk holds the permanent lock only while opening the file and locating the boundary. It must release the lock before record validation and the first yield. It must preserve task 01's lock-timeout behavior for competing operations.
- Verify automatic descriptor cleanup after full exhaustion, block `break`, another non-local exit, and an exception.
- Verify cleanup when terminal Enumerable operations, including `find`, `first`, and `take`, stop early and unwind `DeadTasks#each`.
- Verify that a fully exhausted no-block Enumerator runs normal `ensure` cleanup. Verify that the returned Enumerator has no Rage-specific public `close` API.
- Exercise standard Ruby mixed internal and external Enumerator behavior. Do not assert a shared cursor, snapshot, descriptor, lookahead, or exhaustion state across those consumption modes.
- Verify that separate top-level Enumerable methods and separate collection `each` calls create separate traversal executions. Start one execution, append a record, and then start another; their independently timed snapshots may differ as documented.
- For corrupt records and an incomplete tail, verify silent skipping and that early termination does not inspect older untouched records.
- Cover two overlapping external Enumerators and nested `each`, including interleaved advancement from same-process Fibers. Verify that exhaustion, `break`, and an exception in one collection traversal do not affect the other.
- Demonstrate the documented partial external/lazy limitation without depending on collection timing: a suspended execution may still own its descriptor and keep an unlinked inode alive. Do not force garbage collection or test finalizer timing. Exhaust or otherwise normally unwind every test traversal before teardown.
- Inject `Rage::Deferred::DeadTasksLockTimeout` during first advancement. Also inject representative `SystemCallError` failures during open, reverse read/seek, and descriptor cleanup.
- Verify that each error propagates unchanged from the operation that encounters it and that acquired resources are released. Cleanup must not mask an active traversal or user-block exception. A storage cleanup exception must propagate when it is the only failure.
- Cover missing task classes, immutable/defensive summary values, absence of task-02 mutation methods or dependencies, silent skipped-record handling, and empty Nil enumeration.
- Run `bundle exec rspec spec/deferred/deferred_spec.rb spec/deferred/backends/disk_spec.rb` plus the new focused listing specs.
- Run the broader `bundle exec rspec spec/deferred` and RuboCop for changed Ruby/spec files.

## Result

- Implementation PR:
