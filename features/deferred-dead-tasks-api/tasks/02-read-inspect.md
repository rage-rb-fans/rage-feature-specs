---
status: ready-for-development
---

# Read and inspect dead tasks

## Goal

Give application code a documented, read-only way to see dead Deferred tasks: enumerate them, look them up by id, and read why each one failed and with which arguments.

## Context

Since Rage 1.28.0 the backend stores dead tasks, but only the internal backend contract can read them, and it returns the storage Hash with a `Marshal` blob inside. Task 01 gives the backend `each_dead_task`, which yields one record at a time. This task adds, on top of it, the public entry point and the two public classes that the later tasks of the feature build on: `find_by_ids`, `first`, and `last` (task 03), delete (task 04), and replay (task 05).

## Requirements

### Public surface

```ruby
dead_tasks = Rage::Deferred.dead_tasks        # => Rage::Deferred::DeadTasks, the same object on every call

dead_tasks.each { |task| ... }                # yields Rage::Deferred::DeadTask, oldest first; returns dead_tasks
dead_tasks.each                               # => Enumerator
dead_tasks.find_by_id(id)                     # => Rage::Deferred::DeadTask or nil

task.id                # => String
task.task_class        # => String, the class name
task.attempts          # => Integer
task.enqueued_at       # => Time
task.failed_at         # => Time
task.exception_class   # => String
task.exception_message # => String
task.backtrace         # => Array<String>
task.args              # => Array, [] when the task had no positional arguments
task.kwargs            # => Hash, {} when the task had no keyword arguments
```

- `DeadTasks` includes `Enumerable`, so `map`, `select`, `count`, `to_a` and the others work through `each`.
- **Order.** "Oldest first" is the order in which the tasks became dead. The backend's `each_dead_task` yields in this order; `each` does not reorder.
- **Snapshot.** The set of records is fixed when `each` starts. A record added to the store after that is not yielded by this call; a record removed after that is still yielded. Every new call of `each` reads the store again.
- **Memory.** `each` and `find_by_id` never build a list of all records. `each` holds one record at a time and yields it before the next one is read. While the backend prepares the read, it also holds one id and one position per stored record (task 01); this does not grow with the size of the records.
- **Repeated ids.** If the store holds two records with the same task id, `each` yields one `DeadTask` for it, built from the newest record (the later failure), at the place of that record in the order, and `find_by_id` returns the same one.
- **No lock while yielding.** The store's lock is released before the first yield, so the block may read the store again.
- **`args` and `kwargs`** are decoded from the stored context on first read, not when the record is yielded. For a task enqueued through `Rage::Deferred.wrap`, `task_class` is the wrapper class and `args` starts with the wrapped object and the method name, as stored.

### Failures

- **The lock cannot be taken.** `each` and `find_by_id` raise `Rage::Deferred::DeadTasksLockTimeout`, as the backend does. Nothing was yielded and nothing changed.
- **A class of the app no longer exists.** `each` and `find_by_id` still return the record, and every reader except `args` and `kwargs` works. `args` and `kwargs` raise the error that decoding raises (`ArgumentError` from `Marshal.load` for a missing class). The error is raised on every read; a failed decode is not remembered.
- **A corrupted record in the store.** The backend skips it and prints its warning, as today; the record is not yielded.
- **A record that passes its checksum but cannot be decoded.** The backend prints its `ERROR:` line and skips the record (task 01): `each` does not yield it, and `find_by_id` returns `nil` for its id. For a repeated id only the newest record is tried.
- **The block of `each` raises or uses `break`.** The call ends there, and the backend ends its read (ADR-003). This task writes nothing, so nothing is left half-done.
- **Called twice.** Every operation is a read and gives the same result until the store changes.
- **`config.deferred.backend = nil`.** `each` yields nothing and `find_by_id` returns `nil`. No error is raised.

### Not in this task

- `DeadTasks#find_by_ids` and the limited form of `DeadTasks` it returns (task 03). In this task a `DeadTasks` always stands for the whole store.
- `DeadTasks#first` and `DeadTasks#last`. Until task 03, `first` is `Enumerable`'s and `first(n)` returns an Array; `last` does not exist. Tasks 02 and 03 ship in the same release.
- `DeadTask#delete`, `DeadTask#replay`, and the grouping of removals inside `each`.
- Any backend change. `each_dead_task` and the contract header are task 01.
- Behavior in a process that is not a running server (task 06).
- A custom `inspect`, a count, or any filter inside the store.

## Design

### Files

| File | Change |
| --- | --- |
| `lib/rage/deferred/dead_tasks.rb` | new: `Rage::Deferred::DeadTasks` |
| `lib/rage/deferred/dead_task.rb` | new: `Rage::Deferred::DeadTask` |
| `lib/rage/deferred/deferred.rb` | `Rage::Deferred.dead_tasks`; two `require_relative` lines after `"metadata"` |
| `CHANGELOG.md` | `[Deferred] Add Rage::Deferred.dead_tasks ...` under `## [Unreleased]` → `Added` |

No backend file, the queue, or `Rage::Deferred::Context` changes. `DeadTask` reads the record Hash whose keys task 01 writes into the contract header: `id`, `task_class`, `attempts`, `enqueued_at`, `failed_at`, `exception_class`, `exception_message`, `backtrace`, and `context` (`Marshal.dump` of the `Rage::Deferred::Context` Array).

### Interfaces

```ruby
module Rage::Deferred
  # @return [Rage::Deferred::DeadTasks]
  def self.dead_tasks
    @__dead_tasks ||= Rage::Deferred::DeadTasks.new(__backend)
  end
end

class Rage::Deferred::DeadTasks
  include Enumerable

  # @private
  def initialize(backend) end

  # @yieldparam task [Rage::Deferred::DeadTask]
  # @return [self, Enumerator]
  def each end

  # @param id [String]
  # @return [Rage::Deferred::DeadTask, nil]
  def find_by_id(id) end
end

class Rage::Deferred::DeadTask
  attr_reader :id, :task_class, :attempts, :exception_class, :exception_message, :backtrace

  # @private
  # record: the Hash the backend returns for one dead task
  def initialize(record) end

  def enqueued_at end   # => Time
  def failed_at end     # => Time
  def args end          # => Array
  def kwargs end        # => Hash
end
```

### How it works

`DeadTasks#each`:

1. Without a block, return `to_enum(:each)`.
2. Call `@__backend.each_dead_task` once, with a block that yields `DeadTask.new(record)` for each record it gets. The backend fixes the set of records, holds no lock while it yields, and gives the records oldest first, one at a time.
3. Return `self`.

`each` builds no Array and does not call `list_dead_tasks`.

`each` has this one path and no early return; ADR-001 says why.

`DeadTasks#find_by_id(id)`: calls `@__backend.find_dead_task(id)` and wraps the result, or returns `nil`.

`DeadTask`: `initialize` copies the plain fields of the record into ivars and keeps the context blob. `enqueued_at` and `failed_at` return `Time.at` of the stored Integer. `args` and `kwargs` call one private method that runs `Marshal.load` on the blob and memoizes the decoded context only when decoding succeeds; they then read it through `Rage::Deferred::Context.get_args` / `get_kwargs` and return `[]` / `{}` for `nil`.

### Blocking I/O

New call sites on the reactor, all through the existing store:

- `each_dead_task` in `each`: two reads of the whole dead-tasks file (ADR-003). The first runs in one piece before the first record is yielded; the second goes record by record, with one `Marshal.load` per yielded record. The lock is taken with `LOCK_NB` and a scheduler-aware `sleep` only to fix the set of records; it is not held during the reads or while the block runs.
- `find_dead_task` in `find_by_id`: one read of the whole file, because the newest record with the id can be anywhere in it, and one `Marshal.load` of that record.
- `Marshal.load` of one context in `args` / `kwargs`: CPU only, the size of one task's arguments.

No new lock, wait, `fsync`, thread, or process call. The cost of `each` and `find_by_id` grows with the size of the store, and the YARD text of both says so; the text of `each` also says that it reads the store twice.

### Boot and request cost

Two more small files are loaded when `Rage::Deferred` is first used; nothing is read from disk, opened, or scheduled until the app calls `Rage::Deferred.dead_tasks`. I chose `require_relative` over `autoload` to match the other files of `lib/rage/deferred/`.

## Implementation constraints

- Follows ADR-001.
- Follows ADR-003.
- This task changes no backend file and adds nothing to the backend contract; it uses `each_dead_task` and `find_dead_task` as task 01 leaves them.
- `each` must not call `list_dead_tasks` or collect the records into an Array before it yields: that brings back the memory use ADR-003 removes.
- `each` yields only from inside the block of `each_dead_task`, which holds no lock of the store while it yields: tasks 04 and 05 call the store from inside the block.
- `DeadTask` never returns the backend's Hash or the context blob, and no reader other than `args` and `kwargs` may decode the context.
- The decoding error of `args` and `kwargs` is passed on unchanged. Rejected: a new public error class for it. No requirement needs one, and it would be one more released name.
- `DeadTasks` takes only the backend. The limited form (an id index) is added by task 03, which is the first to need it.
- `DeadTask` takes only the record. A reference to its collection is added by task 04, which is the first to need it.
- No change to an existing method signature, config key, or telemetry span. `Rage::Deferred::DeadTasksLockTimeout` keeps its name and meaning.
- `# frozen_string_literal: true` in both new files. `initialize` of both classes is tagged `# @private`; internal ivars are `@__`-prefixed.
- Every user-facing method has YARD `@param` / `@return` / `@example` tags. Nothing newer than Ruby 3.3.0; no new gem.

## Acceptance criteria

- [ ] TC-01 (core, AC-01) When a task is in the dead-tasks store, `Rage::Deferred.dead_tasks.each` yields a `Rage::Deferred::DeadTask` with that task's id.
- [ ] TC-02 (core, AC-13) When `each` is called with a block, it yields one `DeadTask` per stored record, in the order the tasks became dead, oldest first.
- [ ] TC-03 (edge) When `each` is called without a block, it returns an `Enumerator` over the same records.
- [ ] TC-04 (core, AC-03) When `find_by_id` is called with the id of a stored record, the readers of the returned `DeadTask` return the stored id, class name, attempts, enqueue time, failure time, exception class, exception message, and backtrace.
- [ ] TC-05 (edge) If `find_by_id` is called with an id that is not in the store, it returns `nil`.
- [ ] TC-06 (core, AC-04) When `args` and `kwargs` are read on a `DeadTask`, they return the positional and keyword arguments the task was enqueued with.
- [ ] TC-07 (edge, AC-04) If a task was enqueued without arguments, `args` and `kwargs` return empty collections.
- [ ] TC-08 (core) If the task class of a stored record can no longer be loaded, `each` still yields a `DeadTask` for that record.
- [ ] TC-09 (edge) If the stored context cannot be decoded because a class no longer exists, reading `args` raises `ArgumentError`.
- [ ] TC-12 (core, AC-12) While `config.deferred.backend` is `nil`, `each` yields nothing.
- [ ] TC-13 (edge, AC-12) While `config.deferred.backend` is `nil`, `find_by_id` returns `nil`.
- [ ] TC-14 (edge) If the store's lock cannot be taken, `each` raises `Rage::Deferred::DeadTasksLockTimeout`.
- [ ] TC-15 (core) While the block of `each` runs, a `find_by_id` call made inside it returns its record.
- [ ] TC-16 (edge) `Rage::Deferred.dead_tasks` returns the same object on every call.
- [ ] TC-18 (edge, AC-21) If a dead task is added to the store while the block of `each` runs, that `each` call does not yield it.
- [ ] TC-19 (edge, AC-22) If the store holds two records with the same task id, `each` yields one `DeadTask` for that id.

## Verification

Tests to add:

- `spec/deferred/dead_tasks_spec.rb` — `Rage::Deferred::DeadTasks` against a Disk backend in a temporary directory (set up as in `spec/deferred/backends/disk_spec.rb`), and against `Backends::Nil`.
- `spec/deferred/dead_task_spec.rb` — `Rage::Deferred::DeadTask` readers, built from records the Disk backend returns.
- `spec/deferred/deferred_spec.rb` — an example for `Rage::Deferred.dead_tasks`.

No existing example checks behavior that this task changes.

Commands:

- `bundle exec rspec spec/deferred/dead_tasks_spec.rb spec/deferred/dead_task_spec.rb spec/deferred/deferred_spec.rb`
- `bundle exec rubocop lib/rage/deferred spec/deferred`
- `bundle exec yardoc --fail-on-warning`

## Result

- Implementation PR:
