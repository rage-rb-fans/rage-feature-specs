---
status: ready-for-development
---

# Read and inspect dead tasks

## Goal

Give application code a documented, read-only way to see dead Deferred tasks: enumerate them, look them up by id, and read why each one failed and with which arguments.

## Context

Since Rage 1.28.0 the backend stores dead tasks, but only the internal backend contract can read them, and it returns the storage Hash with a `Marshal` blob inside. This task adds the public entry point and the two public classes that the later tasks of the feature build on: `find_by_ids`, `first`, and `last` (task 02), delete (task 03), and replay (task 04).

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
- **Order.** "Oldest first" is the order in which the tasks became dead. The backend returns newest first; `each` reverses that.
- **Snapshot.** `each` reads all records before the first yield and yields from that snapshot. A record added to the store after that is not yielded by this call; a record removed after that is still yielded. Every new call of `each` reads the store again.
- **No lock while yielding.** The store's lock is released before the first yield, so the block may read the store again.
- **`args` and `kwargs`** are decoded from the stored context on first read, not when the record is yielded. For a task enqueued through `Rage::Deferred.wrap`, `task_class` is the wrapper class and `args` starts with the wrapped object and the method name, as stored.

### Failures

- **The lock cannot be taken.** `each` and `find_by_id` raise `Rage::Deferred::DeadTasksLockTimeout`, as the backend does. Nothing was yielded and nothing changed.
- **A class of the app no longer exists.** `each` and `find_by_id` still return the record, and every reader except `args` and `kwargs` works. `args` and `kwargs` raise the error that decoding raises (`ArgumentError` from `Marshal.load` for a missing class). The error is raised on every read; a failed decode is not remembered.
- **A corrupted record in the store.** The backend skips it and prints its warning, as today; the record is not yielded.
- **The block of `each` raises or uses `break`.** The call ends there. This task writes nothing, so nothing is left half-done.
- **Called twice.** Every operation is a read and gives the same result until the store changes.
- **`config.deferred.backend = nil`.** `each` yields nothing and `find_by_id` returns `nil`. No error is raised.

### Not in this task

- `DeadTasks#find_by_ids` and the limited form of `DeadTasks` it returns (task 02). In this task a `DeadTasks` always stands for the whole store.
- `DeadTasks#first` and `DeadTasks#last`. Until task 02, `first` is `Enumerable`'s and `first(n)` returns an Array; `last` does not exist. Tasks 01 and 02 ship in the same release.
- `DeadTask#delete`, `DeadTask#replay`, and the grouping of removals inside `each`.
- Behavior in a process that is not a running server (task 05).
- A custom `inspect`, a count, or any filter inside the store.

## Design

### Files

| File | Change |
| --- | --- |
| `lib/rage/deferred/dead_tasks.rb` | new: `Rage::Deferred::DeadTasks` |
| `lib/rage/deferred/dead_task.rb` | new: `Rage::Deferred::DeadTask` |
| `lib/rage/deferred/deferred.rb` | `Rage::Deferred.dead_tasks`; two `require_relative` lines after `"metadata"` |
| `lib/rage/deferred/backends/disk.rb` | comment only: the contract header says what `list_dead_tasks` and `find_dead_task` return |
| `CHANGELOG.md` | `[Deferred] Add Rage::Deferred.dead_tasks ...` under `## [Unreleased]` → `Added` |

No backend code, the queue, or `Rage::Deferred::Context` changes.

The contract header in `backends/disk.rb` is the only place the backend contract is written down. `DeadTask` reads the returned Hash, so the header's entries for `list_dead_tasks` and `find_dead_task` now list its keys (`id`, `task_class`, `attempts`, `enqueued_at`, `failed_at`, `exception_class`, `exception_message`, `backtrace`, `context`) and say that `context` is `Marshal.dump` of the `Rage::Deferred::Context` Array. This states the return shape that already exists; it adds no method.

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
2. Call `@__backend.list_dead_tasks` once. The backend takes the lock, reads, and releases the lock before it returns the Array of Hashes, newest first.
3. Walk that Array with `reverse_each` and yield `DeadTask.new(record)` for each record, then return `self`.

`each` has this one path and no early return; ADR-001 says why.

`DeadTasks#find_by_id(id)`: calls `@__backend.find_dead_task(id)` and wraps the result, or returns `nil`.

`DeadTask`: `initialize` copies the plain fields of the record into ivars and keeps the context blob. `enqueued_at` and `failed_at` return `Time.at` of the stored Integer. `args` and `kwargs` call one private method that runs `Marshal.load` on the blob and memoizes the decoded context only when decoding succeeds; they then read it through `Rage::Deferred::Context.get_args` / `get_kwargs` and return `[]` / `{}` for `nil`.

### Blocking I/O

New call sites on the reactor, all through the existing store:

- `list_dead_tasks` in `each`: one read of the whole dead-tasks file under the store's lock, plus one `Marshal.load` per record. The lock is taken with `LOCK_NB` and a scheduler-aware `sleep`; it is not held while the block runs.
- `find_dead_task` in `find_by_id`: the same full read.
- `Marshal.load` of one context in `args` / `kwargs`: CPU only, the size of one task's arguments.

No new lock, wait, `fsync`, thread, or process call. The cost of `each` and `find_by_id` grows with the size of the store, and the YARD text of both says so.

### Boot and request cost

Two more small files are loaded when `Rage::Deferred` is first used; nothing is read from disk, opened, or scheduled until the app calls `Rage::Deferred.dead_tasks`. I chose `require_relative` over `autoload` to match the other files of `lib/rage/deferred/`.

## Implementation constraints

- Follows ADR-001.
- The backend contract does not grow and no backend code changes; the only edit to a backend file is the contract header comment.
- `each` yields only after `list_dead_tasks` has returned. Never yield inside a backend call or while the store's lock is held: tasks 03 and 04 call the store from inside the block.
- `DeadTask` never returns the backend's Hash or the context blob, and no reader other than `args` and `kwargs` may decode the context.
- The decoding error of `args` and `kwargs` is passed on unchanged. Rejected: a new public error class for it. No requirement needs one, and it would be one more released name.
- `DeadTasks` takes only the backend. The limited form (an id index) is added by task 02, which is the first to need it.
- `DeadTask` takes only the record. A reference to its collection is added by task 03, which is the first to need it.
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
