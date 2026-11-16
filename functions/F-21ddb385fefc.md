<!-- Trusted provenance: F-21ddb385fefc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-21ddb385fefc

## Purpose

F-21ddb385fefc forwards an opaque input into an ordered sequence of indirect dependency invocations, transforms dependency results, performs mixed-width memory mutations, and reaches a final unresolved indirect transfer.

No application-level purpose is established. Numeric markers, repeated offsets, and self-referential stores do not establish parsing, formatting, allocation, container management, or device control.

This contract distinguishes three evidence levels:

- **Established static evidence:** behavior explicitly represented in the supplied material.
- **Inferred intent:** interpretations that are not acceptance requirements.
- **documented_only:** unresolved behavior for which this contract supplies no invented guarantee.

Established static evidence is not proof that the representation accurately describes executable behavior. No runtime tests have been performed.

## Interface

### Explicit input

The function accepts one opaque **32-bit value**. Its signedness, valid range, pointer interpretation, and associated object layout are not established.

The first represented invocation passes this value unchanged to F-U01. No preceding input validation or direct input dereference is represented.

### Implicit inputs and preconditions

Execution additionally depends on incoming control state, indirect targets, dependency result channels, existing memory, and address-bearing global values.

Every reached invocation requires a valid executable target. Every reached memory access requires storage valid for its access width and direction. This includes low absolute addresses; an environment with an inaccessible null page is not automatically compatible.

No buffer size, alignment guarantee, ownership convention, or initialization protocol is established. Whether unaligned accesses are supported is **documented_only**.

### Output and completion

No explicit return value is represented.

If the final transfer F-U03 is reached, its four represented arguments are:

| Position | Value |
|---|---|
| First | Numeric constant 0x72 |
| Second | A saved 32-bit word read at offset +0x64 from a late-loaded address-bearing value |
| Third | A saved 32-bit word read at offset +0x14 from an earlier linked base |
| Fourth | A saved 32-bit word subsequently read at offset +0x74 from the same late-loaded base used for the second argument |

The second argument is also used in a final destination calculation. The third argument survives intervening mutations and is not a final reread. The fourth argument is read before the last destination lookup and final stores.

The representation places a return after F-U03. Whether this transfer actually returns, dispatches permanently, terminates, or uses another control-flow mechanism is **documented_only**. No success indication is established.

## Behavior

### Dependency phase

The function first performs a lengthy straight-line sequence of indirect invocations. No explicit conditional branch or loop is represented in this phase.

Invocation order matters. An invocation with an unused result remains potentially observable; an invocation with no explicit arguments may still consume implicit machine state. Dependencies are not assumed pure, interchangeable, or normally returning.

Represented argument patterns include:

- The unchanged explicit input at entry.
- No-explicit-argument invocations.
- A returned word shifted left by eight bits.
- Low and high components of a represented 64-bit result.
- An unchanged high component accompanied by shifted components.
- Duplicate arguments derived from the same saved result.
- Literal zeroes and values established as zero in the representation.
- Saved results reused across intervening invocations.
- Masked fields, sometimes shifted again before forwarding.

Some secondary result-channel values lack an explicit producer assignment. Their provenance and calling-convention meaning are **documented_only**. They must not be substituted with zero or equated to a nearby primary result without additional evidence.

### Widths and arithmetic

Represented words are 32 bits; aggregate results are sometimes 64 bits. Byte stores retain eight bits, and halfword stores retain sixteen bits. Explicit unsigned-byte reads widen without sign extension.

For a 64-bit aggregate, the low component comprises bits 0–31 and the high component bits 32–63. Logical extraction beginning at bit 44 produces at most twenty significant bits, not an unchanged high word.

Explicit masks include widths of 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, and one bit. Masking before shifting is distinct from shifting an unrestricted word.

**Width-based inference:** ordinary word arithmetic appears intended to retain the low 32 result bits. Under this interpretation:

- A left shift by eight discards the original upper eight bits.
- A left shift by 25 retains only the original low seven bits.
- Successive left shifts by 28 and 28 produce zero.
- Successive left shifts by 30 and seven produce zero.
- A later right shift cannot recover bits discarded by an earlier truncated shift.
- Multiplication by 0x02000000 retains the same word-sized pattern as a left shift by 25.

Signed left shifts, signed multiplication overflow, address wraparound, and exceptional shift counts do not acquire portable semantics merely from the representation. Their exact executable treatment remains **documented_only**.

Unsigned right shifts are logical. A late right shift operates on a signed word, with its count derived from the low eight bits of a computed value. Ordinary counts and counts exceeding 31 must not silently be given a particular language’s semantics.

### Memory layout and aliasing

Computed 32-bit values are used as addresses, scaled addresses, and stored numeric values. No coherent object type is established.

| Access width | Represented locations |
|---|---|
| 8 bits | Derived addresses including offsets +1, +4, +5, +8, +9, +0x0D, +0x10, +0x11, +0x15, and +0x19 |
| 16 bits | Scaled addresses without an additional field offset |
| 32 bits | Derived addresses including offsets +4, +0x10, +0x14, +0x20, +0x24, +0x34, +0x40, +0x44, +0x54, +0x64, and +0x74 |

The table describes access shapes, not independent structure members. A byte at +0x15 can overlap a word at +0x14; a byte at +0x11 can overlap a word at +0x10.

Address calculations include doubling, quadrupling, multiplication by eight, shifts, and subtraction of 0x96. Pointer, index, encoded-reference, and representation-artifact interpretations remain unresolved.

Absent a proven memory model, repeated reads are separate observations and repeated writes are separate effects. Final memory equality alone does not justify eliminating them. Endianness is not established, so byte-level consequences of overlapping wider accesses require an explicit environmental assumption.

### Initial mutation boundary

Direct memory mutations begin after the dependency sequence.

The high word of the final aggregate result supplies a base for a word store at +0x14. Another saved derived base receives a byte at +9 containing the narrowed high word. A word store at a destination derived from the aggregate’s low component intervenes before that same +9 byte receives a different saved value.

The next region has two relevant bases: a large scaled base and a separate small doubled base. Their descriptions do not imply non-aliasing.

The ordering requirement is precise: the word at the large base plus 0x14 is cleared before the unsigned byte at that base plus five is read. The first clear at the small base plus 0x14 follows that byte read. The byte’s saved, zero-extended value is then stored as a word at the large base plus 0x34.

The word at the large base plus 0x14 is read back next. Only after this readback does the second clear at the small base plus 0x14 occur. The saved readback’s low byte is subsequently written twice at the small base plus 0x15.

Thus, the two small-base clears **straddle the readback**. Moving both before it, or moving the widened-byte store across either boundary, changes the represented access ordering.

The byte read remains a saved value when widened into the later word store. The word readback likewise remains saved after the second clear. Neither may be replaced solely by reasoning from an earlier clear without establishing the relevant memory behavior.

### Linked and scaled mutations

Subsequent operations use a value obtained from absolute storage at 0x05 as an address-bearing value. The access width of that absolute read is unresolved.

This region includes successive byte overwrites at +0x11, loads and replacements of +0x54 words, word stores at +0x34, and a +0x74 lookup that supplies another destination base. Repeated self-reference stores at +0x24 and repeated byte stores at +9 remain distinct effects.

One assignment uses **two separate word reads from the same +0x74 location**: one supplies the destination base, and the other supplies the stored value. Both are inputs to that one store at the selected base plus 0x54. There is no established store between those two reads, and their relative evaluation order is not established.

Consequently, the assignment is not unconditionally a self-reference. If the reads can return different words, the destination may derive from one word while the payload is the other.

Further processing scales saved values, publishes them through +0x14, +0x34, +0x44, and +0x54 locations, and obtains addresses through unsigned-byte reads. Widening a byte before using it as an address limits that immediate address-bearing value to 0–255; it does not establish that the resulting access is valid.

Reads from apparently identical +5 locations occur at separate points among intervening writes. They are not established to return identical bytes.

### Numeric markers and repeated publication

A later region writes 0x3A as a byte at a quadrupled address, as a halfword at an eightfold address, and as a word at offset +0x20 from the quadrupled address.

