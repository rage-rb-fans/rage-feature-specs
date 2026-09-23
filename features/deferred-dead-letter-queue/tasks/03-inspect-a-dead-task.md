---
status: ready-for-development
---

# Inspect a dead task

## Goal

Support exact-ID lookup and detailed inspection of a dead task, including final exception information and lazily decoded original arguments, while keeping damaged or incompatible contexts recoverable.

## Context

Task 02 establishes `Rage::Deferred.dead_tasks`, the `DeadTasks` collection, stable enumeration, and passive read-only `DeadTask` summary entries. Task 01's record keeps exception metadata outside an opaque marshaled execution context, so most inspection can remain available when the task class or context can no longer be loaded.

This task depends on task 02 and owns REQ-08; REQ-09's exception-detail readers; REQ-10; REQ-17's exact-lookup validation, silent corruption handling, and invalid-newer fallback; REQ-18; the detailed-inspection and exact-lookup parts of REQ-19 and REQ-20; REQ-26's exact-lookup error/cleanup behavior; REQ-28 and REQ-29; the exact-lookup portion of AC-03; AC-04; and the lookup/inspection portions of AC-07 and AC-11. It does not own Task 02's enumeration lifecycle or summary-only listing fields. Task 05 consumes the decoded arguments and error model for retry. Follow [ADR-001](../adr/001-public-dead-task-object-model.md).

## Requirements

- Add `DeadTasks#find_by_id(id)`. It accepts an exact String ID, returns the newest fully valid logical `DeadTask` with that ID, and returns `nil` when no fully valid record matches. Non-String IDs raise `TypeError`; values are not coerced. A newer invalid duplicate must not hide an older fully valid match.
- Do not define or override `DeadTasks#find` or `DeadTasks#detect`. They remain inherited from `Enumerable`, including block predicates, the optional fallback callable used when no entry matches, and standard no-block Enumerator behavior. Exact-ID lookup must not be implemented as an overload of `find`.
- Lookup does not resolve or instantiate the stored task class. A renamed, removed, anonymous, or otherwise unresolvable task class does not prevent metadata inspection.
- Extend `DeadTask` with read-only `exception_class`, `exception_message`, and `backtrace` readers. Class names and messages remain Strings; backtrace is an immutable/defensive Array of Strings. Do not reconstruct or raise the original exception.
- Add lazy `args` and `kwargs` readers. Deserialize the private stored context only on first detailed access, normalize stored nil values to `[]` and `{}`, and preserve Ruby 3.3 positional/keyword separation.
- Return defensive copies or frozen values so callers cannot mutate the entry's stored replay input through `args`, `kwargs`, `backtrace`, or nested access supported by the chosen model. Repeated reads must have documented, consistent identity/mutation behavior.
- Define a dedicated public dead-task deserialization error that identifies the task ID and preserves the original exception as `cause`.
- A valid top-level record remains listable and findable when its opaque context is truncated, malformed, references missing Ruby constants, or uses an unsupported/incompatible context layout. Accessing `args` or `kwargs` raises the dedicated error and does not modify or delete the record.
- Exact lookup uses the shared public-record schema and each backend's integrity rules. On Disk, it skips malformed frames, CRC failures, an incomplete tail, unreadable or non-Hash top-level payloads, missing or wrongly typed required fields, and inner/outer ID mismatches, and returns `nil` if no fully valid record for the requested authoritative outer ID remains.
- Disk lookup silently skips invalid physical records and excludes an incomplete tail. It does not inspect older untouched records after finding a fully valid match and emits no logging or terminal output for recoverable corruption.
- Invalid public input raises `TypeError` before backend access. Snapshot-acquisition `Rage::Deferred::DeadTasksLockTimeout` and filesystem open/read/seek failures are operational failures rather than skipped items and propagate unchanged from `find_by_id`; do not translate them into the dedicated context-deserialization error or another generic public error.
- Exact lookup releases locks and descriptors through `ensure`. Cleanup must not mask an exception already being propagated. If storage cleanup is the only failure, its original exception propagates instead of being silently suppressed. This does not change the dedicated error translation required later when a valid entry's opaque execution context cannot be deserialized.
- `DeadTasks` validates the public ID and delegates exact lookup through a private backend capability. It must not inspect the backend class or contain Disk-, Nil-, or database-specific branches. The existing private backend method name `find_dead_task(id)` may remain.
- Exact lookup is independent of every active enumeration session. Calling `find_by_id` while an Enumerator is paused must not advance, close, rewind, or replace that Enumerator's snapshot, and the lookup must not reuse its descriptor, cursor, `seen_ids`, or decoded batch.
- Disk, Nil, and future adapters must produce the same observable `find_by_id` result and the same public `DeadTask` entry shape. Disk may reverse-scan a fixed complete-record view, Nil returns `nil` without side effects, and a future database adapter may use an indexed query. Do not promise equal time or I/O complexity across adapters.
- Add shared backend-contract examples for exact lookup. Exercise them for Disk and Nil in this feature and structure them so a future adapter can reuse them. Do not add a database backend in this task.
- Never use `eval` and never expose opaque Marshal bytes. Document that detailed argument data may contain credentials or personal data and that Marshal input is trusted local application data only.
- Keep decoding and entry-construction helpers private with YARD `@private`. Fully document public readers, lookup return/error behavior, lazy decoding, immutability, and sensitive-data implications.
- Keep this task's additions passive. Exact lookup and detailed readers must not add or require a collection reference, raw backend, operation delegate, or another mutation dependency. If task 04 has already introduced private action wiring, preserve it unchanged rather than replacing it with inspection-specific construction.
- Preserve the backend abstraction: Nil lookup returns `nil`, performs no decoding, and creates no files; adding another conforming adapter must not require changes to `DeadTasks#find_by_id`.

## Design

After `DeadTasks#find_by_id` validates its String argument, delegate to the backend's private exact-lookup capability and wrap the backend-neutral internal record in the public model from task 02. Keep this path polymorphic: `DeadTasks` calls the capability and does not branch on backend class.

Disk uses task 02's frame/schema validator and authoritative outer-ID rule. It may traverse backwards from a fixed complete-record boundary and stop at the first fully valid record whose outer ID matches. It must continue past newer invalid duplicates rather than relying on task 01's current framing-only lookup behavior. Records older than that match remain untouched. Nil returns `nil`. A future database adapter can satisfy the same private capability with an indexed query; this task neither defines its schema nor implements it.

Exception fields are copied directly from the normalized record. Context decoding is isolated behind the entry and cached only after successful validation; a failed decode remains retryable after application code or constants are restored.

Do not couple basic inspection to task constant resolution. Decode only the context required to obtain positional and keyword arguments, and translate supported Marshal/layout failures at the public boundary while retaining their causes.

## Implementation constraints

- Do not change task 01's serialized record or context layout in this task.
- Do not add deletion, retry, Rake formatting, filtering, or arbitrary search.
- Do not override, alias, or overload inherited `Enumerable#find` or `Enumerable#detect`.
- Do not load opaque context during list, summary-reader, or exception-reader access.
- Do not silently turn damaged/incompatible contexts into empty arguments.
- Do not delete or compact a record merely because its context cannot be decoded.
- Do not add mutation methods or mutation wiring; task 04 owns their introduction.
- Do not branch on backend class in `DeadTasks` or promise backend-independent exact-lookup complexity.
- Do not scan past a valid match or emit logging or terminal output merely to report recoverable corruption.
- Do not rescue and translate lookup lock-timeout or filesystem failures. Context-deserialization translation applies only after a fully valid top-level record has been returned.
- Do not add a database backend.
- Preserve Ruby 3.3.0 behavior and the public names established by task 02.

## Acceptance criteria

- [ ] `find_by_id` accepts only an exact String and returns the newest fully valid matching entry or `nil`, with fallback past invalid newer duplicates.
- [ ] Inherited `find` and `detect` retain standard Enumerable predicate, optional fallback callable, and no-block behavior without any exact-ID overload.
- [ ] Disk and Nil satisfy reusable shared exact-lookup contract examples and expose identical public result/entry shapes without backend-class branching in `DeadTasks`.
- [ ] `find_by_id` can run while another traversal is paused without changing that traversal's snapshot or lifecycle.
- [ ] Exception class, message, and backtrace remain inspectable without resolving the task class or decoding the context.
- [ ] Args/kwargs decode lazily with correct positional/keyword fidelity and cannot be used to mutate stored replay input.
- [ ] Corrupt or incompatible contexts raise the dedicated error with ID and cause while leaving summary/exception metadata and storage intact.
- [ ] Corrupt top-level records and an incomplete tail are silently skipped without payload disclosure, and lookup stops inspecting older records after a fully valid match.
- [ ] Exact-lookup lock-timeout and filesystem errors propagate unchanged, resources are cleaned in `ensure`, cleanup does not mask an active exception, and an otherwise sole storage cleanup exception is not suppressed.
- [ ] Nil lookup remains empty and side-effect free, while adapter-specific lookup complexity remains outside the public guarantee.
- [ ] All public lookup/read/error APIs have complete YARD documentation.

## Verification

- Add focused specs for `find_by_id` exact/missing/non-String lookup and detailed entry readers through Disk and Nil. Reuse task 02's Disk corruption matrix to verify fallback past invalid newer duplicates and consistent authoritative outer-ID handling.
- Verify inherited `find` and `detect` select entries with a block predicate, return the optional fallback callable's value only when no entry matches, do not invoke that fallback after a match, and return their standard Enumerator forms when called without a block.
- Add shared backend-contract examples for existing, missing, duplicate, invalid-newer-duplicate, result-shape, and Nil-style empty lookup behavior. Apply the relevant shared examples to Disk and Nil and make them reusable by future adapters.
- Cover exact lookup that finds a match after invalid items, returns `nil` after invalid items, stops before older invalid records, and encounters an operational failure after invalid items. Verify recoverable corruption is skipped silently and does not interact with a paused enumeration.
- Verify `DeadTasks#find_by_id` delegates through the private backend capability without testing backend class identity. A lightweight conforming backend double may prove the collection needs no adapter-specific branch; do not implement a database adapter.
- Pause an external Enumerator after its first entry, call `find_by_id`, and resume the Enumerator. Verify the lookup uses an independent backend operation and does not advance, close, rewind, or replace the paused snapshot.
- Cover task classes that are loadable, renamed, removed, anonymous, or replaced by non-Class constants without breaking basic inspection.
- Cover empty, positional-only, keyword-only, and mixed arguments under Ruby 3.3; verify lazy decode and defensive/frozen values.
- Cover truncated context bytes, invalid Marshal, missing referenced constants, incompatible context shapes, repeat access after failure, and preservation of the dead record.
- Cover exception metadata and backtrace immutability without exception reconstruction, plus silent handling of invalid top-level records.
- Inject snapshot-acquisition `Rage::Deferred::DeadTasksLockTimeout` and representative open/read/seek/cleanup `SystemCallError` failures. Verify they propagate unchanged from `find_by_id`, acquired resources are released, an active failure is not masked by cleanup, and a sole storage cleanup failure propagates. Keep these distinct from opaque-context deserialization-error examples.
- Run the focused inspection specs, `bundle exec rspec spec/deferred/deferred_spec.rb spec/deferred/backends/disk_spec.rb`, the broader `bundle exec rspec spec/deferred`, and RuboCop for changed files.

## Result

- Implementation PR:
