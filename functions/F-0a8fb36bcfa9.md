<!-- Trusted provenance: F-0a8fb36bcfa9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0a8fb36bcfa9

## Purpose

F-0a8fb36bcfa9 performs an ordered sequence of indirect dependency invocations, transforms dependency-produced bit patterns, updates memory through computed and fixed addresses, and finishes with an unresolved indirect invocation.

The supplied static representation establishes execution shape, data relationships, access widths, and ordering. It does not establish an application-level purpose. Repeated field offsets and character-like constants do not establish formatting, decoding, rendering, allocation, or collection-management semantics.

This contract distinguishes:

- **Established static evidence:** operations and relationships explicitly represented in the supplied material.
- **[INFERENCE]:** conclusions conditional on an arithmetic or execution model not independently supplied.
- **documented_only:** unresolved behavior that does not support an unconditional executable acceptance requirement.

All behavioral requirements are conditional on execution reaching the operation concerned and on the supplied representation accurately describing it. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits, opaque | Forwarded unchanged to the initial dependency. |
| Second argument | Signed 32 bits | Forwarded unchanged as the initial dependency’s second operand; shifted left six positions for its third operand. |

Neither argument has an established pointer, size, handle, enumeration, or ownership meaning. No argument validation precedes the initial invocation.

### Implicit inputs

Behavior additionally depends on:

- Incoming execution context selecting the initial indirect destination.
- Dependency results, including represented 64-bit values and incompletely specified secondary result components.
- Existing contents of computed-address storage.
- Fixed-location storage read during execution.
- The unresolved final destination.

Fixed-location inputs are **execution-time reads, not entry-state snapshots**. Earlier dependencies and computed stores may change their contents before capture.

### Output and completion

The represented interface has no explicit return value.

The final invocation receives four operands: 0x72 and three captured 32-bit values. A return follows in the representation, but the final destination’s actual return, termination, fault, or control-transfer behavior is **documented_only**.

No success flag, error code, exception translation, rollback, or completion notification is established.

### Preconditions for completion

Every executed destination must be callable, every executed memory access must have valid storage for its represented width, and every preceding dependency must return for execution to reach final dispatch. Some accesses require low-address storage if interpreted literally.

These are conditions for successful execution, not checks performed by the function. Alignment requirements, memory-map validity, and calling convention remain **documented_only**.

## Behavior

### Indirect dependency phase

A long straight-line sequence of indirect invocations precedes the explicit direct-memory phase. No input-dependent branch or loop is explicitly represented.

The initial destination is selected by adding 0xffe to incoming execution context. It receives the unchanged first argument, unchanged second argument, and second argument shifted left six positions.

Later destinations are derived from represented control-location values using the same displacement. Similar invocation shapes do not establish identical dependency implementations.

Call sites present zero through four explicit operands. A site with no explicit operands may still consume implicit state and have side effects. Results are consumed as single 32-bit values, low and high components of represented 64-bit values, and secondary components without complete ordinary return signatures. Some captures remain in use across intervening calls.

No dependency is assumed to preserve globals, maintain memory validity, or refrain from allocation or release.

### Widths, extraction, and arithmetic

Most represented intermediates are 32 bits. Some extraction explicitly uses unsigned 64-bit containers.

Transformations include fixed left shifts through 31 positions, unsigned right shifts, field masking, power-of-two multiplication, subtraction of 0x96, and byte conversion or truncation. Masks select fields including 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, and one bit. Explicit unsigned-byte loads are zero-extended when stored as words.

For unsigned 64-bit extraction:

| Right-shift count before 32-bit extraction | Relationship to the original high word |
|---:|---|
| 32 | Unchanged high word |
| 34 | High word shifted right two positions |
| 36 | High word shifted right four positions |
| 44 | High word shifted right 12 positions |

Some extractions intentionally consume an older retained result rather than the latest dependency result.

**[INFERENCE]** Under a 32-bit bitvector model, shifts and multiplication discard bits above bit 31. Two successive left shifts by 28 positions therefore produce zero. Widening the intermediate is not equivalent.

Signed overflow, signed left shifts, signed right shifts of negative values, and address wraparound remain **documented_only** where the execution model is not supplied. The late variable right shift uses a count obtained from a computed value’s low eight bits; behavior for counts above 31 is unresolved.

### Opening direct-memory effects

The last dependency result supplies low and high components for address construction. Earlier captures supply scaled bases and a constrained field.

The opening accesses establish these ordering boundaries:

- A 32-bit store at the high component plus 0x14 precedes a byte store at another base plus nine.
- A 32-bit store at the low component scaled by 256, plus 0x54, precedes a second store to that same byte destination.
- A computed offset-0x14 word is cleared before a byte is read at offset five of its associated base.
- A different word is cleared before that captured byte is zero-extended into the associated offset-0x34 word.

The **constrained field** is formed by shifting a 32-bit result left eight positions, retaining 15 low bits, then shifting right six positions. Under the bitvector interpretation, it ranges from zero through 508 in multiples of four. The **doubled constrained field** ranges from zero through 1016 in multiples of eight.

The cleared offset-0x14 word is subsequently read and retained as an address base. Without intervening aliasing modification, it yields zero. A later computed store through this captured base plus 0x54 consequently targets absolute address 0x54 in that case. This is a conditional computed-address effect, distinct from the explicit fixed-location store described below.

### Early fixed-location capture and retained replacement word

After two consecutive byte stores at the doubled constrained field plus 0x15, the function reads fixed-location storage at **0x05**, expands its value into a 32-bit quantity, and retains it as a base.

The representation does not separately declare the fixed symbol’s storage width. Its exact load width is **documented_only**; a 32-bit load must not be assumed merely from the width of the expanded result.

After this capture:

- Absolute address **0x01** receives the low byte of the doubled constrained field.
- The captured 0x05-derived base plus 0x15 receives the low byte of the constrained field.
- Other computed stores occur, including the conditional base-plus-0x54 store described above.
- The 32-bit word at the captured 0x05-derived base plus 0x54 is read and retained before that field is replaced.

The **retained replacement word** below means this pre-replacement word. It is not a subsequent reload of the replaced field.

Earlier computed writes may affect the read at 0x05 or the captured base’s offset-0x54 word through aliasing. Later writes do not retroactively replace either capture.

### Explicit fixed-location stores at 0x54 and 0x15

After intervening word and byte updates, the doubled constrained field’s offset-0x14 word is written with a dependency-derived value shifted left 26 positions. The explicit **32-bit store at absolute address 0x54** follows. Its value is the retained replacement word shifted left once.

The same computed offset-0x14 word then receives the retained replacement word, followed by another identical word store there. Thus the fixed-location 0x54 store occurs **between** the dependency-derived offset-0x14 write and the following replacement-word writes. It must not be moved outside that boundary.

This fixed-location store is required whenever reached, regardless of whether the earlier computed base-plus-0x54 store happened to target 0x54. Their addresses may coincide, but their values, provenance, and execution positions differ.

Next, two consecutive computed byte stores at the doubled constrained field plus 0x15 each receive the low byte of four times the retained replacement word. **After both stores**, the explicit assignment at absolute address **0x15** receives the low byte of the doubled constrained field.

The fixed-location 0x15 assignment does not copy either preceding byte value. If the doubled constrained field is zero, all three stores target 0x15, and the explicit assignment is the last of those three events.

### Memory-linked updates

The middle phase repeatedly captures words and uses them as later bases. Established effects include:

- Clearing, replacing, and repeatedly overwriting 32-bit fields.
- Storing a base value through storage reached from that base.
- Following offset-0x74 words before accessing offset-0x54 fields.
- Copying low bytes of larger values into nearby byte fields.
- Loading unsigned bytes to select further addresses.
- Capturing words at offsets four and 0x64 for later use.

Repeated loads from apparently identical storage remain separate reads. In particular, a word reached through an offset-0x74 field can be read more than once within a represented update; the material does not justify replacing every occurrence with one capture.

These access shapes do not establish object boundaries, list membership, parent-child relationships, or ownership transfer. Earlier stores may change subsequent values and addresses through aliasing.

### Mixed-width constants and adjusted base

A later region stores 0x3a as an eight-bit value, a 16-bit value, and a 32-bit value at computed destinations. Their widths remain distinct even where numeric values agree.

The **adjusted base** is formed from four times the retained replacement word by subtracting 0x96. A **working base** is then formed by multiplying that adjusted base by four. These retain their represented 32-bit arithmetic limitations.

The working base’s offset-0x20 word receives, in order:

- 0x20.
- 0x2f.
- A zero-extended byte read from the working base plus four.
- 0x20 again.

The byte read occurs after the first two word stores. Aliasing may therefore affect its value.

Only after this sequence does the function capture a fixed-location 32-bit word as the **publication base**. It then stores the 32-bit constant 0x20 at absolute address **0x60**. Moving the publication-base capture after that store is not equivalent.

“Publication” identifies this document’s access phase, not a proven synchronization or application semantic.

### Publication groups

Each represented group contains, in order:

1. A 32-bit store at a scaled base plus 0x54.
2. A byte store of the working base’s low byte at publication base plus 0x19.
3. A byte store of the publication base’s low byte at working base plus 0x11.
4. A 32-bit store of the scaled base at publication base plus 0x34.

The first group uses eight times the retained replacement word as its scaled base. Following groups use eight times the adjusted base. The offset-0x54 stores use the same retained value derived from a previously loaded word multiplied by four.

The groups are explicit repetitions, not an input-controlled loop. Their exact multiplicity is not independently validated in this contract. Captured bases and values remain in use even if stores alias their original source locations.

### Late capture and fixed-location write at 0x3a

After all represented publication groups, the function captures the **32-bit word at absolute address 0xa9**. It then reads a computed offset-0x74 word before writing the byte at absolute address **0x3a**.

The 0x3a value has a specific relationship to the adjusted base:

- The adjusted base is first converted to the representation’s plain character type.
- That converted value is multiplied by eight.
- 0x30 is added.
- The result is assigned to the eight-bit fixed-location destination.

Conversion precedes multiplication and addition. The stored value is not derived from the captured 0xa9 word, and the conversion must not silently be moved after the arithmetic.

Plain-character signedness, out-of-range conversion behavior, and unresolved final assignment conversions remain **documented_only**. Under a separately assumed modulo-256 byte model, the stored byte is the low eight bits of the resulting arithmetic value; that model is not an unconditional platform guarantee.

Further captures from computed offset-0x14 and offset-four words follow the 0x3a assignment.

### Three fresh reads at 0x67

The retained 0xa9 word’s low byte is assigned to absolute address **0x67**. Its full word is retained for a later computed offset-0x44 store.

The function then performs three distinct reads of byte storage at 0x67:

1. After an intervening computed offset-0x44 word read, it reads 0x67 for a byte assignment to absolute address **0x21**.
2. After further computed writes, including a 32-bit offset-0x10 store, it reads 0x67 for a byte store at working base plus nine.
3. After additional writes and a 32-bit offset-0x54 read, it reads 0x67 again for another byte store at working base plus nine.

Intervening writes may alias 0x67, so these reads need not agree. They must not generally be replaced with the captured 0xa9 word’s low byte or with the first byte read.

The intervening word read before the first copy is independently significant: it may fault or have device-specific effects.

### Final mutations and dispatch

Late effects include an eight-bit store of 0x6c, replacement of fields at offsets 0x10, 0x44, 0x54, and 0x74, and the variable signed right shift described above.

A byte store at working base plus 0x10 precedes a **32-bit store** there using a word read from working base plus 0x40. These are different access widths and events.

Closing effects include a 16-bit store of 0x65 at twice a loaded base, a 32-bit store of decimal 100 at a computed offset-0x54 destination, and a word store at twice another captured base.

The final dependency receives, in order:

- 0x72.
- A word captured from offset 0x64 of a late-loaded base.
- The earlier late-phase capture from an offset-0x14 field.
- A later word captured from offset 0x74 of that late-loaded base.

These are retained operands, not final reloads after subsequent mutations. No cleanup or rollback precedes dispatch.

## Side effects

### Access layout

This inventory describes observed accesses, not a common structure declaration.

| Width | Represented offsets or destinations |
|---|---|
| 8 bits | Offsets 1, 5, 8, 9, 0x0d, 0x10, 0x11, 0x15, 0x19; fixed low-address storage; scaled destinations |
| 16 bits | Scaled destinations receiving 0x3a and 0x65 |
| 32 bits | Offsets 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, 0x74; fixed-location words |

Explicit fixed-location writes include bytes at 0x01, 0x15, 0x21, 0x3a, and 0x67, plus words at 0x54 and 0x60. Fixed reads include the incompletely typed capture at 0x05, three late byte reads at 0x67, and a word capture at 0xa9.

Computed accesses may additionally target these same locations. Byte and word accesses may overlap; byte order controls their interaction and is not supplied.

### Errors, ordering, and ownership

Repeated or overwritten stores remain represented access events. Final-state equality alone does not establish equivalence for aliasing storage, device memory, fault ordering, or external observers.

No bounds, null, alignment, or dependency-result checks are represented. No recovery path is supplied. A failing call or access may leave earlier effects intact; atomicity is not established.

Direct operations identify no allocation, release, retain, or ownership transfer. Unknown dependencies may perform such operations or invalidate storage. Synchronization, thread safety, reentrancy, and dependency preservation guarantees are **documented_only**.

## Dependencies

Only the subject function has a supplied canonical opaque identity. The following are document-local opaque role IDs, not recovered implementation identities:

| Opaque ID | Role | Established interface evidence |
|---|---|---|
| F-D001 | Initial indirect destination | Three 32-bit operands; context-derived destination. |
| F-D002 | Intermediate indirect destinations | Varying operand counts; single-word, two-word, and incompletely specified result components. |
| F-D003 | Final unresolved destination | Four operands, beginning with 0x72. |

F-D002 groups unresolved sites without asserting one shared implementation. Calling conventions, implicit inputs, global effects, ownership effects, error signaling, and termination remain **documented_only**.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Forwarding and extraction

For second-argument value three, F-D001 receives three unchanged as its second operand and 0xc0 as its third. The first argument remains unchanged.

Under the 32-bit bitvector model, second-argument bit pattern 0x04000000 produces a zero third operand.

A represented 64-bit result whose high word is 0x80000010 yields 0x20000004 after unsigned right shift by 34 positions and 32-bit extraction. An explicit unsigned-byte load of 0xff expands to 255, not −1.

### Distinct stores at 0x54

If the cleared address field remains zero until capture, its later computed base-plus-0x54 store targets absolute address 0x54.

Separately, if the retained pre-replacement word from the 0x05-derived base’s offset-0x54 field is three, the explicit fixed-location word store writes six at 0x54. Its ordering is after the dependency-derived write at doubled constrained field plus 0x14 and before the following writes of three there.

Skipping the explicit store because a computed store previously reached 0x54 fails this criterion.

### Explicit assignment at 0x15

Suppose the constrained field is zero and the retained replacement word is three. The two consecutive computed offset-0x15 stores each write 0x0c to absolute address 0x15. The following explicit assignment writes zero there, because it uses the doubled constrained field’s low byte.

Retaining 0x0c as the outcome of those three accesses would omit an established effect.

### Character conversion at 0x3a

If the retained replacement word is 0x26 and arithmetic is ordinary nonoverflowing signed arithmetic, the adjusted base is two. Character conversion preserves two; multiplication by eight and addition of 0x30 produce 0x40 for the byte assignment.

If the adjusted base is 0x102, the result is conditional on character-conversion semantics. Under conversion retaining the low eight bits, the converted value is two and the stored byte is again 0x40. This example does not establish that conversion model universally.

### Captures and fresh reads

An earlier aliasing write to 0x05 must affect its later capture. The publication-base capture follows the offset-0x20 sequence and precedes the 0x60 word store. The 0xa9 capture follows publication and precedes the subsequent offset-0x74 read and 0x3a write.

If the retained 0xa9 word has low byte 0x12 and 0x67 remains unchanged before its first fresh read, 0x21 receives 0x12. If an intervening computed store changes 0x67 to 0xab, the next copy reads 0xab. The third read may differ again.

### Final dispatch

If preceding operations complete, F-D003 receives 0x72 followed by the specified retained words. Reloading their source fields after mutations is not generally equivalent. Normal return remains conditional on the final destination returning.

## Uncertainties

The following remain **documented_only**:

- Application meaning of arguments, fields, and constants.
- Accuracy and literal executability of the complete static representation.
- Individual dependency identities, signatures, secondary-result origins, and effects.
- Signed overflow, address arithmetic, negative signed right shifts, and excessive shift counts.
- Plain-character signedness and unresolved character or assignment conversions.
- Storage width of the fixed symbol read at 0x05.
- Byte order, alignment, memory validity, and accessibility of low addresses.
- Structure boundaries, aliasing, concurrency, ownership, and lifetime guarantees.
- Independently validated publication repetition count.
- Error recovery and final-destination return behavior.

These uncertainties do not obscure the established explicit stores at 0x54 and 0x15, their data provenance and ordering, or conversion-before-arithmetic at 0x3a.

**[INFERENCE]** Repeated offsets suggest structured-state manipulation. Character-like constants suggest possible text-related intent. Neither interpretation imposes application-level acceptance requirements.

## Evidence

The sole behavioral evidence is the supplied static representation of F-0a8fb36bcfa9. Prior prose, comments, and reviewer feedback are not independent proof.

The representation supports the argument forwarding, mixed-width accesses, retained-result relationships, execution-time captures, and ordering boundaries described here. It specifically distinguishes the conditional computed store reaching 0x54 from the later explicit word store, establishes the explicit 0x15 assignment after two computed byte stores, and places character conversion before multiplication and addition for 0x3a.

No dependency implementations, validated calling convention, memory map, or runtime observations are supplied. No runtime tests have been performed.
