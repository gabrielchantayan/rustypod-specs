<!-- Trusted provenance: F-91a2fecb90e8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-91a2fecb90e8

## Purpose

F-91a2fecb90e8 performs an ordered chain of unresolved indirect invocations, transforms invocation results and implicit entry state, and then performs extensive mixed-width memory mutation before a final unresolved control transfer.

The supplied static representation does not establish an application-level purpose. Character-like constants and pointer-following patterns do not justify identifying the operation as formatting, serialization, initialization, object construction, or resource release.

This contract distinguishes:

- **Established static evidence:** operations, widths, expressions, and ordering represented in the supplied material.
- **Inferred intent:** interpretations that are not acceptance requirements.
- **documented_only:** executable behavior not established by the available evidence.

The contract describes represented access and invocation events. It does not establish that every event is externally observable on the actual machine, or that execution can reach the end. No runtime tests have been performed.

## Interface

### Inputs

The displayed interface has no explicit parameters. It nevertheless consumes implicit execution state, including:

- An incoming continuation-related value selecting the first indirect target.
- Another incoming machine-state value used in early shifted arguments without an established initialization within this function.
- Primary invocation results.
- Secondary machine-result values represented independently of primary results.
- Combined results represented as 64-bit quantities.
- Global and computed-address memory contents.

The complete entry calling convention is **documented_only**. A no-argument interface must not be interpreted as independence from the caller.

### Outputs

No explicit return value, status code, returned object, or output-buffer contract is established.

Represented outputs consist of indirect invocation events, direct memory writes, dependency side effects, and the terminal unresolved invocation or transfer. An apparent return follows that final event in the representation; normal return at the machine level remains **documented_only**.

### Widths and arithmetic

| Operation or quantity | Represented width |
|---|---:|
| Byte access | 8 bits |
| Halfword store | 16 bits |
| Word access | 32 bits |
| Combined invocation result | 64 bits |
| Most address-bearing arithmetic | 32 bits |

Byte and halfword stores retain the low eight and sixteen supplied bits, respectively. Explicit unsigned-byte reads used as integers are zero-extended.

Unsigned 64-bit extraction must occur before narrowing. Right shifts of a full combined result by 34, 36, and 44 bits retain its upper thirty, twenty-eight, and twenty bits, respectively.

Many arithmetic operands are represented as signed 32-bit integers. Overflowing arithmetic, signed left shifts, and oversized shift counts prevent treating this material as a portable arithmetic specification. Statements below involving modulo-\(2^{32}\) arithmetic are conditional on confirmed machine-level bit-vector semantics.

Byte order, alignment rules, address mapping, device-memory behavior, and aliasing are not established.

### Preconditions and ownership

Reaching later events requires executable invocation targets, suitable incoming machine state, valid memory accesses at their represented widths, and dependencies that return when subsequent operations require them. These are execution requirements, not validations performed by the function.

No ownership transfer, allocation, release, reference-count operation, or lifetime guarantee is established. Following or storing a word as an address does not establish that it denotes an owned object. Unknown dependencies must not be assumed to preserve globals or to acquire or release memory.

## Behavior

### Indirect-invocation phase

The initial phase is a long straight-line sequence of indirect invocations, with no explicit loop or conditional branch shown.

The first target is selected using incoming continuation-related state plus displacement 4094. Later targets are represented through fixed-location-derived values plus the same displacement. These expressions do not identify dependency semantics.

Invocations have zero through four explicit arguments. Their data flow includes:

- Repeated use of one shifted result in multiple argument positions.
- Retention of results across intervening invocations.
- Separate use of low and high words of combined results.
- Consumption of secondary machine-result channels.
- Explicit zero arguments and values reset to zero before later transformations.
- Masking, scaling, and successive shifts of retained values.

Calls with ignored primary results remain potentially side-effecting. Similar-looking calls are distinct ordered events, not an established retry or polling mechanism.

No dependency preservation contract is available. Memory later read by this function may already have been changed by an earlier dependency.

### Bit transformations

The represented masks retain fields of 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, and 11 bits, among other narrower selections. Masking is extraction, not input validation.

For example, retaining twenty bits and then shifting right by ten produces an unsigned quantity from 0 through 1023. Successive left shifts do not implicitly widen a 32-bit intermediate: shifting left by 28 and subsequently by another 28 yields zero under 32-bit bit-vector semantics.

A late right shift uses the low eight bits of an address-bearing quantity as its count. Counts exceeding 31 are possible. The supplied material does not establish count masking or the result of an oversized shift.

### Direct-memory address relationships

The following descriptive labels are contract terminology, not recovered object or field names:

| Label | Relationship |
|---|---|
| Retained shifted value | Low word of a late combined result shifted left by 25 |
| Shifted base | Another late invocation result multiplied by \(2^{25}\) |
| Small base | Twice a late result after left shift by eight, retention of fifteen bits, and right shift by six |
| Link word | Word read at offset +0x54 from the zero-extended byte read at absolute address 0x05 |
| Adjusted base | Four times the retained link word, less 150 |
| Scaled adjusted base | Four times the adjusted base |
| Repeated-record base | Word read from a fixed global location before the repeated-store region |

These are arithmetic relationships, not structure boundaries.

Under 32-bit bit-vector semantics, the quantity doubled to form the small base ranges from 0 through 508 in multiples of four. The small base therefore ranges from 0 through 1016 in multiples of eight. It is not established as a heap address.

### Early mutation and overlapping accesses

The direct-memory phase begins with a word store at high-word-derived offset +0x14. It then writes a byte at retained-shifted-value offset +9, stores the shifted base at an address formed by scaling the final combined result’s low word by 256 and adding +0x54, and writes again to the earlier byte destination.

The first byte comes from the combined result’s high word; the second comes from the retained shifted value. Their **provenance differs**, but their low bytes may coincide. Both stores remain distinct represented events even when they supply the same byte.

Subsequent ordering includes:

- Clearing the word at shifted-base offset +0x14 before reading the byte at shifted-base offset +5.
- Clearing the word at small-base offset +0x14 before storing the zero-extended byte at shifted-base offset +0x34.
- Reading the word at shifted-base offset +0x14 after those operations.
- Clearing the small-base word again, followed by two byte stores at small-base offset +0x15 using the low byte of that retained word read.

The +0x15 byte destination lies within the four-byte span beginning at +0x14. Its effect on the word’s numeric value depends on byte order.

The word read after a clear cannot unconditionally be replaced by zero: intervening computed-address stores may alias it. More generally, every represented read uses memory at its own access point, not an assumed earlier snapshot.

### Low-address and global-access chronology

The following ordering is established independently of whether computed addresses alias these locations:

| Region | Established ordering |
|---|---|
| Early low-address access | Byte read at 0x05 precedes byte write at 0x01 |
| Link-derived mutation | Word write at 0x54 precedes byte write at 0x15 |
| Repeated-region setup | Fixed-global word read supplying the repeated-record base precedes word write of 0x20 at 0x60 |
| Final-region setup | Word read at 0xa9 precedes byte write at 0x3a |
| Mutable-byte propagation | Byte write at 0x67 precedes a read there for the byte write at 0x21, followed by two later reads at 0x67 for computed-address stores |

The fixed-global read occurs **before**, not after, the write at 0x60 in the supplied representation. Requiring the reverse order would contradict the available static evidence.

The byte read at 0x05 is retained and zero-extended for later address formation. The subsequent write at 0x01 supplies the low byte of the small base. Later, a word derived from the link word is written at 0x54 before the low byte of the small base is written at 0x15.

The word retained from 0xa9 is read before the write at 0x3a. That byte write supplies the low eight bits of eight times the adjusted base plus 0x30.

### Pointer following and access layout

The direct-memory phase follows word contents and uses zero-extended bytes as address bases. Examples include following words at +0x74, reading a byte at a derived base plus +5, and using that byte to locate another word at +0x74.

Represented offsets include:

| Offset | Access forms |
|---|---|
| +1 | Byte store |
| +4 | Byte or word read |
| +5 | Byte read |
| +8, +9, +0xd | Byte stores |
| +0x10 | Byte and word stores |
| +0x11 | Byte store |
| +0x14 | Word read or store |
| +0x15, +0x19 | Byte stores |
| +0x20, +0x24, +0x34 | Word stores |
| +0x40 | Word read |
| +0x44, +0x54, +0x74 | Word reads or stores |
| +0x64 | Word read |

This inventory does not establish a common structure type across bases.

Duplicate writes remain separate events. A nested expression containing two reads of the same pointer-bearing location does not establish one cached read; the precise machine-level ordering within such an expression is **documented_only** where the representation does not resolve it.

Before the repeated region, a word destination at scaled-adjusted-base offset +0x20 receives, in order, 0x20, 0x2f, a zero-extended byte read from offset +4 of that base, and 0x20 again. The byte read occurs after the first two writes and before the byte-derived word store. A final-memory-only description loses this relationship.

Other stores supply 0x3a at byte, halfword, and word widths to differently scaled destinations.

### Repeated-store region

The supplied material contains a long explicitly repeated store region, not an explicit loop.

The first group and subsequent groups share the order:

1. A word store at a scaled destination plus +0x54.
2. A byte store at repeated-record-base offset +0x19, supplied from the scaled adjusted base.
3. A byte store at scaled-adjusted-base offset +0x11, supplied from the repeated-record base.
4. A word store at repeated-record-base offset +0x34.

The first group’s first destination uses eight times the link word; subsequent groups use eight times the adjusted base. Likewise, the first group’s final supplied word uses eight times the link word, whereas subsequent groups use eight times the adjusted base.

These are differences in expressions and provenance, not guaranteed numerical inequality. The first and later expressions may produce equal addresses and equal supplied values.

The repeated-record base is retained from the fixed-global read. The groups do not represent repeated reads of that global. This does not imply that the global’s memory contents remain unchanged.

Exact group cardinality is **documented_only** in this contract. No guessed count is an acceptance requirement. The distinct represented stores must not otherwise be collapsed merely because they appear redundant in ordinary, nonaliasing RAM.

### Final-region mutable-byte reads

The retained word read at 0xa9 has two distinct roles: its low byte initially supplies the store at 0x67, and the retained full word later supplies a computed-address word store.

Propagation through 0x67 is not propagation from an immutable retained byte:

- After storing the retained word’s low byte at 0x67, the function reads a word at a computed base plus +0x44.
- It then reads the current byte at 0x67 and supplies that byte to the write at 0x21.
- Several computed-address writes follow, including a byte write at scaled-adjusted-base offset +0x10, word writes through the repeated-record base, and a variable-shift-derived byte write through another base.
- A word read at scaled-adjusted-base offset +0x40 supplies a later word store at +0x10, after the earlier byte store at that starting offset.
- Further record writes occur before another read of 0x67 supplies a byte store at scaled-adjusted-base offset +9.
- Additional computed-address byte and word writes, followed by a word read at scaled-adjusted-base offset +0x54, precede the third read of 0x67, again supplying a byte store at offset +9.

Computed-address writes may alias 0x67, including through a word spanning that byte. Therefore, the three reads need not agree with one another or with the initially stored low byte.

The retained full word from 0xa9 is not re-read when later stored elsewhere. Conversely, the two later byte stores explicitly depend on fresh reads of 0x67, not on truncation of that retained full word.

### Terminal operations

The final region also includes a byte store of 0x6c, pointer-following word reads, a halfword store of 0x65 at twice a followed word value, and a word store of decimal 100 at a followed base plus +0x54.

The last direct store writes a word at twice another followed word value. The terminal unresolved invocation then receives four represented arguments:

- Literal 0x72.
- A retained word read at offset +0x64 from a followed base.
- An earlier retained word read at offset +0x14.
- A recently read word at offset +0x74 from the followed base used near the end.

No explicit validation separates the final store from this invocation. Its target, calling convention, effects, and return behavior remain unresolved.

## Side effects

Direct effects include reads and writes through computed addresses and accesses to low absolute addresses:

| Address | Access |
|---|---|
| 0x01 | Byte write |
| 0x05 | Byte read |
| 0x15 | Byte write |
| 0x21 | Byte write |
| 0x3a | Byte write |
| 0x54 | Word write |
| 0x60 | Word write of 0x20 |
| 0x67 | Byte write and three subsequent byte reads |
| 0xa9 | Word read |

Computed accesses may overlap these locations or one another. An access through a computed address is not necessarily independent of a named global.

The function is not transactional. No rollback, bounds checking, null rejection, locking, synchronization, exception handler, or explicit error-cleanup path is represented. Invalid accesses or dependency failures may interrupt execution after earlier effects.

Concurrency safety, reentrancy, idempotence, and device-memory visibility are **documented_only**.

## Dependencies

F-91a2fecb90e8 is the only supplied opaque function ID. The dependency map provides no opaque IDs identifying the invoked targets; none are invented here.

The unresolved dependency surface comprises the entry-state-selected target, the fixed-location-derived targets, and the terminal target.

Unknown contracts include argument interpretation, primary and secondary result channels, combined-result meaning, memory mutation, global preservation, ownership effects, failure behavior, and normal return. No dependency is established as an allocator, reader, formatter, dispatcher, or release operation.

## Acceptance examples

These are conditional static expectations, not executed tests.

### Combined-result width

For unsigned 64-bit result 0x1234567889abcdef, the low word is 0x89abcdef, the high word is 0x12345678, and shifting the full result right by 36 yields 0x01234567. A narrowed 32-bit shift is not interchangeable.

### Equal bytes from different provenance

If the early high-word-derived byte and retained-shifted-value-derived byte both equal zero, acceptance still retains two ordered byte stores to their common destination. A numerical change is not required.

### Equal repeated-group expressions

Under defined 32-bit arithmetic, a retained link word of decimal 50 gives an adjusted base of decimal 50. Both first-group and subsequent-group final word expressions then supply decimal 400. Equal values do not eliminate any store.

### Global ordering

If all accesses complete, the read at 0x05 precedes the write at 0x01, and the write at 0x54 precedes the write at 0x15. The fixed-global read supplying the repeated-record base precedes the write at 0x60.

### Mutable 0x67

Suppose the retained 0xa9 word has low byte 0xab. If an intervening computed-address write changes 0x67 to 0xcd before a later read there, the corresponding later byte store uses 0xcd. Requiring 0xab would incorrectly replace a fresh memory read with a retained value.

### Overlapping clear and byte store

A word clear at base plus +0x14 covers four bytes. A later byte store at +0x15 changes a byte within that span. Acceptance preserves both accesses and uses the actual byte order to interpret any subsequent word read.

### Partial execution and terminal boundary

An inaccessible derived address has no represented fallback or explicit error return. Earlier effects need not be undone. If execution reaches the final invocation, its first represented argument is 0x72; prior stores do not prove that the target returns.

## Uncertainties

The following remain **documented_only**:

- Application purpose and field meanings.
- Complete implicit entry-state requirements.
- Dependency identities, signatures, ownership effects, and preservation guarantees.
- Whether represented combined results reflect genuine 64-bit return contracts.
- Instruction-level overflow, signed-shift, and oversized-shift semantics.
- Byte order, alignment, mapping, aliasing, and external visibility.
- Exact repeated-store cardinality in this contract.
- Machine-level ordering within unresolved multi-read expressions.
- Fault recovery and terminal-transfer return behavior.

The uncertainty about dependencies does not justify treating mutable globals as constants. Likewise, different expressions do not establish different numeric values.

## Evidence

Evidence is limited to the supplied static representation. Established observations include implicit entry-state consumption, ordered indirect invocations, fixed-width transformations, mixed-width memory access, overlapping and repeated stores, low-address global accesses, three fresh reads of 0x67 after its initial write, and the final four-argument unresolved invocation.

The representation places the fixed-global read supplying the repeated-record base before the word write at 0x60. Reverse ordering is not supported by that evidence.

Comments, warning text, the prior draft, and reviewer assertions are not independent proof. No implementation access, dependency inspection, instruction-level validation, or runtime testing has been performed.
