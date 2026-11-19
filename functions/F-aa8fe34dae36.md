<!-- Trusted provenance: F-aa8fe34dae36; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aa8fe34dae36

## Purpose

F-aa8fe34dae36 performs an extended sequence of indirect dependency invocations, transforms their results, mutates memory through fixed and computed addresses, and ends with an indirect invocation carrying four explicit arguments.

**Established static evidence:** three 32-bit input slots, no explicit return value, ordered indirect invocations, arithmetic on apparent 32-bit and paired 64-bit results, and a concluding region of mixed-width memory accesses.

**Inferred intent:** some accesses resemble packed-field processing and record linking. No application-level operation, validated object model, allocation protocol, or synchronization protocol is established.

This contract describes the supplied static representation. It does not assert that the representation supplies a complete executable model.

## Interface

### Explicit inputs and completion

| Element | Width | Established behavior |
|---|---:|---|
| First input | 32 bits | No explicit consumption is shown. |
| Second input | 32 bits | No explicit consumption is shown. |
| Third input | Signed 32-bit representation | Its two-bit left shift supplies the second explicit argument of an early indirect invocation. |
| Result | No explicit value | A normal return follows the terminal invocation only if that invocation returns. |

Unused explicit input slots do not establish irrelevance under an unknown ABI. Dependencies may consume implicit machine state.

Execution additionally depends on incoming control state, dependency results and secondary result lanes, fixed-address state, and memory reached through computed addresses.

### Widths, layout, and arithmetic

Byte, halfword, and word accesses are respectively 8, 16, and 32 bits. Apparent paired results are 64 bits, with separately consumed low and high 32-bit portions.

Byte stores retain the low eight bits. Unsigned byte reads are zero-extended when propagated into word stores. Halfword stores retain sixteen bits.

Offsets describe access locations, not validated structure members. Endianness, record size, alignment, address-space layout, and permissible aliasing are not established.

Some arithmetic is represented as unsigned; other shifts, additions, and multiplications use signed 32-bit representations. This contract does not impose portable overflow or exceptional-shift semantics. Examples using modular arithmetic explicitly assume 32-bit bit-vector operations.

### Preconditions for complete execution

Every reached invocation target must be executable under the actual ABI. Every reached memory access must be valid at its width and alignment. Arithmetic and secondary result lanes must have usable target semantics.

No explicit checks enforce these conditions. Zero addresses and low fixed addresses are not rejected. No normal completion is promised when a dependency fails to return, an access faults, or arithmetic lacks defined execution semantics.

## Behavior

### Dependency-driven preparation

The preparation region contains a long ordered chain of indirect invocations. The first target depends on incoming control state plus 0xffe. Later targets are also indirectly constructed.

Visible argument relationships include:

- Primary results shifted left by eight.
- Low portions of paired results shifted left by eight, alongside unchanged or separately shifted high portions.
- Repeated copies of a transformed value in distinct argument positions.
- Explicit zero arguments.
- Retained results consumed after intervening invocations.
- The third input’s two-bit left shift.

Masks select low-order fields, including widths of 1, 8, 11–22, and 24 bits. Masking before shifting excludes higher bits before repositioning. Truncating left shifts do not become reversible merely because a later right shift occurs. Right shifts of a paired result by more than 32 positions select from its high portion.

No invocation may be assumed pure, memory-preserving, or ownership-neutral. Some consumed secondary lanes have unresolved producing interfaces. They remain unknown inputs, not implicit zeros.

### Descriptive address roles

The following names identify value relationships, not source identifiers or established objects:

| Role | Meaning |
|---|---|
| Packed base | A prepared low result shifted left by 25. |
| Scale origin | A later prepared primary result. |
| Scaled base | Scale origin multiplied by \(2^{25}\). |
| Prepared word | Another prepared primary result shifted left by eight. |
| Small field | Low fifteen bits of a prepared result shifted left by eight, then logically shifted right by six. |
| Small base | Twice the small field. |
| Cleared-field capture | Word reloaded from scaled base + 0x14 after the initial clear sequence. |
| Scaling operand | Word captured from a fixed-byte-derived base + 0x54. |
| Fourfold base | Four times the captured scaling operand. |
| Eightfold value | Scaling operand shifted left by three. |
| Adjusted precursor | Fourfold base minus 0x96. |
| Adjusted base | Four times the adjusted precursor. |
| Publication base | Word captured from separate fixed state before the store at 0x60. |

A captured value remains the value read at its capture point. It is not refreshed when its originating memory changes.

### Initial direct mutation and overlapping clears

Direct memory mutation follows the preparation invocations.

The opening writes, in order, publish the packed base as a word at the last paired result’s high portion + 0x14; write that high portion’s low byte at packed base + 9; publish the scaled base at the paired result’s low portion multiplied by 256 + 0x54; and overwrite packed base + 9 with the packed base’s low byte.

The clear sequence then:

- Clears the word at scaled base + 0x14.
- Reads the byte at scaled base + 5.
- Clears the word at small base + 0x14.
- Stores the zero-extended captured byte at scaled base + 0x34.
- Reloads scaled base + 0x14 into the cleared-field capture.
- **Clears small base + 0x14 a second time.**
- Performs two separate byte stores at small base + 0x15, both using the cleared-field capture’s low byte.

The last two byte stores overlap the second cleared word. Consequently, that small-base word need not remain zero.

The function next reads fixed address 0x05 as an unsigned byte and writes the small base’s low byte to fixed address 0x01. The captured fixed byte becomes a computed base; it is not a pointer-sized read.

### Linked/scaled mutations and fixed-address publication

The fixed-byte-derived base + 0x15 receives the small field’s low byte. Scaled base + 0x11 receives the prepared word’s low byte; the prepared word is then stored at cleared-field capture + 0x54; scaled base + 0x11 is subsequently zeroed.

The scaling operand is captured from fixed-byte-derived base + 0x54 before that field is overwritten with the cleared-field capture. Small base + 0x34 then receives the small field as a word.

A word read from scaled base + 0x74 becomes a linked destination. Its +0x54 field first receives scale origin shifted left by 26. A separate relationship reads small base + 0x74 twice: one read supplies a destination base, the other a stored word at that destination + 0x54. Their relative evaluation order within that expression is not established; their distinct read events must not be merged.

The scaling operand is written to scaling operand + 0x24. The linked destination’s +9 byte receives its low byte, and its +0x54 word receives the prepared word. The self-derived +0x24 store and linked +9 byte store then recur.

The next ordering is significant:

- Small base + 0x14 receives scale origin shifted left by 26.
- **Fixed address 0x54 receives a 32-bit store of the captured scaling operand shifted left by one.**
- Small base + 0x14 receives the captured scaling operand twice, as distinct word stores.
- Small base + 0x15 receives the fourfold base’s low byte twice.
- **Fixed address 0x15 receives an 8-bit store of the small base’s low byte.**
- Fourfold base + 0x54 receives the prepared word.
- Small base + 0x34 receives the scaling operand shifted left by two.

Thus the fixed 0x54 and 0x15 mutations occur between computed-address updates, not in a separate initialization phase. Either can affect subsequent accesses through aliasing.

Further mutations use a fresh word from small base + 0x74, doubled to form another destination, and a word reached through the byte at fourfold base + 5 plus offset 0x74. These destinations receive word writes at +0x44 and byte writes at +9, +0x0d, and +0x15. Self-derived word stores also occur at +0x14.

The fourfold base’s +0x64 and +4 fields supply additional addresses. A byte at the latter address +5 is read twice at separate use points. The second byte supplies the base of another +0x74 word read. Call this last captured word the **link operand**, and twice it the **doubled-link base**.

Before the constant-store region, the fourfold base’s +0x15 byte is overwritten with several distinct captured low bytes, its +0x54 word receives the scaling operand shifted left by two, its +9 byte receives the doubled-link base’s low byte, and its +0x14 word receives the eightfold value. Intermediate writes remain separate trace events.

### Constants and repeated publication

The link operand determines three constant destinations:

| Destination | Width | Value |
|---|---:|---:|
| Four times link operand | 8 bits | 0x3a |
| Eight times link operand | 16 bits | 0x003a |
| Four times link operand + 0x20 | 32 bits | 0x0000003a |

At adjusted base + 0x20, four ordered word stores carry 0x20, 0x2f, a zero-extended byte read from adjusted base +4, and 0x20. The byte read occurs between the second and third stores.

**The publication base is captured from separate fixed state after those four stores and before the word store of 0x20 at fixed address 0x60.** That captured base survives the 0x60 store and every subsequent computed-address write; no publication-base reload is shown.

The publication payload is four times the link operand. The initial publication group writes:

- That payload at eight times scaling operand + 0x54.
- The adjusted base’s low byte at publication base + 0x19.
- The publication base’s low byte at adjusted base + 0x11.
- Eight times scaling operand at publication base + 0x34.

Subsequent repeated groups preserve this four-access order but use eight times the adjusted precursor for the first destination base and final stored scaling value.

The repeated groups do not advance their destinations or reload the publication base. They cannot be replaced by one final-state update when access traces matter. The exact repetition count remains **documented_only** here.

### Final captures and fixed-address interleaving

After repeated publication, the function reads a word at fixed address 0xa9. Call this the **fixed-state capture**.

It next captures the word at publication payload + 0x74. Before the following two captures, it writes fixed address 0x3a as a byte, with value:

\[
\operatorname{low8}(8 \times \text{adjusted precursor} + 0x30).
\]

Only after that byte write does it capture the word at doubled-link base + 0x14, followed by the word at adjusted base +4. Call these the **restoration capture** and **adjusted-field capture**, respectively.

The 0x3a write must not move after these reads: its address can overlap memory supplying a later capture.

The following byte writes use the adjusted base’s low byte at doubled-link base +0x0d, constant 0x6c at adjusted base +8, and the publication base’s low byte at adjusted base +0x0d and +9. Doubled-link base +0x54 then receives the doubled-link base as a word.

Fixed address 0x67 receives the fixed-state capture’s low byte.

### Live 0x67 reads and the offset-0x40 dependency

A fresh word read at doubled-link base +0x44 supplies a signed shift operand and several stores. After that read, fixed address 0x67 is read for the byte store to fixed address 0x21.

The loaded +0x44 word supplies a byte at adjusted base +0x10 and a word at publication base +0x54. Its signed right shift supplies a byte at doubled-link base +1; the shift count is the adjusted base’s low byte. Publication base +0x54 is then overwritten with the adjusted base.

Next, **the word at adjusted base +0x40 is captured**. The function stores the earlier adjusted-field capture at publication base +0x74, then stores the newly captured +0x40 word at adjusted base +0x10.

This interleaving distinguishes capture from reload. If the publication-base +0x74 store aliases adjusted base +0x40, it does not change the already captured value written to adjusted base +0x10.

Publication base +0x44 then receives the doubled-link base.

Fixed address 0x67 is read again for a byte store at adjusted base +9. The function subsequently writes the adjusted base’s low byte at doubled-link base +9 and the fixed-state capture as a word at adjusted base +0x44. It reads adjusted base +0x54 to obtain a **late destination**, then reads fixed address 0x67 again for another byte store at adjusted base +9.

These are live reads, not reuse of the initially written byte. Intervening computed-address writes may overlap 0x67.

The supplied representation establishes three reads after initialization of 0x67: one for 0x21 and two for adjusted base +9. It does not establish a third computed-address consumer.

### Restoration, final linking, and terminal invocation

The late destination’s +0x0d byte receives the low byte of the earlier publication-payload +0x74 capture. Doubled-link base +0x54 then receives the restoration capture.

A fresh word at adjusted base +0x14 is captured before further byte writes. Doubled-link base +0x11 receives the earlier payload-field capture’s low byte; adjusted base +9 receives the late destination’s low byte; adjusted base +0x44 receives the late destination as a word.

A word read from doubled-link base +4 supplies another base. A halfword value of 0x0065 is stored at twice that base before its +0x64 word is read.

The late destination’s +0x11 byte receives the captured adjusted-base +0x14 word’s low byte, and its +0x54 word receives decimal 100. The function then captures the newer base’s +0x74 word and doubled-link base +0x64 word.

Finally, doubled-link base +9 receives the newer base’s low byte; the captured doubled-link +0x64 word supplies a destination whose +0x54 field receives the captured newer-base +0x64 word; and a word store at twice that latter captured word receives the captured doubled-link +0x64 word.

The terminal indirect invocation receives:

1. 0x72.
2. The captured newer-base +0x64 word.
3. The restoration capture.
4. The later captured newer-base +0x74 word.

Its target and effects are unresolved. Normal return is conditional on that invocation returning.

## Side effects

Memory effects are neither isolated nor transactional. Fixed state, computed destinations, and partially overlapping byte and word fields may alias. Earlier writes can therefore change later loaded values and addresses.

Duplicate writes remain distinct accesses. No barrier, atomicity, locking, rollback, retry, or reentrancy guarantee is established. Textual access order does not independently establish hardware visibility guarantees.

No explicit allocation, retention, release, or ownership transfer is established. Address-like values are not presumed owned or newly allocated. Unknown dependencies may mutate globals, allocate, release, or invalidate memory; no preservation or lifetime guarantee is granted across them.

No explicit error result, validation branch, exception translation, or recovery path is shown. Execution may stop before later effects.

## Dependencies

The supplied identity map establishes only F-aa8fe34dae36. No opaque IDs are provided for preparation or terminal targets.

**documented_only:** dependency identities, target resolution, calling conventions, implicit arguments, secondary result lanes, global clobbers, memory effects, ownership effects, and termination behavior.

Invocation sites are not assumed to identify either distinct functions or one shared function. No unsupported dependency identities are invented.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Input width.** Under 32-bit modular arithmetic, third input 3 contributes 12; 0x40000000 contributes zero after the two-bit left shift.

2. **Unsigned propagation.** A byte read of 0xfe at scaled base +5 supplies word value 0x000000fe at scaled base +0x34.

3. **Second clear and overlap.** If the cleared-field capture’s low byte is 0xab, both byte writes follow the second small-base +0x14 clear. With little-endian ordinary memory and no other overlapping changes, that word becomes 0x0000ab00, not zero.

4. **Fixed-address ordering.** An access trace must place the word store to 0x54 between the first scale-derived small-base +0x14 store and the following scaling-operand stores. The byte store to 0x15 follows both fourfold-byte writes and precedes the computed +0x54 store.

5. **Publication capture.** If a later computed write changes the separate fixed-state slot, publication still uses its earlier captured base. Reloading that slot after the 0x60 store is not equivalent.

6. **0x3a interleaving.** If the byte at 0x3a overlaps the word supplying the restoration capture, that capture observes the post-write memory, assuming ordinary stable memory.

7. **Live 0x67 consumption.** If an intervening doubled-link +9 write overlaps 0x67, the second adjusted-base +9 consumer may receive a different byte from the first. Propagating one saved initialization byte is insufficient.

8. **Offset-0x40 capture.** If adjusted base +0x40 yields 0x12345678, an intervening aliasing publication-base +0x74 store does not replace that captured value: adjusted base +0x10 receives 0x12345678.

9. **Completion.** A nonreturning terminal target establishes no normal completion. A returning target reaches the shown return without an explicit result value.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose, public parameter meanings, and actual ABI.
- Dependency semantics and implicit result provenance.
- Signed overflow, address overflow, negative right shifts, and excessive shift counts.
- Endianness, alignment, memory-map validity, object boundaries, and runtime feasibility.
- Exact repeated-publication count.
- Ownership and preservation across unknown dependencies.
- Terminal target identity and behavior.

The feedback mentions three computed-address consumers of 0x67. Only two are established by the supplied representation, plus the separate fixed-address 0x21 consumer. Any additional computed-address consumer remains **documented_only**, not an added access requirement.

No runtime tests have been performed. Successful execution, faults, final memory contents, and application-level outcomes have not been observed.

## Evidence

This contract uses static relationships in the supplied representation, independently of the prior draft’s claims.

**Established static evidence:** input widths; indirect invocations; masks and shifts; mixed-width accesses; both small-base word clears; fixed stores at 0x54 and 0x15; publication-base capture before the 0x60 store; the interleaved 0x3a write; separate 0x67 reads; the captured +0x40 dependency; restoration of captured values; and terminal explicit arguments.

**Inferred intent:** packed-field processing and record-like linking.

**Unknown:** dependency contracts, validated layouts, executable fidelity, and application-level success criteria. These unknowns are not resolved by assigning conventional meanings to constants, offsets, or address-like values.
