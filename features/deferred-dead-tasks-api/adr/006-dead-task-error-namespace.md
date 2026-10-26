---
status: rejected
---

# ADR-006: Dead-task errors under `DeadTask`

> **Rejected on 2026-10-26.** It was dropped together with [ADR-005](005-storage-errors.md). `Rage::Deferred::DeadTaskRetryError` and `Rage::Deferred::DeadTaskContextDeserializationError` keep their names at the top of `Rage::Deferred` (Option A).

## Context

Two error classes belong to one public class, `Rage::Deferred::DeadTask`, but are defined at the top of `Rage::Deferred`, in `lib/rage/deferred/deferred.rb`:

- `Rage::Deferred::DeadTaskRetryError` — `DeadTask#retry` cannot use the stored task class, or cannot remove the dead record after the enqueue (ADR-002, steps 2 and 7).
- `Rage::Deferred::DeadTaskContextDeserializationError` — `DeadTask#args`, `#kwargs`, or `#retry` cannot decode the stored context (ADR-002, step 3).

Only `DeadTask` raises them. The `DeadTask` prefix in both names repeats a namespace that Ruby already has. Other error classes in `Rage::Deferred` (`PushTimeout`, `StorageError`) belong to the whole module, so the top level does not show which errors come from which object.

Tasks 02 and 03 added both classes, and they are not released yet: the dead-tasks API is under `## [Unreleased]` in `CHANGELOG.md`. A move now breaks no app. After the release that ships the dead-tasks API, the same move needs a deprecation.

## Options considered

### Option A — Keep the names at the top of `Rage::Deferred` (current)

- Benefits: no change; ADR-002 and ADR-005 stay as they are.
- Drawbacks: after the release, the names can only change with a deprecation, so the chance to place them with their class is gone.

Lost: the engineer chose to place the errors with the class that raises them while it is still free.

### Option B — Move them under `DeadTask` and keep their names

`Rage::Deferred::DeadTask::DeadTaskRetryError` and `Rage::Deferred::DeadTask::DeadTaskContextDeserializationError`.

- Benefits: the class names stay the same, so a search for the old name still finds the class.
- Drawbacks: `DeadTask::DeadTask…` repeats the namespace in every `rescue`.

Lost: the repetition adds length and no meaning.

### Option C — Move them under `DeadTask` with shorter names (selected)

`Rage::Deferred::DeadTask::RetryError` and `Rage::Deferred::DeadTask::ContextDeserializationError`.

- Benefits: the namespace says that the error comes from a dead task, and the class name says what failed. Inside `DeadTask`, the code names them as `RetryError` and `ContextDeserializationError`.
- Drawbacks: ADR-002 and ADR-005 use the old names until they are patched.

## Decision

Option C.

- `Rage::Deferred::DeadTaskRetryError` becomes `Rage::Deferred::DeadTask::RetryError`.
- `Rage::Deferred::DeadTaskContextDeserializationError` becomes `Rage::Deferred::DeadTask::ContextDeserializationError`.
- Both are defined inside `class Rage::Deferred::DeadTask`, in `lib/rage/deferred/dead_task.rb`. Each keeps its superclass (`StandardError`), its class text, and when it is raised.
- The old names are removed, with no alias and no deprecation, because they were never released.

This replaces the class names only, not the behavior, in:

- ADR-002: steps 2, 3, and 7 of `retry`, and "A new public error class, `Rage::Deferred::DeadTaskRetryError`" in its Consequences;
- ADR-005: the names `DeadTaskRetryError` and `DeadTaskContextDeserializationError` in its Context and Decision.

## Consequences

- An app rescues `Rage::Deferred::DeadTask::RetryError` and `Rage::Deferred::DeadTask::ContextDeserializationError`. The old names raise `NameError`, but no released version had them.
- The error classes of `Rage::Deferred` that remain at the top level (`PushTimeout`, `StorageError`) are the ones that do not belong to one object.
- The move must ship in the same release as the dead-tasks API, or before it. After that release, a change to these names needs a deprecation.
- No runtime cost: the classes are defined once, when `dead_task.rb` loads.
