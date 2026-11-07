<!-- Trusted provenance: F-efcf69a135e2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-efcf69a135e2

## Purpose

F-efcf69a135e2 prepares shared operation state, waits for a gate dependency to permit progress, performs an ordered sequence of register-like accesses, and waits for a completion condition. On a normally terminating path, it calls a final dependency, replaces a shared 32-bit value with that dependency’s result, and returns either **0** or **96**.

The return rule depends exclusively on the final status-byte read: **12 produces 0; every other byte value produces 96**. An earlier completion condition does not independently establish success.

A significant initialization distinction is established by the supplied body: the operation-record reference is read **twice**. The first read captures the base used for later record accesses. The second read supplies the destination for the initial status write. Those reads are not established to return the same reference.

The register-like accesses and polling suggest coordination with a device or asynchronously updated state. That is inferred intent, not an established hardware or protocol contract.

## Interface

### Arguments and result

| Input | Represented width | Established use |
|---|---:|---|
| Address/control value | Signed 32-bit integer | Shifted left by one bit; the result is written to register-block offset 12. |
| Payload A | 32 bits | Stored without interpretation at captured operation-record offset 8. |
| Payload B | 32 bits | Stored without interpretation at captured operation-record offset 12. |

The returned value is a 32-bit quantity. On normal return, its only possible values are 0 and 96.

The payloads are opaque values. The supplied body does not establish whether they represent addresses, lengths, identifiers, or ordinary scalars.

### Shared references and storage

**Operation-record reference source.** This globally referenced value is read twice at entry:

- The **first read** supplies the captured record base used for all subsequent record-field accesses except the initial status write.
- The **second read** supplies the base used solely for that initial status write.

The following layout applies to the accessed record storage:

| Byte offset | Width | Access |
|---:|---:|---|
| 1 | 8 bits | Initial value 10 is written through the second reference read. Later status reads and any write of 11 use the first, captured reference. |
| 4 | 32 bits | Cleared through the captured reference. |
| 8 | 32 bits | Receives Payload A through the captured reference. |
| 12 | 32 bits | Receives Payload B through the captured reference. |

The two references may designate the same storage, but that is not guaranteed by this body. No other record fields are explicitly accessed.

**Shared handle slot.** Its reference is captured during initialization. Its 32-bit contents are read on every gate attempt and replaced during finalization.

**Register-like block.** Its reference is captured before the shared-handle-slot reference.

| Byte offset | Width | Access |
|---:|---:|---|
| 0 | 32 bits | Read, combined with mask 128, then written back. |
| 4 | 32 bits | Written with 240. |
| 12 | 32 bits | Written with the shifted address/control value. |
| 16 | 32 bits | Repeatedly read until zero at three distinct stages. |

**Polling-token source.** A globally stored 32-bit value is read once after the write of 240. That captured value is used for later polling dependency calls.

### Preconditions and ownership

Normal execution requires valid, suitably accessible storage for every explicit access. In particular, if the two initial operation-record references differ, both destinations must support the accesses made through them. Validity of the first reference alone is insufficient for the initial status write.

The body performs no explicit pointer, alignment, size, or argument validation. It establishes no non-aliasing requirement or guarantee.

No allocation, payload-buffer copy, or explicit memory release is visible. Dependency ownership and lifetime effects remain unknown. The final dependency is not established to release a resource.

## Behavior

### 1. Read references and initialize state

The initial observable sequence is:

1. Read the operation-record reference source and retain that base for later record accesses.
2. Read the same reference source again.
3. Write the 8-bit value **10** at offset 1 from the base obtained by the second read.
4. Capture the register-block reference.
5. Capture the shared-handle-slot reference.
6. Write the 32-bit value **0** at offset 4 of the captured record.
7. Write Payload A at offset 8 of the captured record.
8. Write Payload B at offset 12 of the captured record.

The second operation-record reference is not retained for later record accesses.

Consequently, initialization does **not** unconditionally establish that the captured record’s status is 10. If the reference changes between the first two reads, status initialization can target different storage from the subsequent field writes, polling reads, completion-status writes, and final status read.

Even when both reads yield the same base, dependencies or external activity may change its status later. No rollback is visible if execution subsequently stalls or a dependency fails to return.

### 2. Wait for the gate

Each gate attempt reads the current 32-bit contents of the captured shared handle slot and calls **F-01** with that value and **100**.

A nonzero result permits progression to register access. The particular nonzero value is otherwise unused.

For a zero result:

- Call **F-02**.
- If F-02 returns nonzero, call **F-03** with **0** and **1**.
- Retry F-01.

If F-02 returns zero, F-03 is skipped.

The slot contents are reloaded for each F-01 call. A dependency or external actor can therefore alter the handle passed on a later attempt. Replacing the global reference to the slot does not itself redirect the already captured reference.

There is no explicit retry count, total deadline, or cancellation check. The constant 100 is a dependency argument, not an established limit on this entire phase.

### 3. Perform the register sequence

After the gate returns nonzero, the function performs these accesses through the captured register-block reference:

1. Read the word at offset 16 repeatedly until a read yields zero.
2. Read the word at offset 0, set bit 7 in the read value, and write the resulting 32-bit value to offset 0.
3. Read offset 16 repeatedly until a read yields zero.
4. Write the address/control value shifted left by one bit to offset 12.
5. Read offset 16 repeatedly until a read yields zero.
6. Write the 32-bit value **240** to offset 4.

Each wait includes at least one read. Any nonzero 32-bit value continues that wait. The three waits are distinct: a zero observed at one stage does not remove a later wait.

The offset-0 update is a read-modify-write sequence, not an established atomic bit-set operation. It preserves the bits from its particular read while setting mask 128. An intervening change to the word is not explicitly protected from being overwritten.

The waits contain no explicit dependency calls, yielding operation, status check, or timeout.

The shift has a 32-bit destination. For a nonnegative input whose doubled value fits the represented signed range, the written value is exactly twice the input. Behavior for negative values or signed-range overflow remains qualified under **Uncertainties**; a portable wraparound rule is not established.

### 4. Capture the polling token

After writing 240, the function reads the polling-token source once. Subsequent F-05 calls use that captured 32-bit value.

Changes to the token source after this read do not cause a reload during this invocation.

### 5. Wait for completion

Each completion iteration begins by reading the status byte at offset 1 of the **first, captured operation-record base**.

This read is interpreted as unsigned. A value greater than **10** leaves the waiting phase directly, without an explicit write of 11. Thus values 11 through 255 all satisfy this exit condition; only the later comparison with 12 determines the return value.

If status is at most 10, the function calls F-02.

#### F-02 returns nonzero

The function calls **F-04** with **61** and **100**.

- Result **5** causes an 8-bit write of **11** to the captured record’s status, followed by finalization.
- Any other result returns execution to the next status check.

F-05 and F-ca2ffda9d056 are not called on this branch.

There is no status recheck between F-04 and the write of 11. If F-04 or external activity changes status to 12 before F-04 returns 5, the explicit write still replaces that status with 11.

#### F-02 returns zero

The function calls:

1. **F-05**, with the captured polling token.
2. **F-ca2ffda9d056**, with **0**.

Only after the second call does it test the previously obtained F-05 result.

- Result **31** causes an 8-bit write of **11** to the captured record’s status, followed by finalization.
- Any other result returns execution to the next status check.

The helper is called even when F-05 returned 31. Its own return value is unused. Changes it makes to shared storage can still affect subsequent behavior.

There is no status recheck before the write of 11. A status update made by either dependency or external activity can therefore be overwritten on the result-31 path.

### 6. Finalize and return

Every normally terminating completion path converges on this sequence:

1. Call **F-06**, with no explicit arguments.
2. Store its 32-bit result in the captured shared handle slot.
3. Read the status byte from the captured record again.
4. Return **0** if that byte equals **12**; otherwise return **96**.

The final status read occurs after both F-06 and the handle-slot write. Earlier status observations do not fix the result.

Writing 11 on a completion path therefore does not irrevocably select 96. A later change to 12 can select zero. Conversely, an earlier observation of 12 does not guarantee zero if the final read observes another value.

## Side effects

The explicit writes affect:

- Status at offset 1 of the record selected by the **second initial reference read**, with value 10.
- Offsets 4, 8, and 12 of the record selected by the **first initial reference read**.
- Potentially, status at offset 1 of that first, captured record, with value 11.
- Register-block offsets 0, 12, and 4, in that order.
- The captured shared handle slot during finalization.

If the two initial record references differ, there may be two separate record objects affected. The first record’s status need not receive any initialization write from this invocation.

Later record accesses do not reload the operation-record reference source. Likewise, later register and handle-slot accesses use their captured references. Mutating referenced storage can affect later reads; replacing the corresponding global reference alone does not redirect those accesses.

Repeated register or status reads may themselves be observable when the backing storage has device-specific semantics. No explicit memory barrier, atomic operation, interrupt masking, cache maintenance, or lock operation appears in the body. Synchronization supplied by dependencies is unknown.

No dependency is assumed to preserve globals, retain or discard arguments, or leave memory ownership unchanged. Its changes can affect subsequent explicit reads. Possible aliasing also means the final handle write, or another explicit write, could affect storage later read through a different captured reference.

The function may fail to return because the gate never succeeds, a register wait never observes zero, completion never occurs, or a dependency does not return. No separate error return for those cases is visible.

## Dependencies

These are opaque function IDs; their descriptions establish call interactions, not external API meanings.

