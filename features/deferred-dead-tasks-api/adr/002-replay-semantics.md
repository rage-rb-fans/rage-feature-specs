---
status: accepted
---

# ADR-002: Replay semantics

## Context

Replay moves a task from the dead-tasks store back to the pending-task WAL. The two stores are separate files with no shared transaction, so a crash can land between the two steps. Four choices follow from the current code:

- **Order of the two writes.** `Queue#abandon_task` already sets the pattern for the other direction: write to the new store, then remove from the old one.
- **The task id.** `Queue#enqueue` takes an optional `task_id:`; without it the backend generates a new one. The dead-tasks store keys records by task id, and `remove_dead_tasks` removes every record with a given id.
- **The attempt count.** The stored context carries the number of attempts made. `Task.__calculate_next_retry_in` returns `nil` once attempts are above the limit.
- **The enqueue path.** `Task.enqueue` opens the `deferred.task.enqueue` telemetry span and runs the enqueue middleware chain, which may change the arguments and write into the context (the user context). The stored context already holds what the chain wrote at the first enqueue. An automatic retry skips both and calls `Queue#enqueue` directly.

Removing from the dead-tasks store takes a lock with retries that `sleep`, so the calling fiber can yield between the enqueue and the removal, and the replayed task can start, fail, and die again inside that window (for example with `max_retries 0`).

## Options considered

### Order: remove from the dead-tasks store first, then enqueue

- Benefits: a task is never in both stores, so it can never run twice because of replay.
- Drawbacks: a crash, a `PushTimeout`, or any error between the two steps loses the task for good.

Lost: `Rage::Deferred` promises at-least-once delivery; this is at-most-once.

### Order: enqueue first, then remove from the dead-tasks store (selected)

- Benefits: the task is always in at least one store.
- Drawbacks: a crash or a `DeadTasksLockTimeout` between the two steps leaves the task both pending and dead. It runs, and its dead record can be replayed a second time.

### Id: reuse the dead task's id

- Benefits: one id for the whole life of the task; `enqueued_at` (derived from the id) keeps the time of the first enqueue.
- Drawbacks: if the replayed task dies again before the removal step runs, its new dead record has the same id, and the removal step deletes it together with the old one. The task is then in neither store.

Lost: it can lose a task without a crash.

### Id: a new id for the replayed task (selected)

- Benefits: the removal step can only remove the record that was replayed.
- Drawbacks: the replayed task has no link to its old dead record, and if it dies again, `enqueued_at` of the new record is the replay time.

### Attempts: keep the stored count

- Benefits: the count shows every attempt the task ever made.
- Drawbacks: the count is already above the limit, so the first failure after replay sends the task straight back to the dead-tasks store, with no retry.

Lost: replay would mean "one more try" for every task class, whatever its retry policy says.

### Attempts: start from zero (selected)

- Benefits: a replayed task behaves like a newly enqueued one: `meta.attempts` is 1 on its first run and the task class's retry policy applies in full.
- Drawbacks: the earlier attempts are no longer visible in a later dead record.

### Enqueue path: the stored context goes straight to `Queue#enqueue`

The path an automatic retry uses today: no `deferred.task.enqueue` span, no enqueue middleware.

- Benefits: the task runs with the context it had (log tags, log context, user context); the enqueue middleware is not applied twice.
- Drawbacks: telemetry handlers and enqueue middleware that must see every enqueue (a counter, a trace) do not see replays. Resetting the attempt count needs a new internal method on `Rage::Deferred::Context`.

Lost: the engineer decided that a replayed task goes through the same path as a regular task.

### Enqueue path: the regular one, `Task.enqueue` with the decoded arguments (selected)

The task class's `enqueue` is called with the stored positional and keyword arguments, as the app would call it.

- Benefits: one enqueue path. The `deferred.task.enqueue` span and the enqueue middleware run, so replays are visible to handlers and middleware. A new context is built, so the attempt count starts from zero and the task gets a new id with no extra code. Backpressure applies as for every enqueue.
- Drawbacks:
  - The enqueue middleware runs a second time for the same task. The stored arguments are the ones the middleware already saw and may have changed at the first enqueue, so a middleware that changes arguments must be safe to apply twice.
  - The log tags, log context, and user context stored with the dead task are not carried over. The new task gets the log tags and log context of the fiber that calls `replay`, and the user context that the middleware and the telemetry handlers write this time.

## Decision

`DeadTask#replay` does this for its record, in this order:

1. Decode the stored context. If that fails, leave the record in the dead-tasks store and stop; the task is not enqueued.
2. Read the task class and the positional and keyword arguments from it, through `Rage::Deferred::Context`.
3. Call the task class's `enqueue` with those arguments and no delay. This is the regular path: the `deferred.task.enqueue` span, the enqueue middleware, a new context with no attempts, and a new task id.
4. Remove the record from the dead-tasks store.

Step 4 of the records replayed inside one `each` call is grouped into one rewrite of the store, applied when `each` ends in any way (see ADR-001). The framework never runs step 4 for a record before step 3 succeeded for it. An error raised in step 3 or step 4 (`PushTimeout`, `DeadTasksLockTimeout`) is raised to the caller and the record stays in the dead-tasks store.

Replay is at-least-once: it may run a task twice, and it never loses one.

## Consequences

- An app that replays must have idempotent tasks, the same as for automatic retries. The YARD text of `replay` says so.
- After a failure between step 3 and step 4 the record is still in the dead-tasks store although its task is already pending. Calling `replay` again runs the task again; calling `delete` is the way to clear it.
- Inside `each`, the window between step 3 and step 4 covers every record replayed in that call, so a crash or a failure in step 4 leaves all of them both pending and dead.
- `replay` does not check that its record is still in the store. Two workers that replay the same record, or a second `replay` after a failed step 4, enqueue the task twice. The engineer accepted this for now; an idempotency key for replay is planned as later work, with its behavior still to be decided.
- `enqueued_at` and `attempts` of a task that dies after a replay describe the replay, not the first life of the task.
- Neither `Rage::Deferred::Context` nor the backend contract gets a new method.
- Telemetry handlers and enqueue middleware see a replay as one more enqueue of the task class. They cannot tell it from an enqueue made by the app.
- Enqueue middleware must be safe to run twice on the same arguments. The YARD text of `replay` says so.
- Log lines of a replayed task carry the log tags of the caller of `replay`, not those of the request that first enqueued the task.
