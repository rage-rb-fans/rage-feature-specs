---
status: done
---

# Streaming read in the backend

## Goal

Give the backend contract a way to read dead tasks one record at a time, so that the public API of the later tasks can read a dead-tasks store of any size without loading it into memory.

## Context

The backend contract can read all dead tasks only through `list_dead_tasks`, which returns an Array of every record. In `Backends::Disk`, both `list_dead_tasks` and `find_dead_task` go through `DeadTasksStorage#read_records`: it puts every line of the file into a Hash under the store's lock and then decodes all of them. The store has no size limit, so one lookup by id holds about twice the store in memory.

This task adds the read that the feature builds on, and nothing public. The boundary with task 02 is the backend contract: task 02 calls `each_dead_task` and `find_dead_task` and knows nothing about files.

## Requirements

### `each_dead_task`

A new method of the backend contract, in `Backends::Disk` and `Backends::Nil`:

```ruby
backend.each_dead_task { |record| ... } # => nil
```

- It yields every dead task, oldest first (the order in which the tasks became dead), one record per call of the block.
- `record` is the Hash that `find_dead_task` returns, with the keys `id`, `task_class`, `attempts`, `enqueued_at`, `failed_at`, `exception_class`, `exception_message`, `backtrace`, and `context`. `context` is the `Marshal.dump` String of the context Array; it is not decoded.
- It yields each task id at most once. If the store holds two records with the same id, the newest one (the later failure) is yielded, at the place of that newest record in the order, and the older one is skipped. This is the rule `list_dead_tasks` has.
- The set of records is fixed when the call starts. A dead task added after that is not yielded by this call. A dead task removed after that is still yielded.
- No lock of the store is held while the block runs. `add_dead_task` and `remove_dead_tasks` called inside the block, by the same fiber or by another one, work as they do outside it.
- It holds one record at a time, plus one id and one position per stored record while it prepares the read, and one position per record while it yields. Its memory does not grow with the size of the stored records.
- `Backends::Nil#each_dead_task` yields nothing and returns `nil`.
- The method is always called with a block. A call without a block is not supported and returns no `Enumerator`.

### `find_dead_task`

The signature, the return value, and the behavior do not change: the record Hash, or `nil` when no record has the id. For an id that is in the store twice, it returns the newest record, as before, the same one that `each_dead_task` yields.

One thing changes in `Backends::Disk`: it no longer loads the store. It holds one line of the file at a time, plus the newest line with the given id, and decodes only that one.

### Failures

- **The store cannot be locked.** `each_dead_task` and `find_dead_task` raise `Rage::Deferred::DeadTasksLockTimeout` after the retries the store already uses. For `each_dead_task` this happens before the block is called for the first time; nothing is left open.
- **A line fails its checksum, or is not a dead-task entry.** It is skipped, and the records around it are yielded. The call prints one line: `WARNING: Detected <n> corrupted dead-lettered task(s)`. `each_dead_task` prints it before it yields the first record.
- **The file ends with half of an entry.** Every entry is one line that ends with a newline. If a process dies in the middle of `add_dead_task`, only the first part of its line is in the file, with no newline at the end. The store already handles this on the write side: the next `add_dead_task` cuts that half line off before it writes. A read must handle it too, because it can run before that next write. The read stops at the last newline of the file: the half line is not yielded and not counted as corrupted. If another process then adds a dead task while the block runs, its entry is written where the half line was; this call does not yield it, because it was added after the call started.
- **A record passes its checksum but cannot be decoded** (`ArgumentError`, `NameError`, `TypeError` from `Marshal.load`). The call prints `ERROR: Can't deserialize the dead-lettered task with id <id>: (<class>) <message>`, skips the record, and goes on. For a repeated id only the newest record is tried: if it cannot be decoded, the id is not yielded, and `find_dead_task` returns `nil` for it.
- **The block raises, or ends the call with `break`.** The error reaches the caller unchanged. The file that the call opened is closed in both cases.
- **The call is interrupted halfway** (the process dies while the block runs). The call writes nothing, so the store is the same as before the call.
- **It runs twice, or two fibers run it at the same time.** Every call opens the store again and has its own set of records and its own list of positions. Two calls do not affect each other.

### What this task does not do

- No public API: `Rage::Deferred.dead_tasks`, `DeadTasks`, and `DeadTask` are task 02.
- `list_dead_tasks` keeps its code and its behavior, in both backends.
- The file format, `add_dead_task`, and `remove_dead_tasks` do not change. A read never repairs or rewrites the file.
- No `Enumerator` from `each_dead_task`, and no `limit` or `offset`.
- The whole file is still read on the reactor for every call: once by `find_dead_task`, twice by `each_dead_task`. Only the memory use gets better.

## Design

All changes are in `lib/rage/deferred/backends/disk.rb` and `lib/rage/deferred/backends/nil.rb`.

### Contract header (`disk.rb`)

The record is described once, and the other reads point to it:

```ruby
# * `add_dead_task` - called when a task has exhausted its retries or aborted them;
# * `each_dead_task` - yield every dead task, oldest first, one at a time; for an id that is stored more than once,
#   only the newest task; the set of dead tasks is fixed when the call starts, and no lock of the storage is held
#   while a task is yielded;
#   each dead task is a Hash with the keys `id`, `task_class`, `attempts`, `enqueued_at`, `failed_at`,
#   `exception_class`, `exception_message`, `backtrace`, and `context`, where `context` is `Marshal.dump` of the
#   `Rage::Deferred::Context` Array;
# * `list_dead_tasks` - return a list of dead tasks as the same Hashes, newest first; for an id that is stored
#   more than once, the newest task;
# * `find_dead_task` - return a single dead task as the same Hash, or `nil` if there is no such task; for an id
#   that is stored more than once, the newest task;
# * `remove_dead_tasks` - permanently delete dead tasks;
```

### `Backends::Disk`

```ruby
# Yield every dead-lettered task, oldest first, one record at a time.
# @yieldparam record [Hash] the same Hash that `find_dead_task` returns
# @return [void]
# @raise [Rage::Deferred::DeadTasksLockTimeout] if the dead-tasks store cannot be locked
def each_dead_task(&)
  @dead_tasks_storage.each(&)
end
```

`find_dead_task(id)` keeps its body (`@dead_tasks_storage.find(id)`).

### `DeadTasksStorage`

Public methods:

| Method | Returns | Change |
| --- | --- | --- |
| `each { \|record\| }` | `nil` | new |
| `find(id)` | `Hash` or `nil` | new body, same behavior |
| `list(limit: nil, offset: 0)` | `Array<Hash>` | none; still uses `read_records` |

Private methods:

- `with_snapshot { |storage, limit| }` (new) — the one place that fixes the set of records. Returns what its block returns.
- `each_entry(storage, limit) { |id, entry, position| }` (new) — one pass over the set of records. `position` is the byte position at which the entry starts. Returns `nil`.
- `complete_size(storage)` (new) — the position just after the last newline of the file: the file size when the file ends with a newline, `0` for an empty file or a file with no newline. It is the backward scan that `repair_torn_tail` has today, moved out of it.
- `repair_torn_tail(storage)` (changed, same behavior) — `storage.truncate(size)` when `size = complete_size(storage)` is smaller than the file size.
- `decode_entry(entry)` (new) — splits one entry (without its newline) into its id and its payload, and returns `Marshal.load` of the payload; on `ArgumentError`, `NameError`, or `TypeError` it prints the `ERROR:` line and returns `nil`. Only `each` and `find` call it.
- `read_records` — not changed. `list` is its only caller after this task.

`with_snapshot`, step by step:

1. Under `with_lock("read tasks from")`: open the live file with `File::RDONLY | File::BINARY` and compute `limit = complete_size(storage)`. The lock is released when the block of `with_lock` returns. If `with_lock` raises, no file is open.
2. `yield storage, limit`, outside the lock.
3. `ensure`: close the file if it was opened.

`each_entry`, step by step:

1. `storage.rewind`, `position = 0`. While `position < limit`: `entry = storage.gets` (stop if it is `nil`); `next_position = position + entry.bytesize`; `entry.chomp!`; `id = entry_id(entry)`; if `id` is `nil`, add one to the count of corrupted lines; otherwise `yield id, entry, position`; then `position = next_position`. The position is counted from the bytes read and never asked from the file: `IO#pos` and `IO#seek` drop the read buffer, so one of them per entry makes every entry cost a read of a full buffer.
2. After the loop, if the count is not zero, print the `WARNING:` line.

Why the read is a fixed set without a lock: the bytes before `limit` never change. `add` appends, and its `repair_torn_tail` cuts only bytes after `limit`, because `limit` is the end of the last complete entry. `remove` writes a new file and renames it over the live path, which leaves the opened file complete. `limit` is at an entry boundary, so `gets` never returns a part of an entry.

`each` reads the set of records twice. The first pass finds, for every id, where its newest entry starts; the second pass reads only those entries and yields them:

```ruby
def each
  with_snapshot do |storage, limit|
    index = {}

    each_entry(storage, limit) do |id, _, position|
      # a repeated id moves to the place of its newest entry
      index.delete(id)
      index[id] = position
    end

    # the ids are not needed any more; keep one Integer per record
    positions = index.values
    index = nil

    next_position = nil

    positions.each do |position|
      # a seek drops the read buffer, so it is made only to jump over a skipped entry
      storage.seek(position) if position != next_position
      entry = storage.gets
      next_position = position + entry.bytesize
      entry.chomp!
      record = decode_entry(entry)

      yield record if record
    end
  end

  nil
end
```

`find(id)` reads the set of records once and keeps the newest entry with the id:

```ruby
def find(id)
  with_snapshot do |storage, limit|
    found = nil
    each_entry(storage, limit) { |entry_id, entry| found = entry if entry_id == id }

    decode_entry(found) if found
  end
end
```

### `Backends::Nil`

```ruby
def each_dead_task
end
```

### Blocking I/O

No new kind of call, but one more read. `find` reads the dead-tasks file once on the reactor, as `read_records` does today. `each` reads it twice: the first pass runs in one piece before the first record is yielded, and the second is split by the calls of the block. Both passes are outside the store's lock. The lock is the existing `with_lock` (`LOCK_NB` with a limited number of retries that `sleep`), held only for one `open` and at most a few small reads at the end of the file. No `fsync`, no blocking `flock`, no thread, no native call beyond `Zlib.crc32` and `Marshal.load` per record, which the current read already makes.

## Implementation constraints

- Follows [ADR-003](../adr/003-streaming-read.md).
- The backend contract grows by `each_dead_task` only. No other method, argument, or return value is added to it.
- `DeadTasksStorage::STORAGE_VERSION` stays `"0"`, and entries written by Rage 1.28.0 are read.
- The block of `each_dead_task` never runs inside `with_lock`. `@locked` is `false` and the `flock` is released before the first `yield`.
- No collection of lines or records is built. The only state that grows during a call is in `each`: the Hash of id to position during the first pass, and the Array of positions after it. `find` keeps one line.
- `find` compares the id of an entry before decoding it; entries with other ids are never passed to `Marshal.load`.
- A read path never truncates, repairs, or `fsync`s the file. Rejected: calling `repair_torn_tail` at the start of a read — it makes a read a write and needs the file opened for writing.
- Rejected: `limit = storage.size`. With an interrupted append at the end of the file, a later `add` writes its entry in place of it, and an entry shorter than the cut bytes would end before `limit` and be yielded although it was added after the call started.
- Rejected by the engineer: the first record of a repeated id wins. It needs one pass only, but `each_dead_task` and `find_dead_task` would then show an older failure than `list_dead_tasks` (ADR-003).
- The second pass of `each` does not check the checksum again and does not read the entries that lost; it seeks to each kept position.
- Rejected: building `list` on `each_entry`, and using `decode_entry` in `read_records`. `list_dead_tasks` and the code under it are not changed in this feature, so a later change to the new read cannot change the old one.
- Rejected: removing `list_dead_tasks`, `DeadTasksStorage#list`, and `read_records` in this task. The method was released in 1.28.0, and ADR-003 leaves its removal to a separate proposal.
- The lock file stays the separate, never renamed `…dead_tasks.lock`; the data file is not locked.
- `# frozen_string_literal: true` stays in both files; nothing newer than Ruby 3.3.0 (anonymous block forwarding `&` is 3.1).
- No `CHANGELOG.md` entry in this task: it changes nothing an app can call. The feature's entry is written with the public API.

## Acceptance criteria

- [ ] TC-01 (core) When `each_dead_task` is called on a Disk backend that holds three dead tasks, it yields their three record Hashes, oldest first.
- [ ] TC-02 (core) If `add_dead_task` is called inside the block of `each_dead_task`, that call of `each_dead_task` does not yield the added task.
- [ ] TC-03 (core) If `remove_dead_tasks` removes a task inside the block of `each_dead_task` before the task was yielded, that call still yields it.
- [ ] TC-04 (core) When `remove_dead_tasks` is called inside the block of `each_dead_task`, it returns the number of removed tasks without raising `DeadTasksLockTimeout`.
- [ ] TC-05 (core) If the store holds two records with the same task id, `each_dead_task` yields only the one that was written last.
- [ ] TC-06 (core) If the store holds two records with the same task id, `find_dead_task` returns the one that was written last.
- [ ] TC-07 (core) When `each_dead_task` is called on `Backends::Nil`, the block is never called.
- [ ] TC-08 (edge) If the dead-tasks store cannot be locked, `each_dead_task` raises `Rage::Deferred::DeadTasksLockTimeout` before the block is called.
- [ ] TC-09 (edge) If a line of the store fails its checksum, `each_dead_task` yields the records of the other lines.
- [ ] TC-10 (edge) If the store ends with an entry that has no final newline, `each_dead_task` yields the complete records before it and nothing for that entry.
- [ ] TC-11 (edge) If the block of `each_dead_task` raises or ends the call with `break`, the file that the call opened is closed.
- [ ] TC-12 (edge) When `find_dead_task` is called with an id that is not in the store, it returns `nil`.
- [ ] TC-13 (edge) If the store holds records A, B, and then a second record of A's id, `each_dead_task` yields B before A.

## Verification

Tests to add, in `spec/deferred/backends/disk_spec.rb`, inside `describe "dead tasks"`:

- `describe "#each_dead_task"` — TC-01 to TC-05, TC-08 to TC-11, and TC-13. For TC-02, a second example with an interrupted append at the end of the file and a shorter entry added inside the block, which also checks that nothing is printed. For TC-10, a second example whose final entry is complete and valid except for its newline. For TC-09, also the `WARNING:` line. For TC-01, the keys of a yielded Hash and that `context` is a String.
- `describe "#find_dead_task"` — TC-06 and TC-12, and one example that a stored id returns its record Hash.
- TC-07 — `spec/deferred/backends/nil_spec.rb` (a new file).

No behavior that an existing example checks is changed, so no existing example changes. The examples that call `list_dead_tasks` must pass unchanged.

The memory limits (one record at a time; `find` decodes only the matching record) have no criterion: they are checked in review against the Implementation constraints.

Commands:

- `bundle exec rspec spec/deferred/backends`
- `bundle exec rubocop lib/rage/deferred/backends spec/deferred/backends`

**Core tests**

Existing examples changed: none. Criteria kept whole: none.
TC-06 and TC-07 are left for the edge step by the engineer's decision: the test of TC-06 passes before the code, and the test of TC-07 passes against an empty `Backends::Nil#each_dead_task`.

<!-- rage-flow:core-tests begin — written by a script, do not edit -->
- TC-01: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task yields the record Hashes of three dead tasks, oldest first" — red: assertion
- TC-01: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task yields Hashes with the contract keys and an undecoded context String" — red: assertion
- TC-02: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task does not yield a task added inside the block" — red: assertion
- TC-03: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task still yields a task removed inside the block before it was yielded" — red: assertion
- TC-04: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task lets remove_dead_tasks inside the block return the removed count" — red: assertion
- TC-05: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task yields only the last written record of a repeated id" — red: assertion
- TC-06: left for the edge step: find_dead_task keeps its behavior, so the test passes before the code
- TC-07: left for the edge step: the test passes against an empty Backends::Nil#each_dead_task
<!-- rage-flow:core-tests end -->

**Edge tests**

Existing examples changed: none.
Ran (first round): `bundle exec rspec spec/deferred/backends` — 61 examples, 0 failures, none skipped or pending.
Ran (review round, 2026-10-02): `bundle exec rspec spec/deferred/backends/disk_spec.rb` — 61 examples, 0 failures, none skipped or pending. All 11 Disk examples below ran, together with the 6 core examples and the 44 existing ones. `spec/deferred/backends/nil_spec.rb` was not changed and not run again in this round; `rage-flow revert-check` ran its example at the base.
Not run: none. No example of this task needs external services.
The second example of TC-06 ("returns the record Hash of a stored id") is the one Verification plans without a criterion; it is listed under TC-06 because the map needs an ID.
TC-08 and the first example of TC-11 fail at the base only because `each_dead_task` does not exist there: `raise_error` reports the `NoMethodError` as a failed expectation. This is the same case as "not proven (NoMethodError)".
What the tests cannot show: the memory limits (checked in review, as Verification says), and a second process that adds or removes a dead task while the block runs — the tests do both from the same process. A dead task added during the first pass of `each_dead_task` cannot be tested without a stub on a private method that the Design does not name, so no example does it: an add inside the block always comes after the first pass.
Review round: the TC-02 example with the interrupted append now also checks that nothing is printed (the half line is not counted as corrupted), and TC-10 has a second example. Both, and the first TC-10 example, fail when the read stops at the file size and not at the last newline; this was checked with a scratch patch loaded with `rspec -r`, not in the checkout. At the base they stay "not proven (NoMethodError)".
Review findings handled: F2, F4, F6
Review mode changed: spec/deferred/backends/disk_spec.rb; code changed: no

<!-- rage-flow:edge-tests begin — written by a script, do not edit -->
- TC-02: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task does not yield a shorter task written over an incomplete final entry inside the block" — revert: not proven (NoMethodError)
- TC-06: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task returns the last written record of a repeated id" — revert: passes without the change — old behavior is kept: find_dead_task keeps its return value, only its memory use changes
- TC-06: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task returns the record Hash of a stored id" — revert: passes without the change — old behavior is kept: find_dead_task keeps its return value, only its memory use changes
- TC-07: spec/deferred/backends/nil_spec.rb — "Rage::Deferred::Backends::Nil#each_dead_task never calls the block" — revert: not proven (NoMethodError)
- TC-08: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task raises before the block is called when the store cannot be locked" — revert: fails
- TC-09: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task skips a line that fails its checksum and yields the records around it" — revert: not proven (NoMethodError)
- TC-10: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task yields nothing for a final entry without a newline" — revert: not proven (NoMethodError)
- TC-10: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task yields nothing for a final entry that is complete except for its newline" — revert: not proven (NoMethodError)
- TC-11: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task when the block does not run to the end closes the file and passes the error on when the block raises" — revert: fails
- TC-11: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task when the block does not run to the end closes the file when the block ends the call with break" — revert: not proven (NoMethodError)
- TC-12: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task returns nil for an id that is not in the store" — revert: passes without the change — old behavior is kept: find_dead_task keeps its return value, only its memory use changes
- TC-13: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task yields a repeated id at the place of its last written record" — revert: not proven (NoMethodError)
<!-- rage-flow:edge-tests end -->

**Review outcome**

Second full review on 2026-10-02, after `task:build` handled the three test findings of the first one. Reviewers: A (runtime correctness), B (contract, cost, surface, and tests), C (code against the Design); none skipped. A and C returned no findings. Reviewed tree 5cc683e, base b8c9a6c. The finding numbers below are those of this review.
Report (a local file): `.workflow/evidence/deferred-dead-tasks-api/01-streaming-read-review.md`.
First full review (same day, reviewed tree 7b2dddf): it applied two fixes to the read cost — the second pass of `each` seeks only to jump over a skipped entry, and `each_entry` counts the position from the bytes read and no longer calls `storage.pos`; the Design was patched to match. It rejected a finding about the contract header text, which is the text the approved Design names. Its three test findings (TC-02, TC-10, and the map entry of TC-07) were handled by `task:build`.
Applied: F3 — the class comment of `DeadTasksStorage` now says that `each` and `find` read after the lock is released. Comment text only; the Design does not change.
Rejected: F2 — `each_dead_task` without a block differs between Disk (`LocalJumpError` at the first yield) and Nil (`nil`); the Requirements say that a call without a block is not supported.
Unmet: none
- Decided: F1 — rejected by the engineer on 2026-10-02; the gap is accepted. No example adds a dead task during the first pass of `each_dead_task`, so no example shows directly that a shorter entry written over the half line is not yielded. TC-02 as written (an add inside the block) is proven, and the rejected `limit = storage.size` is guarded: with a scratch patch that uses it, three examples fail (two on the `WARNING:` line, the TC-10 example with a complete entry on the yielded ids).
Examples that did not run: none. All 18 mapped examples ran and passed; none was skipped or pending, and no example of this task needs external services.
Checks: `bundle exec rubocop` — no offenses. `bundle exec yardoc --fail-on-warning` — one warning outside this task, `lib/rage/sse/message.rb` ("Undocumentable Struct assignment to Rage::SSE::Message"); not fixed.

<!-- rage-flow:final-run begin — written by a script, do not edit -->
Ran: bundle exec rspec spec/deferred/backends/disk_spec.rb spec/deferred/backends/nil_spec.rb — exit 0
Ran: bundle exec rspec spec/deferred/backends — exit 0
Ran: bundle exec rubocop lib/rage/deferred/backends spec/deferred/backends — exit 0
- TC-01: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task yields the record Hashes of three dead tasks, oldest first" — passed
- TC-01: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task yields Hashes with the contract keys and an undecoded context String" — passed
- TC-02: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task does not yield a task added inside the block" — passed
- TC-02: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task does not yield a shorter task written over an incomplete final entry inside the block" — passed
- TC-03: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task still yields a task removed inside the block before it was yielded" — passed
- TC-04: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task lets remove_dead_tasks inside the block return the removed count" — passed
- TC-05: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task yields only the last written record of a repeated id" — passed
- TC-06: left for the edge step: find_dead_task keeps its behavior, so the test passes before the code
- TC-06: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task returns the last written record of a repeated id" — passed
- TC-06: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task returns the record Hash of a stored id" — passed
- TC-07: left for the edge step: the test passes against an empty Backends::Nil#each_dead_task
- TC-07: spec/deferred/backends/nil_spec.rb — "Rage::Deferred::Backends::Nil#each_dead_task never calls the block" — passed
- TC-08: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task raises before the block is called when the store cannot be locked" — passed
- TC-09: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task skips a line that fails its checksum and yields the records around it" — passed
- TC-10: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task yields nothing for a final entry without a newline" — passed
- TC-10: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task yields nothing for a final entry that is complete except for its newline" — passed
- TC-11: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task when the block does not run to the end closes the file and passes the error on when the block raises" — passed
- TC-11: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task when the block does not run to the end closes the file when the block ends the call with break" — passed
- TC-12: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #find_dead_task returns nil for an id that is not in the store" — passed
- TC-13: spec/deferred/backends/disk_spec.rb — "Rage::Deferred::Backends::Disk dead tasks #each_dead_task yields a repeated id at the place of its last written record" — passed
Open: none
Tested: HEAD b8c9a6c, files digest d9de859 (lib/rage/deferred/backends/disk.rb, lib/rage/deferred/backends/nil.rb, spec/deferred/backends/disk_spec.rb, spec/deferred/backends/nil_spec.rb)
<!-- rage-flow:final-run end -->

## Result

- Implementation PR: [rage-rb/rage#403](https://github.com/rage-rb/rage/pull/403)
- Merge commit: [49f690e](https://github.com/rage-rb/rage/commit/49f690e8075ffce7af4df7b422cac39a4630aec9)
