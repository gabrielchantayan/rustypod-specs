<!-- Trusted provenance: F-509a1eb4ed5b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-509a1eb4ed5b

## Purpose

F-509a1eb4ed5b accepts a 32-bit integer, performs an ordered sequence of indirect invocations, transforms dependency-produced values, mutates memory through computed and literal addresses, and invokes a terminal indirect dependency. No explicit return value is represented.

**Established static evidence:** the supplied representation contains a call-dominated prefix followed by direct memory mutations. No explicit conditional branch, loop, validation path, or recoverable error result is represented. Repeated stores are individually represented operations, not evidence of a runtime loop.

**[INFERENCE]** The shifts, masks, and linked accesses suggest state manipulation or dispatch. No application-level purpose is established. Parsing, encoding, initialization, and object construction are not assumed requirements.

This contract distinguishes captured values from fresh memory reads. Computed storage may overlap other computed storage or literal-address storage; equivalent final contents alone do not establish equivalent behavior.

## Interface

### Inputs and outputs

| Item | Contract |
|---|---|
| Explicit input | One signed 32-bit integer; semantic domain unknown. |
| Explicit output | None represented. |
| Observable outputs | Indirect invocations, memory accesses, state mutations, and terminal dispatch. |
| Explicit error result | None represented. |

The input reaches the first dependency unchanged. A later invocation receives the input shifted left by 25 positions. The body does not directly dereference the input.

Implicit inputs include incoming control-link state, execution-location-derived targets, dependency results, incompletely specified post-call register values, existing memory, an address-global word, and the terminal target. The explicit input alone does not determine behavior.

### Widths and layout

A byte is 8 bits, a halfword 16 bits, and a word 32 bits. Paired dependency results are represented as 64 bits, with separately consumed lower and upper 32-bit words.

Offsets are byte offsets. Byte and halfword stores retain the low eight and sixteen bits respectively. Unsigned byte loads widened to words produce values from 0 through 255.

Computed accesses include byte fields at +1, +4, +5, +8, +9, +0x0d, +0x10, +0x11, +0x15, and +0x19; word fields include +4, +0x10, +0x14, +0x20, +0x24, +0x34, +0x40, +0x44, +0x54, +0x64, and +0x74. Halfword stores use doubled or eightfold-derived addresses.

These offsets do not establish object types. A byte at +0x15 overlaps a word beginning at +0x14; a byte at +0x11 overlaps a word beginning at +0x10. Endianness affects their composition.

### Preconditions and ownership

Reaching the represented end requires executable indirect targets, compatible calling conventions, suitable implicit entry state, supported memory accesses, and completion of preceding operations.

No allocation, release, ownership transfer, reference-count protocol, or caller-supplied output buffer is identifiable in the body. Dependency ownership effects remain unknown. No dependency is assumed to preserve globals, retain memory, or release memory.

## Behavior

### Execution and arithmetic model

Later operations are reached only if earlier calls return and earlier accesses complete. The absence of explicit branches does not guarantee termination or successful completion: dependencies may block, fault, or transfer control elsewhere.

The ordering below describes represented operations, not atomicity, transaction boundaries, barriers, or hardware memory-ordering guarantees.

Most scalar transformations are 32-bit. Under a **conditional 32-bit machine-word interpretation**, multiplication and left shifts retain the low 32 bits. Consequently:

- A left shift by eight followed by logical right shift by eight retains the original low 24 bits.
- Multiplication by 0x02000000 retains information only from the original low seven bits.
- Successive left shifts by 28 and 28 discard all original bits.
- Successive left shifts by 31 and 30 likewise produce zero.

These are bit-vector consequences. Signed overflow, signed left-shift validity, and address wraparound remain **documented_only** rather than assumed source-language guarantees.

### Dependency-driven prefix

The first invocation receives the unchanged input. A later invocation receives its left shift by 25.

The remaining prefix consumes scalar results, paired results, retained earlier values, explicit zero arguments, and incompletely specified post-call register values. Common argument forms include a scalar shifted left by eight, or a paired result supplied as a shifted lower word and an unchanged upper word. Other calls receive shifted upper words, masked fields, duplicated arguments, or retained values transformed after intervening invocations.

Masks select fields of 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, and 11 bits, and a single bit. Logical right shifts do not sign-extend. Some 64-bit extractions select an upper word logically shifted right by two, four, or twelve.

An invocation with no represented explicit arguments may still consume implicit state. An unused result does not make a call dispensable. Explicit zero arguments do not establish reset semantics.

### Retained values entering direct mutation

The following names describe provenance, not declared objects:

| Descriptive value | Derivation |
|---|---|
| **Retained scalar** | An unscaled scalar dependency result retained near the end of the prefix. |
| **Scaled base** | Retained scalar multiplied by 0x02000000. |
| **Small quantity** | Another scalar result shifted left by eight, masked with 0x7fff, then logically shifted right by six. |
| **Doubled-small base** | Twice the small quantity. |
| **Retained shifted value** | An earlier scalar result shifted left by eight. |
| **Retained high-shift value** | An earlier paired result’s lower word shifted left by 25. |
| **Final pair** | The last paired result before direct mutation. |

Under word semantics, the small quantity ranges from 0 through 508 in multiples of four. The doubled-small base ranges from 0 through 1016 in multiples of eight. These small address-like values are not presumed valid pointers.

The retained scalar and scaled base are distinct. Later stores using the retained scalar shifted left by 26 do **not** shift the already-scaled base or a memory-derived seed.

### Opening mutations and literal-byte capture

The final pair’s upper word supplies the base of a +0x14 word store containing the retained high-shift value. The retained high-shift value also supplies the base of two +9 byte stores: first the final upper word’s low byte, then its own low byte. Between these byte stores, a word at 256 times the final lower word, plus 0x54, receives the scaled base.

The opening clear/readback interleaving is significant. The scaled-base +0x14 word is cleared before the scaled-base +5 byte is captured. The doubled-small-base +0x14 word is then cleared, and the captured byte is widened unsigned into scaled-base +0x34. Only afterward is scaled-base +0x14 read back. A second doubled-small-base +0x14 clear follows that readback, followed by two byte stores of the readback’s low byte at doubled-small-base +0x15.

The literal byte at **0x05 is captured and zero-extended before the byte write at 0x01**. The 0x01 write receives the doubled-small base’s low byte. The captured 0x05 byte supplies the base of a +0x15 byte store containing the small quantity’s low byte.

The retained shifted value supplies a byte store at scaled-base +0x11 and a word store at readback-value +0x54; scaled-base +0x11 is subsequently cleared.

A word is then captured from the 0x05-byte-derived base +0x54 before that field is overwritten with the earlier readback. This captured word is the **linked seed**. Its provenance is memory, not the explicit input or retained scalar.

### Linked updates and retained-scalar stores

The doubled-small-base +0x34 word receives the small quantity. A word loaded from scaled-base +0x74 becomes the **early link**. Early-link +0x54 first receives the **retained scalar shifted left by 26**, and later receives the retained shifted value.

A self-value store through doubled-small-base +0x74 uses separately represented reads of that field for the destination base and stored value. Their relative evaluation order within the expression is not established; both reads must remain distinguished unless equivalence is demonstrated.

The linked seed is stored twice at linked-seed +0x24, interleaved with early-link +9 byte writes and the early-link +0x54 update.

The doubled-small-base +0x14 word next receives the **retained scalar shifted left by 26**. Immediately afterward, the literal word at **0x54 receives twice the linked seed**. Subsequent stores to doubled-small-base +0x14 receive the linked seed itself.

Two byte stores at doubled-small-base +0x15 receive the low byte of four times the linked seed. **After both, literal byte 0x15 receives the doubled-small base’s low byte.**

Define the **fourfold-seed base** and **eightfold-seed value** as four and eight times the linked seed. Fourfold-seed-base +0x54 successively receives the retained shifted value and the fourfold-seed base. Doubled-small-base +0x34 receives the fourfold-seed value.

Another fresh doubled-small-base +0x74 word load, doubled, produces the **secondary-link base**. It supplies linked +0x44 updates and later receives its own byte and word mutations.

### Three distinct +5 byte accesses

The byte-driven link selection has three accesses with two different address provenances:

| Access | Address and consequence |
|---|---|
| First | Byte at fourfold-seed-base +5, widened unsigned, selects a word load at that byte value +0x74. The result is the **first byte-selected link**. |
| Second | A subsequent word load at fourfold-seed-base +4 supplies the **indirect-byte base**. A byte read at indirect-byte-base +5 is widened unsigned and stored into first-byte-selected-link +0x44. |
| Third | After intervening writes, indirect-byte-base +5 is read again. This fresh byte selects another +0x74 word load, whose result is the **selected link** used in the final region. |

The first byte-selected link’s +0x44 word initially receives the secondary-link base, before the second byte access later replaces it with an unsigned byte value.

Between the first and second byte accesses, represented mutations include fourfold-seed-base byte fields +0x15 and +0x0d, its +0x14 and +0x54 words, and updates through a word freshly loaded from fourfold-seed-base +0x64. The fourfold-seed-base +4 load occurs after those operations. Secondary-link-base +9 and +0x44 are updated before the second byte access.

After the second byte capture, fourfold-seed-base +9 receives the secondary-link base’s low byte; first-byte-selected-link +0x44 receives the captured unsigned byte. Before the third byte read, fourfold-seed-base +0x14 receives the eightfold-seed value, secondary-link-base +0x15 receives the indirect-byte base’s low byte, and secondary-link-base +0x44 receives the fourfold-seed base.

The third read is therefore a reread of the **indirect-byte** address, not the original fourfold-seed +5 address. Neither its address nor its result may be replaced by the first access without an aliasing argument.

Following that capture and before the later +0x74 load, fourfold-seed-base +0x15 receives the eightfold-seed value’s low byte. Subsequent mutations include that field receiving the indirect-byte base’s low byte and fourfold-seed-base +9 receiving twice the selected link’s low byte.

### Literal payloads and repeated stores

Twice the selected link becomes the **doubled-link base**.

The literal value 0x3a is stored as a byte at four times the selected link, a halfword at eight times it, and a word at four times it plus 0x20.

The **biased quantity** is the fourfold-seed base minus 150. The **adjusted base** is four times that quantity. Adjusted-base +0x20 receives, in order, word values 0x20, 0x2f, an unsigned byte freshly loaded from adjusted-base +4, and 0x20.

The address-global word is then captured as the **retained global base**, before literal word **0x60 receives 0x20**. Later operations use the captured base without reloading the global.

A repeated region preserves these ordered four-store groups:

| Group | Effects in order |
|---|---|
| Initial | Word at eightfold-seed value +0x54 receives fourfold selected link; retained-global-base +0x19 byte receives adjusted-base low byte; adjusted-base +0x11 byte receives retained-global-base low byte; retained-global-base +0x34 word receives eightfold-seed value. |
| Subsequent | Same widths and ordering, but the first destination uses eight times the biased quantity plus 0x54, and the final word value is eight times the biased quantity. |

Both the first destination and final payload differ between the initial and subsequent groups. No calls or global-base reloads are represented within these groups. Exact repetition multiplicity remains **documented_only** here; repeated intermediate writes must not be reduced to final assignments.

### Late captures and literal-byte dependencies

After the repeated region, literal word **0xa9 is read and retained**. A word is then captured from fourfold selected link +0x74. Literal byte **0x3a** receives the low byte of eight times the biased quantity plus 0x30. Words are subsequently captured from doubled-link-base +0x14 and adjusted-base +4.

Before literal byte **0x67** receives the retained 0xa9 word’s low byte, the body updates doubled-link-base +0x0d, writes 0x6c at adjusted-base +8, updates adjusted-base +0x0d and +9, and writes the doubled-link base into its own +0x54 word.

Three fresh 0x67 reads follow:

- After a doubled-link-base +0x44 word capture, a 0x67 byte read supplies the literal byte write at **0x21**.
- After further mutations, another 0x67 byte read supplies adjusted-base +9.
- After a doubled-link-base +9 byte write, an adjusted-base +0x44 word write of the retained 0xa9 word, and an adjusted-base +0x54 word capture, a third 0x67 read supplies adjusted-base +9 again.

Computed writes may alias 0x67. These are not merely repeated uses of the original captured byte.

### Final mutations and dispatch

The doubled-link-base +0x44 capture supplies an adjusted-base +0x10 byte write and a retained-global-base +0x54 word write. A later write to that global-base field replaces it with the adjusted base.

A byte store at doubled-link-base +1 uses that captured word’s signed right shift, with a count equal to the adjusted base’s low eight bits. The count can exceed 31; its machine-specific result remains **documented_only**.

A fresh adjusted-base +0x40 word supplies adjusted-base +0x10. The earlier adjusted-base +4 capture supplies retained-global-base +0x74; retained-global-base +0x44 receives the doubled-link base.

The adjusted-base +0x54 capture becomes a late linked base. Its +0x0d receives the earlier fourfold-selected-link +0x74 capture’s low byte. The same byte reaches doubled-link-base +0x11. Doubled-link-base +0x54 is restored from its earlier +0x14 capture. A fresh adjusted-base +0x14 word later supplies the late linked base’s +0x11 byte. Adjusted-base +9 and +0x44 receive portions of that late linked base.

Near the end, doubled-link-base +4 supplies a **new base**. A halfword 0x65 is stored at twice that base, before its +0x64 word is loaded. The late linked base’s +0x54 receives decimal 100 before new-base +0x74 is captured.

A fresh doubled-link-base +0x64 word supplies another linked destination. Doubled-link-base +9 receives the new base’s low byte; that linked destination’s +0x54 receives the new-base +0x64 capture. The final direct word store, at twice the new-base +0x64 capture, contains the doubled-link-base +0x64 capture.

The terminal dependency receives four represented arguments:

1. 0x72.
2. The new-base +0x64 capture.
3. The earlier doubled-link-base +0x14 capture.
4. The new-base +0x74 capture.

No terminal result is consumed. A return is represented afterward; actual transfer and return guarantees remain unresolved.

## Side effects

Direct effects include byte, halfword, and word accesses through integer-derived addresses and literal low-address storage. Some accesses are unaligned.

Important dependencies include the 0x05 read preceding the 0x01 write; the retained-scalar +0x14 store preceding the linked-seed-derived 0x54 write; the address-global capture preceding the 0x60 write; the 0xa9 capture preceding the 0x3a write; and three fresh 0x67 reads.

No rollback, lock, barrier, transaction, or cleanup handler is represented. Completed effects are not visibly undone after interruption. Dependency effects may include changes to globals, external state, or ownership. Thread safety, reentrancy, idempotence, and repeat-call safety are not established.

## Dependencies

Only the supplied function has a resolved opaque identity.

| Document-local opaque ID | Role |
|---|---|
| F-U001 | Initial indirect target derived from incoming control-link state with represented displacement 0xffe; receives unchanged input. |
| F-U002 | Collection of subsequent execution-location-derived targets; not asserted to be one function. Supplies and consumes scalar words, paired results, and incompletely specified register outputs. |
| F-U003 | Unresolved terminal target receiving the four arguments above. |

Actual identities, ABI, implicit inputs, preservation rules, ownership effects, errors, and return guarantees are **documented_only**. None is assumed pure.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Input propagation.** With input 1 and compatible returning dependencies, F-U001 receives 1. The later input-derived left shift supplies 0x02000000 under word semantics.

2. **Retained-scalar provenance.** If the retained scalar is 1, the scaled base is 0x02000000, while the later retained-scalar shift by 26 supplies 0x04000000. Shifting the already-scaled base by 26 would instead give zero under word semantics and is not equivalent.

3. **Opening readback order.** If computed storage overlaps, the scaled-base +0x14 readback occurs after the first doubled-small clear and widened-byte store, but before the second doubled-small clear. Reading after both clears does not preserve the represented sequence.

4. **Literal source distinction.** A linked seed of 3 supplies 6 to literal word 0x54. A doubled-small base of 0x120 supplies 0x20 to literal byte 0x15, independently of the preceding fourfold-seed-derived byte stores.

5. **Three byte-access provenances.** Suppose fourfold-seed-base +5 yields 0x12, and the word at 0x86 yields 0x400. The first byte-selected link is 0x400. If the later fourfold-seed-base +4 word supplies 0x300 and byte 0x305 yields 0x81, the middle update stores word value 129 at 0x444. It does not use 0x12 or read fourfold-seed-base +5 again.

6. **Same-address reread.** Continuing conditionally, if an intervening computed write changes byte 0x305 to 0x22, the third byte access selects the word at 0x96. The earlier 0x81 capture remains the source of its completed +0x44 store.

7. **Fresh literal reads.** If the 0xa9 capture ends in 0xa5, the initial 0x67 write carries 0xa5. If a later overlapping store changes 0x67 to 0x3c, subsequent reads use the byte present then, not necessarily 0xa5.

8. **Partial completion and terminal capture.** If an earlier dependency does not return, later operations are not reached. If F-U003 is reached, its third argument remains the earlier doubled-link-base +0x14 capture even if later writes changed that memory.

## Uncertainties

The following remain **documented_only**:

- Application purpose, valid input domain, and actual function boundary.
- Indirect target identities, ABI, and implicit register production.
- Endianness, alignment behavior, signed arithmetic semantics, and address wraparound.
- Signed-right-shift behavior, especially counts above 31.
- Memory mapping, aliasing, device semantics, and concurrency requirements.
- Exact multiplicity of the long repeated-store region.
- Dependency preservation, allocation, retention, release, and failure behavior.
- Actual terminal transfer and normal-return guarantees.

The retained-scalar provenance and the three distinct +5 byte dependencies are established represented relationships, not unresolved alternatives.

## Evidence

This contract uses static examination of the supplied representation. Evidence includes represented access widths, constants, value provenance, captures, separate-operation ordering, arithmetic transformations, repeated stores, and terminal arguments. Comments and prior feedback do not independently establish behavior.

Descriptive base names express dependencies without asserting object layouts or ownership. Higher-level intent is explicitly marked **[INFERENCE]**; arithmetic consequences are conditional where machine semantics remain unresolved.

No runtime tests, dependency executions, memory traces, or target-ABI checks have been performed. No independently verified provenance or snapshot hash is asserted.
