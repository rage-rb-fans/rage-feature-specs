---
status: ready-for-development
---

# Backend-neutral storage errors

## Goal

Make every failure of the dead-tasks store reach the app as `Rage::Deferred::StorageError`, with the store's own error as `cause`, and remove `Rage::Deferred::DeadTasksLockTimeout`. After this task, the public `@raise` tags of the dead-tasks API name no error that belongs to the Disk backend. The task also moves the two dead-task errors under `Rage::Deferred::DeadTask`, as `DeadTask::RetryError` and `DeadTask::ContextDeserializationError`.

## Context

Tasks 01–03 built the dead-tasks API on the Disk backend. Its public methods now document Disk's errors:

- `DeadTask#delete` and `DeadTasks#delete` list `Rage::Deferred::DeadTasksLockTimeout` and `SystemCallError`.
- `DeadTask#retry` lists `DeadTasksLockTimeout` and `SystemCallError` ("when lookup or synchronization fails").
- In `lib/rage/deferred/backends/disk.rb`, `DeadTasksStorage#with_lock` raises `DeadTasksLockTimeout` after `LOCK_MAX_ATTEMPTS` tries. Every `File`, `IO`, `flock`, and `fsync` error (`Errno::*`, `IOError`) propagates unchanged from `add`, `each`, `find_by_id`, `remove`, and `TasksStorage#sync!`.

A later Redis or SQL backend would raise its own driver's errors, so code that rescues these errors breaks when the backend changes. ADR-005 decides that the backend raises one backend-neutral class instead.

`DeadTasksLockTimeout` was released in 1.28.0. In 1.28 only `Queue#abandon_task` can raise it, inside the fiber that processes a task, so no public method raises it to app code. ADR-005 removes it without a deprecation.

`Rage::Deferred::DeadTaskRetryError` and `Rage::Deferred::DeadTaskContextDeserializationError` are defined at the top of `Rage::Deferred`, in `lib/rage/deferred/deferred.rb`, but only `Rage::Deferred::DeadTask` raises them. Tasks 02 and 03 added them, and they are not released yet (the dead-tasks API is under `## [Unreleased]`), so they can move without a deprecation. ADR-006 decides the new names.

## Terms

- **Store error** — an error that the store itself raises: in the Disk backend, `SystemCallError` and `IOError` from a call on a file, a lock, or a directory, and a lock that times out.

## Requirements

### `Rage::Deferred::StorageError`

```ruby
begin
  Rage::Deferred.dead_tasks.delete(ids)
rescue Rage::Deferred::StorageError => e
  e.message # => "Could not delete tasks from the dead tasks store"
  e.cause   # => #<Errno::ENOSPC: No space left on device>
end
```

- A new public class, `Rage::Deferred::StorageError < StandardError`, in `lib/rage/deferred/deferred.rb`.
- Its message names the operation that failed. `cause` holds the store error, when there is one.

### Which backend methods raise it

| Backend method | Message | When |
| --- | --- | --- |
| `add_dead_task` | `Could not add a task to the dead tasks store` | a store error while locking, opening, repairing, writing, or syncing |
| `each_dead_task` | `Could not read tasks from the dead tasks store` | a store error while locking, opening, scanning, seeking, reading, or closing |
| `find_dead_task` | `Could not read tasks from the dead tasks store` | the same |
| `remove_dead_tasks` | `Could not delete tasks from the dead tasks store` | a store error while locking, writing the new file, syncing, renaming, or unlinking |
| `flush_tasks` | `Could not sync the pending tasks storage` | a store error from the `fsync` of the WAL file or of its directory |
| any of the four dead-task methods | the message of that method, above | the lock times out after `LOCK_MAX_ATTEMPTS` tries; `cause` is a `DeadTasksStorage::LockTimeout` with `Could not lock the dead tasks store to <operation> it` (today's message) |

- **An error raised by the block of `each_dead_task`** (the app's block in `DeadTasks#each`): it reaches the caller unchanged, never as a `StorageError`, even when it is a `SystemCallError`.
- **Decoding errors** (`ArgumentError`, `NameError`, `TypeError` from `undump` or `Marshal.load`): unchanged. Disk still prints its `ERROR: Can't deserialize ...` line and skips the record.
- **Any other error** (a bug in the backend, `Marshal.dump` failing on the context in `add`): unchanged, not wrapped.
- **Cleanup**: a failure while closing the snapshot file or releasing the lock does not replace an error already in flight. When it is the only failure, it is raised as a `StorageError` with the message of the operation.
- `remove_task`, `add_task`, and `pending_tasks` are not changed. `Task.enqueue` is released, and what it raises needs its own decision (ADR-005).

### Removed

- `Rage::Deferred::DeadTasksLockTimeout` is removed, with no alias and no deprecation. Nothing in `lib/` names it after this task.

### Dead-task errors under `DeadTask`

```ruby
begin
  task.retry
rescue Rage::Deferred::DeadTask::RetryError => e
  # the task class cannot be used, or the record could not be removed after the enqueue
rescue Rage::Deferred::DeadTask::ContextDeserializationError => e
  # the stored arguments cannot be loaded
end
```

- `Rage::Deferred::DeadTaskRetryError` becomes `Rage::Deferred::DeadTask::RetryError`, and `Rage::Deferred::DeadTaskContextDeserializationError` becomes `Rage::Deferred::DeadTask::ContextDeserializationError` (ADR-006).
- Each keeps its superclass (`StandardError`), its class text, and when it is raised, with the same messages and the same `cause`.
- The old names are removed, with no alias and no deprecation. Nothing in `lib/` names them after this task.
- The `@raise` tags of `DeadTask#retry`, `#args`, and `#kwargs` name the new classes. `CHANGELOG.md` names neither class and does not change for them.

### Public methods

- `DeadTasks#each`, `DeadTasks#find_by_id`, `DeadTasks#delete`, and `DeadTask#delete` raise `StorageError` when the store fails.
- `DeadTask#retry`:
  - **Step 1 fails** (`find_dead_task`): `StorageError` reaches the caller. Nothing is enqueued.
  - **Step 4 fails** (`enqueue`): unchanged. Whatever `Task.enqueue` raises reaches the caller, including a `SystemCallError` from the Disk backend's `add_task`. The record stays.
  - **Step 5 fails** (`flush_tasks`): `StorageError` reaches the caller. Step 7 does not run and the record stays.
  - **Step 7 fails** (`remove_dead_tasks`): unchanged, `DeadTask::RetryError` (see below); its `cause` is now the `StorageError`.
- `DeadTasks#find_by_id` and `DeadTasks#delete` keep raising `TypeError` for a bad argument, before the store is touched.

### Documentation

- `StorageError` has a class text: when it is raised, that `cause` holds the store's own error, and that it is the same class for every backend.
- `@raise` tags:
  - `DeadTask#delete`, `DeadTasks#delete`: `[Rage::Deferred::StorageError] when an operation on the dead-tasks store fails`, in place of the `DeadTasksLockTimeout` and `SystemCallError` tags.
  - `DeadTasks#each`, `DeadTasks#find_by_id`: the same tag, added.
  - `DeadTask#retry`: `[Rage::Deferred::StorageError] when the lookup or the sync of the pending tasks fails`, in place of the `DeadTasksLockTimeout` and `SystemCallError` tags. The method text says that errors raised by `enqueue` reach the caller unchanged.
- The contract header at the top of `lib/rage/deferred/backends/disk.rb` gets the rule: a backend raises `Rage::Deferred::StorageError` (or a subclass) with the store's error as `cause` when its store fails, wraps nothing else, and never wraps an error raised by the block of `each_dead_task`.
- The `@raise` tags of the Disk methods name `StorageError` in place of `DeadTasksLockTimeout` and `SystemCallError`. The tags of `DeadTasksStorage` name `DeadTasksStorage::LockTimeout` for a lock timeout, and `SystemCallError` and `IOError` for the other store errors, which `Disk` wraps.
- `CHANGELOG.md`, the existing `[Deferred]` entry under `## [Unreleased]`, names `Rage::Deferred::StorageError`. A line under `### Changed` says that `Rage::Deferred::DeadTasksLockTimeout` is removed and that a lock timeout now raises `Rage::Deferred::StorageError`.

### Not in this task

- The errors of `add_task`, and so of `Task.enqueue`.
- A subclass for a store that is busy (a lock that times out, a connection timeout).
- The I/O in `DeadTasksStorage#initialize` and `TasksStorage#initialize`, which runs at boot and is not a contract method.

## Design

### Files

| File | Change |
| --- | --- |
| `lib/rage/deferred/deferred.rb` | new `StorageError`; `DeadTasksLockTimeout`, `DeadTaskRetryError`, and `DeadTaskContextDeserializationError` removed |
| `lib/rage/deferred/backends/disk.rb` | contract header; each `Disk` contract method for the dead-tasks store, and `flush_tasks`, wraps its store errors; new internal `DeadTasksStorage::LockTimeout`, which `with_lock` raises; `@raise` tags |
| `lib/rage/deferred/dead_task.rb` | new `RetryError` and `ContextDeserializationError` inside the class; every `raise` and `@raise` tag uses them; `@raise` tags and the text of `retry` |
| `lib/rage/deferred/dead_tasks.rb` | `@raise` tags |
| `CHANGELOG.md` | the existing entry and one `### Changed` line |

`lib/rage/deferred/backends/nil.rb` does not change: it has no store that can fail.

### Interfaces

```ruby
module Rage::Deferred
  # (class text, see Documentation)
  class StorageError < StandardError
  end
end

class Rage::Deferred::Backends::Disk
  class DeadTasksStorage
    # @private
    # The dead-tasks store could not be locked after `LOCK_MAX_ATTEMPTS` tries.
    class LockTimeout < StandardError
    end
  end

  private

  # Run a block of store calls; raise a store error as `StorageError` with `cause`.
  # @param message [String] the message of the `StorageError`
  def wrap_store_errors(message)
end

class Rage::Deferred::DeadTask
  # (the class text of DeadTaskRetryError, unchanged)
  class RetryError < StandardError
  end

  # (the class text of DeadTaskContextDeserializationError, unchanged)
  class ContextDeserializationError < StandardError
  end
end
```

### Steps

The wrap sits at the backend contract boundary: in the methods of `Disk`. Each of them is a call into `DeadTasksStorage` or `TasksStorage`, so the wrap of each contract method is in one visible place, next to the contract header. `DeadTasksStorage` raises its store errors unchanged, as before this task.

- `Disk#wrap_store_errors(message)`: `yield`, with `rescue SystemCallError, IOError, DeadTasksStorage::LockTimeout => e` that raises `Rage::Deferred::StorageError, message` with `cause: e`. It rescues nothing else.
- `Disk#add_dead_task`, `Disk#find_dead_task`, `Disk#remove_dead_tasks`: the call into `@dead_tasks_storage` runs inside `wrap_store_errors` with the message of the Requirements table (`"Could not add a task to the dead tasks store"`, `"Could not read tasks from the dead tasks store"`, `"Could not delete tasks from the dead tasks store"`). This also covers a failure of `flock(File::LOCK_UN)` in the `ensure` of `with_lock`, and a failure of `storage.close` in the `ensure` of `with_snapshot`.
- `Disk#each_dead_task`: the call to `@dead_tasks_storage.each` runs inside a `rescue SystemCallError, IOError, DeadTasksStorage::LockTimeout` that raises `StorageError` (`"Could not read tasks from the dead tasks store"`) with `cause: e`, except for an error raised while the app's block runs. A local flag, set only by a `rescue` around the `yield` when the app's block raises, tells them apart: when the flag is set, the error is re-raised unchanged. When the block leaves early with `break`, `return`, or `throw`, the flag stays unset, so a cleanup error of the store is still wrapped. The flag stays set when the block raises, and `with_snapshot` keeps the block's error when `storage.close` also fails, so that error reaches the caller unchanged too.
- `Disk#flush_tasks`: `@tasks_storage.sync!` runs inside `wrap_store_errors("Could not sync the pending tasks storage")`. The `@storage_has_changes` state is still not cleared on a failure.
- `with_lock`: the timeout raises `DeadTasksStorage::LockTimeout` with today's message, in place of `DeadTasksLockTimeout`. `Disk` wraps it like the other store errors, so the `StorageError` has the message of the operation and the `LockTimeout` as `cause`.
- `DeadTasksStorage#add`, `#remove`, `#find_by_id`, `#each`, and `with_snapshot` keep the code they had before this task; only their `@raise` tags change (see Documentation).
- Remove `class DeadTasksLockTimeout` and its comment from `deferred.rb`.
- Move `DeadTaskRetryError` and `DeadTaskContextDeserializationError`, with their class texts, from `deferred.rb` to the top of the body of `class Rage::Deferred::DeadTask` in `dead_task.rb`, as `RetryError` and `ContextDeserializationError`. `deferred.rb` already requires `dead_task.rb`, so they load at the same point as before. Inside `DeadTask`, the `raise` calls name them as `RetryError` and `ContextDeserializationError`; the `@raise` tags use the full names.

### Blocking I/O

None added. The task adds `rescue` clauses around existing calls and changes the class of one `raise`. No new file, lock, `fsync`, or wait.

## Implementation constraints

- Follows ADR-005. ADR-005 replaces, for store errors only, the `@raise` line in the contract snippet of ADR-003, the "an `fsync` error reaches the caller unchanged" words in ADR-002, and the "keeps its name and meaning" constraint of the spec.
- An error raised by the app's block in `each_dead_task` is never wrapped: no `rescue` inside `DeadTasksStorage#each` covers the `yield`, and the `rescue` in `Disk#each_dead_task` re-raises an error raised while the block runs unchanged. A `SystemCallError` raised by the app's block wrapped as a `StorageError` would hide the app's own error.
- Only `SystemCallError`, `IOError`, and `DeadTasksStorage::LockTimeout` are wrapped. A bare `rescue` or `rescue StandardError` turns backend bugs and `Marshal.dump` failures into storage errors.
- The decoding `rescue ArgumentError, NameError, TypeError` blocks in `each` and `find_by_id` do not change.
- `Backends::Nil` and the backend contract's method list do not change.
- `DeadTasksLockTimeout` gets no alias, no `deprecate_constant`, and no `const_missing` hook (ADR-005).
- Follows ADR-006. `DeadTaskRetryError` and `DeadTaskContextDeserializationError` get no alias, no `deprecate_constant`, and no `const_missing` hook.
- No change to `add_task`, `remove_task`, `pending_tasks`, or `Queue`. The WAL is out of scope: `TasksStorage` does not change, and the `rescue` for `flush_tasks` sits in `Disk#flush_tasks`, which only `DeadTask#retry` calls.

## Acceptance criteria

- [ ] TC-01 (core, AC-27) If the dead-tasks store cannot be locked, `dead_tasks.delete` raises `Rage::Deferred::StorageError`.
- [ ] TC-02 (core, AC-27) If a file operation fails while `dead_tasks.delete` rewrites the store, it raises `Rage::Deferred::StorageError` whose `cause` is that `SystemCallError`.
- [ ] TC-03 (core, AC-27) If a file operation fails while `each` reads the store, `each` raises `Rage::Deferred::StorageError` whose `cause` is that `SystemCallError`.
- [ ] TC-04 (core, AC-27) If a file operation fails while `find_by_id` reads the store, it raises `Rage::Deferred::StorageError` whose `cause` is that `SystemCallError`.
- [ ] TC-05 (core) If a file operation fails while a dead task is added, `add_dead_task` raises `Rage::Deferred::StorageError` whose `cause` is that `SystemCallError`.
- [ ] TC-06 (core) If the `fsync` of the WAL fails, `flush_tasks` raises `Rage::Deferred::StorageError` whose `cause` is that `SystemCallError`.
- [ ] TC-07 (edge) When the block of `each` raises a `SystemCallError`, the same error reaches the caller, not a `StorageError`.
- [ ] TC-08 (edge) If the snapshot file cannot be closed and no other error is in flight, `each` raises `Rage::Deferred::StorageError`.
- [ ] TC-09 (edge) If the block of `each` raises and the snapshot file then cannot be closed, the error of the block reaches the caller.
- [ ] TC-10 (edge) If `flush_tasks` fails in `retry`, `retry` raises `Rage::Deferred::StorageError` and the record stays in the dead-tasks store.
- [ ] TC-11 (edge) If the removal in `retry` fails with a file error, `retry` raises `Rage::Deferred::DeadTask::RetryError` whose `cause` is a `Rage::Deferred::StorageError`.
- [ ] TC-12 (edge) When the code is loaded, `Rage::Deferred.const_defined?(:DeadTasksLockTimeout)` is `false`.
- [ ] TC-13 (core) If `retry` cannot use the stored task class, it raises `Rage::Deferred::DeadTask::RetryError`.
- [ ] TC-14 (core) If the stored context cannot be decoded, `args` raises `Rage::Deferred::DeadTask::ContextDeserializationError`.
- [ ] TC-15 (edge) When the code is loaded, neither `Rage::Deferred::DeadTaskRetryError` nor `Rage::Deferred::DeadTaskContextDeserializationError` is defined.

## Verification

Tests to add or change:

- `spec/deferred/backends/disk_spec.rb` — TC-05, TC-06, and TC-07 to TC-09 at the backend level. The existing examples that expect `DeadTasksLockTimeout` or a raw `Errno::*` from `add_dead_task`, `each_dead_task`, `find_dead_task`, `remove_dead_tasks`, and `flush_tasks` change to expect `StorageError`, with the original error as `cause` where they check identity today. The examples for an error raised by the caller's block keep expecting that error.
- `spec/deferred/dead_task_deletion_spec.rb` — TC-01 and TC-02 through the public API over a real Disk store; the examples that expect `DeadTasksLockTimeout` or `Errno::*` from `delete` change.
- `spec/deferred/dead_task_retry_spec.rb` — TC-10 and TC-11; the example that builds a `DeadTasksLockTimeout` for the existence check builds a `StorageError` instead. The example in which `add_task` raises `Errno::ENOSPC` keeps expecting that error unchanged (step 4).
- `spec/deferred/dead_tasks_spec.rb` — TC-03 and TC-04 through the public API over a real Disk store, and TC-07 through `DeadTasks#each`. TC-07 is also covered in `disk_spec.rb` at the backend level.
- `spec/deferred/deferred_spec.rb` — TC-12 and TC-15.
- `spec/deferred/dead_task_retry_spec.rb` and `spec/deferred/dead_tasks_spec.rb` — TC-13 and TC-14: the existing examples that expect `DeadTaskRetryError` or `DeadTaskContextDeserializationError` change to the new names.

Commands:

- `bundle exec rspec spec/deferred/backends/disk_spec.rb spec/deferred/dead_task_deletion_spec.rb spec/deferred/dead_task_retry_spec.rb spec/deferred/dead_tasks_spec.rb spec/deferred/deferred_spec.rb`
- `bundle exec rubocop lib/rage/deferred/deferred.rb lib/rage/deferred/backends/disk.rb lib/rage/deferred/dead_task.rb lib/rage/deferred/dead_tasks.rb`
- `bundle exec yardoc --fail-on-warning`
- `rg -n DeadTasksLockTimeout lib` returns nothing; in `spec`, only the TC-12 example names it.
- `rg -n 'DeadTaskRetryError|DeadTaskContextDeserializationError' lib` returns nothing; in `spec`, only the TC-15 example names them.

**Core tests**

- Changed to the new behavior: `spec/deferred/dead_task_deletion_spec.rb` "propagates a rename failure without deleting or disclosing the payload" is now "raises StorageError for a rename failure without deleting or disclosing the payload" (it rescues `StorageError` and checks its message and `cause`; the checks for no payload in the message and for the kept record stay).
- Changed to the new behavior: `spec/deferred/backends/disk_spec.rb` "propagates WAL synchronization failure without clearing dirty state" is now "raises StorageError for a WAL synchronization failure without clearing dirty state" (it expects `StorageError` with the message and that failure as `cause`; the dirty-state check stays).
- The task starts after `54dfc4c` (the engineer's choice): the commits from `origin/main` up to it are task 03.

<!-- rage-flow:core-tests begin — written by a script, do not edit -->
- TC-01: spec/deferred/dead_task_deletion_spec.rb — "Rage::Deferred::DeadTasks Disk deletion raises StorageError from collection deletion when another open file description holds the lock" — red: assertion
- TC-02: spec/deferred/dead_task_deletion_spec.rb — "Rage::Deferred::DeadTasks Disk deletion raises StorageError for a rename failure without deleting or disclosing the payload" — red: assertion
- TC-03: spec/deferred/dead_tasks_spec.rb — "Rage::Deferred::DeadTasks Disk storage errors raises StorageError from each when opening the store fails, with that error as cause" — red: assertion
- TC-04: spec/deferred/dead_tasks_spec.rb — "Rage::Deferred::DeadTasks Disk storage errors raises StorageError from find_by_id when opening the store fails, with that error as cause" — red: assertion
- TC-05: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #add_dead_task raises StorageError when the append cannot be synced, with that error as cause" — red: assertion
- TC-06: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk#flush_tasks raises StorageError for a WAL synchronization failure without clearing dirty state" — red: assertion
<!-- rage-flow:core-tests end -->

**Edge tests**

- New: `disk_spec.rb` "lets a SystemCallError raised by the block reach the caller unchanged" (TC-07) and "raises StorageError with a lock timeout cause, not the unrelated error, inside an unrelated rescue" (an edge case of TC-01; named "raises a lock timeout with no cause inside an unrelated rescue" before the second revision below); `dead_task_retry_spec.rb` "raises DeadTaskRetryError with a StorageError cause when the Disk removal fails" (TC-11); `deferred_spec.rb` "does not define DeadTasksLockTimeout" (TC-12). TC-09 is covered by the existing "does not let cleanup replace an active user exception".
- Changed to the new behavior, with the engineer's confirmation: 26 existing examples that expected `DeadTasksLockTimeout` or a raw `Errno::*` now expect `StorageError`, with the original error as `cause` where they checked it. In `disk_spec.rb`: the `#flush_tasks` directory example, now "raises StorageError for a directory synchronization failure without clearing dirty state"; 2 `#add_dead_task` lock examples; 8 `#each_dead_task` failure examples; 7 `#find_dead_task` failure examples, of which "propagates snapshot lock timeouts unchanged" and "propagates open failures unchanged" now end in "as StorageError"; 2 `#remove_dead_tasks` examples. In `dead_task_deletion_spec.rb`: the entry lock timeout, the temp-file failure (now "raises StorageError for a temp-file failure without deleting or disclosing the payload"), and the directory sync failure. In `dead_task_retry_spec.rb`: the two "keeps the dead record when … fsync fails during Disk flush_tasks" examples, and "propagates an existence-check lock timeout unchanged", whose fake backend now raises `StorageError`. This last example is not in the map: it checks that `retry` passes on a fake backend's error, which no criterion names.
- Ran: `disk_spec.rb`, `dead_task_deletion_spec.rb`, `dead_task_retry_spec.rb`, `dead_tasks_spec.rb`, `deferred_spec.rb` on Ruby 3.3.0 (the Ruby on `PATH`) — 216 examples, 0 failures, none pending or skipped. No example in these files needs external services. Other Rubies were not run.
- 26 mapped examples are "not proven" at the base: `Rage::Deferred::StorageError` does not exist there, so they stop with `NameError` before their assertion.
- Review round 1: new `disk_spec.rb` "raises StorageError when a record cannot be read after the forward scan" (TC-03, F2): the read of one record fails after the forward scan, and `each_dead_task` raises `StorageError` with that failure as `cause` and closes the snapshot. It makes `File#seek` raise only for a `SEEK_SET` to a positive offset, which only the read of a record does. A run by hand with the `wrap_store_errors` around that read removed made it fail (expected `StorageError`, got `Errno::EIO`); the revert check cannot show this, because the class does not exist at the base. New `dead_tasks_spec.rb` "Disk block errors" "lets a SystemCallError raised by the block of each reach the caller unchanged" (TC-07, F5), over a real Disk store; it passes without the change because the behavior is kept on purpose. Both ran on Ruby 3.3.0, and passed.
Review findings handled: F2, F5
Review mode changed: spec/deferred/backends/disk_spec.rb, spec/deferred/dead_tasks_spec.rb; code changed: no
- Review round 2: the engineer did not accept the by-hand check for F2. The F2 example now checks the class of the error by its name (`"Rage::Deferred::StorageError"`) and its message and `cause` inside the `raise_error` block, instead of naming the constant, so that without the change it fails on an assertion (it gets `Errno::EIO`) and not on a `NameError`. The revert check now shows `revert: fails` for it. The stub is unchanged: only the read of one record after the forward scan fails. It ran on Ruby 3.3.0 with the rest of `disk_spec.rb` (106 examples, 0 failures).
Review findings handled: F2
Review mode changed: spec/deferred/backends/disk_spec.rb; code changed: no
- Revision on 2026-10-20, at the engineer's request: the code now follows the Design. Before, `wrap_store_errors` was a method of `DeadTasksStorage` and wrapped the calls inside `add`, `each`, `find_by_id`, `remove`, and `with_snapshot`. Now `Disk#wrap_store_errors(message)` wraps `add_dead_task`, `find_dead_task`, `remove_dead_tasks`, and `flush_tasks`, and `Disk#each_dead_task` uses the flag around the block. `DeadTasksStorage` has its code from before this task again; only its `@raise` tags and the `with_lock` timeout changed. No test changed. All 218 examples of the five test files ran on Ruby 3.3.0 and passed, among them every core and edge example in the map; none was skipped or pending. The block below was not run again: it compares the tests with the code at the base, and the tests did not change.
- Second revision on 2026-10-20, at the engineer's request: the lock timeout now follows the Requirements and the Design. Before, `with_lock` raised `Rage::Deferred::StorageError` with the lock message ("Could not lock the dead tasks store to <operation> it") and no `cause`, and `DeadTasksStorage::LockTimeout` did not exist. Now `with_lock` raises the new `DeadTasksStorage::LockTimeout` with that message, and `Disk#wrap_store_errors` and the `rescue` of `Disk#each_dead_task` wrap it, so the `StorageError` has the message of the operation and the `LockTimeout` as `cause`. The `@raise` tags of `DeadTasksStorage` name `LockTimeout`. Changed to the new behavior, with the engineer's confirmation: `disk_spec.rb` "raises a lock timeout with no cause inside an unrelated rescue" is now "raises StorageError with a lock timeout cause, not the unrelated error, inside an unrelated rescue" (it expects the message of the operation, a `LockTimeout` cause with the lock message, and no cause on the `LockTimeout`); in `disk_spec.rb` "raises when another open file description holds the lock" (`#add_dead_task`), "propagates snapshot lock timeouts on first advancement", "propagates snapshot lock timeouts as StorageError", and "raises instead of reporting zero when the store is already locked", and in `dead_task_deletion_spec.rb` "propagates a lock timeout from entry deletion without reporting false", the message check changed from a pattern that both messages match to the exact message of the operation, with a `LockTimeout` cause. The core test of TC-01 did not change: its pattern `/delete tasks from/` matches the new message too. Ran: all 218 examples of the five test files on Ruby 3.3.0 (the Ruby on `PATH`), 0 failures, none pending or skipped; before the examples changed, only the cause-nil example failed. Other Rubies were not run. The block below was run again against base `54dfc4c`; its results did not change.
- Review round 3 (F7): new `disk_spec.rb` "wraps a descriptor cleanup error after the block leaves early" (TC-08): the block of `each_dead_task` leaves with `break`, the snapshot close raises `Errno::EIO`, and the example expects `StorageError` with that error as `cause`, checking the class by its name so that without the change it fails on an assertion. Changed, with the engineer's confirmation: `dead_task_retry_spec.rb` "keeps the dead record when #{operation} fsync fails during Disk flush_tasks" (the body shared by the wal and directory examples, TC-10) now checks the class by its name, then the message and `cause`; the checks for the kept record and the pending task did not change. The revert check, against base `54dfc4c`, shows `revert: fails` for the new TC-08 example and both TC-10 examples. The other three TC-08 examples stay `not proven (NameError)`. Ran: `disk_spec.rb` and `dead_task_retry_spec.rb` on Ruby 3.3.0, 163 examples, 0 failures, none pending or skipped. Other Rubies were not run.
Review findings handled: F7
Review mode changed: spec/deferred/backends/disk_spec.rb, spec/deferred/dead_task_retry_spec.rb; code changed: no

<!-- rage-flow:edge-tests begin — written by a script, do not edit -->
- TC-01: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #add_dead_task raises when another open file description holds the lock" — revert: not proven (NameError)
- TC-01: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #add_dead_task caps lock-retry backoff" — revert: not proven (NameError)
- TC-01: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task propagates snapshot lock timeouts on first advancement" — revert: not proven (NameError)
- TC-01: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task propagates snapshot lock timeouts as StorageError" — revert: not proven (NameError)
- TC-01: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #remove_dead_tasks raises instead of reporting zero when the store is already locked" — revert: not proven (NameError)
- TC-01: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #remove_dead_tasks raises StorageError with a lock timeout cause, not the unrelated error, inside an unrelated rescue" — revert: not proven (NameError)
- TC-01: spec/deferred/dead_task_deletion_spec.rb — "Rage::Deferred::DeadTasks Disk deletion propagates a lock timeout from entry deletion without reporting false" — revert: not proven (NameError)
- TC-02: spec/deferred/dead_task_deletion_spec.rb — "Rage::Deferred::DeadTasks Disk deletion raises StorageError for a temp-file failure without deleting or disclosing the payload" — revert: not proven (NameError)
- TC-02: spec/deferred/dead_task_deletion_spec.rb — "Rage::Deferred::DeadTasks Disk deletion raises when directory synchronization fails after replacement" — revert: not proven (NameError)
- TC-02: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #remove_dead_tasks releases the lock when the rewrite raises" — revert: not proven (NameError)
- TC-03: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task propagates an open failure from first advancement" — revert: not proven (NameError)
- TC-03: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task propagates lazy snapshot-acquisition failures and closes the opened descriptor" — revert: not proven (NameError)
- TC-03: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task propagates forward-scan seek failures and closes the snapshot" — revert: not proven (NameError)
- TC-03: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task propagates forward-scan read failures and closes the snapshot" — revert: not proven (NameError)
- TC-03: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task propagates a lock cleanup error inside an unrelated rescue" — revert: not proven (NameError)
- TC-03: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task raises StorageError when a record cannot be read after the forward scan" — revert: fails
- TC-04: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task propagates open failures as StorageError" — revert: not proven (NameError)
- TC-04: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task propagates forward-scan seek failures and closes the snapshot" — revert: not proven (NameError)
- TC-04: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task propagates forward-scan read failures and closes the snapshot" — revert: not proven (NameError)
- TC-04: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task propagates an operational failure after corruption and a provisional match" — revert: not proven (NameError)
- TC-04: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task does not let cleanup replace an active lookup failure" — revert: not proven (NameError)
- TC-06: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk#flush_tasks raises StorageError for a directory synchronization failure without clearing dirty state" — revert: not proven (NameError)
- TC-07: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task lets a SystemCallError raised by the block reach the caller unchanged" — revert: passes without the change — the error of the block already reached the caller unchanged before this task; TC-07 requires that this stays so
- TC-07: spec/deferred/dead_tasks_spec.rb — "Rage::Deferred::DeadTasks Disk block errors lets a SystemCallError raised by the block of each reach the caller unchanged" — revert: passes without the change — the error of the block already reached the caller of DeadTasks#each unchanged before this task; TC-07 requires that this stays so
- TC-08: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task propagates a descriptor cleanup error when it is the only failure" — revert: not proven (NameError)
- TC-08: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task propagates a descriptor cleanup error inside an unrelated rescue" — revert: not proven (NameError)
- TC-08: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task propagates a descriptor cleanup error when it is the only failure" — revert: not proven (NameError)
- TC-08: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task wraps a descriptor cleanup error after the block leaves early" — revert: fails
- TC-09: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task does not let cleanup replace an active user exception" — revert: passes without the change — a close failure already did not replace the error of the block before this task; TC-09 requires that this stays so
- TC-10: spec/deferred/dead_task_retry_spec.rb — "Rage::Deferred::DeadTask retry keeps the dead record when wal fsync fails during Disk flush_tasks" — revert: fails
- TC-10: spec/deferred/dead_task_retry_spec.rb — "Rage::Deferred::DeadTask retry keeps the dead record when directory fsync fails during Disk flush_tasks" — revert: fails
- TC-11: spec/deferred/dead_task_retry_spec.rb — "Rage::Deferred::DeadTask retry raises DeadTaskRetryError with a StorageError cause when the Disk removal fails" — revert: fails
- TC-12: spec/deferred/deferred_spec.rb — "Rage::Deferred error classes does not define DeadTasksLockTimeout" — revert: fails
<!-- rage-flow:edge-tests end -->

**Review outcome**

- Light re-review on 2026-10-20 of tree b53024f, after round 3: Reviewer B only, on `spec/deferred/backends/disk_spec.rb` and `spec/deferred/dead_task_retry_spec.rb` (the files task:build changed for F7; code changed: no). No new findings.
- Full review, round 3, on 2026-10-20 of tree 83b56f4 (HEAD 54dfc4c), base 54dfc4c (the engineer's choice; the merge base with `origin/main` also holds tasks 01–03). It ran because the second build revision changed code. Reviewers A (runtime), B (contract, surface, and tests), and C (Design match) ran; none was skipped. C had no findings.
- Earlier rounds on the same day: a full review of tree 1cd2f2d, and two light re-reviews (trees 00673dd and 3769575, Reviewer B only, no new findings).
- Report: `rage-feature-specs/.workflow/evidence/deferred-dead-tasks-api/07-storage-errors-review.md` (a local file). Finding numbers continue across rounds.
- Applied in round 3: F6 (after `break`, `return`, or `throw` out of the block of `each`, a close error of the snapshot is now wrapped as `StorageError`; the flag in `Disk#each_dead_task` is set only when the block raises, and the Design's Steps text says so), F8 (the class text of `StorageError` names the operations that raise it and says that `enqueue` errors are not wrapped). Applied in round 1: F3b, F4. Rejected: none.
- Decided: F7 — proven by `disk_spec.rb` "wraps a descriptor cleanup error after the block leaves early" (TC-08: the block leaves with `break` and the close error comes out as `StorageError`) and the two `dead_task_retry_spec.rb` "keeps the dead record when … fsync fails during Disk flush_tasks" examples (TC-10); all three check the class by name and show `revert: fails`.
- Decided: F1 — kept out of this task, as recommended: `next`/`peek` on the Enumerator that `DeadTasks#each` returns without a block can lose a `StorageError` or a record in a running server when the scheduler parks the enumerator fiber. It existed before this task and goes to a separate `bug` feature that first reproduces it.
- Decided: F2 — proven by `disk_spec.rb` "raises StorageError when a record cannot be read after the forward scan" (TC-03): it checks the error class by name, and the revert check shows it failing without the change (`revert: fails`).
- Decided: F3 — rejected by the engineer: `DeadTask#retry` gets no `@raise [SystemCallError]` tag back; the tags stay as the Requirements give them, and the method text says that errors of `enqueue` reach the caller unchanged.
- Decided: F5 — proven by `dead_tasks_spec.rb` "Disk block errors" "lets a SystemCallError raised by the block of each reach the caller unchanged" (TC-07 through `DeadTasks#each`; it passes without the change because the behavior is kept).
- Unmet: TC-08 (fix applied, F6)
- Not run: Ruby 3.4 and 4.x are not installed here. `bundle exec yardoc --fail-on-warning` exits 1 on one warning, `Undocumentable Struct assignment to Rage::SSE::Message`, from `lib/rage/sse/message.rb`, which this task does not change. The plain-words check `rg -n DeadTasksLockTimeout lib` was run by hand and finds nothing.
- All 39 mapped examples ran and passed on Ruby 3.3.0; none is excluded by default or needs external services. The script names `disk_spec.rb`, `dead_task_deletion_spec.rb`, and `dead_tasks_spec.rb` as core test files changed since the red run: the changes are to edge examples in those files (the build revisions), not to the core examples.

<!-- rage-flow:final-run begin — written by a script, do not edit -->
Ran: bundle exec rspec spec/deferred/backends/disk_spec.rb spec/deferred/dead_task_deletion_spec.rb spec/deferred/dead_task_retry_spec.rb spec/deferred/dead_tasks_spec.rb spec/deferred/deferred_spec.rb (Ruby 3.3.0) — exit 0
Ran: bundle exec rspec spec/deferred/backends/disk_spec.rb spec/deferred/dead_task_deletion_spec.rb spec/deferred/dead_task_retry_spec.rb spec/deferred/dead_tasks_spec.rb spec/deferred/deferred_spec.rb (Ruby 3.3.0) — exit 0
Ran: bundle exec rubocop lib/rage/deferred/deferred.rb lib/rage/deferred/backends/disk.rb lib/rage/deferred/dead_task.rb lib/rage/deferred/dead_tasks.rb — exit 0
Ran: bundle exec yardoc --fail-on-warning — exit 1
Not tested: Ruby 3.4.x — not installed in ~/.rubies, and not the Ruby on PATH
Not tested: Ruby 4.x — not installed in ~/.rubies, and not the Ruby on PATH
- TC-01: spec/deferred/dead_task_deletion_spec.rb — "Rage::Deferred::DeadTasks Disk deletion raises StorageError from collection deletion when another open file description holds the lock" — passed
- TC-01: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #add_dead_task raises when another open file description holds the lock" — passed
- TC-01: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #add_dead_task caps lock-retry backoff" — passed
- TC-01: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task propagates snapshot lock timeouts on first advancement" — passed
- TC-01: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task propagates snapshot lock timeouts as StorageError" — passed
- TC-01: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #remove_dead_tasks raises instead of reporting zero when the store is already locked" — passed
- TC-01: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #remove_dead_tasks raises StorageError with a lock timeout cause, not the unrelated error, inside an unrelated rescue" — passed
- TC-01: spec/deferred/dead_task_deletion_spec.rb — "Rage::Deferred::DeadTasks Disk deletion propagates a lock timeout from entry deletion without reporting false" — passed
- TC-02: spec/deferred/dead_task_deletion_spec.rb — "Rage::Deferred::DeadTasks Disk deletion raises StorageError for a rename failure without deleting or disclosing the payload" — passed
- TC-02: spec/deferred/dead_task_deletion_spec.rb — "Rage::Deferred::DeadTasks Disk deletion raises StorageError for a temp-file failure without deleting or disclosing the payload" — passed
- TC-02: spec/deferred/dead_task_deletion_spec.rb — "Rage::Deferred::DeadTasks Disk deletion raises when directory synchronization fails after replacement" — passed
- TC-02: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #remove_dead_tasks releases the lock when the rewrite raises" — passed
- TC-03: spec/deferred/dead_tasks_spec.rb — "Rage::Deferred::DeadTasks Disk storage errors raises StorageError from each when opening the store fails, with that error as cause" — passed
- TC-03: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task propagates an open failure from first advancement" — passed
- TC-03: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task propagates lazy snapshot-acquisition failures and closes the opened descriptor" — passed
- TC-03: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task propagates forward-scan seek failures and closes the snapshot" — passed
- TC-03: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task propagates forward-scan read failures and closes the snapshot" — passed
- TC-03: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task propagates a lock cleanup error inside an unrelated rescue" — passed
- TC-03: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task raises StorageError when a record cannot be read after the forward scan" — passed
- TC-04: spec/deferred/dead_tasks_spec.rb — "Rage::Deferred::DeadTasks Disk storage errors raises StorageError from find_by_id when opening the store fails, with that error as cause" — passed
- TC-04: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task propagates open failures as StorageError" — passed
- TC-04: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task propagates forward-scan seek failures and closes the snapshot" — passed
- TC-04: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task propagates forward-scan read failures and closes the snapshot" — passed
- TC-04: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task propagates an operational failure after corruption and a provisional match" — passed
- TC-04: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task does not let cleanup replace an active lookup failure" — passed
- TC-05: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #add_dead_task raises StorageError when the append cannot be synced, with that error as cause" — passed
- TC-06: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk#flush_tasks raises StorageError for a WAL synchronization failure without clearing dirty state" — passed
- TC-06: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk#flush_tasks raises StorageError for a directory synchronization failure without clearing dirty state" — passed
- TC-07: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task lets a SystemCallError raised by the block reach the caller unchanged" — passed
- TC-07: spec/deferred/dead_tasks_spec.rb — "Rage::Deferred::DeadTasks Disk block errors lets a SystemCallError raised by the block of each reach the caller unchanged" — passed
- TC-08: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task propagates a descriptor cleanup error when it is the only failure" — passed
- TC-08: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task propagates a descriptor cleanup error inside an unrelated rescue" — passed
- TC-08: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task propagates a descriptor cleanup error when it is the only failure" — passed
- TC-08: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task wraps a descriptor cleanup error after the block leaves early" — passed
- TC-09: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task does not let cleanup replace an active user exception" — passed
- TC-10: spec/deferred/dead_task_retry_spec.rb — "Rage::Deferred::DeadTask retry keeps the dead record when wal fsync fails during Disk flush_tasks" — passed
- TC-10: spec/deferred/dead_task_retry_spec.rb — "Rage::Deferred::DeadTask retry keeps the dead record when directory fsync fails during Disk flush_tasks" — passed
- TC-11: spec/deferred/dead_task_retry_spec.rb — "Rage::Deferred::DeadTask retry raises DeadTaskRetryError with a StorageError cause when the Disk removal fails" — passed
- TC-12: spec/deferred/deferred_spec.rb — "Rage::Deferred error classes does not define DeadTasksLockTimeout" — passed
Open: none
Tested: HEAD 54dfc4c, files digest f09b81d (CHANGELOG.md, lib/rage/deferred/backends/disk.rb, lib/rage/deferred/dead_task.rb, lib/rage/deferred/dead_tasks.rb, lib/rage/deferred/deferred.rb, spec/deferred/backends/disk_spec.rb, spec/deferred/dead_task_deletion_spec.rb, spec/deferred/dead_task_retry_spec.rb, spec/deferred/dead_tasks_spec.rb, spec/deferred/deferred_spec.rb)
<!-- rage-flow:final-run end -->

## Result

- Implementation PR:
