---
status: ready-for-development
---

# Delete and replay one dead task

## Goal

Add `Rage::Deferred::DeadTask#delete` and `Rage::Deferred::DeadTask#retry`, so that an app can remove a dead task or run it again through the public API, and `Rage::Deferred::DeadTasks#delete` for an app that has only the id, or a list of ids. They are the writes of the feature.

## Context

Task 02 built the read side: `Rage::Deferred.dead_tasks` (`each`, `find_by_id`) and the read-only `Rage::Deferred::DeadTask`. Both classes are in the `[Unreleased]` section of the changelog, so they are not public yet and can still change without a deprecation shim.

A `DeadTask` today holds only its record. It has no way to reach the store, so it cannot remove itself. The backend already has what this task needs to remove records: `remove_dead_tasks(ids)` removes every record with the given ids in one rewrite of the store and returns the number of removed ids. The new WAL entry of a retried task is fsynced only by the `fsync_frequency` timer, so this task adds an optional `flush` to the Disk backend (ADR-002).

`DeadTasks#each` holds no lock of the store while it yields (task 01), so a `delete` or `retry` called inside its block can take the lock.

## Terms

- **Removal** — one call of `remove_dead_tasks` for the id of one record: one rewrite of the dead-tasks store.

## Requirements

### `DeadTask#delete`

```ruby
task = Rage::Deferred.dead_tasks.find_by_id("1759312800-4242-7")
task.delete                                         # => true
Rage::Deferred.dead_tasks.each { |t| t.delete if t.task_class == "SendWelcomeEmail" }
```

- Makes one removal for the id of the record, at once. Returns `true` when it removed a record. It works the same on a record from `find_by_id`, from an Array (`to_a`, `select`), and inside the block of `each`.
- It does not decode the context, so it works for a record whose task class or argument class no longer exists.
- The removal is the existing `remove_dead_tasks`, unchanged. How it treats repeated ids (tested in `spec/deferred/backends/disk_spec.rb`) and a crash during the rewrite (the store is replaced by rename) is the backend's behavior, and this task does not test it again.
- **Runs twice, or the record is already gone** (removed by another worker, or by an earlier `delete` or `retry`): no error is raised, and it returns `false`.
- **The store cannot be locked**: `Rage::Deferred::DeadTasksLockTimeout` reaches the caller and the record stays.

### `DeadTask#retry`

```ruby
task.retry                                          # => true or false
Rage::Deferred.dead_tasks.each(&:retry)
```

It does the eight steps of ADR-002, in this order:

1. Look the record up with `find_dead_task`. If it is gone, return `false`; nothing is enqueued.
2. Resolve the task class from the class name stored in the record, without loading it, and check that it is a class that includes `Rage::Deferred::Task`.
3. Decode the stored context and read the positional and keyword arguments from it.
4. Call the task class's `enqueue` with those arguments and no `delay` or `delay_until`.
5. Call the backend's `flush` when it has one: the Disk backend fsyncs the active WAL file and its directory.
6. When `Iodine.running?` is false, print a warning with the task id that says a server restart is required.
7. Make one removal for the id of the record.
8. Return `true`.

- The retried task is a new task: it has a new id, its attempts are counted from the first one (`meta.attempts` is 1 on its first run), and its task class's retry policy applies in full.
- The arguments are decoded for this call alone and are not frozen. The frozen copies that `args` and `kwargs` return are not passed on: a `perform` or an enqueue middleware that changes an argument in place would raise `FrozenError` for a retried task only.
- A record stored without positional or without keyword arguments is retried without them, on every supported Ruby version (`**nil` is a `TypeError` on Ruby 3.3).
- A task enqueued through `Rage::Deferred.wrap` is retried like any other: its task class is `Rage::Deferred::Proxy::Wrapper`.
- **Step 1 raises** (`DeadTasksLockTimeout`): the error reaches the caller unchanged. Nothing is enqueued.
- **Step 2 fails** (the class no longer exists, or it is not a Deferred task): `Rage::Deferred::DeadTaskRetryError` is raised, with the `NameError` as its `cause` when there is one. Nothing is enqueued and the record stays.
- **Step 3 fails** (an argument class no longer exists, or the blob is broken): `Rage::Deferred::DeadTaskContextDeserializationError` is raised, with the first error as its `cause`. Nothing is enqueued and the record stays.
- **Step 4 or 5 raises** (`Rage::Deferred::PushTimeout`, an error from an enqueue middleware, an `fsync` error): the same error reaches the caller, not wrapped. Step 7 does not run and the record stays.
- **Step 7 raises** (`DeadTasksLockTimeout`, an I/O error): `Rage::Deferred::DeadTaskRetryError` is raised, with that error as its `cause` and a message that says the task may be in both stores. The task is already pending and will run; its record is still in the dead-tasks store. `delete` clears it; a second `retry` runs the task a second time.
- **Runs twice, one after the other**: the second call finds no record in step 1 and returns `false`. Two calls whose steps 1 both run before either step 7 enqueue the task twice (ADR-002).
- **Interrupted between step 4 and step 7** (the process dies): with the Disk backend, the task is in both stores after step 5. It runs after the restart, and its record is still dead.
- **An enqueue middleware that does not yield**: the task is not enqueued, `enqueue` returns as usual, and step 7 removes the record: the task is in neither store, and `retry` returns `true`. This is kept. The middleware's choice applies to a retry as to every enqueue (ADR-002: the regular path), and telling a dropped enqueue from a real one would need a new signal out of `Task.enqueue`, which is on the path of every enqueue. The YARD text of `retry` says so.

### `DeadTasks#delete(id_or_ids)`

```ruby
Rage::Deferred.dead_tasks.delete("1759312800-4242-7")   # => 1 or 0
Rage::Deferred.dead_tasks.delete([id_1, id_2])          # => the number of removed records
```

For an app that has only the id, or a list of ids (ADR-004).

- Takes one String id or one flat Array of String ids. Removes them with one removal, without reading the records first, and returns the number of distinct records removed. Repeated ids are removed once.
- An empty Array returns `0` and does not touch the store.
- **A value that is not a String or a flat Array of Strings**: `TypeError`, before the store is touched. The message does not show the value.
- The rest is as for `DeadTask#delete`.

There is no `DeadTasks#retry(id)`. An app that has only the id calls `find_by_id(id)&.retry`.

### Enqueue outside a running server

`Rage::Deferred::Queue#schedule` returns at once while `Iodine.running?` is false. A task enqueued in such a process, by `retry` or by any other enqueue, is written to the WAL and not scheduled; it runs after the next server boot. This changes every enqueue made outside a server, not only retries.

### Documentation

- YARD `@return`, `@raise`, and `@example` tags on the three methods. The text of each says that a call rewrites the whole store, so its cost grows with the size of the store.
- The YARD text of `retry` also says: it checks that the record still exists and returns `false` when it does not; outside a running server the retried task waits for a later server boot; tasks must be idempotent, because a retry may run a task twice; enqueue middleware runs again and must be safe to run twice on the same arguments; an enqueue middleware that does not yield drops the task, and the record is still removed, so the task is in neither store; the new task gets the log tags and log context of the caller.
- The YARD text of `DeadTasks#delete` says when to use it instead of `DeadTask#delete`, and shows the list form.
- `Rage::Deferred::DeadTaskRetryError` has a class text that says when it is raised and that `cause` holds the original error.
- The class text of `DeadTask` and the examples of `Rage::Deferred.dead_tasks` show `delete` and `retry`.
- The existing `[Deferred]` entry under `## [Unreleased]` in `CHANGELOG.md` is updated to name deleting and retrying; no second entry.

### Not in this task

- Grouping the removals made inside one `each` call into one rewrite (task 05). Here `each(&:delete)` costs one rewrite per record.
- Running a task retried outside a server in a server that is already running (task 06). This task only stops the scheduling there, warns, and documents the limit.
- Protection against two retries of the same record that interleave (a non-goal of the feature).
- A stored keyword argument named `delay` or `delay_until` (only an enqueue middleware can add one) is read by `enqueue` as its own option on retry. Not handled.
- A new telemetry span, a config key, or a required backend contract method.

## Design

### Files

| File | Change |
| --- | --- |
| `lib/rage/deferred/dead_task.rb` | `delete`, `retry`; the constructor takes the backend; private `resolve_task_class` and `decode_task_arguments` |
| `lib/rage/deferred/dead_tasks.rb` | passes the backend to `DeadTask`; new `delete(id_or_ids)` |
| `lib/rage/deferred/deferred.rb` | new `DeadTaskRetryError`; YARD examples of `dead_tasks` |
| `lib/rage/deferred/backends/disk.rb` | new `flush`, and `TasksStorage#sync!` |
| `lib/rage/deferred/queue.rb` | `schedule` returns at once while `Iodine.running?` is false |
| `CHANGELOG.md` | the existing `[Unreleased]` entry |

### Interfaces

```ruby
class Rage::Deferred::DeadTask
  # @private — still `private_class_method :new`; called as `DeadTask.send(:new, record, backend)`
  def initialize(record, backend)      # keeps the backend in @backend

  # @return [Boolean] true when the record was removed, false when it was already gone
  # @raise [Rage::Deferred::DeadTasksLockTimeout]
  def delete

  # @return [Boolean] true when the steps completed, false when the record was already gone
  # @raise [Rage::Deferred::DeadTaskRetryError]
  # @raise [Rage::Deferred::DeadTaskContextDeserializationError]
  # @raise [Rage::Deferred::DeadTasksLockTimeout]
  def retry

  private

  def resolve_task_class               # @return [Class]
  def decode_task_arguments            # @return [Array<(Array, Hash)>], never nil
end

class Rage::Deferred::DeadTasks
  # @param id_or_ids [String, Array<String>]
  # @return [Integer] the number of distinct records removed
  # @raise [TypeError] when the value is not a String or a flat Array of Strings
  # @raise [Rage::Deferred::DeadTasksLockTimeout]
  def delete(id_or_ids)
end

class Rage::Deferred::DeadTaskRetryError < StandardError; end

class Rage::Deferred::Backends::Disk
  # @private — optional for a backend; `DeadTask#retry` calls it after a `respond_to?` check
  def flush                            # fsyncs the active WAL file and its directory
end
```

### Steps

- `DeadTasks#each` and `#find_by_id` build the record object with a private `build_entry(record)`, which calls `Rage::Deferred::DeadTask.send(:new, record, @backend)`.
- `DeadTasks#delete(id_or_ids)`: wraps a String in an Array; raises `TypeError, "dead task ids must be Strings"` unless every element is a String; returns `0` for an empty Array; otherwise returns `@backend.remove_dead_tasks(ids.uniq)`.
- `DeadTask#delete` returns `@backend.remove_dead_tasks(@id) == 1`.
- `DeadTask#retry`:
  1. `return false unless @backend.find_dead_task(@id)`.
  2. `resolve_task_class`: `Object.const_get(@task_class, false)`; a `NameError` becomes `DeadTaskRetryError` with `cause:`; a value that is not a `Class` including `Rage::Deferred::Task` raises `DeadTaskRetryError`.
  3. `decode_task_arguments`: `Marshal.load(@__context)` (not frozen), then `Context.get_args` and `Context.get_kwargs`, `nil` becoming `[]` and `{}`; any error becomes `DeadTaskContextDeserializationError` with `cause:`.
  4. `task_class.enqueue(*args, **kwargs)`, outside any `rescue`.
  5. `@backend.flush if @backend.respond_to?(:flush)`.
  6. `warn "Dead task #{@id} was retried; server restart is required." unless Iodine.running?`.
  7. `@backend.remove_dead_tasks(@id)`; any error becomes `DeadTaskRetryError` with `cause:`.
  8. Return `true`.
- `Disk#flush` calls `TasksStorage#sync!`: `@storage.fsync`, then open the storage directory read-only and `fsync` it. It does not change the timer's state.
- `Queue#schedule` starts with `return unless Iodine.running?`.
- `decode_context` (the readers `args` and `kwargs`) does not change.

### Blocking I/O

New callers of existing kinds of blocking call, and one new call site, all on the reactor when called in a server:

- `DeadTasks#delete` and `DeadTask#delete`, and step 7 of `retry`: `remove_dead_tasks` reads the whole store, writes a new file, and calls `fsync`, rename, and directory `fsync`, under the store's non-blocking lock. The feature spec accepts this cost; the YARD text names it.
- Step 1 of `retry`: one `find_dead_task` read of the whole store.
- Step 3 of `retry`: `Marshal.load` of one context (CPU, the size of one record). Step 4: the regular enqueue (a WAL write).
- Step 5 of `retry` (new call site): `fsync` of the active WAL file and of its directory, once per call. The feature spec accepts it (ADR-002).
- Waits: none added. The existing ones have a limit: the lock retries end in `DeadTasksLockTimeout`, backpressure ends in `PushTimeout`.

## Implementation constraints

- Follows ADR-001, ADR-002, and ADR-004, except the one rule ADR-001 and ADR-002 state about grouping the removals made inside one `each` call: the feature spec makes it a separate, optional task, and requires that every removal in this task is applied at once.
- `Rage::Deferred::Context` and `Rage::Deferred::Task` are not changed. The backend contract gets no required method: `flush` is optional, and `retry` checks for it with `respond_to?`.
- `DeadTasks#delete` never passes a value that is not a flat Array of Strings to the backend.
- The `rescue` in `decode_task_arguments` must not cover step 4. A `PushTimeout` wrapped as a decode error would hide the real cause.
- Step 7 never runs before steps 4 and 5 returned. No `ensure` around the removal.
- `retry` does not call `Queue#enqueue` directly and does not pass `task_id:`; the new id and the attempt count come from the regular path.
- `retry` never passes on the frozen values of `args` and `kwargs`, and never caches its own decode.
- Ruby 3.3.0: `**nil` is not allowed, so `decode_task_arguments` turns a `nil` into `{}`.
- `DeadTask` and `DeadTasks` are not released yet: the constructor change needs no shim.

## Acceptance criteria

- [ ] TC-01 (core, AC-07) When `delete` has returned for a `DeadTask` found with `find_by_id`, `find_by_id` returns `nil` for its id.
- [ ] TC-02 (core, AC-16) When `dead_tasks.each(&:delete)` has returned, `each` yields no records.
- [ ] TC-03 (core, AC-08) When `retry` is called on a `DeadTask`, its task's `perform` runs again with the stored positional and keyword arguments.
- [ ] TC-04 (core, AC-09) When `dead_tasks.each(&:retry)` has returned, `each` yields no records.
- [ ] TC-05 (edge, AC-15) When `dead_tasks.each(&:retry)` is called, the `perform` of every dead task runs again.
- [ ] TC-06 (core, AC-10) When a retried task runs for the first time, `meta.attempts` inside its `perform` is 1.
- [ ] TC-07 (core, AC-11) If the stored context cannot be decoded, `retry` leaves the record in the dead-tasks store.
- [ ] TC-08 (core) If the enqueue of a retried task raises, the record stays in the dead-tasks store.
- [ ] TC-09 (edge, AC-11) If the stored context cannot be decoded, `retry` raises `Rage::Deferred::DeadTaskContextDeserializationError`.
- [ ] TC-10 (edge) If the record cannot be removed after a retried task was enqueued, `retry` raises `Rage::Deferred::DeadTaskRetryError`.
- [ ] TC-11 (edge) When `delete` is called for a record that is no longer in the store, it returns `false`.
- [ ] TC-12 (edge) When the `perform` of a retried task changes one of its arguments in place, no `FrozenError` is raised.
- [ ] TC-13 (edge) When `delete` removes a record, it returns `true`.
- [ ] TC-14 (core, AC-24) When `dead_tasks.delete` is called with the id of a stored record, `find_by_id` returns `nil` for that id.
- [ ] TC-18 (edge) When `retry` is called for a record that is no longer in the store, it returns `false` and nothing is enqueued.
- [ ] TC-19 (core, AC-26) When `dead_tasks.delete` is called with an Array of stored ids, `find_by_id` returns `nil` for each of them.
- [ ] TC-20 (edge) If `dead_tasks.delete` is called with a value that is not a String or a flat Array of Strings, it raises `TypeError`.
- [ ] TC-21 (edge) If the stored task class no longer exists, `retry` raises `Rage::Deferred::DeadTaskRetryError`.
- [ ] TC-22 (core) When `retry` removes a record from a Disk store, the retried task's WAL entry has already been fsynced.
- [ ] TC-23 (edge) While `Iodine.running?` is false, an enqueued task is written to the WAL and not scheduled.

## Verification

Tests, as on the implementation branch:

- `spec/deferred/dead_task_deletion_spec.rb` — `DeadTask#delete` and `DeadTasks#delete`, over a fake backend and over a real `Rage::Deferred::Backends::Disk`.
- `spec/deferred/dead_task_retry_spec.rb` — `DeadTask#retry`, over a fake backend and over a real Disk store.
- `spec/deferred/backends/disk_spec.rb` — `Disk#flush`.
- `spec/deferred/queue_spec.rb` — `Queue#schedule` while `Iodine.running?` is false.
- One example or more per criterion. TC-03 also needs: positional arguments only, keyword arguments only, no arguments, and a task enqueued through `Rage::Deferred.wrap`. TC-02, TC-04, and TC-05 over a real Disk store with 3 or more records, checking that each id is yielded once and none is left.
- TC-08 with `Rage::Deferred::PushTimeout`; the example also checks that the same error reaches the caller.
- TC-18 also checks that nothing is enqueued; TC-20 that the store is not touched.

Existing examples changed: `spec/deferred/queue_spec.rb` stubs `Iodine.running?` to `true` for its existing examples.

Commands:

- `bundle exec rspec spec/deferred/dead_task_deletion_spec.rb spec/deferred/dead_task_retry_spec.rb spec/deferred/dead_tasks_spec.rb spec/deferred/backends/disk_spec.rb spec/deferred/queue_spec.rb`
- `bundle exec rubocop lib/rage/deferred/dead_task.rb lib/rage/deferred/dead_tasks.rb lib/rage/deferred/deferred.rb lib/rage/deferred/backends/disk.rb lib/rage/deferred/queue.rb spec/deferred/dead_task_deletion_spec.rb spec/deferred/dead_task_retry_spec.rb`
- `bundle exec yardoc --fail-on-warning`

The examples that need `within_reactor` may be skipped without `ENABLE_EXTERNAL_TESTS=true`; the record names the ones that did not run.

## Result

- Implementation PR:
