<!-- Trusted provenance: F-c3e0e7187deb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c3e0e7187deb

## Purpose

F-c3e0e7187deb performs an ordered sequence of indirect invocations, then mutates shared memory through computed and absolute addresses, and finally performs an unresolved indirect transfer.

The supplied static representation establishes call relationships, arithmetic transformations, mixed-width accesses, and intermediate writes. It does not establish an application-level purpose. Text formatting, object initialization, collection maintenance, and device communication are not supported interpretations.

This contract distinguishes:

- **Established static evidence:** operations, access widths, and ordering visible in the supplied representation.
- **Conditional interpretation:** consequences that depend on a stated machine-width or memory-model assumption.
- **documented_only:** behavior not resolved by the available material.

Observable behavior includes intermediate accesses, not only final memory contents. The contract does not authorize eliminating repeated stores or calls whose results are unused.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters and no declared result. Execution nevertheless consumes implicit inputs:

- An incoming 32-bit register-carried word.
- An incoming link-related value used to select the first indirect target.
- Additional register-carried outputs from dependencies.
- Word-sized and wide dependency results.
- Existing memory containing bytes, words, and address-like values.

The first invocation receives two equal arguments formed by shifting the incoming word left seven bits. Subsequent invocations use retained results, additional register outputs, constants, masks, and shifts.

There is no established success code, error code, returned object, or caller-owned output buffer. Results observable outside the operation are:

1. Dependency invocations and their represented arguments.
2. Effects performed by those dependencies.
3. Direct memory reads and writes.
4. The final four-argument indirect transfer.

A return is represented after the final transfer. Whether that transfer actually returns is **documented_only**.

### Widths and memory layout

| Quantity or access | Represented width | Contractual significance |
|---|---:|---|
| Explicit byte read | 8 bits | Unsigned where a byte value is widened |
| Byte store | 8 bits | Supplies the source value’s low eight bits |
| Halfword store | 16 bits | Supplies a complete 16-bit value |
| Word read or store | 32 bits | Used for numbers and address-like values |
| Wide result container | 64 bits | Supports low/high-word extraction and wider shifts |

A byte store does not itself write neighboring bytes. It neither stores the source’s upper 24 bits nor establishes any relationship between those bits and neighboring memory.

Offsets in the behavioral descriptions below are decimal unless prefixed with `0x`. They are byte displacements. Observed offsets include 1, 4, 5, 8, 9, 13, 16, 17, 20, 21, 25, 32, 36, 52, 64, 68, 84, 100, and 116.

These accesses do not establish structure definitions, object boundaries, or disjoint allocations. Computed addresses may overlap each other or absolute locations. A later read therefore uses memory as it exists at that access, not an assumed earlier snapshot.

Endianness, alignment requirements, address-space validity, and the mapping of word-sized values to executable or data addresses remain unresolved.

### Preconditions and errors

Reaching the represented end requires:

- Every reached indirect target to be executable under the actual calling convention.
- Every dependency needed for continuation to return usable result state.
- Every reached data access to be supported by the runtime memory map.
- Required implicit register values to exist where consumed.
- Arithmetic and address calculations to have meaningful executable semantics.

No visible validation enforces these conditions. No explicit error-reporting, recovery, or rollback path is established. A fault or nonreturning dependency may leave earlier effects intact.

## Behavior

### Indirect-invocation phase

The preparation phase is a long, straight-line sequence of indirect invocations. No explicit conditional branch or loop is represented in that phase.

The initial target depends on the incoming link-related value plus 4094. Later targets are formed from embedded code-associated locations plus the same displacement. Similar target construction does not prove that the invocations share an implementation.

The beginning establishes these relationships:

- The first invocation receives the incoming word shifted left seven bits in both argument positions.
- Two subsequent invocations have no represented arguments; the latter supplies a wide result.
- The next invocation receives that result’s low word shifted left eight bits, its unchanged high word, and its low word shifted left twelve bits.

Later argument families include equal pairs, words shifted left eight bits, unchanged high words, differently shifted versions of the same result, explicit zeros, and retained values from substantially earlier invocations. Some invocations have four represented arguments.

Masks select fields of several widths, including 11 through 22 bits and 24 bits. Masking and shifting are distinct operations: an unmasked transformation cannot replace a masked one merely because some inputs produce the same value.

Calls with no represented arguments remain observable. Calls whose results are unused also remain observable. Neither category is established as a no-op.

Several explicitly zero-initialized retained values undergo shifts without a represented reassignment. Under the conditional word interpretation below, those values remain zero. Dependencies receiving them must still be invoked.

Dependency effects precede the direct-memory phase. No assumption is made that these invocations preserve globals, preserve other memory, or create the storage subsequently accessed.

### Arithmetic interpretation

The representation mixes signed-looking and unsigned-looking word expressions. Signed-looking left shifts and multiplications may exceed a signed 32-bit range. A portable language-level overflow contract is not established.

Under a **conditional 32-bit machine-word interpretation**:

- Word left shifts discard bits above bit 31.
- Word multiplication and address arithmetic retain their low 32 bits.
- Unsigned right shifts introduce zeros.
- Narrow stores truncate to their access width.
- Explicit wide unsigned shifts occur before narrowing.

Consequences include:

- A left shift followed by a right shift need not recover the original value.
- A wide unsigned shift by 34 followed by word narrowing extracts the high word shifted right two bits.
- Wide shifts by 36 and 44 similarly use the high word shifted right four and twelve bits.
- Successive word left shifts with individual counts below 32 and a combined count of at least 32 eliminate all original bits.
- Scaling by two, four, eight, 256, or \(2^{25}\) is arithmetic, not evidence of valid array indexing or allocation size.

The final variable right shift uses a count derived from a retained value’s low byte. Counts can therefore range from 0 through 255. Semantics for counts of 32 or greater are **documented_only**; masking, zero-result, and fault behavior must not be invented.

### Entry into direct memory mutation

The final wide preparation result supplies two address-related components. The first direct effects have this order:

1. A retained shifted word is stored at offset 20 from the high-word-derived base.
2. The low byte of that high word is stored at offset 9 from another retained shifted base.
3. An earlier result scaled by \(2^{25}\) is stored as a word at offset 84 from the final low word scaled by 256.
4. The offset-9 byte is overwritten with the low byte of the retained shifted word.

The overwritten byte store remains an observable event.

The \(2^{25}\)-scaled base then has its offset-20 word cleared. Its offset-5 byte is read. A separate small scaled base has its offset-20 word cleared before the captured byte is stored, zero-extended, at offset 52 from the larger base.

The small base originates from a masked shifted word: under the conditional word interpretation, the masked extraction is at most 508, and doubling it produces a base at most 1016. This does not establish accessibility of any resulting address.

### Interlinked accesses and absolute-memory effects

The following region combines retained numeric values, pointer-like reads, scaled destinations, and repeated stores. Its absolute accesses are part of the behavior, not incidental references to “small addresses.”

**First consecutive-byte pair and address `0x01`.** After a word is read at offset 20 from the previously cleared larger base, the small doubled base’s offset-20 word is cleared again. Two consecutive byte stores write the low byte of the captured word to offset 21 from that small doubled base.

The global byte at `0x05` is then read and widened unsigned. After that read, a byte store at absolute address `0x01` supplies the low byte of the small doubled base. The captured `0x05` byte subsequently supplies an address base for an offset-21 byte store and offset-84 word accesses. The `0x01` store does not cause that already captured byte to be reread.

Other interleaved effects include an offset-17 byte write followed later by a zero overwrite, a word store through the captured offset-20 value plus 84, and an offset-84 read through the global-byte-derived base before that location is overwritten.

**Linked and self-valued writes.** Word reads at offset 116 select later destinations. A captured offset-84 word is stored at offset 36 from its own numeric address, with this self-valued write repeated. Related byte writes at offset 9 are also repeated.

One represented expression reads the same offset-116 location twice to form a self-valued offset-84 store. It does not establish that only one read occurs, nor independently resolve the relative evaluation order of those two reads. Storage with read side effects cannot automatically be modeled as a single capture.

**Absolute word store at `0x54`.** The small doubled base’s offset-20 word first receives an earlier result shifted left 26 bits. A word store at absolute address `0x54` then supplies twice the captured offset-84 word. Afterward, the small base’s offset-20 location receives the captured word itself, twice.

Thus the `0x54` store is between the first and subsequent offset-20 writes. If their destinations alias, final-state reasoning must respect this order.

**Second consecutive-byte pair and address `0x15`.** A later pair of consecutive byte stores again targets offset 21 from the small doubled base, now using the low byte of a factor-of-four value. After both stores, absolute address `0x15` receives the low byte of the small doubled base.

The region continues with related factors of two, four, and eight, offset-68 and offset-84 word writes, byte-selected offset-116 reads, and additional self-valued offset-20 writes. Repeated reads of an offset-5 byte remain separate accesses; earlier computed writes may affect their values.

### Constant stores and repeated-store region

A later selected word supplies three destinations for the numeric constant 58:

- A byte store at four times that word.
- A halfword store at eight times that word.
- A word store at offset 32 from four times that word.

These are distinct widths and destinations.

A retained factor-of-four base is then adjusted downward by 150. Four times this adjusted value supplies another base. At offset 32 from that base, the ordered word-store values are 32, 47, an unsigned byte widened to a word, and 32. The byte comes from offset 4 of the same base and is read after the stores of 32 and 47.

The global word supplying the repeated-store base is captured **before** the absolute word store of 32 at `0x60`. The repeated-store region uses this captured base. It is not a post-`0x60` read. Even if the absolute store changes the global’s storage, there is no represented reload at this boundary.

Following the `0x60` store, an offset-84 word store through an earlier factor-of-eight destination supplies four times the selected word. The long repeated region then interleaves:

- A byte at offset 25 from the captured global base, containing the low byte of the adjusted factor-of-four base.
- A byte at offset 17 from that adjusted factor-of-four base, containing the low byte of the captured global base.
- A word at offset 52 from the captured global base.
- A word at offset 84 from the adjusted factor-of-eight base.

The first offset-52 word uses the earlier, unadjusted factor-of-eight value. Subsequent groups use the adjusted factor-of-eight value. The final visible repetition ends after the offset-52 store and lacks the following offset-84 store.

Exact repetition cardinality is **documented_only** in this contract. This limitation does not permit replacing the region with one set of final values.

### Final captures, absolute bytes, and transfer

For this section, “working base” means the retained adjusted factor-of-four base; “linked doubled base” means the retained doubled word obtained from the later byte-selected offset-116 read. These are descriptive roles, not validated object types.

**Capture boundary.** After the repeated stores, the operation reads a word at absolute address `0xA9`, then a word at offset 116 from the retained four-times-selected-word base. Next, it stores a byte at absolute address `0x3A`.

Under the conditional word interpretation, the `0x3A` byte is the low eight bits of eight times the adjusted value plus 48. This store precedes the captures at offset 20 from the linked doubled base and offset 4 from the working base. Those captures must not be moved ahead of the absolute store without establishing that the move preserves observable behavior.

The captured offset-20 word is retained for the final transfer. The captured offset-4 word is later stored at offset 116 from the captured global base.

**Global-byte propagation.** Following several computed byte stores and a self-valued offset-84 word store through the linked doubled base, the low byte of the captured `0xA9` word is stored at `0x67`.

The operation then reads the word at offset 68 from the linked doubled base. After that read, it reads the byte at `0x67` and copies it to absolute address `0x21`.

Later, the byte at `0x67` is read twice more, each time supplying a byte store at offset 9 from the working base. These are later reads, not unconditional reuse of the original `0xA9` low byte. Computed writes between the initial `0x67` store and those reads may alias `0x67`.

**Remaining linked mutations.** The final region also includes:

- A byte store of 108 at working-base offset 8.
- Byte copies of retained address-like values.
- A byte store at linked-doubled-base offset 1 using the variable right shift.
- Two ordered word writes at captured-global-base offset 84: first a word read through linked-doubled-base offset 68, then the working base itself.
- A working-base offset-64 word read followed by a word copy to offset 16.
- Word writes at offsets 68, 84, and 116 using captured values and later pointer-like reads.
- Restoration of the earlier captured offset-20 word to linked-doubled-base offset 84.
- A later working-base offset-20 read whose low byte supplies a different destination.

Near the end, a word read at offset 4 from the linked doubled base selects another base. A halfword store of 101 targets twice that word. Subsequent reads through offsets 100 and 116 are separated by computed byte and word writes, including a word store of 100.

The final mutations store a captured offset-100 word at offset 84 through another offset-100-derived destination, then store that destination word at twice the captured word’s numeric value.

The unresolved transfer receives, in order:

1. The constant 114.
2. The retained word read at offset 100 from the base selected by the late offset-4 read.
3. The earlier captured linked-doubled-base offset-20 word.
4. The later word read at offset 116 from that late-selected base.

## Side effects

The operation is not established as pure, atomic, transactional, idempotent, or reentrant.

Direct effects include byte, halfword, and word accesses; repeated stores; overwrites; address-dependent mutations; and absolute-memory effects at `0x01`, `0x15`, `0x21`, `0x3A`, `0x54`, `0x60`, and `0x67`. Absolute reads include the byte at `0x05` and word at `0xA9`.

The represented straight-line ordering matters wherever accesses alias or storage exposes access events. This does not independently establish hardware barriers, concurrency guarantees, or device-memory semantics.

No allocation, release, retain, ownership transfer, or lifetime-management contract is established. An address-like dependency result does not prove allocation. A self-valued store does not prove ownership, reference counting, or collection membership.

Earlier effects may survive a later fault or nonreturning transfer. No restoration path is visible.

## Dependencies

F-c3e0e7187deb is the only supplied opaque function identity. No opaque IDs are supplied for the indirect targets; assigning identities would invent evidence.

Dependency boundaries comprise the initial implicit-state-selected invocation, the subsequent location-relative invocation sequence, and the final unresolved transfer. These descriptions do not imply three shared implementations.

Dependency identities, actual signatures, additional register results, memory effects, failure behavior, ownership rules, and global preservation are **documented_only**. In particular, dependencies must not be assumed to preserve globals or release memory.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Initial arguments

Given an incoming word of 1 and conditional 32-bit shift semantics, the first invocation receives 128 in both represented argument positions.

### Byte width

Given a byte-store source of `0x12345678`, the supplied byte is `0x78`. The access itself does not write neighboring bytes or store the source’s upper 24 bits.

### Unsigned widening

If the larger scaled base’s offset-5 byte read yields `0xE1`, its corresponding offset-52 word store supplies `0x000000E1`, not a sign-extended value.

### Wide extraction

Given a wide result with high word `0x80000000`, unsigned shifting by 34 and narrowing to a word yields `0x20000000`.

### Captured repeated-store base

If the global-base capture yields an accessible address, and the later `0x60` store changes that global’s storage through aliasing, the repeated region still uses the captured address. Re-reading the global after the absolute store would not match the represented capture boundary.

### Absolute access order

A trace must place the `0x05` byte read before the `0x01` byte store. It must also place the absolute `0x54` word store between the specified offset-20 writes, and the `0x15` byte store after the later consecutive offset-21 pair.

### Final capture order

If `0x3A` aliases either later capture location, that capture observes memory after the `0x3A` byte store, subject to platform semantics. Moving both captures before the store is not justified.

### Fresh global-byte reads

If `0x67` initially receives `0x78`, but an intervening computed write changes it to `0xA5` before a later represented read, that read supplies `0xA5` under ordinary byte-memory semantics. Reusing `0x78` would incorrectly suppress the read.

### Intermediate constant writes

Traceable storage exposes the offset-32 word values 32, 47, the widened byte result, and 32 in that order. Producing only the final 32 is insufficient.

## Uncertainties

The following remain **documented_only**:

- Application purpose and semantic meaning of constants.
- Complete calling convention and implicit register interface.
- Dependency identities and executable behavior.
- Whether wide results are genuine 64-bit returns or paired register outputs.
- Signed-looking overflow and shift semantics.
- Large variable-shift counts.
- Endianness, alignment, memory-map validity, and storage volatility.
- Object boundaries, aliasing restrictions, ownership, and lifetimes.
- Exact repeated-store cardinality in this contract.
- Evaluation order of duplicated reads within the noted compound expression.
- Whether the final transfer returns.
- Completeness and accuracy of the supplied representation.

These limitations prevent a fully executable replacement specification. They do not justify deleting calls, collapsing intermediate writes, inventing validation, or substituting cached values for represented later reads.

## Evidence

This contract is based on the supplied static representation associated with F-c3e0e7187deb, not on the prior draft or reviewer assertions as independent authority.

The representation supports the implicit-input interface, indirect invocation relationships, mixed-width accesses, arithmetic expressions, absolute-memory transitions, corrected capture boundaries, later `0x67` reads, and final four-argument transfer.

Word-wrap consequences and arithmetic examples are conditional interpretations. Application intent and dependency semantics remain unknown.

No tools, independent implementation access, runtime execution, smoke runs, or tests were used. Acceptance examples state conditional obligations, not observed results.
