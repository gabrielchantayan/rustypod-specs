<!-- Trusted provenance: F-21aadcb2ba6f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-21aadcb2ba6f

## Purpose

F-21aadcb2ba6f performs a represented sequence of indirect calls, selector-based byte writes, indexed 32-bit writes, and one direct call to **F-674e75a953e8**. The supplied static evidence does not establish an application-level purpose.

The routine depends on incoming machine state beyond its four displayed arguments. It also consumes secondary call results and two separately displayed evaluations of an external 32-bit storage location. Its memory accesses include low absolute addresses; no containing object or allocation is established.

The representation warns of invalid instruction data and ends at an instruction-decoding termination marker, not a normal return. This contract describes the represented path, conditional on each preceding call returning and each preceding access succeeding. It does not establish that the apparent function boundary or entire path is executable.

Interpretations such as serialization, allocation, object construction, device programming, or cleanup are **documented_only**.

## Interface

### Explicit arguments

| Argument | Displayed width and interpretation | Established use |
|---|---|---|
| First | 32 bits, otherwise untyped | Passed to the first indirect call. |
| Second | 32 bits, otherwise untyped | No explicit use in the represented body. |
| Third | Signed 32-bit integer | Shifted left by 26 for an early indirect call. |
| Fourth | Signed 32-bit integer | Shifted left by 10 for another early indirect call. |

The absence of an explicit use of the second argument does not establish that unknown dependencies ignore its incoming machine representation.

### Additional inputs and preconditions

The represented path requires:

- An incoming address-like value for the initial byte read at offset 1.
- An incoming byte-like value for the initial byte write.
- An incoming link/control value used to derive the first indirect target.
- Readable external 32-bit storage, evaluated twice consecutively after the first indexed word write.
- Ordinary and secondary machine-register results from indirect calls.
- Accessible memory at addresses derived from those values, including low addresses.

The two external-storage evaluations are separate in the representation. The first supplies an unscaled unsigned 32-bit capture. The second supplies a fourfold intermediate represented as signed 32-bit. They must not be merged into one capture or assumed equal.

No dependency call intervenes between those evaluations. Nevertheless, the evidence does not establish whether they correspond to distinct hardware transactions or whether the storage can change between them.

### Outputs, errors, and termination

The displayed interface declares no return value, and no normal return is visible. Outputs consist of direct memory writes and effects of dependencies.

There is no visible validation, error-result branch, rollback, or recovery path. The terminal bad-instruction marker does not establish whether actual execution traps, faults, transfers control, or continues outside the representation.

### Widths and address notation

Direct memory accesses are one byte or four bytes wide. A selector is an unsigned byte. A selector-directed byte destination is address **17 plus the selector**, ranging from 17 through 272 inclusive. Byte writes retain the low eight bits of the displayed value.

A **word slot indexed by a value** means address **20 plus four times that value**, accessed for four bytes. This is address notation, not evidence of an array, allocation, bounds, or ownership.

Unsigned right shifts are logical. Unsigned 32-bit arithmetic retains 32 bits. Signed shifts, signed scaling, conversions into signed intermediates, and pointer construction require machine-level interpretation not supplied here. Conclusions involving those operations use explicit arithmetic assumptions where necessary.

## Behavior

### Entry access and initial word

The first direct access reads one byte at the incoming address-like value plus 1. The routine writes the incoming byte-like value to address 17 plus that selector before making any dependency call.

The first indirect target is derived from the incoming link/control value. That call receives the first explicit argument. Four further indirect calls precede a call receiving the zero-extended entry selector and the third argument shifted left by 26.

After another indirect call, a dependency supplies the **initial word**, a displayed unsigned 32-bit result. A subsequent call receives that word twice and the fourth argument shifted left by 10. Another call supplies a secondary result used as the index of a word-slot write containing the initial word.

### Two external-storage evaluations

Immediately after that write, the representation evaluates the same external storage twice, in this order:

1. A 32-bit value is captured without scaling as an unsigned value, here called the **unscaled external capture**.
2. The storage is evaluated again, and four times this second evaluation is retained in a signed 32-bit intermediate, here called the **scaled external capture**.

No dependency call occurs between these evaluations. The following dependency call receives the secondary index used for the initial word-slot write.

These captures have separate dataflow:

- The unscaled external capture supplies the later argument formed by masking to 30 bits and shifting right by 10.
- The scaled external capture supplies the first pair of external-derived selector addresses and the subsequent left-shifted and multiplied address calculations.

Later dependency effects on external storage do not constitute additional evaluations of that storage in the represented path.

### Initial extraction and first byte-source word

Following an indirect-call chain, an index-producing call selects a word slot. The routine writes the initial word logically shifted right by 12 to that slot, then passes four times its index to another dependency.

A later call supplies the **first byte-source word**, an unsigned 32-bit value.

The routine reads a selector at the scaled external capture plus 1 and writes the low byte of four times the first byte-source word to address 17 plus that selector. The following call receives the unscaled external capture masked to its low 30 bits and logically shifted right by 10.

After additional calls, including an index-producing call, the routine rereads the selector at the scaled external capture plus 1. It writes the low byte of the first byte-source word masked to 30 bits and shifted right by 20.

These selector reads occur on opposite sides of dependency calls. Their values and resulting byte destinations need not match.

The next call receives the intervening index-producing result shifted left by two. After another call chain, a call receives another result shifted left by two and the most recently read selector.

### Shifted external-derived addresses

The scaled external capture is shifted left by 26. This shifted intermediate supplies an address for two separately displayed four-byte reads, each at the shifted intermediate plus 28. An indirect call receives both read values.

The two expressions address the same location. Their relative evaluation order and their correspondence to distinct hardware transactions are not established. Neither read may simply be replaced with a presumed preserved value.

The shifted intermediate is also multiplied by 1,024. After the two-read call and another indirect call, the routine reads a selector at this multiplication-derived value plus 1 and writes zero to the selected byte destination.

A subsequent call receives zero, a secondary unsigned 32-bit result, and the multiplication-derived value. After more calls, the routine rereads the selector at that same address and writes the low byte of that secondary result logically shifted right by 22.

Further calls receive returned values shifted left by two. A later call receives the multiplication-derived value shifted left by 26 together with another shifted result.

If these signed intermediate operations are interpreted as 32-bit wrapping operations, the multiplication-derived value is zero: a 26-bit left shift followed by multiplication by 1,024 discards all remaining bits. Under that assumption, its selector address is 1. The signed representation alone does not establish that arithmetic model.

### Direct dependency and central-value writes

An indirect call supplies the **central value**, represented as signed 32-bit. The routine directly calls **F-674e75a953e8**, passing the central value twice.

A subsequent indirect call supplies a secondary unsigned 32-bit result. The routine writes the central value to the word slot indexed by that secondary result.

The following call receives the secondary result and that result masked to 30 bits and logically shifted right by 10. After additional calls, another returned index selects a word-slot write containing the secondary result itself. Four times this new index is passed to the following dependency.

No allocation, release, transfer, or preservation guarantee is established for F-674e75a953e8.

### Second byte-source word and further selectors

A later indirect call supplies the **second byte-source word**, an unsigned 32-bit value.

The routine reads a selector at eight times the central value plus 1. It writes the low byte of four times the second byte-source word to the selected destination. The next call receives that fourfold word value.

After another call chain, the routine reads a selector at 64 times the second byte-source word plus 1. It writes the low byte of that word masked to 30 bits and shifted right by 20. Subsequent calls receive shifted dependency results; one also receives this selector.

A later call receives the central value shifted left by 10 in both displayed argument positions. After another call, an explicitly zero-valued intermediate produces a selector read at address 1. The routine writes zero to the selected destination.

The next call receives a new secondary unsigned 32-bit result twice and that result multiplied by 16. Following more calls, a selector is read at 16 times this secondary result plus 1. The routine writes the low byte of the secondary result logically shifted right by 22.

That selector is later passed to a dependency together with another result shifted left by two.

### Split result and low-half scaling

After another call, a dependency receives zero in both displayed argument positions. The following call supplies a displayed 64-bit result.

The result is separated into:

- A low unsigned 32-bit half.
- A high 32-bit half represented as signed.

The routine writes zero to the word slot indexed by the high half. It passes that high half to a dependency and, after a call chain, writes the high half into another dependency-selected word slot. The next call receives four times that destination index.

After further calls, the routine writes a four-byte value using the retained low half:

- Destination: 20 plus the low half multiplied by 268,435,456.
- Stored value: the low half multiplied by 67,108,864.

These low-half products are unsigned 32-bit expressions. They are not unbounded mathematical products or ordinary unscaled dereferences.

The latter product is passed to a dependency. Following another call chain, a returned index selects a word-slot write containing the low half masked to 29 bits and logically shifted right by nine. Four times this index is then passed to a dependency.

### Five scaled-index groups

The remaining word-oriented section contains five groups. Each obtains a dependency result, forms its fourfold value, and writes that value to address **20 plus sixteen times the original result**. This destination differs from an ordinary word slot indexed by the original result.

Their observable order and distinguishing behavior are:

1. **First group.** After the sixteenfold-address write, two four-byte reads are displayed at the fourfold value plus 20. A call receives one read logically shifted right by 12, the fourfold value, and the other read shifted left by two. Their relative read order is unresolved. Later calls select an ordinary word slot into which this group’s fourfold value is written.

2. **Second group.** After its sixteenfold-address write, a call receives the group’s fourfold value twice and the retained low half of the earlier 64-bit result shifted left by 26. A subsequent dependency-selected ordinary word slot receives the group’s fourfold value.

3. **Third group.** Its sixteenfold-address write precedes a call receiving its fourfold value. After more calls, an ordinary word slot receives that value.

4. **Fourth group.** Its sixteenfold-address write is followed immediately by a four-byte read at its fourfold value plus 20, before the following dependency chain. A later ordinary word-slot write contains this captured word logically shifted right by 12.

5. **Fifth group.** Following its sixteenfold-address write, a call receives three arguments: the low six bits of the preceding ordinary slot’s fourfold index shifted left by 14; this group’s fourfold value; and the preceding ordinary slot’s original index shifted left by 30. Later calls select an ordinary word slot receiving the fifth group’s fourfold value.

Calls following the ordinary slot writes receive the corresponding fourfold destination indices.

The fourth group’s captured read is explicitly after its store. If their addresses overlap, that ordering is material. The first group’s reads also follow its store.

### Final byte-write section

A later dependency supplies the **final byte-source word**, an unsigned 32-bit value.

The selector address is 1 plus 268,435,456 times the index used for the ordinary slot containing the fourth group’s right-shifted captured word. The routine reads that selector and writes the low byte of four times the final byte-source word to the selected destination.

An indirect call receives the fourfold final word twice and the final word shifted left by four. A subsequent call supplies a result passed to another dependency.

After further calls, the routine rereads the same selector address. It writes the low byte of the final byte-source word masked to 30 bits and logically shifted right by 20.

The remaining visible calls receive additional dependency results shifted left by two, including a call that also receives the reread selector. Another indirect call precedes the terminal invalid-instruction marker.

## Side effects

Established direct effects are ordered byte and four-byte accesses and dependency invocations. There is no visible synchronization, bounds checking, rollback, or ownership bookkeeping.

Selectors and word locations may alias each other, external storage, or dependency-accessible memory. No non-aliasing guarantee is supplied. The two external-storage evaluations follow the initial indexed word write; if that write overlaps the storage, both evaluations occur after it.

Unknown dependencies may modify globals, selectors, and future word-read locations. They may allocate, release memory, or fail to return. Their effects cannot be assumed absent.

Direct byte destinations are low addresses, not established offsets within an implicit object. Word-address formulas likewise establish no allocation or object layout.

Ownership, lifetime, alignment, accessibility, and permission requirements beyond the displayed access widths remain **documented_only**.

## Dependencies

