---
status: rejected
---

# ADR-005: Backend-neutral storage errors

> **Rejected on 2026-10-26.** After a discussion with the maintainer, this error handling is skipped. Store errors keep propagating unchanged (Option A), and `Rage::Deferred::DeadTasksLockTimeout` stays. The commit that implemented it ([task 07](../tasks/07-storage-errors.md)) was dropped from the branch.

## Context

The public methods of this feature document the errors that the Disk backend raises:

- `DeadTask#delete`, `DeadTask#retry`, and `DeadTasks#delete` list `Rage::Deferred::DeadTasksLockTimeout` and `SystemCallError` in their `@raise` tags.
- ADR-003 puts `@raise [Rage::Deferred::DeadTasksLockTimeout]` into the contract of `each_dead_task`.
- ADR-002 says that an `fsync` error in step 5 of `retry` reaches the caller unchanged.
- [deferred-dead-letter-queue ADR-001](../../deferred-dead-letter-queue/adr/001-public-dead-task-object-model.md) and REQ-26 of [its spec](../../deferred-dead-letter-queue/spec.md) say that lock and filesystem failures "propagate unchanged", and that Rage does not translate them into a generic error.

Both errors belong to the file store. A lock that times out after a number of retries is how the Disk backend shares one file between workers, and `Errno::*` is what file operations raise. A Redis backend would raise `Redis::TimeoutError` or `Redis::CannotConnectError`, and a SQL backend would raise its driver's errors. An app that writes `rescue SystemCallError` around `dead_tasks.delete` today stops catching storage failures after the backend changes, and nothing tells it so. The public API would then promise one store's errors to every later backend, the same problem ADR-001 avoids for the storage record.

Three facts limit the solution:

- `Rage::Deferred::DeadTasksLockTimeout` was released in 1.28.0 with the dead-tasks store. In 1.28 only `Queue#abandon_task` can raise it, inside the fiber that processes a task, so no public method raises it to app code. The constant itself is public, and the spec keeps its name and meaning.
- `DeadTasks#each` runs the app's block inside `each_dead_task`, and `DeadTask#retry` calls the task class's `enqueue`, which runs the app's enqueue middleware. A `rescue` around either call also catches the app's own errors.
- The errors that carry meaning for the app stay: `DeadTaskRetryError` (the task may now be in both stores), `DeadTaskContextDeserializationError` (the stored arguments cannot be loaded), and `PushTimeout` (queue policy). They mean the same thing for every backend.

## Options considered

### Option A — Errors propagate unchanged (current)

- Benefits: no new class; the app sees the exact error.
- Drawbacks: the public `@raise` tags name one store's errors, and they change when the backend changes.

Lost: it makes the Disk backend's errors part of the public API.

### Option B — One error class per operation

`TaskAddFailed`, `DeadTaskWriteFailed`, `DeadTaskRemoveFailed`, and so on, each with the original error as `cause`.

- Benefits: the class names the operation that failed.
- Drawbacks: the caller already knows the operation, because it called the method. `rescue DeadTaskRemoveFailed` around `dead_tasks.delete` tells it nothing more than one shared class does. Each class is public surface that every later backend must raise correctly, for each operation.

Lost: more public classes with no different reaction for the app.

### Option C — The public classes wrap every error from the backend

`DeadTasks` and `DeadTask` rescue errors around their backend calls and raise one shared class.

- Benefits: a later backend gets the wrapping without any work.
- Drawbacks: `DeadTasks#each` would wrap the errors of the app's own block, and `retry` would wrap the errors of the enqueue middleware. Bugs in a backend (`NoMethodError`) would look like storage failures. The public classes cannot tell a storage failure from any other error; only the backend knows which of its errors come from the store.

Lost: it hides the app's own errors and backend bugs behind a storage error.

### Option D — The backend raises one `Rage::Deferred::StorageError` (selected)

A new public class, `Rage::Deferred::StorageError < StandardError`. The backend contract says that a backend raises it, or a subclass of it, when its store fails, with the store's own error as `cause`. The public methods document only `StorageError`.

- Benefits: one class for the app to rescue, the same for every backend. The details stay in `cause`, which error reporters and Ruby's error output show. Each backend decides which of its errors are storage failures, because only it knows. No cost until something is raised.
- Drawbacks: the backend contract changes its meaning, so every later backend must wrap its store's errors. The app cannot tell "busy, try again later" from "the store is broken" by class alone.

A subclass for "busy, try again later" (a lock that times out, a connection timeout) is the only split that would change what the app does. It is not added now; it can be added later as a subclass without a break.

### `DeadTasksLockTimeout`: keep it as a subclass of `StorageError`

`DeadTasksLockTimeout < StorageError`. The Disk backend keeps raising it when the lock times out. The public `@raise` tags name only `StorageError`.

- Benefits: no break. Code written for 1.28 that rescues `DeadTasksLockTimeout` still works, and code that rescues `StorageError` also catches it.
- Drawbacks: a constant that names one store's mechanism stays in the public namespace for good, and the app can still write code that depends on it.

Lost: the engineer chose to remove the Disk-specific name from the public API.

### `DeadTasksLockTimeout`: raise `StorageError` and remove the class (selected)

The Disk backend raises `StorageError` when the lock times out. `DeadTasksLockTimeout` is removed in the release that ships this feature, with no alias and no deprecation period.

- Benefits: no Disk-specific name stays in the public API, and there is no shim to keep and remove later.
- Drawbacks: code that names `DeadTasksLockTimeout` raises `NameError` when Ruby evaluates the name, for example when an error reaches `rescue DeadTasksLockTimeout`.

The engineer judged the removal safe to release without a deprecation: in 1.28 no public method raises `DeadTasksLockTimeout` to app code (see Context), so an app has had no place to rescue it.

## Decision

Option D, and `DeadTasksLockTimeout` is removed.

- `Rage::Deferred::StorageError < StandardError` is a new public class, defined with the other errors in `lib/rage/deferred/deferred.rb`. Its message names the operation that failed; `cause` holds the store's own error.
- When the Disk backend cannot lock the dead-tasks store after all retries, it raises `StorageError` with the message of the operation, like any other failure of the store. Its `cause` is an internal error of the dead-tasks store, `Rage::Deferred::Backends::Disk::DeadTasksStorage::LockTimeout` (`# @private`), with the message it uses today. Apps rescue `StorageError`, never the internal class.
- `Rage::Deferred::DeadTasksLockTimeout` is removed in the release that ships this feature, with no alias and no deprecation.
- These backend methods raise `StorageError` for a failure of the store: `add_dead_task`, `each_dead_task`, `find_dead_task`, `remove_dead_tasks`, and `flush_tasks`. In the Disk backend a failure of the store is `SystemCallError` and `IOError`, plus a lock that times out.
- A backend wraps only the errors of its store. Other errors propagate unchanged: a bug in the backend, an error from the app's block, and the decoding errors that the Disk backend already logs and skips (`ArgumentError`, `NameError`, `TypeError`).
- A backend never wraps an error raised by the block of `each_dead_task`. The block is the app's code; the `rescue` covers the store's calls, not the `yield`. The Disk backend already yields outside its `rescue` for decoding errors, for the same reason.
- The rules for cleanup stay: a cleanup failure does not replace an error already in flight, and when cleanup is the only failure, it is raised, as a `StorageError`.
- The header of `lib/rage/deferred/backends/disk.rb`, where the contract is written, gets this rule. `Backends::Nil` has no store that can fail, so it meets the contract without a change.
- The `@raise` tags of `DeadTask#delete`, `DeadTask#retry`, and `DeadTasks#delete` name `Rage::Deferred::StorageError` in place of `DeadTasksLockTimeout` and `SystemCallError`. `DeadTasks#each` and `DeadTasks#find_by_id` get the same tag.
- `DeadTask#retry` step 7 (ADR-002) is unchanged: a failed removal raises `DeadTaskRetryError`, and its `cause` is now the `StorageError`.
- `add_task`, and so the released `Task.enqueue`, are not changed. Wrapping the errors of `enqueue` changes what a released method raises, and needs its own decision.

This replaces, for errors of the store only:

- the `@raise` line in the contract snippet of ADR-003, which becomes `@raise [Rage::Deferred::StorageError] if the dead-tasks store fails`;
- in ADR-002, the words "an `fsync` error" in "An error raised in step 4 or 5 (`PushTimeout`, an enqueue middleware error, an `fsync` error) reaches the caller unchanged": a failure in step 5 now reaches the caller as a `StorageError`, and the record still stays;
- in deferred-dead-letter-queue ADR-001 and REQ-26, the rule that lock and filesystem failures propagate unchanged and are not translated into a generic error;
- in the spec, the constraint that `DeadTasksLockTimeout` "keeps its name and meaning";
- every other mention of `DeadTasksLockTimeout` as the error a lock timeout raises, in ADR-002, ADR-003, the spec, and deferred-dead-letter-queue ADR-001: a lock timeout now raises `StorageError`. The behavior they describe (when the lock times out, and what state it leaves) does not change.

## Consequences

- An app rescues one class, `Rage::Deferred::StorageError`, for storage failures of the dead-tasks API, whatever the backend. The store's error is in `cause`.
- Every later backend must wrap the errors of its store in `StorageError`, and must not wrap the errors of the app's block. A backend that forgets lets its native errors reach the app; nothing checks this automatically.
- `DeadTask#retry` can still raise whatever `Task.enqueue` raises in step 4, including `SystemCallError` from the Disk backend's `add_task`, until the errors of `enqueue` are decided separately. The YARD text of `retry` says that errors from `enqueue` reach the caller unchanged.
- A constant released in 1.28.0 is removed without a deprecation. Code that names it raises `NameError`.
- The app cannot tell a busy store from a broken one by class, on any backend. A backend-neutral "busy" subclass can be added later without a break.
- `Queue#abandon_task` sees a `StorageError` instead of a raw `Errno::*` when the dead-tasks store fails. Nothing there rescues it today, so the behavior is the same.
- One more public class, and the `@raise` tags and the `[Deferred]` changelog entry name it. No new blocking call and no cost on the happy path: a `rescue` costs nothing until an error is raised.