After a derived value is adjusted by subtracting 0x96, its quadruple supplies a base used extensively in the remaining processing. At that base plus 0x20, four successive word stores contain 0x20, 0x2F, the zero-extended current byte read at base plus four, and 0x20 again.

A separate address-bearing global word is loaded after this marker sequence and before a word containing 0x20 is written at absolute address 0x60.

The repeated publication region uses that **saved global base**, not a fresh global lookup for every group. Each represented group orders a word store at a scaled destination plus 0x54 before a byte store at global-base plus 0x19, a byte store at adjusted-quadruple-base plus 0x11, and a word store at global-base plus 0x34.

The initial group uses an earlier unadjusted scaled value; subsequent groups use the adjusted scaled value. The stored byte payloads are saved narrowings, not fresh memory reads. Exact repetition multiplicity is **documented_only** here; this does not authorize collapsing repeated effects.

No string length, terminator, encoding, output buffer, or external sink is established by these marker values.

### Late reads, saved values, and low-address rereads

After publication, the function saves the word at absolute address 0xA9 and reads a word at +0x74 from the quadrupled marker-derived base. It then writes a narrowed arithmetic result to absolute byte 0x3A before saving words from the linked base plus 0x14 and the adjusted-quadruple base plus four.

The following mutations include byte payloads at +0x0D and +9, the byte marker 0x6C at adjusted-quadruple-base plus eight, and a self-valued word at linked-base plus 0x54.

The low byte of the saved 0xA9 word is then stored at absolute address **0x67**. A word read from linked-base plus 0x44 intervenes before the copy to absolute address **0x21**. That copy reads the **current byte at 0x67**; it does not reuse the saved low byte from 0xA9.

The intervening +0x44 read supplies later stores and the signed shift narrowed into linked-base plus one. At the saved global base plus 0x54, a store of that loaded word precedes a store of the adjusted-quadruple base. A word read at adjusted-quadruple-base plus 0x40 later supplies a word store at that base plus 0x10. The earlier saved +4 word is separately stored at global-base plus 0x74.

Two later stores at **adjusted-quadruple-base plus nine** each perform a **fresh byte read from absolute 0x67**:

- The first occurs after the +0x40 load and intervening +0x74, +0x10, and +0x44 stores.
- Between the two, a byte is stored at linked-base plus nine, the full saved 0xA9 word is stored at adjusted-quadruple-base plus 0x44, and a word is loaded from adjusted-quadruple-base plus 0x54.
- The second +9 store uses another current read of 0x67 after that +0x54 load.

The current byte at 0x67 need not equal the originally stored low byte at any of these later observations. Derived stores may alias it, and memory reads may have environment-dependent effects. The copy to 0x21 and both later +9 stores must therefore remain distinguishable from saved-byte reuse.

### Final mutation and transfer boundary

The word loaded from adjusted-quadruple-base plus 0x54 becomes another destination base. A saved +0x74 word is narrowed for byte stores at this destination plus 0x0D and linked-base plus 0x11. The earlier saved linked-base +0x14 word is restored at linked-base plus 0x54.

A new word read at adjusted-quadruple-base plus 0x14 supplies a later narrowed byte store. Further +9 and +0x44 stores publish the recently loaded destination base.

A word read from linked-base plus four supplies a late child base. A halfword containing 0x65 is written at twice that child value before the child’s +0x64 word is read and saved.

A byte derived from the new +0x14 read and a word containing decimal 100 are stored through the recently loaded destination base. The child’s +0x74 word is then saved, followed by a distinct word read from linked-base plus 0x64.

Before F-U03, the function writes a narrowed child value at linked-base plus nine, stores the saved child +0x64 word at the last selected base plus 0x54, and stores that selected base as a word at twice the saved child +0x64 value.

These final stores do not refresh the saved transfer arguments.

## Side effects

Direct effects include arbitrary derived-memory reads and writes, byte stores at absolute addresses 0x01, 0x15, 0x21, 0x3A, and 0x67, word stores at 0x54 and 0x60, and reads involving 0x05 and 0xA9.

Absolute 0x67 is both written and later reread. Absolute 0xA9 supplies a saved full word whose later full-width use is distinct from rereading its narrowed copy.

Dependencies may independently change memory, globals, control state, or resource lifetime. No preservation guarantee is assumed.

No direct allocation or release operation is identifiable. This establishes neither the absence of dependency-managed resources nor an ownership transfer. Saved addresses do not imply retained ownership or lifetime extension.

No locking, barrier, atomicity guarantee, rollback, validation branch, error result, retry policy, or cleanup path is represented. Invalid accesses, invalid targets, dependency failure, or exceptional control flow may stop execution after partial effects. Their reporting mechanism is **documented_only**.

## Dependencies

The following opaque IDs are document-local unresolved references:

| ID | Role | Established boundary |
|---|---|---|
| F-U01 | Entry dependency | Receives the explicit 32-bit input unchanged |
| F-U02 | Family of intermediate indirect dependencies | Ordered invocations receiving explicit argument tuples or no explicit arguments |
| F-U03 | Final unresolved transfer | Receives the four values specified in Interface |

F-U02 does not denote one proven callable identity. Similar argument shapes do not establish identical targets.

No dependency is assumed to preserve globals, preserve memory, return normally, allocate, retain, own, or release memory.

## Acceptance examples

These are conditional static obligations, not executed tests.

1. **Input forwarding.** For input 0x89ABCDEF, the first represented dependency receives that exact 32-bit pattern.

2. **Word truncation.** Under 32-bit bit-vector arithmetic, shifting low component 0x12345678 left by eight yields 0x34567800.

3. **Mask significance.** Selecting the low 24 bits of 0xFF123456 before shifting left by six yields 0x048D1580. Shifting the unrestricted word instead yields a different word.

4. **Aggregate extraction.** Logical extraction above bit 44 from 0xABCDE123456789AB yields 0xABCDE, not the entire high word.

5. **Unsigned widening.** An unsigned-byte read of 0xE7 produces word payload 0x000000E7 when widened.

6. **Straddled clears.** An access observer must see the first small-base clear before the widened-byte store, the large-base readback after that store, and the second small-base clear after the readback. Both clears before the readback are not equivalent.

7. **Two-read assignment.** If the two +0x74 reads can return different words, one determines the destination and the other the payload. Acceptance cannot require identical words or invent a store between those reads.

8. **Fresh low-address copy.** If 0x67 contains 0x12 immediately after narrowing the saved absolute word, but contains 0x34 when the subsequent copy reads it, 0x21 receives 0x34.

9. **Independent later rereads.** If the first later +9 store reads 0x56 from 0x67 and the second reads 0x78, their payloads are respectively 0x56 and 0x78. Reusing either the original low byte or the first reread is incorrect.

10. **Marker observations.** Where intermediate writes are observable, the +0x20 word sequence includes 0x20, 0x2F, the widened byte, and 0x20; retaining only the final value loses represented effects.

11. **Partial completion.** A failure after earlier writes establishes no restoration requirement.

12. **Final arguments.** If F-U03 is reached, its first argument is 0x72 and its remaining arguments are the specified saved words, even if later aliasing stores changed their original memory locations.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and intended outcome.
- Individual dependency identities, contracts, and calling convention.
- Secondary result-channel provenance.
- Accuracy of represented indirect control flow.
- Exact repeated-publication multiplicity.
- Memory-map validity, alignment, endianness, volatility, and read side effects.
- Relative evaluation order of the two reads feeding the single +0x74-derived assignment.
- Signed overflow, address wraparound, and exceptional shift semantics.
- Concurrency, external observer, and resource-lifetime guarantees.
- Whether F-U03 returns normally.

Bit packing, reference linking, and marker emission are **inferred intent**, not established application semantics. The explicit late rereads of 0x67 are established; equality of their returned bytes is not.

## Evidence

This contract uses only the supplied static representation: explicit widths, dependency boundaries, masks and shifts, address calculations, ordered memory effects, saved-value reuse, low-address rereads, and final transfer arguments.

Comments, symbolic names, the prior draft, and reviewer feedback were not treated as independent proof. The mutation ordering and reread distinctions described here follow the represented accesses themselves.

No implementation access, dependency inspection, runtime execution, or runtime tests have been performed.
