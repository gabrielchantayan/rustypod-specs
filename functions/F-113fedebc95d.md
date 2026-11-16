<!-- Trusted provenance: F-113fedebc95d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-113fedebc95d

## Purpose

F-113fedebc95d performs an ordered sequence of indirect invocations, derives numerical fields and addresses from their outputs, updates memory through mixed-width accesses, and finishes with an unresolved indirect transfer.

**Established static evidence:** the supplied representation establishes explicit argument transformations, retained-value dependencies, specified access widths, and the ordering of represented invocations and memory effects.

**Inferred intent:** no application-level purpose is established. Repeated offsets, pointer-like chains, and character-like constants do not identify an object model, traversal algorithm, text format, or protocol.

**documented_only:** indirect-target identities, complete calling conventions, some arithmetic semantics, and environmental requirements remain unresolved. This is a static behavioral contract; no runtime tests have been performed.

## Interface

### Explicit inputs and outputs

The represented interface has three 32-bit input slots and no explicit result value.

| Input | Width | Established use |
|---|---:|---|
| First input | 32 bits | No explicit body reference. |
| Second input | 32 bits | No explicit body reference. |
| Third input | Unsigned 32 bits | Bits 10–19 supply an early invocation argument, scaled by 32. |

The third-input-derived argument ranges from 0 through 32,736 in multiples of 32. Other bits do not affect that argument. This does not establish whole-invocation equivalence: unresolved interfaces may consume additional incoming machine state.

Observable outputs consist of indirect invocations, memory accesses, and the terminal transfer. No explicit status, error object, returned allocation, or success indicator is established.

### Implicit inputs and preconditions

Execution also consumes incoming control-link state, an incoming general-purpose machine word, ordinary and combined invocation results, secondary machine-word outputs, and memory reached through calculated addresses.

Fixed-address inputs include storage at 0x05, a word at 0x087FBBBC, a word at 0xA9, and three separately represented byte reads at 0x67. These inputs are read at their represented positions, not necessarily from entry-state memory. The original read width of storage at 0x05 is unresolved; its value is converted to an unsigned 32-bit quantity.

Every reached address must support its specified access width and alignment, or the environment must define unsupported-access behavior. Indirect targets must execute compatibly with the actual machine interface and provide the outputs subsequently consumed. No visible checks establish these conditions.

Null, low, overlapping, or otherwise unsuitable addresses receive no explicit special handling. A return is represented after the terminal transfer, but whether that transfer returns remains **documented_only**.

## Behavior

### Invocation phase and numerical widths

The initial phase contains many ordered indirect invocations, without an explicit conditional branch, loop, or retry. The first target depends on incoming control-link state plus 0xFFE. Subsequent targets depend on embedded control-location values plus the same displacement. Target identities and possible equivalence are unresolved.

Explicit arguments include zeros, shifted results, low and high halves of combined results, masked fields, transformed incoming machine state, and secondary outputs. Some values remain dependencies across intervening invocations. An invocation displayed without explicit arguments does not establish absence of hidden inputs.

An unused ordinary result does not make an invocation removable. Its target may modify memory, change hidden state, affect storage lifetimes, fail, or prevent continuation. Global preservation is not established.

The representation distinguishes 8-bit bytes, 16-bit halfwords, 32-bit words, and combined 64-bit results. For unsigned combined results, the low half is bits 0–31 and the high half is bits 32–63. Logical whole-result right shifts by 34, 36, and 44 correspond to high-half logical right shifts by 2, 4, and 12. This does not establish memory endianness.

Masks retain fields of widths including 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, and 11 bits. Their application meanings are unknown.

Unsigned 32-bit left shifts discard bits above bit 31; later right shifts cannot recover them. Signed shifts, signed multiplication, and address arithmetic also occur. Signed overflow, exceptional shift counts, and address wrapping remain **documented_only**. Bit-vector examples below are conditional interpretations, not guarantees of defined signed-language behavior.

A late small field is obtained from a returned word shifted left by eight, masked to 15 bits, then logically shifted right by six. Under the represented 32-bit interpretation, it ranges from 0 through 508 in multiples of four.

### Descriptive values and bases

These labels describe dependencies, not declared objects or source identifiers.

| Label | Meaning |
|---|---|
| Shifted-call address | A retained call-derived word shifted left by 25. |
| Multiplied-call base | A different retained call-derived word multiplied by \(2^{25}\). |
| Doubled-field base | Twice the small field; range 0–1,016 in multiples of eight. |
| Early fixed-input base | Unsigned 32-bit conversion of storage read at 0x05. |
| Cleared-word readback | Word reread at multiplied-call base plus 0x14. |
| Early retained word | Word subsequently read at early fixed-input base plus 0x54. |
| Unadjusted scaled base | Four times the early retained word. |
| Adjusted quantity | Unadjusted scaled base minus 0x96. |
| Adjusted base | Four times the adjusted quantity. |
| Late fixed-input base | Word read at 0x087FBBBC. |
| Doubled-chain base | Twice a later chain-derived word. |
| Quadrupled-chain base | Four times that same chain-derived word. |

These bases need not denote disjoint storage or valid object boundaries. Arithmetic qualifications apply throughout.

### Initial accesses and zero readback

The final combined invocation result supplies a high-half base and a low-half-derived address. Initial effects include, in order:

- A word write at the high-half base plus 0x14, storing the shifted-call address.
- A byte write at shifted-call address plus 9, storing the narrowed high half.
- A word write at the low half multiplied by 256 plus 0x54, storing the multiplied-call base.
- Another byte write at shifted-call address plus 9, storing the narrowed shifted-call address.

Different value dependencies do not imply different narrowed bytes.

The multiplied-call base plus 0x14 is then cleared as a word. Before its reread, the represented accesses are, in order: an unsigned byte read at multiplied-call base plus 5; a word clear at doubled-field base plus 0x14; and a word write at multiplied-call base plus 0x34, storing the widened byte. The next access rereads multiplied-call base plus 0x14.

**For ordinary byte-addressable storage, with completed accesses and no external mutation, the cleared-word readback is zero.** The plus-0x34 word is separated from the plus-0x14 word and does not overlap it. The other intervening write stores only zero bits. Even if that clear overlaps the original cleared word, it cannot introduce nonzero bits. The byte read does not modify ordinary storage. No indirect invocation intervenes.

A nonzero readback therefore cannot be justified by aliasing among these ordinary accesses alone. Remaining uncertainty requires unresolved external mutation or nonordinary device, mapping, or access semantics.

After the reread, another word clear at doubled-field base plus 0x14 precedes two byte writes at that base plus 0x15. Both byte writes use the narrowed retained readback. Under the ordinary-storage conditions above, both write zero. Later memory changes do not retroactively change this retained value.

### Early fixed input and fixed-address effects

The read at 0x05 follows those duplicate byte writes and precedes the byte write at 0x01. Its converted value becomes the early fixed-input base. Address 0x01 receives the narrowed doubled-field base; early fixed-input base plus 0x15 receives the narrowed small field.

Before reading early fixed-input base plus 0x54, the routine writes a narrowed earlier call-derived value at multiplied-call base plus 0x11, writes that call-derived value as a word at cleared-word readback plus 0x54, and clears the offset-0x11 byte.

Under ordinary-storage zero readback, that intermediate word destination is fixed address 0x54. Its value is still the earlier call-derived word, not necessarily zero. Thus zero readback does not imply that all later reached storage contains zero.

The word loaded from early fixed-input base plus 0x54 becomes the early retained word. The same destination is subsequently overwritten with the cleared-word readback—zero under the stated ordinary-storage conditions. The retained early word remains the value obtained before that overwrite. Intervening writes and aliases must be considered when determining what its read observes.

Later, the routine performs this ordered cluster:

- A word write at doubled-field base plus 0x14, using an earlier call-derived word shifted left by 26.
- A word write at fixed address 0x54, storing the early retained word shifted left by one.
- Two word writes at doubled-field base plus 0x14, each storing the unshifted early retained word.

The fixed-address shifted-word write is distinct from the earlier readback-derived destination at 0x54.

Two later byte writes at doubled-field base plus 0x15 store the narrowed unadjusted scaled base. Immediately afterward, fixed address 0x15 receives the narrowed doubled-field base. The following word write at unadjusted scaled base plus 0x54 uses an earlier retained call-derived word shifted left by eight.

### Pointer-like chains and layouts

Further accesses follow words at offsets 4, 0x14, 0x44, 0x54, 0x64, and 0x74. Some chains use an unsigned byte read at offset 5 as the base for a word read at offset 0x74. No range, lifetime, or null checks are represented.

| Relative offset | Represented access types |
|---|---|
| 1 | Byte write |
| 4 | Byte and word reads |
| 5 | Unsigned byte reads |
| 8, 9, 0x0D, 0x11, 0x15, 0x19 | Byte writes |
| 0x10 | Byte and word writes |
| 0x14 | Word reads and writes |
| 0x20, 0x24, 0x34 | Word writes |
| 0x40 | Word read |
| 0x44, 0x54 | Word reads and writes |
| 0x64 | Word reads |
| 0x74 | Word reads and a word write |

The table combines different bases; it does not define one structure.

This region includes repeated self-value writes, repeated byte writes, and later rereads of pointer-like sources. Their multiplicity matters where writes affect subsequent loaded addresses or narrow accesses overlap wider storage.

Separate scaled-address effects store 0x3A as a byte at the quadrupled-chain base, as a halfword at twice that base, and as a word at that base plus 0x20.

### Four overwrites and repeated updates

Adjusted base plus 0x20 receives four ordered word writes: 0x20, 0x2F, a zero-extended byte read from adjusted base plus 4, and 0x20 again. The byte read occurs after the first two writes and before the third; it is not an entry-state snapshot.

After all four writes, a word read at 0x087FBBBC supplies the late fixed-input base. That read precedes the word write of 0x20 at fixed address 0x60 and the repeated-update region. The base remains retained even if later writes overlap its original storage.

Each repeated group has this observable order: a word write at an eightfold-scaled quantity plus 0x54 stores the quadrupled-chain base; a byte write at late fixed-input base plus 0x19 stores the narrowed adjusted base; a byte write at adjusted base plus 0x11 stores the narrowed late fixed-input base; and a word write at late fixed-input base plus 0x34 stores the eightfold-scaled quantity.

The leading group scales the early retained word. Subsequent groups scale the adjusted quantity. The represented multiplicity must be preserved; these are repeated accesses using retained bases, not an established element-by-element traversal.

### Terminal retained values and three reads at 0x67

After the repeated groups, the routine retains the word at 0xA9 and the word at quadrupled-chain base plus 0x74. Fixed address 0x3A receives a byte equivalent, under byte-width bit-vector interpretation, to adjusted quantity multiplied by eight plus 0x30.

Words at doubled-chain base plus 0x14 and adjusted base plus 4 are retained. Subsequent writes include the narrowed adjusted base at doubled-chain base plus 0x0D, 0x6C at adjusted base plus 8, the narrowed late fixed-input base at adjusted offsets 0x0D and 9, and the doubled-chain base as a word at its own offset 0x54.

Fixed address 0x67 receives the narrowed retained 0xA9 word. Three later byte reads are separate dependencies:

**First read:** after a word read at doubled-chain base plus 0x44, the byte read at 0x67 supplies the byte write at fixed address 0x21.

Before the second read, the loaded offset-0x44 word supplies a byte write at adjusted base plus 0x10 and a word write at late fixed-input base plus 0x54. A variable-shift result then supplies a byte write at doubled-chain base plus 1. Only afterward does a second word write at late fixed-input base plus 0x54 store the adjusted base.

The variable right shift uses the loaded offset-0x44 word as operand and the low eight bits of the adjusted base as count. Negative signed operands and counts outside the valid word-width range remain **documented_only**. Potential aliasing requires retaining its byte write between the two surrounding word writes.

Next, adjusted base plus 0x40 is read. The earlier retained adjusted-offset-4 word is written at late fixed-input base plus 0x74; the newly loaded offset-0x40 word is written at adjusted base plus 0x10; and the doubled-chain base is written at late fixed-input base plus 0x44.

**Second read:** a fresh byte read at 0x67 supplies a byte write at adjusted base plus 9. A byte write at doubled-chain base plus 9 follows. The retained full 0xA9 word is stored at adjusted base plus 0x44, then adjusted base plus 0x54 is read as a word.

**Third read:** another fresh byte read at 0x67 supplies another byte write at adjusted base plus 9.

Intervening writes may overlap 0x67. Unlike the earlier zero-clear interval, this interval contains potentially nonzero writes through retained bases. Caching the first byte is therefore not generally equivalent.

### Remaining effects and terminal transfer

The retained quadrupled-chain offset-0x74 word supplies narrowed values for byte writes at offsets 0x0D and 0x11. The retained doubled-chain offset-0x14 word is restored at doubled-chain base plus 0x54.

The adjusted-offset-0x54 loaded word supplies later offset-9 and offset-0x44 writes. Further chains read offsets 4, 0x64, and 0x74. A halfword at twice a loaded word receives 0x65. A reached offset-0x54 word receives decimal 100. Final effects include a word write at a loaded base plus 0x54 and a word write at twice another loaded value.

The terminal unresolved transfer receives four represented arguments, in order: 0x72; a word recently loaded through an offset-0x64 dependency; the earlier retained doubled-chain offset-0x14 word; and a word recently loaded through an offset-0x74 dependency. No explicit cleanup or rollback follows.

## Side effects

Direct effects include ordered byte, halfword, and word writes, repeated overwrites, low-address accesses, and potentially overlapping narrow and wide views.

Fixed-address writes include bytes at 0x01, 0x15, 0x21, 0x3A, and 0x67, and words at 0x54 and 0x60. Ordinary-storage zero readback also makes the intermediate readback-plus-0x54 destination fixed address 0x54.

Aliasing does not inherently make every read unpredictable: an overlapping zero-clear preserves an already cleared word. Later nonzero overlapping writes can affect later reads, as specified above.

No ownership transfer, allocation, release, retention guarantee, or lifetime guarantee is established. Unknown targets must not be assumed to preserve globals or reachable memory.

No validation, error reporting, rollback, or transaction boundary is represented. Interrupted execution can leave preceding effects visible. Atomicity, synchronization, reentrancy, fault delivery, and recovery remain **documented_only**.

## Dependencies

F-113fedebc95d is the only supplied opaque function ID. No opaque dependency IDs identify its indirect targets; identities are not invented.

| Boundary | Established information | Unresolved information |
|---|---|---|
| Entry invocation | Control-link-derived target and consumed result | Identity, full inputs, ABI, effects |
| Intermediate invocations | Ordered occurrences and explicit transformations | Hidden inputs, preservation, ownership, completion |
| Secondary outputs | Machine-word outputs consumed later | Complete producer and ABI guarantees |
| Terminal transfer | Four explicit arguments beginning with 0x72 | Identity, transfer mechanism, return behavior |

## Acceptance examples

These are conditional static criteria, not executed tests.

- **Input extraction:** third inputs 0, 1,024, and 0x000FFFFF supply the particular derived argument 0, 32, and 32,736.
- **Combined width:** a combined result with high half 0x80000000 yields 0x20000000 after a logical whole-result right shift by 34.
- **Intermediate truncation:** under 32-bit bit-vector arithmetic, two successive left shifts by 28 produce zero; unbounded intermediate arithmetic is not equivalent.
- **Narrowing:** a byte store of 0x12345678 supplies 0x78. An unsigned byte read of 0xFE widens to 254.
- **Zero readback despite overlap:** in ordinary storage without external mutation, an overlapping doubled-field word clear cannot make the cleared multiplied-call-offset-0x14 word nonzero. Its reread and the two following readback-derived byte writes supply zero.
- **Zero-derived destination:** under those same conditions, the intervening word write through readback plus 0x54 targets fixed address 0x54. The subsequent early-base-plus-0x54 overwrite stores zero, while its preceding load remains independently retained.
- **Nonordinary readback:** a nonzero reread requires additional environmental behavior, such as external mutation or device read semantics; ordinary aliasing among the represented intervening accesses is insufficient.
- **Early fixed input:** conversion of the 0x05 input to 0x2000 selects early-base accesses at 0x2015 and 0x2054. The input read precedes the write at 0x01.
- **Later fixed word:** an early retained word of 0x1234 supplies 0x2468 to the later fixed-address 0x54 write, between the specified doubled-field-offset-0x14 writes.
- **Fixed byte:** doubled-field base 0x128 and early retained word 0x1234 make the two scaled-value byte writes supply 0xD0, while fixed address 0x15 receives 0x28.
- **Late fixed input:** a word of 0x3000 read at 0x087FBBBC selects destinations including 0x3019 and 0x3034. Its read follows all four adjusted-offset-0x20 overwrites and precedes the write at 0x60.
- **Interleaving:** the variable-shift byte write remains between the two late-base-plus-0x54 word writes.
- **Retained word:** retaining 0x12345678 from adjusted base plus 4 and obtaining late base 0x3000 supplies that word to the reached store at 0x3074.
- **Distinct byte reads:** if intervening effects make the three 0x67 reads return 0x12, 0x34, and 0x56, the dependent stores supply 0x12 at 0x21, then 0x34 and 0x56 at adjusted base plus 9.
- **Interruption:** a nonreturning intermediate target prevents subsequent effects from being reached; no rollback is established.

## Uncertainties

The following remain **documented_only**:

- Application purpose and completeness of the represented interface.
- Indirect-target identities, hidden inputs, calling conventions, and preservation guarantees.
- Original access width and conversion details of storage at 0x05.
- Signed overflow, exceptional shifts, address wrapping, and endianness.
- Alignment support, permissions, mapping behavior, and fault handling.
- Actual layouts, storage lifetimes, external mutation, and device/access semantics.
- Ownership, allocation, retention, and release behavior.
- Whether the terminal transfer returns.

These uncertainties do not negate established dependencies. In particular, ordinary-storage aliasing alone does not invalidate the zero-readback conclusion.

## Evidence

The supplied static representation establishes the invocation sequence, explicit transformations, access widths, retained values, and described access ordering. Prior prose, feedback, comments, and warning annotations are not independent behavioral proof.

The zero-readback conclusion follows from the represented clear, intervening byte read, overlapping-or-disjoint zero-clear, separated plus-0x34 word write, and reread. It requires ordinary storage semantics and absence of external mutation, not disjointness of the two clears.

The representation also establishes the separate fixed-address 0x54 value dependencies, the doubled-field source of fixed address 0x15, the late fixed-input read position, the interleaved variable-shift byte write, and three distinct reads at 0x67.

No dependency implementations, verified declaration for storage at 0x05, executable traces, or runtime observations are supplied. No tools or runtime tests were used.
