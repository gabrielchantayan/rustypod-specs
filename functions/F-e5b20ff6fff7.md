<!-- Trusted provenance: F-e5b20ff6fff7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e5b20ff6fff7

## Purpose

F-e5b20ff6fff7 writes selected fields in caller-supplied storage. It copies a value from a shared 32-bit source word into the destination, writes one byte with the value one, and clears five other fields. It does not clear the entire destination region.

The established behavior consists of one source-word read followed by seven destination stores. Treating this as object initialization or reset is **inferred intent**. The supplied static evidence does not establish an object type, field meanings, or lifecycle requirements.

The source word’s relationship to the destination is unknown. If they overlap, the copied value can depend on the destination’s previous contents. The contract therefore distinguishes the absence of explicit reads through the destination pointer from independence of the result from destination memory.

## Interface

### Input and storage extent

The function accepts one pointer to writable storage. All destination offsets in this contract are byte offsets from that pointer.

The directly written destination bytes extend from offset 0 through offset 75, with gaps. A contiguous writable allocation of at least 76 bytes is sufficient for the destination extent, provided the address calculations and access alignment are also valid. This does not establish the complete size of any surrounding object.

The destination must support:

| Starting byte offset | Access width | Covered byte offsets |
|---|---:|---|
| 0 | 32 bits | 0–3 |
| 4 | 8 bits | 4 |
| 5 | 8 bits | 5 |
| 68 | 8 bits | 68 |
| 69 | 8 bits | 69 |
| 70 | 8 bits | 70 |
| 72 | 32 bits | 72–75 |

A separate, shared 32-bit source word must be readable when the source read occurs. Its address is fixed by the supplied body rather than passed as another argument. The evidence does not establish whether that address is disjoint from the destination.

There are no explicit checks for a null pointer, available extent, alignment, permissions, or overlap. A null destination is not an established no-op case.

### Output

The function has no explicit return value and provides no success or failure status. Its ordinary result is the modification of memory.

Let **S** be the 32-bit value obtained by the source read during this invocation. After normal completion, under ordinary memory semantics and without external modification, the written destination fields have these values:

| Byte offset | Width | Result |
|---|---:|---|
| 0–3 | 32 bits | S |
| 4 | 8 bits | 0 |
| 5 | 8 bits | 1 |
| 68 | 8 bits | 0 |
| 69 | 8 bits | 0 |
| 70 | 8 bits | 0 |
| 72–75 | 32 bits | 0 |

The assignment at offset 0 preserves the source’s 32-bit value without visible arithmetic or conversion to a narrower field. Its byte representation follows the target’s word representation; byte order is not independently established by the supplied evidence.

### Ownership and validity

The function neither allocates nor releases storage. It makes no visible transfer of the destination pointer to another function.

The caller must provide storage that remains valid for the required accesses. Validity includes the source read as well as the destination writes. If source and destination overlap, the overlapping bytes must support both relevant access types.

Synchronization with other participants is external to this function. No visible locking, retry mechanism, or transactional boundary is present.

## Behavior

### Explicit access order

The supplied static body establishes the following sequence:

1. Read the shared 32-bit source word.
2. Write the value read to the 32-bit destination field beginning at offset 0.
3. Write zero to destination byte 4.
4. Write one to destination byte 5.
5. Write a 32-bit zero beginning at destination offset 72.
6. Write zero to destination byte 68.
7. Write zero to destination byte 70.
8. Write zero to destination byte 69.
9. Return without an explicit result.

The source is read before any destination store. There is no later explicit source read.

The final three stores are ordered by offsets 68, 70, and 69, not by increasing address. The 32-bit store at offset 72 precedes all three.

This is the explicit access sequence represented by the supplied body. It does not establish processor-level visibility order, access atomicity, memory barriers, or the physical shape of memory transactions.

### Partial initialization and previous contents

There are no explicit reads through the destination pointer. Each destination operation is a store.

That fact does **not** establish that previous destination contents cannot influence the assigned results. The shared source word may overlap destination storage. In that case, the source read obtains some or all of its value from bytes that also belong to the destination.

For example, if the shared source word occupies destination bytes 72–75, their initial 32-bit value becomes S. The function copies that value into destination bytes 0–3 before clearing bytes 72–75. Thus, an initial value in a field that is later cleared can determine the final value at offset 0.

With a non-aliasing source, and absent external changes or unusual memory semantics, the assigned results do not depend on the destination’s previous contents. The source read determines the first assigned word, and constants determine the other six stores.

No explicit destination store covers:

- Bytes 6–67.
- Byte 71.
- Any byte at or beyond offset 76.

In ordinary memory, without external modification or alternative mappings that make different addresses affect the same storage, those bytes retain their previous contents. They may still have been read as part of the shared source word if that word overlaps them.

The function is not equivalent to clearing the first 76 bytes and then restoring selected fields. Such a replacement would overwrite bytes the supplied body does not write. It could also change the copied value if it cleared an overlapping source before reading it.

### Source overlap

The source read captures S before the first store. Subsequent destination stores do not cause the function to reread or revise S.

Consequently:

- If the source is exactly the destination word at offset 0, the first store writes back the value just read.
- If the source is exactly the destination word at offset 72, its original value is copied to offset 0 before the source location is cleared.
- If the source overlaps another written destination region, those later stores may change the source word’s stored value after S has been read.
- If the source only overlaps destination bytes that are not written, that overlap can influence S without itself causing those bytes to change.

These cases are conditional consequences of overlap, not evidence that overlap occurs in actual callers. No separate overlap-handling branch is present.

### Widths and address arithmetic

Each word access is 32 bits, covering four bytes. Each byte store is eight bits.

The second destination word starts at byte offset 72 and ends at byte offset 75. It is not a word at byte offset 18: 18 is the corresponding count of four-byte units. Likewise, the operation at offset 68 writes only one byte, despite its address being expressed relative to a four-byte-oriented pointer.

All offsets are fixed. There is no input-dependent loop, length calculation, signed comparison, or arithmetic transformation of S. Zero and one are exactly representable in their destination widths.

Some address expressions use a 32-bit integer representation of the destination address. The supplied evidence therefore does not support a portable guarantee for arbitrary wider addresses. The precondition is that every address expression denotes the intended location without harmful truncation, overflow, or wraparound. Exact language-level and machine-level consequences outside that precondition remain unresolved.

### Repeated invocation

Every ordinary invocation performs the same explicit access sequence; there is no check-before-write path.

If the value read as S remains unchanged between invocations, the written fields receive the same values each time.

Source stability must not be assumed merely because no external actor changes memory. If the source overlaps a destination field, one invocation may alter the source value that a later invocation reads. For example, a source at destination offset 72 is cleared during the first invocation, so a second invocation reads zero under otherwise unchanged ordinary-memory conditions.

## Side effects

The explicit memory effects are:

- One shared 32-bit source read.
- Two 32-bit destination stores.
- Five eight-bit destination stores.

There are no visible function calls, allocations, releases, callbacks, or input/output operations.

There is no explicit store naming the source word as a separate destination. Nevertheless, destination writes can modify that word if it overlaps the destination. More generally, writing through the destination pointer can update shared or global storage if the caller supplies such an address.

The operation provides no explicit rollback. If an access fails after earlier accesses have taken effect, the function contains no recovery path that restores the original contents. The evidence does not establish the fault mechanism, whether execution continues, or which partial effects remain observable.

## Dependencies

No function dependencies are visible in F-e5b20ff6fff7.

Its sole explicit external data dependency is the shared 32-bit source word. The established behavior depends on the value obtained by its read, not on a presumed semantic role.

The source word is not established to be a type identifier, dispatch reference, constant marker, or any other particular kind of value. Those interpretations remain **documented_only**.

Execution also depends on the environment’s address representation, access permissions, alignment requirements, word representation, and memory-fault behavior. No assumptions about an unknown callee’s preservation of globals or ownership behavior are needed because no calls are present.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

### Disjoint source and patterned destination

Given valid ordinary writable destination storage, a disjoint readable source containing S, and a recognizable initial destination pattern:

- The destination word at offset 0 becomes S.
- Byte 4 becomes zero; byte 5 becomes one.
- Bytes 68, 69, and 70 become zero.
- The destination word at offset 72 becomes zero.
- Bytes 6–67 and byte 71 retain their initial pattern.
- No explicit store targets bytes beyond offset 75.

Under these conditions, changing the destination’s initial contents does not change the assigned values. Clearing the untouched bytes would violate the specified partial-write behavior.

### Source aliases the word at offset 72

Given that the readable source word is exactly destination bytes 72–75 and initially contains a nonzero 32-bit value A:

- The source read obtains A.
- Destination bytes 0–3 receive A.
- Destination bytes 72–75 subsequently become zero.
- The remaining stores occur with their specified widths, values, and order.

With no intervening external changes, a second invocation reads zero from offset 72 and therefore writes zero at offset 0. Expecting both invocations to copy A would be incorrect.

### Source aliases the word at offset 0

Given that the source is exactly destination bytes 0–3, initially containing A:

- The source read obtains A.
- The first store writes A back to those bytes.
- The other six stores still occur.

The absence of a final change in the first word does not mean its read or store was skipped.

### Distinct source values

Given two otherwise valid invocations whose source reads yield distinct values:

- Each invocation writes its respective read value at offset 0.
- All remaining assigned fields receive the specified constants.
- A source value of zero introduces no explicit alternative path.

### Access-order observation

Given an observation mechanism that faithfully exposes the represented accesses without changing their semantics, the source read precedes these destination stores:

32 bits at offset 0; eight bits at 4; eight bits at 5; 32 bits at 72; eight bits at 68; eight bits at 70; eight bits at 69.

Final memory contents alone do not demonstrate this ordering.

### Invalid storage

Given an inaccessible source or destination address, successful completion and a defined error return are not promised. Silent acceptance, rollback, and a particular fault outcome are not established.

## Uncertainties

The following remain **documented_only**:

- The destination’s type, complete size, field meanings, and lifecycle.
- The source word’s semantic role, initialization, and update policy.
- Whether actual callers supply storage overlapping the source.
- Target byte order and exact alignment requirements.
- Support for unaligned word accesses.
- Exact address-conversion behavior outside the valid-address precondition.
- Whether memory is ordinary, shared, device-backed, or alternatively mapped.
- Atomicity, cross-thread visibility, and hardware transaction details.
- Fault behavior and the visibility of partial effects.
- Any initialization required beyond the fields written here.

An initialization purpose is plausible. It does not establish that untouched bytes are padding, that the resulting storage is a complete valid object, or that repeated invocations are idempotent when the source aliases a field that is modified.

## Evidence

This contract is based solely on the supplied static body: the source-word read, destination access widths, fixed offsets, assigned constants, represented order, and absence of explicit branches, calls, or a return value.

The overlap analysis follows from reading the source before performing any destination store. It does not depend on an assumed non-aliasing guarantee.

No independent implementation access, runtime execution, or runtime tests have been performed. The acceptance examples express conditional consequences of the static evidence, not observed execution results.
