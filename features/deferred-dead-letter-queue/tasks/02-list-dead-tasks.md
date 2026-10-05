---
status: ready-for-development
---

# List dead tasks

## Goal

Expose the durable dead-task store as an oldest-first Enumerable. The collection yields passive, read-only summaries and decodes each selected record only when it is about to be yielded.

## Context

Task 01 added the Disk/Nil backend facade and durable dead-task storage. Each disk record contains summary failure metadata and an opaque marshaled execution context.

The disk file can contain more than one physical record with the same authoritative outer task ID. The public collection treats those physical records as one **logical dead task** and selects the newest frame-valid record. Because Rage exclusively controls marshaling and writing this versioned store, a newer frame-valid record hides every older copy even if its selected payload later cannot be decoded or used as a dead-task record.

This is the first public-API task and depends only on task 01.

Task 02 owns:

- REQ-05 through REQ-07;
- only these listing fields from REQ-09: `id`, `task_class`, `attempts`, `enqueued_at`, and `failed_at`;
- only the enumeration parts of REQ-17: frame validation, silent physical-corruption handling, newest-frame-valid duplicate selection, and selected-winner failure behavior;
- the listing-related parts of REQ-19 and REQ-20;
- REQ-24 and REQ-25;
- only the enumeration and cleanup parts of REQ-26;
- REQ-27; and
- the public-collection parts of AC-02, AC-03, AC-07, and AC-11.

Task 02 does not own REQ-08 or REQ-10. It also does not own the exact-lookup parts of REQ-17 and REQ-26, or Task 03's detailed exception and context inspection.

Task 02 defines the stable traversal behavior that later entry actions rely on. It does not add `delete`, `retry`, or their private mutation wiring.

Task 01 introduced eager private `list_dead_tasks` and `find_dead_task` backend primitives. Task 02's argument-free `each_dead_task` traversal supersedes the list primitive, while the existing forward eager lookup does not satisfy task 03's exact-lookup contract. This task removes both obsolete Disk and Nil facade methods and their `Disk::DeadTasksStorage#list` and `#find` implementations. It retains task 01's `remove_dead_tasks` primitive for later mutation tasks; task 03 introduces a correct private exact-lookup capability from scratch.

Task 03 adds exact-ID inspection through `find_by_id` and lazy context decoding. It keeps the inherited Enumerable `find`. Task 04 adds deletion and the first private entry-action wiring. Task 05 extends that wiring for retry.

Follow [ADR-001](../adr/001-public-dead-task-object-model.md).

### Terms used below

- A **stable snapshot** is the sequence of complete physical records ending at a fixed byte boundary captured when traversal starts. Later changes to the live path do not change that sequence.
- A **logical dead task** is the newest frame-valid record for one authoritative outer task ID within that snapshot.
- A **payload** is the top-level stored record data that Rage decodes and wraps as a `DeadTask`. Its opaque execution context remains serialized in this task.
- A **read descriptor** is the open file handle used to read the snapshot.
- A **traversal execution** is one run of the collection's `each` body. It owns its snapshot descriptor and boundary, forward scan cursor, winner index, and transient current decoded record.
- A **winner index** maps each authoritative outer task ID to the serialized-payload location of its newest frame-valid physical record in the snapshot. It preserves insertion order by deleting and reinserting a repeated frame-valid authoritative outer ID at its newest physical position, so iterating the completed index yields the collection's oldest-first order without retaining every decoded payload.
- The **snapshot boundary** is the byte position immediately after the last complete newline-terminated record that exists when traversal starts. An unfinished tail after that position is not part of the snapshot.
- **Rename-based compaction** is task 01's removal process: write surviving records to a temporary file, then rename that file over the live data path.

## Requirements

### Public collection and entries

- Add the memoized `Rage::Deferred.dead_tasks` accessor.
- The accessor returns a `Rage::Deferred::DeadTasks` collection initialized with the shared memoized backend obtained through `Rage::Deferred.__backend`. Resolving that backend while creating the collection is allowed.
- Repeated accessor calls return the same collection object. The wrapper may retain:
  - the shared backend object;
  - immutable collaborators or configuration; and
  - the collection-level mutation behavior added by later tasks.
- The collection must not construct a backend separately. The accessor passes the object returned by `Rage::Deferred.__backend`, which guarantees that the collection and queue use the identical backend object whether the queue or collection initializes it first.
- Obtaining the collection may initialize the backend, including its normal initialization side effects. It must not open a DLQ snapshot descriptor or scan, decode, or preload DLQ records.
- `DeadTasks` includes `Enumerable`.
- The shared collection wrapper must not retain state that belongs to one traversal. In particular, it must not retain a snapshot descriptor or boundary, scan cursor, winner index, decoded record, current Enumerator, or traversal registry.
- Define `Rage::Deferred::DeadTask` as a passive, read-only summary value. At this stage, an entry contains only stored record data.
- A `DeadTask` must not retain the `DeadTasks` collection, the raw backend, an operation delegate, or any other mutation dependency.
- Entry construction and raw backend records remain private.

### Enumeration and snapshot boundary

- Block-based `each` is the primary and recommended traversal form. It accepts no arguments.
- Calling `each` without a block returns a new normal Ruby `Enumerator`, as an `enum_for`-style implementation would. It has standard Ruby 3.3 behavior. Returning this unadvanced Enumerator must not open a DLQ snapshot descriptor or scan or decode DLQ records. This task does not add `close` or another Rage-specific lifecycle method to that Enumerator.
- Each block traversal and each ordinary top-level Enumerable operation invokes `DeadTasks#each` and owns one traversal execution. Separate calls such as `find`, `map`, `filter_map`, `count`, or `each` use separate cursors and snapshots. Repeated calls on the memoized collection do not share traversal state.
- Returning a valid no-block Enumerator must not establish or decode a snapshot. Its first traversal execution establishes the snapshot.
- Traversal yields logical dead tasks oldest-first according to the physical positions of their selected winning records. Therefore `dead_tasks.first(20)` returns the oldest 20 logical dead tasks in the captured snapshot. A duplicated ID whose winning record is newer than other tasks is ordered at that newer winning record's position, not at the position of an older superseded copy.
- When a valid traversal execution starts, use the wrapper's retained backend. For Disk, acquire task 01's permanent dead-task lock, open the current live data file for the snapshot, and locate the snapshot boundary. Release the lock after capturing the descriptor and boundary. Setup must not scan or validate every complete record.
- If the file ends with an incomplete tail that has no terminating newline, silently exclude the entire tail. Setup may read backwards in fixed-size chunks to find the preceding newline while holding the lock. It must not parse or decode earlier complete records.
- Never read a snapshot record past the captured boundary. An append after setup, including an append that first repairs an incomplete tail through task 01's real `add` path, is not part of that traversal.
- Keep the snapshot descriptor open. If compaction renames a replacement over the live path, the descriptor still reads the original inode up to the fixed boundary.
- Appending, repairing a torn tail, or rename-compacting the live store between yields must not skip, duplicate, reorder, or replace remaining logical entries when traversal continues normally.
- Every traversal execution created by a separate collection operation owns its own snapshot descriptor, boundary, cursor, winner index, and transient current decoded record. This includes overlapping external Enumerators, nested block traversal, and same-process Fiber-interleaved traversal. Exhausting, breaking, or raising in one collection traversal must not alter another.
- Each traversal establishes its snapshot independently. For example, first advance one Enumerator, append a new record, and then first advance a second Enumerator. The first snapshot excludes the new record, while the second may include it.
- This task guarantees reentrant use of the memoized wrapper and isolation between traversals. It does not introduce a broader thread-safety promise beyond Rage's existing Iodine/Fiber execution model.
- Tests must start and pause an enumeration, change the live store through task 01's internal backend methods, and then resume enumeration. They must not depend on the public `delete` or `retry` methods added by later tasks.

### Standard Enumerator behavior

- A normally consumed lazy pipeline uses its underlying traversal execution. One ordinary top-level Enumerable method call likewise uses one traversal execution.
- Ruby may keep internal and external iteration state separately on one Enumerator. If a caller mixes `next` or `peek` with block `enumerator.each`, Enumerable methods, or derived iterators, standard Ruby behavior applies. Rage does not synchronize those cursors or promise that they use one snapshot or descriptor.
- Standard `next_values`, `peek_values`, `rewind`, `with_index`, `lazy`, and other Enumerator behavior remains unchanged. This task adds no shared lookahead, generation tracking, or cross-surface exhaustion rules.
- Callers that need independent complete traversals should start separate collection operations. Mixed manual consumption of one returned Enumerator is allowed but receives no Rage-specific lifecycle guarantee.
- Separate collection traversals are reentrant under Rage's single-threaded Fiber model. This task adds no broader thread-safety guarantee.

### Forward selection scan, oldest-first delivery, and memory

- Before yielding its first entry, read every complete physical record sequentially forwards from byte zero through the snapshot boundary. Frame-validate each record and build a per-traversal insertion-ordered winner index without calling `Marshal.load` or otherwise inspecting a top-level serialized payload.
- When a frame-valid authoritative outer ID first appears, insert its serialized-payload location at that physical position. When the same frame-valid ID appears again, delete its prior index entry and reinsert it with the newer serialized-payload location. Deletion followed by reinsertion moves that logical task to its newest winning record's physical position. A newer frame-invalid duplicate does not change the existing winner. A newer frame-valid duplicate always replaces it; a later payload decoding or shape problem must not fall back to an older copy.
- After the forward selection scan completes, visit the selected winning locations in index insertion order, which is oldest-first by winning physical position. Seek, re-read, decode, wrap, and yield one winner at a time. Do not decode another selected winner before control returns from the current yield.
- Do not cache an eager array of every decoded entry or payload. The complete winner index is required because a later physical record can supersede an earlier frame-valid record and move that logical task to a newer position; traversal therefore cannot yield during the selection scan.
- Traversal memory includes one winner-index entry per distinct frame-valid authoritative outer ID, bounded physical-line scan storage, and at most the one decoded record being prepared for or suspended at the current yield. Caller-retained yielded entries remain outside this internal bound. Total traversal memory is O(distinct frame-valid outer IDs) and is not bounded independently of store size.
- `first`, `take`, `find`, an early block exit, or partial external iteration cannot avoid the complete forward selection scan, because a later physical record can supersede an apparently older task with the same ID. After selection, early termination prevents every later winner from being re-read, decoded, wrapped, or yielded. `first(n)` and `take(n)` decode only the records they actually consume.
- A caller may still deliberately materialize all public entries with a standard Enumerable method such as `to_a`; caller-retained memory is outside the iterator guarantee.

### Frame validity and duplicate selection

A physical Disk record is frame-valid for Task 02 when and only when all of these checks pass:

1. The record is complete and ends with a newline inside the snapshot boundary.
2. Its frame structure, separators, checksum width, and operation are valid.
3. Its CRC matches.
4. Its authoritative outer framed task ID is non-empty.

The forward winner scan performs only those checks. It must not call `Marshal.load` or require the selected payload to decode to a Hash, contain required keys or expected value types, contain only String backtrace entries, store context as a String, or have an inner `record[:id]` equal to the authoritative outer ID. The outer framed ID remains authoritative for deduplication, the public `id`, and later exact-ID actions.

Silently skip a malformed frame or invalid CRC and continue reading. Snapshot setup likewise silently excludes an incomplete tail. These records and fragments represent recoverable physical corruption, not operational failures.

After the complete winner index is built, decode selected winners one at a time immediately before yielding them. If a selected top-level Marshal payload fails to decode, propagate the original decoding exception unchanged and do not fall back to an older duplicate. If decoding succeeds but the payload has a shape that cannot produce the documented Rage-written public record, any resulting natural error likewise does not trigger storage-level fallback. Semantically invalid but correctly checksummed payloads, schema drift without a storage-version bump, manually constructed records, and inner/outer-ID mismatches are outside the supported store contract.

### Recoverable corruption and operational failures

- Malformed frames, invalid CRCs, and an excluded incomplete tail are skipped without logging or terminal output. Enumeration inspects every complete physical record during its required forward selection scan; it performs no additional scan merely to diagnose corruption. A selected winner's payload failure is not recoverable corruption and propagates as specified above.
- `Rage::Deferred::DeadTasksLockTimeout` during snapshot acquisition is an operational failure, not a skipped item. Filesystem errors from open, read, seek, or descriptor cleanup are also operational failures. Propagate these errors unchanged from the lazy advancement that performs the operation. Do not translate them into a generic listing error. For block traversal, they propagate from the `each` call.
- Release every acquired lock and each normally unwound traversal descriptor through `ensure`. If traversal, application code, or storage work is already raising an exception, cleanup must not replace it. If storage cleanup is the only failing operation, propagate that cleanup exception instead of suppressing it.

### Locking and cleanup

- Hold the dead-task lock only while opening the live file and capturing the snapshot boundary. Release the lock before frame validation, selected-record `Marshal.load`, entry wrapping, or yielding.
- Do not hold the lock while calling application code. A caller's block can be slow or can start another store operation that needs the same lock.
- Close the read descriptor automatically when traversal is exhausted. Also close it when block or internal traversal unwinds through `break`, another non-local exit, or an exception. This includes ordinary terminal Enumerable methods and their early-exit paths when they unwind `each`.
- After compaction, a long-lived descriptor can keep the old, unlinked snapshot file on disk. The file's space is released when the descriptor closes.

For no-block and lazy iteration:

- Full exhaustion runs the traversal's normal `ensure` cleanup.
- A partially consumed external or derived/lazy Enumerator can keep its descriptor open if the traversal execution remains suspended. After rename-based compaction, that descriptor can also keep the old unlinked inode on disk.
- Because the selection scan finishes before the first yield, that suspended execution also retains its complete winner index.
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
- Treat argument-free `each_dead_task` as the private backend listing contract. Remove `list_dead_tasks` from the Disk and Nil backend facades, remove `Disk::DeadTasksStorage#list`, and remove their implementation documentation and eager-list coverage. Update remaining storage specs that used `list_dead_tasks` only to inspect state so they use `each_dead_task` or direct storage evidence as appropriate.
- Remove the current `find_dead_task` methods from the Disk and Nil backend facades and `Disk::DeadTasksStorage#find`, together with their implementation documentation and tests. Task 03 introduces a conforming private exact-lookup capability when it adds public `find_by_id`; this task must not preserve the current eager forward lookup as an interim backend contract.
- Keep `remove_dead_tasks` on the Disk and Nil backend facades. Its behavior and the underlying `remove` storage primitive remain unchanged for tasks 04 and 05.
- Keep storage and snapshot helpers private. Use YARD `@private` where Ruby visibility cannot express that boundary.
- Document all of the following:
  - the accessor, collection, and passive summary readers;
  - direct use of the shared backend identity by the collection and queue, and traversal-lazy DLQ snapshot work;
  - the argument-free public `each` API and one-at-a-time selected-record decoding;
  - stable snapshots, the complete sequential forward selection scan, delete-and-reinsert winner indexing, and oldest-first delivery;
  - silent skipping of malformed frames, invalid CRCs, and an incomplete tail;
  - newest-frame-valid selection without payload loading, and unchanged selected-winner decoding failures without older fallback;
  - unchanged propagation of operational errors;
  - automatic cleanup for block traversal and normally unwound terminal Enumerable operations;
  - standard Ruby Enumerator behavior and the absence of a custom `close` API;
  - nondeterministic descriptor cleanup for abandoned partial external or lazy traversal;
  - the lifetime of an old, unlinked inode; and
  - the sensitive-data risk.
- Reverse file reading, frame-only winner-index construction, selected-record re-reading, and selected top-level Marshal decoding are synchronous work in the calling Fiber. Document this behavior. Snapshot setup does not scan the complete store under lock, but every non-empty traversal scans the complete snapshot before its first yield and a large traversal can delay other work in that worker.
- Document that Marshal payloads are trusted local application data and that physical frame validation does not make untrusted Marshal data safe to decode.
- Preserve Ruby 3.3.0 compatibility and the existing `config.deferred.backend` contract.
- Do not add polling or work to successful deferred-task execution.

## Design

### How traversal works

1. `Rage::Deferred.dead_tasks` resolves the shared memoized backend through `Rage::Deferred.__backend`, passes that object directly to a new collection when needed, and then returns the memoized collection. Backend initialization may occur here, but no DLQ snapshot is opened and no DLQ record is scanned, decoded, or preloaded.
2. A no-block `DeadTasks#each` call returns a normal lazy Enumerator without starting DLQ snapshot work. The memoized collection does not register executions or copy their state onto itself. `each` accepts no arguments.
3. When a block, Enumerable operation, or external Enumerator actually starts traversal, the collection uses its retained backend. Disk then takes the permanent lock and opens the current live data file for the snapshot. If the queue already initialized the backend, the collection receives that exact object. If collection access initializes it first, the queue later receives that exact object.
4. Disk finds the last newline at or before the current end of file. The fixed snapshot boundary is the position immediately after that newline. If the file already ends in a newline, the boundary is its current size. If the file contains no newline, the boundary is zero. An incomplete tail is silently excluded.
5. Disk releases the lock but keeps the read descriptor. All later snapshot reads are limited to bytes before the captured boundary.
6. Starting at byte zero, Disk walks every complete record sequentially forwards through the fixed boundary. For each record, it validates only newline completeness inside the boundary, frame structure and operation, CRC, and a non-empty authoritative outer ID. It performs no top-level Marshal load, schema/type check, or inner/outer-ID comparison while building the index. A frame-invalid record is silently skipped and does not change the current winner for its ID.
7. A frame-valid authoritative outer ID is inserted with its serialized-payload location. A repeated frame-valid ID deletes the earlier index entry and reinserts the ID with the newer location, moving the logical task to its newest winning physical position. The scan must reach the fixed boundary before any entry is yielded.
8. Disk visits winning locations in insertion order. For each location, it seeks, reads, decodes, wraps, and yields that winner before proceeding to the next location. A selected payload decoding error propagates unchanged and does not cause fallback. While suspended at a yield, Disk has not decoded the next winner.
9. When traversal exhausts or block/internal control flow unwinds, `ensure` closes that execution's descriptor and discards its winner index without masking an exception already in flight.

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
iterator = dead_tasks.each
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
- Keep task 01 marked done and do not rewrite its historical specification or result. Task 02 supersedes only its eager private listing and exact-lookup primitives.
- Do not add public exact-ID lookup or context/argument readers. Those belong to task 03.
- Do not add `delete`, `retry`, or mutation wiring. Those belong to tasks 04 and 05.
- Do not inject or retain any mutation-capable object in a `DeadTask`.
- Do not omit the complete winner index required to reconcile oldest-first output with newest-frame-valid duplicate selection. Keep only the selected serialized-payload location metadata needed to re-read winners; do not retain every physical record or decoded payload.
- Do not store or cache traversal state on the memoized `DeadTasks` collection.
- Do not start DLQ snapshot work while creating or returning the memoized collection or returning a valid unadvanced no-block Enumerator. In particular, do not open a snapshot descriptor or scan, decode, or preload DLQ records before traversal starts.
- Do not construct a backend inside `DeadTasks`. Pass it the shared memoized backend returned by `Rage::Deferred.__backend` so the queue and collection always use one backend instance.
- Do not replace standard Ruby Enumerator behavior with a shared-cursor, lookahead, generation, or mixed-consumer protocol.
- For each frame-valid authoritative outer ID, delete its existing winner-index entry, if any, and reinsert the ID with the current serialized-payload location. Do not load or inspect the serialized payload while constructing the winner index.
- Do not read or frame-parse beyond the captured complete-record boundary.
- Do not hold a filesystem lock while decoding a record or calling a user block.
- Do not constantize application classes.
- Do not deserialize or expose raw opaque context bytes.
- Do not retain `RECORD_BATCH_SIZE`, a decoded-record batch, or any batch-size parameter or configuration. Decode and yield one selected winner at a time; this does not bound the winner index, caller-retained entries, or the size of one stored physical record.
- Do not add a public `close` method to the returned Enumerator or its derivatives. Do not claim that abandoning a partial external or lazy traversal releases resources deterministically.
- Garbage collection or a carefully implemented finalizer may provide best-effort cleanup, but tests and public guarantees must not depend on its timing. Do not add an idle timeout.
- Do not scan extra records or emit logging or terminal output merely to report malformed frames, invalid CRCs, or incomplete tails.
- Do not rescue a selected winner's top-level Marshal decoding failure, translate it, or continue to an older duplicate.
- Do not rescue and translate `Rage::Deferred::DeadTasksLockTimeout` or filesystem `SystemCallError` failures into a generic public traversal error.

## Acceptance criteria

- [ ] `Rage::Deferred.dead_tasks` is memoized. Repeated calls return the same shared public collection object, initialized with the shared memoized backend obtained through `Rage::Deferred.__backend`.
- [ ] The collection retains that backend object directly. The collection and queue use one identical memoized backend object regardless of which one initializes it first.
- [ ] The shared collection retains no traversal state. Every no-block `each` call returns a distinct normal Ruby Enumerator. Each block traversal and top-level Enumerable operation owns a private traversal execution.
- [ ] Obtaining the collection may resolve and initialize the backend. Obtaining the collection or a valid unadvanced Enumerator does not open a DLQ snapshot descriptor or scan, decode, or preload DLQ records.
- [ ] `DeadTasks#each` is argument-free. Disk retains no `RECORD_BATCH_SIZE` or decoded-record batch and decodes selected winners one at a time.
- [ ] The no-block return value is a normal lazy Ruby Enumerator. It has standard Ruby 3.3 behavior and no Rage-specific `close`, shared-cursor, lookahead, generation, or mixed-consumer lifecycle.
- [ ] Separate top-level Enumerable operations and separate collection `each` calls use separate traversal executions and snapshots. Mixed internal and external consumption of one returned Enumerator follows standard Ruby behavior without Rage synchronization.
- [ ] Overlapping collection Enumerators, nested traversal, and same-process Fiber-interleaved traversal remain independent. Exhaustion, `break`, or an exception in one collection traversal does not affect another.
- [ ] Snapshot boundaries are established independently on first advancement, so a later-starting Enumerator may include an append that an already-started Enumerator excludes.
- [ ] Under the permanent lock, Disk setup captures an open descriptor and the last complete-record boundary. It does not scan or decode all earlier complete records. It releases the lock before validation or yielding.
- [ ] Before its first yield, traversal scans all complete snapshot records sequentially forwards, selects the newest frame-valid record for every authoritative outer ID into a complete insertion-ordered winner index without calling `Marshal.load`, and then yields the winners oldest-first by selected physical position. A repeated frame-valid ID is deleted and reinserted at its newest position. In addition to that index and bounded physical-line scan storage, traversal holds at most the one decoded record being prepared for or suspended at the current yield.
- [ ] For duplicate outer IDs, the newest frame-valid record wins. A newer malformed frame or invalid CRC does not hide an older frame-valid record, but a newer frame-valid record with an unreadable, non-Hash, schema-incompatible, wrongly typed, invalid-backtrace/context, or inner/outer-ID-mismatched payload does hide every older duplicate.
- [ ] Malformed frames, invalid CRCs, and an incomplete tail are silently skipped without payload disclosure or diagnostic output. On Disk, the outer framed ID is consistently used as the public and logical ID.
- [ ] Each selected winner is decoded immediately before its corresponding yield. A selected top-level Marshal decoding failure propagates unchanged and does not fall back to an older duplicate.
- [ ] A normally continuing snapshot is unchanged by append, task 01 torn-tail repair plus append, or rename-based compaction performed through backend or storage hooks between yields.
- [ ] `first(20)` returns the oldest 20 logical tasks by winning-record position. It performs the complete selection scan before yielding and decodes exactly those 20 winners, not any later winner. `first`, `take`, and other early exits likewise perform no selected-record predecode beyond records actually yielded.
- [ ] Enumeration closes snapshot resources on exhaustion, block `break`, non-local exit, exception, and normally unwound terminal Enumerable early exit. It never holds the dead-task lock while decoding or running user code.
- [ ] A partially consumed external or lazy traversal may retain its descriptor and an old unlinked inode until that execution exhausts, unwinds, or is garbage-collected. Rage offers no deterministic public cancellation API in this task.
- [ ] Snapshot-acquisition lock timeout and open/read/seek/cleanup errors propagate unchanged from the lazy operation that encounters them. Cleanup runs through `ensure` and does not mask an existing exception. When cleanup is the only failure, its original storage exception propagates.
- [ ] Summary entries are passive, read-only values and expose the documented typed metadata.
- [ ] Summary entries do not resolve task classes, deserialize contexts, or retain a collection, backend, operation delegate, or other mutation dependency.
- [ ] Nil enumeration is empty and creates no persistence or background activity.
- [ ] Argument-free `each_dead_task` is the only private dead-task backend read primitive after this task. Disk and Nil no longer define `list_dead_tasks` or `find_dead_task`, `Disk::DeadTasksStorage` no longer defines `list` or `find`, and implementation documentation no longer advertises those obsolete eager methods. The existing `remove_dead_tasks` facade method and its storage behavior remain available and unchanged.
- [ ] Public APIs and private helpers have the required YARD documentation.

## Verification

- Add focused specs for the accessor, collection, passive summary entries, Disk snapshot primitive, sequential forward enumeration scan, and Nil traversal behavior.
- Start with `Rage::Deferred`'s `@__backend` unset. Cover collection creation, memoized wrapper identity, and lazy no-block Enumerator creation. Verify that collection access resolves the configured backend once and passes that exact object directly to `DeadTasks`. Collection access and each valid unadvanced Enumerator must not invoke DLQ traversal, open a snapshot descriptor, or scan, decode, or preload DLQ records. Every no-block `each` call returns a distinct normal Ruby Enumerator, while block-based traversal executions are independent.
- Cover both initialization orders. If the queue resolves `Rage::Deferred.__backend` first, the collection must receive that exact object. If collection access resolves it first, the queue must later use that exact object. Verify that only one configured backend instance is constructed in either order.
- Verify that `each` accepts no arguments and that its unadvanced no-block form invokes no DLQ traversal method and opens no snapshot descriptor.
- Cover oldest-first order and one-at-a-time selected-record decoding. Prove specifically that `first(20)` returns and decodes only the oldest 20 logical tasks by winning-record position.
- Cover multiple physical records for one authoritative outer ID. The newest frame-valid record wins. Verify fallback to the next older frame-valid record only when a newer record has a malformed frame or invalid CRC. Verify that unreadable top-level Marshal payloads, decoded non-Hash values, missing fields, wrong field types, invalid backtrace/context values, and inner/outer-ID mismatches are not checked during selection and do not permit fallback past a newer frame-valid record.
- Do not retain tests that require storage selection to validate required keys/types, String-only backtraces, String context, or inner/outer-ID equality.
- Create earlier complete records and an incomplete tail. Start the snapshot, then call task 01's real add path so it repairs the tail and appends. Verify that the traversal yields only the original complete logical records, in unchanged order.
- Through backend or storage hooks, append and run rename-based compaction between yields. Verify that remaining snapshot entries are not skipped, duplicated, reordered, or replaced.
- Use a representative many-record fixture. Verify the complete O(distinct frame-valid outer IDs) winner index, bounded physical-line scan storage, at most one traversal-owned decoded record, and the absence of an all-payload cache.
- Instrument traversal to prove that even `first(1)` completes the forward selection scan before its first yield and that the complete winner-index scan performs no `Marshal.load`. Verify `first(1)` performs exactly one selected-record decode, while `first(20)` and `take(20)` perform exactly 20 when enough winners exist, with no later winner read or decode.
- Instrument one-at-a-time delivery to prove that each selected winner is decoded immediately before its corresponding yield. Verify that its top-level decoding failure propagates unchanged and does not inspect an older duplicate.
- Verify that Disk holds the permanent lock only while opening the file and locating the boundary. It must release the lock before record validation and the first yield. It must preserve task 01's lock-timeout behavior for competing operations.
- Verify automatic descriptor cleanup after full exhaustion, block `break`, another non-local exit, and an exception.
- Verify cleanup when terminal Enumerable operations, including `find`, `first`, and `take`, stop early and unwind `DeadTasks#each`.
- Verify that a fully exhausted no-block Enumerator runs normal `ensure` cleanup. Verify that the returned Enumerator has no Rage-specific public `close` API.
- Exercise standard Ruby mixed internal and external Enumerator behavior. Do not assert a shared cursor, snapshot, descriptor, lookahead, or exhaustion state across those consumption modes.
- Verify that separate top-level Enumerable methods and separate collection `each` calls create separate traversal executions. Start one execution, append a record, and then start another; their independently timed snapshots may differ as documented.
- For malformed frames, invalid CRCs, and an incomplete tail, verify silent skipping without diagnostic output and confirm that early termination still completes the full selection scan required for duplicate resolution and oldest-first order.
- Cover two overlapping external Enumerators and nested `each`, including interleaved advancement from same-process Fibers. Verify that exhaustion, `break`, and an exception in one collection traversal do not affect the other.
- Demonstrate the documented partial external/lazy limitation without depending on collection timing: a suspended execution may still own its descriptor and keep an unlinked inode alive. Do not force garbage collection or test finalizer timing. Exhaust or otherwise normally unwind every test traversal before teardown.
- Inject `Rage::Deferred::DeadTasksLockTimeout` during first advancement. Also inject representative `SystemCallError` failures during open, forward read/seek, and descriptor cleanup.
- Verify that each error propagates unchanged from the operation that encounters it and that acquired resources are released. Cleanup must not mask an active traversal or user-block exception. A storage cleanup exception must propagate when it is the only failure.
- Cover missing task classes, immutable/defensive summary values, absence of task-02 mutation methods or dependencies, silent skipped-record handling, and empty Nil enumeration.
- Remove eager `list_dead_tasks`/`DeadTasksStorage#list` and current `find_dead_task`/`DeadTasksStorage#find` coverage. Replace state assertions that used the eager list facade only as a test helper with `each_dead_task` or direct storage evidence. Verify that Disk, Nil, and `Disk::DeadTasksStorage` no longer expose the removed read methods while `remove_dead_tasks` continues to work.
- Run focused Disk, Nil, collection, and `DeadTask` specs, including `bundle exec rspec spec/deferred/deferred_spec.rb spec/deferred/backends/disk_spec.rb` and the focused listing specs.
- Run the broader `bundle exec rspec spec/deferred` and `bundle exec rake` when dependencies permit.
- Run RuboCop for changed Ruby/spec files, YARD with `--fail-on-warning`, Ruby syntax checks for changed Ruby files, and `git diff --check`. Report unavailable checks and their exact dependency failures.

## Result

- Implementation PR:
