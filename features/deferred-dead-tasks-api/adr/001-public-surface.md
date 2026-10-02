---
status: accepted
---

# ADR-001: Shape of the public surface

## Context

The dead-tasks store is reachable only through the backend contract (`list_dead_tasks`, `find_dead_task`, `remove_dead_tasks`), which is internal and returns a Hash per record. That Hash holds the storage fields as written to disk, including `context`: a `Marshal` blob of the `Rage::Deferred::Context` Array.

The public API needs four operations (list, find, delete, replay). The engineer also requires three things: `dead_tasks.each(&:replay)` works (the collection is enumerable and a record can replay itself); `each` replaces a separate list method; and batching is never a public argument. Two more things limit its shape:

- Whatever is returned to the app becomes a contract. The storage record and the layout of the context Array must stay free to change.
- Decoding the context can raise (`NameError`, `ArgumentError`) when a task class or an argument class no longer exists. The storage keeps the context as a blob for this reason: reading a record must not depend on the app's classes.

## Options considered

### Option A — Module methods on `Rage::Deferred` that return the backend Hashes

`Rage::Deferred.dead_tasks(limit:, offset:)`, `.find_dead_task(id)`, `.delete_dead_tasks(ids)`, `.replay_dead_tasks(ids)`, each passing the backend's result through.

- Benefits: no new class; the smallest change.
- Drawbacks: the Hash keys and the `context` blob become public, so the storage format can no longer change without a break. The app has to call `Marshal.load` and know the index of the arguments in the context Array to see them. Four more methods on the `Rage::Deferred` module, each with a `dead_task` word in its name.

It lost because it makes the storage format public.

### Option B — Module methods on `Rage::Deferred` that return `DeadTask` objects

The same four module methods, but records are wrapped in a `Rage::Deferred::DeadTask` value object.

- Benefits: hides the storage record; one new class instead of two.
- Drawbacks: `Rage::Deferred` today has one user-facing method (`wrap`); this adds four that all belong to one topic. Later additions (a count, a filter) would each add another module method.

It lost on grouping only; it is the closest alternative.

### Option C — A `DeadTasks` object with batch writes only; records are read-only `DeadTask` objects

One module method, `Rage::Deferred.dead_tasks`, returns a memoized `Rage::Deferred::DeadTasks` with `list(limit:, offset:)`, a lookup by id, `delete(*ids)`, and `replay(*ids)`. Records are `Rage::Deferred::DeadTask` value objects with named readers and no write methods; `args` and `kwargs` decode the context on first read.

- Benefits: hides the storage record; one entry point on `Rage::Deferred`; every write is one rewrite of the store, whatever the number of ids.
- Drawbacks: `each(&:replay)` does not work, and the app has to collect ids by hand to act on a filtered set. The list of ids and `limit`/`offset` are batching arguments in the public API.

It lost because it does not meet the engineer's requirements.

### Option D — Option C, plus `Enumerable` on `DeadTasks` and `replay` / `delete` on `DeadTask`

`DeadTasks` includes `Enumerable`; `DeadTask#replay` and `DeadTask#delete` act on one record. The batch methods and `list` of Option C stay as the cheap path.

- Benefits: `each(&:replay)` works, and an app with a large store can clear it in one rewrite.
- Drawbacks: two ways to do each write and two ways to read. The id lists stay a public batching argument, which every later backend then has to support with the same cost promise.

It lost because the engineer ruled out batching as a public argument.

### Option E — `Enumerable` on `DeadTasks`, writes only on `DeadTask` (selected)

`DeadTasks` includes `Enumerable`; `each` yields `DeadTask` objects oldest first and returns an `Enumerator` without a block. `find_by_id` looks up one record and `find_by_ids` looks up several in one read. `DeadTask#replay` and `DeadTask#delete` are the only writes. There is no `list`, no `limit`/`offset`, and no write method that takes several ids.

- Benefits: one way to read and one way to write; `dead_tasks.each(&:replay)` and `dead_tasks.select { ... }.each(&:delete)` work; the shape is the one Ruby developers know from other queues' dead sets; the framework stays free to group work as each backend needs.
- Drawbacks:
  - Each record-level write rewrites the whole store, so the framework has to group them internally: removals made inside one `each` call are applied in one rewrite when `each` ends. Writes made outside a `DeadTasks#each` block (on the one record that `find_by_id`, `first`, or `last` returns, or on the Array that `select` or `to_a` returns) are not grouped.
  - No pagination: every read loads all records.
  - A record object keeps a reference to its collection and can act on a record that another worker already removed.
  - `Enumerable#find` takes a block, so the lookup by id cannot be named `find`.

### Naming the replay method `retry`

Rejected by the engineer: `retry` is a Ruby keyword. It is legal as a method name only when called on a receiver, and it would mix the operation up with the automatic retries of a pending task.

## Decision

Option E. `Rage::Deferred.dead_tasks` returns the same `Rage::Deferred::DeadTasks` object on every call. It includes `Enumerable` and has `each`, `find_by_id`, and `find_by_ids`. `Rage::Deferred::DeadTask` has the readers, `replay`, and `delete`.

- `each` reads a snapshot of the records, releases the store's lock, and only then yields. The lock is not re-entrant for a fiber, so yielding under it would make every `replay` or `delete` inside the block time out.
- `find_by_ids(ids)` returns a `DeadTasks` collection limited to the records with those ids, oldest first; unknown ids are skipped. The engineer added it as a read: N calls of `find_by_id` read the whole store N times, and one `find_by_ids` reads it once. It is a second method, not an Array form of `find_by_id`, so each method has one return type.
- `find_by_ids` does not return an Array, because the engineer requires `find_by_ids(ids).each(&:delete)` to be grouped. Grouping lives in `DeadTasks#each`; the framework has no way to group writes made inside `Array#each`.
- `first` and `last` mean first and last in time: `first(n)` is the `n` tasks that became dead first, `last(n)` the `n` that became dead last. The engineer requires this whatever order the backend returns records in (`list_dead_tasks` returns newest first).
- `each` yields oldest first, so that `first` and `last` agree with `each`, `to_a`, and every other `Enumerable` method. The rejected alternative keeps `each` newest first and gives `first` the opposite meaning; then `dead_tasks.first` and `dead_tasks.to_a.first` are different records.
- `first(n)` and `last(n)` return a `DeadTasks` collection, not an Array, because the engineer requires `first(20).each(&:delete)` to be grouped. Without an argument they return one `DeadTask` or `nil`. Both are defined on `DeadTasks`: `Enumerable#first(n)` returns an Array, and `Enumerable` has no `last`.
- `find_by_ids` raises `ArgumentError` for a value that is not an Array, and the constructor of `DeadTasks` never reads a missing id list as "the whole store". Otherwise `find_by_ids(nil).each(&:delete)`, with `nil` from a missing parameter, would clear the store.
- A limited `DeadTasks` holds ids, not a snapshot of records, and reads the store on every `each`, like the whole-store form. A held snapshot would yield records that are already gone.
- `each` has one path, with no early return for an empty collection, because grouped removals are applied where `each` ends and a second exit would have to be kept equal to the first.
- No public write method takes a list of ids or a batch size, and no method takes a limit or an offset. Grouping of writes is internal to the framework: `replay` and `delete` called inside the block of `each` collect their removals, and `each` applies them in one rewrite when it ends, also on `break` and on an error.
- The lookup by id is `find_by_id`, so `Enumerable#find` keeps its meaning.
- `DeadTask#task_class` returns the class name as a String, not the class, because the class may no longer exist.

## Consequences

- The storage record and the context layout stay internal. A change to either needs a change in `DeadTask` only.
- `DeadTask` is a public class: its reader and method names are a contract from the first release that contains them.
- A record whose arguments cannot be decoded can still be yielded, inspected for its error, and deleted; only reading `args` or `kwargs`, and replay, fail for it.
- The cost of clearing a large store depends on the framework alone. The app has no argument to tune it. `dead_tasks.each { |t| t.delete if ... }` costs one rewrite; `dead_tasks.select { ... }.each(&:delete)` costs one per record, because the writes happen after `each` has ended. The YARD text must show the first form.
- A record deleted inside an `each` block stays visible to other readers until that `each` ends.
- Every `Enumerable` method and `last` read the whole store, `first(n)` and `count` included.
- `DeadTasks` has two forms: the whole store, and the store limited to a set of records (from `find_by_ids`, `first(n)`, or `last(n)`). Both must behave the same in `each`, `first`, `last`, and grouping.
- `first(n)` differs from `Enumerable#first(n)` in its return type. Code that needs an Array calls `to_a`.
- A `DeadTask` is not a plain value: it holds its collection, so it must not be kept across a reload or passed to another process.
