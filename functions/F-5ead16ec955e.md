<!-- Trusted provenance: F-5ead16ec955e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5ead16ec955e

## Purpose

F-5ead16ec955e performs an ordered sequence of indirect dependency invocations, transforms their results into packed 32-bit values, mutates memory through fixed and computed addresses, and makes a final unresolved indirect invocation.

No higher-level application purpose is established. Repeated offsets do not prove an object schema, and character-like constants do not prove text construction.

This contract distinguishes:

- **Established static evidence:** operations, widths, dependencies, and ordering represented in the supplied source.
- **Inferred intent:** possible interpretations, not acceptance requirements.
- **documented_only:** unresolved behavior for which no concrete requirement can be established.

All execution requirements are conditional on the static representation accurately describing the executable path. They do not certify the validity of its addresses or indirect targets.

## Interface

### Explicit inputs

Four 32-bit argument positions are represented.

| Position | Established use |
|---|---|
| First | No explicit use is shown. |
| Second | No explicit use is shown. |
| Third | Supplies operands shifted left by 24 bits during early dependency interactions. |
| Fourth | Supplies two equal operands, each shifted left by seven bits, to an early dependency invocation. |

The third and fourth positions are displayed as signed quantities, but their application domains are unknown.

Under a conditional 32-bit bit-vector interpretation, shifting the third argument by 24 retains only its original low eight bits; shifting the fourth by seven discards its original high seven bits. These facts concern the displayed operands, not every possible machine-state input to a dependency.

### Implicit inputs

Execution also depends on:

- An incoming return-link value used to derive the first indirect target.
- Later location-derived indirect targets.
- Primary, paired, and secondary-register dependency results.
- Mutable memory accessed through computed addresses and loaded words.
- Fixed-location memory, including low addresses.
- A separately represented global word supplying a later base.
- The final unresolved indirect target.

Four explicit argument values therefore do not constitute a complete invocation model.

### Outputs and completion

No explicit return value is established. Outputs comprise dependency interactions and memory effects.

The final invocation receives four represented 32-bit operands:

1. 0x72.
2. A late loaded word.
3. A saved word read earlier in the post-repetition phase.
4. Another late loaded word.

Their precise origins are specified below. A return follows the invocation in the representation; whether the target returns is **documented_only**.

### Preconditions

Normal completion requires every reached target and memory access to be valid for the actual execution environment, including access widths, alignment, calling convention, and incoming machine state. Dependencies must return wherever later behavior is expected.

No argument validation, address validation, capacity check, or recovery path is represented. Zero-valued computed bases are not explicitly rejected.

## Behavior

### Arithmetic and access conventions

A **byte** is eight bits, a **halfword** 16 bits, and a **word** 32 bits. Paired dependency results are displayed as 64 bits.

For width-sensitive examples, the **conditional bit-vector interpretation** means that 32-bit arithmetic retains the low 32 bits, left shifts discard outgoing bits, unsigned right shifts insert zeros, and narrow stores retain the low eight or 16 bits. This interpretation does not resolve signed overflow, signed right shifts, or architecture-specific addressing.

Low-half extraction before a shift must remain distinct from shifting a whole 64-bit result. Logical shifts of a 64-bit result by 34, 36, and 44 followed by 32-bit extraction respectively yield its high half shifted right by two, four, and twelve.

Masks select several field widths, including 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, and 24 bits. Intermediate truncations matter. No saturation, rounding, rejection, or overflow error is shown.

Addresses below are numeric access locations, not proven object fields. All ordering remains relevant when accesses alias.

### Dependency-driven preparation

A long straight-line dependency phase precedes direct mutation. No explicit loop or conditional branch is represented.

Three invocations without displayed explicit operands precede the invocation receiving the fourth argument shifted left by seven twice. Subsequent interactions combine dependency results shifted left by eight with the third argument shifted left by 24, including both operand orders.

Later interactions combine shifted results, extracted halves, secondary-register values, zeros, masked values, and saved intermediates. Calls without displayed operands remain observable interactions; they are not established as pure operations or global-preserving calls. Similar argument shapes do not establish shared target identity.

Explicitly zero intermediates remain zero through represented shifts under the bit-vector interpretation. That does not remove any intervening dependency interaction.

### Opening mutation and fixed-location boundaries

The following descriptive labels identify values, not source identifiers:

- **First packed base:** an earlier result shifted left by 25.
- **Second packed base:** a separate result multiplied by 2²⁵.
- **Saved shifted value:** another result shifted left by eight.
- **Selector:** a result shifted left by eight, masked to 15 bits, then logically shifted right by six.
- **Selector base:** twice the selector.
- **Readback word:** the opening word read from the second packed base plus 0x14.
- **Fixed-byte base:** the zero-extended byte subsequently read at fixed address 0x05.
- **Linked base:** the word subsequently read from the fixed-byte base plus 0x54.

Under bit-vector semantics, the selector is four times the underlying result’s low seven bits: 0 through 508 in steps of four. The selector base is 0 through 1016 in steps of eight.

The final paired preparation result supplies the opening high-half base and a low-half base scaled by 256. The opening writes publish the first packed base as a word at high-half-plus-0x14, write the high half’s low byte at first-packed-base-plus-9, publish the second packed base at scaled-low-half-plus-0x54, and overwrite first-packed-base-plus-9 with the first packed base’s low byte. That overwrite is zero under the bit-vector interpretation.

Next, the second packed base’s offset 0x14 receives a zero word. Its offset 5 supplies a byte; the selector base’s offset 0x14 then receives a zero word, and the byte is zero-extended into a word at second-packed-base-plus-0x34. The readback word is then loaded. Another zero word at selector-base-plus-0x14 precedes two identical byte writes of the readback word’s low byte at selector-base-plus-0x15.

The next fixed accesses have specific positions among computed accesses:

| Fixed access | Required surrounding order |
|---|---|
| Byte read at 0x05 | Occurs after both selector-base-plus-0x15 byte writes. Its zero-extended result becomes the fixed-byte base. |
| Byte write at 0x01 | Immediately follows that read and stores the selector base’s low byte. |
| Word write at 0x54 | Occurs later, between a word write of twice the second packed base at selector-base-plus-0x14 and two word writes of the linked base to that same computed location. The fixed word receives twice the linked base. |
| Byte write at 0x15 | Occurs after those two linked-base word writes and two byte writes of four times the linked base at selector-base-plus-0x15. It stores the selector base’s low byte. |

Between the fixed 0x01 and 0x54 writes, the fixed-byte base’s offset 0x15 receives the selector’s low byte. The second packed base’s offset 0x11 receives the saved shifted value’s low byte, then zero, with a word write of the saved shifted value through the readback word’s offset 0x54 between them.

The linked base is read from fixed-byte-base-plus-0x54 before that location is overwritten with the readback word. Further operations publish the selector at selector-base-plus-0x34 and follow words at offset 0x74 into additional destinations. Repeated linked-base self-publications and low-byte writes remain distinct operations.

Thus the fixed accesses are not an unordered set of global effects. For example, the fixed 0x01 write can change memory addressed using the already saved fixed-byte base, but it does not retroactively change that saved base.

### Linked and scaled updates

Subsequent accesses use loaded words and their doubled, fourfold, and eightfold values as addresses. They include byte accesses at offsets 5, 9, 0x0D, and 0x15, and word accesses at offsets 4, 0x14, 0x24, 0x34, 0x44, 0x54, 0x64, and 0x74.

Observable features include:

- Repeated writes to the same location, sometimes with different values.
- Word self-publications through loaded bases.
- Bytes read at offset 5, zero-extended, then used to derive word-read addresses.
- Later reads of locations already subject to computed writes.
- Byte writes within regions also written as words.

One operation contains two reads of selector-base-plus-0x74: one derives the destination, the other supplies the stored value. Their relative evaluation order is unresolved. The two reads must not simply be treated as one cached read where reads are observable.

For the later phases, define:

- **Fourfold base:** four times the linked base.
- **Terminal link:** the later word reached at offset 0x74 through a byte read at offset 5 of another loaded base.
- **Doubled terminal base:** twice the terminal link.
- **Adjusted base:** the fourfold base minus 0x96.
- **Publication base:** four times the adjusted base.
- **Publication value:** four times the terminal link.
- **Global base:** the separately loaded global word.

These labels preserve value relationships without implying valid pointers or disjoint storage.

### Constant stores and repeated publication

The terminal link supplies destinations for three stores of 0x3A: a byte at its fourfold address, a halfword at its eightfold address, and a word at fourfold-address-plus-0x20.

Publication-base-plus-0x20 receives, in order, the words 0x20 and 0x2F, then a word containing the zero-extended byte read from publication-base-plus-4, then 0x20 again. These are distinct writes.

The repeated-publication phase begins with a separate prefix:

- The global base is loaded.
- Fixed address 0x60 receives the word 0x20.
- A computed word store publishes the publication value at eight-times-linked-base-plus-0x54.
- The first byte/byte/word group follows.

That first group writes the publication base’s low byte at global-base-plus-0x19, the global base’s low byte at publication-base-plus-0x11, and eight times the linked base as a word at global-base-plus-0x34.

A word store of the publication value at eight-times-adjusted-base-plus-0x54 follows. Subsequent groups repeat the same two byte publications, but publish eight times the **adjusted base** at global-base-plus-0x34. Between these groups, the computed word store continues using eight-times-adjusted-base-plus-0x54.

The represented region ends after a byte/byte/word group, without another following computed store. The exact repetition count remains **documented_only** here. Neither the leading computed store nor repeated stores may be collapsed into an assumed final state.

The global base is retained as a saved value; the representation does not reload its supplying global word during this region.

### Post-repetition fixed read and saved values

After the final repeated word publication, the following read/write order is established:

1. A word is read at fixed address 0xA9 and saved as the **fixed snapshot**.
2. A word is read at publication-value-plus-0x74 and saved as the **terminal snapshot**.
3. Fixed address 0x3A receives the low byte of eight times the adjusted base plus 0x30.
4. A word is read at doubled-terminal-base-plus-0x14 and saved as the **restoration snapshot**.
5. A word is read at publication-base-plus-4 and saved as the **base snapshot**.

The fixed 0x3A write therefore occurs before the last two saved reads. If it overlaps either read, those reads must observe the resulting memory state. Conversely, later stores do not change already saved snapshots.

Next, doubled-terminal-base-plus-0x0D receives the publication base’s low byte. The publication base receives 0x6C at offset 8 and the global base’s low byte at offsets 0x0D and 9. Doubled-terminal-base-plus-0x54 receives the doubled terminal base as a word.

### Fixed byte publication and reloads

Fixed address 0x67 then receives the fixed snapshot’s low byte. A word read at doubled-terminal-base-plus-0x44 intervenes before a **fresh byte read at 0x67** supplies the byte written at fixed address 0x21.

The intervening computed read must retain its position. No computed write is shown between this initial 0x67 write and the read supplying 0x21; environmental read side effects or concurrent modification remain unresolved.

The loaded word also supplies a byte at publication-base-plus-0x10, a word at global-base-plus-0x54, and a signed right-shift expression whose low byte is written at doubled-terminal-base-plus-1. The shift count is the publication base’s low eight bits. Counts of 32 or greater must not silently be assigned modulo-32 semantics.

Global-base-plus-0x54 is then overwritten with the publication base. A word read from publication-base-plus-0x40 precedes publication of the base snapshot at global-base-plus-0x74, a word write at publication-base-plus-0x10, and publication of the doubled terminal base at global-base-plus-0x44.

Only then does another **fresh read at 0x67** supply the byte written at publication-base-plus-9. A byte publication at doubled-terminal-base-plus-9 and a word write of the fixed snapshot at publication-base-plus-0x44 follow. A word is then read at publication-base-plus-0x54, becoming the **late destination**. Another **fresh read at 0x67** supplies another byte write at publication-base-plus-9.

The later 0x67 reads cannot be replaced with the original fixed snapshot’s low byte: intervening computed writes can overlap 0x67.

### Final mutation and transfer

The terminal snapshot’s low byte is written at late-destination-plus-0x0D. The restoration snapshot is written at doubled-terminal-base-plus-0x54. A word read at publication-base-plus-0x14 is saved before further terminal-snapshot byte publication, a byte overwrite at publication-base-plus-9, and a word overwrite at publication-base-plus-0x44.

A word loaded from doubled-terminal-base-plus-4 supplies another base. Twice that base receives the halfword 0x65. Its offset 0x64 supplies the **transfer word**; its offset 0x74 later supplies the **final word**. Between these reads, the late destination receives a saved low byte at offset 0x11 and the word decimal 100 at offset 0x54.

A further word read at doubled-terminal-base-plus-0x64 supplies the **final destination**. The closing updates publish the newer loaded base’s low byte at doubled-terminal-base-plus-9, the transfer word at final-destination-plus-0x54, and the final destination as a word at twice the transfer word.

The unresolved invocation then receives, in order: 0x72, the transfer word, the restoration snapshot, and the final word.

## Side effects

No stable structure layout, allocation boundary, or disjointness between bases is established. Fixed and computed accesses may overlap. Consequently, a computed store can change a later fixed read, and a fixed write can change a later computed read or address.

Byte writes may partially overwrite words; resulting word values depend on byte order. A prior zero store does not guarantee a later zero read if intervening writes overlap it.

No allocation, release, ownership transfer, lifetime guarantee, lock, atomic operation, or memory barrier is identified. Dependency effects are unknown; dependencies must not be assumed to preserve globals or retain or release memory.

No explicit error result, rollback, retry, exception handler, or failure notification is represented. Partial execution may leave earlier effects intact.

## Dependencies

F-5ead16ec955e is the only supplied opaque function ID. No separately established opaque IDs are available for its indirect dependencies; none are invented here.

The dependency boundary includes the entry-link-derived invocation, subsequent location-derived invocations, result-producing interactions, and the final unresolved invocation. Target identities, identity relationships, complete interfaces, memory effects, ownership behavior, errors, and return behavior remain **documented_only**.

## Acceptance examples

These are conditional static criteria, not executed tests.

- **Argument bits:** Fourth-argument values differing only in their high seven bits produce equal displayed shifted operands under bit-vector semantics. This does not establish whole-execution equivalence.
- **Selector boundary:** An underlying low-seven-bit value of 0x7F produces selector 508 and selector base 1016. Zero produces both zero, without represented rejection.
- **Opening overwrite:** First-packed-base-plus-9 receives the high-half-derived byte before its later zero overwrite. Observing only the final zero is insufficient.
- **Byte extension:** An opening offset-5 byte value of 0xE3 produces the word 0x000000E3 at the specified offset 0x34.
- **Fixed-byte dependency:** A fixed 0x05 read of 0xE3 makes 0xE3 the saved fixed-byte base. The following 0x01 write does not substitute a new base.
- **Repetition prefix:** Even if destinations overlap, the fixed word write at 0x60 and leading computed store precede the first byte/byte/word group.
- **Saved-read aliasing:** If the fixed 0x3A write overlaps the restoration-snapshot read, that read observes the modified bytes under the environment’s byte order.
- **Reload distinction:** If an intervening computed store changes 0x67, the next byte publication sourced from 0x67 uses the newly read byte, not a cached snapshot byte.
- **Widths:** The halfword write of 0x65 and word write of decimal 100 are not interchangeable with broader or narrower writes.
- **Partial failure:** A nonreturning preparation dependency prevents later direct writes; no reversal of earlier effects is established.

## Uncertainties

The following remain **documented_only**:

- Higher-level purpose and argument meanings.
- Accuracy of the represented boundary, calling convention, and straight-line path.
- Dependency identities, interfaces, and secondary-register semantics.
- Byte order, alignment rules, signed overflow, and variable-shift behavior.
- Validity and meaning of fixed low-address accesses.
- Ownership, lifetime, concurrency, and device-memory semantics.
- Exact repeated-publication count.
- Relative evaluation order of the paired offset-0x74 reads.
- Final target identity and whether it returns.
- A concrete final memory image under arbitrary aliasing.

Repeated offsets suggest structured storage, and character-like constants suggest possible data construction. Both are inferred intent only.

## Evidence

The supplied static source is the sole behavioral evidence. The contract derives widths, value relationships, fixed-access boundaries, publication-prefix ordering, saved-read dependencies, reload distinctions, and final operands from that representation.

Comments, prior prose, and reviewer feedback are not independent evidence. No callee implementations, machine-level trace, memory map, or independent interface documentation were supplied.

No runtime tests have been performed. No successful execution or concrete final memory state is claimed.