| Function ID | Established interaction |
|---|---|
| **F-01** | Receives the current 32-bit handle-slot contents and 100. Zero retries; nonzero advances. |
| **F-02** | Called after unsuccessful gate attempts and during completion waiting. Zero versus nonzero selects the following action. |
| **F-03** | Receives 0 and 1 after a failed gate attempt when F-02 is nonzero. Result unused. |
| **F-04** | Receives 61 and 100 on the nonzero F-02 completion branch. Result 5 selects the status-11 exit. |
| **F-05** | Receives the captured polling token on the zero F-02 completion branch. Result 31 selects the status-11 exit after the helper call. |
| **F-ca2ffda9d056** | Receives 0 after every F-05 call. Result unused. |
| **F-06** | Called once on each normally terminating path. Its 32-bit result replaces the captured handle slot before final status evaluation. |

Descriptions such as acquisition, delay, event wait, polling, or release remain interpretations rather than established dependency contracts.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless stated otherwise, storage is valid, dependencies return, and there are no additional interfering writes or aliases.

### Different initial record references

Suppose the first operation-record reference read identifies record A and the second identifies distinct record B.

- B’s status receives 10.
- A’s fields at offsets 4, 8, and 12 receive zero, Payload A, and Payload B.
- Completion status reads and any write of 11 use A.
- The final return comparison uses A, not B.

If A’s status is already 12 and remains so, the completion phase exits directly and the function returns zero after finalization. B’s initialized status of 10 does not force A through a polling branch.

### Stable initial reference

If both initial reads identify the same record, its status receives 10 before its other initialization writes. This conclusion depends on reference stability across those two reads, not on the first capture alone.

### Gate retry observes changed contents

If F-01 returns zero, F-02 returns nonzero, and F-03 changes the captured handle slot, the next F-01 call receives the changed value and 100. Register programming waits for a nonzero F-01 result.

If F-02 instead returns zero, that attempt does not call F-03.

### Ordered register writes

For address/control input 9, if each register wait eventually observes zero:

- Offset 0 receives its read value with mask 128 set.
- Offset 12 receives 18.
- Offset 4 receives 240.

A separate zero observation at offset 16 precedes each write.

### Direct completion

If the first completion status read from the captured record observes 12, no completion-phase F-02, F-04, or F-05 call occurs. F-06 still runs and its result is stored. A final status of 12 produces zero.

An otherwise identical case with status remaining 13 produces 96.

### Completion through F-04

If captured-record status is at most 10, F-02 returns nonzero, and F-04 returns 5, the function writes status 11 and finalizes. A final status of 11 produces 96.

If F-04 sets status to 12 before returning 5, the subsequent write of 11 overwrites that change.

### Completion through F-05 and the helper

If F-02 returns zero and F-05 returns 31, F-ca2ffda9d056 still receives 0. Status is then written as 11, even if the helper has just changed it to 12.

If F-05 instead returns a value other than 31 and the helper sets status to 12, the next status check can exit directly without writing 11.

### Finalization changes the outcome

If waiting ends with status 11 but F-06 changes it to 12, and the handle-slot write does not alter that status, the final result is zero.

Conversely, a waiting-phase observation of 12 followed by a final status of 10 produces 96. F-06’s numeric result determines the stored handle value, not the return code directly.

## Uncertainties

The following remain **documented_only**:

- Stability of the operation-record reference across its two initial reads. Equality of those references is not established.
- The meanings of fields, offsets, constants, status values, and return code 96.
- Whether backing storage is ordinary memory, device registers, or another shared mechanism.
- Atomicity, visibility, and memory-ordering guarantees for asynchronous updates.
- Exact target arithmetic for negative address/control inputs or signed left shifts exceeding the representable range. A low-32-bit shift interpretation is not established as a portable rule.
- Dependency behavior beyond the visible arguments, result comparisons, and result uses.
- Whether 100 represents a duration, count, mode, or other quantity.
- Resource ownership, argument retention, and acquisition or release semantics.
- Aliasing among referenced objects, including overlap between status storage and the handle slot.
- Reentrancy and safety of concurrent invocations.
- Behavior following invalid accesses, device faults, dependency failures, or nonlocal control transfers.

## Evidence

**Established static evidence:** The supplied body establishes two initial operation-record reference reads, the distinct uses of those reads, subsequent reference-capture order, explicit access widths and offsets, constants, dependency-call ordering, branch conditions, status writes, final handle replacement, and final comparison with 12.

**Inferred intent:** The gate, register waits, command-like write, and alternative completion branches suggest a coordinated external operation. Device identity, protocol meaning, timeout units, and resource-release semantics are not established.

**Verification limit:** No runtime tests have been performed. The acceptance examples describe conditional consequences of the supplied static behavior, not observed executions.
