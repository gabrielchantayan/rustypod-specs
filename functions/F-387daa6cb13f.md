<!-- Trusted provenance: F-387daa6cb13f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-387daa6cb13f

## Purpose

F-387daa6cb13f selects an entry in a fixed-stride table, reads an address from the start of that entry, and stores the four-byte integer value 1 at a fixed offset from that address.

The established behavior is a memory update, not a query or allocation operation. The supplied representation contains no condition, loop, validation, or call to another function.

The destination may be a flag or state field, but that interpretation is inferred intent only. Its name, meaning, and relationship to any larger state machine are not established.

## Interface

**Input:** one signed short integer, used as a table index.

For this contract, the represented input width is 16 bits, with a range of −32,768 through 32,767. The supplied representation does not independently document the calling convention or how a caller’s wider argument becomes this short value. Any narrowing performed before entry is outside this function’s established behavior.

**Output:** no return value. Successful completion is observable through the destination memory store.

**Ambient input:** a global value supplying the table base address. This is separate from the explicit index argument. Its initialization, ownership, and stability are not shown.

**Relevant layout:**

| Item | Width or displacement |
|---|---|
| Table-entry stride | 396 bytes, hexadecimal 0x18C |
| Address field within a selected entry | At entry offset 0 |
| Selected address read | Four bytes, represented as a 32-bit integer used as an address |
| Destination displacement from that address | 2,444 bytes, hexadecimal 0x98C |
| Destination store | Four-byte integer value 1 |

The 396-byte stride does not establish a complete entry layout. Only the address field at the beginning of the selected entry is accessed.

The four-byte address representation is evidence of a 32-bit address model for this operation. It is not evidence that the same representation is valid on a different-width platform.

## Behavior

### Address selection

The signed input index selects a location relative to the ambient table base. Its byte displacement is the index multiplied by 396.

The index is not explicitly converted to an unsigned value, clamped, or checked against a table size. Negative inputs therefore produce negative displacements under the represented signed interpretation.

For a 16-bit signed input, the multiplication has these bounds:

- Minimum displacement: −12,976,128 bytes.
- Maximum displacement: 12,975,732 bytes.

Those displacements fit within a signed 32-bit integer. This does not establish that adding a displacement to the table base is safe: that depends on the base value, address arithmetic, and mapped storage.

The function reads the selected address from the first four bytes at the resulting table location. It then forms the destination by adding 2,444 bytes to that address.

### Memory update

The destination receives the four-byte integer value 1.

This is a full-width replacement, not a bitwise operation. The function does not preserve other bits within that four-byte destination. If the previous value is zero, a different integer, or already 1, the specified stored value is still 1.

There is no explicit read of the destination’s previous value and no read-modify-write operation. The supplied representation does not request writes to neighboring bytes.

### Order of accesses

The established logical access order is:

1. Obtain the ambient table base.
2. Read the four-byte address at the selected table location.
3. Write the four-byte value 1 at 2,444 bytes beyond that address.
4. Return without a value.

The destination depends on the selected address read. The function does not first test whether that address is null or whether the destination is writable.

This order describes the source-level dependencies. It does not establish volatile access semantics, synchronization barriers, hardware transaction boundaries, or externally visible timing.

### Preconditions for ordinary completion

Ordinary completion requires that:

- The ambient table base can be obtained.
- The selected table location supports the four-byte address read.
- The loaded address and fixed displacement identify a writable four-byte destination.
- Address calculations are valid under the execution environment’s arithmetic and pointer rules.
- Any applicable alignment requirements are met.

No table length or allocation boundary is supplied. Consequently, an index cannot be classified as valid solely from its numeric value. Index zero can fail with an invalid base, while a negative index could succeed if the base and surrounding storage intentionally permit it.

## Side effects

The explicit memory mutation is one four-byte store of the integer value 1.

The function contains no explicit allocation, release, ownership transfer, reference-count change, callback, logging, or error reporting. It neither receives nor returns an ownership token. Accessing the selected object does not establish that the function owns it.

No separate update to the table entry or ambient base is requested. However, the supplied representation establishes no non-aliasing guarantees. If the computed destination overlaps the selected entry, the storage holding the ambient base, or some other meaningful state, that storage is affected by the same four-byte store. The logical role “destination field” does not prevent such overlap.

Successful repeated calls write the same value when they resolve to the same destination. They are not generally guaranteed to remain directed at that destination: intervening changes to the ambient base or selected address can redirect a later call. Aliasing from an earlier store can also matter.

The operation is not established as atomic or thread-safe. Concurrent modifications of the table base, selected address, or destination have no documented coordination here.

The integer value is specified independently of byte order. On a little-endian target, its bytes would be 01 00 00 00; on a big-endian target, they would be 00 00 00 01. The supplied representation alone does not establish the target byte order.

## Dependencies

No callable dependencies are shown. There are therefore no additional opaque function IDs to list.

The function depends on three externally supplied storage properties:

- A usable ambient table base.
- A readable four-byte address at the selected entry.
- A writable four-byte region at the fixed displacement from that address.

The mechanisms that establish these properties are outside the available evidence. No initialization routine, allocator, cleanup routine, bounds provider, or synchronization mechanism can be attributed to this function.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not reported runtime results.

1. **Zero index.** Given a valid table base and a readable address at its first four bytes, index 0 selects that address. If the region 2,444 bytes beyond it is writable, that region receives the four-byte integer value 1.

2. **Positive index.** Given a valid entry beginning 396 bytes after the table base, index 1 reads the address at that entry’s beginning. The store is relative to the loaded address, not directly relative to the table base.

3. **Negative index.** Given readable entry storage beginning 396 bytes before the table base, index −1 selects that storage. There is no explicit negative-index rejection. Successful completion still requires a usable loaded address and writable destination.

4. **Existing destination value.** Given a valid destination whose four-byte value is 0, 7, or all one bits, the stored result is the integer 1. The operation does not merely set the lowest bit while preserving the rest.

5. **Destination already equal to 1.** Given valid storage already containing 1, the function still specifies a store of 1. It does not explicitly inspect the old value to avoid the write.

6. **Entry padding.** Given an otherwise valid layout, bytes elsewhere in the 396-byte entry do not participate in address selection. They are not explicitly read or modified, except if the computed destination aliases them.

7. **Unusable selected address.** Given an unreadable selected table location or an unwritable destination, no graceful rejection is established. An acceptance criterion requiring a status return, ignored request, or recovery action would exceed the available evidence.

8. **Extreme inputs.** The minimum and maximum signed 16-bit inputs produce the displacement bounds stated above. Their successful execution remains conditional on valid address arithmetic and sufficient accessible storage; arithmetic fit of the multiplication alone is insufficient.

## Uncertainties

The following are **documented_only**:

- The table’s entry count and intended legal index range.
- Whether callers guarantee nonnegative inputs.
- The meaning of the destination value 1.
- The size, type, lifetime, and ownership of the selected object.
- Initialization and replacement rules for the ambient base and entry addresses.
- Target byte order and exact alignment requirements.
- Address-addition overflow and wraparound behavior.
- Concurrency guarantees, memory ordering, and atomicity.
- The execution environment’s response to inaccessible or malformed addresses.
- The exact calling convention and argument-narrowing rules.

There is no explicit null check. This does not imply a particular null-address failure: the loaded value is first offset by 2,444 bytes, and access behavior depends on the environment.

There is no evidence of a special sentinel index, an error return, or a recoverable failure path. Those behaviors must not be invented to make invalid inputs appear safe.

## Evidence

**Established static evidence:** one signed-short input; a 396-byte index scale; a four-byte selected-address read; a 2,444-byte destination displacement; a four-byte store of integer 1; and return without a value. No branch, loop, or function call is shown.

**Inferred intent:** the selected address likely identifies an object, and the destination likely represents a flag or state field. Neither interpretation is required for the memory behavior described above.

**Verification boundary:** this contract is based only on the supplied static representation. No implementation access, runtime tests, memory observations, or fault experiments have been performed. The acceptance examples are conditions for evaluation, not claims of observed execution.
