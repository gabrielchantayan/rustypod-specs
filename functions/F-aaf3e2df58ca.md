<!-- Trusted provenance: F-aaf3e2df58ca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aaf3e2df58ca

## Purpose

F-aaf3e2df58ca performs an ordered series of indirect dependency invocations, transforms returned and ambient machine values, mutates fixed and computed memory locations, and ends with an unresolved indirect transfer.

No domain-level purpose is established. The available representation does not justify describing this function as a parser, formatter, allocator, initializer, or serializer. Repeated offsets and character-like constants do not establish an object model or text format.

This contract distinguishes:

- **Established static evidence:** behavior explicitly represented in the supplied material.
- **[INFERENCE]:** an interpretation, not a behavioral requirement.
- **documented_only:** an unresolved interface, machine property, or dependency behavior.

All behavior is conditional on execution reaching the relevant operation. Valid accesses, returning dependencies, and normal completion are not established.

## Interface

### Explicit inputs

The represented interface has three explicit inputs and no declared result.

| Input | Represented width | Established use |
|---|---:|---|
| First input | Signed 32 bits | Shifted left by 15 positions for the fourth argument of the first invocation. |
| Second input | 32 bits, interpretation unspecified | Forwarded unchanged as the second argument of that invocation. |
| Third input | 32 bits, interpretation unspecified | Forwarded unchanged as its third argument. |

No explicit length, capacity, output destination, status destination, or ownership descriptor appears.

The function does not directly dereference the first input. Whether dependencies interpret its transformed value as an address is unknown.

### Ambient inputs

The represented interface also depends on:

- An incoming machine value shifted left by 12 positions.
- Incoming control state determining the first invocation target.
- Secondary machine-result values not described by complete return signatures.
- Fixed-location memory contents.
- Memory reached through returned values, loaded values, and arithmetic-derived addresses.
- The unresolved final transfer target.

These inputs must not be replaced with invented explicit parameters or assumed zero values.

### Outputs and completion

Observable outputs include dependency interactions, memory accesses, memory mutations, and the final transfer. There is no declared success or failure result.

The final transfer has four represented arguments:

1. The constant 0x72.
2. A saved 32-bit word loaded through a late memory-derived base at offset 0x64.
3. A saved 32-bit word loaded from the final record base at offset 0x14.
4. A later 32-bit word loaded through that late memory-derived base at offset 0x74.

An ordinary return follows in the representation. Whether the transfer target returns, terminates execution, or transfers elsewhere is **documented_only**.

### Preconditions

Successful traversal requires executable indirect targets, suitable ambient state, accessible fixed and computed addresses, and compatibility with actual alignment and access-width requirements. Dependency-produced values must support their subsequent uses if those uses are to succeed.

The function visibly validates none of these conditions. Zero or negative explicit inputs do not select a visible alternative path.

## Behavior

### Dependency invocation phase

The first target is derived from incoming control state by adding 0xFFE. Its arguments, in order, are the ambient scalar shifted left by 12, the second input, the third input, and the first input shifted left by 15.

A long straight-line series of further indirect invocations follows. Their targets are generally derived from embedded address-like values or address references with the same displacement.

Calls consume represented 32-bit results, halves of represented 64-bit results, secondary machine results, constants, and saved values carried across intervening calls. Displayed argument counts vary. An invocation without displayed arguments does not establish absence of ambient inputs or side effects.

No explicit loop, conditional branch, early-success path, or local error path appears in this phase. Dependencies may nevertheless fail, change memory, or not return.

### Widths and arithmetic

A **byte**, **halfword**, and **word** denote represented widths of 8, 16, and 32 bits. A represented 64-bit result has distinct lower and upper 32-bit words.

Whole-result unsigned right shifts by 34, 36, and 44 occur before narrowing to 32 bits. They are not shifts of the lower word. Unsigned right shifts insert zeros.

Masks select fields of widths including 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, and one bit. Mask placement relative to shifting is significant. Byte and halfword stores retain only the low eight and sixteen bits respectively.

Several expressions use signed 32-bit quantities. Signed overflow, signed left shifts, address arithmetic, and signed right shifts do not establish portable results for all bit patterns. Where this contract discusses truncating arithmetic, it explicitly assumes 32-bit operations retaining the low 32 bits. Address width and actual machine arithmetic remain **documented_only**.

### Opening direct-access boundary

The following descriptive names identify values, not implementation identifiers or declared structures:

- **Final lower word** and **final upper word:** halves of the last dependency-phase result.
- **Shifted payload:** an earlier saved result shifted left by 25.
- **Scaled base:** another earlier result multiplied by 0x02000000.
- **Selector:** an earlier result shifted left by eight, masked with 0x7FFF, then shifted right by six.
- **Selector base:** twice the selector.

Under truncating 32-bit arithmetic, the selector is four times its originating result’s low seven bits: a multiple of four from 0 through 508.

The direct phase begins with a word store of the shifted payload at final upper word plus 0x14, followed by a byte store of the final upper word at shifted payload plus 9. A word store of the scaled base at 256 times final lower word plus 0x54 intervenes before the same byte destination receives the shifted payload’s low byte.

Next, a zero word is stored at scaled base plus 0x14, and a byte is read at scaled base plus 5. A zero word is stored at selector base plus 0x14 before the saved byte, zero-extended, is written as a word at scaled base plus 0x34.

A fresh word read at scaled base plus 0x14 follows. The zero-word store at selector base plus 0x14 is repeated, and two successive byte stores place the freshly read word’s low byte at selector base plus 0x15.

The fresh word need not be zero. Intervening destinations may alias its source. Duplicate accesses remain part of the represented trace.

### Early fixed-location accesses and computed mutation

Immediately after those duplicate byte stores, the function reads the fixed location **0x05**, converts the value to a 32-bit unsigned quantity, and then writes the low byte of the selector base to **0x01**.

The declaration of the source at 0x05 is absent. Its actual read width and any extension performed before the 32-bit conversion are **documented_only**. It must not be specified unconditionally as an unsigned-byte read.

Call the converted value the **fixed-read base**. The fixed write at 0x01 precedes:

- A byte store at fixed-read base plus 0x15.
- Two byte stores at scaled base plus 0x11, with a computed word store between them.
- A word read at fixed-read base plus 0x54.

Call that last saved word the **retained word**. The source location is subsequently overwritten with the earlier word read from scaled base plus 0x14. Thus the retained word is a pre-overwrite value, not a later reload.

Further accesses include selector-base-relative word stores, a word read at scaled base plus 0x74, self-relative stores using the retained word, and byte stores through another loaded base. A compound operation reads selector base plus 0x74 twice, using those reads for a computed word destination and its stored value. No independent ordering of those two subexpression reads is established.

A later fixed write has this placement:

- After a word store at selector base plus 0x14 of a value derived from the earlier scaled-base origin, a **word store at 0x54** receives twice the retained word.
- Selector base plus 0x14 then receives the retained word twice.
- Two byte stores at selector base plus 0x15 receive the low byte of four times the retained word.
- A **byte store at 0x15** then receives the selector base’s low byte.

The fixed word write at 0x54 and later byte write at 0x15 therefore occur amid computed accesses, not in a separate initialization stage. Their positions matter if computed destinations or subsequent sources overlap them.

The remaining computed mutation uses loaded words directly or scaled by two, four, or eight. It includes word accesses at offsets 0x14, 0x24, 0x34, 0x44, 0x54, 0x64, and 0x74; unsigned-byte reads at offset 5; and byte stores at offsets 9, 0x0D, 0x11, and 0x15. A loaded byte can itself become the base of a later word read.

These are access relationships, not a recovered structure layout.

### Constants and repeated transfer block

A later loaded word determines three destinations receiving 0x3A: a byte destination at four times that word, a halfword destination at eight times it, and a word destination 0x20 bytes beyond the byte destination.

Call twice this loaded word the **final record base**. Call four times it the **fourfold record base**.

A separate computed base, called the **transfer base**, is formed by subtracting decimal 150 from four times the retained word and multiplying the result by four. These calculations remain subject to the arithmetic qualifications above.

At transfer base plus 0x20, successive word stores write 0x20, 0x2F, a freshly read unsigned byte from transfer base plus 4 widened to a word, and 0x20 again. The byte read follows the first two stores and can observe their effects under aliasing.

The function then reads an **anchor** from fixed location **0x087FBBBC**. The global declaration is unavailable: consumption as a 32-bit value does not independently establish the physical read width.

After the anchor read, a **word store of 0x20 at 0x60** occurs before the repeated computed-write block. The saved anchor is not reread following that store, even if the store affects memory associated with it.

Each represented repetition writes, in order:

- A word containing the fourfold record base to a scaled destination plus 0x54.
- The transfer base’s low byte at anchor plus 0x19.
- The anchor’s low byte at transfer base plus 0x11.
- A scaled-base word at anchor plus 0x34.

The first repetition uses eight times the retained word for the destination base and final stored word. Subsequent repetitions use eight times the adjusted value obtained before forming the transfer base. Thus both the first destination and the first final word differ in their source expressions.

This is a statically repeated block, not a visible conditional loop. Exact machine-level repetition count is **documented_only**. Repeated writes are not interchangeable with a single final-state update.

### Final mutation: interleaved reads and fixed-byte transitions

The opening order after the repeated block is significant:

- A word read at fixed address **0xA9**.
- A computed word read at fourfold record base plus 0x74.
- A byte write at fixed address **0x3A**.
- A word read at final record base plus 0x14, saving the later third transfer argument.
- A word read at transfer base plus 4, saving a value later written at anchor plus 0x74.

The byte written at 0x3A is the low eight bits of eight times the adjusted value used to form the transfer base, plus 0x30. The two saved reads occur **after** this fixed mutation; they are not entry snapshots of the final phase.

Computed byte stores then update final record base plus 0x0D and transfer base plus 8, 0x0D, and 9. The plus-8 destination receives 0x6C. A word self-reference is stored at final record base plus 0x54.

The fixed-byte transitions are:

1. The saved word from 0xA9 supplies its low byte to **0x67**.
2. A computed word read at final record base plus 0x44 intervenes.
3. A **fresh read of 0x67** supplies the byte written to **0x21**.

That read must not be replaced with reuse of the original saved byte. The intervening computed read may have externally visible memory behavior; ordinary-memory stability is an additional assumption.

The computed word just read supplies a byte at transfer base plus 0x10 and a word at anchor plus 0x54. It also supplies the signed right-shift expression whose low byte is stored at final record base plus 1. The shift count is the transfer base’s low eight bits and may exceed 31; actual large-count and signed-shift semantics are **documented_only**.

Further word mutations overwrite anchor plus 0x54 with the transfer base, load transfer base plus 0x40, store the saved restoration value at anchor plus 0x74, update transfer base plus 0x10, and store the final record base at anchor plus 0x44.

Only then does a second **fresh read of 0x67** supply a byte store at transfer base plus 9.

A byte store at final record base plus 9 and a word store of the saved 0xA9 word at transfer base plus 0x44 follow. A fresh word read at transfer base plus 0x54 then occurs before a third **fresh read of 0x67** supplies another byte store at transfer base plus 9.

These three reads of 0x67 are distinct observations. Intervening computed stores may alias 0x67; repeated byte results are not guaranteed.

Subsequent mutations use the low byte of the saved fourfold-record-relative word, restore the saved third transfer argument at final record base plus 0x54, and freshly read transfer base plus 0x14. Transfer base plus 9 and plus 0x44 are later overwritten using the word freshly read from transfer base plus 0x54.

Finally, a word read at final record base plus 4 supplies a new base. A halfword 0x65 is stored at twice that base before a word is read at its offset 0x64. Further operations include a word store of decimal 100, a fresh word read at that base plus 0x74, another base read at final record base plus 0x64, and a word store through twice the saved offset-0x64 value. The unresolved four-argument transfer follows.

## Side effects

Memory mutation is nontransactional. No rollback, validation, synchronization, atomicity, memory barrier, or local recovery mechanism is visible.

Fixed locations and computed destinations share the same potential address space. They must not be modeled as independent globals merely because their descriptions differ. Byte and halfword accesses can overlap word regions; resulting word contents depend on endianness and actual access semantics.

No direct allocation or release operation is identified. Dependencies may allocate, release, retain addresses, transfer ownership, or invalidate earlier values. No caller-owned-buffer contract or dependency-result lifetime guarantee is established.

There is no visible error result, exception handler, cleanup path, or explicit response to an invalid target, inaccessible address, or alignment failure. Earlier effects need not be undone when execution stops.

## Dependencies

Only F-aaf3e2df58ca has a supplied resolved identity. The following opaque references are local descriptive labels:

| Opaque reference | Role | Resolution |
|---|---|---|
| F-dep-001 | First target derived from incoming control state | documented_only |
| F-dep-002 | Ordered family of subsequent indirect invocation targets | Individual identities and contracts documented_only |
| F-dep-003 | Final indirect-transfer target | documented_only |

These labels do not establish distinct implementations, conventional signatures, preservation of globals, ownership behavior, memory validity, or normal return.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Initial forwarding.** With ordinary 32-bit shift behavior, ambient scalar 2 and first input 1 produce first and fourth arguments 8192 and 32768. The middle arguments retain their input bit patterns.

2. **Whole-result extraction.** A 64-bit result whose upper word is 0x80000000 yields 0x20000000 when shifted unsigned right by 34 and narrowed. Shifting only its lower word is not equivalent.

3. **Selector boundary.** Under truncating arithmetic, an originating low-seven-bit field of 127 produces selector 508 and selector base 1016, not selector 511.

4. **Early fixed-access order.** An observable read at 0x05 precedes the byte write at 0x01, which precedes the computed fixed-read-base-relative stores and word read. Moving the fixed write past those accesses does not satisfy the represented order.

5. **Anchor capture.** If the word write at 0x60 aliases anchor-related memory, the repeated block still uses the anchor captured before that write. Rereading 0x087FBBBC afterward is not equivalent.

6. **Intervening fixed mutation.** If the byte destination 0x3A overlaps either later saved-word source, those words are read after the byte mutation. Capturing both words before it is incorrect.

7. **Fresh fixed-byte observations.** If intervening computed writes change 0x67, later transfer-base-plus-9 stores consume the then-current reads. They need not reproduce the low byte originally saved from 0xA9.

8. **Width distinction.** With valid nonoverlapping destinations, the three 0x3A stores affect one, two, and four bytes. Three byte stores are insufficient.

9. **Duplicate visibility.** For device-visible destinations, both opening zero-word stores and both duplicate byte stores remain separate represented accesses.

10. **Final transfer.** If all preceding operations complete, F-dep-003 receives 0x72 followed by the specified saved offset-0x64 word, saved record-offset-0x14 word, and later offset-0x74 word. No success status follows from reachability alone.

## Uncertainties

The following remain **documented_only**:

- Domain purpose, intended layout, and accuracy of the represented entry interface.
- Dependency identities, signatures, side effects, failures, and ownership rules.
- Production and lifetime of secondary machine results.
- Actual address width, arithmetic overflow, signed shifts, and large shift counts.
- Endianness, alignment, mapping, and validity of low-address accesses.
- Access width and extension behavior for the undeclared global read at 0x05.
- Physical access width of the undeclared anchor read at 0x087FBBBC.
- Whether addresses denote ordinary memory, device registers, encoded references, or invalid locations.
- Hardware ordering within compound expressions.
- Instruction-level access trace and repetition count.
- Final target and completion behavior.

[INFERENCE]: Repeated offsets and self-relative stores resemble record mutation. This does not establish a structure definition or ownership contract.

## Evidence

This contract derives solely from the supplied static representation. Its evidence includes explicit access widths, ordering between separate represented operations, fixed-address interactions, arithmetic transformations, repeated writes, saved-versus-fresh value distinctions, and the final transfer arguments.

Embedded comments, warnings, prior prose, and reviewer feedback are not independent proof. No implementation access, external dependency inspection, instruction-level validation, runtime execution, or runtime tests have been performed.
