---
status: todo
---

# List dead tasks

## Goal

Expose the durable dead-task store as a newest-first Enumerable. It yields passive, read-only summaries and decodes record payloads in bounded batches instead of loading every entry at once. This task also applies the selected owner-only default and mode-preserving compaction policy to dead-task Disk files.

## Context

Task 01 added the Disk/Nil backend facade and durable dead-task storage. Each disk record contains summary failure metadata and an opaque marshaled execution context. Task 01 is complete and remains the historical record of the implementation that shipped; this task owns the additive file-permission hardening described below rather than rewriting Task 01.

The disk file can contain more than one physical record with the same task ID. The public collection treats those physical records as one **logical dead task** and yields the newest record that is fully valid. A newer damaged record must not hide an older usable copy.

This is the first public-API task and depends only on task 01. It owns REQ-05 through REQ-07; only the listing fields `id`, `task_class`, `attempts`, `enqueued_at`, and `failed_at` from REQ-09; only REQ-17's enumeration schema/corruption, skipped-item counting, and traversal-warning behavior; listing-related parts of REQ-19 and REQ-20; REQ-24 and REQ-25; REQ-26's enumeration/cleanup behavior; REQ-27; REQ-30 through REQ-33; and the public-collection parts of AC-02, AC-03, AC-07, AC-11, and AC-12. It does not own REQ-08, REQ-10, REQ-17's exact-lookup behavior, REQ-26's exact-lookup behavior, or Task 03's detailed exception/context inspection.

Task 02 defines the stable traversal behavior that later entry actions rely on. It does not add `delete`, `retry`, or their private mutation wiring. Task 03 adds exact-ID inspection through `find_by_id` and lazy context decoding while preserving inherited Enumerable `find`. Task 04 adds deletion and the first private entry-action wiring. Task 05 extends that wiring for retry. Follow [ADR-001](../adr/001-public-dead-task-object-model.md) and [ADR-003](../adr/003-dead-task-file-permissions.md).

### Terms used below

- A **stable snapshot** is the sequence of complete physical records ending at a fixed byte boundary captured when traversal starts. Later changes to the live path do not change that sequence.
- A **logical dead task** is the newest fully valid record for one task ID within that snapshot.
- A **payload** is the top-level stored record data that Rage decodes and wraps as a `DeadTask`. Its opaque execution context remains serialized in this task.
- A **read descriptor** is the open file handle used to read the snapshot.
- The **snapshot boundary** is the byte position immediately after the last complete newline-terminated record that exists when traversal starts. An unfinished tail after that position is not part of the snapshot.
- **Rename-based compaction** is task 01's removal process: write surviving records to a temporary file, then rename that file over the live data path.

## Requirements

### Public collection and entries

- Add the memoized `Rage::Deferred.dead_tasks` accessor.
- The accessor returns a `Rage::Deferred::DeadTasks` collection over `Rage::Deferred.__backend`. This must be the same backend instance used by the queue.
- Repeated accessor calls return the same collection object. The wrapper may retain that backend reference, immutable collaborators or configuration, and the collection-level mutation behavior added by later tasks.
- Obtaining the collection must not open, scan, or preload the store.
- `DeadTasks` includes `Enumerable`.
- The shared collection wrapper must retain no state belonging to an individual traversal: no snapshot descriptor or boundary, reverse-reader cursor, `seen_ids`, decoded batch, warning count, closed/rewound flag, current Enumerator, or traversal registry.
- Define `Rage::Deferred::DeadTask` as a passive, read-only summary value. At this stage, an entry contains only stored record data.
- A `DeadTask` must not retain the `DeadTasks` collection, the raw backend, an operation delegate, or any other mutation dependency.
- Entry construction and raw backend records remain private.

### Disk file permissions

- When Disk creates the live dead-task file, permanent lock file, or compaction temporary file, request mode `0600`. The normal process umask may remove permissions from this request, but no newly created file may become more permissive than `0600`.
- Do not change the mode of an existing live or lock file during backend initialization. This preserves administrator-selected modes and means a legacy `0644` file stays `0644` until an operator changes it.
- A compaction temporary file is internal scratch rather than an administrator-facing persistent file. Do not modify a stale pre-existing `.tmp` path during unlocked initialization. While holding the permanent dead-task lock and before reusing that path for compaction, open or create the temporary file without first truncating it, remove all group and other mode bits, and do not add owner permissions that are already absent. Only then may Disk truncate and write it.
- While holding the permanent lock, obtain the current live file's mode bits from the opened live file. Before a completed temporary file can replace it, apply those exact mode bits to the temporary file, then `fsync` the temporary file so both its contents and mode metadata are durable. Preserve task 01's remaining order: rename the temporary file over the live path, then `fsync` the parent directory.
- If hardening the temporary file or applying the live mode fails, propagate the filesystem error and do not rename the temporary file over the live path. Preserve task 01's existing uncertainty if the rename succeeds but the later directory `fsync` fails.
- Preserve mode bits only. Do not add ownership-changing behavior or require `chown` capability. A replacement file can therefore have the process and group ownership assigned when its temporary inode was created.
- Do not change the configured directory's mode. Do not add file-mode configuration. Do not change pending-task WAL permissions. These requirements apply only to dead-task Disk storage.
- Nil must continue to create no files. A future backend is responsible for defining and testing its own storage-permission policy.
- Treat one operating-system account for all Rage workers as the supported default. Cross-user operation depends on an administrator pre-provisioning a trusted, non-world-writable directory, compatible shared group/setgid ownership, suitable existing file modes such as `0660`, and compatible umasks. Removing owner read or write permissions may make Disk storage unusable.

### Enumeration and snapshot boundary

- `each(batch_size: 100)` called without a block returns an actual Ruby `Enumerator` instance. The directly returned root Enumerator additionally exposes public `close`. It may be implemented by subclassing or extending `Enumerator`, but its concrete class name and internal state are private.
- Every `each` call creates a private traversal session. Every no-block call returns a distinct root Enumerator; repeated calls on the memoized collection must not return the same Enumerator or share traversal state. Block-based calls also use independent sessions.
- `batch_size` must be a positive Integer. Validate it immediately when `DeadTasks#each` is called, before returning a no-block Enumerator or starting block traversal. Any other value raises `ArgumentError` without opening, scanning, or decoding the store.
- Returning a valid iterator must not establish or decode a snapshot. Its first request for an entry establishes the snapshot.
- Traversal yields logical dead tasks newest-first.
- To start a Disk traversal, acquire task 01's permanent dead-task lock, open the current live data file, and locate the snapshot boundary. Release the lock after capturing the descriptor and boundary. Do not scan or validate all complete records during setup.
- If the file ends in an incomplete, non-newline-terminated tail, exclude the entire tail and count it as one skipped physical fragment. Finding the preceding newline may read backwards in fixed-size chunks while the lock is held, but setup must not parse or decode earlier complete records.
- Never read a snapshot record past the captured boundary. An append after setup, including an append that first repairs an incomplete tail through task 01's real `add` path, is not part of that traversal.
- Keep the snapshot descriptor open. If compaction renames a replacement over the live path, the descriptor continues to read the original inode and fixed boundary.
- Appending, repairing a torn tail, or rename-compacting the live store between yields must not skip, duplicate, reorder, or replace remaining logical entries when traversal continues normally.
- Overlapping external Enumerators, nested block traversal, and same-process Fiber-interleaved traversal each own their snapshot descriptor, boundary, cursor, `seen_ids`, decoded batch, warnings, and lifecycle state. Closing, rewinding, exhausting, breaking, or raising in one traversal must not alter another traversal.
- Snapshot timing is independent. If one Enumerator first advances, a new record is then appended, and a second Enumerator first advances afterwards, the first snapshot excludes the new record while the second may include it.
- This task guarantees reentrant use of the memoized wrapper and isolation between traversals. It does not introduce a broader thread-safety promise beyond Rage's existing Iodine/Fiber execution model.
- Tests must start and pause an enumeration, change the live store through task 01's internal backend methods, and then resume enumeration. These tests must not depend on the public `delete` or `retry` methods added by later tasks.

### Lazy reverse traversal and memory

- Read complete physical records backwards from the snapshot boundary. Validate a record only when traversal reaches it.
- Maintain a per-traversal `seen_ids` set. Add an ID only after that physical record passes every validity check below. Once an ID is in the set, older physical records with that ID cannot replace it.
- A newer invalid duplicate does not add its ID to `seen_ids`. Traversal continues backwards so an older fully valid record with that ID can become the logical entry.
- Decode and wrap no more than `batch_size` valid payloads at one time. Do not build or cache an eager array of every entry, every record offset, or every payload.
- Aside from the reverse reader's bounded byte buffer and the bytes for the physical record currently being checked, traversal memory consists of the incremental `seen_ids` set and at most one decoded payload batch. A complete traversal can therefore retain O(distinct valid IDs encountered) metadata. It is not memory-bounded independently of store size.
- Stopping early must stop further reverse traversal. Records in older, not-yet-requested batches must not be parsed or decoded. Work already performed for the current batch remains bounded by `batch_size`.
- A caller may still deliberately materialize all public entries with a standard Enumerable method such as `to_a`; caller-retained memory is outside the iterator guarantee.

### Record validity and duplicate selection

A physical record is fully valid for Task 02 only when all of these checks pass:

1. The record is complete and ends with a newline inside the snapshot boundary.
2. Its frame has the expected operation, separators, and checksum width, and its CRC matches.
3. The outer framed task ID is a non-empty String. This outer ID is authoritative for deduplication and all later exact-ID actions.
4. Decoding the top-level dumped payload with `Marshal.load` succeeds and returns a Hash.
5. The Hash contains the symbol keys `:id`, `:task_class`, `:attempts`, `:enqueued_at`, `:failed_at`, `:exception_class`, `:exception_message`, `:backtrace`, and `:context`.
6. `:id`, `:task_class`, `:exception_class`, and `:exception_message` are Strings; `:attempts`, `:enqueued_at`, and `:failed_at` are Integers; `:backtrace` is an Array containing only Strings; and `:context` is a String containing the opaque stored bytes.
7. The inner `record[:id]` exactly equals the authoritative outer framed ID.

Extra Hash keys do not invalidate a record. Task 02 verifies that `:context` has the expected stored-byte type but never deserializes those bytes. Task 03 defines what happens when a valid top-level record contains an execution context that cannot itself be decoded.

When any required check fails, skip that physical record, increment this traversal's skipped-item count, and continue. The excluded incomplete tail counts as one skipped physical fragment when snapshot setup actually encounters it. Skipped items are recoverable corruption observations, not operational failures, and this task must not silently treat an invalid record as a valid duplicate merely to simplify counting.

### Skipped-item warnings and operational failures

- The skipped-item count belongs to one traversal. Increment it only for invalid physical records or the incomplete tail actually encountered by that traversal's lazy work. Do not parse untouched older records merely to compute a complete snapshot count.
- When a traversal with a nonzero count finalizes through exhaustion, block `break`, another non-local exit, explicit root `close`, `rewind`, or an exception, call `Rage.logger.warn` exactly once for that session. An unadvanced session and a session whose count is zero emit no warning.
- The warning must state that the aggregate count was encountered during this traversal. It must not claim to describe the whole snapshot or store and must not include task IDs, payloads, stored fields, raw bytes, or serialized exception details. Never use `puts` for this report.
- Repeated idempotent `close` calls must not repeat the warning. `rewind` finalizes and reports the old session, then leaves the Enumerator unadvanced; its next advancement starts a new session with a new zero count. Overlapping, nested, and Fiber-interleaved traversals keep independent counts and warning-finalization state.
- Abandoning a root or derived iterator without deterministic cleanup provides no warning-timing or warning-delivery guarantee, just as it provides no deterministic descriptor cleanup. Do not add eager scanning, forced garbage collection, or finalizer timing guarantees merely to emit a warning.
- `Rage::Deferred::DeadTasksLockTimeout` during snapshot acquisition and filesystem errors from open, read, seek, permissions, or descriptor cleanup are operational failures, not skipped items. Propagate them unchanged from the lazy advancement that performs the operation; do not translate them into a generic listing error. For block traversal, they propagate from the `each` call.
- Release any acquired lock and descriptor through `ensure`. If traversal, application code, or storage work is already raising, cleanup and warning delivery must not replace that original exception. If storage cleanup is the only failing operation, propagate that original cleanup exception rather than silently suppressing it. This requirement does not redefine `Rage.logger`'s general failure policy.

### Locking and cleanup

- Hold the dead-task lock only while opening the live file and capturing the snapshot boundary. Release it before frame validation, `Marshal.load`, entry wrapping, or yielding.
- Do not hold the lock while calling application code. A caller's block can be slow or can start another store operation that needs the same lock.
- Close the read descriptor automatically when traversal is exhausted or block/internal traversal unwinds through `break`, another non-local exit, or an exception.
- Finalize the traversal-local skipped-item warning in the same lifecycle cleanup, subject to the exception-precedence rules above.
- A long-lived descriptor can keep an old, unlinked snapshot file on disk after compaction. Its space is released when the descriptor closes.

For the root external iterator returned directly by `each`:

- `close` immediately closes any open snapshot descriptor, discards the old snapshot cursor and decoded batch, and finalizes the session's skipped-item warning. It returns `nil` after successful cleanup; an operational cleanup failure propagates unchanged.
- `close` is safe and idempotent before first advance, during partial traversal, and after normal exhaustion. Calling it before first advance must not open the store.
- After `close`, `next`, `next_values`, `peek`, and `peek_values` raise `StopIteration`. Block-based `each` invoked on that root yields no entries. No external advancement or iteration surface may open a new descriptor or resume the closed snapshot before `rewind`.
- `rewind` first performs the same cleanup and warning finalization as `close`, resets the iterator to its initial unadvanced state without opening the store, and returns the same iterator object after successful cleanup. Its next advancement through the Enumerator API establishes a fresh snapshot of the store as it exists then and starts a new skipped-item count.
- A caller that manually advances an iterator and may stop early must retain that root iterator and call `close` in `ensure`. Dropping the iterator and waiting for garbage collection is not deterministic cleanup.
- Derived external iterators, including values returned by `with_index` or `lazy`, may not expose `close`. A caller using them for partial external iteration must first retain the root returned by `DeadTasks#each`, derive from that root, and close the root. A directly derived external iterator for which the caller did not retain the root is outside the deterministic-cleanup guarantee. Closing the root guarantees release of its snapshot descriptor and prevents the derived traversal from resuming that snapshot; it does not promise that derived objects expose their own cleanup method or discard values they already buffered.
- An unreachable root or snapshot IO may be reclaimed by Ruby's normal object cleanup, or by a carefully implemented custom finalizer, as best-effort leak protection. This is not part of the deterministic API contract and no correctness or lifecycle test may depend on when GC runs. If implementation uses a custom finalizer, it must not strongly capture the iterator and should close the owned IO/state object rather than a raw file-descriptor integer that the operating system could have reused. A custom finalizer is optional, not required by this task.
- Block-based enumeration is the preferred ordinary path because Rage performs cleanup automatically.

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
- Document the accessor, immediate side-effect-free `batch_size` validation, collection, passive summary readers, stable-snapshot behavior, lazy reverse traversal, complete-traversal `seen_ids` memory, decoded-payload batching, traversal-scoped `Rage.logger.warn` reporting and redaction, unchanged operational-error propagation, automatic block cleanup, root external-iterator `close`/`rewind` lifecycle, derived-iterator limitation, old-inode lifetime, and sensitive-data risk.
- Document that reverse file reading and top-level Marshal decoding are synchronous work in the calling Fiber. Setup no longer scans the complete store under lock, but a large traversal can still delay other work in that worker.
- Preserve Ruby 3.3.0 compatibility and the existing `config.deferred.backend` contract.
- Do not add polling or work to successful deferred-task execution.

## Design

### How traversal works

1. `Rage::Deferred.dead_tasks` creates or returns the collection. It does not touch the store.
2. `DeadTasks#each` validates `batch_size` without touching the store, then creates a new private traversal session. The memoized collection does not register that session or copy its state onto itself.
3. When that traversal actually starts, Disk takes the permanent lock and opens the current live data file.
4. Disk finds the last newline at or before the current end of file. The position immediately after that newline is the fixed snapshot boundary. If the file already ends in a newline, its current size is the boundary; if it contains no newline, the boundary is zero. An excluded incomplete tail increments this traversal's skipped-item count once.
5. Disk releases the lock but keeps the read descriptor. All later snapshot reads are limited to bytes before the captured boundary.
6. A bounded reverse-line reader walks complete records from the boundary toward byte zero. It frame-validates and top-level-decodes records lazily.
7. For each record, Disk checks the complete schema and inner/outer ID match. An invalid record increments the traversal's count but does not reserve its ID. A fully valid record whose ID has not been seen is added to `seen_ids`, wrapped, and included in the next bounded batch. Older records for that ID cannot be yielded.
8. The iterator yields entries newest-first and performs no further storage work while suspended. It closes the descriptor and emits at most one nonzero-count warning when traversal exhausts or block/internal control flow unwinds, and finalizes immediately when the root external iterator receives `close` or `rewind`. Cleanup and warning state affect only this traversal session and do not mask an exception already in flight.

This design makes the snapshot stable in three different storage cases:

- A normal append writes after the fixed boundary, so the iterator never reads it.
- Task 01's torn-tail repair truncates only the excluded bytes after the last complete newline, then appends at or after the same boundary. Earlier snapshot bytes do not change.
- Rename-based compaction changes the live path, but the open descriptor still refers to the original snapshot inode.

Nil provides an empty version of the same internal traversal interface.

At the public boundary, wrap backend record hashes in passive `DeadTask` values. Keep opaque context bytes as private record data so task 03 can add detailed inspection later. Do not decode or expose those bytes in this task.

Block-based traversal needs no caller cleanup:

```ruby
dead_tasks.each do |entry|
  break if selected?(entry)
end
```

For partial external traversal, retain and close the root iterator:

```ruby
iterator = dead_tasks.each(batch_size: 100)

begin
  inspect_one(iterator.next)
ensure
  iterator.close
end
```

If a derived iterator is used, the root still owns cleanup:

```ruby
root = dead_tasks.each
indexed = root.with_index

begin
  inspect_one(indexed.next)
ensure
  root.close
end
```

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

### How compaction preserves permissions

1. During initialization, create missing live and lock files with a requested mode of `0600`; leave either file's existing mode unchanged. Do not modify the shared `.tmp` path without the permanent lock.
2. During compaction, acquire the permanent lock and open the current live file. Read its mode bits from that descriptor so the value belongs to the same live inode whose records are being copied.
3. While still holding that lock, open or create the temporary file without truncating it. Remove group and other access before truncating or writing any dead-task bytes, including when reusing a stale file.
4. Write the surviving records. If no requested ID exists, remove the temporary file and leave the live file and its mode unchanged.
5. If the temporary file will replace the live file, apply the captured live mode bits to it, `fsync` it, rename it over the live path, and `fsync` the directory.

This order keeps new scratch contents private while they are being built and prevents rename-based compaction from silently resetting an administrator's established live-file mode. It preserves mode bits, not ownership.

## Implementation constraints

- Do not change task 01's record format, filename version, write ordering, retention, or crash guarantees.
- Keep task 01 marked done and do not rewrite its specification or result. Implement the new dead-task permission policy in this task.
- Do not add public exact-ID lookup or context/argument readers. Those belong to task 03.
- Do not add `delete`, `retry`, or mutation wiring. Those belong to tasks 04 and 05.
- Do not inject or retain any mutation-capable object in a `DeadTask`.
- Do not expose an eager `all` API.
- Do not build the former complete ID/offset snapshot index.
- Do not store or cache traversal state on the memoized `DeadTasks` collection.
- Do not defer `batch_size` validation until first advancement or touch the backend while validating it.
- Do not mark an ID as seen until one of its records has passed frame, top-level decode, schema, and ID-consistency validation.
- Do not read or frame-parse beyond the captured complete-record boundary.
- Do not hold a filesystem lock while decoding a record or calling a user block.
- Do not constantize application classes.
- Do not deserialize or expose raw opaque context bytes.
- Do not describe `batch_size` as a bound on `seen_ids`, caller-retained entries, or the size of one stored physical record.
- Do not rely on garbage collection, finalizers, or an idle timeout as the cleanup mechanism for partial external traversal. Best-effort unreachable-object cleanup is allowed, but it is not a substitute for `close` or block unwinding.
- Do not add `close` to arbitrary derived Enumerators or claim that abandoning a root or derived iterator releases resources deterministically.
- Do not scan additional records merely to complete a skipped-item count, use `puts` for corruption reporting, emit more than one warning per finalized session, or disclose stored data in the warning.
- Do not rescue and translate `Rage::Deferred::DeadTasksLockTimeout` or filesystem `SystemCallError` failures into a generic public traversal error.

## Acceptance criteria

- [ ] `Rage::Deferred.dead_tasks` is memoized, uses the queue's configured backend instance, and returns the same shared public collection object on repeated calls.
- [ ] The shared collection retains no traversal state, every no-block `each` call returns a distinct root Enumerator, and block-based calls likewise own private traversal sessions.
- [ ] Obtaining the collection or an unadvanced Enumerator does not open, scan, or decode the store.
- [ ] `DeadTasks#each` validates `batch_size` immediately; invalid values raise `ArgumentError` before a no-block Enumerator is returned or block traversal begins and cause no backend or storage access.
- [ ] The directly returned root is a Ruby `Enumerator` and adds public, idempotent `close` returning `nil` after successful cleanup; closing before first advance, during partial traversal, or after exhaustion leaves every advancement and iteration surface exhausted without opening or resuming a snapshot.
- [ ] `rewind` closes the old snapshot, returns the same iterator after successful cleanup, remains lazy, and makes the next advance establish a fresh snapshot that can include changes made since the previous one.
- [ ] Overlapping external Enumerators, nested traversal, and same-process Fiber-interleaved traversal remain independent; cleanup, rewind, exhaustion, `break`, or an exception in one does not affect another.
- [ ] Snapshot boundaries are established independently on first advancement, so a later-starting Enumerator may include an append that an already-started Enumerator excludes.
- [ ] Disk setup captures an open descriptor and the last complete-record boundary under the permanent lock without scanning or decoding all earlier complete records, then releases the lock before validation or yielding.
- [ ] Traversal lazily reads complete records backwards, yields newest-first, retains an incremental `seen_ids` set instead of a full offset index, and holds at most one decoded payload batch.
- [ ] For duplicate IDs, the newest fully valid record is yielded. A newer malformed, CRC-invalid, Marshal-unreadable, non-Hash, schema-invalid, or inner/outer-ID-mismatched record does not hide an older fully valid record.
- [ ] Missing or wrongly typed required fields are skipped and counted without payload disclosure. On Disk, the outer framed ID is consistently used as the public and logical ID.
- [ ] Each traversal reports one aggregate nonzero skipped-item count through `Rage.logger.warn` exactly once when that session finalizes, describes it as encountered during that traversal, and never scans untouched records to complete the count. Zero-count and unadvanced traversals emit nothing.
- [ ] Exhaustion, early block exit, explicit `close`, `rewind`, and exception finalization emit the warning when required; repeated `close` does not duplicate it, rewound sessions start a new count, and overlapping or nested traversals keep independent counts.
- [ ] A normally continuing snapshot is unchanged by append, task 01 torn-tail repair plus append, or rename-based compaction performed through backend or storage hooks between yields.
- [ ] Stopping before another batch prevents parsing or decoding records belonging only to untouched older batches.
- [ ] Enumeration closes snapshot resources on exhaustion, block `break`, non-local exit, exception, and explicit root-iterator `close`, and never holds the dead-task lock while decoding or running user code.
- [ ] Snapshot-acquisition lock timeout and open/read/seek/permission/cleanup errors propagate unchanged from the lazy operation that encounters them. Cleanup occurs through `ensure`, does not mask an existing exception, and propagates its original storage exception when cleanup is the only failure.
- [ ] Summary entries are passive, read-only values and expose the documented typed metadata.
- [ ] Summary entries do not resolve task classes, deserialize contexts, or retain a collection, backend, operation delegate, or other mutation dependency.
- [ ] Nil enumeration is empty and creates no persistence or background activity.
- [ ] New Disk live, lock, and temporary files request `0600`; a restrictive umask can make them less permissive but never more permissive.
- [ ] Initialization preserves existing live and lock modes, while stale temporary files are hardened under the permanent lock before the backend can write dead-task bytes.
- [ ] A successful compaction replacement has the same mode bits as the live file it replaced, with the mode applied before the temporary-file `fsync`; permission-setting failures do not rename the replacement.
- [ ] Nil still creates no files, and Disk does not change directory modes, pending-WAL permissions, or file ownership.
- [ ] Public APIs and private helpers have the required YARD documentation.

## Verification

- Add focused specs for the accessor, collection, passive summary entries, Disk snapshot primitive, reverse-line reader, and Nil traversal behavior.
- Cover lazy collection creation, memoized wrapper identity, lazy root-Enumerator creation, distinct root identity for every no-block `each` call, independent block-based sessions, and actual `Enumerator` type behavior. Verify every invalid `batch_size` raises `ArgumentError` immediately from `each`, returns no Enumerator, invokes no backend/storage method, and opens no descriptor.
- Cover newest-first order and batch boundaries. Instrument decoding to prove that stopping before a later batch does not decode records belonging only to that older batch.
- Cover multiple physical records for one ID: newest valid wins; each of a newer malformed frame, invalid CRC, unreadable top-level Marshal payload, decoded non-Hash, missing field, wrong field type, and inner/outer ID mismatch falls back to the next older fully valid record.
- Verify that the opaque context is type-checked as a String but not deserialized. Cover invalid backtrace contents and all summary-field types.
- Create earlier complete records plus an incomplete tail, start the snapshot, then call task 01's real add path so it repairs the tail and appends. Verify that only the original complete logical records are yielded in unchanged order.
- Through backend or storage hooks, append and run rename-based compaction between yields. Verify that remaining snapshot entries are not skipped, duplicated, reordered, or replaced.
- Use a representative many-batch fixture to verify incremental `seen_ids` growth, the separate decoded-payload bound, and absence of an all-offset or all-payload cache.
- Verify that Disk holds the permanent lock only while opening the file and locating the boundary, releases it before record validation and the first yield, and preserves task 01's lock-timeout behavior for competing operations.
- Cover `close` before first advance, partial `next` followed by `close`, repeated idempotent `close`, and `close` after exhaustion. Verify that `close` returns `nil`; post-close `next`, `next_values`, `peek`, and `peek_values` raise `StopIteration`; block-based `each` on the closed root yields nothing; and none of these surfaces opens or resumes a snapshot before `rewind`.
- Cover automatic cleanup after normal exhaustion, block `break`, a non-local exit, and an exception.
- For corrupt records and an incomplete tail, verify `Rage.logger.warn` receives exactly one aggregate, redacted warning per nonzero-count traversal on exhaustion, block `break`/non-local exit, explicit `close`, `rewind`, and exception. Verify the message says the count was encountered during that traversal and does not claim a whole-store total.
- Verify an unadvanced traversal and a zero-count traversal emit nothing; early termination does not inspect older untouched records; repeated `close` does not duplicate the warning; a post-`rewind` session starts a fresh count; and overlapping, nested, and Fiber-interleaved traversals report independent counts.
- Cover two overlapping external Enumerators and nested `each`, including interleaved advancement from same-process Fibers. Close or rewind one while the other continues, and verify exhaustion, `break`, and an exception affect only the traversal where they occur.
- Start one Enumerator, append a record, and then start a second Enumerator. Verify that their independently timed first advancements produce independent snapshot boundaries: the first excludes the append and the second may include it.
- Cover `rewind` before and after advancement: it returns the root iterator, closes the old descriptor, opens nothing itself, and causes the next advance to see a fresh snapshot including intervening appends while preserving normal newest-first rules.
- After partial traversal and rename compaction, verify that `close` releases the descriptor and the old unlinked inode can be reclaimed.
- Cover partial external traversal through `with_index` and `lazy`: retain and close the root iterator, verify the descriptor closes, and do not require the derived iterator itself to expose `close`.
- Do not use forced GC or finalizer timing as evidence for deterministic cleanup. If best-effort unreachable-object cleanup is implemented, test it only as a non-contractual safety net and keep all lifecycle guarantees covered through explicit `close` or block unwinding.
- Inject `Rage::Deferred::DeadTasksLockTimeout` during first advancement and representative `SystemCallError` failures during open, reverse read/seek, permissions, and descriptor cleanup. Verify each propagates unchanged from the operation that encounters it, acquired resources are released, warning delivery does not mask an active traversal or user-block exception, and a storage cleanup exception propagates when it is the only failure.
- Cover missing task classes, immutable/defensive summary values, absence of task-02 mutation methods or dependencies, redacted skipped-record reporting, and empty Nil enumeration.
- Verify the mode bits of newly created live, lock, and temporary files under an ordinary umask and a more restrictive umask. Isolate or restore process-global umask changes so they cannot leak into other examples.
- Pre-create live and lock files with representative legacy/custom modes, initialize Disk, and verify those modes are unchanged. Pre-create a permissive stale temporary file and verify unlocked initialization does not modify it and locked compaction strips group/other access before any truncation or write.
- Exercise both successful and no-op compaction. Verify the replacement preserves representative live mode bits such as `0600`, legacy `0644`, and shared-group `0660`; the no-op path leaves the live inode and mode unchanged; and the temporary-file `fsync` occurs after its final mode is applied.
- Inject failures while hardening the temporary file and while applying the live mode. Verify the live path is not replaced, filesystem errors propagate, and no dead-task payload is disclosed. Verify no ownership-changing operation is required.
- Verify the configured directory mode and pending-task WAL file modes are unchanged by this task, and Nil creates no files.
- Run `bundle exec rspec spec/deferred/deferred_spec.rb spec/deferred/backends/disk_spec.rb` plus the new focused listing specs.
- Run the broader `bundle exec rspec spec/deferred` and RuboCop for changed Ruby/spec files.

## Result

- Implementation PR:
