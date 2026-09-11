# Rage Feature Specifications

This repository contains collaborative feature specifications for the [Rage Ruby framework](https://github.com/rage-rb/rage).

Feature specifications describe the motivation, expected behavior, scope, and acceptance criteria for a change before implementation begins. They also provide developers and AI agents with a shared source of context during implementation.

## Structure

```text
features/
└── feature-name/
    ├── spec.md
    ├── tasks/
    │   └── 01-task-name.md
    └── adr/
        └── 001-decision-name.md

templates/
├── feature.md
├── task.md
└── adr.md
```

- `features/` contains one directory per feature.
- `spec.md` is the main feature document and links to its tasks and decisions.
- `tasks/` contains small, independently implementable units of work.
- `adr/` contains architecture decision records when a feature requires a significant technical decision.
- `templates/` contains starting templates for each document type.

## Feature status

Each feature declares its status in the front matter of `spec.md`:

- `draft` — the specification is being discussed and refined.
- `implementation` — the specification is agreed and ready to implement.
- `done` — the implementation is merged and its acceptance criteria are verified.

Feature directories remain in place when their status changes so links to specifications, tasks, and decisions stay stable.
