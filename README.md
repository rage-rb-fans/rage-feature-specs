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

.workflow/                       # optional local reports; gitignored
└── evidence/
```

- `features/` contains one directory per feature.
- `spec.md` is the main feature document and links to its tasks and decisions.
- `tasks/` contains small, independently implementable units of work.
- `adr/` contains architecture decision records when a feature requires a significant technical decision.
- `templates/` contains starting templates for each document type.
- `.workflow/evidence/` may hold local review, gap, and verification reports; it is not canonical state and is not committed.

## Task status

Parent feature specifications have no lifecycle status. Each task declares its own status:

- `draft` — requirements are being discussed or amended.
- `ready-for-development` — the published task is approved for implementation.
- `done` — implementation is merged, verified, and linked from the Result section.

Feature directories remain in place so links to specifications, tasks, and decisions stay stable. The parent task checklist summarizes progress without gating independently reviewable tasks.
