<!-- Trusted provenance: F-f8aad93b23bd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f8aad93b23bd

## Purpose

F-f8aad93b23bd represents a sequence of indirect calls, byte-selected writes, and arithmetic-addressed 32-bit stores. Incoming values, memory contents, and dependency results supply subsequent call arguments, addresses, and stored values.

No application-level purpose is established. The evidence does not identify serialization, allocation, device control, or a particular data structure.

The supplied static representation reports invalid instruction data and truncated control flow. This contract describes its represented path, not a proven executable path. It distinguishes:

- **Established static evidence:** represented accesses, widths, data dependencies, and ordering.
- **Conditional arithmetic consequences:** results assuming wrapping 32-bit arithmetic.
- **documented_only unknowns:** matters requiring instruction validation, dependency implementations, calling-convention information, or runtime observation.

No runtime tests have been performed.

## Interface

### Explicit inputs

Four 32-bit input positions are displayed:

| Position | Interpretation | Represented use |
|---|---|---|
| First | Uninterpreted 32-bit value | Passed to the first indirect call. |
| Second | Uninterpreted 32-bit value | No explicit use. |
| Third | Signed 32-bit quantity | Passed after a left shift of 26 positions. |
| Fourth | Signed 32-bit quantity | Passed after a left shift of 10 positions. |

The second position’s lack of explicit use does not prove that dependencies cannot observe corresponding machine state.

### Additional effective inputs

The path also depends on:

- An incoming address for the first byte read.
- An incoming byte for the first write.
- Incoming control state used to form the first indirect-call destination.
- Two separately represented reads of one global 32-bit storage location.
- Secondary register values exposed after call boundaries.
- Memory reached through arithmetic-derived addresses.
- Results and effects of unresolved dependencies.

Their provenance and validity are not established.

### Outputs and termination

The displayed interface has no ordinary return value. Its represented path ends at an invalid-instruction termination marker, not a normal return.

Actual terminal behavior is **documented_only**. The evidence does not establish a deliberate trap, process exit, particular exception, or valid continuation.

### Widths, layouts, and preconditions

A byte read yields an unsigned value from 0 through 255. A byte write retains only the low eight bits. Word accesses transfer 32 bits; byte order is unknown.

A **word slot indexed by a value** means the address formed by multiplying that value by four and adding 20. This describes address arithmetic, not an established array layout or object boundary.

A **selector-directed write** means reading an unsigned byte at a specified selector address, adding 17 to that byte, and writing one byte at the resulting address. Its destination is therefore between addresses 17 and 272, inclusive—not necessarily within the object containing the selector.

Conditional arithmetic examples use 32-bit bit patterns modulo \(2^{32}\). Logical right shifts introduce zeros. This does not establish portable semantics for the represented signed shifts and signed multiplication.

Progress through the path requires each reached memory access and indirect call to be valid, and each dependency preceding a later operation to return. No validation of these conditions is represented.

## Behavior

### Ordering model

No explicit branch or loop appears in the supplied path. Calls occur throughout; their arguments below are the explicitly displayed arguments only. An apparently argument-free call is not proven to consume no machine-state inputs.

“After further calls” preserves the presence of intervening dependencies without assigning them invented identities. Those dependencies are not assumed to preserve memory or globals.

Saved values and subsequent memory reads are distinct. In particular, the two global reads must not be coalesced.

### 1. Initial dispatch and two distinct global captures

The path begins by reading a selector byte at the incoming address plus one, then writing the incoming byte at that selector plus 17.

The first indirect call receives the first explicit input. Its destination depends on incoming control state. After additional calls, another call receives the saved selector, widened without sign extension, and the third input shifted left by 26.

Further calls produce a 32-bit **initial word**. A call receives that word twice and the fourth input shifted left by 10. After another call, the initial word is stored in the word slot indexed by a secondary register result.

The immediately following memory-access order is:

1. The initial-word store.
2. A first read of the global word, retained as the **saved global value**.
3. A separate second read of the same global storage, multiplied by four to form the **global-derived address base**.
4. The next indirect call, receiving the preceding secondary result.

There is **no represented call between the store and either global read**. The store may nevertheless affect either read if its destination overlaps the global. Equality of the two read results is not established; memory or device semantics remain unknown.

After further calls, another index is obtained. The initial word logically shifted right by 12 is stored in that index’s word slot. The next call receives four times that index.

### 2. Byte fields using the second global capture

Further calls produce the **first byte payload**.

A selector is read at the global-derived address base plus one. Its directed destination receives the low byte of four times the first byte payload.

The next relevant call receives the **saved global value from the first global read**, masked with `0x3fffffff` and logically shifted right by 10. This argument does not use the second global capture.

After intervening calls produce another result, the selector is read again at the global-derived address base plus one. The newly selected destination receives the low byte of the first byte payload masked with `0x3fffffff` and shifted right by 20.

A subsequent call receives four times the intervening result. After further calls, another call receives four times a newer result and the most recently read selector.

Thus, the masked argument uses the first global capture, while both selector addresses use four times the second capture. The selector reads are independent observable accesses and can select different destinations.

### 3. Shift-derived address, duplicate reads, and clearing

The global-derived address base is shifted left by 26 to form the **shift-derived base**. Multiplying that base by 1,024 forms the **multiplied base**.

Following the preceding selector-bearing call and another call, two word-read expressions access the shift-derived base plus 28. Both values are supplied to one indirect call. Their relative evaluation order is not established; the representation does not justify replacing them with one snapshot.

After another call, a selector is read at the multiplied base plus one, and zero is written to its directed destination.

Under wrapping 32-bit arithmetic, the multiplied base is zero for every second global capture: multiplying by four, shifting left by 26, and multiplying by 1,024 discards all original bits. Consequently, that selector address is conditionally address one.

The next call receives zero, a secondary 32-bit register value, and the multiplied base.

### 4. Secondary-value field and direct dependency

After further calls, a selector is read again at the multiplied base plus one. Its directed destination receives the low byte of the preceding secondary value logically shifted right by 22.

A call receives four times an intervening result. After further calls, another receives the multiplied base shifted left by 26 and four times a newer result.

The following call produces the **retained word**. F-674e75a953e8 is invoked with the retained word in both explicit argument positions.

After a subsequent indirect call, the retained word is stored in the word slot indexed by the secondary result then exposed. This index is the **retained index**.

A call receives the retained index and its masked form: `0x3fffffff` followed by a logical right shift of 10. Further calls produce a destination index; the retained index is stored in that destination’s word slot. The next call receives four times the destination index.

No preservation, ownership, or ordinary return-value contract is established for F-674e75a953e8.

### 5. Another payload and an explicitly zero-based selector

Further calls produce the **second byte payload**.

A selector read at eight times the retained word plus one directs a write of the low byte of four times this payload. A call then receives four times the payload.

After intervening calls, a selector is read at 64 times the second byte payload plus one. Its directed destination receives the low byte of the payload masked with `0x3fffffff` and shifted right by 20.

A call receives four times an intervening result. After further calls, another receives four times a newer result and the saved selector from the 64-times-payload address.

Later, a call receives the retained word shifted left by 10 in both argument positions. After another call, a selector is read at address one and zero is written to its directed destination.

This address-one read follows from an explicitly retained zero used in the address calculation; it does not require a dependency to return zero.

### 6. Secondary result and split-width return

At the next relevant call boundary, a secondary 32-bit value is passed twice, followed by 16 times that value.

After further calls, a selector is read at 16 times that secondary value plus one. Its directed destination receives the low byte of the secondary value logically shifted right by 22.

A call receives four times an intervening result. Following further calls, another receives the saved selector first and four times a newer result second. After another call, zero is passed in both explicit argument positions.

The following call is represented as yielding 64 bits. Its **upper half** and **lower half** are each 32 bits. This partition does not establish an in-memory 64-bit layout.

Zero is stored in the word slot indexed by the upper half. A call receives that upper half. Further calls produce another destination index; the upper half is stored in that destination’s word slot. The next call receives four times that destination index.

### 7. Lower-half-derived stores

After additional calls, the lower half multiplied by `0x04000000` is stored at the address formed from the lower half multiplied by `0x10000000`, plus 20.

A call receives the stored product. Following further calls, the lower half masked with `0x1fffffff` and logically shifted right by nine is stored in a newly selected word slot. The next call receives four times that slot index.

The data product and address product retain different bits under wrapping arithmetic. Neither is an unrestricted mathematical integer.

### 8. Repeated scaled stores and publication

The next region contains five occurrences of a common address pattern. Each obtains an index, forms a **scaled value** equal to four times that index, and stores that value at 16 times the index plus 20.

The first four occurrences proceed as follows:

- **First occurrence:** After the scaled store, two word-read expressions access the scaled value plus 20. A call receives one read logically shifted right by 12, the scaled value, and the other read shifted left by two. The read-expression order is unspecified. After intervening calls, the scaled value is stored in a newly selected word slot; four times that publication index is passed onward.
- **Second occurrence:** After the scaled store, a call receives the scaled value twice and the earlier 64-bit result’s lower half shifted left by 26. Following calls, this scaled value is published into another word slot; four times its publication index is passed onward.
- **Third occurrence:** After the scaled store, a call receives the scaled value. Further calls precede publication of that value into another word slot and forwarding of four times the publication index.
- **Fourth occurrence:** After the scaled store, one word is read at the scaled value plus 20 and retained. A call receives the scaled value. Further calls produce the **fourth publication index**. The retained read, logically shifted right by 12, is stored in that index’s word slot, and four times the index is passed onward.

The scaled-store address and the subsequent read address generally differ: they use multipliers of 16 and four respectively before adding 20.

### 9. Fifth scaled store and final byte fields

After further calls, the fifth scaled store occurs. A call then receives, in order:

- Four times the fourth publication index, masked with `0x3f`, then shifted left by 14.
- The fifth scaled value.
- The fourth publication index shifted left by 30.

Following further calls, the fifth scaled value is published into another word slot. Four times this newer publication index is passed onward.

Later calls produce the **final byte payload**. The selector address for both final byte fields is `0x10000000` times the **fourth publication index**, plus one. It does not use the newer publication index.

The first selector read directs a write of the low byte of four times the final payload. A call receives that fourfold value twice and 16 times the payload.

The next result-producing call’s 32-bit result is explicitly forwarded to the following indirect call. Further calls occur before the selector is read again at the same arithmetic address. Its newly selected destination receives the low byte of the final payload masked with `0x3fffffff` and shifted right by 20.

A call receives four times an intervening result. After further calls, another receives four times a newer result and the latest selector. One final represented indirect call precedes the invalid-instruction marker.

## Side effects

Explicit side effects include byte writes to selector-derived low addresses, 32-bit stores to arithmetic-derived addresses, and dependency calls.

No bounds, null, alignment, permission, or destination-validity checks are represented. Locations may alias the global, selector bytes, previous stores, or dependency-visible memory. A saved value is distinct from a later reread of its originating location.

The initial-word store precedes both global reads without an intervening call. Later dependencies may affect memory observed by subsequent accesses, but must not be inserted conceptually between those three initial operations.

The evidence establishes no allocation, release, reference counting, ownership transfer, object lifetime, or buffer-size contract. Dependencies are not assumed to preserve globals or own or release memory.

No error-status branch, recovery, or rollback is represented. Invalid accesses, invalid call destinations, or non-returning dependencies may stop progress. Previously reached writes are not represented as being undone.

## Dependencies

### F-674e75a953e8

Receives the retained 32-bit word in both explicit argument positions. No ordinary result is explicitly assigned from this call.

Its return convention, effects, ownership behavior, preservation guarantees, and termination properties are **documented_only**.

### Unresolved indirect dependencies

All remaining calls use computed destinations. The first depends on incoming control state; later destinations use represented address constructions, predominantly a common displacement with some exceptions.

Opaque identities are unavailable for these calls. They remain unresolved rather than receiving invented function IDs.

Their displayed uses include 32-bit results, secondary register values, and one 64-bit result. These uses do not establish complete public signatures or a calling convention.

## Acceptance examples

These are conditional static examples, not executed tests. Each assumes execution reaches the relevant operation.

1. **Initial selector:** A selector of 255 and incoming byte `0xa5` produce a byte write of `0xa5` at address 272.

2. **Separate global captures:** Suppose the first global read yields `0x00000400` and the second yields 3. Under the stated arithmetic interpretation, the masked, right-shifted call argument is 1, while the early payload selector address is 13. Substituting either capture for the other violates the represented dependencies.

3. **Initial access order:** If the first word-store destination overlaps the global, that store occurs before both global reads. The next indirect call occurs after both reads, not between the store and a read.

4. **Byte truncation:** A first byte payload of `0x41` produces a fourfold value of `0x104`, but its byte write stores `0x04`.

5. **Masked upper field:** A payload of `0xffffffff` yields `0x3ff` after masking with `0x3fffffff` and shifting right by 20. The byte write stores `0xff`.

6. **Word extraction:** An initial word of `0x12345678` yields `0x00012345` for its right-shift-by-12 word store.

7. **Selector reread:** If two associated selector reads yield 3 and 9, their directed writes target addresses 20 and 26. Reusing the first selector is not equivalent.

8. **Wrapping collapse:** Under 32-bit wrapping arithmetic, the multiplied base is zero regardless of the second global read, making the corresponding selector address one.

9. **Split result:** An upper result half of 7 makes the immediate zero store target address 48. The lower half remains independently unconstrained.

10. **Different scaled locations:** A repeated-store index of 3 stores 12 at address 68. Its scaled-value-plus-20 read accesses address 32.

11. **Interrupted progress:** A dependency that does not return prevents later represented operations from being required. No completion status or rollback follows from the evidence.

## Uncertainties

The following remain **documented_only**:

- Whether the invalid-instruction region describes executable behavior.
- Actual calling convention and incoming machine-state provenance.
- Secondary-result timing, capture, and register-preservation details.
- Target semantics for signed overflow, shifts, and address formation.
- Endianness, alignment, mapped ranges, device-memory effects, and atomicity.
- Whether the two global reads return equal values; coalescing is not justified.
- Relative ordering of read expressions within a single argument list.
- Complete arguments, effects, ownership, exceptions, and termination of dependencies.
- The global storage’s relationship to overlapping symbols.
- Behavior beyond the truncation marker.

**Inferred intent:** repeated bit-field extraction and scaled-address patterns may be meaningful to a larger computation, but no domain interpretation is established or required.

## Evidence

The contract uses only the supplied static representation: displayed widths, access expressions, call sequencing, value dependencies, the mapped direct dependency, and control-flow warnings.

The distinct global captures follow from two separately represented reads: one retains the unscaled value, while the next supplies the fourfold address base. The initial word store precedes both, and the next call follows both.

No implementation access, instruction validation, runtime execution, or runtime tests have been performed. Acceptance examples express conditional arithmetic and ordering consequences, not observed execution.
