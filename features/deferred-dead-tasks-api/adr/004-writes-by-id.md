---
status: accepted
---

# ADR-004: Delete and replay by id on the collection

## Context

ADR-001 makes `DeadTask#delete` and `DeadTask#retry` the only writes. An app that has only the id of a dead task (from a log line, an admin page, an error report) must first call `find_by_id` and then the write: two calls, and for `delete` one extra read of the whole store, because the backend can remove a record by id without reading it first.

ADR-001 rejected a collection with write methods (its Option D) for two reasons: the writes took lists of ids, which is public batching, and there were two ways to do each write. The engineer now asks for write methods by id on the collection. In the implementation, the engineer chose a `delete` that also takes a list of ids, and no replay by id.

Two facts limit the shape:

- `remove_dead_tasks` removes by id without reading the record. Replay must read the record, because it needs the stored arguments.
- The backend's `remove_dead_tasks` accepts an Array of ids. A public method that passes its argument through would accept a list.

## Options considered

### Option A — Writes only on `DeadTask` (ADR-001 as it is)

- Benefits: one way to write.
- Drawbacks: an app with an id makes two calls, and `find_by_id(id).delete` reads the store once more than needed.

Lost: the engineer wants one call for an app that has the id.

### Option B — Both writes implemented on the collection; `DeadTask` calls them

`DeadTask#delete` calls `DeadTasks#delete(@id)`, and `DeadTask#retry` calls `DeadTasks#replay(@id)`.

- Benefits: each write has one implementation.
- Drawbacks: `DeadTasks#replay(id)` reads the store to find the record, so `dead_tasks.each(&:retry)` reads the whole store once more for every record, although the record object already holds the arguments.

Lost: it makes the most common bulk use slower in proportion to the size of the store.

### Option C — Accept several ids

`delete(*ids)` or `delete(ids)` on the collection.

First lost: public batching, which ADR-001 rules out. Its `delete(ids)` form was then chosen (see Option E).

### Option D — `delete(id)` implemented on the collection, `replay(id)` built on the record

- `DeadTasks#delete(id)` removes the records with this id, with one call of `remove_dead_tasks` and no read of the record. `DeadTask#delete` calls it with its own id.
- `DeadTasks#replay(id)` finds the record with `find_by_id` and calls `DeadTask#retry` on it. `DeadTask#retry` keeps the implementation, because it already holds the arguments.
- Both take exactly one String id and raise `TypeError` for anything else, like `find_by_id`. An Array is never passed through to the backend.

- Benefits: one call for an app that has the id; `dead_tasks.delete(id)` costs one rewrite and no extra read; `each(&:retry)` costs no extra read.
- Drawbacks: two ways to do each write. The implementation of `delete` lives on the collection and the one of replay on the record.

Lost: the engineer chose Option E.

### Option E — `delete` on the collection takes one id or a list; no replay by id (selected)

- `DeadTasks#delete(id_or_ids)` takes one String id or one flat Array of String ids and removes them with one call of `remove_dead_tasks`, with no read of the records.
- No `DeadTasks#replay(id)`. An app that has only the id calls `find_by_id(id)&.retry`.
- `DeadTask#delete` calls `remove_dead_tasks` on the backend with its own id.

- Benefits: an app can remove a filtered set of records with one rewrite, also outside `each` and without task 05: `delete(dead_tasks.filter_map { |t| t.id if ... })`. One method to delete by id, whatever the number of ids.
- Drawbacks: a list of ids is a public batching argument, which every later backend has to support with the same cost promise. `delete` has two forms with one return type. Two ways to delete, and no one-call replay by id.

## Decision

Option E.

- `DeadTasks#delete(id_or_ids)` returns the number of distinct records it removed, an Integer. Repeated ids are removed once. An empty Array returns `0` without reading the store. A value that is not a String or a flat Array of Strings raises `TypeError` before the store is touched; the message does not show the value.
- There is no `DeadTasks#replay(id)`.
- `DeadTask#delete` returns `true` when it removed one record, `false` when the record was already gone. `DeadTask#retry` returns `true` or `false` (ADR-002).

This replaces the sentence of ADR-001 that makes `DeadTask#delete` and `DeadTask#retry` the only writes, and its rule that no public write takes a list of ids, for `DeadTasks#delete` only. The rest of ADR-001 stays: no other public write takes a list of ids, and none takes a batch size.

## Consequences

- The app has two ways to delete and one way to replay. The YARD text of `DeadTasks#delete` says when to use it instead of `DeadTask#delete`.
- A `DeadTask` holds the backend (ADR-001) and calls `remove_dead_tasks` on it directly; the two `delete` methods do not share code.
- `find_by_id(id)&.retry` reads the whole store twice: the lookup, then the check in step 1 of ADR-002.
- `DeadTasks` gets one public method name, `delete`. `Enumerable` does not have it, so no `Enumerable` method changes its meaning.
- `DeadTasks#delete` and `DeadTask#delete` return different types: an Integer count and a Boolean.
