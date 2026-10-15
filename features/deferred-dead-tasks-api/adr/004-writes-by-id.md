---
status: accepted
---

# ADR-004: Delete and replay by id on the collection

## Context

ADR-001 makes `DeadTask#delete` and `DeadTask#replay` the only writes. An app that has only the id of a dead task (from a log line, an admin page, an error report) must first call `find_by_id` and then the write: two calls, and for `delete` one extra read of the whole store, because the backend can remove a record by id without reading it first.

ADR-001 rejected a collection with write methods (its Option D) for two reasons: the writes took lists of ids, which is public batching, and there were two ways to do each write. The engineer now asks for write methods by id on the collection, each with one id.

Two facts limit the shape:

- `remove_dead_tasks` removes by id without reading the record. Replay must read the record, because it needs the stored arguments.
- The backend's `remove_dead_tasks` accepts an Array of ids. A public method that passes its argument through would accept a list.

## Options considered

### Option A — Writes only on `DeadTask` (ADR-001 as it is)

- Benefits: one way to write.
- Drawbacks: an app with an id makes two calls, and `find_by_id(id).delete` reads the store once more than needed.

Lost: the engineer wants one call for an app that has the id.

### Option B — Both writes implemented on the collection; `DeadTask` calls them

`DeadTask#delete` calls `DeadTasks#delete(@id)`, and `DeadTask#replay` calls `DeadTasks#replay(@id)`.

- Benefits: each write has one implementation.
- Drawbacks: `DeadTasks#replay(id)` reads the store to find the record, so `dead_tasks.each(&:replay)` reads the whole store once more for every record, although the record object already holds the arguments.

Lost: it makes the most common bulk use slower in proportion to the size of the store.

### Option C — Accept several ids

`delete(*ids)` or `delete(ids)` on the collection.

Lost: public batching, which ADR-001 rules out.

### Option D — `delete(id)` implemented on the collection, `replay(id)` built on the record (selected)

- `DeadTasks#delete(id)` removes the records with this id, with one call of `remove_dead_tasks` and no read of the record. `DeadTask#delete` calls it with its own id.
- `DeadTasks#replay(id)` finds the record with `find_by_id` and calls `DeadTask#replay` on it. `DeadTask#replay` keeps the implementation, because it already holds the arguments.
- Both take exactly one String id and raise `TypeError` for anything else, like `find_by_id`. An Array is never passed through to the backend.

- Benefits: one call for an app that has the id; `dead_tasks.delete(id)` costs one rewrite and no extra read; `each(&:replay)` costs no extra read.
- Drawbacks: two ways to do each write. The implementation of `delete` lives on the collection and the one of `replay` on the record.

## Decision

Option D.

- `DeadTasks#delete(id)` returns `true` when it removed a record, `false` when no record had this id.
- `DeadTasks#replay(id)` returns `true` when a record with this id was found and its task was enqueued, `false` when no record had this id; then nothing is enqueued. Errors are the same as for `DeadTask#replay`.
- `DeadTask#delete` returns the result of `DeadTasks#delete`. `DeadTask#replay` returns `nil`: it does not look the record up, so it has nothing to report.

This replaces the sentence of ADR-001 that makes `DeadTask#delete` and `DeadTask#replay` the only writes. The rest of ADR-001 stays: no public write takes a list of ids or a batch size.

## Consequences

- The app has two ways to delete and two ways to replay. The YARD text of the collection methods names the record methods and says when to use each.
- A `DeadTask` still holds its collection (ADR-001); it reaches the store only through `DeadTasks#delete`.
- `dead_tasks.replay(id)` reads the store once to find the record, like `find_by_id(id).replay`; it is a shorter call, not a cheaper one.
- `DeadTasks` gets two public method names, `delete` and `replay`. `Enumerable` has neither, so no `Enumerable` method changes its meaning.
