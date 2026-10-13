---
status: done
---

# Read and inspect dead tasks

## Goal

Give application code a documented, read-only way to see dead Deferred tasks: enumerate them, look them up by id, and read why each one failed and with which arguments.

## Context

Since Rage 1.28.0 the backend stores dead tasks, but only the internal backend contract can read them, and it returns the storage Hash with a `Marshal` blob inside. Task 01 gives the backend `each_dead_task`, which yields one record at a time. This task adds, on top of it, the public entry point and the two public classes that the later tasks of the feature build on: delete and replay (task 03), and `find_by_ids`, `first`, and `last` (task 04).

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

- `DeadTasks#find_by_ids` and the limited form of `DeadTasks` it returns (task 04). In this task a `DeadTasks` always stands for the whole store.
- `DeadTasks#first` and `DeadTasks#last`. Until task 04, `first` is `Enumerable`'s and `first(n)` returns an Array; `last` does not exist. Task 04 is optional; if it is built, it ships in the same release as task 02.
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
- `each` yields only from inside the block of `each_dead_task`, which holds no lock of the store while it yields: task 03 calls the store from inside the block.
- `DeadTask` never returns the backend's Hash or the context blob, and no reader other than `args` and `kwargs` may decode the context.
- The decoding error of `args` and `kwargs` is passed on unchanged. Rejected: a new public error class for it. No requirement needs one, and it would be one more released name.
- `DeadTasks` takes only the backend. The limited form (an id index) is added by task 04, which is the first to need it.
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

**Core tests**

Existing examples changed: none.
Kept whole: TC-04 — parts: id; class name; attempts; enqueue time; failure time; exception class; exception message; backtrace
Kept whole: TC-06 — parts: `args`; `kwargs`
Kept whole: TC-07 — parts: `args`; `kwargs`

<!-- rage-flow:core-tests begin — written by a script, do not edit -->
- TC-01: spec/deferred/deferred_spec.rb — "Rage::Deferred.dead_tasks yields a DeadTask with the id of a stored dead task" — red: assertion
- TC-02: spec/deferred/dead_tasks_spec.rb — "Rage::Deferred::DeadTasks #each yields one DeadTask per stored record, oldest first" — red: assertion
- TC-04.1: spec/deferred/dead_task_spec.rb — "Rage::Deferred::DeadTask returns the stored id, class name, attempts, enqueue time, failure time, exception class, exception message, and backtrace" — revert: not proven (NameError)
- TC-04.2: spec/deferred/dead_task_spec.rb — "Rage::Deferred::DeadTask returns the stored id, class name, attempts, enqueue time, failure time, exception class, exception message, and backtrace" — revert: not proven (NameError)
- TC-04.3: spec/deferred/dead_task_spec.rb — "Rage::Deferred::DeadTask returns the stored id, class name, attempts, enqueue time, failure time, exception class, exception message, and backtrace" — revert: not proven (NameError)
- TC-04.4: spec/deferred/dead_task_spec.rb — "Rage::Deferred::DeadTask returns the stored id, class name, attempts, enqueue time, failure time, exception class, exception message, and backtrace" — revert: not proven (NameError)
- TC-04.5: spec/deferred/dead_task_spec.rb — "Rage::Deferred::DeadTask returns the stored id, class name, attempts, enqueue time, failure time, exception class, exception message, and backtrace" — revert: not proven (NameError)
- TC-04.6: spec/deferred/dead_task_spec.rb — "Rage::Deferred::DeadTask returns the stored id, class name, attempts, enqueue time, failure time, exception class, exception message, and backtrace" — revert: not proven (NameError)
- TC-04.7: spec/deferred/dead_task_spec.rb — "Rage::Deferred::DeadTask returns the stored id, class name, attempts, enqueue time, failure time, exception class, exception message, and backtrace" — revert: not proven (NameError)
- TC-04.8: spec/deferred/dead_task_spec.rb — "Rage::Deferred::DeadTask returns the stored id, class name, attempts, enqueue time, failure time, exception class, exception message, and backtrace" — revert: not proven (NameError)
- TC-06.1: spec/deferred/dead_task_spec.rb — "Rage::Deferred::DeadTask returns the positional and keyword arguments the task was enqueued with" — red: assertion
- TC-06.2: spec/deferred/dead_task_spec.rb — "Rage::Deferred::DeadTask returns the positional and keyword arguments the task was enqueued with" — red: assertion
- TC-08: spec/deferred/dead_tasks_spec.rb — "Rage::Deferred::DeadTasks #each still yields a DeadTask when the task class of the record no longer exists" — red: assertion
- TC-12: spec/deferred/dead_tasks_spec.rb — "Rage::Deferred::DeadTasks #each with the Nil backend yields nothing and returns the collection" — red: assertion
- TC-15: spec/deferred/dead_tasks_spec.rb — "Rage::Deferred::DeadTasks #each returns the record from find_by_id called inside the block of each" — red: assertion
<!-- rage-flow:core-tests end -->

**Edge tests**

Existing examples changed: none.
Core test changed after its red run: TC-04 compares `backtrace` with the stored backtrace, because the store keeps only the first 20 lines; its lines in the `core-tests` block are from `rage-flow revert-check --core`.
Ran: all 23 examples of `spec/deferred/dead_tasks_spec.rb`, `spec/deferred/dead_task_spec.rb`, and `spec/deferred/deferred_spec.rb` passed; none was skipped or pending.
Not run: none.
The tests still describe the two new classes by name as a string, so that the example names in the two blocks stay valid.
TC-14 holds a real `flock` on the store's lock file from a second file handle, with no reactor; it does not show the scheduler-aware `sleep` between attempts. Its `revert: fails` is the `NameError` of the missing class inside `raise_error`, not a behavior of the base.

<!-- rage-flow:edge-tests begin — written by a script, do not edit -->
- TC-03: spec/deferred/dead_tasks_spec.rb — "Rage::Deferred::DeadTasks #each returns an Enumerator over the same records without a block" — revert: not proven (NameError)
- TC-05: spec/deferred/dead_tasks_spec.rb — "Rage::Deferred::DeadTasks #find_by_id returns nil for an id that is not in the store" — revert: not proven (NameError)
- TC-07.1: spec/deferred/dead_task_spec.rb — "Rage::Deferred::DeadTask returns empty collections for a task enqueued without arguments" — revert: not proven (NameError)
- TC-07.2: spec/deferred/dead_task_spec.rb — "Rage::Deferred::DeadTask returns empty collections for a task enqueued without arguments" — revert: not proven (NameError)
- TC-09: spec/deferred/dead_task_spec.rb — "Rage::Deferred::DeadTask raises ArgumentError from args if a class of the stored context no longer exists" — revert: not proven (NameError)
- TC-13: spec/deferred/dead_tasks_spec.rb — "Rage::Deferred::DeadTasks #find_by_id with the Nil backend returns nil" — revert: not proven (NameError)
- TC-14: spec/deferred/dead_tasks_spec.rb — "Rage::Deferred::DeadTasks #each raises DeadTasksLockTimeout if the lock of the store cannot be taken" — revert: fails
- TC-16: spec/deferred/deferred_spec.rb — "Rage::Deferred.dead_tasks returns the same object on every call" — revert: not proven (NoMethodError)
- TC-18: spec/deferred/dead_tasks_spec.rb — "Rage::Deferred::DeadTasks #each does not yield a dead task added while the block runs" — revert: not proven (NameError)
- TC-19: spec/deferred/dead_tasks_spec.rb — "Rage::Deferred::DeadTasks #each yields one DeadTask for two records with the same task id" — revert: not proven (NameError)
<!-- rage-flow:edge-tests end -->

**Review outcome**

