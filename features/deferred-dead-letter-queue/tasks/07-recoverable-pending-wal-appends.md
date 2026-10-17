---
status: draft
---

# Preserve recoverable pending-WAL appends

## Goal

Keep complete pending-WAL records recoverable when an adopted WAL or recovered source ends with an interrupted append.

## Context

The existing pending WAL appends bytes after unfinished tails. Recovery can reject a later complete record because the record joins the unfinished line. Copying an unfinished recovery source can cause the same defect in the destination.

This task repairs the existing pending-WAL defect for ordinary enqueue and recovery. It does not repair the dead-task file. This task follows task 01 as deferred independent work. It does not precede or block [task 05](05-retry-dead-tasks.md). It implements REQ-31 and AC-12. Follow [ADR-002](../adr/002-dead-task-retry-lifecycle.md).

This task requires neither task 05's acceptance Boolean nor its optional backend flush. Ordinary enqueue retains timer-based synchronization.

## Requirements

- Before appending to an adopted pending WAL, establish its last complete newline-terminated record boundary. Preserve the complete prefix. Exclude only the unfinished suffix from subsequent appends. An entirely unfinished file has an empty complete prefix.
- Copy only the complete-record prefix of each recovered source. Do not join an unfinished suffix with another source or a new append. Preserve complete records and their existing order within each source. Preserve existing CRC validation, duplicate handling, task IDs, and add/remove recovery semantics.
- A successful append must write its complete intended record bytes. A short or interrupted write must not report success for an unfinished record. If Disk cannot complete a short write, raise `IOError` unless an original storage exception already exists. If an append fails, propagate the original storage exception. Do not acknowledge the pending task or schedule new work from that failed enqueue.
- After a failed append, establish a complete-record boundary before any later append. If Disk cannot establish that boundary, refuse the later append and propagate the storage failure. Do not remove complete prior records. A failed append can still leave a complete record; retain existing at-least-once uncertainty.
- If boundary detection, repair, reading, or copying fails, propagate the storage exception. Do not treat operational failures as ordinary corruption. Retain each source still needed for recovery.
- Keep a recovery or rotation source until the replacement data and required directory entries are confirmed durable. Keep required file descriptors open until copying and synchronization finish. A delay of `fsync_frequency` alone must not authorize source cleanup. If synchronization fails, retain the source and propagate the failure through the operation or callback that performs synchronization.
- Timer synchronization must preserve unsynchronized writes after failure. Successful synchronization must not clear dirty state belonging to newer writes. Boundary repairs and copied records must receive the same durability treatment as other pending writes.
- Use bounded buffers to find boundaries and copy records. Do not load a complete WAL or task collection for tail repair. Do not deserialize task contexts to find a physical boundary.
- Preserve the clean ordinary append path. Check adopted files before use and repair after failed writes. Do not scan the complete WAL before each known-clean append. Do not force a durability wait on every ordinary enqueue.
- Preserve worker ownership locks, storage format/version, backend interfaces, Ruby 3.3.0 support, and existing filesystem assumptions. Do not add a database backend, background adoption, or a new journal protocol.

## Design

Use the pending WAL's existing newline record framing. Establish safe destination boundaries before new appends and safe source boundaries before recovery copying. Repair only an unfinished final suffix. Keep complete bytes available for the existing CRC-based recovery reader.

Use the existing file ownership and recovery lifecycle. Keep boundary work outside application middleware and user code. Preserve Fiber interleaving while source copying or synchronization yields. No other operation may close a descriptor that the current copy or synchronization still needs.

This deferred task owns pending-WAL boundary repair and safe recovery/rotation source cleanup. Task 05 can proceed independently with active-WAL and directory synchronization under the existing storage lifecycle. Task 05 does not implement or depend on these future repairs. Neither task changes ordinary enqueue into synchronous persistence.

## Implementation constraints

- Do not change the dead-task file, lookup behavior, public retry API, or public enqueue return value in this task.
- Do not depend on task 05 or add its optional backend flush API here.
- Do not change task IDs, complete-record framing, or the storage version.
- Do not truncate complete prior records or repair arbitrary complete-record corruption.
- Preserve the primary exception during cleanup. Do not rescue `Exception`.
- Keep WAL access local and trusted. Do not expose task payloads in new diagnostics.
- Document the fix in the changelog and pending-WAL recovery documentation.

## Acceptance criteria

- [ ] TC-01 (AC-12) After adopting an unfinished WAL, a later successful append remains a separate recoverable record. Complete prior records remain recoverable.
- [ ] TC-02 (AC-12) Recovery copies only complete source prefixes. Later sources and appends remain separate recoverable records.
- [ ] TC-03 (AC-12) Interrupted and short writes cannot return success for unfinished records. A later append either restores safe framing or raises.
- [ ] TC-04 (AC-12) Storage failures propagate without deleting complete prior records or a required recovery source.
- [ ] TC-05 (AC-12) Source cleanup follows confirmed file and directory durability. Failed synchronization preserves the source and unsynchronized-write state.
- [ ] TC-06 (AC-12) Boundary repair uses bounded buffers. Known-clean ordinary appends require no complete-WAL scan or per-append forced synchronization.
- [ ] TC-07 (AC-12) Existing task IDs, CRC handling, file format, and worker ownership semantics remain compatible.

## Verification

- Cover adopted files with a complete prefix plus an unfinished suffix, an entirely unfinished file, and a clean empty file.
- Append valid add/remove records and reopen storage. Verify complete prior records and later records follow existing recovery semantics.
- Cover multiple recovery sources, unfinished source tails, and source boundaries crossing copy-buffer boundaries. Follow copying with ordinary appends.
- Simulate short writes, partial-write exceptions, and failures after complete bytes reach the WAL. Verify uncertainty without false success for unfinished records.
- Cover repair failure, failed recovery copy, failed file/directory synchronization, and cleanup failure. Verify required sources remain and primary exceptions survive.
- Exercise repeated rotation and recovery while Fibers append or synchronize. Verify required descriptor lifetimes and no timer-only cleanup.
- Verify failed timer synchronization remains eligible for another attempt. Cover newer writes during synchronization without lost dirty state.
- Verify bounded buffering without Marshal decoding or a complete-WAL scan on each known-clean append.
- Run focused Disk specs, `bundle exec rspec spec/deferred/backends/disk_spec.rb`, and `bundle exec rspec spec/deferred`.
- Run RuboCop for changed Ruby files and applicable documentation checks. Report unavailable checks.

## Result

- Implementation PR:
