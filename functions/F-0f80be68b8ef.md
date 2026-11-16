<!-- Trusted provenance: F-0f80be68b8ef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0f80be68b8ef

## Purpose

F-0f80be68b8ef performs a long, ordered sequence of unresolved indirect invocations, derives values from explicit input bits and dependency results, mutates memory using several access widths, and ends with an unresolved indirect transfer.

The established contract concerns represented arithmetic, access widths, captured values, and ordering. It does not establish an application-level purpose, a conventional object model, or a complete executable interface.

[INFERENCE] Some accesses resemble linked-record updates. Several constants also coincide with character values. Neither observation establishes a collection operation, text generation, or an output stream.

Throughout this document:

- **Established** means directly supported by the supplied static representation.
- **[INFERENCE]** identifies an interpretation requiring additional assumptions.
- **documented_only** identifies unresolved behavior, not permission to supply guessed semantics.

## Interface

### Explicit arguments

| Argument | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No explicit use in the visible body. |
| Second | 32 bits | No explicit use in the visible body. |
| Third | Unsigned 32 bits | Supplies an extracted field to an early invocation. |

The third argument contributes bits 10–19 inclusive, scaled by 32. The resulting argument ranges from 0 through 32,736, in increments of 32. Bits outside that field do not affect this particular extraction.

The first two arguments are not proven globally irrelevant. Unresolved calling conventions and implicit machine-state inputs prevent that conclusion.

### Implicit inputs

Execution also depends on:

- Incoming control-link state used to derive the first invocation target.
- An incoming register value used in shifted invocation arguments.
- Secondary machine-result state following unresolved invocations.
- Global storage and absolute low-address storage.
- Words and bytes used to derive subsequent memory addresses.
- The target of the final indirect transfer.

These inputs have no established default initialization. Zero or null is not an established valid substitute.

### Results and completion

No explicit scalar return value is represented. Observable results comprise dependency interactions, memory reads and writes, and the final indirect transfer.

A return follows that transfer in the representation. Reaching it requires the target to return normally. The function has no independent normal-completion guarantee.

### Preconditions

Completion of the represented path requires executable invocation targets and permitted memory accesses at every reached address and width. Low numerical addresses are used directly; an environment prohibiting those accesses may not support this path.

Address arithmetic, byte order, alignment, and shift behavior must match the actual execution environment. No visible validation establishes these requirements.

## Behavior

### Dependency interactions and arithmetic

The initial phase consists of ordered indirect invocations. The first target is derived from incoming control-link state; later targets are derived from embedded control-location values using a common displacement. No visible loop or conditional branch controls this chain.

Straight-line representation does not imply uninterrupted execution. Any dependency may fault, modify storage, transfer control elsewhere, or fail to return.

Visible argument relationships include:

- A 32-bit result shifted left by eight.
- Low and high 32-bit portions of represented 64-bit results.
- Shifted low words, high words, and secondary result-state values.
- Masked low-order fields of several widths, including 24, 22, 20, 19, 17, 14, 13, 12, 11, and one bit.
- Explicit zeros and repeated arguments derived from the same captured value.
- Logical selections above the low 32-bit result word.

Invocations without visible arguments are not proven to consume no machine arguments. None of these invocations is established as pure.

Most scalar values and address components are represented as 32-bit quantities. Byte stores retain eight bits; halfword stores retain sixteen. Explicit unsigned byte reads widen to values from 0 through 255.

Under a 32-bit bitvector interpretation, left shifts and scaled arithmetic retain the low 32 bits. A left shift followed by a right shift can therefore lose information permanently. Several successive shifts discard all original bits under that interpretation. Expressions represented using signed arithmetic do not, by themselves, establish portable overflow or shift semantics.

### Values entering the mutation phase

The following descriptive roles identify captured values, not object types:

| Role | Meaning |
|---|---|
| Primary base | An earlier 32-bit dependency result multiplied by \(2^{25}\). |
| Tag base | A different captured result shifted left by 25. |
| Shifted payload | A separately captured result shifted left by eight. |
| Constrained field | A late result shifted left by eight, masked to 15 bits, then logically shifted right by six. |
| Secondary base | Twice the constrained field. |
| Final result pair | Low and high words of the last result in the invocation chain. |

The constrained field ranges from 0 through 508 in increments of four. Consequently, the secondary base ranges from 0 through 1,016 in increments of eight. These addresses are not established allocation bases.

### Opening memory mutations

The opening writes establish the following ordering constraints:

- A word at high-final-result plus 0x14 receives the captured tag-base value.
- The byte at tag-base plus 0x09 first receives the low byte of the high final result.
- A word at 256 times the low final result, plus 0x54, receives the primary base.
- The same tag-base byte is overwritten with the low byte of the captured tag-base value.

The primary-base word at 0x14 is then cleared. The byte at primary-base plus 0x05 is read before clearing the secondary-base word at 0x14. That byte is widened without sign extension and stored as a word at primary-base plus 0x34.

The primary-base word at 0x14 is subsequently reread. Its value is not established as zero: the intervening accesses may overlap its storage. Another secondary-base clear follows, then two separate byte stores of the reread value at secondary-base plus 0x15.

Next, an unsigned byte is read from absolute address 0x05. Only afterward is absolute address 0x01 written with the low byte of the secondary base. The captured selector byte becomes an address base for accesses at offsets 0x15 and 0x54.

A word at selector-base plus 0x54 is captured before that location is overwritten. Call this captured word the **selector-field word**. Later scaling and publication addresses depend on this captured value, not on an assumed reread of the overwritten location.

### Record-shaped updates

The intervening mutation region uses offsets 0x14, 0x24, 0x34, 0x44, 0x54, 0x64, and 0x74, together with byte fields. Loaded words become address bases; other bases are formed by doubling, quadrupling, or multiplying captured values by eight.

Established features include:

- Repeated self-valued word stores at offset 0x24.
- Duplicate byte stores at offset 0x15.
- Successive stores of different values to offset 0x54.
- Words loaded at offset 0x74 feeding later indirect accesses.
- A word loaded at offset 0x64 supplying another destination base.
- Address-like values truncated into byte fields at offsets including 0x09, 0x0D, 0x11, and 0x15.
- Two separate reads of a byte at the same computed offset 0x05, separated by other mutations.

Neither duplicate stores nor repeated reads may be replaced by an assumed final state when access events or aliasing matter.

[INFERENCE] The layout resembles record linking and metadata changes. Names such as “next,” “previous,” “owner,” and “reference count” are not established.

### Constants and the publication-shaped region

For the following relationships, **adjusted value** means four times the selector-field word minus 0x96. **Working base** means four times that adjusted value. Arithmetic remains subject to the width and signedness qualifications above.

A later offset-0x74 lookup supplies a word called the **late link value**. Twice that value is the **closing base**; four times it is the **publication payload**.

The constant 0x3A is written:

- As a byte at four times the late link value.
- As a halfword at eight times that value.
- As a word at four times that value plus 0x20.

These are distinct width-sensitive memory operations, not an established contiguous string.

The word at working-base plus 0x20 receives, in order, 0x20, 0x2F, a zero-extended byte read from working-base plus 0x04, and 0x20 again. The byte read follows the first two stores.

A 32-bit global supplying the **publication base** is then read. This read occurs **before** the word store of 0x20 to absolute address 0x60. If that store overlaps the global, the publication base remains the previously captured value; moving the global read afterward would change the contract.

The publication-shaped writes have a meaningful first-to-later transition:

| Event within an instance | Initial instance | Subsequent instances |
|---|---|---|
| Word destination at offset 0x54 | Eight times the selector-field word, plus 0x54 | Eight times the adjusted value, plus 0x54 |
| Value written there | Publication payload | Publication payload |
| Byte at publication-base plus 0x19 | Low byte of working base | Same captured byte |
| Byte at working-base plus 0x11 | Low byte of publication base | Same captured byte |
| Word at publication-base plus 0x34 | Eight times the selector-field word | Eight times the adjusted value |

Within each instance, the offset-0x54 word store precedes the two byte stores, which precede the publication-base offset-0x34 word store.

Thus the initial instance differs in **both** its first destination address and its final stored word. It is not merely an instance with a different last-field value. These relationships use captured arithmetic values; overlapping stores do not cause the publication-base global to be reread.

The representation contains many subsequent instances. Their exact multiplicity remains documented_only in this contract. The repetition is not established as a loop, retry protocol, or idempotent operation.

### Closing-region capture boundary

The closing region has an explicit read/write/read boundary:

- First, a word is read from absolute address 0xA9.
- Next, a word is read from publication-payload plus 0x74.
- Then absolute byte address 0x3A is written with the low eight bits of eight times the adjusted value plus 0x30.
- Only afterward are words read from closing-base plus 0x14 and working-base plus 0x04.

The last two words are therefore post-write captures. The 0x3A write must not be moved after them without establishing that doing so preserves memory observations.

The captured closing-base offset-0x14 word is retained for a later restoration-shaped store and for the third argument of the final transfer. The captured working-base offset-0x04 word is later stored at publication-base plus 0x74.

### Three distinct reads of absolute address 0x67

Following further byte stores and a self-valued word store at closing-base plus 0x54, absolute byte address 0x67 receives the low byte of the captured word from 0xA9.

There are then **three distinct reads** of 0x67:

1. **First read:** occurs after reading the word at closing-base plus 0x44. Its byte is copied to absolute address 0x21.
2. **Second read:** occurs after intervening computed-address writes, a word read at working-base plus 0x40, and further word stores. Its byte is written at working-base plus 0x09.
3. **Third read:** occurs after another byte store, a word store at working-base plus 0x44 using the captured 0xA9 word, and a word read at working-base plus 0x54. Its byte again replaces working-base plus 0x09.

These are not three uses of an immutable captured byte. Computed-address accesses between the reads can alias 0x67. Even the computed word read immediately preceding the first read follows the initial 0x67 write.

Other closing effects include:

- Byte 0x6C at working-base plus 0x08.
- Byte and word stores sharing working-base offset 0x10.
- Successive word stores at publication-base plus 0x54, first using the captured closing-base offset-0x44 word, then the working base.
- A byte at closing-base plus 0x01 derived by right-shifting that captured signed word by the low eight bits of the working base.
- A store of the captured working-base offset-0x04 word at publication-base plus 0x74.
- A word read at working-base plus 0x54 whose result supplies another destination base.
- A later word read at working-base plus 0x14 supplying a truncated byte value.

The signed variable-count shift is documented_only for negative operands or counts outside the 32-bit operand width.

### Final accesses and transfer

A word read at closing-base plus 0x04 supplies the **final base**. A halfword 0x65 is written at twice that base before reading its word at offset 0x64.

The offset-0x64 word is captured before further writes, including a word value 100 at offset 0x54 of the destination obtained from working-base plus 0x54. The final-base word at offset 0x74 is captured afterward.

A word read at closing-base plus 0x64 supplies another destination. Closing writes place the captured final-base offset-0x64 word at that destination plus 0x54, and place that destination value as a word at twice the captured offset-0x64 value.

The final unresolved transfer receives four visible arguments:

1. 0x72.
2. The captured final-base word at offset 0x64.
3. The earlier, post-0x3A-write capture from closing-base plus 0x14.
4. The captured final-base word at offset 0x74.

The second and fourth arguments are captured values, not guaranteed fresh reads at transfer time.

## Side effects

### Access widths and layout

Offsets are access locations, not validated structure definitions.

| Width | Established examples |
|---|---|
| 8 bits | Computed offsets 0x01, 0x05, 0x08, 0x09, 0x0D, 0x10, 0x11, 0x15, 0x19 |
| 16 bits | Scaled destinations receiving 0x3A and 0x65 |
| 32 bits | Computed offsets 0x04, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, 0x74 |

Explicit absolute low-address accesses include bytes at 0x01, 0x05, 0x15, 0x21, 0x3A, and 0x67, and words at 0x54, 0x60, and 0xA9. The word at 0xA9 may require support for unaligned access.

No non-aliasing guarantee is established. Accesses of different widths may overlap. Byte order affects the resulting word values.

The ordering described here is static representation order, not a claim of atomicity, synchronization, or externally visible hardware transaction granularity.

### Ownership and errors

No identified allocation, release, ownership acquisition, or ownership transfer is established. Unknown dependencies may perform any of these operations.

No explicit bounds check, null check, error return, rollback, or transaction boundary is visible. A fault or dependency failure can occur after earlier side effects. Recovery is documented_only.

## Dependencies

The only supplied authoritative opaque function ID is F-0f80be68b8ef. Opaque identities for its indirect dependencies are unavailable; their identity mapping remains documented_only.

The unresolved dependency classes are:

- The incoming-control-derived first target.
- The ordered location-derived invocation targets.
- The final indirect-transfer target.

These descriptions do not establish that any two invocations resolve to distinct functions or to the same function. No dependency is assumed to preserve globals, leave memory unchanged, return normally, or acquire or release ownership.

## Acceptance examples

These are conditional criteria derived statically, not runtime observations.

1. **Input extraction:** Third-argument values 0 and 0x3FF contribute zero to the identified field argument. Value 0x400 contributes 0x20; value 0xFFFFF contributes 0x7FE0.

2. **Extraction-only equivalence:** Values 0x00000400 and 0xFFF00400 both contribute 0x20. This does not establish equivalence of the complete execution trace.

3. **Unsigned widening:** If the primary-base byte at 0x05 is 0xE7 when read, the identified word store receives 231, not −25.

4. **Publication address transition:** Under non-overflowing arithmetic, a selector-field word of 0x40 gives adjusted value 0x6A. The first publication word destination is 0x254; subsequent destinations are 0x3A4. The publication-base offset-0x34 values transition from 0x200 to 0x350.

5. **Global capture before 0x60:** If the publication-base global overlaps the word store at 0x60, publication still uses the value read before that store, assuming the captured value itself remains intact.

6. **Closing capture boundary:** If closing-base plus 0x14 overlaps byte address 0x3A, its capture must incorporate the preceding byte write according to the environment’s byte order. Capturing the old word before the write is not equivalent.

7. **Separate 0x67 observations:** If intervening computed-address writes change 0x67, the second and third reads use the respective later contents. The copy to 0x21 uses the first read, not either later read.

8. **Repeated writes:** An observer capable of distinguishing the working-base offset-0x20 stores must observe 0x20, 0x2F, the captured byte value, and 0x20 in that order. A single final store is insufficient.

9. **Completion:** If all prior operations complete and the final target returns normally, execution reaches the represented return. A nonreturning target provides no such guarantee.

## Uncertainties

The following remain documented_only:

- Complete calling convention and implicit machine-state interface.
- Dependency opaque IDs, signatures, effects, and exact complete invocation trace.
- Fidelity of represented function boundaries and indirect transfers.
- Signed overflow, address wrapping, and variable-count shift semantics.
- Byte order, alignment rules, valid memory regions, and storage extents.
- Memory-mapped access behavior, concurrency, and reentrancy.
- Exact multiplicity of the long repeated-publication region.
- Intended record types, invariants, ownership rules, and application purpose.
- Fault handling and normal-return conditions.

These limits prevent the document from serving as a complete replacement-implementation specification.

## Evidence

Evidence consists solely of the supplied static representation for F-0f80be68b8ef. Comments, prior prose, and reviewer statements are not independent behavioral authority.

The publication destination transition, publication-global read before the 0x60 store, intervening 0x3A write between closing captures, and three separate 0x67 reads are supported by the represented accesses and their ordering.

No implementation access, dependency inspection, runtime tests, or smoke execution has been performed. Acceptance examples are conditional static deductions.
