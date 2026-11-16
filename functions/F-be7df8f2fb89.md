<!-- Trusted provenance: F-be7df8f2fb89; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-be7df8f2fb89

## Purpose

F-be7df8f2fb89 performs a dependency-driven sequence of indirect invocations, transforms returned words, and applies ordered, mixed-width memory updates before a final indirect transfer.

The supplied static representation does not establish a higher-level purpose. It does not demonstrate parsing, text construction, object initialization, or maintenance of a particular collection. Character-like constants and pointer-like stores do not establish those interpretations.

This contract distinguishes:

- **Established static evidence:** represented operations, widths, data dependencies, and access ordering.
- **[INFERENCE]:** possible intent, not an acceptance requirement.
- **documented_only:** behavior left unresolved by this contract or the available evidence.

The contract is partial at dependency and execution-model boundaries. It does not establish that the represented path executes successfully in an ordinary process address space.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits, uninterpreted | Forwarded unchanged to the first indirect invocation. |
| Second argument | Signed 32 bits | Shifted left six positions for the second argument of the first invocation. |

Neither explicit argument is directly dereferenced by the represented body. Their interpretation by dependencies is unknown; either could represent an address, handle, packed value, or ordinary number.

No explicit buffer extent, capacity, ownership designation, or error-output argument is represented.

### Implicit inputs

Behavior also depends on:

- Entry control state determining the first indirect target.
- Subsequent indirect targets and their calling conventions.
- Primary and secondary dependency-result channels.
- Memory contents at computed addresses.
- Fixed-storage reads, notably the byte at 0x05, the word at 0x087fbbbc, the late word at 0xa9, and successive bytes at 0x67.
- An unresolved final indirect target.

Fixed-storage contents are consumed when their reads are reached, not necessarily as they existed at entry. Earlier dependencies or computed stores may have changed them.

### Outputs and completion

No explicit return value is represented. Observable outputs consist principally of dependency interactions and memory effects.

The body ends with a four-argument indirect invocation and a represented return. Normal completion requires that execution reach that invocation and that the invocation return. There is no established success flag, failure flag, or completed-output object.

### Preconditions

Successful execution requires callable indirect targets, meaningful consumed result channels, and valid storage for every reached access. The execution environment must supply any required alignment, address mapping, and interpretation of fixed storage.

These are prerequisites, not validations performed by the function. No bounds checks, null checks, or target checks are visible.

## Behavior

### Dependency-driven phase

A long, straight-line sequence of indirect invocations precedes the first explicit data-memory store. No input-controlled loop or conditional branch is represented in this phase.

The first invocation receives the unchanged first argument and the second argument shifted left six positions. Its target depends on entry control state. Later targets are derived from represented code-location values or references using a common displacement.

The sequence consumes primary words, secondary result words, and represented 64-bit results split into lower and upper 32-bit halves. Later invocations receive combinations of these values, shifts, masks, retained earlier results, and explicit zeros.

Invocations with no represented arguments must not be interpreted as having no implicit inputs or effects. Repeated argument patterns do not justify combining calls: their targets and intervening state can differ. No dependency is established to be pure, repeatable, global-preserving, or memory-preserving.

The complete call-by-call argument trace and dependency semantics remain **documented_only** in this compact contract.

### Widths and arithmetic

Most represented calculations operate on 32-bit words. Selected result containers are 64 bits; this does not establish a 64-bit address space. Memory accesses include 8-, 16-, and 32-bit widths.

Represented transformations include logical right shifts of unsigned words, left shifts, low-field masks, upper-word extraction, and extraction above bit 31 of a 64-bit result. Some operations extract bits from an earlier retained result despite intervening invocations.

A **32-bit bitvector interpretation**, where explicitly assumed below, retains only the low 32 bits of arithmetic and left-shift results. It is not an unconditional interpretation of signed operations. Signed left shifts and multiplication may exceed the represented signed range; portable overflow behavior is not established.

Operation order matters. A left shift followed by a right shift can lose high bits. Successive left shifts can discard every original bit even when each individual count is below 32.

Byte stores retain eight bits; halfword stores retain sixteen. Explicit unsigned byte loads widen to values from 0 through 255.

### Initial memory boundary

The following descriptive roles distinguish retained values; they are not object or field names:

| Role | Derivation |
|---|---|
| Dispatch-scaled word | A retained dependency-result word shifted left 25 positions. |
| Late shifted word | A different retained dependency-result word shifted left eight positions. |
| High-scaled base | Another dependency result multiplied by \(2^{25}\). |
| Narrow-derived base | A result shifted left eight, masked to 15 low bits, logically shifted right six, then doubled. |
| Last result pair | Lower and upper words of the final dependency result before explicit stores. |

The opening effects preserve this order:

| Order | Access |
|---:|---|
| 1 | Store the dispatch-scaled word as 32 bits at the last upper word plus 0x14. |
| 2 | Store the low byte of that upper word at the dispatch-scaled word plus 9. |
| 3 | Store the high-scaled base as 32 bits at 256 times the last lower word, plus 0x54. |
| 4 | Overwrite the same offset-9 byte with the low byte of the dispatch-scaled word. |
| 5 | Store a 32-bit zero at the high-scaled base plus 0x14. |
| 6 | Read an unsigned byte at the high-scaled base plus 5. |
| 7 | Store a 32-bit zero at the narrow-derived base plus 0x14. |
| 8 | Store the previously read byte, widened to 32 bits, at the high-scaled base plus 0x34. |
| 9 | Read the current 32-bit word at the high-scaled base plus 0x14. |

The last read is not unconditionally zero: the intervening store through the narrow-derived base can overlap it.

Further stores again clear the narrow-derived base’s offset-0x14 word and twice write the low byte of the just-read word at offset 0x15. Only after these effects does the fixed byte read at 0x05 occur.

### Early fixed-storage dependency and computed updates

The byte read at fixed address 0x05 is widened and retained as an address base. It precedes the byte write at fixed address 0x01, which receives the low byte of the narrow-derived base.

The retained fixed-byte-derived base is used for a byte store at offset 0x15 and a word read at offset 0x54. That word is retained before the offset-0x54 location is overwritten. Later operations use the retained word in address calculations, self-referential word stores, and scaling by two, four, and eight.

Subsequent effects include a word write at fixed address 0x54 and a byte write at fixed address 0x15. Computed accesses may overlap these locations or the earlier fixed storage.

The memory phase includes word reads at offsets 4, 0x14, 0x44, 0x54, 0x64, and 0x74; byte reads at offset 5; repeated overwrites; and stores of retained values rather than refreshed memory contents.

**Address indirection is demonstrated through values loaded at offsets 0x54, 0x64, and 0x74. Offset 0x44 has a different role.** Its late word load supplies stored data and the operand of a signed right shift. That loaded word is not subsequently used as a store address in the represented data flow.

A byte load can itself supply the base for a later word read at offset 0x74. No object separation or minimum allocation extent follows from these access forms.

### Overwrites and the retained repeated-region base

Before the long repeated store region, 0x3a is written by separate byte, halfword, and word accesses at related scaled addresses.

A computed offset-0x20 word then receives four ordered writes:

1. 0x20.
2. 0x2f.
3. A widened unsigned byte read from the related computed base plus 4.
4. 0x20 again.

The byte read occurs between the second and third writes. Its value reflects memory at that point.

**After all four writes, a 32-bit word is read at fixed address 0x087fbbbc. That value is retained as the repeated-region base. The read precedes the fixed 32-bit write of 0x20 at address 0x60.**

Thus the repeated-region base is not a dependency return or a reload from one of the repeatedly updated offsets. Earlier effects can influence the fixed word before its read. Once loaded, the base remains the retained value even if later stores overlap 0x087fbbbc.

The repeated region relates four kinds of effects:

- A scaled-address word store at offset 0x54.
- A byte store at offset 0x19 from the retained fixed-word-derived base.
- A byte store at offset 0x11 from another retained computed base.
- A word store at offset 0x34 from the fixed-word-derived base.

Within each represented group, the offset-0x54 word store precedes the two byte stores, which precede the offset-0x34 word store. The first group uses a different scaled value from the subsequent repeated groups.

This is a fixed straight-line repetition, not a demonstrated variable-length traversal. The exact group count and exhaustive alias-sensitive trace remain **documented_only** here. Repeated stores must not be collapsed merely because their final values appear redundant.

### Late fixed-word read and live fixed-byte reads

After the repeated region, a 32-bit word is read at fixed address 0xa9. This word is retained separately from later byte reads at 0x67.

The established ordering and uses are:

- The word read at 0xa9 precedes a computed word read at offset 0x74.
- A fixed byte write at 0x3a follows those reads. Its value is derived from a retained adjusted value multiplied by eight and increased by 0x30, narrowed to eight bits.
- A computed offset-0x14 word is then read and retained for the final invocation’s third argument. Another word is retained from a computed offset 4.
- Computed byte and word stores follow, including a byte value 0x6c at offset 8 and a self-base word store at offset 0x54.
- The low byte of the retained word from 0xa9 is written to fixed address 0x67.
- A computed offset-0x44 word is loaded.
- The byte at 0x67 is read for the byte write at fixed address 0x21.
- After further computed stores, 0x67 is read again for a computed offset-9 byte store.
- After additional computed stores and a word read at offset 0x54, 0x67 is read again for another computed offset-9 byte store.

These are distinct reads of storage, not repeated uses of a saved byte. In particular, the offset-0x44 word read lies between the write to 0x67 and its read for 0x21. Later computed stores may overlap 0x67 and change either subsequent byte read.

The retained full word from 0xa9 is also stored as 32-bit data at a computed offset 0x44. That store does not reread 0xa9 and is not defined by the then-current byte at 0x67.

The loaded offset-0x44 word is stored, narrowed for another byte store, and used as the signed right-shift operand. The shift count is the low eight bits of a retained computed value. The shifted result is narrowed to a byte. Counts of 32 or greater and negative-operand behavior require an execution model and remain **documented_only**.

### Final updates and transfer

The remaining region includes:

- Address indirection through a word loaded at offset 0x54.
- A halfword write of 0x65 at twice a loaded word.
- A word write of decimal 100 at an indirectly obtained base plus 0x54.
- Word loads at offsets 0x64 and 0x74.
- Further byte stores, an indirect offset-0x54 word store, and a word store at twice a retained loaded value.

The final invocation receives:

| Argument | Represented value |
|---|---|
| First | 0x72. |
| Second | A retained word loaded at offset 0x64 from a late base obtained through an offset-4 word read. |
| Third | The earlier retained offset-0x14 word read shortly after the fixed byte write at 0x3a. |
| Fourth | A retained word loaded at offset 0x74 from the same late base used for the second argument. |

The second argument is loaded before the fourth. Further writes occur after both loads and before the invocation. None of these arguments is automatically refreshed when its original storage changes.

## Side effects

### Storage and layout

Offsets are byte offsets, not established structure fields.

| Width | Represented access forms |
|---|---|
| 8 bits | Offsets including 1, 5, 8, 9, 0x0d, 0x10, 0x11, 0x15, and 0x19; computed scaled addresses; fixed storage. |
| 16 bits | Addresses formed by multiplying loaded values by two or eight. |
| 32 bits | Offsets including 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74; fixed storage. |

The meaningful fixed reads are the early byte at 0x05, the repeated-region base word at 0x087fbbbc, the late word at 0xa9, and the successive bytes at 0x67. Their ordering is specified above. Fixed writes include bytes at 0x01, 0x15, 0x21, 0x3a, and 0x67, and words at 0x54 and 0x60.

No disjointness between fixed and computed storage is established. Narrow writes can overlap word reads. Endianness is unresolved, so numerical consequences of overlapping mixed-width accesses require an explicit byte-order assumption.

### Ownership and errors

No identifiable explicit allocation, release, retain, or ownership-transfer operation is represented. Unknown dependencies may nevertheless allocate, release, invalidate, or transfer storage.

No explicit error detection, rollback, recovery, locking, atomic operation, or memory barrier is visible. A fault or nonreturning dependency can prevent completion after earlier observable effects. The function is not established to be transactional or confined to argument-owned memory.

Concurrency, reentrancy, and fault semantics remain **documented_only**.

## Dependencies

F-be7df8f2fb89 is the only supplied opaque function identity. No opaque IDs or contracts are supplied for its indirect targets; this document does not invent identities for them.

Unresolved boundaries include the entry-state-derived invocation, later location-derived invocations, primary and secondary result channels, and the final indirect transfer.

Argument interpretation, global mutation, memory effects, ownership effects, exceptional behavior, and return guarantees remain unknown at each boundary. In particular, dependencies must not be assumed to preserve the fixed storage read later by this function.

## Acceptance examples

These are conditional consequences, not runtime test results.

### Input transformation

Under a 32-bit bitvector interpretation, a second argument of 3 produces 192 for the first invocation’s second argument. A second-argument bit pattern of 0x04000000 produces zero after the six-position shift. The first argument remains unchanged.

### Narrow-derived range

Under the same interpretation, shifting left eight before selecting the low 15 bits leaves a multiple of 256. After the right shift and doubling, the narrow-derived base is a multiple of eight from 0 through 1016.

### Offset-0x44 data use

If the late offset-0x44 read yields a word whose numerical value is not a writable address, that fact alone does not invalidate this data flow: the word supplies stored data and a shift operand, not a store destination. Other independently computed addresses must still be valid.

### Ordered fixed-base acquisition

If an earlier computed store changes the word at 0x087fbbbc before its read, the repeated region uses the word present at the read. Acceptance must place this read after the four offset-0x20 writes and before the word write at 0x60.

If a later repeated-region store changes 0x087fbbbc, subsequent groups still use the retained base; they do not reload that fixed word.

### Retained word versus live byte

Suppose the read at 0xa9 yields 0x12345678. The initial write at 0x67 receives 0x78. The later full-word data store sourced from that retained read receives 0x12345678.

If a computed store subsequently changes 0x67 to 0xab before one of the later byte reads, the corresponding computed offset-9 store receives 0xab. It must not reuse 0x78 merely because that was originally written to 0x67.

### Observable overwrites and retained arguments

An observer able to distinguish the four offset-0x20 writes must see their ordered values, including the intervening byte-derived value; a final 0x20 alone is insufficient.

Likewise, changing storage after it supplied the final invocation’s third argument does not change that retained argument. No successful completion is promised if a dependency does not return or a reached address faults.

## Uncertainties

The following remain **documented_only**:

- Higher-level purpose and intended consumer.
- Actual ABI, secondary-result conventions, and indirect-target identities.
- Complete invocation trace and dependency contracts.
- Function-boundary accuracy and final-transfer classification.
- Pointer width, address normalization, alignment, and byte order.
- Signed overflow, signed shifts, and oversized variable-shift counts.
- Accessibility and external meaning of fixed storage.
- Initial fixed-storage contents and mutations by unknown dependencies.
- Whether computed accesses alias fixed storage, including 0x087fbbbc and 0x67.
- Exact repetition count and exhaustive memory trace outside the explicitly described ordering.
- Allocation extents, ownership, lifetime, synchronization, and fault behavior.

The existence, width, ordering, and retained-versus-reread roles of the specified fixed reads are established; their environmental values and meaning are not.

**[INFERENCE]** Masks and shifts may manipulate packed fields, and repeated address-like stores may update linked state. Neither interpretation establishes a format or data-structure invariant.

## Evidence

This contract uses the supplied static representation and opaque dependency map. Prior prose, reviewer feedback, embedded comments, and warnings are not independent proof.

Static evidence supports the input widths, indirect invocations, result extraction, arithmetic transformations, mixed-width memory effects, fixed-storage dependencies, overwrite ordering, offset-0x44 data use, demonstrated indirection through offsets 0x54/0x64/0x74, and retained final arguments.

No runtime execution, implementation access, smoke run, or tests were performed. Acceptance examples are conditional specification examples, not observed outcomes.
