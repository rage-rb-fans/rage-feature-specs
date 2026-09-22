---
status: proposed
---

# ADR-002: Dead-task retry lifecycle

## Context

A dead-task record contains the original serialized execution context and final failure metadata. Retrying must choose whether to reuse that internal context or represent a fresh public enqueue. It must also order the new pending WAL write and dead-record removal without a transaction across the two files.

Issue #369 explicitly deferred periodic adoption of WAL files created out of band. A Rake or console process can therefore persist work that a running worker will not discover until restart. The issue discussion also requested that `Queue#schedule` do nothing unless Iodine is running.

## Options considered

### Option A: Remove the dead record, then enqueue

This prevents two visible copies, but an enqueue failure or crash permanently loses the only recoverable copy. It violates the feature's at-least-once priority.

### Option B: Replay the serialized internal context, then remove

Calling the queue directly can preserve logger and user context and reuse internal values, but bypasses the public enqueue API, enqueue middleware, and enqueue telemetry. It couples the public feature to the private context layout.

### Option C: Perform a fresh public enqueue, then remove

Resolve the stored task class, decode args/kwargs, and call its normal `enqueue`. This creates a new task ID and context, resets attempts, and runs current middleware and telemetry. Only after enqueue succeeds is the dead record deleted. A failure deleting after enqueue can leave both copies, and concurrent operators can enqueue duplicates.

### Option D: Add orphan-WAL adoption before exposing retry

Workers periodically discover WALs created by Rake or console processes. This improves immediacy but introduces a separate scheduler/storage ownership feature that the issue discussion explicitly deferred.

## Decision

Adopt Option C and continue to defer Option D, subject to user acceptance of this proposed ADR.

Retry is a fresh enqueue of the original positional and keyword arguments under the current application code and configuration. It does not preserve the original logger/user context, task ID, attempt count, delay, or exception object. The dead record is removed only after enqueue succeeds.

`Queue#schedule` first checks `Iodine.running?` and performs no scheduling work when false. Retry outside a running server writes the pending WAL, emits a clear restart-required warning, and then removes the dead record after persistence succeeds. Periodic orphan-WAL adoption belongs to a future feature.

## Consequences

- Enqueue middleware and telemetry can observe and reject an operator retry using current code.
- Class renames, removed task classes, and undecodable arguments prevent retry but not metadata inspection.
- Retry has at-least-once failure behavior. A crash or dead-store deletion failure after pending persistence may leave both records; repeating retry can enqueue duplicate work.
- Concurrent retry/delete operations require operator coordination and do not provide cancellation or exactly-once guarantees.
- Out-of-process retry is durable but not immediate. Operators must restart the Rage server before the orphaned WAL is adopted.
- The public API does not depend on the private serialized context layout beyond a dedicated decoder for args/kwargs.
