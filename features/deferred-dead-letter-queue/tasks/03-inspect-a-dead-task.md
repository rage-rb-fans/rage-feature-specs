---
status: ready-for-development
---

# Inspect a dead task

## Goal

Support exact-ID lookup and detailed inspection of a dead task, including final exception information and lazily decoded original arguments, while keeping records with genuine context deserialization failures recoverable.

## Context

Task 02 establishes `Rage::Deferred.dead_tasks`, the `DeadTasks` collection, stable enumeration, and passive read-only `DeadTask` summary entries. Task 01's record keeps exception metadata outside an opaque marshaled execution context, so most inspection can remain available when the task class or context can no longer be loaded.

Task 02 removes task 01's private eager `find_dead_task` facade methods and `Disk::DeadTasksStorage#find` because that implementation does not provide this task's fixed complete-record snapshot, frame-only newest-winner selection, or cleanup guarantees. This task introduces the private exact-lookup capability anew for Disk and Nil; it does not inherit an existing lookup implementation.

This task depends on task 02 and owns REQ-08; REQ-09's exception-detail readers; REQ-10; REQ-17's exact-lookup frame validation, silent physical-corruption handling, newest-frame-valid selection, and selected-winner failure behavior; REQ-18; the detailed-inspection and exact-lookup parts of REQ-19 and REQ-20; REQ-26's exact-lookup error/cleanup behavior; REQ-28 and REQ-29; the exact-lookup portion of AC-03; AC-04; and the lookup/inspection portions of AC-07 and AC-11. It does not own Task 02's enumeration lifecycle or summary-only listing fields. Task 05 consumes the decoded arguments and error model for retry. Follow [ADR-001](../adr/001-public-dead-task-object-model.md).

## Requirements

- Add `DeadTasks#find_by_id(id)`. It accepts an exact String ID, returns the newest frame-valid logical `DeadTask` with that authoritative outer ID, and returns `nil` when no frame-valid record matches. Non-String IDs raise `TypeError`; values are not coerced.
- Do not define or override `DeadTasks#find` or `DeadTasks#detect`. They remain inherited from `Enumerable`, including block predicates, the optional fallback callable used when no entry matches, and standard no-block Enumerator behavior. Exact-ID lookup must not be implemented as an overload of `find`.
- Lookup does not resolve or instantiate the stored task class. A renamed, removed, anonymous, or otherwise unresolvable task class does not prevent metadata inspection.
- Extend `DeadTask` with read-only `exception_class`, `exception_message`, and `backtrace` readers. Class names and messages remain Strings; backtrace is an immutable/defensive Array of Strings. Do not reconstruct or raise the original exception.
- Add lazy `args` and `kwargs` readers. On first detailed access, deserialize the private stored context with `Marshal.load(..., freeze: true)`, extract the values through `Rage::Deferred::Context.get_args` and `.get_kwargs`, normalize accessor-returned `nil` values to frozen `[]` and `{}`, and preserve Ruby 3.3 positional/keyword separation.
- Cache the positional and keyword values only after both have been extracted successfully. Repeated reads after success return the same frozen values. A failed decode or extraction is not cached, so access can be retried after application constants or code are restored.
- The frozen Marshal object graph, frozen nil-normalization values, and frozen backtrace data must prevent callers from mutating the entry's stored replay input through `args`, `kwargs`, `backtrace`, or nested access supported by the chosen model.
- Define a dedicated public dead-task context-deserialization error. An exception raised by `Marshal.load` or either `Rage::Deferred::Context` accessor is translated at this boundary; the public error identifies the task ID and preserves the original exception as `cause`.
- A Rage-written top-level record remains listable and findable when its opaque context bytes are truncated or malformed, or when Marshal loading references a missing Ruby constant. Accessing `args` or `kwargs` raises the dedicated error and does not modify or delete the record.
- Treat the opaque context as trusted Rage-written data for the current storage version. Do not proactively check that the decoded context is an Array, enforce a minimum context length, or check that extracted positional and keyword values are an Array and Hash. Missing slots follow the current `Rage::Deferred::Context` accessors' `nil` behavior and therefore normalize to empty collections. Manually written or schema-drifted contexts are outside the current-version contract; any incompatible layout introduced by Rage must come with a future storage-version bump and its compatibility guardrails.
- Exact lookup does not provide a shared public-record schema/type-validation layer. On Disk, selection checks only that a candidate is newline-terminated inside the captured snapshot boundary, has valid frame structure and the expected operation, has a valid CRC, has a non-empty authoritative outer framed ID, and has that outer ID equal to the requested ID. It does not load or inspect the top-level Marshal payload before selecting the match.
- Lookup must not require the selected payload to decode to a Hash, contain required keys or expected value types, contain only String backtrace entries, store context as a String, or have an inner `record[:id]` equal to the authoritative outer ID. The outer framed ID is authoritative for selection and the public `id`.
- Disk lookup silently skips malformed frames and CRC failures and excludes an incomplete tail. It scans every complete record sequentially forwards through the fixed snapshot boundary, replacing the remembered serialized-payload location whenever another frame-valid record has the requested authoritative outer ID. A newer frame-invalid matching record does not replace an older frame-valid match. After the complete scan selects the newest frame-valid match, lookup must not fall back to an older duplicate because the selected payload later fails to decode or has an unexpected shape.
- Decode the selected winner only after selection. A selected top-level Marshal decoding error propagates unchanged. Any natural failure while turning a successfully decoded, semantically incompatible payload into the documented public model likewise propagates without storage-level fallback. These failures are distinct from the dedicated error used when a successfully constructed entry later cannot deserialize its opaque execution context.
- Invalid public input raises `TypeError` before backend access. Snapshot-acquisition `Rage::Deferred::DeadTasksLockTimeout` and filesystem open/read/seek failures are operational failures rather than skipped items and propagate unchanged from `find_by_id`; do not translate them into the dedicated context-deserialization error or another generic public error.
- Exact lookup releases locks and descriptors through `ensure`. Cleanup must not mask an exception already being propagated. If storage cleanup is the only failure, its original exception propagates instead of being silently suppressed. This does not change the dedicated error translation required later when a valid entry's opaque execution context cannot be deserialized.
- Introduce a private exact-lookup capability on the Disk and Nil backend facades when adding `DeadTasks#find_by_id`. The capability may be named `find_dead_task(id)`, but it is a new task-03 contract and implementation rather than preservation of task 01's removed method. `DeadTasks` validates the public ID and delegates through this capability without inspecting the backend class or containing Disk-, Nil-, or database-specific branches.
- Exact lookup is independent of every active enumeration session. Calling `find_by_id` while an Enumerator is paused must not advance, close, rewind, or replace that Enumerator's snapshot, and the lookup must not reuse its descriptor, cursor, winner index, or current decoded record.
- Disk, Nil, and future adapters must produce the same observable `find_by_id` result and the same public `DeadTask` entry shape. Disk performs a complete sequential forward scan of a fixed complete-record view, Nil returns `nil` without side effects, and a future database adapter may use an indexed query. Do not promise equal time or I/O complexity across adapters.
- Add shared backend-contract examples for exact lookup. Exercise them for Disk and Nil in this feature and structure them so a future adapter can reuse them. Do not add a database backend in this task.
- Never use `eval` and never expose opaque Marshal bytes. Document that detailed argument data may contain credentials or personal data and that Marshal input is trusted local application data only.
- Keep decoding and entry-construction helpers private with YARD `@private`. Fully document public readers, lookup return/error behavior, lazy decoding, immutability, and sensitive-data implications.
- Keep this task's additions passive. Exact lookup and detailed readers must not add or require a collection reference, raw backend, operation delegate, or another mutation dependency. If task 04 has already introduced private action wiring, preserve it unchanged rather than replacing it with inspection-specific construction.
- Preserve the backend abstraction: Nil lookup returns `nil`, performs no decoding, and creates no files; adding another conforming adapter must not require changes to `DeadTasks#find_by_id`.

## Design

After `DeadTasks#find_by_id` validates its String argument, delegate to the backend's private exact-lookup capability and wrap the backend-neutral internal record in the public model from task 02. Keep this path polymorphic: `DeadTasks` calls the capability and does not branch on backend class.

Disk implements the new private capability using task 02's fixed-boundary sequential forward framing rules and authoritative outer-ID rule. It scans every complete record from byte zero through the snapshot boundary without loading top-level payloads, remembers the serialized-payload location of each frame-valid matching outer ID, and replaces that location when a later frame-valid match appears. After the scan completes, it decodes only the newest remembered match. A newer frame-invalid match leaves the prior frame-valid location intact, while a selected payload decoding or record-shape problem never falls back to an older match. Enumeration and exact lookup may share suitable private framing/snapshot helpers or implement the same forward rules directly; this task does not require a new abstraction. Task 01's removed eager lookup must not be restored. Nil implements the new capability by returning `nil`. A future database adapter can satisfy the same private capability with an indexed query; this task neither defines its schema nor implements it.

Exception fields are copied directly from the selected Rage-written record. Context decoding is isolated behind the entry, uses frozen Marshal loading plus the existing `Rage::Deferred::Context` accessors, and is cached only after both argument values are extracted successfully. A failed decode or accessor call remains retryable after application code or constants are restored.

Do not couple basic inspection to task constant resolution. Decode only the context required to obtain positional and keyword arguments, and translate Marshal or accessor exceptions at the public boundary while retaining their causes. Rely on the current version's Rage-owned writer and `Context` accessors instead of adding speculative layout/type validation; introduce such guardrails with a future storage-version change.

## Implementation constraints

- Do not change task 01's serialized record or context layout in this task.
- Do not add deletion, retry, Rake formatting, filtering, or arbitrary search.
- Do not override, alias, or overload inherited `Enumerable#find` or `Enumerable#detect`.
- Do not load opaque context during list, summary-reader, or exception-reader access.
- Normalize only `nil` returned by the current `Rage::Deferred::Context` accessors to empty arguments; do not add Array shape, minimum-length, or extracted-value type checks for the current storage version.
- Do not delete or compact a record merely because its context cannot be decoded or accessed through `Rage::Deferred::Context`.
- Do not add mutation methods or mutation wiring; task 04 owns their introduction.
- Do not branch on backend class in `DeadTasks` or promise backend-independent exact-lookup complexity.
- Do not restore or delegate to task 01's removed eager `DeadTasksStorage#find`; implement the task-03 exact-lookup semantics as a new private capability.
- Do not retain or introduce a reverse-record scan or `ReverseLineReader`. Once exact lookup uses the forward scan, remove that helper and its dedicated implementation coverage because enumeration no longer uses it either.
- Do not return or decode before the complete forward frame-selection scan reaches the fixed snapshot boundary. Do not emit logging or terminal output merely to report malformed frames, invalid CRCs, or an incomplete tail.
- Do not call `Marshal.load` while searching for a matching frame. Decode only the selected match.
- Do not rescue or translate a selected match's top-level Marshal decoding failure, and do not continue to an older duplicate after a decoding or record-shape failure.
- Do not rescue and translate lookup lock-timeout or filesystem failures. Context-deserialization translation applies only after a public entry has been successfully constructed from the selected top-level record.
- Do not add a database backend.
- Preserve Ruby 3.3.0 behavior and the public names established by task 02.

## Acceptance criteria

- [ ] `find_by_id` accepts only an exact String and returns the newest frame-valid matching entry or `nil`. A newer malformed frame or invalid CRC permits an older frame-valid match; a newer frame-valid match always wins.
- [ ] Inherited `find` and `detect` retain standard Enumerable predicate, optional fallback callable, and no-block behavior without any exact-ID overload.
- [ ] Task 03 introduces a private exact-lookup capability for Disk and Nil from scratch. Both satisfy reusable shared exact-lookup contract examples and expose identical public result/entry shapes without backend-class branching in `DeadTasks`.
- [ ] Disk enumeration and exact lookup retain no reverse-record scan or `ReverseLineReader`; both use the documented fixed-boundary sequential forward approach.
- [ ] `find_by_id` can run while another traversal is paused without changing that traversal's snapshot or lifecycle.
- [ ] Exception class, message, and backtrace remain inspectable without resolving the task class or decoding the context.
- [ ] Args/kwargs decode lazily with correct positional/keyword fidelity and cannot be used to mutate stored replay input.
- [ ] Marshal deserialization and `Rage::Deferred::Context` accessor failures raise the dedicated error with ID and cause, are not cached, and leave summary/exception metadata and storage intact.
- [ ] Context decoding uses `Marshal.load(..., freeze: true)` plus `Rage::Deferred::Context.get_args`/`.get_kwargs`, normalizes accessor-returned `nil` values, caches only complete successful extraction, and does not duplicate context Array shape, minimum-length, or extracted Array/Hash type validation in the current storage version.
- [ ] Malformed frames, invalid CRCs, and an incomplete tail are silently skipped without payload disclosure. Lookup performs no top-level Marshal load during its complete sequential forward scan, remembers only the newest frame-valid matching payload location, and decodes it only after the scan reaches the fixed boundary.
- [ ] The selected winner is decoded only after frame selection. Its top-level Marshal decoding error propagates unchanged, and neither that error nor a non-Hash, missing or wrongly typed field, invalid backtrace/context value, or inner/outer-ID mismatch causes fallback to an older duplicate.
- [ ] Exact-lookup lock-timeout and filesystem errors propagate unchanged, resources are cleaned in `ensure`, cleanup does not mask an active exception, and an otherwise sole storage cleanup exception is not suppressed.
- [ ] Nil lookup remains empty and side-effect free, while adapter-specific lookup complexity remains outside the public guarantee.
- [ ] All public lookup/read/error APIs have complete YARD documentation.

## Verification

- Add focused specs for `find_by_id` exact/missing/non-String lookup and detailed entry readers through Disk and Nil. Reuse task 02's Disk physical-corruption matrix to verify fallback past a malformed frame or invalid CRC and consistent authoritative outer-ID handling.
- Verify inherited `find` and `detect` select entries with a block predicate, return the optional fallback callable's value only when no entry matches, do not invoke that fallback after a match, and return their standard Enumerator forms when called without a block.
- Add shared backend-contract examples for existing, missing, duplicate, newer-frame-invalid duplicate, selected-winner failure, result-shape, and Nil-style empty lookup behavior. Apply the relevant shared examples to Disk and Nil and make them reusable by future adapters.
- Cover exact lookup that finds a match despite malformed frames or invalid CRCs, returns `nil` when no frame-valid match exists, replaces an older frame-valid match with a newer one, retains an older match when a newer matching frame is invalid, and encounters an operational failure after skipped physical corruption or after a provisional match. Verify physical corruption is skipped silently and does not interact with a paused enumeration.
- Instrument lookup to prove it scans every complete snapshot record forwards and performs no `Marshal.load` before the fixed boundary is reached. Verify it decodes only the newest remembered frame-valid match, and that an unreadable selected payload raises its original decoding error without falling back to an older duplicate.
- Remove reverse-reader-specific specs and replace their read/seek failure coverage with equivalent sequential forward-scan coverage.
- Cover selected payloads that decode as a non-Hash, omit or mistype fields, contain invalid backtrace/context values, or have an inner/outer-ID mismatch. Do not require shared schema/type rejection or older-record fallback for any of these payload problems.
- Verify `DeadTasks#find_by_id` delegates through the new private backend capability without testing backend class identity or depending on task 01's removed lookup implementation. A lightweight conforming backend double may prove the collection needs no adapter-specific branch; do not implement a database adapter.
- Pause an external Enumerator after its first entry, call `find_by_id`, and resume the Enumerator. Verify the lookup uses an independent backend operation and does not advance, close, rewind, or replace the paused snapshot.
- Cover task classes that are loadable, renamed, removed, anonymous, or replaced by non-Class constants without breaking basic inspection.
- Cover empty, positional-only, keyword-only, and mixed arguments under Ruby 3.3; verify lazy decode and defensive/frozen values.
- Cover truncated context bytes, invalid Marshal, missing referenced constants, a raised context-accessor failure, repeat access after failure, and preservation of the dead record. Do not add a proactive rejection matrix for manually fabricated context shapes, missing slots, or wrong extracted value types.
- Verify `Marshal.load(..., freeze: true)`, delegation to both `Rage::Deferred::Context` accessors, nil normalization, stable successful identities, successful-only caching, and immutability of nested/shared argument graphs.
- Cover exception metadata and backtrace immutability without exception reconstruction, plus silent handling of invalid top-level records.
- Inject snapshot-acquisition `Rage::Deferred::DeadTasksLockTimeout` and representative open/read/seek/cleanup `SystemCallError` failures. Verify they propagate unchanged from `find_by_id`, acquired resources are released, an active failure is not masked by cleanup, and a sole storage cleanup failure propagates. Keep these distinct from opaque-context deserialization-error examples.
- Run the focused inspection specs, `bundle exec rspec spec/deferred/deferred_spec.rb spec/deferred/backends/disk_spec.rb`, the broader `bundle exec rspec spec/deferred`, and RuboCop for changed files.

## Result

- Implementation PR:
