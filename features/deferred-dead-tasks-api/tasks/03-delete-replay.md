---
status: ready-for-development
---

# Delete and replay one dead task

## Goal

Add `Rage::Deferred::DeadTask#delete` and `Rage::Deferred::DeadTask#replay`, so that an app can remove a dead task or run it again through the public API, and `Rage::Deferred::DeadTasks#delete(id)` and `#replay(id)` for an app that has only the id. They are the writes of the feature.

## Context

Task 02 built the read side: `Rage::Deferred.dead_tasks` (`each`, `find_by_id`) and the read-only `Rage::Deferred::DeadTask`. Both classes are in the `[Unreleased]` section of the changelog, so they are not public yet and can still change without a deprecation shim.

A `DeadTask` today holds only its record. It has no way to reach the store, so it cannot remove itself. The backend already has what this task needs: `remove_dead_tasks(ids)` removes every record with the given ids in one rewrite of the store and returns the number of removed ids. The backend contract does not change in this task.

`DeadTasks#each` holds no lock of the store while it yields (task 01), so a `delete` or `replay` called inside its block can take the lock.

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
- **Runs twice, or the record is already gone** (removed by another worker, or by an earlier `delete` or `replay`): no error is raised, and it returns `false`.
- **The store cannot be locked**: `Rage::Deferred::DeadTasksLockTimeout` reaches the caller and the record stays.

### `DeadTask#replay`

```ruby
task.replay                                         # => nil
Rage::Deferred.dead_tasks.each(&:replay)
```

It does the four steps of ADR-002, in this order:

1. Decode the stored context.
2. Read the task class and the positional and keyword arguments from it.
3. Call the task class's `enqueue` with those arguments and no `delay` or `delay_until`.
4. Make one removal for the id of the record. Return `nil`.

- The replayed task is a new task: it has a new id, its attempts are counted from the first one (`meta.attempts` is 1 on its first run), and its task class's retry policy applies in full.
- The arguments are decoded for this call alone and are not frozen. The frozen copies that `args` and `kwargs` return are not passed on: a `perform` or an enqueue middleware that changes an argument in place would raise `FrozenError` for a replayed task only.
- A record stored without positional or without keyword arguments is replayed without them, on every supported Ruby version (`**nil` is a `TypeError` on Ruby 3.3).
- A task enqueued through `Rage::Deferred.wrap` is replayed like any other: its task class is `Rage::Deferred::Proxy::Wrapper`.
- **Step 1 or 2 fails** (the task class or an argument class no longer exists, or the blob is broken): `Rage::Deferred::DeadTaskContextDeserializationError` is raised, with the first error as its `cause`. Nothing is enqueued and the record stays.
- **Step 3 raises** (`Rage::Deferred::PushTimeout`, an error from an enqueue middleware, `NoMethodError` when the class is no longer a Deferred task): the same error reaches the caller, not wrapped. Step 4 does not run and the record stays.
- **Step 4 raises** (`DeadTasksLockTimeout`): the error reaches the caller. The task is already pending and will run; its record is still in the dead-tasks store. `delete` clears it; a second `replay` runs the task a second time.
- **Runs twice**: every call enqueues the task again. `replay` does not check that its record is still in the store (ADR-002).
- **Interrupted between step 3 and step 4** (the process dies): the task is in both stores. It runs after the restart, and its record is still dead.
- **An enqueue middleware that does not yield**: the task is not enqueued, `enqueue` returns as usual, and step 4 removes the record: the task is in neither store. This is kept. The middleware's choice applies to a replay as to every enqueue (ADR-002: the regular path), and telling a dropped enqueue from a real one would need a new signal out of `Task.enqueue`, which is on the path of every enqueue. The YARD text of `replay` says so.

### `DeadTasks#delete(id)` and `DeadTasks#replay(id)`

```ruby
Rage::Deferred.dead_tasks.delete("1759312800-4242-7")   # => true or false
Rage::Deferred.dead_tasks.replay("1759312800-4242-7")   # => true or false
```

For an app that has only the id (ADR-004). Each takes exactly one id.

- `delete(id)` makes one removal for the id, without reading the record first. It returns `true` when it removed a record, and `false` when no record had this id. The rest is as for `DeadTask#delete`.
- `replay(id)` finds the record as `find_by_id` does, then does what `DeadTask#replay` does, with the same errors. It returns `true` when the task was enqueued, and `false` when no record had this id; then nothing is enqueued.
- **An id that is not a String**, an Array of ids included: `TypeError`, with the message of `find_by_id`. Nothing is removed or enqueued.

### Documentation

- YARD `@return`, `@raise`, and `@example` tags on the four methods. The text of each says that a call rewrites the whole store, so its cost grows with the size of the store.
- The YARD text of `replay` also says: it is supported inside a running server only (in the console or a Rake task the replayed task waits for a later server boot); tasks must be idempotent, because a replay may run a task twice; enqueue middleware runs again and must be safe to run twice on the same arguments; an enqueue middleware that does not yield drops the task, and the record is still removed, so the task is in neither store; the new task gets the log tags and log context of the caller.
- The YARD text of `DeadTasks#delete` and `#replay` names the `DeadTask` methods: use those inside `each`, and these when the app has only the id.
- The class text of `DeadTask` and the examples of `Rage::Deferred.dead_tasks` show `delete` and `replay`.
- The existing `[Deferred]` entry under `## [Unreleased]` in `CHANGELOG.md` is updated to name deleting and replaying; no second entry.

### Not in this task

- Grouping the removals made inside one `each` call into one rewrite (task 05). Here `each(&:delete)` costs one rewrite per record.
- Replay outside a running server (task 06). This task adds no check and no error for it; it only documents the limit.
- Protection against a second replay of the same record (a non-goal of the feature).
- A stored keyword argument named `delay` or `delay_until` (only an enqueue middleware can add one) is read by `enqueue` as its own option on replay. Not handled.
- A new telemetry span, a log line, a config key, or a backend contract method.

## Design

### Files

| File | Change |
| --- | --- |
| `lib/rage/deferred/dead_task.rb` | `delete`, `replay`; the constructor takes the collection; `read_context` replaces the body of `decode_context` |
| `lib/rage/deferred/dead_tasks.rb` | passes itself to `DeadTask`; new `delete(id)` and `replay(id)` |
| `lib/rage/deferred/deferred.rb` | YARD examples of `dead_tasks` only |
| `CHANGELOG.md` | the existing `[Unreleased]` entry |

### Interfaces

```ruby
class Rage::Deferred::DeadTask
  # @private — still `private_class_method :new`; called as `DeadTask.send(:new, record, dead_tasks)`
  def initialize(record, dead_tasks)   # keeps the collection in @__dead_tasks

  # @return [Boolean] true when a record was removed, false when none had this id
  # @raise [Rage::Deferred::DeadTasksLockTimeout]
  def delete

  # @return [nil]
  # @raise [Rage::Deferred::DeadTaskContextDeserializationError]
  # @raise [Rage::Deferred::PushTimeout]
  # @raise [Rage::Deferred::DeadTasksLockTimeout]
  def replay

  private

  # @return [Array<(Class, Array or nil, Hash or nil)>] the task class, args, and kwargs
  # @raise [Rage::Deferred::DeadTaskContextDeserializationError]
  def read_context(freeze:)
end

class Rage::Deferred::DeadTasks
  # @param id [String]
  # @return [Boolean] true when a record was removed, false when none had this id
  # @raise [TypeError] when the id is not a String
  # @raise [Rage::Deferred::DeadTasksLockTimeout]
  def delete(id)

  # @param id [String]
  # @return [Boolean] true when the task was enqueued, false when no record had this id
  # @raise [TypeError] and the errors of DeadTask#replay
  def replay(id)
end
```

The tests reach a `DeadTask` through `DeadTasks.new(backend).find_by_id(id)` or `each`, and call `delete(id)` and `replay(id)` on the same `DeadTasks`, with a fake backend that answers `each_dead_task`, `find_dead_task`, and `remove_dead_tasks(ids)`. A stored context is built as `Marshal.dump(Rage::Deferred::Context.build(task_class, args, kwargs))`. The replayed task goes through `Rage::Deferred.__queue`.

### Steps

- `DeadTasks#each` and `#find_by_id` build the record object with `Rage::Deferred::DeadTask.send(:new, record, self)`. `DeadTasks` gets no new instance variable.
- `DeadTasks#delete(id)` raises `TypeError, "dead task id must be a String"` unless `id.is_a?(String)`, then returns `@backend.remove_dead_tasks(id) > 0`. It is the one place where the public API removes a record.
- `DeadTasks#replay(id)`: `task = find_by_id(id)` (which already checks the type); `false` when it is `nil`; otherwise `task.replay` and `true`.
- `DeadTask#delete` returns `@__dead_tasks.delete(@id)`.
- `DeadTask#read_context(freeze:)` runs `Marshal.load(@__context, freeze:)` and reads `get_task`, `get_args`, and `get_kwargs` through `Rage::Deferred::Context`, inside the one `rescue => e` that raises `DeadTaskContextDeserializationError` with `cause: e` (the code that is in `decode_context` today). It returns the three values as they are, `nil` included.
- `decode_context` becomes: `_, args, kwargs = read_context(freeze: true)`, then the `nil` defaults and the two instance variables, as today. It stores nothing when `read_context` raises.
- `DeadTask#replay`:
  1. `task, args, kwargs = read_context(freeze: false)` — steps 1 and 2 of ADR-002. Nothing is cached.
  2. `task.enqueue(*args, **(kwargs || {}))` — step 3, outside any `rescue`.
  3. `@__dead_tasks.delete(@id)` — step 4. Its result is not used.
  4. Return `nil`.

### Blocking I/O

No new kind of blocking call; new callers of existing ones, all on the reactor when called in a server:

- `DeadTasks#delete` (and through it `DeadTask#delete` and step 4 of `replay`): `remove_dead_tasks` reads the whole store, writes a new file, and calls `fsync`, rename, and directory `fsync`, under the store's non-blocking lock. The feature spec accepts this cost; the YARD text names it.
- `DeadTasks#replay`: one `find_dead_task` read of the store first.
- `replay`: `Marshal.load` of one context (CPU, the size of one record), then the regular enqueue (a WAL write, `fsync` by `fsync_frequency`).
- Waits: none added. The two existing ones have a limit: the lock retries end in `DeadTasksLockTimeout`, backpressure ends in `PushTimeout`.

## Implementation constraints

- Follows ADR-001 and ADR-002, except the one rule both state about grouping the removals made inside one `each` call: the feature spec makes it a separate, optional task, and requires that every removal in this task is applied at once.
- Follows ADR-004, which replaces the sentence of ADR-001 that makes the `DeadTask` methods the only writes.
- Rejected: `DeadTask#replay` calling `DeadTasks#replay(@id)` (ADR-004, Option B). `each(&:replay)` would read the whole store once more per record.
- Rejected: passing the argument of `DeadTasks#delete` to the backend unchecked. `remove_dead_tasks` accepts an Array, so `delete([a, b])` would be a public batch write.
- The backend contract, `Rage::Deferred::Context`, `Rage::Deferred::Queue`, and `Rage::Deferred::Task` are not changed.
- `replay` returns `nil`, not a Boolean. It enqueues the task even when the record is already gone, so a Boolean would only say whether the dead record was still there, which is easy to read as "the task was replayed".
- `replay` never passes on the frozen values of `args` and `kwargs`, and never caches its own decode.
- The `rescue` in `read_context` must not cover step 3. A `PushTimeout` wrapped as a decode error would hide the real cause.
- Step 4 never runs before step 3 returned. No `ensure` around the removal.
- `replay` does not call `Queue#enqueue` directly and does not pass `task_id:`; the new id and the attempt count come from the regular path.
- No `respond_to?(:enqueue)` check and no error class of its own for a class that is no longer a Deferred task; the `NoMethodError` is clear enough.
- Ruby 3.3.0: `**nil` is not allowed, so step 3 uses `kwargs || {}`.
- `DeadTask` and `DeadTasks` are not released yet: the constructor change needs no shim.

## Acceptance criteria

- [ ] TC-01 (core, AC-07) When `delete` has returned for a `DeadTask` found with `find_by_id`, `find_by_id` returns `nil` for its id.
- [ ] TC-02 (core, AC-16) When `dead_tasks.each(&:delete)` has returned, `each` yields no records.
- [ ] TC-03 (core, AC-08) When `replay` is called on a `DeadTask`, its task's `perform` runs again with the stored positional and keyword arguments.
- [ ] TC-04 (core, AC-09) When `dead_tasks.each(&:replay)` has returned, `each` yields no records.
- [ ] TC-05 (edge, AC-15) When `dead_tasks.each(&:replay)` is called, the `perform` of every dead task runs again.
- [ ] TC-06 (core, AC-10) When a replayed task runs for the first time, `meta.attempts` inside its `perform` is 1.
- [ ] TC-07 (core, AC-11) If the stored context cannot be decoded, `replay` leaves the record in the dead-tasks store.
- [ ] TC-08 (core) If the enqueue of a replayed task raises, the record stays in the dead-tasks store.
- [ ] TC-09 (edge, AC-11) If the stored context cannot be decoded, `replay` raises `Rage::Deferred::DeadTaskContextDeserializationError`.
- [ ] TC-10 (edge) If the store cannot be locked after a replayed task was enqueued, `replay` raises `Rage::Deferred::DeadTasksLockTimeout`.
- [ ] TC-11 (edge) When `delete` is called for a record that is no longer in the store, it returns `false`.
- [ ] TC-12 (edge) When the `perform` of a replayed task changes one of its arguments in place, no `FrozenError` is raised.
- [ ] TC-13 (edge) When `delete` removes a record, it returns `true`.
- [ ] TC-14 (core, AC-24) When `dead_tasks.delete` is called with the id of a stored record, `find_by_id` returns `nil` for that id.
- [ ] TC-15 (core, AC-25) When `dead_tasks.replay` is called with the id of a stored record, the task's `perform` runs again with the stored arguments.
- [ ] TC-16 (edge) When `dead_tasks.replay` is called with an id that no record has, it returns `false`.
- [ ] TC-17 (edge) If `dead_tasks.delete` is called with an Array of ids, it raises `TypeError`.

## Verification

Tests to add, in a new `spec/deferred/dead_task_spec.rb`:

- One example or more per criterion. TC-03 also needs: positional arguments only, keyword arguments only, no arguments, and a task enqueued through `Rage::Deferred.wrap`.
- TC-08 with `Rage::Deferred::PushTimeout`; the example also checks that the same error reaches the caller.
- TC-16 also checks that nothing is enqueued; TC-17 that nothing is removed.
- TC-03, TC-05, TC-06, TC-12, and TC-15 need `perform` to run: `within_reactor`, or `Iodine.run_after` stubbed as in `spec/deferred/queue_spec.rb`.

Existing examples: none is known to break. The examples in `spec/deferred/dead_tasks_spec.rb` build every `DeadTask` through `DeadTasks`, and the one that checks `Marshal.load(..., freeze: true)` for `args` still holds.

Commands:

- `bundle exec rspec spec/deferred/dead_task_spec.rb spec/deferred/dead_tasks_spec.rb`
- `bundle exec rubocop lib/rage/deferred/dead_task.rb lib/rage/deferred/dead_tasks.rb lib/rage/deferred/deferred.rb spec/deferred/dead_task_spec.rb`
- `bundle exec yardoc --fail-on-warning`

The examples that need `within_reactor` may be skipped without `ENABLE_EXTERNAL_TESTS=true`; the record names the ones that did not run.

## Result

- Implementation PR:
