---
status: todo
---

# List dead tasks

## Goal

Establish the public dead-task retrieval model with passive read-only summary entries, and expose the durable store as a newest-first Enumerable whose traversal lazily decodes bounded payload batches instead of loading all entries at once.

## Context

Task 01 implemented the Disk/Nil backend facade and durable record storage. Disk records contain summary failure metadata plus an opaque marshaled execution context, and duplicate physical records with one task ID represent one logical dead task using the latest record.

This task is the first public-API task and depends only on task 01. It implements the listing and summary portions of REQ-05 through REQ-09, REQ-17, REQ-19, REQ-20, and REQ-24 through REQ-26, plus the public-collection portions of AC-02, AC-03, AC-07, and AC-11. It establishes the stable traversal contract that later allows `dead_tasks.each(&:delete)` and `dead_tasks.each(&:retry)` to remain correct, but it does not add those methods or their private mutation wiring. Exact-ID inspection and lazy context decoding belong to task 03; task 04 introduces deletion and the private entry-action wiring, and task 05 extends that wiring for retry. Follow [ADR-001](../adr/001-public-dead-task-object-model.md).

## Requirements

- Add the memoized public accessor `Rage::Deferred.dead_tasks`, returning a `Rage::Deferred::DeadTasks` collection over `Rage::Deferred.__backend`, the same backend instance used by the queue. Obtaining the collection must not scan or preload the store.
- Define the public `Rage::Deferred::DeadTasks` collection and a passive, read-only `Rage::Deferred::DeadTask` summary value. At this stage an entry contains only its stored record data; it must not retain the `DeadTasks` collection, raw backend, an operation delegate, or any other mutation dependency. Entry construction and raw backend data remain private.
- `DeadTasks` includes `Enumerable`. `each(batch_size: 100)` returns an Enumerator when no block is given, rejects non-positive or non-Integer batch sizes with `ArgumentError`, and yields logical entries newest-first. Returning the Enumerator does not establish or decode the snapshot; traversal starts when that Enumerator is first advanced.
- Disk enumeration observes a stable snapshot of the latest valid record for each logical ID. Records appended after traversal starts are excluded. If the live store is appended to or compacted by rename through backend/storage hooks between yields, a normally continuing traversal must not skip, duplicate, reorder, or replace entries remaining in the starting snapshot. Task 02 specifies and verifies this behavior directly without public entry mutation methods; tasks 04 and 05 later rely on it for mutation-safe entry actions.
- Snapshot setup performs a full store scan under the shared lock and retains ID/offset metadata proportional to the number of distinct logical task IDs. Subsequent traversal lazily decodes and wraps at most `batch_size` record payloads at once. The batching bound applies to decoded payloads, not total snapshot memory, which is not bounded independently of store cardinality. The collection and iterator must not build or cache an eager array of all entries; a caller may still explicitly choose to do so with standard `Enumerable` methods such as `to_a`.
- Hold the shared dead-task lock only while establishing the snapshot. Never hold it while yielding public entries. Keep the snapshot read descriptor valid across rename-based compaction and close it in `ensure` on exhaustion, `break`, non-local exit, or exception.
- At this stage, `DeadTask` exposes the summary readers `id`, `task_class`, `attempts`, `enqueued_at`, and `failed_at`. IDs and class names are Strings, attempts is an Integer, and timestamp readers return `Time` values. Mutable values exposed to callers are defensive/frozen values rather than the backend record itself.
- Listing and summary readers do not resolve or instantiate the task class and do not deserialize the opaque execution context.
- Skip malformed, CRC-invalid, or unreadable top-level records, report a warning count without printing payloads, and continue yielding other records. Preserve task 01's latest-record handling for duplicate IDs.
- With the Nil backend, the accessor returns the same public collection type, enumeration yields nothing, and no storage file or background work is created.
- Keep storage/snapshot helpers private with YARD `@private`. Document the accessor, collection, passive entry summary readers, stable-snapshot behavior, the distinction between payload batching and proportional snapshot-index memory, full-scan lock behavior, the possibility that a large synchronous scan delays same-worker Fibers or causes competing lock timeouts, and the sensitive-data warning.
- Preserve Ruby 3.3.0 compatibility and the existing `config.deferred.backend` contract. Do not add polling or work to successful deferred-task execution.

## Design

Extend the internal backend facade with a snapshot/batch primitive consumed by `DeadTasks#each`. Accessing `Rage::Deferred.dead_tasks` only constructs the collection; the backend snapshot is opened on actual iteration. On Disk, open and scan the live file by path while holding the permanent lock, retain only the latest valid byte location for each ID in insertion order, and keep the opened read descriptor as the immutable snapshot. Release the lock before seeking, decoding, wrapping, and yielding each batch. An open descriptor continues to reference its original inode if another process appends to or compacts the live path by rename, so traversal continues against the unchanged snapshot while later operations target the live store separately. Nil supplies an empty implementation of the same internal interface.

Wrap backend record hashes at the public boundary. Retain any opaque context bytes privately as record data on the passive entry so task 03 can add detailed inspection without changing the collection model, but do not decode or expose them in this task.

## Implementation constraints

- Do not change task 01's record format, filename version, write ordering, retention, or crash guarantees.
- Do not add public exact-ID lookup, context/argument readers, delete, or retry in this task. Do not inject or retain any mutation-capable object in a `DeadTask`; task 02 establishes only passive summary values and the traversal contract needed by later tasks.
- Do not expose an eager `all` API.
- Do not hold a filesystem lock while calling a user block.
- Do not constantize application classes or expose raw Marshal bytes.
- A long-lived snapshot may retain an unlinked inode after compaction; document that disk space is released when the descriptor closes.
- Do not describe the O(distinct logical IDs) index as total-memory-bounded. Snapshot setup's full locked scan and proportional metadata are accepted constraints that may be optimized later without changing the public entry API.

## Acceptance criteria

- [ ] `Rage::Deferred.dead_tasks` is memoized, uses the queue's configured backend instance, and returns the shared public collection model.
- [ ] Obtaining the collection or an unadvanced Enumerator does not scan or preload records; actual Disk traversal performs the documented full locked setup scan, retains metadata proportional to distinct logical IDs, is newest-first, deduplicates by latest task ID, observes a stable snapshot, and materializes no more than one payload batch at a time.
- [ ] Live-store append or rename-based compaction performed through backend/storage hooks between yields cannot skip, duplicate, reorder, or replace entries remaining in a normally continuing snapshot traversal, establishing the storage-level behavior required by later entry actions without adding those actions in this task.
- [ ] Enumeration closes snapshot resources on every exit path and never holds the dead-task lock while user code runs.
- [ ] Summary entries are passive read-only values that expose the documented typed metadata without resolving task classes, deserializing contexts, or retaining the collection, backend, operation delegate, or another mutation dependency.
- [ ] Corrupt top-level records are skipped and counted without payload disclosure while valid records remain enumerable.
- [ ] Nil enumeration is empty and creates no persistence or background activity.
- [ ] Public APIs and private helpers have the required YARD documentation.

## Verification

- Add focused specs for the accessor, collection, entry summary model, Disk snapshot primitive, and Nil snapshot behavior.
- Cover lazy collection/Enumerator creation, enumerator return, invalid batch sizes, ordering, batch boundaries, duplicate IDs, append after snapshot setup, compaction through backend/storage hooks during traversal, proportional snapshot-index growth, and payload materialization bounds independent of that index. Use a representative many-batch fixture to verify that no eager entry/payload array is retained while the expected all-ID index remains.
- Verify that Disk holds the shared lock for the complete setup scan, releases it before the first yield, and preserves task 01's existing lock-timeout behavior for conflicting operations.
- Cover normal exhaustion, `break`, non-local exit, and raised-block cleanup; verify the filesystem lock is released before every yield.
- Cover malformed lines, invalid CRCs, unreadable top-level records, missing task classes, summary value types/immutability, absence of mutation methods/dependencies on task-02 entries, and empty Nil enumeration.
- Run `bundle exec rspec spec/deferred/deferred_spec.rb spec/deferred/backends/disk_spec.rb` plus the new focused listing specs.
- Run the broader `bundle exec rspec spec/deferred` and RuboCop for changed Ruby/spec files.

## Result

- Implementation PR:
