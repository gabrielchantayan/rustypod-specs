<!-- Trusted provenance: F-ae88479cbac7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ae88479cbac7

## Purpose

F-ae88479cbac7 performs an ordered sequence of computed-target calls, byte accesses, 32-bit word accesses, and one direct invocation of F-674e75a953e8. The supplied representation ends at an invalid-instruction condition; it contains no normal return.

No application-level purpose is established. Scaling, bit extraction, and repeated address forms do not establish an allocator, serializer, device interface, or other public abstraction.

This contract describes static evidence, conditional on execution reaching each operation. Instruction-data warnings and overlapping global storage limit confidence in the reconstruction. They do not justify replacing its unusual accesses with conventional object-relative operations.

## Interface

### Explicit arguments

The represented interface has four 32-bit arguments:

| Position | Interpretation | Explicit use |
|---|---|---|
| First | Opaque 32-bit value | Passed to the first computed-target call. |
| Second | Opaque 32-bit value | No explicit use shown. |
| Third | Signed 32-bit integer | Shifted left by 26 for an early call. |
| Fourth | Signed 32-bit integer | Shifted left by 10 for another early call. |

The unused second argument is not proof that its incoming register contents are irrelevant to unresolved dependencies.

### Additional effective inputs

The representation consumes incoming state beyond these arguments:

- An address from which a byte is read at displacement 1.
- A byte written through the first byte-derived destination.
- An unsigned 32-bit value whose logical right shift by 9 supplies another byte-read base.
- A link-related value used to construct the first call target.

Their initialization and calling convention are documented_only.

Later operations also consume secondary register results associated with calls. Their provenance and lifetime are not fully established. They must not be replaced with primary return values, zeros, or assumed preserved register contents.

### Results and preconditions

The displayed return type is void. No normal completion, status code, result object, or recovery interface is shown.

Reaching an operation requires its preceding dependencies to return and its memory accesses and computed call targets to be usable in the execution environment. The function performs no visible validation establishing those conditions.

### Widths and layout

Byte loads produce unsigned 8-bit values. Byte stores retain only the low 8 bits of their source. Word accesses are 32 bits. One call result is represented as 64 bits and separated into high and low 32-bit portions.

Addresses are numeric calculations, not established object layouts. Important forms include:

- A byte read at a computed base plus 1.
- A byte write at the loaded byte plus 17.
- A word access at four times an index plus 20.
- A word write at sixteen times an index plus 20.

Every byte-derived write destination is therefore between addresses 17 and 272, inclusive. Nothing establishes that this range is valid writable memory.

Endianness, alignment rules, allocation boundaries, and the relationship between numeric addresses and pointers are undocumented.

## Behavior

### Ordering and arithmetic interpretation

The stages below describe visible dataflow and ordering, not an inferred application algorithm. Calls with unused primary results remain observable operations; they cannot be removed as pure or redundant.

Captured values remain distinct from subsequent memory reads. Intervening dependencies may change memory or external state. Repeated loads must not be merged merely because their address expressions match.

Unsigned calculations use 32-bit values. The representation also contains signed multiplication and signed left shifts that can exceed the signed 32-bit range. Modular address simplifications below are conditional on 32-bit wrapping machine arithmetic; they are not claims that every displayed signed expression has defined portable semantics.

### Initial byte effects and primary results

The first explicit memory operation reads a byte at the hidden incoming address plus 1. Its value selects a destination by adding 17. The hidden incoming byte is then stored there. This destination is not relative to the original read address.

The first computed-target call receives the first explicit argument. After four further calls with no displayed arguments, a call receives the third argument shifted left by 26 and the initial byte, zero-extended.

A subsequent call produces the **first primary result**. Another call receives that result twice and the fourth argument shifted left by 10.

After another call, a byte is read at one plus the hidden unsigned entry value logically shifted right by 9. Zero is stored at that byte plus 17.

The first primary result logically shifted right by 12 is retained for a later byte-read base.

### Secondary-result update and second primary result

A call receives a secondary result twice and zero in its third displayed argument position. Further calls produce a primary result that will be shifted left by 2 and passed onward.

Before that shifted argument is delivered, the function reads a byte at one plus the retained right-shifted first primary result. It writes a secondary result logically shifted right by 22 to the loaded byte plus 17.

Because this is a byte store, only source bits 22 through 29 survive; bits 30 and 31 do not.

Later calls include one receiving the byte selector just loaded and zero. A subsequent call produces the **second primary result**. Another receives that result twice and the first primary result shifted left by 10.

After another call, the second primary result is stored as a 32-bit word at four times a secondary result plus 20.

### Two distinct global reads

Immediately after that word store, the representation reads the same global storage twice, in order:

1. The first read supplies a saved 32-bit value.
2. The second read supplies a value multiplied by four, producing a separately retained scaled base.

These are separate reads, not one read retained in two forms. No intervening call is shown, but equality still requires a stability assumption about the storage and execution environment.

Their downstream uses differ:

- The first saved value is masked to 30 bits and logically shifted right by 10 for a later call argument.
- The second read determines the scaled base used by subsequent byte accesses and shift-derived addresses.

No semantic role or complete declaration for the overlapping global storage is supplied.

### Global-derived byte updates

After intervening calls, a call-produced index selects a word destination at four times that index plus 20. The value stored is the second primary result logically shifted right by 12.

Further calls produce a new 32-bit value. A byte is read at the retained scaled base plus 1, and the low byte of the new value shifted left by 2 is written to the loaded byte plus 17.

A subsequent call receives the masked, right-shifted value from the **first** global read.

After more calls, the function separately rereads the byte at the scaled base plus 1. It writes the low byte of the new value, masked to 30 bits and logically shifted right by 20, to the newly selected destination. This stores source bits 20 through 27.

The two byte selectors need not agree. Calls or other memory activity may change the source between reads.

### Shift-derived addressing and direct dependency

The scaled base derived from the **second** global read is shifted left by 26. A call receives two separately displayed word loads from that shifted base plus 28.

Both loads precede that invocation, but their relative evaluation order is not established. They are not guaranteed to observe one snapshot.

The shifted base is also multiplied by 1024. At that product plus 1, the function reads a byte and writes zero to the loaded byte plus 17.

A later call receives zero, a secondary result, and the retained product. After additional calls, the product-derived byte address is read again. A secondary result logically shifted right by 22 supplies the byte written to the newly selected destination.

Further calls consume shifted primary results and the retained product. A subsequent primary result becomes the **direct-call input**. F-674e75a953e8 receives that input in both displayed argument positions. No explicit use of its return value is shown.

After another computed-target call:

- The direct-call input is stored at four times a secondary result plus 20.
- A call receives that secondary result and its low-30-bit value logically shifted right by 10.
- Following more calls, the represented secondary value is stored at four times a new call-produced index plus 20.

These effects establish no preservation or ownership guarantee for F-674e75a953e8.

### Further byte updates and zero-derived addressing

A later call produces another 32-bit value. A byte is read at eight times the direct-call input plus 1. Four times the new value supplies the byte written to the resulting destination.

After more calls, a byte is read at 64 times the new value plus 1. The next byte store selects bits 20 through 27 of that same value.

Subsequent calls pass shifted results and byte selectors. One call receives the direct-call input shifted left by 10 twice.

The next explicit byte read is at address 1. Its base is an explicitly zero-valued intermediate multiplied by \(2^{26}\). Zero is written to the loaded byte plus 17.

A later call receives a secondary result twice and sixteen times that result. Following further calls, a byte is read at sixteen times the represented secondary value plus 1. That value logically shifted right by 22 supplies the byte store.

Calls then consume the byte selector and shifted primary results. A later call receives zero in both displayed argument positions.

### Split result and word stores

The next call produces a result represented as 64 bits. Its portions have distinct uses:

1. Zero is stored at four times the high 32-bit portion plus 20.
2. After calls, the high portion is stored at four times a new call-produced index plus 20.
3. After further calls, the low portion multiplied by \(2^{26}\) is stored at the low portion multiplied by \(2^{28}\), plus 20.
4. That \(2^{26}\)-scaled low portion is also passed to a call.
5. Following more calls, the low portion masked to 29 bits and logically shifted right by 9 is stored at another four-times-index destination.

The 64-bit representation does not establish that the two portions form one application-level quantity. A paired-register return convention remains possible but unproven.

### First three scaled-index occurrences

Each of the next three occurrences has the following memory pattern:

- A call produces an index.
- Four times that index is retained.
- The retained value is stored at sixteen times the original index plus 20.
- After intervening calls, that retained value is stored at four times another call-produced index plus 20.

Their intervening call arguments differ:

- **First occurrence:** A call receives two separately displayed loads from the retained value plus 20, one logically shifted right by 12 and the other shifted left by 2, with the retained value between them. Their relative load order and agreement are not established.
- **Second occurrence:** A call receives the retained value twice and the earlier split result’s low portion shifted left by 26.
- **Third occurrence:** A call receives the retained value once.

The sixteen-times-index destination and four-times-index destination are different calculations. Neither establishes a structure field or array boundary.

### Fourth occurrence: captured-word transfer

The fourth occurrence starts similarly: a call-produced index is multiplied by four, and that scaled value is stored at sixteen times the index plus 20.

Its continuation is different from the first three:

- A word is read at the retained scaled value plus 20, before the following calls.
- A call receives the retained scaled value.
- After intervening calls produce a new destination index, the **captured word logically shifted right by 12** is stored at four times that destination index plus 20.

The retained scaled index is not the value of this later store. Changes to the source word after its capture do not replace the captured value.

Call the new destination index the **transfer index**, and its four-times-scaled value the **transfer scale**. Both are retained for later uses.

### Fifth occurrence and bit fields

A fifth call-produced index is scaled by four. The scaled value is stored at sixteen times that index plus 20.

A subsequent call receives:

- The low six bits of the transfer scale, shifted left by 14.
- The fifth occurrence’s scaled value.
- The transfer index shifted left by 30.

Under 32-bit wrapping arithmetic, the transfer scale’s low two bits are zero. Thus the first argument is more constrained than an arbitrary six-bit field shifted left by 14.

After additional calls, the fifth occurrence’s scaled value is stored at four times another call-produced index plus 20.

### Final byte updates and termination

A later call produces the **final byte-source result**. A byte is read at the transfer index multiplied by \(2^{28}\), plus 1. The final result shifted left by 2 supplies the byte stored at the loaded byte plus 17.

A call receives that shifted value twice and the final result shifted left by 4. A following call result is explicitly forwarded to the next call.

After further calls, the transfer-index-derived byte address is read again. Bits 20 through 27 of the final byte-source result supply the next byte store.

Additional calls consume shifted primary results and the most recently loaded byte selector. The represented path then terminates at invalid instruction data, without a normal return.

## Side effects

Explicit memory effects include unsigned byte reads, byte writes, and 32-bit reads and writes. No visible bounds, null, alignment, or permission checks precede them.

Repeated addresses may alias each other, global storage, or externally meaningful memory. No disjointness guarantee is available.

Under 32-bit modular arithmetic, shifting the retained four-times-global base left by 26 and then multiplying by \(2^{10}\) produces zero. The associated product-derived byte reads consequently use address 1. This simplification remains conditional because the representation includes signed operations.

Other large address calculations can also wrap. No saturation, rejection, or overflow-reporting behavior is shown.

Dependencies may introduce additional effects. The representation establishes no allocation, release, ownership transfer, rollback, transaction boundary, or cleanup guarantee. A fault or non-returning dependency can leave preceding writes observable.

## Dependencies

- **F-674e75a953e8:** Invoked once with the same captured 32-bit value twice. Its implementation, memory effects, register effects, ownership obligations, and completion behavior are documented_only.
- **Unresolved computed-target dependencies:** No stable opaque function IDs or complete contracts are supplied. Their targets, signatures, secondary results, and effects remain documented_only. Empty displayed argument lists do not establish complete machine-level interfaces.
- **Execution environment:** Supplies hidden entry state, global storage, address-space behavior, and computed-call execution. No operating-system, device, or runtime contract is established.

No dependency is assumed to preserve globals or own, retain, or release memory.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Initial destination.** If the first byte read yields 255 and the incoming stored byte is 171, the first explicit write stores one byte, 171, at address 272.

2. **Distinct global observations.** If the first global read yields 1024 and the second yields 3, the saved-value call argument is 1, while the scaled base is 12 and its byte-read address is 13. Deriving both from either observation is incorrect.

3. **Byte truncation.** A secondary result of \(0xFFFFFFFF\), logically shifted right by 22 for a byte store, produces \(0xFF\), not a ten-bit write.

4. **Scaled low byte.** A value of 65 shifted left by 2 produces stored byte 4. The destination still comes from a separate byte read.

5. **Reread visibility.** If dependencies change a selector source between two reads, each subsequent write uses its own selector. Reusing the earlier selector is not equivalent.

6. **Fourth-occurrence transfer.** Suppose its original index is 3. It stores 12 at address 68, then reads a word at address 32. If that captured word is \(0x12345000\) and the transfer index is 7, the later store writes \(0x00012345\) at address 48—not 12. A subsequent change at address 32 does not change the captured value.

7. **Split-result order.** The zero word store indexed by the high portion precedes the later store of that high portion, with the intervening calls preserved.

8. **Conditional arithmetic collapse.** With 32-bit modular arithmetic, the shift-then-multiply product derived from the second global read is zero, so its byte-read address is 1.

9. **Partial effects and termination.** A fault in the first byte-derived write need not permit later operations. If execution instead reaches the represented end, ordinary successful return is not an equivalent replacement for the invalid-instruction condition.

## Uncertainties

The following remain documented_only:

- Accuracy and completeness of the reconstructed control flow.
- Application purpose and intended public abstraction.
- Complete calling convention, hidden inputs, and secondary-result attribution.
- Computed-target dependency identities and contracts.
- Signed overflow, signed shifting, and actual address-width semantics.
- Global declaration, overlap behavior, and stability between its two reads.
- Endianness, alignment, permissions, concurrency, and device-memory behavior.
- Ownership, allocation, release, exception handling, and cleanup obligations.
- Whether executable behavior reaches the represented terminal condition.

No memory-safety, normal-completion, or functional-usefulness guarantee follows.

## Evidence

**Established static evidence:** Four explicit 32-bit arguments; hidden incoming state; ordered byte and word effects; two consecutive, distinct global reads with separate downstream uses; numerous computed-target calls; one direct call to F-674e75a953e8; a split 64-bit result; three scaled-value transfers followed by a different captured-word transfer; and terminal invalid instruction data.

**Inferred intent:** Repeated scaling and bit extraction may manipulate packed or address-like quantities. No such interpretation is adopted as a behavioral requirement.

**Evidence limits:** The supplied representation reports problematic instruction data and overlapping global symbols. No dependency implementations or independent executable evidence are available. No runtime tests have been performed.
