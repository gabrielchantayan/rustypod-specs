<!-- Trusted provenance: F-634f9fbe35da; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-634f9fbe35da

## Purpose

F-634f9fbe35da copies a four-word descriptor and an associated value into a destination object, sets a state byte, computes an aligned storage quantity, attempts to obtain a nonzero storage handle, and delegates final setup to F-411b2d4e0f85.

The operation is not atomic. Destination updates precede storage acquisition. A zero result from fallback acquisition causes an early return without rolling back those updates.

The descriptor resembles a pair of bounds, and the arithmetic resembles a stride and total-size calculation. Those meanings are inferred intent, not established semantics. This contract therefore describes descriptor words, arithmetic quantities, and opaque handles without assigning geometric units or ownership rules.

## Interface

### Arguments

| Argument | Representation | Established use |
|---|---|---|
| Destination | 32-bit address-valued argument | Base of directly accessed fields; also passed to final setup |
| Descriptor | Address of four consecutive 32-bit words | Read repeatedly, partly copied into the destination, and passed to dependencies |
| Resource value | Opaque 32-bit value | Passed twice to F-b4551e3e3ef6 |
| Associated value | Opaque 32-bit value | Stored in the destination and passed to F-b4551e3e3ef6 |
| Final context | Opaque 32-bit value | Passed to F-411b2d4e0f85 |

Descriptor words are designated D0, D1, D2, and D3, at byte offsets 0, 4, 8, and 12 from the descriptor address. These designations identify memory positions, not immutable entry-time values.

No explicit checks establish pointer validity, alignment, object extent, nonoverlap, or suitable descriptor ranges.

### Directly accessed destination layout

| Byte offset | Width | Direct access |
|---|---:|---|
| 0x98 | 32 bits | Written from D0 |
| 0x9c | 32 bits | Written from a captured D1 |
| 0xa0 | 32 bits | Written from a captured D2 |
| 0xa4 | 32 bits | Written from a captured D3 |
| 0xa8 | 32 bits | Written with the associated value |
| 0xb0 | 32 bits | Read as an existing handle; cleared on the fallback path |
| 0xb4 | 8 bits | Written with one |

The address at destination plus 4 is passed to the final dependency. Its layout and required extent are not established.

The state update at 0xb4 is a one-byte write. It does not establish a write to the remaining bytes of any larger field.

### Output and preconditions

There is no explicit return value. In particular, the early return after failed fallback acquisition does not return an error status.

Normal completion requires accessible memory for the explicit reads and writes, valid address calculations, and satisfaction of dependency-specific preconditions. The caller does not enforce those requirements.

Descriptor and destination overlap is not explicitly prohibited. Such overlap can change later descriptor reads through this function’s own writes, even when every dependency leaves memory unchanged.

## Behavior

### Initial dependency

F-854a6c2c6fa3 is called without explicit arguments before any visible descriptor read or destination-field write.

Its effects are unknown. Descriptor values used afterward are therefore not necessarily the values present at function entry.

### Descriptor capture and destination updates

After the initial call returns, the function captures D1, then D2, then D3.

It next reads D0 and writes that value to destination offset 0x98. It then writes the captured D1, D2, and D3 values to offsets 0x9c, 0xa0, and 0xa4, respectively, in that order.

The associated value is written at offset 0xa8. The byte at offset 0xb4 is then set to one.

Consequences of this ordering:

- All four values used for the descriptor copy are read before the first destination write.
- The D1 and D2 values passed to the next dependency are the captured values, not values reread after destination updates.
- Later arithmetic uses fresh descriptor accesses rather than the copied values.
- Overlapping destination writes can therefore change arithmetic inputs while leaving the earlier captured values unchanged.
- The state byte is set before storage acquisition succeeds or fails. It is not, by itself, evidence of successful final setup.

### Resource-related call

F-b4551e3e3ef6 receives seven arguments in this order:

1. Resource value.
2. Zero.
3. Captured D1.
4. Captured D2.
5. Original descriptor address.
6. Resource value again.
7. Associated value.

No result from this dependency is consumed. Its effects on memory, resources, ownership, and control flow are unknown.

### Aligned quantity

F-8939cedb2807 is called without explicit arguments. Its 32-bit result is retained both for the following arithmetic and for final setup. Denote this result by B.

The function freshly reads D3 and D1 after that call and forms the signed 32-bit difference D3 − D1. The supplied representation does not independently establish the relative evaluation order of those two reads.

For representable signed intermediate arithmetic, define:

- T = B × (D3 − D1) + 7.
- Q = T divided by 8, truncated toward zero.
- S = the 32-bit unsigned value obtained by adding 3 to Q modulo 2³² and clearing the lowest two bits.

The displayed arithmetic obtains Q through a sign-dependent adjustment and signed right shift. Under arithmetic signed-right-shift semantics, it adds no adjustment for nonnegative T and an adjustment of seven for negative T before shifting by three.

The final mask is 0xfffffffc. S is consequently a 32-bit unsigned quantity with its lowest two bits clear.

For ordinary nonnegative Q, where adding three does not wrap, S is Q rounded upward to a multiple of four. That interpretation must not be extended indiscriminately to negative or wrapping values.

No explicit validation rejects:

- Negative descriptor differences.
- Zero or negative B.
- Overflow in signed descriptor subtraction.
- Overflow in signed multiplication or addition.
- A storage quantity that wraps in unsigned arithmetic.

The representation does not establish a portable result for overflowing signed operations. Such cases remain documented_only.

### Existing-handle path

The 32-bit field at destination offset 0xb0 is tested.

If it is zero, the existing-handle dependency is skipped.

If it is nonzero, the field is read for use as a handle argument. Fresh descriptor reads supply D2 and D0 for a signed difference D2 − D0. Their relative evaluation order is not separately established.

The requested storage quantity is S multiplied by that difference. Provided the signed subtraction is representable, the difference is converted to unsigned 32-bit arithmetic for multiplication by S. The resulting quantity is the low 32 bits of the product; unsigned overflow is not rejected.

F-bc7ecdbf25ab receives:

1. The existing handle.
2. This storage quantity.
3. The constant 0x1c.

A nonzero result becomes the usable handle. It need not equal the original handle. A zero result selects fallback acquisition.

### Fallback path

Fallback occurs when the existing-handle field is zero or F-bc7ecdbf25ab returns zero.

The first direct action on this path is a 32-bit zero write to destination offset 0xb0.

Only afterward does the function freshly read D2 and D0 and recompute the storage quantity using the retained S. Therefore, either of the following can change the fallback quantity relative to an earlier reuse request:

- F-bc7ecdbf25ab modifies descriptor memory before returning zero.
- The direct clear at destination offset 0xb0 overlaps descriptor memory used by the new calculation.

S and B are not recomputed on this path.

F-638beded7699 receives:

1. The recomputed 32-bit storage quantity.
2. The constant 0x1c.

A zero result causes an immediate return without calling F-411b2d4e0f85. A nonzero result becomes the usable handle.

This caller does not directly install the returned handle at destination offset 0xb0.

### Final setup

After either storage dependency returns a nonzero handle, F-411b2d4e0f85 receives:

1. Destination address plus 4.
2. Original descriptor address.
3. The successful nonzero handle.
4. S.
5. B.
6. Original destination address.
7. Final context.

The descriptor argument is its original address, not a snapshot of its earlier contents. The dependency can therefore encounter descriptor contents changed by prior calls or overlapping destination writes.

No result from final setup is consumed. No subsequent direct destination write is shown.

## Side effects

The direct descriptor-copy writes, associated-value write, and state-byte write occur before the storage-success decision. They are not undone on fallback failure.

Fallback directly clears the existing-handle field before making its storage request. If the fallback dependency returns zero without changing that field, the field remains zero.

Direct writes may also modify descriptor words when the two memory regions overlap. This includes the initial destination updates and the later fallback clear. Their effects must be considered at each subsequent descriptor read, not merely at entry.

No separate release operation is visible. A failed existing-handle request is followed by clearing the field, but that alone establishes neither a leak nor a release: F-bc7ecdbf25ab may have ownership effects not supplied here.

Likewise, passing a successful handle to final setup does not establish ownership transfer, retention, installation, or eventual release. No unknown dependency is assumed to preserve globals, descriptor contents, destination fields, or resource ownership.

There is no explicit diagnostic, rollback, or retry loop. Faults, exceptions, nonlocal exits, and dependency-specific error channels remain outside the established behavior.

## Dependencies

| Opaque ID | Established interaction | Unresolved semantics |
|---|---|---|
| F-854a6c2c6fa3 | First call; no explicit arguments | Initialization, synchronization, memory and global effects |
| F-b4551e3e3ef6 | Seven arguments after destination updates; result unused | Resource behavior, mutation, ownership, failure |
| F-8939cedb2807 | No explicit arguments; 32-bit result retained as B | Meaning, units, range, side effects |
| F-bc7ecdbf25ab | Called only after a nonzero existing-handle test; receives handle, quantity, and 0x1c | Resize or replacement behavior, ownership, failure effects |
| F-638beded7699 | Called after the fallback clear; receives quantity and 0x1c | Acquisition semantics, zero-size behavior, ownership |
| F-411b2d4e0f85 | Called only with a nonzero storage result; receives seven arguments | Final state changes, installation, ownership, errors |

These identities establish call relationships, not implementation contracts for the dependencies.

## Acceptance examples

These are conditional static expectations, not executed tests.

Unless explicitly overridden, every example assumes:

- All required memory accesses and address calculations are valid.
- Signed intermediate arithmetic is representable, with the signed-shift interpretation described above.
- Dependencies return normally and do not change relevant memory except where stated.
- Direct destination writes do not alter descriptor words used by subsequent reads. This assumption includes the descriptor-copy writes, associated-value write, state-byte write, and fallback clear at offset 0xb0. Disjoint descriptor and destination storage is sufficient to satisfy it.

### No existing handle; fallback succeeds

Assume D0 = 10, D1 = 20, D2 = 18, D3 = 25, B = 3, and the existing-handle field is zero.

The fresh difference D3 − D1 is 5. T is 22, Q is 2, and S is 4. The fallback storage quantity is 4 × (18 − 10) = 32.

Expected behavior:

- Destination descriptor fields receive 10, 20, 18, and 25.
- The associated value is stored and the state byte becomes one.
- F-bc7ecdbf25ab is skipped.
- Offset 0xb0 is cleared before the fallback quantity is read and calculated.
- F-638beded7699 receives 32 and 0x1c.
- If it returns a nonzero handle, final setup receives that handle, S = 4, and B = 3.

### Existing handle succeeds

Under the same descriptor values and B, assume a nonzero existing handle.

F-bc7ecdbf25ab receives the existing handle, 32, and 0x1c. If it returns nonzero:

- F-638beded7699 is skipped.
- This caller does not directly clear offset 0xb0.
- Final setup receives the returned handle, even if it differs from the original handle.

### Existing-handle failure changes the descriptor

Assume S has already been computed as 4. The existing-handle request uses quantity 32. Before returning zero, F-bc7ecdbf25ab changes the descriptor so that the next D2 − D0 difference is 12.

Provided the fallback clear does not interfere with those descriptor words, F-638beded7699 receives 48 and 0x1c. S remains 4.

### Direct aliasing changes the numerical result

This example intentionally overrides the direct-write noninterference assumption.

Place the descriptor at destination plus 0x90, initially containing 10, 20, 18, and 25. Let B be 3, the existing-handle field be zero, and dependencies make no relevant memory changes.

D2 overlaps destination offset 0x98, and D3 overlaps destination offset 0x9c. The initial captures retain D1 = 20, D2 = 18, and D3 = 25. The subsequent destination writes change descriptor D2 to 10 and D3 to 20.

The arithmetic therefore observes D3 − D1 = 0, giving T = 7, Q = 0, and S = 0. The fallback request is zero, not 32.

The destination copy still receives the earlier captured D2 and D3 values at offsets 0xa0 and 0xa4. Captured copy values and later arithmetic inputs need not agree.

### Fallback acquisition fails

If F-638beded7699 returns zero, final setup is skipped. There is no returned status or rollback. The state byte was already written with one.

If dependencies do not modify the directly written destination fields, those writes remain in place, including the zero at offset 0xb0.

### Zero difference

If D3 equals D1 at the reads used for alignment, T is 7, Q is zero, and S is zero. For a representable later D2 − D0 subtraction, the requested quantity is zero.

The function still follows its existing-handle or fallback path. Whether a zero-size request succeeds is dependency-defined.

### Negative intermediate

If B = 8 and the observed D3 − D1 difference is −2, T is −9 and Q is −1. Unsigned addition of three followed by low-bit clearing gives S = 0.

This demonstrates the signed-division and unsigned-alignment sequence; it does not establish that negative differences are valid inputs.

## Uncertainties

The following remain documented_only:

- Semantic meanings and units of descriptor words, B, S, and 0x1c.
- Accepted descriptor ranges and negative-span validity.
- Behavior outside representable signed arithmetic and the underlying signed-shift and conversion conventions.
- Required alignment, destination extent, and nested layout at destination plus 4.
- Relative evaluation order of reads within individual arithmetic expressions.
- Dependency mutation of memory, globals, or external resources.
- Ownership and lifetime of original and returned handles.
- Meaning of the state byte.
- Dependency behavior for zero or wrapped storage quantities.
- Concurrency, synchronization, and visibility guarantees.
- Error reporting beyond the visible zero/nonzero storage decisions.

Aliasing effects are not inherently unknown: where overlap and values are specified, the explicit write order constrains subsequent reads. Unknown overlap must not be silently treated as nonoverlap.

## Evidence

Established claims derive from static inspection of the supplied function representation: argument widths, memory access widths and offsets, sequential writes, explicit calls, branch conditions, arithmetic operations, and return structure.

The aliasing example follows from the overlap between descriptor positions and destination write offsets. It requires no dependency-induced mutation.

Geometric interpretation, stride terminology, and allocation-like intent remain inferences rather than dependency guarantees.

No runtime tests have been performed. All acceptance examples are conditional consequences of the supplied static representation, not observations of execution.