Full review on 2026-10-05. Reviewers A (runtime correctness), B (contract, cost, surface, and tests), and C (code against the Design) ran; none was skipped.
Reviewed code: `alex/deferred-dead-tasks-listing` plus one commit, HEAD 1aa7849, the whole range from b8c9a6c (merge base with `origin/main`). The engineer chose this range. It holds the backend part and the public classes in the same commits, and it is not the code the two records above were written for.
Report (a local file): `.workflow/evidence/deferred-dead-tasks-api/02-read-inspect-review.md`.
Applied: F2, F3, F7, F13, F14, F15, F16, F18, F19, F20, F21, F22, F23. Rejected: F24 (the Design chose `require_relative` over `autoload`).
Unmet: TC-09 (fix applied)
Final run: `rage-flow final-run` failed and wrote no block. None of the 25 mapped examples of the two records above is in the output: `spec/deferred/dead_task_spec.rb` does not exist on this code, and the other two files have other examples. No task example ran through the final run, so no criterion is proven.
Checks: `bundle exec rubocop` found no offenses. `bundle exec yardoc --fail-on-warning` fails on one warning outside this task, `lib/rage/sse/message.rb:13`.
Applied again on 2026-10-08: the branch was rewritten after the review (new HEAD 8521e86; `lib/` differs from the reviewed code only in comments and YARD text, the tests changed and were not reviewed again). The same 13 findings are applied on it; F22 needed no change, the branch has the changelog entry. The test line numbers in the lines below are those of the reviewed code. On 8521e86 the failing examples are `spec/deferred/dead_tasks_spec.rb:105`, `:116`, `:133`, `:247`, `:262`, `:289`, `:302`, `:341`, `:362`, `:371` and `spec/deferred/backends/disk_spec.rb:490`, `:936`; the example of F6 passes there.
The Design says no backend file changes; on this code `backends/disk.rb` and `backends/nil.rb` are changed, also by F3, F13, F14, F18, and F19. The Design is not patched: F1 decides first whether the backend part stays.
- F1 (needs the engineer's decision): `list_dead_tasks` is removed from the Disk and Nil backends, and `DeadTasksStorage#list`, `#find`, `#read_records`, `#entry_id` are gone; all were released in 1.28.0, and ADR-003 says they stay. Options: put the 1.28.0 code back beside the new read path; put the public methods back on top of the new frame parser, which stops listing a record with an empty id; keep the removal with a new ADR and a deprecation. Recommended: the first.
- F3b (needs the engineer's decision): a record with a valid checksum whose payload is not a dumped String still stops `each` and `find_by_id` with `RuntimeError`. Options: leave it as in 1.28.0; add `RuntimeError` to the rescue list, and change the examples at `spec/deferred/backends/disk_spec.rb:488` and `:957`. Recommended: leave it.
- F4 (needs the engineer's decision): `each` without a block, then `next`, runs the lock retry `sleep` in a blocking fiber and can freeze the worker for up to about 1.45 s. Options: a YARD note on `DeadTasks#each`; no `sleep` in `with_lock` for a blocking fiber under a scheduler. Recommended: the YARD note.
- F5 (needs the engineer's decision): `Rage::Deferred.dead_tasks` builds the Disk backend on first call, also in the master process before the fork, where the workers then share one lock file and one WAL storage. Options: a lazy backend in `DeadTasks`; fork-safe state in the Disk backend; a YARD warning on `Rage::Deferred.dead_tasks`. Recommended: the YARD warning in this task, and fork-safe state as its own task.
- F17 (needs the engineer's decision): `args` and `kwargs` return deep-frozen objects (`Marshal.load(…, freeze: true)`); the Design does not say so. Options: keep the freeze and write it into the Design; drop it. Recommended: keep it.
- F2b (needs a test, for task:build): TC-09 — an example where `args` raises `ArgumentError` for a class that no longer exists; the examples at `spec/deferred/dead_tasks_spec.rb:213`, `:228`, `:255`, `:268` require the removed error class and fail.
- F6 (needs a test, for task:build): the example at `spec/deferred/dead_tasks_spec.rb:65` expects frozen timestamps, which commit 1aa7849 removed, and checks that `delete` and `retry` do not exist; it fails.
- F7b (needs a test, for task:build): TC-05 — the example at `spec/deferred/dead_tasks_spec.rb:108` requires a `TypeError` from `find_by_id` that the code no longer raises.
- F8 (needs a test, for task:build): every line of the `core-tests` and `edge-tests` blocks names an example that is not in this code; the records need a red run and a revert check on it.
- F9 (needs a test, for task:build): TC-01 — an example that goes through `Rage::Deferred.dead_tasks.each` with a task in a Disk store.
- F10 (needs a test, for task:build): TC-08 — an example that hides the task class of a stored record and checks that `each` still yields it.
- F11 (needs a test, for task:build): TC-12, TC-13 — one example each through `Rage::Deferred.dead_tasks` with `config.deferred.backend = nil`.
- F12 (needs a test, for task:build): TC-15 — an example that calls `find_by_id` inside the block of `each` on a Disk backend.
- F14b (needs a test, for task:build): the examples at `spec/deferred/backends/disk_spec.rb:495` and `:915` expect no output for corrupted records; the code now prints the `WARNING:` line, as the Requirements say, and they fail.
- F16b (needs a test, for task:build): TC-16 — the examples at `spec/deferred/dead_tasks_spec.rb:97`, `:307`, `:328`, `:337` read `@backend` from a `DeadTasks` object; the ivar is `@__backend` now, and they fail.
- F25 (needs a test, for task:build): TC-03 — an example that reads the Enumerator from `each` and compares it with the block form.
- F26 (needs a test, for task:build): TC-14 — an example where a second handle holds the lock file and `DeadTasks#each` raises `DeadTasksLockTimeout`.

## Result

- Implementation PR: [rage-rb/rage#403](https://github.com/rage-rb/rage/pull/403)
- Merge commit: [49f690e](https://github.com/rage-rb/rage/commit/49f690e8075ffce7af4df7b422cac39a4630aec9)
