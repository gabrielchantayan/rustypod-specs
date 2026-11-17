<!-- Trusted provenance: F-6acff232ad8d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6acff232ad8d

## Purpose

F-6acff232ad8d performs an ordered sequence of indirect invocations and fixed-width transformations, followed by computed-memory and fixed-address mutations, and a terminal unresolved indirect invocation.

**Established static evidence:** the supplied representation exposes argument transformations, memory-access widths, value dependencies, and operation ordering. It does not establish a recognizable application-level purpose or a validated calling convention.

**[INFERENCE]** Recurring field offsets and reciprocal-looking stores suggest interconnected record-like storage. They do not establish a list, tree, allocator, serializer, device protocol, or ownership model.

This contract describes represented behavior conditional on execution reaching each operation. Unresolved target semantics and incomplete details are marked `documented_only`; they are not replaced with conventional assumptions.

## Interface

### Explicit inputs

| Input | Represented width | Visible use |
|---|---:|---|
| First input | 32 bits | No explicit use |
| Second input | 32 bits | No explicit use |
| Third input | Signed 32 bits | Shifted left by 24 positions for two early invocation arguments |

Under a 32-bit bit-pattern interpretation, the third input contributes only its low eight bits, placed in bits 24–31. This is not a claim that the represented signed shift has portable overflow semantics.

The first two inputs are not proven irrelevant to actual execution. Unresolved calling conventions may expose incoming machine state beyond the arguments shown.

### Implicit inputs and outputs

Execution additionally depends on:

- The incoming return-link value.
- Ordinary indirect-invocation results and additional result-register contents.
- Existing fixed-address and computed-memory contents.
- The unresolved terminal target.

The first invocation target is the incoming return-link value plus 0xFFE. Subsequent targets are generally formed from fixed continuation-like values plus that displacement. Executability, instruction-state requirements, and dependency identities are `documented_only`.

No explicit return value is represented. Outputs are invocation events, argument bit patterns, memory accesses, mutations, and the terminal transfer. A return appears after the terminal invocation, but whether that invocation returns is unknown.

### Preconditions and ownership

No validation of pointers, lengths, alignment, targets, or arithmetic ranges is visible. Progress requires every reached invocation and access to be supported by the execution environment. Low addresses and values derived from individual bytes are used as addresses; ordinary protected-process assumptions are insufficient.

No allocation, release, reference counting, ownership transfer, or lifetime guarantee is established. Unknown callees must not be assumed to preserve globals or to retain or release memory.

## Behavior

### Ordering and arithmetic

The visible body has an invocation phase, a direct-memory phase, and a terminal invocation. No explicit conditional branch, retry, bounds check, or recovery path appears. This is represented ordering, not proof that all operations execute: any unresolved invocation may fail, diverge, or transfer control elsewhere.

Access widths are 8-bit bytes, 16-bit halfwords, and 32-bit words. Most arithmetic and address-forming quantities are represented at 32 bits. Some invocation results are represented at 64 bits and consumed through separate low and high 32-bit portions.

Byte stores retain eight bits; halfword stores retain sixteen. Bytes widened to words are zero-extended. Unsigned right shifts are logical. Signed overflow, negative signed left shifts, and exceptional shift counts remain `documented_only`.

Conditional arithmetic examples below use low-32-bit truncation after each 32-bit operation. Sequential truncation matters: two successive left shifts by 28 do not preserve a wider intermediate and produce zero under this convention.

Visible extraction widths include:

| Transformation | Information retained |
|---|---:|
| Low 24 bits, then right shift by 8 | 16 bits |
| Low 20 bits, then right shift by 10 | 10 bits |
| Low 22 bits | 22 bits before scaling |
| Low 19 bits | 19 bits before scaling |
| Low 17 bits, then right shift by 4 | 13 bits |
| Unsigned 64-bit right shift by 44 | At most 20 bits |

### Invocation phase

The invocation sequence repeatedly passes shifted results, explicit zeros, masked fields, saved earlier results, and additional-register results. Observable forms include duplicated arguments, low words shifted by eight, and high words passed directly or after further shifting.

The third explicit input supplies its 24-position-shifted contribution twice. In the first invocation it follows a dependency result shifted by eight; in the second it precedes another such dependency-derived argument. This establishes argument positions, not equivalent dependency behavior.

Invocations without represented arguments remain observable boundaries even when their ordinary results are unused. Likewise, arguments derived from quantities explicitly set to zero do not make the receiving invocation side-effect-free.

The complete per-site invocation identity and argument trace are `documented_only` here. No dependency equivalence is inferred from matching argument patterns.

### Direct-memory terminology and layout

The following descriptive quantities identify value dependencies, not recovered object types:

- **Compact index:** a result shifted left by eight, masked to 15 bits, then logically shifted right by six. It ranges from 0 to 508 in multiples of four.
- **Doubled index:** twice the compact index, ranging from 0 to 1,016 in multiples of eight.
- **Large scaled base:** an earlier result multiplied by 2²⁵.
- **Saved shifted base:** an earlier low result word shifted left by 25.
- **Saved shifted word:** another earlier result shifted left by eight.
- **Early global base:** the 32-bit value obtained from storage associated with address 5.
- **Captured scalar:** the word later read at offset 0x54 from the early global base.
- **Quadrupled scalar:** four times the captured scalar.
- **Adjusted scalar:** the quadrupled scalar minus 0x96.
- **Publication base:** four times the adjusted scalar.
- **Repeated-region base:** the word captured from a separate global after the transient stores described below.

Offsets are byte displacements. Visible byte fields include offsets 1, 4, 5, 8, 9, 0x0D, 0x10, 0x11, 0x15, and 0x19. Word fields include 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74. Offset 100 means 0x64, not 0x100.

No bases are established as disjoint. Fixed addresses may alias computed fields, and narrow accesses may overlap word accesses.

### Opening memory effects

The first direct-memory effects occur in this order:

1. The saved shifted base is stored as a word at offset 0x14 from the final invocation’s high result word.
2. That high word’s low byte is stored at offset 9 from the saved shifted base.
3. The large scaled base is stored at offset 0x54 from the final low result word multiplied by 256.
4. The offset-9 byte destination is overwritten with the saved shifted base’s low byte.
5. The large scaled base’s offset-0x14 word is cleared, followed by a byte read at its offset 5.
6. The doubled index’s offset-0x14 word is cleared.
7. The byte just read is widened into the large scaled base’s offset-0x34 word.
8. The large scaled base’s offset-0x14 word is read.

The doubled index’s offset-0x14 word is then cleared again. Its offset-0x15 byte receives the low byte of the last word read, twice.

Neither the word read following the clear nor the repeated stores may be replaced by assumed constants or a final-state-only description.

### Early global read and fixed-address mutations

The storage associated with address 5 is read **after** those duplicated offset-0x15 byte stores and **before** the byte write to fixed address 0x01. Its captured value becomes the early global base. The underlying storage width is not declared in the supplied material; its conversion to a 32-bit base is visible.

Fixed address 0x01 receives the low byte of the doubled index. Subsequently:

- The early global base’s offset-0x15 byte receives the compact index’s low byte.
- The large scaled base’s offset-0x11 byte receives the saved shifted word’s low byte.
- The previously read offset-0x14 word is used as a base for an offset-0x54 store of the saved shifted word.
- The large scaled base’s offset-0x11 byte is cleared.
- The early global base’s offset-0x54 word is captured as the captured scalar, before that field is overwritten with the earlier offset-0x14 read.
- The compact index is published at the doubled index’s offset 0x34.

Later computed accesses load the large scaled base’s offset-0x74 word, publish through that loaded value, and access the doubled index’s offset-0x74 word. The latter source appears more than once in a single store expression; separate occurrences must not universally be treated as one cached value. Their intra-expression read order is unresolved.

The captured scalar is stored into its own offset-0x24 field twice, interleaved with byte publications through another loaded base.

The fixed word at **0x54 receives twice the captured scalar**. This occurs after the doubled index’s offset-0x14 word receives an earlier result shifted by 26, and before that field receives the captured scalar twice.

The doubled index’s offset-0x15 byte then receives the quadrupled scalar’s low byte twice. **Fixed address 0x15 next receives the doubled index’s low byte**, before the subsequent offset-0x54 store through the quadrupled scalar and offset-0x34 publication through the doubled index.

These are ordered mutations with data dependencies, not merely a list of touched addresses.

### Computed-record mutations and transient stores

The intervening computed-memory region performs word and byte publications using the captured scalar, its two-, four-, and eight-times-scaled derivatives, and loaded field values. Important boundaries include:

- Loading offset-0x74 words before scaling them.
- Loading a byte at offset 5, zero-extending it, and using it as the base for an offset-0x74 word read.
- Loading offset-0x64 words before further mutations.
- Re-reading byte sources after intervening writes.
- Storing loaded values into their own offset-0x14 fields.
- Overwriting offset-0x44 and offset-0x54 fields after earlier publications.

Define the **selected word** as the later offset-0x74 word read through a freshly loaded, zero-extended offset-5 byte. Twice this word is the **selected doubled base**; four times it is the **selected publication value**.

The constant 0x3A is stored as a byte at four times the selected word, as a halfword at eight times that word, and as a word at offset 0x20 from four times that word. The scalar adjustment by subtraction of 0x96 is formed between the byte and halfword stores.

The publication base’s offset-0x20 word then receives four stores, in order:

1. 0x20.
2. 0x2F.
3. A zero-extended byte freshly read at offset 4 from the publication base.
4. 0x20.

After the fourth store, the separate global word supplying the repeated-region base is read. **Only after that read does fixed word address 0x60 receive 0x20.** Neither global capture may be moved across these boundaries without an additional equivalence argument covering aliasing and external effects.

### Repeated publications

After the fixed store at 0x60, the initial publication group:

- Stores the selected publication value at offset 0x54 from eight times the captured scalar.
- Stores the publication base’s low byte at the repeated-region base’s offset 0x19.
- Stores the repeated-region base’s low byte at the publication base’s offset 0x11.
- Stores eight times the captured scalar at the repeated-region base’s offset 0x34.

Subsequent represented groups preserve the same four-store order, but use **eight times the adjusted scalar** for both the offset-0x54 destination base and the offset-0x34 published word.

The byte values and repeated-region base are captured quantities, not global reloads within each group. Nevertheless, the stores remain distinct accesses. The region is explicitly repeated, not controlled by an input count. Exact multiplicity is `documented_only` in this contract.

### Post-region capture and mutable fixed byte storage

Immediately after the repeated region:

- The word at fixed address **0xA9 is captured**.
- The word at offset **0x74 from the selected publication value** is captured.
- Fixed address **0x3A receives the low byte of eight times the adjusted scalar plus 0x30**.

Thus the 0xA9 capture precedes both that offset-0x74 load and the 0x3A store.

Next, the selected doubled base’s offset-0x14 word and the publication base’s offset-4 word are captured. Computed byte mutations publish the low bytes of the publication and repeated-region bases; the publication base’s offset-8 byte receives 0x6C, and the selected doubled base is published into its own offset-0x54 word.

**Fixed address 0x67 then receives the captured 0xA9 word’s low byte.** A word is subsequently read from offset 0x44 of the selected doubled base. **Address 0x21 then receives a fresh byte read from 0x67**, rather than a guaranteed reuse of the captured word’s low byte.

Further computed-memory writes use the loaded offset-0x44 word and the publication base. One byte store uses that signed word shifted right by a count equal to the publication base’s low eight bits. Exceptional count behavior remains unresolved.

The following reload boundaries are significant:

- After intervening word reads and writes, including an offset-0x10 word overwrite, the publication base’s offset-9 byte receives a **fresh read of 0x67**.
- After another byte mutation, the full captured 0xA9 word is stored at the publication base’s offset 0x44.
- The publication base’s offset-0x54 word is then read to obtain a later base.
- Its offset-9 byte receives **another fresh read of 0x67**.

These reads may differ because intervening computed stores can alias address 0x67. The full captured word remains a saved value; the mutable byte at 0x67 is not interchangeable with it.

### Final mutations and terminal invocation

The low byte of the captured post-region offset-0x74 word is published at offset 0x0D from the later base and offset 0x11 from the selected doubled base. The saved offset-0x14 word is restored through the selected doubled base’s offset 0x54.

A fresh word read at offset 0x14 from the publication base supplies a later offset-0x11 byte publication. Additional stores overwrite the publication base’s offset-9 byte and offset-0x44 word using the later base.

Near termination:

- A word read at offset 4 from the selected doubled base supplies an address-forming value.
- A halfword 0x65 is stored at twice that value.
- Words at offsets 0x64 and 0x74 from that value are captured, with intervening byte and word writes.
- The later base’s offset-0x54 word receives decimal 100.
- A separate offset-0x64 read from the selected doubled base supplies a publication destination.
- The first offset-0x64 capture is stored through that destination’s offset 0x54.
- A word containing the destination value is stored at twice the first offset-0x64 capture.

The terminal invocation receives four represented arguments:

1. 0x72.
2. The first final offset-0x64 capture.
3. The saved post-region offset-0x14 word from the selected doubled base.
4. The final offset-0x74 capture.

No application-level or ownership meaning is established.

## Side effects

Observable effects include unresolved callee effects, repeated and overlapping stores, fixed-address mutations, dependent reads, and indirect control transfer.

The function is not established as pure, transactional, idempotent, thread-safe, or reentrant. No rollback, synchronization, atomic-access guarantee, error return, or exception translation is visible. Failure may leave every previously completed effect in place.

Endianness, alignment tolerance, device-memory semantics, and external modification are unknown. Narrow stores do not inherently replace complete neighboring words. Final memory contents alone cannot validate the access sequence.

## Dependencies

F-6acff232ad8d is the only supplied resolved opaque function ID. No resolved opaque IDs are available for its indirect dependencies.

Dependency identities are therefore `documented_only`, including the entry-link-derived invocation, continuation-derived sequence, result-producing invocations, and terminal target. No private dependency names or invented resolved IDs are assigned.

Calling conventions, global effects, ownership behavior, termination, and errors are unknown. Similar arguments do not prove two sites invoke the same dependency.

## Acceptance examples

All examples are conditional analytical checks, not executed tests.

- **Third-input truncation:** under 32-bit bit-pattern shifts, inputs 0x000000AB and 0x123456AB both contribute 0xAB000000 at the two stated argument positions. Whole-function equivalence does not follow.

- **Compact-index boundary:** low seven result bits of 0x7F produce compact index 508 and doubled index 1,016. The byte stores to 0x01 and later 0x15 therefore carry 0xF8. Both stores retain their distinct ordering boundaries.

- **Captured scalar:** if the early global base’s offset-0x54 read yields 0x12345678, fixed word 0x54 receives 0x2468ACF0 under the stated truncation convention. A later change to the source field does not change that saved scalar.

- **Byte widening:** an offset-5 byte of 0xE1 supplies word value 0x000000E1, not 0xFFFFFFE1.

- **Transient publication:** an access-sensitive observer distinguishes offset-0x20 stores of 0x20, 0x2F, the freshly read byte widened to a word, and 0x20. The repeated-region global read follows all four.

- **Adjusted fixed byte:** adjusted scalar 1 produces byte 0x38 at address 0x3A; adjusted scalar 0x20 produces 0x30 after byte truncation.

- **Captured word versus reload:** if the captured 0xA9 word is 0x123456E1, its publication to 0x67 carries 0xE1. If supported aliasing or external effects subsequently change 0x67, later reads for 0x21 and computed offset-9 stores use the then-current byte. The later full-word publication still uses captured value 0x123456E1.

- **Interrupted execution:** a nonreturning indirect invocation imposes no requirement that later stores occur. No visible fallback supplies replacement results.

## Uncertainties

The following remain `documented_only`:

- Actual calling convention and completeness of the explicit interface.
- Indirect dependency identities and complete per-site invocation trace.
- Reliability and meaning of additional-register results.
- Application purpose, record types, and valid address domains.
- Underlying declaration width of storage associated with address 5.
- Signed overflow, signed-shift semantics, and exceptional shift counts.
- Ordering between reads within a single multi-read expression.
- Exact repeated-region multiplicity.
- Endianness, alignment, atomicity, global aliasing, and external modification.
- Callee ownership, lifetime, global-preservation, and error behavior.
- Whether the terminal invocation returns.
- Fidelity of the represented control flow to the original execution.

## Evidence

This contract uses static inspection of the supplied representation of F-6acff232ad8d. Established evidence consists of represented widths, transformations, memory offsets, constants, captures, reload boundaries, value dependencies, and cross-operation ordering.

**[INFERENCE]** Record-like organization is suggested by recurring offsets and cross-publications; no application interpretation is established.

No implementation access, dependency inspection, runtime execution, smoke run, or runtime tests have been performed. Acceptance examples are analytical consequences of explicit assumptions, not observed results.
