<!-- Trusted provenance: F-defc2fecf605; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-defc2fecf605

## Purpose

F-defc2fecf605 passes a selected two-byte entry to a dependency, shifts subsequent entries toward the selected position, and updates metadata associated with the containing object.

The established behavior resembles removal from a packed entry sequence. That interpretation is inferred intent, not a guarantee that the routine validates an index, releases an entry’s resources, or maintains every invariant of a larger container.

The routine performs the dependency call before shifting entries or updating its own metadata. Consequently, the dependency can affect later behavior by changing memory that this routine subsequently reads.

## Interface

The routine accepts two arguments and returns no explicit value:

- **Object address:** identifies a writable object containing the fields described below.
- **Entry index:** a signed 32-bit integer used to calculate the initial entry address and control the subsequent shift.

The visible representation uses 8-bit bytes, 16-bit halfwords, and 32-bit integer/address values. Address validity, alignment requirements, and native halfword byte order are execution-environment prerequisites, not checked conditions.

### Object fields

Offsets are byte offsets from the object address.

| Offset | Width | Visible use |
|---|---:|---|
| `0x01` | 8 bits | Written with `1` at completion. |
| `0x08` | 8 bits | Unsigned displacement used to locate two metadata bytes in the backing storage. Read separately for each metadata write. |
| `0x0E` | 16 bits | Unsigned displacement used once to locate the entry sequence in the backing storage. |
| `0x12` | 16 bits | Read, increased by two, and written back as a 16-bit quantity near completion. |
| `0x14` | 16 bits | Unsigned entry count used repeatedly during shifting, then decremented and stored. |
| `0x44` | 32 bits | Backing-storage address, captured once before the dependency call. |

The entry sequence consists of two-byte slots. For the dependency argument, the first byte of a selected slot is the high byte and the second byte is the low byte. This byte significance is explicit and does not depend on native halfword byte order.

The two metadata bytes are located relative to the captured backing-storage address, at the unsigned displacement from object offset `0x08`, plus three and plus four respectively.

### Preconditions

For normal removal-like behavior, the caller must supply:

- A valid object with readable and writable fields at the listed offsets.
- A nonnegative index strictly below the entry count.
- Backing storage covering the selected slot, all slots subsequently read during compaction, and both metadata destinations.
- Address and index calculations that remain representable and address valid storage.
- Storage that remains valid after the dependency returns.

These are caller obligations rather than checks performed by this routine. A nonzero count and an in-range index are not enforced before the initial entry read or dependency call.

## Behavior

### Initial selection and dependency call

The routine first reads and captures the backing-storage address at object offset `0x44`.

It then reads the unsigned 16-bit displacement at offset `0x0E` and locates the selected slot at:

**captured backing address + entry displacement + twice the supplied index.**

Both slot bytes are read to form a 16-bit value: the byte at the lower address contributes bits 15–8, and the next byte contributes bits 7–0. The relative evaluation order of these two byte reads is not established by the supplied evidence.

The routine calls **F-fe2cdc07d7b0** with:

1. The original object address.
2. The assembled two-byte value.

This call occurs even if the supplied index is negative, equal to the count, or greater than the count, provided execution reaches the call without an invalid access or arithmetic failure. There is no preliminary count validation.

The dependency’s result is not used. Its signature, return convention, side effects, and ownership behavior are otherwise undocumented.

### Entry compaction

After the dependency returns, the routine increments its working index before comparing that index with the current unsigned 16-bit count at object offset `0x14`.

If the incremented signed index is less than that freshly read count, the routine:

- Reads the first byte of the following slot and writes it over the first byte of the current slot.
- Reads the second byte of the following slot and writes it over the second byte of the current slot.
- Advances its slot cursor by two bytes.

It then repeats the increment-and-comparison step.

The first byte assignment precedes the second byte assignment. The second source byte is therefore read after the first destination write, rather than both source bytes being buffered together. Ordinary contiguous, non-overlapping object and backing-storage layouts make that distinction immaterial; unusual aliasing can make it observable.

The count is read again at each loop condition. It is not captured before the dependency call, nor fixed for the duration of the shift.

Under ordinary conditions—an unchanged count, an in-range index, and storage that does not overlap the object fields—the routine moves every later slot one position toward the selected slot. Selecting the last slot causes no compaction writes.

There is no clearing of the old final slot. After removing a non-final slot under ordinary conditions, the former last slot’s bytes remain in their original location as well as having been copied one slot earlier.

### Count update

After compaction terminates, the routine reads the unsigned 16-bit count again, subtracts one using a 32-bit intermediate, and stores the low 16 bits back at object offset `0x14`.

For a positive count, this produces the expected decrement. A count of zero produces a stored count of `65535`; the routine does not report underflow.

The count used for this decrement is the value read after the loop, not necessarily the value used in any preceding loop comparison.

### Metadata-byte updates

The routine next writes two bytes into the captured backing storage.

For the first write:

- It reads the unsigned byte at object offset `0x08`.
- It adds that displacement and three to the captured backing address.
- It writes the byte corresponding to bits 15–8 of the previously computed decrement result.

The decrement result is converted to an unsigned 32-bit value before the right shift. In particular, decrementing zero yields a first metadata byte of `0xFF`.

For the second write:

- It reads the displacement at object offset `0x08` again.
- It reads the 16-bit field at object offset `0x14` again.
- It writes the low eight bits of that field at the captured backing address plus the newly read displacement plus four.

The first metadata write precedes these second-write reads. The two metadata bytes therefore need not form one coherent count representation if memory overlaps, the dependency established an unusual aliasing layout, or an external agent changes relevant memory.

In the ordinary non-overlapping case, these writes encode the decremented count in high-byte-first order.

### Final field updates

The routine then reads the 16-bit field at object offset `0x12`, adds two, and stores the result back into that 16-bit field. The resulting stored bit pattern is addition modulo `65536`; there is no carry field or overflow notification.

Finally, it writes byte value `1` at object offset `0x01` and returns.

No earlier value of that byte is tested. Completion does not depend on whether it was already `1`.

## Side effects

The routine’s directly established effects are:

- One call to F-fe2cdc07d7b0.
- Zero or more two-byte slot shifts, implemented as ordered byte assignments.
- One 16-bit count write.
- Two metadata-byte writes.
- One 16-bit write increasing the field at offset `0x12` by two.
- One byte write setting the field at offset `0x01` to `1`.

The captured backing-storage address and the initial slot cursor remain the basis for backing-storage accesses after the dependency returns. If the dependency changes the address field at object offset `0x44` or the displacement field at `0x0E`, this routine does not reload them to relocate its cursor.

By contrast, the count and metadata displacement are read after the dependency call. Dependency changes to those fields can influence the shift length and metadata destinations.

No allocation, deallocation, explicit error reporting, or rollback is directly visible. This does **not** establish that the dependency avoids allocation, releases nothing, or preserves other state.

The routine itself does not establish an ownership transfer for the selected entry. It passes the entry’s assembled value, not the address of that entry slot. Whether that value identifies an owned resource is unknown.

If the dependency does not return, none of this routine’s post-call writes occur. If execution fails partway through later accesses, there is no visible transactional mechanism to undo earlier writes.

## Dependencies

### F-fe2cdc07d7b0

**Established interface at this call site:** receives the original object address and a value assembled from the selected entry’s two bytes.

**Established ordering:** called after selection and before compaction or this routine’s metadata updates.

**Unknown behavior:** validation, callbacks, reentrancy, global-state changes, object mutation, allocation, resource release, storage relocation, and failure signaling are all `documented_only`.

Correct use cannot assume that this dependency preserves the object fields, backing storage, or unrelated global state. In particular, callers must account for its actual storage-lifetime behavior before relying on post-call accesses.

No other explicit function dependency is visible.

## Acceptance examples

These examples are conditional behavioral expectations derived from static evidence. They are not runtime test results.

### Removing a middle entry

Assume:

- Count is `4`.
- Index is `1`.
- Entries, in address order, are `10 11`, `20 21`, `30 31`, and `40 41`.
- The dependency returns normally without changing relevant memory.
- Object fields, entry storage, and metadata bytes do not overlap.

Expected behavior:

- F-fe2cdc07d7b0 receives the object address and value `0x2021`.
- The first entry remains `10 11`.
- The next two logical entries become `30 31` and `40 41`.
- The former final physical slot still contains `40 41`.
- Count becomes `3`.
- The metadata bytes become `00 03`.
- The 16-bit field at offset `0x12` increases by two.
- The byte at offset `0x01` becomes `1`.

### Removing the final entry

With count `3` and index `2`, under the same stability assumptions:

- The dependency receives the final slot’s value.
- No entry bytes are shifted.
- Count becomes `2`.
- The metadata bytes become `00 02`.
- The remaining final field updates still occur.

The selected slot is not erased.

### Crossing a count-byte boundary

With count `256` and index `255`, assuming stable, valid, non-overlapping storage:

- There is no compaction.
- Count becomes `255`.
- The metadata bytes become `00 FF`.

With count `257` and index `256`, those metadata bytes instead become `01 00`.

### Zero count with readable selected storage

Assume count is zero, index is zero, and the initial selected-slot read is nevertheless valid.

If the dependency leaves the count unchanged:

- The dependency is still called.
- No compaction occurs.
- Count becomes `65535`.
- The metadata bytes become `FF FF` under non-overlapping storage.
- The remaining field updates occur.

This demonstrates unchecked underflow, not an accepted empty-container removal API.

### Dependency changes the count

Assume initial count `4`, index `1`, and a dependency that changes the count to `2` while preserving valid storage.

After return, the first incremented index is `2`, which is not less than the newly read count. No compaction occurs. The subsequent decrement changes the count from `2` to `1`.

This behavior depends on live count reads rather than the pre-call count.

### Sixteen-bit increment wraparound

If the field at offset `0x12` contains bit pattern `0xFFFF`, successful completion stores `0x0001` there. Count handling and metadata writes still precede this update.

## Uncertainties

- **Semantic role — inferred intent:** removal from a packed sequence is strongly suggested, but the meanings of the entries and metadata fields are not established.
- **Dependency contract — `documented_only`:** no evidence establishes what F-fe2cdc07d7b0 owns, releases, validates, or preserves.
- **Native representation — `documented_only`:** native halfword byte order, alignment requirements, and platform-specific out-of-range signed-conversion behavior are not independently specified.
- **Extreme arithmetic — `documented_only`:** negative indices, overflow in doubling an index, address overflow, and signed overflow while incrementing an index have no portable, validated outcome established here. They must not be treated as safe wrapping behavior.
- **Aliasing — conditional:** the evidence does not prove that object fields, entries, and metadata occupy disjoint storage. Ordered writes and later rereads matter if they overlap.
- **Concurrency — `documented_only`:** no synchronization or coherent multi-field snapshot is visible. Concurrent mutation is not established as supported.
- **Failure handling — `documented_only`:** no explicit recoverable-error path is visible; platform behavior for invalid accesses is unspecified here.

## Evidence

This contract is based solely on static inspection of the supplied routine: access widths and offsets, byte assembly, dependency-call position, loop conditions, retained addresses, reread fields, and write ordering.

Established behavior is separated above from inferred container semantics and undocumented environmental or dependency guarantees. No implementation outside the supplied routine was available. No runtime tests have been performed.
