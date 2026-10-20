---
status: accepted
---

# ADR-003: Streaming read of the dead-tasks store

## Context

The only way to read all dead tasks through the backend contract is `list_dead_tasks`, which returns an Array of every record. In the Disk backend it first puts every line of the file into a Hash and then decodes all of them, so one read holds about twice the store in memory, with every context blob and backtrace. The store has no size limit (retention is a non-goal of this feature), so this memory grows without a bound.

The engineer requires that reading dead tasks does not load the store into memory.

Three things limit the solution:

- ADR-001: `DeadTasks#each` must not yield while the store's lock is held, because `delete` and `replay` are called inside its block and the lock is not re-entrant for a fiber.
- The same task id can be in the store twice: a crash between the write to the dead-tasks store and the removal from the pending-task WAL makes the task run and die again. `list_dead_tasks` keeps the newest of the two, which it can only know after it has read the whole store.
- The backend contract is a cost every later backend pays; it grows only for a reason that holds for any store.

## Options considered

### Option A — Keep `list_dead_tasks`

- Benefits: no backend change; the public classes are already built on it.
- Drawbacks: memory grows with the store on every `each`, `find_by_id`, `count`, and `first`.

It lost because it does not meet the engineer's requirement.

### Option B — Page through `list_dead_tasks(limit:, offset:)`

The contract method already takes `limit` and `offset`.

- Benefits: no contract change.
- Drawbacks: in Disk every page reads and decodes the whole file again before it drops the rest, so memory does not go down and the work grows with the square of the store. Pages are separate reads, so a removal between two pages shifts the offsets and a record is skipped or yielded twice.

It lost on both memory and correctness.

### Option C — Yield from inside the store's lock

A method that reads line by line and yields each record while it holds the lock.

- Benefits: the simplest consistent read.
- Drawbacks: a `delete` or `replay` inside the block waits for a lock its own fiber holds and ends in `DeadTasksLockTimeout`. Other workers cannot add dead tasks for as long as the block runs.

It lost because it breaks ADR-001.

### Option D — A new contract method that yields one record at a time from a snapshot view (selected)

`each_dead_task` yields each record, oldest first. The backend fixes the set of records when the call starts and holds no lock while it yields.

- Benefits: memory is one record plus the ids in the store; the order is the store's own order, so nothing has to be reversed; the public semantics of ADR-001 (snapshot, no lock while yielding) stay the same.
- Drawbacks: the contract grows by one method. The newest-wins rule for a repeated id cannot be kept in one pass.

### Repeated ids: the first record wins

The backend remembers the ids it has yielded in this call and skips a later record with the same id.

- Benefits: one pass.
- Drawbacks: memory still grows with the number of ids, though not with the size of the records. For a repeated id, `each` and `find_by_id` show the older failure, and `list_dead_tasks` shows the newer one.

Lost: the engineer requires that the newest record of a repeated id is the one shown.

### Repeated ids: read the store twice to keep "newest wins" (selected)

A first pass finds the position of the newest record of each id, a second pass reads those records and yields them.

- Benefits: the same rule as `list_dead_tasks`, so every read of the contract agrees, and `find_dead_task` keeps its behavior. The record shown is the latest failure of the task.
- Drawbacks: two reads of the store on the reactor for every `each`; the first one runs in one piece before the first record is yielded. Memory grows with the number of ids (one id and one position per record), though not with the size of the records.

## Decision

Option D, and the newest record wins.

The backend contract gets one method:

```ruby
# Yield every dead task, oldest first, one record at a time.
# @yieldparam record [Hash] the same Hash that `find_dead_task` returns
# @return [void]
# @raise [Rage::Deferred::DeadTasksLockTimeout] if the dead-tasks store cannot be locked
def each_dead_task
end
```

[ADR-005](005-storage-errors.md) replaces the `@raise` line: `@raise [Rage::Deferred::StorageError] if the dead-tasks store fails`. `DeadTasksLockTimeout` is removed.

Every backend must keep these rules, which are stated without naming files:

- It yields each task id at most once, in the order the tasks became dead, oldest first. For an id that is stored more than once, it yields the newest record, at the place of that record.
- The set of records is fixed when the call starts: a record added later is not yielded, and a record removed later is still yielded.
- No lock of the store is held while a record is yielded.
- It holds one record at a time, plus what it needs to find the newest record of each id. Its memory does not grow with the size of the stored records.
- `Backends::Nil` yields nothing.

`find_dead_task` keeps its signature and its behavior. For a repeated id it returns the newest record, the same one that `each_dead_task` yields, and the Disk backend stops loading the whole store to answer it.

In the Disk backend the snapshot is the live file as it is when the call starts: the file is opened and its size is noted under the store's lock, the lock is released, and the file is read line by line up to that size: once to find the newest record of each id, and once more to yield. A removal replaces the live file by rename and leaves the opened file complete; an append lands after the noted size. The file is closed when the call ends, also on `break` and on an error in the block.

`list_dead_tasks` is not changed and not removed in this feature. Nothing in the framework calls it after this change. It was released in 1.28.0, so its removal needs its own proposal with a deprecation.

## Consequences

- `Rage::Deferred::DeadTasks#each` is built on `each_dead_task` and no longer reverses an Array. Every `Enumerable` method reads record by record; a method that builds a list (`to_a`, `select`, `sort_by`) still holds what it builds.
- The whole store is read on the reactor twice for every `each`, and once for `find_by_id`. The memory use gets better; the read time of `each` gets worse.
- While an `each` runs, the Disk backend keeps the old file open. If a removal replaces the live file in that time, the disk space of the old file is freed only when that `each` ends. An `Enumerator` from `each` without a block that is left half-read keeps the file open until it is collected.
- For a repeated id, `each` and `find_by_id` show the last failure, as `list_dead_tasks` does. `remove_dead_tasks` removes every record of an id, as before.
- The contract has a method that no code calls (`list_dead_tasks`) until it is deprecated and removed.
- A later backend must be able to give a fixed view of the store without a lock (a Redis or SQL backend would use a cursor over an ordered key or a snapshot read).
