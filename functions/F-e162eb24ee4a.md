<!-- Trusted provenance: F-e162eb24ee4a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e162eb24ee4a

## Purpose

F-e162eb24ee4a selects eligible records from a fixed table, marks them selected, and links the selected records in ascending index order. It reports the selection count and chain endpoints through caller-supplied output locations.

The function returns a 32-bit status distinguishing a fulfilled request, a nonempty partial selection, and failure or no selection. Dependency failures share the same return status as an empty selection. A failure returned after selection does not undo the table or output changes.

**Established behavior:** the table contains 128 candidate records. A record is eligible when its state byte equals exactly 1. Selection changes that byte to 3, supplies a chain link, and assigns a caller-provided 32-bit value.

**Inferred intent:** the two dependencies appear to bracket access to shared state. Their synchronization semantics are not established. This contract calls them the entry and exit dependencies without assuming that they acquire or release a lock.

## Interface

The function takes six arguments and returns a 32-bit value.

| Argument | Interpretation |
|---|---|
| Table base | A source-level 32-bit address identifying the record table and adjacent dependency-related storage. |
| Requested count | A signed 32-bit selection target. It is compared against the count after each selection. |
| Count output | Address of a writable 32-bit signed integer. |
| First-index output | Address of a writable 32-bit unsigned integer. |
| Last-index output | Address of a writable 32-bit unsigned integer. |
| Assignment value | A signed 32-bit value written into each selected record. The value −1 disables the scan. |

The ordinary caller-facing interpretation requires valid accessible storage, appropriate alignment for 32-bit accesses, and output locations that do not overlap one another or the table. The visible implementation does not validate these conditions. Overlap is not rejected, but it can change subsequent reads and therefore invalidate the ordinary count and chain guarantees described below.

### Table layout

Candidate indices range from 0 through 127. Each candidate occupies 20 bytes. Relative to the start of each candidate:

| Offset | Width | Role |
|---|---:|---|
| 8 | 1 byte | State: exactly 1 is eligible; selection writes 3. |
| 12 | 4 bytes | Next selected index, or the terminal value `0xFFFFFFFF`. |
| 16 | 4 bytes | Assignment value. |

The visible body does not directly access the first eight bytes of a record or bytes 9 through 11.

The candidate table occupies 2,560 bytes, ending immediately before base offset `0xA00`. Each dependency receives:

1. The 32-bit value read from base offset `0xA08`.
2. The address at base offset `0xA00`.

The meanings, size requirements, and ownership of that adjacent storage are unknown.

### Return values

| Value | Meaning when both dependencies return zero |
|---:|---|
| 0 | Selection reached the requested-count comparison. |
| 1 | The scan selected at least one record but exhausted the table before satisfying that comparison. |
| 2 | No record was selected, or the assignment value disabled scanning. |

A nonzero result from either dependency causes the function to return 2. Thus, return 2 is not sufficient to determine whether records were changed.

## Behavior

### Entry dependency and initialization

The function first reads the 32-bit dependency value at base offset `0xA08` and calls **F-f0216cff9365**, passing that value and the address at base offset `0xA00`.

If the dependency returns nonzero:

- The function returns 2.
- It performs no direct output initialization.
- It performs no direct candidate-table scan or modification.
- It does not call the exit dependency.

These statements do not exclude side effects performed by the entry dependency itself.

If the entry dependency returns zero, the function initializes the caller’s outputs in this order:

1. Writes 0 to the count output.
2. Writes `0xFFFFFFFF` to the first-index output.

It does not initialize the last-index output at this point.

If the assignment value equals −1, the function skips the scan and proceeds to the exit dependency. Consequently, with ordinary nonoverlapping storage, this path leaves count at 0, first index at `0xFFFFFFFF`, and the last-index output unchanged by the visible body.

### Candidate selection

For any assignment value other than −1, the function examines candidates in strictly ascending index order, beginning at 0 and ending no later than 127.

For each candidate it reads the state byte at record offset 8.

- A value other than 1 leaves that candidate unselected.
- A value of exactly 1 selects the candidate and causes its state byte to be written as 3.
- Other nonzero state values are not treated as eligible.

When selecting the first candidate, the function writes its index to the first-index output.

When selecting a later candidate, it updates the previously selected candidate:

1. Writes the current candidate’s index to the previous candidate’s next-index field.
2. Writes the assignment value to the previous candidate’s assignment field.

For each selected candidate, after the state change and the first-index or previous-record updates, the function:

1. Reads the current count output.
2. Increments that value by one and writes it back.
3. Treats the current candidate as the newest selected candidate.
4. Compares the incremented count against the requested count using signed comparison.

This ordering matters: the current candidate’s state has already changed before its predecessor is linked to it. The current candidate’s own next-index and assignment fields are not necessarily finalized immediately.

Under the ordinary nonoverlap precondition, the count starts at zero and increases at most 128 times. It therefore cannot overflow a signed 32-bit integer.

### Reaching the requested count

The stopping condition is:

> The requested count is less than or equal to the count after the current selection.

When this condition is met, the function:

1. Writes the current candidate’s index to the last-index output.
2. Writes `0xFFFFFFFF` to the current candidate’s next-index field.
3. Writes the assignment value to the current candidate’s assignment field.
4. Proceeds to the exit dependency with a provisional return value of 0.

No later candidate is examined.

Because the comparison occurs only after selecting a record, zero and negative requested counts do not request an empty selection. If at least one eligible record exists and scanning is enabled, the first eligible record is selected and the provisional result is 0.

A positive requested count above 128 cannot be met by an ordinary scan.

### Exhausting the table

If all 128 candidate positions have been considered without satisfying the stopping condition, the function reads the count output.

If that count is zero:

- It leaves the last-index output unchanged.
- It performs no chain termination or assignment-field write.
- Its provisional return value is 2.

If that count is nonzero:

1. It writes the most recently selected index to the last-index output.
2. It writes `0xFFFFFFFF` to that record’s next-index field.
3. It writes the assignment value to that record’s assignment field.
4. Its provisional return value is 1.

With ordinary nonoverlapping storage, every selected record therefore has the assignment value and a next-index field pointing either to the next selected record or to the terminal sentinel. Nonselected records are not directly modified.

### Exit dependency and final result

Every path following a zero return from the entry dependency calls **F-2bb3dbfa7286** once, including the disabled-scan and empty-selection paths.

Immediately before that call, the function reads the 32-bit value at base offset `0xA08` again. It passes the newly read value and the address at base offset `0xA00`.

The second dependency is not guaranteed to receive the same first argument as the entry dependency: the visible body reloads it rather than retaining the initial value.

- If the exit dependency returns zero, the function returns its provisional status.
- If the exit dependency returns nonzero, the function returns 2.

No rollback is visible. In particular, failure of the exit dependency can occur after state bytes, links, assignment fields, and all three outputs have been updated.

## Side effects

### Direct writes

After a successful entry-dependency result, the visible body always writes the count and first-index outputs, in that order.

The last-index output is written only if the scan selects at least one candidate under the ordinary nonoverlap precondition. It is not a reliable output on entry failure, a disabled scan, or an empty scan.

For selected candidates, the visible body writes:

- State byte: 3.
- Next-index field: another selected index or `0xFFFFFFFF`.
- Assignment field: the supplied assignment value.

There is no direct allocation, deallocation, ownership transfer, or record-payload copy in the visible body. The caller retains responsibility for supplying accessible storage. These observations do not establish what either dependency does with memory or other resources.

### Observation and concurrency

The visible ordering is not a promise of atomicity or safe concurrent observation. A selected record can temporarily have state 3 while retaining its old link or assignment field. A predecessor can also be linked before the successor’s terminal fields are finalized.

Whether such intermediate states are externally observable depends on synchronization, memory-model behavior, and dependency semantics that are not supplied.

### Aliasing and invalid storage

No null checks, bounds checks on caller-provided addresses, or overlap checks are visible.

If outputs overlap table fields, the dependency-related storage, or one another, writes can affect later eligibility checks, count reads, chain construction, or the argument reloaded for the exit dependency. Such cases do not have a simple “select the first matching records” guarantee.

Invalid or misaligned addresses have no established recoverable error behavior. The source provides no basis for promising status 2 instead of a memory fault or other platform-specific result.

## Dependencies

### F-f0216cff9365

**Role in this function:** entry gate.

**Arguments:** a 32-bit value loaded from base offset `0xA08`, followed by the address at base offset `0xA00`.

**Result handling:** zero permits initialization and selection; nonzero immediately produces status 2.

**Unknowns:** synchronization effects, memory effects, global-state changes, resource ownership, and the meaning of individual nonzero results.

### F-2bb3dbfa7286

**Role in this function:** exit gate after a successful entry result.

**Arguments:** a fresh 32-bit load from base offset `0xA08`, followed by the address at base offset `0xA00`.

**Result handling:** zero preserves the provisional status; nonzero replaces it with 2 without visible rollback.

**Unknowns:** the same categories as for the entry dependency. This function does not establish that the dependency restores state, releases a resource, or preserves any previously written output.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless stated otherwise, storage is valid and nonoverlapping, both dependencies return zero, and neither dependency changes relevant table or output state.

### Exact positive request

Given eligible candidates at indices 2, 5, and 9, a requested count of 2, and assignment value 7:

- Return value is 0.
- Count is 2; first index is 2; last index is 5.
- Candidates 2 and 5 have state 3 and assignment value 7.
- Candidate 2 points to index 5.
- Candidate 5 has terminal next-index value `0xFFFFFFFF`.
- Candidate 9 remains unchanged and is not examined after the stopping condition is reached.

### Partial selection

Given eligible candidates only at indices 4 and 127, requested count 3, and assignment value 12:

- The full candidate range is scanned.
- Return value is 1.
- Count is 2; first index is 4; last index is 127.
- The chain is 4 followed by 127, followed by the terminal sentinel.
- Both selected candidates have state 3 and assignment value 12.

### No eligible candidates

Given no state byte equal to 1 and assignment value other than −1:

- Return value is 2.
- Count is 0.
- First index is `0xFFFFFFFF`.
- Last-index storage retains its previous value.
- No candidate fields are directly written.

A state byte of 2, 3, or `0xFF` does not qualify.

### Zero or negative request

Given at least one eligible candidate, requested count 0 or −8, and assignment value 6:

- Exactly the first eligible candidate is selected.
- Return value is 0.
- Count is 1.
- First and last indices both identify that candidate.
- Its next-index field is terminal.

If no eligible candidate exists, the result instead follows the empty-selection case.

### Disabled scan

Given assignment value −1, regardless of requested count or candidate states:

- No candidate state bytes are read by the scan.
- No candidate fields are directly written.
- Count becomes 0 and first index becomes `0xFFFFFFFF`.
- Last-index storage is not directly written.
- The exit dependency is called.
- Return value is 2.

### Dependency failure

If the entry dependency returns nonzero, the visible body leaves all outputs and candidate fields untouched and returns 2.

If selection otherwise fulfills a request but the exit dependency returns nonzero, the function returns 2 while retaining the visible selection writes. Return 2 must therefore not be interpreted as “nothing changed.”

## Uncertainties

- **documented_only — Dependency semantics:** their implementations are unavailable. Locking, resource release, global preservation, and memory ownership cannot be established.
- **documented_only — Execution environment:** the representation supports 32-bit addresses and values, but byte order, alignment guarantees, and hardware memory-ordering behavior are not established.
- **documented_only — Address arithmetic:** ordinary behavior requires the table and adjacent fields to be addressable without invalid address computation. Behavior at address-space boundaries is not established.
- **documented_only — Aliasing:** no explicit no-alias declaration is shown. The ordinary contract uses nonoverlapping storage; overlapping cases remain order-sensitive rather than validated or rejected.
- **Inferred intent:** state 1 appears to mean available and state 3 appears to mean selected or reserved. Only the exact comparison and write are established.
- **Inferred intent:** the next-index fields form a selected-record chain. No downstream traversal or consumer requirements are available.
- **Unknown exceptional behavior:** dependency exceptions, nonlocal exits, and memory faults are outside the established return-status contract.

## Evidence

This contract is based solely on the supplied static function representation and its opaque dependency mapping.

Static evidence establishes:

- A 128-candidate scan with a 20-byte record stride.
- Eligibility by an exact one-byte comparison against 1.
- Selection state 3.
- Four-byte link and assignment fields at offsets 12 and 16.
- Signed requested-count comparison after selection.
- Separate first-index and last-index outputs.
- Terminal link value `0xFFFFFFFF`.
- Assignment value −1 as the scan-disable condition.
- Entry and exit dependency calls, with a fresh dependency-value read for each.
- Status override to 2 on a nonzero dependency result.
- No visible rollback.

No runtime tests have been performed. The acceptance examples express consequences of the static behavior under their stated preconditions, not observed execution results.
