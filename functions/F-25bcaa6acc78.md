<!-- Trusted provenance: F-25bcaa6acc78; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-25bcaa6acc78

## Purpose

F-25bcaa6acc78 executes an ordered sequence of unresolved indirect calls, transforms call results and implicit incoming state, performs mixed-width memory updates, and finishes with another unresolved indirect operation.

The supplied representation establishes no business-level purpose. Repeated offsets and literal values do not establish initialization, serialization, text output, allocation, or a particular data structure.

This contract distinguishes:

- **Established static evidence:** represented calls, arithmetic, memory-access widths, and dependencies.
- **Conditional interpretation:** consequences requiring an explicit arithmetic or memory model.
- **documented_only:** behavior not resolved sufficiently to impose an unconditional requirement.

Completion is conditional on all preceding calls returning and all reached accesses completing. No runtime tests have been performed.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters and no explicit return value. Its effective inputs nevertheless include:

- Incoming control state selecting the first indirect destination.
- An incoming unsigned 32-bit value used in early field extraction.
- Primary and secondary outputs of unresolved calls.
- Words and bytes read through computed addresses.
- A byte read at fixed address 0x05.
- A **32-bit word read at fixed address 0x087FBBBC**, supplying the shared root for the repeated-store sequence and subsequent root-relative writes.
- A late 32-bit word read at fixed address 0xA9.
- Three separate late byte reads at fixed address 0x67.
- The unresolved destination of the final indirect operation.

There is no established success indicator, returned object, output-buffer contract, or error code. Observable outputs consist of calls and memory effects.

### Widths and arithmetic

| Representation | Width | Required distinction |
|---|---:|---|
| Byte | 8 bits | Stores retain the low eight bits. Numeric byte loads are zero-extended where represented. |
| Halfword | 16 bits | Stores affect two bytes, including stores of small constants. |
| Word | 32 bits | Used for memory transfers, values, address components, and saved results. |
| Paired result | 64 bits | Low and high 32-bit portions are selected separately. |

A paired result need not represent one semantic 64-bit number.

For established unsigned 32-bit expressions, left shifts discard bits above bit 31 and right shifts are logical. Arithmetic must not be silently interpreted as unbounded integer arithmetic. The supplied signed expressions do not independently establish overflow behavior, negative-value shift behavior, or address-wrap semantics.

Byte order, alignment, pointer representation, permitted address ranges, and the execution model for signed arithmetic remain **documented_only**. Address calculations below identify represented dependencies; wrapped 32-bit interpretations are conditional where signed arithmetic is involved.

### Preconditions and ownership

Every reached indirect destination must be callable, and every reached memory location must permit the represented access width. No validation of these conditions is represented.

Zero is not rejected as an address. Nonzero values are not validated as pointers.

No allocation, release, retention, or ownership transfer is established. Unresolved dependencies may perform any of those actions. Their preservation of globals, memory, or implicit result state must not be assumed.

## Behavior

### Indirect-call phase

The first indirect destination is formed from incoming control state plus displacement 4094. Later destinations are formed from embedded control-reference values plus that displacement. The representation does not establish a data-memory dispatch-table lookup.

The calls occur in a fixed represented sequence without a represented branch selecting a shorter sequence. Calls with zero operands and calls whose primary result is unused remain part of the observable behavior.

Argument dependencies include:

- Primary results shifted left by eight.
- High words of paired results passed independently of low words.
- Identical transformed values in multiple argument positions.
- Explicit zeros alongside unresolved secondary outputs.
- Results retained across intervening calls.
- Masked fields shifted before use.
- Secondary outputs used without an independently resolved callee contract.

Similarity of argument shapes does not establish identical destinations or effects. The exhaustive occurrence-by-occurrence call trace and complete signatures are **documented_only** here; this contract does not authorize removing or merging occurrences.

### Field transformations

The incoming unsigned word’s top ten bits are shifted left by seventeen. The result ranges from zero through 134,086,656 and has seventeen low zero bits.

An early retained unsigned word is masked to twenty bits, shifted right by ten, then left by five. The result ranges from zero through 32,736.

Other represented argument constructions include:

- Low twenty-four bits shifted left by six: at most thirty significant bits.
- Low twenty-two bits shifted left by eight or seven: at most thirty or twenty-nine significant bits.
- Low nineteen bits shifted left by two: at most twenty-one significant bits.
- Low twenty-four bits shifted right by twenty or nineteen: respectively four or five significant bits.
- Unsigned 64-bit results shifted right by 34, 36, or 44: respectively at most thirty, twenty-eight, or twenty significant bits.

These are width constraints, not recovered field meanings.

Several retained contributions are explicitly zeroed and subsequently only shifted. Those contributions remain zero; nearby calls and separately obtained secondary outputs do not thereby become optional or zero.

Intermediate truncation matters. A left shift followed by a right shift does not generally recover the original value.

### Values used by the direct-memory phase

The following descriptive terms are contract terminology, not recovered implementation identifiers:

| Term | Provenance |
|---|---|
| **Anchor** | A retained primary result multiplied by \(2^{25}\). |
| **Transformed base** | The low word of an earlier paired result shifted left by 25, retained through later calls. |
| **Narrow field** | Another primary result shifted left by eight, masked to fifteen bits, then shifted right by six. |
| **Narrow base** | Twice the narrow field. |
| **Retained shifted word** | An earlier primary result shifted left by eight. |
| **Final pair** | The last paired call result before explicit memory accesses. |
| **Small base** | The zero-extended byte subsequently read at fixed address 0x05. |
| **Saved word** | A word read at small-base offset 0x54 before that location is overwritten. |
| **Shared root** | The word subsequently read at fixed address 0x087FBBBC. |

Under a 32-bit bit-vector interpretation, both the anchor and transformed base have 25 low zero bits. The narrow field ranges from zero through 508 in steps of four; the narrow base ranges from zero through 1016 in steps of eight.

### Opening memory effects

The opening stores distinguish the final pair’s high word from the value stored through it: **the word at high-word offset 0x14 receives the transformed base**, not the high word itself.

At transformed-base offset 9, a byte store first receives the low byte of the final pair’s high word. Between that store and its overwrite with the low byte of the transformed base, a word at offset 0x54 from the final pair’s low word multiplied by 256 receives the anchor.

The anchor’s offset-0x14 word is then cleared. A byte read at anchor offset 5 precedes a clear at narrow-base offset 0x14; that byte is subsequently stored, zero-extended, as a word at anchor offset 0x34. Only afterward is anchor offset 0x14 read back. Another clear at narrow-base offset 0x14 precedes two separate stores of the read-back word’s low byte at narrow-base offset 0x15.

The read-back is not unconditionally zero: intervening accesses may alias the cleared word or have externally observable memory semantics.

The fixed-address byte read at 0x05 occurs after those two byte stores. Its zero-extended value supplies the small base. Subsequent effects include:

- A byte write at fixed address 1 carrying the narrow base’s low byte.
- A byte write at small-base offset 0x15 carrying the narrow field’s low byte.
- A byte at anchor offset 0x11 written from the retained shifted word and then cleared, with a computed word store between those writes.
- The saved-word read at small-base offset 0x54, followed by an overwrite of that location using the earlier anchor read-back.
- A word store of the narrow field at narrow-base offset 0x34.

### Loaded-address propagation

Loaded words subsequently determine destinations at offsets including 0x24, 0x44, 0x54, 0x64, and 0x74. Some locations receive their own address-derived value. Several byte stores retain only the low byte of such values.

The saved word supplies fourfold and eightfold scaled values. Repeated stores to narrow-base offset 0x14, byte stores at offset 0x15, and a word store at fixed address 0x54 remain distinct effects even where values repeat.

Later propagation includes a byte load at offset 5 of a saved-word-derived base; its zero-extended result supplies a base for an offset-0x74 word read. Another offset-5 byte load occurs later, rather than necessarily reusing the earlier byte. The resulting loaded word is doubled and used as a destination base.

For the remaining phases:

- **Loaded element** denotes the later word obtained through that byte-derived offset-0x74 access.
- **Doubled element** denotes twice the loaded element.
- **Payload base** denotes four times the loaded element.
- **Adjusted value** denotes four times the saved word minus 150.
- **Computed base** denotes four times the adjusted value.

These terms impose no allocation or disjointness assumptions.

### Literal stores and shared-root acquisition

Before shared-root acquisition, the loaded element supplies destinations for:

- A byte store of 0x3A at four times the loaded element.
- A halfword store of 0x3A at eight times the loaded element.
- A word store of 0x3A at payload-base offset 0x20.

Computed-base offset 0x20 then receives four ordered word stores: 0x20, 0x2F, a zero-extended byte loaded at computed-base offset 4, and 0x20.

**After the final one of those word stores, the function reads the 32-bit word at fixed address 0x087FBBBC into the shared root. The word store of 0x20 at fixed address 0x60 follows that read.** The root is therefore an independently read memory input, not merely a retained call result or an unspecified earlier load.

The acquired root value is retained for the repeated sequence and later root-relative writes. No explicit reload of 0x087FBBBC is represented during that sequence. This does not establish that its storage is immutable: earlier or later computed accesses may alias it.

### Repeated-store sequence

After the fixed-address-0x60 store, each represented group has the same four-access shape, in this order:

- A word destination at an eightfold-scaled base plus 0x54 receives the payload base.
- Shared-root offset 0x19 receives the computed base’s low byte.
- Computed-base offset 0x11 receives the shared root’s low byte.
- Shared-root offset 0x34 receives the eightfold-scaled base value.

The first group uses eight times the saved word. Subsequent groups use eight times the adjusted value. Thus the first root-offset-0x34 assignment differs from the later assignments, even though the byte-store values remain the same.

This is a finite represented repetition, not a data-dependent loop. Its exact repetition count is **documented_only** in this contract. The groups must not be collapsed to final memory contents where access count, faults, aliasing, or external observation matters.

### Late fixed-global ordering and reloads

The late phase begins only after the repeated stores. Its first saved input is the **32-bit word at fixed address 0xA9**.

That word read precedes, in represented order:

1. A word read at payload-base offset 0x74.
2. A byte write at fixed address 0x3A, carrying the low eight bits of eight times the adjusted value plus 0x30.
3. A saved word read at doubled-element offset 0x14.
4. A saved word read at computed-base offset 4.

Those two saved word reads are distinct from the earlier saved word used for address scaling.

Before fixed address 0x67 is written, computed stores affect doubled-element offset 0x0D, computed-base offsets 8, 0x0D, and 9, and doubled-element offset 0x54. The offset-8 byte receives 0x6C; the offset-0x54 word receives the doubled element itself.

**Fixed address 0x67 then receives the low byte of the saved 0xA9 word.** The function subsequently reads a word at doubled-element offset 0x44, then reads a byte from 0x67 for the byte write at fixed address 0x21.

Two further, separate byte reads from 0x67 supply two separate byte stores at computed-base offset 9:

- The first follows intervening computed byte and word accesses, including writes at shared-root offsets 0x54, 0x74, and 0x44, and a word copy into computed-base offset 0x10.
- Between that first offset-9 store and the second, the function writes a byte at doubled-element offset 9, stores the saved full 0xA9 word at computed-base offset 0x44, and reads a word at computed-base offset 0x54.

Thus there are **three represented byte reads of 0x67**: one for fixed address 0x21 and one for each of the two computed offset-9 stores. They are not interchangeable with a cached low byte of the original 0xA9 word. Intervening computed stores may alias 0x67, and external memory behavior is unresolved.

### Late copies and final operation

The late phase also contains a byte store at computed-base offset 0x10, later overwritten by a word copied from computed-base offset 0x40. The shared root’s offset-0x54 word first receives the loaded doubled-element-offset-0x44 word and then receives the computed base, with a computed byte store between them.

A byte at doubled-element offset 1 receives the low byte of a signed word shifted right by a count derived from the computed base’s low byte. Negative operands and counts outside the 32-bit width remain **documented_only**; no count masking is established.

After the second 0x67-dependent offset-9 store, further effects include:

- Propagation of the payload-offset-0x74 word’s low byte to computed destinations.
- Restoration of the saved doubled-element-offset-0x14 word into doubled-element offset 0x54.
- A word read at computed-base offset 0x14.
- Later overwrites of computed-base offset 9 and offset 0x44 using the word loaded at computed-base offset 0x54.
- A word read at doubled-element offset 4, followed by a halfword store of 0x65 at twice that loaded value.
- A word read at offset 0x64 from that loaded base.
- A byte store derived from the saved computed-base-offset-0x14 word and a word store of decimal 100 through the earlier computed-base-offset-0x54 result.
- A word read at the loaded base’s offset 0x74, followed by a word read at doubled-element offset 0x64 and final linking-style word stores. “Linking-style” describes the address/value relationship, not an established data structure.

The final unresolved operation receives four represented operands: 0x72, the word read at the loaded base’s offset 0x64, the saved doubled-element-offset-0x14 word, and the word read at the loaded base’s offset 0x74. A return follows only if that operation returns.

## Side effects

Observable behavior includes call order and operands, access widths, read provenance, repeated accesses, intermediate writes, and final indirect control transfer.

The fixed-address inventory includes:

| Address | Represented direct access |
|---|---|
| 0x01 | Byte write |
| 0x05 | Byte read |
| 0x15 | Byte write |
| 0x21 | Late byte write sourced by a read of 0x67 |
| 0x3A | Late computed-value byte write |
| 0x54 | Word write |
| 0x60 | Word write of 0x20, after shared-root acquisition |
| 0x67 | Byte write, then three separate byte reads |
| 0xA9 | Late word read |
| 0x087FBBBC | Word read supplying the shared root |

Computed accesses can overlap these locations. Distinct descriptive names do not establish separate storage. Offset 0x15 can overlap an offset-0x14 word; a word at offset 0x10 can overwrite an earlier byte there.

No transaction, rollback, retry, cleanup handler, or error conversion is represented. A fault or nonreturning dependency may leave earlier effects visible. Idempotence, thread safety, and reentrancy are not established.

## Dependencies

Only F-25bcaa6acc78 has a supplied resolved identity.

The following opaque labels are documentation placeholders:

- **F-U01:** the initial unresolved indirect-call occurrence.
- **F-U02:** the remaining unresolved call population; not a claim of one shared callee.
- **F-U03:** the final unresolved indirect operation.

Their identities, complete signatures, side effects, ownership behavior, and return guarantees are **documented_only**. No global-preservation or memory-preservation guarantee is assumed.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Incoming field boundary.** Incoming unsigned values 0x003FFFFF and 0x00400000 produce, respectively, zero and 0x00020000 for the top-ten-bit transformation.

2. **Masked field maximum.** A retained word whose low twenty bits are all one produces 0x7FE0 after selecting their upper ten bits and shifting left by five.

3. **Byte extension.** An anchor-offset-5 byte of 0xE7 is stored numerically as word value 231 at anchor offset 0x34.

4. **Opening store provenance.** If the final pair’s high word selects accessible storage, its offset-0x14 word receives the transformed base. Substituting the high word as the stored value is not accepted.

5. **Root acquisition order.** If all relevant accesses complete, the computed-offset-0x20 final store of 0x20 precedes the word read at 0x087FBBBC, which precedes the fixed-address-0x60 word store. Reading the root before those computed writes is not equivalent when aliasing or observation matters.

6. **Repeated-root use.** If the root read yields an accessible address, that acquired value selects the sequence’s root-relative destinations. A later change to the global word does not itself establish a represented reload.

7. **Late input order.** The 0xA9 word read must precede the payload-offset-0x74 read, fixed-address-0x3A byte write, and both subsequent saved word reads.

8. **Reload-sensitive behavior.** If an intervening computed store changes 0x67, a later represented read uses the byte then present there. Reusing the original 0xA9 low byte is not accepted. The three reads may yield different bytes.

9. **Mixed widths and intermediate values.** A halfword store of 0x3A is not replaceable by a byte store. Likewise, computed-offset-0x20 writes of 0x20, 0x2F, the byte-derived word, and 0x20 remain ordered effects.

10. **Final transfer.** If all preceding operations complete, F-U03 receives 0x72 first and the three saved words identified above afterward. No explicit success result is established.

## Uncertainties

The following remain **documented_only**:

- Business purpose and intended data model.
- Accuracy of the represented indirect-destination mechanism.
- Complete calling convention, implicit inputs, and per-occurrence dependency contracts.
- Signed overflow, signed shifts, excessive shift counts, and address wrapping.
- Byte order, alignment, permissions, and low-memory mapping.
- Alias relationships, concurrency, and external memory observers.
- Exact repeated-group count.
- Allocation, release, retention, ownership, faults, and termination.

The provenance and ordering of the 0x087FBBBC word read, and the late 0xA9/0x67 access dependencies, are established static evidence, not unresolved items.

**[INFERENCE]** Repeated offsets may reflect structured storage. Literal values may participate in an encoding. Neither interpretation is an acceptance requirement.

## Evidence

This contract derives from static inspection of the supplied representation: call-result dependencies, width-specific accesses, arithmetic constraints, literal stores, fixed-address inputs, repeated-store structure, and final operands.

In particular, the representation supplies the shared-root read between the final computed-offset-0x20 store and fixed-address-0x60 store, and separately represents all three late byte reads at 0x67.

Comments, prior documentation, and reviewer assertions are not independent semantic authority. No dependency implementation inspection, runtime execution, or tests were performed. No snapshot hash is asserted.
