<!-- Trusted provenance: F-879bffe6f42b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-879bffe6f42b

## Purpose

F-879bffe6f42b conditionally writes a supplied byte to an address calculated from a base input and an unsigned index.

Indices from 0 through 127 are accepted. For an accepted index, the function calculates a 32-bit integer address using a stride of 20 bytes and a fixed offset of 8 bytes, converts that integer to a pointer, and writes one byte. It then returns the 32-bit value 0.

Indices from 128 through 4,294,967,295 are rejected. For a rejected index, the function returns the 32-bit value 2 without calculating or accessing the destination.

The integer address calculation wraps modulo 2³². This arithmetic behavior is established independently of the execution environment’s integer-to-pointer mapping. A wrapped result is neither necessarily valid nor necessarily invalid as a destination.

The stride and offset describe the observed addressing pattern. They do not establish an array type, record schema, or application-level meaning for the byte.

## Interface

| Item | Established representation | Role |
|---|---|---|
| Base | Signed 32-bit integer | Converted to unsigned 32-bit form during address calculation |
| Index | Unsigned 32-bit integer | Selects the accepted or rejected path and, if accepted, the destination displacement |
| Value | One byte, eight bits | Bit pattern assigned to the selected destination |
| Result | Four-byte value | Exactly 0 following the store, or 2 for a rejected index |

The base is an integer used to construct an address. It is not first dereferenced to obtain another pointer.

The value is an uninterpreted eight-bit quantity. The function does not validate its numeric meaning, character encoding, or use as a flag. Every eight-bit pattern follows the same control flow for a given index.

For an accepted index, defined normal completion requires that conversion of the calculated integer address produce a pointer through which a one-byte write is valid. The function does not receive an allocation size or check mapping, protection, ownership, or lifetime.

That precondition applies to the **calculated, wrapped address**, not necessarily to a conventional unwrapped address obtained by extending the base into a larger integer type. Overflow of the 32-bit address calculation alone does not establish whether the precondition holds.

For a rejected index, destination-memory validity is irrelevant to the shown branch: the function does not dereference the base or any address derived from it.

The calling convention, argument locations, treatment of wider caller-side arguments, and register state outside the four-byte result are not established.

## Behavior

### Index classification

The function compares the index with 128 as an unsigned 32-bit value.

- Indices 0 through 127 take the store path.
- Indices 128 through 4,294,967,295 take the rejection path.

This is not a signed bounds check. For example, the index bit pattern FFFFFFFF represents 4,294,967,295 at this interface and is rejected. Interpreting those bits elsewhere as signed negative one does not change this behavior.

### Accepted-index address calculation

For an accepted index, the address calculation has three arithmetic stages:

1. Multiply the index by 20.
2. Add the base to that product.
3. Add 8 to the result.

All three operations use unsigned 32-bit arithmetic. The unsigned index causes the base to be converted to unsigned 32-bit form before the addition involving the base. The constants 20 and 8 participate in the same unsigned arithmetic.

For a negative base, this conversion produces its value modulo 2³². For example, a base value of −16 becomes the unsigned value 4,294,967,280, represented in hexadecimal as FFFFFFF0.

Let **B** denote the base after conversion to unsigned 32-bit form and **I** the unsigned index. The final integer address is:

**A = (B + 20 × I + 8) modulo 4,294,967,296**

Each arithmetic stage has 32-bit unsigned semantics. The combined formula is an equivalent description of the resulting integer value; it does not imply a wider intermediate calculation.

Because the range check precedes this calculation, the accepted-index product ranges only from 0 through 2,540. The combined index-dependent displacement ranges from 8 through 2,548. Neither quantity overflows 32 bits on the accepted path. Nevertheless, adding either quantity to the converted base can wrap.

Relative to an ordinary, nonwrapping interpretation of the base:

- Index 0 selects the byte at offset 8.
- Index 1 selects the byte at offset 28.
- Consecutive selected bytes are 20 bytes apart.
- Index 127 selects the byte at offset 2,548.

These offsets describe the integer calculation. Pointer mapping and object validity remain separate requirements.

### Store and result

After calculating the integer address, the function converts it to a pointer suitable for the byte assignment and stores the supplied eight-bit value through that pointer.

No previous destination value is explicitly read. The function does not compare old and new values or skip the assignment when they are equal.

Only one destination byte is explicitly assigned. The function does not initialize, clear, copy, or otherwise update the remaining bytes of any implied 20-byte slot.

After the assignment completes, the function selects and returns the four-byte value 0. Normal completion with that result therefore follows the store; it is not an advance indication that a later store will occur.

### Rejected-index behavior

For an index of 128 or greater, the function selects and returns the four-byte value 2.

The rejected path does not evaluate the indexed destination expression, convert a calculated address to a pointer, or access destination memory. The base and supplied byte do not affect this result.

### Storage extent

The function checks an index bound, not a buffer bound. An accepted index does not prove that the calculated destination is writable.

Under an ordinary contiguous, nonwrapping address mapping, writable storage covering offsets 0 through 2,548 is sufficient to contain every selected byte. A full allocation of 128 nominal 20-byte slots would also suffice under those assumptions, but that allocation size is not required or verified by the function.

For a particular invocation, only the selected byte must support the shown write. The function does not inspect intervening bytes.

## Side effects

The only explicit destination-memory operation is the accepted path’s one-byte store. There is no explicit read of caller-provided memory.

The relevant observable sequence is:

- Classify the unsigned index.
- On acceptance, calculate and convert the destination address, then perform the byte store.
- Return 0 after the store completes.
- On rejection, return 2 without a destination access.

This sequence does not establish an order for independently evaluating arithmetic operands, nor does it establish hardware-level visibility or synchronization guarantees.

There is no explicit allocation, deallocation, ownership transfer, logging, resource acquisition, or update to a separate status object. The function does not return a pointer or change the caller’s base input.

The function establishes no locking, cross-thread synchronization, or memory-ordering contract. The one-byte assignment alone does not establish a concurrency-safe API or a guarantee of atomicity for every possible destination.

If the calculated integer maps to an invalid or nonwritable destination, normal completion is not established. Faults, device-memory effects, and other consequences depend on the execution environment. The result 2 is specifically the index-rejection result, not a general memory-error result. No recovery path for an unsuccessful store is shown.

## Dependencies

No dependency calls are present in the supplied body. Consequently, no additional opaque function IDs are needed.

The function depends on the execution environment for:

- Interpretation of the calculated 32-bit integer as a pointer.
- Availability and writability of the resulting destination.
- Effects of writes to ordinary, shared, protected, or device-backed memory.
- Fault handling and externally observable memory ordering.

These are environmental dependencies, not established helper behavior. No unseen callee is assumed to validate memory, preserve globals, allocate storage, or release ownership.

## Acceptance examples

These are conditional criteria derived from static evidence, not reports of executed tests.

### First selectable byte

Given index 0, value A5 in hexadecimal, and a base whose calculated address maps to writable memory:

- The unsigned integer address is the converted base plus 8, modulo 2³².
- The selected byte is assigned the A5 bit pattern.
- The function returns the four-byte value 0 after the store.
- No neighboring byte is explicitly assigned.

### Adjacent selection

Given index 1 and a valid calculated destination:

- The index-dependent displacement is 28.
- The selected address is the converted base plus 28, modulo 2³².
- Only that selected byte is explicitly written.
- The function returns 0.

Under an ordinary nonwrapping mapping, this selects a byte 20 bytes after the destination for index 0, rather than the next contiguous byte.

### Last accepted index

Given index 127:

- The product of the index and 20 is 2,540.
- The total displacement is 2,548.
- The destination integer is the converted base plus 2,548, modulo 2³².
- If that integer converts to a valid writable byte destination, the supplied byte is stored and the function returns 0.

### Wrap during the base addition

Given a signed base of −16 and index 1:

- Base conversion produces hexadecimal FFFFFFF0.
- The product is hexadecimal 14.
- Adding the product to the converted base wraps to hexadecimal 00000004.
- Adding 8 produces hexadecimal 0000000C.

The established integer address is therefore 12. If its pointer conversion yields a writable destination, the function writes there and returns 0. Wrapping does not itself cause index rejection or establish a memory fault.

### Wrap during the final addition

Given a signed base of −8 and index 0:

- Base conversion produces hexadecimal FFFFFFF8.
- Adding the zero product leaves that value unchanged.
- Adding 8 wraps to integer address zero.

The integer result is established. Whether its pointer conversion permits a write is not established. This example does not assert that address zero is writable or that the store completes.

### First rejected index

Given index 128:

- The function returns 2.
- No destination address is calculated or accessed.
- No writable-memory assumption is needed for the base.

### Maximum unsigned index

Given index 4,294,967,295:

- The unsigned comparison rejects the index.
- The function returns 2 without an indexed store.

No accepted-path multiplication or address wrapping is needed to determine this outcome.

### Rewriting an existing value

Given an accepted index and a valid destination already containing the supplied byte:

- The assignment is still performed.
- The function returns 0 after the store.
- There is no distinct “unchanged” result.

### Invalid accepted destination

Given an accepted index whose calculated address does not support the required byte write, normal completion is not established. The function does not turn this condition into result 2.

## Uncertainties

**Inferred intent:** The limit, stride, and fixed offset suggest 128 slots with a byte-valued field at offset 8. No actual record type or meaning for that field is established.

**documented_only — address interpretation:** The unsigned 32-bit arithmetic and its modulo wraparound are established. The integer-to-pointer mapping, including the interpretation of zero or small integer addresses, is not established.

**documented_only — storage:** Allocation extent, object boundaries, lifetime, and ownership outside this function are unknown. A negative signed base or wrapped calculation alone does not establish destination validity or invalidity.

**documented_only — result semantics:** The returned bit patterns are exactly 0 and 2 on the described paths. Broader application meanings for those values are unknown.

**documented_only — environment:** Fault handling, memory-mapped device effects, concurrent-access behavior, and external visibility guarantees are unresolved.

## Evidence

The supplied static body establishes:

- A signed 32-bit base input, unsigned 32-bit index, one-byte value, and four-byte result.
- An unsigned comparison against 128.
- An accepted-path address expression containing multiplication by 20 and additions of the base and 8.
- Unsigned 32-bit evaluation of that multiplication and both additions, including unsigned conversion of the base.
- Conversion of the resulting integer to a pointer followed by one byte assignment.
- Selection of result 0 after the assignment.
- Selection of result 2 on rejection without destination access.
- No dependency calls, destination-content checks, or additional explicit data-memory writes.

No runtime tests have been performed. The acceptance examples express static behavioral criteria and conditional environmental assumptions, not observed execution results.
