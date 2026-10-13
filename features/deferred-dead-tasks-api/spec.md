---
title: Public API for dead Deferred tasks
tier: multi
---

# Public API for dead Deferred tasks

## Motivation

Since Rage 1.28.0, a Deferred task that uses up or aborts its retries is written to a durable dead-tasks store, but an application has no supported way to reach that store: the only access is the internal backend contract. After this feature ships, application code can list dead tasks, inspect one of them, delete them, and replay them through a documented API on `Rage::Deferred`, without touching the backend or the files on disk.

## Proposed behavior

```ruby
dead_tasks = Rage::Deferred.dead_tasks            # => Rage::Deferred::DeadTasks

# DeadTasks is Enumerable; `each` yields every dead task, oldest first
dead_tasks.each { |task| puts "#{task.task_class}: #{task.exception_message}" }
dead_tasks.each(&:replay)
dead_tasks.each { |task| task.delete if task.task_class == "SendWelcomeEmail" }
dead_tasks.each                                   # => Enumerator

task = dead_tasks.find_by_id("1759312800-4242-7") # => Rage::Deferred::DeadTask or nil

# Optional (task 04): collections limited to some of the records
dead_tasks.find_by_ids([id_1, id_2])              # => Rage::Deferred::DeadTasks, limited to these ids
dead_tasks.find_by_ids([id_1, id_2]).each(&:delete)

dead_tasks.first(20)                              # => Rage::Deferred::DeadTasks, the 20 tasks that became dead first
dead_tasks.last(20)                               # => Rage::Deferred::DeadTasks, the 20 tasks that became dead last
dead_tasks.first(20).each(&:delete)
dead_tasks.first                                  # => Rage::Deferred::DeadTask or nil

task.id                # => "1759312800-4242-7"
task.task_class        # => "SendWelcomeEmail" (a String, the class may no longer exist)
task.attempts          # => 21
task.enqueued_at       # => Time
task.failed_at         # => Time
task.exception_class   # => "Net::ReadTimeout"
task.exception_message # => "Net::ReadTimeout with #<TCPSocket:(closed)>"
task.backtrace         # => Array<String>
task.args              # => [42]
task.kwargs            # => { email: "user@example.com" }

task.replay                                       # enqueue this task again and remove its record
task.delete                                       # remove this record
```

`each` is the way to read the whole store; there is no `list`, no `limit`, and no `offset`. `DeadTask#replay` and `DeadTask#delete` are the only writes, and each acts on one record. No public write method takes a list of ids or a batch size.

`replay` and `delete` remove their record from the store at once, with one rewrite of the store per call. This is the same on the record that `find_by_id` returns and inside the block of `each`, so `dead_tasks.each(&:replay)` works and costs one rewrite per record.

Two parts of this API are optional, and the feature is complete without them: `find_by_ids`, `first(n)`, and `last(n)` (task 04), and the grouping of the removals made inside one `each` call into one rewrite (task 05). No other task depends on either of them. Whichever of the two is built second makes the grouping work on the limited collections.

`find_by_ids` takes an Array of ids and returns a `DeadTasks` collection limited to the dead tasks that have them. It is the same class as `Rage::Deferred.dead_tasks`, so it is `Enumerable`, yields oldest first, and, when task 05 is built, its `each` groups removals in the same way. Ids that are not in the store are skipped, and an id given twice yields one record. `to_a` gives a plain Array.

`first` and `last` mean first and last in time, whatever order the backend returns its records in: `first(20)` is the 20 tasks that became dead first (the oldest) and `last(20)` the 20 that became dead last (the newest). `each` yields in the same order, oldest first, so `first` and `last` always agree with `each`, `to_a`, and the other `Enumerable` methods.

`first(n)` and `last(n)` return a `DeadTasks` collection limited to those records, not an Array, so that `first(20).each(&:delete)` and `last(20).each(&:replay)` can be grouped (task 05). Without an argument, `first` and `last` return one `DeadTask` or `nil`. Both are defined on `DeadTasks`: `Enumerable#first(n)` returns an Array, and `Enumerable` has no `last`.

The lookup by id is named `find_by_id` because `Enumerable#find` takes a block; a `find(id)` would replace it and break `dead_tasks.find { ... }`.

The four operations the request names map to: read = `each`, inspect = `find_by_id` plus the readers of `DeadTask` (and `find_by_ids`, when task 04 is built), delete = `DeadTask#delete`, replay = `DeadTask#replay`.

### Constraints

Each one names what breaks when it is ignored.

- **The backend contract grows by exactly one method, `each_dead_task`.** See [ADR-003](adr/003-streaming-read.md). The rest of the feature is built from it, `find_dead_task`, `remove_dead_tasks`, and the task class's `enqueue`. Replay lives in the backend-independent layer. Every further contract method is a cost each later backend (Redis, SQL) pays forever, and `Backends::Nil` must meet it too.
- **Reading dead tasks does not load the store into memory.** `each` and `find_by_id` hold one record at a time, plus the ids in the store. The store has no size limit, so a read that builds an Array of all records uses memory without a bound; `list_dead_tasks` is not used by this feature for that reason.
- **The on-disk format does not change.** `DeadTasksStorage::STORAGE_VERSION` stays `"0"`. Records written by Rage 1.28.0 must be listed, deleted, and replayed; a format change makes dead tasks that already exist in production unreadable.
- **The public objects hide the storage record.** `DeadTask` exposes named readers, never the backend's Hash and never the `Marshal` blob of the context. Returning the Hash makes the storage format and the layout of `Rage::Deferred::Context` a public API that can no longer change.
- **`each` never yields while the store's lock is held.** The set of records is fixed when `each` starts, the lock is released, and only then are records read and yielded one by one (ADR-003). The lock is not re-entrant for a fiber: a `replay` or `delete` called inside the block would wait for a lock its own fiber holds and end in `DeadTasksLockTimeout`, so `each(&:replay)` could never work. A record that another worker removes after the snapshot is still yielded.
- **Optional (task 04): `find_by_ids` reads the store once.** It is built on `each_dead_task` and skips the records with other ids, whatever the number of ids. Calling `find_dead_task` per id reads the whole file once per id on the reactor, which is the cost this method exists to avoid; a new backend method for it would grow the contract.
- **Batching of writes is internal to the framework.** No public write method accepts a list of ids, a batch size, or any other batching argument. Removing one record rewrites the whole store (write, `fsync`, rename, directory `fsync`), so N single removals cost N rewrites on the reactor. A public batching argument would make one store's cost model a contract for every later backend.
- **A `replay` or `delete` removes its record at once.** Each call makes its own `remove_dead_tasks` call, one rewrite of the store, wherever it is made: on the one record that `find_by_id` returns, on the Array that `select` or `to_a` returns, and inside the block of `DeadTasks#each`. For `replay`, the removal (step 4 of ADR-002) runs right after the enqueue of the same record. `each(&:replay)` and `each(&:delete)` work this way, with one rewrite per record, and the YARD text of `replay` and `delete` must say what that costs. A removal that is held back for later, with no grouping to apply it, leaves a replayed task in the dead-tasks store, where the next `replay` runs it a second time.
- **Optional (task 05): removals made inside one `each` call are grouped into one rewrite.** When task 05 is built, `DeadTask#replay` and `DeadTask#delete` called inside the block of `DeadTasks#each` collect their removals, and `each` applies them with one `remove_dead_tasks` call when it ends. This holds for every `DeadTasks` collection, so with task 04 `find_by_ids(ids).each(&:delete)` and `first(20).each(&:delete)` are one rewrite too; `find_by_ids`, `first(n)`, and `last(n)` must not return a plain Array, whose `each` the framework cannot group. It applies them however it ends: normal return, `break`, or an error raised in the block. Removals that are dropped on `break` or on an error leave replayed tasks in the dead-tasks store, where the next `each(&:replay)` runs them a second time. Until `each` ends, a record deleted in the block is still visible to other readers; without task 05 it is removed at once, so this task changes what other readers see. A `replay` or `delete` called outside an `each` block (on the one record that `find_by_id`, `first`, or `last` returns, or on the Array that `select` or `to_a` returns) still removes its record at once, with one rewrite per call.
- **`each`, `find_by_id`, and `find_by_ids` never decode the context.** The arguments are decoded only when `args` or `kwargs` is read. A task class or an argument class that was removed from the app would otherwise make the whole list raise, and the list is the tool for finding exactly those records.
- **Replay keeps the task in at least one store at every moment.** See [ADR-002](adr/002-replay-semantics.md): the task is written to the pending-task WAL first and removed from the dead-tasks store second. The opposite order loses the task when the process dies between the two steps.
- **Replay uses the regular enqueue path.** See [ADR-002](adr/002-replay-semantics.md): the task class's `enqueue` is called with the stored arguments, so the `deferred.task.enqueue` span and the enqueue middleware run as for a task the app enqueues. A second, replay-only path would hide replays from the app's telemetry handlers and middleware, and would have to be kept equal to the regular one by hand.
- **Replay gives the task a new id and a full retry budget.** See [ADR-002](adr/002-replay-semantics.md). Reusing the id lets the removal step delete a new dead record of the same task; keeping the old attempt count sends the task back to the dead-tasks store on its first failure.
- **Only additive changes to released surfaces.** `Rage::Deferred::DeadTasksLockTimeout` was released in 1.28.0 and keeps its name and meaning; every operation of this API can raise it. No existing method signature or config key changes, and no config key is added.
- **No cost for apps that do not call the API.** Nothing is read, opened, or scheduled at boot or per request; `Rage::Deferred.dead_tasks` builds its object on first call. Work added to boot or to the enqueue path is paid by every app.
- **No new blocking call sites beyond the existing store.** Every operation reads the whole dead-tasks file on the reactor; the store's lock is held only to fix the set of records, not during the read. The YARD text of each method must say that the cost grows with the size of the store. `each` reads the file twice, to yield the newest record of a repeated id (ADR-003). A blocking `flock`, an `fsync` outside the store, or a third pass over the file freezes the worker for every request in it.
- **No new telemetry span.** Users match spans with wildcards (`handle "deferred.*"`), so a new span id starts firing handlers that already exist.
- **User-facing methods carry YARD `@param`/`@return`/`@example` tags, and the feature has a `[Deferred]` entry under `## [Unreleased]` in `CHANGELOG.md`.** Most contributors never see this repository; the code and the changelog are where they learn the API.
- **No new gem and nothing newer than Ruby 3.3.0.**

## Non-goals

- Retention: automatic cleanup, a size limit, or expiry of dead tasks.
- Counting, filtering, or searching dead tasks inside the store (by class, by error, by time). The `Enumerable` methods (`count`, `select`, `first`) work, and so does `last` when task 04 is built, but each of them reads the whole store; `first(20)` is not cheaper than `each`.
- A CLI command, a Rake task, or a web page on top of the API.
- Automatic replay, and changing the arguments of a task before replay.
- Pagination (`limit`, `offset`) and faster reads: an index, a cursor, or a read that covers only part of the file.
- Public batch operations: a method that deletes or replays a list of ids in one call.
- Changes to how a task becomes dead, or to the on-disk format and the write path of the dead-tasks storage. Removing `list_dead_tasks`, which this feature stops using, is also left out: it was released in 1.28.0 and needs its own deprecation.
- Protection against a second replay of the same dead task. In this feature, a record that two workers replay at the same time, or that is replayed again after a failed removal, is enqueued twice (see ADR-002). An idempotency key for replay is planned as later work; whether a second replay is not written to the WAL or is written but not run is not decided yet.

## Acceptance criteria

AC-18, AC-19, and AC-20 (task 04) and AC-23 (task 05) belong to optional tasks; the feature is complete without them.

- [ ] AC-01: When a Deferred task has used up its retries, `Rage::Deferred.dead_tasks.each` yields a `Rage::Deferred::DeadTask` with that task's id.
- [ ] AC-03: When `find_by_id` is called with the id of a stored record, the returned `DeadTask` exposes the task class name, attempts, enqueue time, failure time, exception class, exception message, and backtrace.
- [ ] AC-04: When `args` and `kwargs` are read on a `DeadTask`, they return the positional and keyword arguments the task was enqueued with.
- [ ] AC-07: When `delete` has returned for a `DeadTask` found with `find_by_id`, `find_by_id` returns `nil` for its id.
- [ ] AC-08: When `replay` is called on a `DeadTask`, the task's `perform` runs again with the same arguments.
- [ ] AC-09: When `dead_tasks.each(&:replay)` has returned, `each` yields no records.
- [ ] AC-10: When a replayed task fails, it is retried under its task class's retry policy with attempts counted from the first one.
- [ ] AC-11: If a stored record cannot be replayed because its context cannot be decoded, `DeadTask#replay` leaves the record in the store.
- [ ] AC-12: While `config.deferred.backend` is `nil`, `each` yields nothing and `find_by_id` returns `nil`.
- [ ] AC-13: When `each` is called with a block, it yields a `DeadTask` for every stored record, oldest first.
- [ ] AC-15: When `dead_tasks.each(&:replay)` is called, the `perform` of every dead task runs again.
- [ ] AC-16: When `dead_tasks.each(&:delete)` has returned, `each` yields no records.
- [ ] AC-17: When `replay` is called in a process that is not a running server (the console, a Rake task), a running server worker runs the task's `perform` without a server restart.
- [ ] AC-18: When `find_by_ids` is called with an Array of ids, the returned collection yields the `DeadTask` objects of the stored records that have those ids.
- [ ] AC-19: When `first(n)` is called, the returned collection yields the `n` tasks that became dead first.
- [ ] AC-20: When `last(n)` is called, the returned collection yields the `n` tasks that became dead last.
- [ ] AC-21: If a dead task is added to the store while the block of `each` runs, that `each` call does not yield it.
- [ ] AC-22: If the store holds two records with the same task id, `each` yields one `DeadTask` for that id.
- [ ] AC-23: When `delete` or `replay` is called on several dead tasks inside the block of one `each` call, their records are removed with one rewrite of the store, when that `each` call ends.

## Tasks

- [x] [01 — Streaming read in the backend](tasks/01-streaming-read.md)
  Adds `each_dead_task` to the backend contract, in Disk and Nil, as ADR-003 defines it, and makes `find_dead_task` in Disk stop loading the whole store. Writes the keys of a dead-task record into the contract header. No public API: the boundary with task 02 is the backend contract. Covers no feature criterion by itself; task 02 proves AC-21 and AC-22 through the public API on top of it.
- [x] [02 — Read and inspect dead tasks](tasks/02-read-inspect.md)
  Leaves the app able to enumerate and find dead tasks through `Rage::Deferred.dead_tasks` (`each`, `find_by_id`) and to read a `Rage::Deferred::DeadTask`. Built on `each_dead_task` from task 01. Read-only: stops before anything that changes the dead-tasks store. A `DeadTasks` here always stands for the whole store; the limited form is task 04. covers AC-01, AC-03, AC-04, AC-12, AC-13, AC-21, AC-22
- [ ] [03 — Delete and replay one dead task](tasks/03-delete-replay.md)
  Adds `DeadTask#delete` and `DeadTask#replay`. Each call acts on one record and removes it from the dead-tasks store at once, with one rewrite of the store per call. The app reaches the record through `find_by_id` or `each` from task 02, and the two methods work inside the block of `each` in the same way, one rewrite per record; nothing here needs task 04 or task 05. `replay` is the only operation that moves a task between the two stores. It is supported inside a running server only, and the YARD text says so. covers AC-07, AC-08, AC-09, AC-10, AC-11, AC-15, AC-16
- [ ] [04 — `find_by_ids`, `first`, and `last`](tasks/04-find-by-ids-first-last.md)
  Optional: no other task depends on it. Adds the limited form of `DeadTasks` and the three methods that return it: `find_by_ids`, `first(n)`, and `last(n)`; also `first` and `last` without an argument. Read-only. If it is built, it must ship in the same release as task 02: until it does, `first(n)` is `Enumerable`'s and returns an Array, and releasing that would make the change of its return type a break. covers AC-18, AC-19, AC-20
- [ ] [05 — Batch delete and replay](tasks/05-batch-delete-replay.md)
  Optional: no other task depends on it. Groups the removals that `delete` and `replay` make inside the block of one `each` call into one rewrite of the store, applied when `each` ends. The grouping is internal to the framework: no public method gets a list of ids or a batch size. It holds for every `DeadTasks` collection that exists when this task is built: the whole store, and the limited collections of task 04 if that task is built first. covers AC-23
- [ ] [06 — Replay outside a running server](tasks/06-replay-outside-server.md)
  Makes `replay` work from the console and from a Rake task, and removes the limit that task 03 documents. The replayed task must run in a server that is already running, without a restart. Today such a process writes the task to its own WAL file and schedules it on a reactor that is not running, so the task waits for a later server boot. covers AC-17

## Decisions

- [ADR-001 — Shape of the public surface](adr/001-public-surface.md)
- [ADR-002 — Replay semantics](adr/002-replay-semantics.md)
- [ADR-003 — Streaming read of the dead-tasks store](adr/003-streaming-read.md)

## References

- Rage issue:
- Rage implementation PR: [rage-rb/rage#403](https://github.com/rage-rb/rage/pull/403) (tasks 01 and 02)
- Storage this feature builds on: [deferred-dead-letter-queue, task 01](../deferred-dead-letter-queue/tasks/01-durable-dead-letter-storage.md), [rage-rb/rage#383](https://github.com/rage-rb/rage/pull/383)