- **F-674e75a953e8** is the sole mapped direct dependency. It receives the central value twice. Its return convention, memory effects, ownership consequences, and termination behavior are unknown.
- **Unresolved indirect dependencies** have no supplied opaque function IDs. Their identities, equivalence across call sites, signatures, and implementations remain **documented_only**. No additional stable identities are invented here.

A call displaying no explicit arguments is not evidence that it consumes no machine state. Secondary-register results and the displayed 64-bit result require ABI validation before being treated as complete interfaces.

All later behavior is conditional on earlier dependencies returning and accesses succeeding.

## Acceptance examples

These examples are conditional static consequences, not executed tests.

1. **Entry boundary.** If the entry selector is 255, the first byte write targets address 272 and stores the incoming byte-like value. It precedes the first indirect call.

2. **Separate external evaluations.** Suppose the first external evaluation yields 1,024 and the second yields 3. The later masked-right-shift-by-10 argument is 1, while the early external-derived selector address is 13. No dependency call intervenes between the evaluations. This example preserves their separate dataflow; it does not establish a mechanism by which the storage changes.

3. **Equal external evaluations.** If both evaluations yield 3, the masked-right-shift argument is zero and the early selector address remains 13. Equality of values does not remove either displayed evaluation.

4. **Initial extraction.** If the initial word is `0xABCDEF12`, its right-shifted word-slot write contains `0x000ABCDE`, provided execution reaches a valid destination.

5. **Byte truncation.** A first byte-source word of `0x1234567F` produces `0xFC` in its fourfold byte write and `0x23` in its later masked-and-shifted byte write.

6. **Selector mutation.** If a selector yields 3 and an intervening dependency changes it to 9 before the specified reread, the corresponding destinations are addresses 20 and 26. Reusing the first selector would not preserve the represented sequence.

7. **Conditional wrapping.** Under 32-bit wrapping interpretation of the signed intermediate operations, shifting the scaled external capture left by 26 and multiplying by 1,024 yields zero. The resulting selector reads use address 1. This calculation depends on the second external evaluation, not the first.

8. **Split result.** With high half 7 and low half 2, the immediate zero word write targets address 48. The later high-half slot contains 7. The unsigned low-half-derived scaled write targets `0x20000014` and stores `0x08000000`.

9. **Store/read aliasing.** If the fourth scaled-index group receives zero, it writes zero at address 20 and then reads that location. In the absence of an independent intervening modification, the captured word is zero.

10. **Interrupted execution.** If an early access faults or a dependency does not return, later effects are not required to occur. No rollback is established.

## Uncertainties

The following remain **documented_only**:

- Correctness of the apparent entry point, function extent, and call boundaries.
- Reachability and machine meaning of the invalid-instruction termination.
- ABI meanings of hidden inputs, secondary results, and the 64-bit result.
- Identity, volatility, aliasing, and environmental modification of external storage.
- Whether its two displayed evaluations produce distinct hardware transactions or equal values.
- Signed overflow, signed left shifts, and conversions into signed intermediates.
- Address width, byte order, alignment, memory permissions, and fault behavior.
- Relative ordering of repeated reads within one call’s argument expressions.
- Indirect-target identities, complete signatures, and side effects.
- Normal completion, error reporting, ownership, and application intent.

No unknown callee is assumed to preserve globals or to own, allocate, or release memory.

## Evidence

This contract uses only the supplied static representation and the supplied mapping for F-674e75a953e8.

Established static evidence includes access widths, selector-directed destinations, displayed call order, two consecutive external-storage evaluations with distinct downstream uses, arithmetic expressions, the direct dependency call, the split 64-bit result, and the terminal bad-instruction marker.

The representation warns of invalid instruction data and overlapping global symbols. Those warnings limit interpretation of the body as a valid standalone routine.

Application-level intent is inferred neither from the arithmetic nor from repeated access patterns. Unresolved behavior is retained as **documented_only**.

No runtime tests, ABI validation, or independent instruction-level verification have been performed. The acceptance examples describe conditional arithmetic and sequencing expectations, not observed execution.
