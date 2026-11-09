<!-- Trusted provenance: F-2e9fe3a9f89f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2e9fe3a9f89f

## Purpose

F-2e9fe3a9f89f conditionally updates an existing object using a caller-supplied record. It initializes two 32-bit accumulation words, incorporates dependency-produced 64-bit contributions, modifies a temporary control representation, and marks an indirectly addressed byte.

“Configuration application” is inferred intent, not an established semantic description. The established behavior is the sequence of accesses, calls, bit operations, and conditional actions described below. The meanings of the record fields, object flags, and numeric mode value 2 are unknown.

The function has no explicit return value or success indicator. A zero validation result or null record pointer skips the main update, but does not skip entry, validation, and exit dependency calls.

## Interface

### Arguments and result

- **Object address:** represented as a 32-bit integer and used both as an address and as an argument to dependencies.
- **Record address:** a pointer to byte-addressable storage.

The result is void. No dependency failure handling is established except the initial validator’s zero/nonzero branch.

All layout offsets below are byte offsets. Layout labels describe observed use, not application semantics.

### Object-associated layout

| Object offset | Access width | Established use |
|---|---:|---|
| `0x0c` | 32 bits | Indirect address used by the final byte update. |
| `0x40` | 32 bits | Indirect address used by a conditional 16-bit read. |
| `0x1ac` | 8 bits | Initially snapshotted; subsequently updated with masks `0x01` and `0x02`. |
| `0x1b0` | 8 bits when directly read | Start of an embedded region whose address is passed to dependencies; later tested as a byte. |
| `0x1b1` | 8 bits | Initially snapshotted and separately reread during final control selection. |
| `0x1b2` | 8 bits | Cleared when the initial record word at `0x08` is zero. |
| `0x1c4` | 32 bits | Initially snapshotted and separately reread during final control selection. |
| `0x1d0` | 32 bits | Low accumulation word. |
| `0x1d4` | 32 bits | High accumulation word. |

The address passed for the embedded region is the object address **plus** `0x1b0`; it is not a pointer loaded from that location.

Two indirect fields are accessed:

- A 16-bit value at offset `0x2e` from the address stored at object offset `0x40`.
- An 8-bit value at offset `0x21` from the address stored at object offset `0x0c`.

The 16-bit value is tested only for zero versus nonzero. Its signed interpretation does not change that decision.

### Record layout

| Record offset | Access width | Established use |
|---|---:|---|
| `0x00` | 8 bits | Gates contribution processing; can be reread after an accumulation. |
| `0x01` | 8 bits | Gates F-62865fe14c87. |
| `0x02` | 8 bits | Gates F-718c1b2625df on the nonzero-word branch. |
| `0x03` | 8 bits | Independently requests high-word mask `0x40`. |
| `0x08` | 32 bits | Selects the initial record-dependent branch; reread when passed to F-718c1b2625df. |
| `0x14` | 32 bits | Passed to F-62865fe14c87 and freshly read afterward for comparison with 2. |

A transfer-shaped dependency receives the record address and a length argument of `0x1c`, meaning 28 bytes. Its actual access extent and transfer semantics are not supplied.

### Preconditions and ownership

Every reached direct or indirect access requires valid storage and suitable alignment or platform support. Dependencies have additional unknown preconditions. Validation success is not established to guarantee address validity.

The validator runs before the null-record check. A null record therefore does not protect an invalid object argument from dependency effects.

The function does not explicitly allocate or free memory. Passing an address does not establish ownership transfer, retention permission, or release responsibility. In particular, dependencies receive an address into temporary storage whose lifetime does not extend beyond this invocation.

Object and record overlap is not rejected. No immutable record snapshot is created.

## Behavior

### Entry and rejection

The function first captures a 32-bit global value, then calls F-U01, the locally labeled entry dependency. It next calls F-3b037d560480 with the object address.

If validation returns zero, or the record pointer is null, the main update is skipped. F-U03, the exit dependency, is still called with the captured global value.

This describes normal returns from the dependencies. A fault or nonlocal exit can prevent later calls.

### Initial snapshots and writes

On the accepted path, the function snapshots these object fields in order:

1. The byte at `0x1ac`.
2. The 32-bit word at `0x1c4`.
3. The byte at `0x1b1`.

These saved values govern a later conditional action; subsequent storage changes do not replace them.

The function then:

1. Calls F-dcc2e15850d5 with the embedded-region address at object offset `0x1b0`.
2. Reads the current byte at `0x1ac` and writes it with bit `0x01` set.
3. Calls F-U02 with that embedded-region address, the record address, and length 28.
4. Reads the current byte at `0x1ac` and writes it with bit `0x02` set.
5. Writes zero to the 32-bit word at `0x1d0`.
6. Writes zero to the 32-bit word at `0x1d4`.

Each flags update preserves the other bits present at its own read. It does not reconstruct the byte from the initial snapshot.

If F-U02 implements an ordinary 28-byte copy, its destination would span object offsets `0x1b0` through `0x1cb`, inclusive. That interpretation would include the subsequently tested control fields, but copying is not an established dependency contract.

Overlapping record storage can already have changed before the first record-field test: dependency effects and both accumulator-clearing writes precede that test.

### Initial record-word decision

The function reads the 32-bit record word at `0x08`.

**If it is zero:**

- The object byte at `0x1b2` is cleared.
- F-0f08aa8495f3 and F-718c1b2625df are skipped.
- Record byte `0x00` is then tested.
- If that byte is nonzero, record byte `0x01` is tested and may enable F-62865fe14c87.

Thus, a zero word at `0x08` does **not** suppress F-62865fe14c87 by itself. If the record overlaps object byte `0x1b2`, the clearing write can affect the subsequent gate reads.

**If the word is nonzero:**

- F-0f08aa8495f3 is called without explicit arguments.
- Record byte `0x00` is then read.
- A zero byte skips both contribution-producing calls.
- Otherwise, record byte `0x02` is read.

When byte `0x02` is zero, processing proceeds directly to the byte-`0x01` gate without another byte-`0x00` test at that point.

When byte `0x02` is nonzero, the function freshly reads record word `0x08` and passes that value to F-718c1b2625df. The argument can differ from the earlier nonzero value; it can even be zero if intervening effects changed it.

### Contribution accumulation and repeated gating

F-718c1b2625df returns a 64-bit contribution. The function:

1. Reads the current low accumulation word and writes its bitwise OR with the returned low 32 bits.
2. Reads the current high accumulation word and writes its bitwise OR with the returned high 32 bits.
3. Rereads record byte `0x00`.
4. If that read is nonzero, reads record byte `0x01`.

The repeated byte-`0x00` test occurs **after both accumulation writes**, not immediately after the dependency returns. Its value can therefore reflect dependency mutations and overlapping accumulation writes.

Whenever the applicable byte-`0x00` gate is nonzero and byte `0x01` reads as nonzero, the function reads record word `0x14` and calls F-62865fe14c87 with the object address and that word.

Its 64-bit result is incorporated in the same order: low-word OR first, high-word OR second. Each update uses the destination word as it exists at that update. There is no arithmetic addition or carry propagation between words.

The high contribution is the returned value’s upper 32 bits. Combining the results does not depend on a memory-endianness interpretation of the two-word layout.

### Conditional first action

Only after F-62865fe14c87 has run and both of its accumulation writes have completed does the function consider F-ea35d0997acc.

The action requires:

- Bit `0x01` in the original flags snapshot is set.
- A fresh read of record word `0x14` equals 2.
- The object address is nonzero.
- The indirect 16-bit field reached through object offset `0x40`, then offset `0x2e`, is nonzero.
- Either the original object word at `0x1c4` was not 2, or the original byte at `0x1b1` was zero.

The fresh record-word read follows the contribution writes, so aliasing can affect this comparison too.

When the first two conditions hold and the object address is nonzero, the indirect 16-bit read precedes the saved-word-and-byte decision. A valid indirect address is therefore required even when the saved values ultimately suppress the action.

If all conditions hold, F-ea35d0997acc is called with the object address. On normal return, a local marker is set for a later call to F-df1f4510064d. No transactional or resource-ownership relationship between these calls is established.

### Independent high-word update

After contribution processing, record byte `0x03` is read. If nonzero, the current high accumulation word is written with bit `0x40` set.

This update does not require byte `0x00`, byte `0x01`, byte `0x02`, or word `0x08` to be nonzero.

If the accumulation words are interpreted as low and high halves of a numerical 64-bit quantity, this mask corresponds to bit 38.

### Temporary control processing

The function calls F-91f06df713c3 with the object address and an eight-byte temporary-area address.

An adjacent temporary byte is then modified. No explicit initialization of that byte appears in the supplied body. Whether F-91f06df713c3 initializes it through a larger aggregate representation remains unknown.

The selection uses current object storage after that dependency:

- Read byte `0x1b0`. If zero, select the set-bit case without reading the other control fields.
- Otherwise, read byte `0x1b1`. If zero, select the clear-bit case.
- Otherwise, read word `0x1c4`. Equality with 2 selects the set-bit case; inequality selects the clear-bit case.

For the temporary byte:

- **Set-bit case:** clear bits `0x08` and `0x10`, set bit `0x40`, and preserve all other bits.
- **Clear-bit case:** clear bits `0x08`, `0x10`, and `0x40`, preserving all other bits.

The function then calls F-d53fb32c2008 with the object address and the same temporary-area address. Whether this dependency consumes the adjacent byte is not established.

If the earlier action marker is set, F-df1f4510064d is called next.

### Completion

The final object-associated update sets bit `0x40` in the byte at indirect offset `0x21` through object offset `0x0c`, preserving its other bits.

The supplied expression references the indirect base for both reading and writing. No stronger guarantee about a single pointer load or evaluation order within that expression is established.

Finally, F-U03 receives the global value captured before F-U01. This does not establish restoration of that global or any surrounding state.

## Side effects

Established direct effects on normally completing accepted paths include:

- Two current-value flag updates at object offset `0x1ac`.
- Clearing both accumulation words before record-dependent contributions.
- Conditional clearing of byte `0x1b2`.
- Conditional OR accumulation of dependency results.
- Conditional setting of high-word bit `0x40`.
- Modification of the adjacent temporary byte.
- Setting bit `0x40` in the indirectly addressed completion byte.

Dependencies may additionally mutate object storage, record storage, globals, or indirect structures. Their preservation, allocation, retention, and release behavior is unknown.

No monotonic final-state guarantee follows from the OR operations alone: later dependencies or overlapping accesses may change previously written storage.

There is no visible atomicity, synchronization, rollback, or recovery protocol. Interrupted execution can leave a partial update.

## Dependencies

F-U01, F-U02, and F-U03 are local opaque labels for dependencies whose mapped function IDs were not supplied.

| Opaque ID | Established interaction |
|---|---|
| F-U01 | Entry call after capturing the global value; no explicit arguments shown. |
| F-3b037d560480 | Receives the object address; zero return skips the main update. |
| F-dcc2e15850d5 | Receives the embedded-region address at object offset `0x1b0`. |
| F-U02 | Receives embedded-region address, record address, and length 28. |
| F-0f08aa8495f3 | No explicit arguments; called on the nonzero initial record-word branch. |
| F-718c1b2625df | Receives a fresh record word from `0x08`; returns a 64-bit contribution. |
| F-62865fe14c87 | Receives object address and record word from `0x14`; returns a 64-bit contribution. |
| F-ea35d0997acc | Conditional first action with the object address. |
| F-91f06df713c3 | Receives object address and temporary-area address before byte modification. |
| F-d53fb32c2008 | Receives the same addresses after byte modification. |
| F-df1f4510064d | Conditional later action following temporary control processing. |
| F-U03 | Exit call receiving the captured 32-bit global value. |

Ignored return values do not establish successful completion of any dependency-defined operation.

## Acceptance examples

These are conditional static acceptance cases, not runtime test results. Calls are assumed to return normally.

1. **Rejected validation.** If F-3b037d560480 returns zero, the function performs no main-body field writes. F-U01, validation, and F-U03 still execute. Their own effects remain unconstrained.

2. **Null record.** Validation still executes. After its normal return, the null record causes the main update to be skipped and F-U03 to run.

3. **Zero initial record word permits the second contribution.** Suppose word `0x08` reads as zero and, after clearing object byte `0x1b2`, record bytes `0x00` and `0x01` both read as nonzero. F-0f08aa8495f3 and F-718c1b2625df are skipped, but F-62865fe14c87 is called.

4. **Inactive primary gate with independent mask.** If the applicable byte-`0x00` test reads zero, neither contribution occurs through that gate. If byte `0x03` subsequently reads nonzero, the high-word `0x40` update still occurs. F-0f08aa8495f3 has already run if the initial word at `0x08` was nonzero.

5. **Two stable contributions.** Assume both contribution paths are enabled, there is no interfering aliasing, and dependencies do not otherwise modify the accumulation words. Results `0x0000000100000002` and `0x0000000400000008` produce low word `0x0000000a` and high word `0x00000005` immediately after accumulation. A subsequently nonzero byte `0x03` changes the high word to `0x00000045`.

6. **Mutation suppresses continuation only at the later read.** If F-718c1b2625df clears record byte `0x00`, F-62865fe14c87 is suppressed only if that byte still reads as zero after both intervening accumulation writes. Suppression follows, for example, when those writes do not overlap or otherwise affect the cleared byte.

7. **Overlapping accumulation can reverse that mutation.** Conditional on reaching F-718c1b2625df with the record starting at object offset `0x1d0`, suppose the dependency clears record byte `0x00` and returns low word `0xffffffff`. The subsequent low-word OR writes all one bits across record bytes `0x00` through `0x03`. The repeated primary-byte test therefore reads nonzero. If byte `0x01` is then read without further interference, it is also nonzero and F-62865fe14c87 is reached. Earlier accumulator initialization means reaching this overlapping case requires the gates to have been made nonzero afterward, for example by F-0f08aa8495f3.

8. **Saved-state suppression.** If F-62865fe14c87 occurs, the fresh mode reads as 2, and the required indirect read is nonzero, an original word of 2 together with an original nonzero byte at `0x1b1` still suppresses both conditional actions, regardless of later changes to those two object fields.

9. **Temporary-byte masks.** If the temporary byte is validly initialized to `0xff` at modification time, the set-bit case produces `0xe7`; the clear-bit case produces `0xa7`. This does not establish that the function initializes the byte or that F-d53fb32c2008 consumes it.

## Uncertainties

The following remain **documented_only**:

- Application meanings of fields, masks, and mode value 2.
- F-U02’s transfer semantics, access extent, overlap rules, and failure behavior.
- Validation guarantees beyond its zero/nonzero return.
- Dependency effects on memory, globals, ownership, retention, and release.
- Initialization and full extent of the temporary representation, including whether the adjacent byte is communicated to either dependency.
- Entry/exit semantics and the purpose of the captured global value.
- Platform endianness, alignment requirements, address-overflow behavior, and invalid-pointer behavior.
- Dependency failures, faults, and nonlocal exits.
- Concurrent access behavior; no synchronization contract is supplied.

## Evidence

This contract is based on the supplied static function representation and opaque dependency mapping. Dependency implementations were not supplied. No runtime tests have been performed.

Established evidence covers explicit widths, offsets, call sequencing, branch structure, masks, saved versus fresh reads, and direct writes. In particular, both accumulation writes precede the repeated primary-byte test after F-718c1b2625df; a dependency’s earlier byte mutation alone cannot determine that test when record/object overlap is allowed.

Configuration semantics, ordinary copying, aggregate temporary initialization, and resource-pair interpretations remain inferred or unknown rather than established behavior.
