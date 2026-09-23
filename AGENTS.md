# Agent instructions

This repository is the specification knowledge base for the [Rage Ruby framework](https://github.com/rage-rb/rage). It contains product and implementation context; production code belongs in the Rage repository.

## Repository model

- Each directory under `features/` represents one feature.
- `spec.md` describes the feature's motivation, behavior, scope, acceptance criteria, tasks, decisions, and references.
- `tasks/` contains independently implementable units of work. A task should contain enough context and technical detail for an implementation agent to complete it without relying on a pull request description or chat history.
- `adr/` contains architecture decision records for decisions that need a durable explanation.
- Use the files in `templates/` when creating a feature, task, or ADR.

## Statuses

Parent feature specifications have no lifecycle status. Each task advances independently:

- `draft`: requirements are still being developed or amended. Do not implement it.
- `ready-for-development`: published and explicitly approved for implementation.
- `done`: implementation merged and verified. The Result section must link to the implementation.

ADR statuses use the values defined by the ADR itself, such as `proposed`, `accepted`, or `superseded`.

## Working with a feature

Before changing or implementing a task:

1. Read the parent `spec.md`, the complete task file, and every ADR linked by either document.
2. Follow applicable instructions in the Rage implementation repository and inspect the current code before proposing changes.
3. Treat the feature and task acceptance criteria as required observable outcomes. Treat implementation constraints and accepted ADRs as requirements.
4. If the documents conflict with each other or with the current Rage implementation, surface the conflict explicitly. Do not silently reinterpret the specification.
5. Keep implementation work scoped to one task unless completing it necessarily requires a related change.
6. Verify the task using its Verification section and the relevant checks from the Rage repository.

After implementation is merged:

1. Change the task status to `done`.
2. Complete its acceptance criteria and Result section with links to the implementation pull request and, when useful, the merge commit.
3. Mark the task complete in the parent feature's task list.
4. Leave the parent specification without a status; its checklist summarizes task progress without gating other tasks.

## Editing specifications

- Write observable behavior and concrete constraints. Avoid depending on unstated conversation context.
- Keep task files self-contained and small enough to produce one reviewable implementation change.
- Preserve links between a feature, its tasks, its ADRs, the Rage issue, and implementation pull requests.
- Record significant technical choices in an ADR instead of hiding them in task implementation notes.
- Do not mark acceptance criteria complete based only on an intended design; require evidence from the merged implementation or its verification.
- Keep optional local review, gap, and verification reports under the gitignored `.workflow/evidence/` directory. They are not canonical task state.
