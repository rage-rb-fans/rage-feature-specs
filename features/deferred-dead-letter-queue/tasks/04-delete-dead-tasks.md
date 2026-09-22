---
status: todo
---

# Delete dead tasks

## Goal

Let trusted operators permanently delete one or more dead tasks through the Ruby API with precise return values, stable-enumeration behavior, and the crash/concurrency guarantees of the durable store.

## Context

Task 02 establishes `Rage::Deferred.dead_tasks`, stable snapshot enumeration, and a passive read-only `DeadTask` entry model with no mutation dependency. Task 01 implements durable multi-ID removal by locked temp-file compaction. This task introduces the private entry-action wiring and exposes deletion publicly; it remains independent of the detailed inspection behavior in task 03.

This task depends on task 02 and implements REQ-11 plus the deletion and enumeration-mutation portions of REQ-15, REQ-19, REQ-20, REQ-24 through REQ-26, AC-03, and AC-06. Follow [ADR-001](../adr/001-public-dead-task-object-model.md).

## Requirements

- Add `DeadTask#delete` and `DeadTasks#delete(*ids)` according to the final active-entry/collection model in ADR-001. In this task, introduce private action wiring to entries: a `DeadTask` retains its exact ID and a narrow delegate supporting `delete(id)`. The concrete delegate may be the creating `DeadTasks` collection, but it is not the traversal snapshot or raw backend, and callers cannot observe or replace it. Entry construction remains private.
- `DeadTask#delete` returns `true` only when that logical record was removed and `false` when it is missing or already removed.
- `DeadTasks#delete(*ids)` accepts String IDs and nested arrays of String IDs, flattens and de-duplicates them, returns the number of distinct logical records removed, and returns `0` for no IDs. Missing/stale IDs do not count and are not errors.
- Validate the complete ID input before mutating storage so invalid input cannot cause a partial deletion. Do not coerce IDs or interpolate them into code or shell commands.
- Delegate one normalized bulk removal to the configured backend so Disk retains task 01's single locked compaction and exact distinct-count semantics. Nil returns `0` and performs no persistence work.
- Preserve stable snapshot traversal when a caller deletes yielded entries. `dead_tasks.each(&:delete)` must attempt every logical entry present in the starting snapshot exactly once despite compaction/rename, and additions after snapshot setup remain excluded.
- Keep `entry.delete` and `dead_tasks.each(&:delete)` as immediate, mutation-safe convenience operations for individual entries or small sets. Because every successful immediate Disk removal can perform a full locked rewrite, deleting N entries this way may approach O(N²) aggregate storage work and is not the recommended large-set maintenance path.
- For a large selected set, prefer collecting exact IDs and passing them to one collection-level call:

  ```ruby
  ids = dead_tasks.filter_map { |dead_task| dead_task.id if removable?(dead_task) }
  dead_tasks.delete(ids)
  ```

  The accumulator retains O(selected IDs) Strings rather than all `DeadTask` entries; this is additional to task 02's incremental set of fully valid IDs already encountered during traversal. Disk performs one locked compaction for the normalized bulk call. Entries can become stale between selection and deletion, so missing IDs retain the documented non-error semantics.
- A stale entry object can delete only its exact logical ID. If another worker/process already removed it, return `false`; never delete a different or newly unrelated entry.
- Propagate `DeadTasksLockTimeout` and filesystem errors from the backend. Do not report success when task 01 cannot confirm durable replacement; preserve its documented uncertainty when rename succeeds but directory `fsync` fails.
- Concurrent deletions remain serialized by task 01's permanent lock. Overlapping callers may split the successful count according to lock order, but together remove only the requested logical IDs without resurrecting records or losing unrelated entries.
- A delete racing with retry does not cancel work already enqueued by the retry. Cross-operation behavior remains at-least-once and is documented for task 05 and task 06.
- Deletion output/errors must not expose record payloads. Fully document single/bulk return values, stale/missing behavior, validation, enumeration mutation, concurrency, and failure uncertainty with YARD.

## Design

Normalize and validate public IDs in `DeadTasks`, call `backend.remove_dead_tasks` once, and translate the single-entry count to a Boolean for `DeadTask#delete`. Extend task 02's centralized private entry construction to inject the deletion delegate only when this task adds `DeadTask#delete`. `DeadTasks` may itself serve as that private delegate; no separate actions class is required. Keep the mechanism narrow but extensible so task 05 can reuse it for `retry(id)` rather than introducing a second, incompatible entry dependency. Do not reimplement compaction in the public layer.

Task 02's open snapshot descriptor continues reading backwards from its fixed complete-record boundary on the original inode after a delete renames the live path. Mutation inside enumeration therefore cannot replace or skip remaining snapshot records. The iterator, not deletion, owns descriptor cleanup.

## Implementation constraints

- Do not change task 01's remove algorithm, lock strategy, file format, or crash guarantees.
- Do not add retry, detailed inspection, Rake commands, automatic retention, or deletion callbacks.
- Do not pass the raw backend or traversal snapshot into entries. This task owns the first mutation-capable dependency on `DeadTask`, limited to the private deletion/action protocol.
- Do not hold a dead-task store lock while yielding user code.
- Do not promise transactionality across separate public delete calls.
- Preserve ADR-001's active-entry and collection-mutation surfaces; do not replace either surface or otherwise change this task's semantics.
- Do not optimize repeated entry deletion by delaying an individual call's storage mutation or changing its immediate Boolean result.

## Acceptance criteria

- [ ] Single-entry deletion returns `true` only for an actual removal and `false` for missing/stale entries.
- [ ] Task 02's passive entries gain one private, narrow deletion delegate in this task; the delegate is not publicly accessible and does not expose the raw backend or active snapshot.
- [ ] Collection deletion handles one, many, nested, duplicate, missing, empty, and invalid inputs with exact documented results and validation-before-mutation.
- [ ] Disk delegates bulk deletion to one task-01 compaction; Nil returns zero without side effects.
- [ ] Deleting during stable enumeration processes the entire starting snapshot exactly once and closes snapshot resources.
- [ ] Documentation distinguishes mutation-safe small-set entry deletion from the one-compaction bulk-delete pattern and states both the O(selected IDs) accumulator and potential O(N²) repeated-compaction costs.
- [ ] Concurrent/overlapping deletions preserve unrelated records and divide successful counts according to lock order.
- [ ] Lock and filesystem failures are propagated without false success or payload disclosure.
- [ ] Public deletion APIs have complete YARD documentation.

## Verification

- Add focused collection/entry specs for single, bulk, nested, duplicate, missing, empty, invalid, and stale inputs and their exact return values.
- Verify entry construction adds the private deletion delegate only with this task, does not expose it publicly, and remains compatible with task 03's passive inspection fields whether task 03 is applied before or after this task.
- Verify invalid mixed input performs no backend removal and Nil performs no filesystem work.
- Exercise `each(&:delete)`, selective deletion inside an enumeration, concurrent append, and rename compaction without skipped/duplicated snapshot entries or leaked descriptors.
- Verify a collection-level call with many selected IDs delegates once to `remove_dead_tasks`, while repeated entry deletion remains immediate and invokes one removal per entry.
- Cover two backend instances/fibers deleting overlapping IDs, lock timeout, temp-write/rename/directory-fsync failures, and preservation of unrelated records using task 01's existing failure hooks.
- Run focused deletion and snapshot specs, `bundle exec rspec spec/deferred/backends/disk_spec.rb`, the broader `bundle exec rspec spec/deferred`, and RuboCop for changed files.

## Result

- Implementation PR:
