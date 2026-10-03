---
status: draft
---

# Task name

## Goal

Describe the concrete outcome of this unit of work and why it is needed.

## Context

Explain the relevant current behavior, constraints, and relationship to the parent feature.

## Terms

- **Term** — what it means in this task.

Define only words used in this task alone; never redefine a term from the feature spec. Remove this section if there are none.

## Requirements

Describe the behavior this task must deliver. Include interfaces, examples, data formats, edge cases, and failure behavior where they make the task unambiguous.

## Design

Describe the intended technical approach in enough detail for an implementation agent to execute it. Diagrams and code examples are encouraged when they communicate the design more clearly.

## Implementation constraints

Record compatibility requirements, invariants, rejected alternatives, and details that must not be changed during implementation. Name every ADR this task follows, for example `Follows ADR-002`.

## Acceptance criteria

- [ ] TC-01 (AC-01) Describe one observable outcome: one situation, one result.
- [ ] TC-02 Describe another observable outcome.

Give each criterion a `TC-NN` ID (task criterion) that is never renumbered or reused, and the feature criteria (`AC-NN`) it helps prove. A list of tests belongs in Verification, not here.

## Open questions

- Q-01: The question. Decides: who answers it. Recommendation: the proposed answer.

When a question is answered, write the answer into the section it belongs to (or an ADR) and delete the question. Remove this section when no question is open.

## Verification

- Tests to add or update
- Commands to run

## Result

- Implementation PR:
