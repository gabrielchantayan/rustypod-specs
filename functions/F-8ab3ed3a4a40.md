<!-- Trusted provenance: F-8ab3ed3a4a40; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8ab3ed3a4a40

## Purpose

F-8ab3ed3a4a40 performs an ordered sequence of indirect dependency interactions, transforms dependency-produced values, and mutates memory through computed addresses and fixed storage locations. Its last represented interaction is an unresolved four-argument indirect transfer.

No application-level purpose is established. Character-like constants do not establish formatting or text output. Repeated offsets and scaled addresses do not establish allocation, collection management, or a particular object type.

This contract distinguishes:

- **Established static evidence:** represented interactions, access widths, value captures, and ordering.
- **Inferred intent:** possible interpretations that impose no acceptance requirements.
- **documented_only:** unresolved behavior or details not fully specified here.

Every operation is conditional on execution reaching it. Valid addressing, dependency return, and normal completion are not guaranteed.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First input | 32 bits | Forwarded unchanged as the first displayed argument of the initial invocation. |
| Second input | Signed 32-bit representation | Forwarded unchanged as the second displayed argument and shifted left six positions for the third. |

Neither input is established to be a pointer, size, count, or index. No validation or rejection branch precedes the initial invocation.

### Implicit inputs

Execution also depends on incoming return-link state, indirect targets, primary and secondary dependency results, existing memory, fixed storage, and a global word captured late as an address base.

The first target is derived by adding 0xFFE to incoming return-link state. Later targets are represented through further address calculations. Their implementations and complete calling conventions are unknown.

### Outputs and completion

The represented interface supplies no explicit return value. Outputs consist of dependency interactions and memory effects.

The final transfer receives four displayed arguments:

1. 0x72.
2. A retained 32-bit word read late from traversal-base offset 0x64.
3. A retained 32-bit word read shortly after the repeated-store block from doubled-lookup-base offset 0x14.
4. A later 32-bit word read from traversal-base offset 0x74, using the same captured base as the second argument.

These are captured observations, not requests to reread their original locations at transfer time.

A return follows the final interaction in the representation. Whether the target returns, transfers control elsewhere, or terminates execution is **documented_only**.

### Preconditions and ownership

Reaching an operation successfully requires executable targets and permitted accesses at its computed address and width. Low addresses, unaligned addresses, and scaled dependency results may be involved.

No allocation, ownership, retention, release, or lifetime protocol is established. Unknown dependencies must not be assumed to preserve globals or to borrow, consume, retain, allocate, or release memory.

## Behavior

### Widths, arithmetic, and observation

A byte is 8 bits, a halfword 16 bits, and a word 32 bits. Several dependency results are represented as 64-bit quantities from which lower and upper 32-bit portions are extracted. This does not establish a language-level 64-bit return convention.

Byte stores retain the low eight bits. Explicit unsigned byte loads are zero-extended when stored as words. Explicit unsigned right shifts insert zeros.

Signed 32-bit representations participate in many left shifts, additions, and multiplications. Portable overflow semantics are not established. Examples using **32-bit truncating arithmetic** are conditional: each intermediate retains its low 32 bits.

One late right shift uses a signed 32-bit operand and a count obtained from the low eight bits of an address. Negative-operand behavior and counts outside 0–31 remain **documented_only**. No count masking is established.

The visible body has no ordinary loop or validation branch. Dependencies can nevertheless branch, mutate memory, fault, or fail to return. Access ordering matters wherever addresses overlap or the environment observes individual accesses. Repeated stores are separate represented effects; final-state equivalence alone is insufficient.

### Dependency-mediated phase

The initial invocation forwards the two inputs and the shifted second input. A long ordered sequence of further indirect invocations follows before direct mutation begins.

Displayed arguments include shifted results, masked results, explicit zeros, both portions of paired results, and earlier values retained across intervening calls. Some invocations display no arguments. That display does not prove that incoming register state is irrelevant.

Calls whose primary results are unused remain observable boundaries. Unknown effects do not justify deleting or exchanging them. The complete invocation-by-invocation argument and result contract is **documented_only**.

### Descriptive address vocabulary

These terms name retained values, not verified objects:

| Term | Meaning |
|---|---|
| Shifted destination | A dependency-produced word shifted left 25 positions. |
| Large scaled base | A separate dependency result multiplied by 0x02000000. |
| Retained shifted word | Another dependency result shifted left eight positions. |
| Small offset | A producer shifted left eight positions, masked with 0x7FFF, then logically shifted right six. |
| Small base | Twice the small offset. |
| Fixed-storage base | The converted value captured from storage associated with address 0x05. |
| Anchor | The old word captured from fixed-storage-base offset 0x54. |
| Lookup value | A later word obtained through a byte-based lookup at offset 0x74. |
| Doubled lookup base | Twice the lookup value. |
| Scaled lookup base | Four times the lookup value. |
| Adjusted scalar | Four times the anchor, minus 0x96. |
| Adjusted base | Four times the adjusted scalar. |
| Captured global base | A global word captured immediately before the fixed word write at 0x60. |

Under 32-bit truncating arithmetic, the small offset is a multiple of four between 0 and 508; the small base is a multiple of eight between 0 and 1016.

### Initial mutation and fixed-storage capture

The first direct word store places the shifted destination at offset 0x14 from the final paired dependency result’s upper word. The function then stores that upper word’s low byte at shifted-destination offset 9.

Next, it stores the large scaled base at offset 0x54 from 0x100 times the final lower result word, then overwrites shifted-destination offset 9 with the shifted destination’s low byte. Both writes to that byte remain distinct events.

The next mutation group clears large-scaled-base word offset 0x14, reads its byte offset 5, clears small-base word offset 0x14, and writes the zero-extended byte at large-scaled-base word offset 0x34. It then reloads large-scaled-base word offset 0x14, clears small-base word offset 0x14 again, and writes the reloaded word’s low byte twice at small-base offset 0x15.

Aliasing prevents replacing that reload with an assumed zero.

The fixed-storage-base capture associated with 0x05 precedes the byte write at fixed address 0x01. The latter receives the small base’s low byte. The width of the 0x05 read is not independently declared.

Using the captured fixed-storage base, the function writes the small offset’s low byte at offset 0x15. It also writes the retained shifted word’s low byte at large-scaled-base offset 0x11, writes that retained word at offset 0x54 through the previously reloaded word, and clears the byte at large-scaled-base offset 0x11.

It then captures the anchor from fixed-storage-base word offset 0x54 and replaces that word with the earlier reload. Later accesses use the captured base and anchor, even if intervening writes overlap their original storage.

### Traversals and intermediate mutation

Further accesses follow words at offset 0x74 from the large scaled base and small base. Self-referential word stores occur at offsets 0x24 and 0x54. Some byte and word stores repeat.

A fixed 32-bit write at 0x54 stores the anchor shifted left once. This precedes the fixed byte write at 0x15, which receives the small base’s low byte. Computed stores at small-base offsets 0x14 and 0x15 intervene. Their destinations are not established to be disjoint from fixed storage.

Subsequent operations use twofold, fourfold, and eightfold forms of loaded values as addresses and data. They include word fields at offsets 0x14, 0x34, 0x44, and 0x54; traversal reads at 4, 0x64, and 0x74; and byte mutations at 9, 0x0D, 0x11, and 0x15.

A byte can itself become the base for a word read at offset 0x74. Multiple reads at byte offset 5 are separated by computed stores. The lookup value retained for the final phases comes from a later lookup; substituting an earlier result is unjustified.

Before the repeated block, the function stores 0x3A:

- As a byte at four times the lookup value.
- As a halfword at eight times that value.
- As a word at scaled-lookup-base offset 0x20.

Adjusted-base word offset 0x20 receives, successively, 0x20, 0x2F, a zero-extended byte freshly read from adjusted-base offset 4, and 0x20. These are memory effects, not established character output.

### Captured global base and repeated stores

The global base is captured before the fixed 32-bit write of 0x20 at 0x60. It is not reloaded during the repeated block, including when computed writes overlap its original storage.

The introductory four-store group has this order:

- A word containing the scaled lookup base at offset 0x54 from eight times the anchor.
- A byte containing the adjusted base’s low byte at captured-global-base offset 0x19.
- A byte containing the captured global base’s low byte at adjusted-base offset 0x11.
- A word containing eight times the anchor at captured-global-base offset 0x34.

The following repeated groups preserve those widths and that four-store order, but use eight times the **adjusted scalar** for the first destination’s base and the fourth stored value. The introductory group therefore differs in both destination and data.

The exact repetition count is not enumerated in this contract and remains **documented_only**. This limitation does not permit collapsing the represented stores into one final-state update.

### Post-block captures

Immediately after the repeated stores, the function captures the fixed word at 0xA9 and then the word at scaled-lookup-base offset 0x74.

It next writes a byte at fixed address 0x3A, containing the low eight bits of eight times the adjusted scalar plus 0x30. Only afterward does it capture the words at doubled-lookup-base offset 0x14 and adjusted-base offset 4.

Thus, the last two captures can observe changes caused by the 0x3A write. The earlier captures remain retained observations.

### Final mutation and fresh reads of 0x67

Byte writes first affect doubled-lookup-base offset 0x0D and adjusted-base offsets 8, 0x0D, and 9. Offset 8 receives 0x6C; the others receive retained base-derived bytes.

A self-referential word store at doubled-lookup-base offset 0x54 precedes the later byte update at adjusted-base offset 0x10.

The function writes the captured 0xA9 word’s low byte to fixed address 0x67, loads doubled-lookup-base word offset 0x44, then reads 0x67 for the byte store at fixed address 0x21.

The offset-0x44 word supplies the byte at adjusted-base offset 0x10 and a word at captured-global-base offset 0x54. Its variable-count signed-right-shifted form supplies doubled-lookup-base byte offset 1. The captured-global-base word offset 0x54 is then overwritten with the adjusted base.

Next, the function reads adjusted-base word offset 0x40. Before storing that word at adjusted-base offset 0x10, it writes the retained adjusted-base offset-4 capture at captured-global-base offset 0x74. It then writes the doubled lookup base at captured-global-base offset 0x44.

A second, fresh read of 0x67 supplies adjusted-base byte offset 9. A byte write at doubled-lookup-base offset 9 and a word write of the retained 0xA9 capture at adjusted-base offset 0x44 follow.

The function then loads adjusted-base word offset 0x54 as a new traversal base. A third, fresh read of 0x67 supplies another byte store at adjusted-base offset 9.

There is one explicit fixed write to 0x67 and three distinct later reads. Computed writes between reads may overlap 0x67; the three copied bytes need not agree.

### Final traversal and transfer

After the third read-and-copy of 0x67, the following interleaving is established:

- The retained scaled-lookup-base offset-0x74 capture supplies its low byte to the newly loaded traversal base’s offset 0x0D.
- The retained doubled-lookup-base offset-0x14 capture is restored as a word at doubled-lookup-base offset 0x54.
- The function reads a word at adjusted-base offset 0x14.
- Only then does the retained offset-0x74 capture supply its low byte to doubled-lookup-base offset 0x11.

The two byte stores therefore straddle both the restoration and the adjusted-base read. They are not a contiguous pair of writes.

The function next writes the new traversal base’s low byte at adjusted-base offset 9 and the full traversal base at adjusted-base word offset 0x44. It then reads doubled-lookup-base word offset 4, retaining that value as the final traversal base.

A halfword containing 0x65 is stored at twice the final traversal base. The word at its offset 0x64 is then captured as the final transfer’s second argument.

Before reading the fourth argument from final-traversal-base offset 0x74, the function writes the previously captured adjusted-base offset-0x14 word’s low byte at the earlier traversal base’s offset 0x11, then writes the word 100 at that base’s offset 0x54.

After capturing the fourth argument, the function loads doubled-lookup-base word offset 0x64 as a destination base. **After this load and before either remaining word store**, it writes the final traversal base’s low byte at doubled-lookup-base offset 9.

It then writes the retained final second argument at destination-base word offset 0x54, followed by a word containing the destination base at twice the final second argument. The intervening byte store remains observable even if a later word store overlaps it. It does not trigger a reload of either retained base.

The final four-argument transfer follows these stores. No rollback appears.

## Side effects

### Layout evidence

Offsets describe access patterns, not verified structures.

| Access width | Relative offsets represented |
|---|---|
| Byte reads | 4, 5 |
| Byte writes | 1, 8, 9, 0x0D, 0x10, 0x11, 0x15, 0x19 |
| Word reads | 4, 0x14, 0x40, 0x44, 0x54, 0x64, 0x74 |
| Word writes | 0x14, 0x20, 0x24, 0x34, 0x44, 0x54, 0x74 |

Additional stores use scaled addresses without field offsets.

Fixed effects include byte writes at 0x01, 0x15, 0x21, 0x3A, and 0x67; three byte reads at 0x67; word writes at 0x54 and 0x60; a word read at 0xA9; and the width-unresolved read associated with 0x05.

No distinct-object assumption is justified. Captured values remain earlier observations, while explicit later reads observe memory at their own positions in the sequence.

### Errors and concurrency

No explicit error result, bounds check, handler, retry, cleanup, or recovery protocol is established. A later failure has no established rollback of earlier writes.

No synchronization, atomicity, memory-barrier, thread-safety, or reentrancy guarantee is established. The represented ordering is not independently verified hardware-level ordering.

## Dependencies

The following opaque IDs are documentation handles, not resolved target identities.

| Opaque ID | Interaction | Unresolved aspects |
|---|---|---|
| F-000000000001 | Initial indirect invocation | Implementation, complete signature, effects |
| F-000000000002 | Remaining ordered invocation family before direct mutation | Individual identities, argument/result contracts, secondary channels, effects |
| F-000000000003 | Final four-argument transfer | Target, convention, return behavior |

F-000000000002 groups potentially distinct dependencies. Grouping does not make invocations interchangeable or establish preservation of memory or global state.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Initial forwarding.** For second input 3, the first invocation displays the unchanged first input, 3, and 192.

2. **Shift boundary.** Under 32-bit truncating arithmetic, second-input pattern 0x04000000 yields zero for the shifted third argument while remaining unchanged as the second argument.

3. **Small-base boundary.** If the small-offset producer’s low seven bits are 0x7F, the small offset is 508 and the small base is 1016 under that interpretation.

4. **Fixed-storage capture.** If a later computed or fixed store changes the storage associated with 0x05, accesses using the fixed-storage base still use its earlier capture.

5. **Fixed-write precedence.** An access observer sees the fixed word write at 0x54 before the fixed byte write at 0x15, with the intervening computed stores preserved.

6. **Retained global base.** Overwriting the global’s original storage during the repeated block does not change the retained base used by subsequent groups.

7. **Post-block aliasing.** If 0x3A lies within the word later captured at doubled-lookup-base offset 0x14, that capture observes memory after the byte write. The preceding 0xA9 capture remains earlier.

8. **Fresh fixed-byte reads.** If computed stores change 0x67 between its reads, each copy uses its own read’s byte, not necessarily the initially written byte.

9. **Restoration before read.** If the restoration at doubled-lookup-base offset 0x54 overlaps adjusted-base word offset 0x14, the read occurs after that restoration. The byte store at doubled-lookup-base offset 0x11 occurs afterward and cannot retroactively change the capture.

10. **Separated retained-byte stores.** An observer sees the retained offset-0x74 byte written through the new traversal base before the restoration, but written at doubled-lookup-base offset 0x11 only after the adjusted-base offset-0x14 read.

11. **Last byte-store position.** The destination-base load from doubled-lookup-base offset 0x64 precedes the byte write at doubled-lookup-base offset 9. That byte write precedes both the destination-base offset-0x54 word store and the word store through twice the final second argument.

12. **Partial failure.** If a late traversal faults, earlier completed stores are not undone by this contract, and no platform-specific fault mechanism is prescribed.

## Uncertainties

The following remain **documented_only**:

- Application purpose, intended input domain, and object layouts.
- Completeness of the represented boundary and control flow.
- Complete dependency trace, identities, signatures, and secondary result channels.
- Native pointer width, byte order, alignment requirements, and signed-overflow behavior.
- Validity and meaning of computed and fixed addresses.
- Width of the read associated with 0x05.
- Exact repeated-group count not enumerated here.
- Variable-count signed-right-shift behavior.
- Ownership and resource lifetimes.
- Normal completion and final-transfer return behavior.

The final restoration/read/byte-store interleaving and the last byte store between the destination-base load and remaining word stores are established static relationships, not unresolved alternatives.

## Evidence

This contract is based on the supplied static representation of F-8ab3ed3a4a40. Prior prose, reviewer feedback, comments, and recovered labels are not independent behavioral authority.

Static evidence establishes represented widths, argument forwarding, dependency boundaries, retained captures, computed access patterns, and alias-sensitive ordering. In particular, the final accesses distinguish the two retained-byte stores around a restoration and read, and include the byte store following the last destination-base load.

No dependency implementations, external interface definitions, platform memory map, or runtime observations were supplied. No runtime tests have been performed. No snapshot hash or independently verified provenance is asserted.
