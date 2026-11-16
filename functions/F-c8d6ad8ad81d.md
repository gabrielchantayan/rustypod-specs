<!-- Trusted provenance: F-c8d6ad8ad81d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c8d6ad8ad81d

## Purpose

F-c8d6ad8ad81d performs an ordered sequence of unresolved indirect invocations, transforms returned and secondary-result words, modifies computed and fixed-location memory, and invokes an unresolved terminal target with four argument words.

The supplied static representation does not establish an application-level purpose. Character-like constants do not establish text output; recurring offsets do not establish a record type, allocation layout, or ownership relationship.

This contract distinguishes:

- **Established static evidence:** represented inputs, widths, value dependencies, access forms, and ordering.
- **Inferred intent:** possible interpretations without acceptance force.
- **documented_only:** unresolved semantics or coverage limits explicitly identified below.

Behavioral requirements are conditional on execution reaching the relevant operation and on the supplied representation accurately describing execution. No runtime execution or tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | Supplied unchanged to the first indirect invocation. |
| Second argument | 32 bits | No explicit use is visible. |
| Third argument | Signed 32 bits | Shifted left by six positions for an early invocation. |

The apparent nonuse of the second argument does not establish that every associated incoming register or machine-state value is irrelevant. Complete dependency signatures and implicit inputs remain unresolved.

### Implicit inputs

Execution depends on incoming link-related state, indirect executable targets, primary and secondary dependency results, computed-address memory, and fixed-location storage.

The first invocation target is derived from incoming link-related state with displacement 0xffe. Later invocation targets generally use code-associated locations with that displacement. Their derivation does not establish their semantics.

### Outputs and completion

No explicit return value is represented. Observable outputs include dependency invocations, direct memory effects, and the terminal invocation.

The terminal invocation receives:

| Argument position | Required value |
|---|---|
| First | The 32-bit constant 0x72. |
| Second | A saved 32-bit word read at offset 0x64 from the late-loaded base, immediately after the halfword store of 0x65. |
| Third | A saved 32-bit word captured earlier at offset 0x14 from the doubled linked base. |
| Fourth | A separate 32-bit word read at offset 0x74 from the late-loaded base, after the intervening computed byte store and decimal-100 word store. |

The second argument is captured **before** those intervening stores, not after them. The fourth argument is captured afterward. Neither is a fresh observation at terminal-invocation time.

A return follows the terminal invocation in the representation. Whether the target returns normally is **documented_only**.

### Preconditions

No explicit validation is visible. Successful continuation requires executable targets and accessible storage for every reached access.

No null policy, length, capacity, alignment guarantee, ownership protocol, or safe address range is established. Arbitrary three-word inputs are not established as safe.

## Behavior

### Invocation phase and arithmetic

The initial phase is a straight-line sequence of indirect invocations interleaved with integer transformations. No explicit conditional branch or loop is visible.

Invocations have zero to four explicit argument words in the representation. Arguments include earlier results, secondary-result words, shifted and masked values, and explicit zeros. An invocation with no explicit arguments is not thereby established as parameterless. A discarded result does not imply an effect-free invocation.

Dependency ordering is observable. Unknown dependencies may alter memory, globals, resources, or control flow. A dependency that does not return prevents subsequent operations from being reached.

Most intermediate words and computed addresses are represented at 32-bit width. Some results are represented as 64-bit quantities, with lower and upper 32-bit words used independently. Material transformations include:

- Left shifts with counts through 31.
- Logical right shifts of unsigned words.
- Masks selecting fields from one bit through 24 bits.
- Multiplication by powers of two.
- Subtraction of 0x96 before further scaling.
- Unsigned 64-bit right shifts by 34, 36, and 44.

A 64-bit shift by 34, 36, or 44 is not a 32-bit shift with a reduced count. Lower-word extraction discards the upper word. Extracting the upper word before shifting is a different operation from shifting the original full-width result.

Unsigned 32-bit operations retain the low 32 result bits. Byte stores retain eight bits; halfword stores retain sixteen. Unsigned byte reads are zero-extended when used for word stores.

Machine-level interpretation of signed overflow, signed left shifts, out-of-range signed conversions, and exceptional shift counts is **documented_only**. Address arithmetic must not be interpreted as unbounded integer arithmetic. Simplifications dependent on truncation require that assumption explicitly.

### Descriptive memory relationships

The following labels describe value relationships, not recovered types:

| Label | Meaning |
|---|---|
| High-position base | An earlier result multiplied by 2²⁵. |
| Shifted base | Another result shifted left by 25 positions. |
| Small base | Twice the unsigned value obtained by shifting a result left eight positions, retaining its low fifteen bits, then shifting right six positions. |
| Sampled low-address byte | The unsigned byte captured from fixed address 0x05. |
| Loaded word | The word read from sampled low-address byte plus 0x54 before that location is replaced. |
| Fourfold base | Four times the loaded word. |
| Adjusted value | The fourfold base minus 0x96. |
| Adjusted base | Four times the adjusted value. |
| Late global base | The global word captured immediately before the fixed-address 0x60 store. |
| Selected linked word | A later word reached using an unsigned offset-5 byte as a numeric base, followed by offset 0x74. |
| Doubled linked base | Twice the selected linked word. |
| Repeated payload | Four times the selected linked word. |
| Late-loaded base | The word read near completion at doubled linked base plus 4. |

These values are not established as valid pointers, disjoint storage, allocated objects, or stable memory regions.

### Opening direct effects

The last initial-phase dependency supplies lower and upper result words. The first direct memory effects establish the following observable ordering:

- A word store at upper result word plus 0x14 receives the shifted base.
- The byte at shifted base plus 9 receives the upper result word’s low byte.
- A word at 256 times the lower result word plus 0x54 receives the high-position base.
- The byte at shifted base plus 9 is replaced with the shifted base’s low byte.
- The high-position base’s offset-0x14 word is cleared, followed by an unsigned byte read at its offset 5.
- The small base’s offset-0x14 word is cleared.
- The sampled byte is stored, zero-extended, at high-position base plus 0x34.
- The high-position base’s offset-0x14 word is reloaded.

The reload is not replaceable with an assumed zero: intervening computed stores can overlap that location.

A further small-base offset-0x14 clear and two identical byte stores at small base plus 0x15 precede the fixed-address 0x05 observation.

### Access widths and layout

Offsets are byte offsets from their applicable bases. They do not describe one coherent record.

| Offset | Represented accesses |
|---|---|
| 0x01 | Byte store, including the variable-shift result. |
| 0x04 | Unsigned byte reads and separate word reads. |
| 0x05 | Unsigned byte reads, sometimes supplying numeric address bases. |
| 0x08 | Byte store of 0x6c. |
| 0x09 | Repeated byte stores. |
| 0x0d | Byte stores through linked and adjusted bases. |
| 0x10 | Byte and word stores. |
| 0x11 | Byte stores, including zero. |
| 0x14 | Word reads, clears, and replacements. |
| 0x15 | Repeated byte stores. |
| 0x19 | Repeated byte stores through the late global base. |
| 0x20 | Word stores of constants and a zero-extended byte. |
| 0x24 | Word stores containing the destination base’s numeric value. |
| 0x34 | Word stores of byte-derived or scaled values. |
| 0x40 | Word read supplying a later offset-0x10 store. |
| 0x44 | Linked word reads and writes. |
| 0x54 | Linked word reads and replacements. |
| 0x64 | Word reads supplying bases or terminal arguments. |
| 0x74 | Word reads and writes, including nested addressing. |

The offset-4 observation in the constant-write region is eight bits. A later offset-4 observation is a full word. They are not interchangeable.

Using an unsigned byte directly as a base for an offset-0x74 word read does not establish a pointer table or element-size multiplication.

Separate field reads remain separate observations. One linked update reads an offset-0x74 field independently for destination formation and stored value. Equality of those observations, and their exact evaluation order within the combined expression, are **documented_only**.

### Fixed-location ordering and linked propagation

Fixed-location accesses are interleaved with computed accesses.

**Address 0x05 before address 0x01.** After the opening duplicate small-base offset-0x15 stores, the unsigned byte at fixed address 0x05 is captured. Fixed address 0x01 then receives the small base’s low byte.

Subsequent addressing uses the captured sample, not a replacement observation. Computed effects before the next fixed write include:

- A byte store through the sample-derived address.
- Byte replacements at high-position base plus 0x11.
- A word store through the earlier offset-0x14 reload.
- Capture of the loaded word from sampled low-address byte plus 0x54, followed by replacement of that field.
- Small-base and linked updates through offsets 0x34, 0x54, and 0x74, plus self-value stores at offset 0x24.

**Address 0x54 before address 0x15.** A small-base offset-0x14 store precedes the fixed-address 0x54 word store, which receives the loaded word shifted left one position. Following that fixed store, the small-base offset-0x14 word receives the loaded word twice. Its offset-0x15 byte then receives the fourfold base’s low byte twice. Only afterward does fixed address 0x15 receive the small base’s low byte.

Further linked propagation uses offsets 0x14, 0x44, 0x54, 0x64, and 0x74. It includes repeated replacements, self-value stores, and byte stores containing truncated numeric bases. Captures made before replacement remain saved values; they are not later reloads.

Intervening computed writes may overlap fixed storage. The fixed accesses cannot be regrouped into an independent initialization pass.

### Constants and repeated stores

One region stores 0x3a using three different widths and destinations:

- A byte at four times the selected linked word.
- A halfword at eight times that word.
- A word at the fourfold destination plus 0x20.

The adjusted base’s offset-0x20 word subsequently receives, in order, 0x20, 0x2f, the zero-extended byte read at adjusted base plus 4, and 0x20 again. This establishes word replacements, not a character-output sequence.

The late global base is captured after those replacements. The word store of 0x20 at fixed address 0x60 immediately follows that capture, with no intervening computed-memory access represented.

The repeated-store region has a distinct initial group:

| Ordered effect | Initial group |
|---|---|
| Word at scaled destination plus 0x54 | Repeated payload, through eight times the loaded word. |
| Byte at late global base plus 0x19 | Adjusted base’s low byte. |
| Byte at adjusted base plus 0x11 | Late global base’s low byte. |
| Word at late global base plus 0x34 | Eight times the loaded word. |

Subsequent groups retain that four-store order but use eight times the **adjusted value** for the scaled destination and offset-0x34 value.

The representation contains explicit repetitions, not a loop condition. This contract does not enumerate their exact cardinality; that coverage limit is **documented_only**. Replacing the region with only its final memory image would omit represented writes.

### Final captures and independent fixed-address samples

After the repeated region, execution captures the word at fixed address 0xa9, then reads a word at repeated payload plus 0x74. Fixed address 0x3a subsequently receives the low-byte result of eight times the adjusted value plus 0x30.

The saved 0xa9 word is retained independently of later memory changes.

Execution then captures:

- The doubled linked base’s offset-0x14 word, retained as the third terminal argument.
- The adjusted base’s offset-4 word, retained for a linked offset-0x74 store.

Computed byte writes at offsets 0x0d, 8, and 9 and a doubled-linked-base offset-0x54 replacement precede the byte store of the saved 0xa9 word’s low byte to fixed address 0x67.

Three subsequent reads of fixed address 0x67 are distinct:

| Observation | Ordering and destination |
|---|---|
| First | After the fixed 0x67 store and a word read at doubled linked base plus 0x44; supplies fixed address 0x21. |
| Second | After computed updates, the variable-shift store, the adjusted offset-0x40 read, and linked stores through offsets 0x74, 0x10, and 0x44; supplies adjusted base plus 9. |
| Third | After a doubled-linked-base offset-9 byte store, the saved 0xa9 word’s store at adjusted base plus 0x44, and a word read at adjusted base plus 0x54; supplies another store at adjusted base plus 9. |

These reads need not yield equal bytes. Computed stores may overlap fixed address 0x67.

The variable-shift byte store uses a signed word shifted right by a count taken from the adjusted base’s low byte. The count can exceed 31; effective machine semantics are **documented_only**.

### Terminal capture boundary

The word read at adjusted base plus 0x54 supplies a linked successor base. Subsequent updates include a byte store through that successor, restoration of the saved third-argument word at doubled linked base plus 0x54, capture of adjusted base plus 0x14, and additional byte and word replacements.

The late-loaded base is then captured from doubled linked base plus 4. The terminal boundary preserves this order:

| Boundary event | Required observation or effect |
|---|---|
| Halfword store | Sixteen-bit 0x65 is stored through twice the late-loaded base. |
| Second-argument capture | The word at late-loaded base plus 0x64 is immediately captured and retained. |
| Intervening byte store | The linked successor’s offset-0x11 byte receives the low byte of the previously captured adjusted offset-0x14 word. |
| Intervening word store | The linked successor’s offset-0x54 word receives decimal 100. |
| Fourth-argument capture | The word at late-loaded base plus 0x74 is captured. |

The second-argument read is therefore **before both intervening stores**. Their computed destinations may overlap the offset-0x64 location. A later reload can produce a different terminal argument and is not equivalent.

After the fourth-argument capture, a separate word read at doubled linked base plus 0x64 supplies a routing value. A byte at doubled linked base plus 9 receives the late-loaded base’s low byte. The routing value’s offset-0x54 word receives the saved second argument. Finally, a word through twice that saved second argument receives the routing value.

The terminal invocation follows with the four retained arguments specified in the interface. No rollback is represented.

## Side effects

Direct effects comprise byte, halfword, and word accesses through computed addresses, a global-word capture, and fixed-location accesses at 0x01, 0x05, 0x15, 0x21, 0x3a, 0x54, 0x60, 0x67, and 0xa9.

Their accessibility, mapping, hardware meaning, and memory attributes are **documented_only**. Low addresses must not be silently interpreted as harmless constants or independent named variables.

Overlapping accesses are possible. Different bases or offsets do not prove disjoint storage. Endianness, alignment constraints, atomicity, thread safety, and reentrancy are not established.

No direct allocation or release primitive is identified. Dependencies may allocate, release, retain, or modify resources; none of those effects is established or excluded. No ownership transfer or lifetime guarantee is established for inputs or loaded words.

No explicit error result, diagnostic, retry, recovery branch, or rollback is visible. Invalid targets, inaccessible addresses, or exceptional arithmetic may prevent continuation. Exact failure behavior is **documented_only**. Earlier effects are not shown being undone.

## Dependencies

F-c8d6ad8ad81d is the only supplied opaque function identity. No distinct opaque IDs are established for its unresolved indirect dependencies; their identities remain **documented_only**.

The unresolved dependency set includes the entry target, later code-location-derived targets, targets exposing secondary-result words, and the terminal target.

No dependency is assumed to preserve globals or memory, return normally, own memory, or release memory. Matching argument shapes do not establish matching target identities. No dependency is assigned a guessed application-level role.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Entry propagation.** If the first invocation is reached, its explicit argument is the first input’s 32-bit pattern.

2. **Unambiguous early shift.** A third input of 1 contributes 64 to the relevant early invocation without requiring signed-overflow assumptions.

3. **Byte observation.** If adjusted base plus 4 yields byte 0xfe in the constant region, the associated word replacement receives 254, not a word assembled from adjacent bytes.

4. **Ordered replacements.** An observer of adjusted base plus 0x20 sees 0x20, 0x2f, the sampled byte value, and 0x20, subject to reaching each store.

5. **Fixed-address capture.** If address 0x05 yields 0x80, later sample-derived addressing uses 128 despite the following write at fixed address 0x01.

6. **Interleaving.** Fixed writes at 0x54 and 0x15 retain the intervening computed word replacements and duplicate byte stores. Making those fixed writes consecutive is not equivalent.

7. **Global capture.** If the 0x60 store overlaps the global supplying the late global base, subsequent direct accesses still use the value captured before that store.

8. **Independent samples.** If the three late reads at 0x67 yield 0x11, 0x22, and 0x33, their corresponding destinations receive those respective bytes.

9. **Saved word.** If 0xa9 yields 0x12345678, the later saved-word store uses that pattern despite subsequent changes to overlapping memory.

10. **Halfword width.** The 0x65 store writes sixteen bits. Replacing it with an eight-bit store can leave different adjacent storage.

11. **Early second-argument capture.** Suppose the offset-0x64 read immediately after the halfword store yields 0x2000. If either subsequent computed store changes that location to 0x3000, the terminal second argument remains 0x2000. The final doubled-second-argument destination is derived from 0x2000.

12. **Later fourth-argument capture.** If either intervening store changes late-loaded base plus 0x74, the fourth-argument read observes the post-store location. Capturing it alongside the earlier second argument is not equivalent.

13. **Saved third argument.** Later replacement of doubled linked base plus 0x14 does not change the third terminal argument already captured there.

14. **Partial execution.** A nonreturning dependency prevents later operations from being reached. No cleanup obligation is established.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and output interpretation.
- Dependency identities, complete signatures, implicit inputs, and secondary-result conventions.
- Valid mappings, alignment, endianness, and memory attributes.
- Aliasing, object boundaries, external mutation, and resource lifetimes.
- Signed overflow, conversions, and exceptional shift semantics.
- Evaluation order within combined memory expressions.
- Exact repeated-store cardinality, not enumerated by this contract.
- Dependency ownership and resource effects.
- Fault behavior and terminal-target return behavior.

These uncertainties do not make the terminal capture boundary unresolved: the represented offset-0x64 read precedes the computed byte and decimal-100 stores, while the offset-0x74 read follows them.

Structured-state construction and character-oriented output remain inferred intent only.

## Evidence

The supplied static representation supports the interface widths, indirect invocation dependencies, integer transformations, direct access widths, fixed-location observations, and terminal argument relationships described above.

The terminal boundary is supported by the represented progression from the 0x65 halfword store to the offset-0x64 word capture, then the successor offset-0x11 byte and offset-0x54 decimal-100 stores, then the offset-0x74 word capture. The earlier second-argument capture remains a retained value through the remaining stores and terminal invocation.

The evidence also distinguishes the saved 0xa9 word from three later byte observations at 0x67, and the constant-region offset-4 byte read from later offset-4 word reads.

The prior document and reviewer feedback are not independent behavioral evidence. No successful execution, valid memory mapping, dependency purity, or runtime result has been established. No runtime tests have been performed.
