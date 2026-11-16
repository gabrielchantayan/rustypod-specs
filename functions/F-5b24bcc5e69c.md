<!-- Trusted provenance: F-5b24bcc5e69c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5b24bcc5e69c

## Purpose

F-5b24bcc5e69c performs an ordered series of indirect invocations, derives packed integer and address-like values from their results, applies mixed-width memory updates, and invokes an unresolved final target.

The supplied static representation establishes these operations, but not their application-level meaning. It does not establish a formatter, serializer, allocator, initializer, or container API. Constants resembling character values do not establish a text-processing contract.

This specification distinguishes:

- **Established static evidence:** represented invocations, arithmetic, accesses, and their ordering.
- **[INFERENCE]:** interpretations or arithmetic conclusions requiring an explicitly stated assumption.
- **documented_only:** unresolved behavior for which no acceptance guarantee is established.

No runtime tests have been performed.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters and no explicit return value. Its implicit inputs include:

- Incoming execution-link state used to form the first invocation target.
- Ordinary and secondary results from indirect invocations.
- Mutable memory accessed through dependency results and transformed values.
- Fixed-address byte and word storage.
- A dedicated stored-base word sampled late in execution.
- An unresolved final dispatch target.

There is no established status result, returned object, output-buffer parameter, or completion flag. Observable outputs are dependency invocations, memory accesses and updates, and the final four-argument invocation.

A return is represented after the final invocation. Normal completion requires every reached dependency and access to complete, including the final invocation returning normally.

### Widths and representation

| Quantity or access | Represented width |
|---|---:|
| Byte | 8 bits |
| Halfword | 16 bits |
| Word, general integer, address-like value | 32 bits |
| Paired invocation result | 64 bits |

Byte and halfword stores retain the low eight and sixteen bits respectively. Byte reads converted to unsigned words are zero-extended.

The low and high halves of a paired result are numeric 32-bit portions. Their extraction does not establish memory endianness. Addresses and offsets below describe the represented arithmetic; they do not establish valid objects or allocation boundaries.

### Preconditions and failure boundary

The visible body does not validate targets, address accessibility, alignment, object extent, or shift counts. Uninterrupted execution therefore requires the environment to support every reached invocation and access.

Some addresses are explicitly small fixed values; others can become small through arithmetic or zero-valued bases. Ordinary protection against low-address access is not shown to be compatible with this behavior.

No checked-error return, rollback, recovery handler, or ownership protocol is established. Fault handling, nonlocal exits, synchronization, and lifetime requirements are **documented_only**.

## Behavior

### Ordered invocation phase

Execution begins with a long straight-line sequence of indirect invocations. The first target is formed from incoming execution-link state plus 0xFFE. Later targets are predominantly formed from position-associated constants or references plus the same displacement.

No conditional branch or loop is visible in this phase. Reaching a later invocation requires earlier invocations to return.

Represented argument patterns include:

- No explicit arguments.
- Explicit zeros.
- A 32-bit result shifted left by eight bits.
- Separate low and high halves of a paired result.
- Repeated arguments containing the same captured value.
- Values retained across intervening invocations.
- Secondary result state whose calling convention is unresolved.

An invocation with no represented arguments may still depend on implicit state. An invocation whose result is unused locally may still have effects. Neither category may be discarded on the assumption that it is pure.

The complete externally meaningful invocation trace and target identities are **documented_only** here. The visible sequence does not establish permission to replace these invocations with one service call.

### Packed arithmetic

The invocation phase contains field extraction, positioning, truncation, and reuse. Established patterns include:

- Selecting bits 10–19 from an unsigned word and positioning that ten-bit field by a further five- or six-bit left shift.
- Retaining the low 24 bits before shifting left by two, four, five, or six bits.
- Retaining the low 22 bits before shifting left by three, seven, or eight bits.
- Unsigned right shifts of a paired result’s high word.
- Unsigned 64-bit right shifts by 34, 36, and 44 bits, followed by conversion to a word. These leave at most 30, 28, and 20 significant bits respectively.
- Masks retaining fields of 19, 17, 14, 13, 12, or 11 bits at particular stages.

A 24-bit field shifted left by six occupies at most 30 bits. A 22-bit field shifted left by eight also occupies at most 30 bits. Unmasked word shifts can lose high bits under a 32-bit bit-vector interpretation.

Several captured values are explicitly zeroed and later shifted. Their represented dataflow remains zero until an explicit replacement. An intervening unknown invocation is not itself evidence that a captured local value changes.

**Arithmetic qualification:** many shifts and multiplications are represented on signed 32-bit quantities. Wrapping arithmetic, valid-input restrictions, and the accuracy of the represented signedness are not independently established. Conclusions requiring modulo-\(2^{32}\) arithmetic are conditional, not universal guarantees. Signed right shifts and counts of at least 32 remain unresolved.

### Entry to direct memory effects

The following descriptive roles identify captured values, not source identifiers:

- **Terminal pair:** the last paired result before direct accesses.
- **Tag base:** an earlier low-word result shifted left by 25 bits.
- **Packed base:** a separately captured word multiplied by \(2^{25}\).
- **Selector:** a result shifted left by eight bits, masked to 15 bits, then shifted right by six.
- **Doubled selector:** twice the selector.

[INFERENCE] Under 32-bit bit-vector arithmetic, the selector is a multiple of four between 0 and 508 inclusive. The doubled selector is a multiple of eight between 0 and 1016 inclusive.

The initial direct-access ordering is significant. A word containing the tag base is first stored at terminal-high-half offset 0x14. The low byte of that high half is then stored at tag-base offset 9. Next, the packed base is stored at offset 0x54 from the terminal low half scaled by 256. Only afterward is tag-base offset 9 overwritten with the tag base’s own low byte.

The body then clears the word at packed-base offset 0x14 and reads the byte at packed-base offset 5. It clears the word at doubled-selector offset 0x14 before storing the captured byte, zero-extended, at packed-base offset 0x34.

Packed-base offset 0x14 is subsequently reread. Another zero-word store to doubled-selector offset 0x14 follows that read. Two successive byte stores at doubled-selector offset 0x15 use the low byte of the captured reread word.

These requirements describe access ordering, not merely a final memory image. In particular, the reread must not be replaced by the earlier literal zero without an established nonaliasing argument.

### Interlinked updates and byte-derived addresses

The next region combines scaled values, loaded words, and byte-derived addresses.

A byte read from fixed address 0x05 supplies a zero-extended base. After that read, a byte is written to fixed address 0x01. Subsequent updates include a byte at the byte-derived base’s offset 0x15 and mixed-width accesses through the packed base, doubled selector, and previously loaded words.

A word at byte-derived-base offset 0x54 is captured before that location is replaced. Call this captured value the **root word**. Its later uses include self-referential stores at offset 0x24 and address derivations scaled by two, four, and eight. These uses retain the captured root word rather than implicitly rereading its original location.

Other represented effects include:

- Loads through offset 0x74 followed by stores through loaded values.
- Repeated word stores and repeated byte stores.
- Byte stores at offsets 9, 0x0D, 0x11, and 0x15.
- Word stores at offsets 0x14, 0x34, 0x44, and 0x54.
- A word store to fixed address 0x54 and a byte store to fixed address 0x15.
- Self-referential word stores at offsets 0x14 and 0x24.

Later, a byte sampled at an address-like value plus 5 becomes the base for a word read at offset 0x74. This is a byte-to-address dependency, not an established table lookup.

A byte source at offset 5 is sampled more than once in this region, with writes between samples. The samples are not interchangeable merely because their address expressions match. The detailed access trace outside the explicitly described ordering constraints remains **documented_only**.

### Constant writes and repeated updates

For this region:

- **Fourfold root** means four times the captured root word.
- **Adjusted precursor** means the fourfold root minus 0x96.
- **Adjusted base** means four times the adjusted precursor.
- **Constant seed** means the late word loaded through a byte-derived base at offset 0x74.
- **Final anchor** means twice the constant seed.
- **Constant base** means four times the constant seed.
- **Retained base** means the dedicated stored-base word sampled before the repeated updates.

These scalings have the arithmetic qualifications stated above.

The constant-writing region stores 0x3A as a byte at the constant base, as a halfword at eight times the constant seed, and as a word at constant-base offset 0x20. These are distinct access widths and destinations.

At adjusted-base offset 0x20, the represented word-store sequence contains 0x20, then 0x2F, then a zero-extended byte sampled from adjusted-base offset 4, then 0x20 again. The byte sample occurs after the 0x2F store and before its corresponding word store. Aliasing may therefore affect that sample.

**The retained base is read from its dedicated storage before the word store of 0x20 to fixed address 0x60.** The captured retained base is subsequently used for the repeated region. Reversing those accesses is not an established equivalent, including where mapped storage or other aliasing makes their order observable.

The repeated region uses ordered groups of four stores:

- A word containing the constant base at offset 0x54 of an eight-times-scaled base.
- The adjusted base’s low byte at retained-base offset 0x19.
- The retained base’s low byte at adjusted-base offset 0x11.
- The eight-times-scaled base as a word at retained-base offset 0x34.

The initial group uses eight times the root word. Subsequent groups use eight times the adjusted precursor. The bases and byte values are captured values, not fresh samples of the dedicated stored-base word.

The region is represented as repeated straight-line stores, not an allocation, retry, append, or traversal loop. Exact repetition cardinality is **documented_only** in this specification; collapsing the repeated groups to one group is not established to preserve behavior.

### Finalization: ordering-sensitive reads

The opening order of finalization is:

1. A word read from fixed address 0xA9.
2. A word read from constant-base offset 0x74.
3. A byte write to fixed address 0x3A.
4. A word read from final-anchor offset 0x14.
5. A word read from adjusted-base offset 4.

The byte written at 0x3A is the truncated result of scaling the adjusted precursor’s low-byte representation by eight and adding 0x30.

The later two word reads occur **after** this byte write. They are not snapshots taken alongside the first two reads. If either read overlaps the written byte, its result depends on that intervening write.

The word captured from final-anchor offset 0x14 is retained for subsequent stores and for the third final-dispatch argument. The earlier word captured from constant-base offset 0x74 supplies truncated bytes later in finalization; it is not the source of the fourth final-dispatch argument.

### Finalization: fixed-byte rereads and mixed-width updates

Following the opening reads, the body writes truncated adjusted-base and retained-base values through derived byte locations. It also writes 0x6C at adjusted-base offset 8 and stores the final anchor at its own offset 0x54.

The low byte of the captured 0xA9 word is then stored at fixed byte address 0x67. A word is read from final-anchor offset 0x44 before **a fresh byte read of 0x67** supplies the write to fixed byte address 0x21.

That captured offset-0x44 word supplies further stores, including a byte at adjusted-base offset 0x10 and a word at retained-base offset 0x54. A byte written at final-anchor offset 1 is derived from a signed right shift of that word; the shift count is the adjusted base’s low eight bits. The behavior for negative values or excessive counts is **documented_only**.

A later word read at adjusted-base offset 0x40 precedes a word write at adjusted-base offset 0x10. Thus the earlier byte write at offset 0x10 is followed by an overlapping word write. The word captured earlier from adjusted-base offset 4 is also stored at retained-base offset 0x74.

There are **two additional, distinct reads of fixed byte 0x67**, each supplying a later byte store at adjusted-base offset 9:

- The first follows the offset-0x40 read and related word updates, including the store at retained-base offset 0x44.
- Before the second, the body writes a byte at final-anchor offset 9, stores the captured 0xA9 word at adjusted-base offset 0x44, and reads a word at adjusted-base offset 0x54.

Consequently, the three reads of 0x67—one for 0x21 and two for adjusted-base offset 9—must remain distinct. None is established to return the originally stored byte. Intervening derived-memory writes may alias 0x67.

The word loaded from adjusted-base offset 0x54 becomes another destination base. Later effects include byte stores derived from the earlier constant-base offset-0x74 word, replacement of final-anchor offset 0x54 with its earlier captured offset-0x14 word, and another word read at adjusted-base offset 0x14.

### Final dispatch boundary

Near completion, a word loaded from final-anchor offset 4 supplies a base whose doubled address receives the halfword 0x0065. A word loaded from that base’s offset 0x64 becomes the second final-dispatch argument. A separate word read from the same base’s offset 0x74 becomes the fourth argument.

Further effects include:

- A byte store using the adjusted-base offset-0x14 sample.
- A word store of decimal 100 at offset 0x54 of the destination base obtained earlier.
- A word read from final-anchor offset 0x64.
- A byte store at final-anchor offset 9.
- A word store through the latest offset-0x64 result.
- A word store through twice the captured second-argument value.

The final indirect invocation receives, in order:

| Position | Represented argument |
|---|---|
| First | 0x72 |
| Second | The late word captured from offset 0x64 of the base loaded through final-anchor offset 4 |
| Third | The earlier word captured from final-anchor offset 0x14 |
| Fourth | The late word captured from offset 0x74 of that same loaded base |

The target, meanings, and actual return behavior are **documented_only**.

## Side effects

The direct-access layout includes the following offsets, without establishing a common structure:

| Width | Observed derived-base offsets |
|---|---|
| Byte | 1, 5, 8, 9, 0x0D, 0x10, 0x11, 0x15, 0x19 |
| Word | 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, 0x74 |

Halfword stores also occur at scaled addresses. Fixed-address storage includes bytes at 0x01, 0x05, 0x15, 0x21, 0x3A, and 0x67, and words at 0x54, 0x60, and 0xA9.

These views may overlap derived accesses. No disjointness, volatility, device mapping, atomicity, or concurrency model is established.

Earlier effects may remain if a later operation fails. Repeated and overwritten stores belong to the represented access sequence even when ordinary nonaliasing memory would retain only the final value.

No allocation, release, ownership transfer, or lifetime guarantee is established. Unknown dependencies must not be assumed to preserve globals, avoid memory mutation, or retain or release memory.

## Dependencies

F-5b24bcc5e69c is the only supplied opaque function identity. No dependency-specific opaque IDs are available; none are invented.

| Boundary | Established role | Unresolved properties |
|---|---|---|
| Entry-derived indirect target | Begins invocation sequence | Identity, signature, effects |
| Position-derived indirect targets | Produce results and consume transformed arguments | Identities, equivalence, state changes |
| Secondary-result boundaries | Supply additional argument state | Calling convention and preservation |
| Final indirect target | Consumes four represented arguments | Identity, semantics, return behavior |

Absent dependency contracts, there is no established application-level end-to-end guarantee.

## Acceptance examples

These examples are conditional static requirements, not executed tests.

1. **Selector restriction.** Under 32-bit bit-vector arithmetic, selector values 1, 2, and 3 are impossible. The maximum is 508, not 511.

2. **Truncation and overwrite.** If the terminal high word ends in 0xAB, the first tag-base offset-9 store contains 0xAB. Its later overwrite uses the captured tag base’s low byte.

3. **Zero-extension.** A sampled byte of 0xFE stored as an unsigned word produces 0x000000FE, not 0xFFFFFFFE.

4. **Stored-base capture order.** If an access observer records the dedicated stored-base read and the fixed-address 0x60 word store, the read precedes the store. Reading the retained base only afterward does not preserve the specified order.

5. **Intervening finalization write.** If final-anchor offset 0x14 overlaps 0x3A, its opening finalization read occurs after the byte write at 0x3A. A pre-write snapshot is not equivalent.

6. **Three independent byte samples.** Suppose the initial 0x67 store contains 0xAB, but an intervening aliasing write changes 0x67 to 0xCD before a later sample. That later destination receives 0xCD. Reusing 0xAB for all three destinations is not justified.

7. **Halfword width.** The 0x65 store writes one 16-bit value, 0x0065. It is not two byte stores of 0x65; byte order remains unresolved.

8. **Captured dispatch argument.** An intervening write to final-anchor offset 0x14 does not replace the already captured third dispatch argument. Its earlier sample remains the argument source.

9. **Failure without rollback.** An inaccessible reached destination has no established conversion into a status return. Earlier completed writes need not be undone.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and supported caller contexts.
- Accuracy of represented invocation boundaries, signatures, and signedness.
- Complete dependency trace and indirect-target identities.
- Secondary-result conventions.
- Signed overflow, address wraparound, and exceptional arithmetic behavior.
- Signed right-shift semantics and excessive shift counts.
- Endianness, alignment, object extents, and address accessibility.
- Memory observability, concurrency, synchronization, and reentrancy.
- Ownership, allocation, release, and lifetime.
- Exact repeated-region cardinality.
- Final target semantics and normal-return guarantees.
- Faults, exceptions, and nonlocal control transfer.

No structural or text-related interpretation is required for acceptance.

## Evidence

This contract uses only the supplied static representation of F-5b24bcc5e69c. Prior prose and reviewer feedback are not independent behavioral evidence.

The representation supports the widths, selected arithmetic relationships, captured-value distinctions, ordering constraints, fixed-address effects, and final argument arrangement described above. In particular, it places the retained-base read before the 0x60 store, interposes the 0x3A byte write between finalization word reads, and represents three separate final-region reads of byte location 0x67.

No dependency inspection, implementation access, runtime execution, smoke run, or test evidence was available. Arithmetic interpretations are qualified, and unresolved behavior remains documented rather than assigned guessed semantics.
