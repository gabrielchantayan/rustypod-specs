<!-- Trusted provenance: F-aed51d3fa722; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aed51d3fa722

## Purpose

F-aed51d3fa722 executes a long sequence of indirect calls and scalar transformations, performs ordered memory mutations, and ends with a four-argument indirect control transfer.

The supplied static representation does not establish an application-level purpose. Repeated offsets, scaled addresses, and character-valued constants do not establish initialization, serialization, parsing, rendering, or object management.

This contract distinguishes:

- **Established static evidence:** calls, arithmetic, access widths, and ordering explicitly represented in the supplied material.
- **Conditional deductions:** consequences under an ordinary 32-bit machine-arithmetic interpretation.
- **documented_only unknowns:** behavior not established, including dependency semantics and execution-environment requirements.

No success value, transactional guarantee, or memory-ownership protocol is established.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters and no explicit return value. Normal return is represented only after the final indirect transfer returns.

Implicit inputs include:

- An incoming return-link value used to construct the first call target.
- An incoming 32-bit register value used in an early shifted argument.
- Register-carried values exposed after calls without a complete result convention.
- Existing memory and globally accessible state.
- An unresolved final control-transfer target.

The exact calling convention, preserved-register set, stack requirements, and complete implicit input set are **documented_only**.

Intermediate calls supply zero through four represented explicit arguments. Some results are represented as combined 64-bit quantities subsequently separated into numerical low and high 32-bit portions. This does not establish whether a dependency returns one 64-bit scalar or multiple register-carried outputs.

### Widths and layout

| Quantity or access | Represented width | Required interpretation |
|---|---:|---|
| Byte read or write | 8 bits | A byte store retains the low eight bits. |
| Halfword write | 16 bits | Two bytes are affected, even for a small constant. |
| Word read or write | 32 bits | Four-byte accesses carry scalar or address-like values. |
| Combined call result | 64 bits | High and low portions identify numerical bit positions. |
| Address-like arithmetic | Apparently 32 bits | Actual address-space and overflow semantics remain unresolved. |

Offsets describe byte displacements, not typed object fields. No object sizes, array bounds, packing rules, or non-aliasing guarantees are established.

Unsigned byte reads expanded into words produce values from 0 through 255. High/low result extraction does not establish memory byte order.

### Preconditions and errors

Reaching normal return requires every reached call target and memory access to be valid for the actual execution environment. Required conditions include executable targets, compatible calling conventions, suitable access permissions, and supported alignment.

Low absolute addresses must be accessible where used. Address-like values loaded from memory must support their subsequent uses.

The body shows no explicit validation, error status, rollback, or recovery path. Null, small, overlapping, or misaligned destinations are not rejected before use. A fault or non-returning dependency can prevent normal return after earlier effects have occurred.

## Behavior

### Execution and observability

The visible body has three broad stages:

1. Indirect calls interleaved with scalar transformations.
2. Direct memory reads and writes.
3. A final indirect transfer.

No explicit conditional branch, data-dependent loop, retry, or early error return is represented in the body. Dependencies may have their own control flow.

The represented ordering of calls and separate memory operations is significant. Repeated stores, overwritten values, and repeated reads cannot be discarded merely because they appear redundant. Computed locations may overlap, accesses may have device semantics, and individual accesses may fault.

Where one expression contains multiple independent reads, the representation does not by itself settle their machine-level evaluation order.

### Indirect calls and arithmetic

The first target is constructed by adding 0xFFE to the incoming return-link value. Later targets are repeatedly constructed from embedded address-like anchors using the same displacement. Distinct call sites are not established as distinct services or as calls to one shared service.

Arguments include recent results shifted left by eight bits, high result portions passed unchanged or shifted, duplicated arguments, explicit zeros, masked fields, and values exposed through incompletely described result conventions. A call with no represented explicit arguments may still consume implicit machine-state inputs.

Discarded results do not make calls removable: dependency effects and failures are unresolved.

Field selection includes masks spanning 1, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, and 24 bits. These do not establish named flags, lengths, encodings, or protocol fields.

Under a 32-bit bit-vector interpretation, each left shift loses bits beyond bit 31. Consequently:

- Left shifts by 25 and then seven bits yield zero.
- Left shifts by eight, then 18, then eight bits yield zero.
- Two successive 28-bit left shifts yield zero.
- Left shifts by 31 and then 30 bits yield zero.

These are conditional machine-width deductions, not guarantees of portable signed arithmetic. Signed-looking overflow and negative-value shifts remain unresolved.

Logical shifts of entire unsigned 64-bit results by 34, 36, or 44 bits yield values equivalent to shifting the numerical high word right by two, four, or twelve bits respectively.

### Opening direct-memory mutations

The following descriptive labels identify arithmetic relationships, not objects:

- **Shifted base:** an earlier retained result multiplied by 2²⁵.
- **Compact base:** twice a value obtained by shifting a result left eight bits, masking with 0x7FFF, and logically shifting right six bits.
- **Opening byte base:** an earlier result shifted left 25 bits.

Under 32-bit bit-vector arithmetic, the compact base ranges from zero through 1016 in multiples of eight. This range does not make its accesses valid.

The opening memory group writes an earlier shifted word at offset 0x14 from the high word of the last combined result. It then writes a byte at opening-byte-base offset nine, writes a retained word at offset 0x54 from 256 times the last result’s low word, and overwrites the earlier byte destination using a **different source expression**. The two byte values are not established as numerically different.

The shifted-base offset-0x14 word is then cleared before a byte is read at shifted-base offset five. The compact-base offset-0x14 word is cleared before that byte is stored, zero-extended, at shifted-base offset 0x34. A word read from shifted-base offset 0x14 follows. The compact-base clear is repeated, followed by two byte stores of the read word’s low byte at compact-base offset 0x15.

The next group reads an unsigned byte at absolute address 0x05 and uses its value directly as a base. A byte store to absolute address 0x01 follows that read. The resulting 0–255 base participates in accesses at offsets 0x15 and 0x54; it is not established as an allocation or table pointer.

### Linked and scaled updates

Subsequent accesses combine word reads at offsets four, 0x14, 0x44, 0x54, 0x64, and 0x74 with byte reads at offset five. Stores use byte and word widths across computed destinations, including bases scaled by two, four, or eight.

A word previously read from the byte-derived base’s offset 0x54 becomes an important retained base. Several later stores publish that retained base or its scaled forms. Self-associated stores do not establish links, reference counts, parent pointers, or ownership.

One nested store reads compact-base offset 0x74 separately for its destination calculation and stored value. Those are distinct represented reads; their relative evaluation order is not established by that expression.

Later, an unsigned byte read at offset five is used directly as the base for a word read at offset 0x74. No entry-stride multiplication intervenes. The resulting word is doubled for later destinations. Additional offset-five reads occur after intervening mutations and must remain separate.

Direct low-address effects include a word store at absolute address 0x54 and a byte store at absolute address 0x15.

### Constants and publication-base capture

For the remaining stages, use these descriptive relationships:

- **Linked word:** the word obtained through the later byte-based offset-0x74 read.
- **Doubled linked base:** twice that linked word.
- **Adjusted base:** four times the retained base, minus 0x96.
- **Adjusted four-times base:** four times the adjusted base.
- **Publication base:** a word loaded from the publication-source global.

Numeric value 0x3A is written with three different widths: a byte at four times the linked word, a halfword at eight times that word, and a word at offset 0x20 from four times that word.

At adjusted-four-times-base offset 0x20, successive word stores contain 0x20, 0x2F, a zero-extended byte read from offset four of that base, and finally 0x20. The byte read occurs after the 0x2F store and before its corresponding word store.

**The publication-source global is read after the final offset-0x20 store and before the word store of 0x20 at absolute address 0x60.** Publication stores begin afterward. The global load cannot be moved across the absolute store without additional environmental guarantees. If the store overlaps the global, publication still uses the value captured before that store.

### Repeated publication

Each publication group has this access order:

- A word store at offset 0x54 of an eight-times destination.
- A byte store of the adjusted four-times base’s low byte at publication-base offset 0x19.
- A byte store of the publication base’s low byte at adjusted-four-times-base offset 0x11.
- A word store of the eight-times destination base at publication-base offset 0x34.

The offset-0x54 store carries four times the linked word. The first group uses eight times the retained base as its destination base. Following groups use eight times the adjusted base.

The publication base and byte-valued publications are retained scalar values, not fresh global reads for each group. Nevertheless, all represented stores remain observable operations. The repetitions are explicit, not a data-dependent loop. Their exact multiplicity is **documented_only** here; deduplication is not asserted equivalent.

### Final global reads and captures

After publication, the routine reads a word at **absolute address 0xA9**, then reads a word at offset 0x74 from four times the linked word. Both precede the byte store at **absolute address 0x3A**. That store contains the low byte of eight times the adjusted base plus 0x30.

Only after the 0x3A store does the routine capture, in order:

- A word at doubled-linked-base offset 0x14.
- A word at adjusted-four-times-base offset four.

The first capture is later restored to doubled-linked-base offset 0x54 and supplied as the third final-transfer argument. The second capture is later written to publication-base offset 0x74.

Before publishing a byte to 0x67, the routine writes low-byte portions of retained bases at doubled-linked-base offset 0x0D and adjusted-four-times-base offsets 0x0D and nine. It also writes literal byte 0x6C at adjusted-four-times-base offset eight and temporarily stores the doubled linked base at its own offset 0x54.

### Separate reads of absolute address 0x67

The routine writes the low byte of the captured 0xA9 word to **absolute address 0x67**. Subsequent uses involve **three separate byte reads**, not unconditional reuse of that captured byte.

The first read follows a word read at doubled-linked-base offset 0x44 and directly supplies the byte store at **absolute address 0x21**.

Before the second 0x67 read, intervening effects include:

- A byte store of the offset-0x44 word at adjusted-four-times-base offset 0x10.
- Two word stores at publication-base offset 0x54, first containing that word and then the adjusted four-times base.
- A shifted byte store at doubled-linked-base offset one.
- A word read at adjusted-four-times-base offset 0x40.
- The captured offset-four word stored at publication-base offset 0x74.
- The offset-0x40 word stored at adjusted-four-times-base offset 0x10.
- The doubled linked base stored at publication-base offset 0x44.

The second 0x67 read supplies a byte store at adjusted-four-times-base offset nine. It is followed by a byte store at doubled-linked-base offset nine, a word store of the captured 0xA9 word at adjusted-four-times-base offset 0x44, and a word read at adjusted-four-times-base offset 0x54.

The third 0x67 read then supplies another byte store at adjusted-four-times-base offset nine.

Intervening computed writes may overlap 0x67. Therefore, neither later read is guaranteed to return the byte originally written there, nor are the three reads guaranteed equal.

The shifted byte store uses the word read at doubled-linked-base offset 0x44 and a shift count taken from the low eight bits of the adjusted four-times base. Signed right-shift behavior and counts outside 0–31 are **documented_only**.

### Remaining mutations and final transfer

The captured word from four-times-linked-word offset 0x74 contributes low-byte stores at offset 0x0D of the base loaded from adjusted-four-times-base offset 0x54, and at doubled-linked-base offset 0x11.

The earlier doubled-linked-base offset-0x14 capture is restored to doubled-linked-base offset 0x54. A separate word read at adjusted-four-times-base offset 0x14 supplies a later byte store. The adjusted-four-times-base offset-nine byte and offset-0x44 word are then overwritten using the base loaded from its offset 0x54.

A word loaded from doubled-linked-base offset four supplies a destination for a halfword store of 0x65 at twice that value. Later word reads from that loaded base’s offsets 0x64 and 0x74 remain distinct accesses. Another word is read from doubled-linked-base offset 0x64.

The final mutations include a word store of decimal 100 at offset 0x54 of the earlier offset-0x54-derived base, a byte store at doubled-linked-base offset nine, a computed offset-0x54 word store, and a word store to twice the word loaded from offset 0x64.

The unresolved final target receives, in order:

1. Numeric value 0x72.
2. The word loaded from offset 0x64 of the base obtained through doubled-linked-base offset four.
3. The earlier doubled-linked-base offset-0x14 capture.
4. The later word loaded from offset 0x74 of the same base used for the second argument’s read.

No explicit result is consumed. If the target returns normally, F-aed51d3fa722 returns without an explicit value.

## Side effects

Direct effects include 8-, 16-, and 32-bit writes to computed destinations and low absolute addresses. Global reads, including the publication-source load and the separate accesses at 0xA9 and 0x67, are part of the represented access ordering.

Dependency effects are unknown. No assumption is made that dependencies preserve globals, allocate or release memory, or leave address mappings unchanged.

No direct allocation, release, ownership-transfer declaration, synchronization, or rollback mechanism is established. Absence of these operations in the visible body does not establish their absence inside dependencies.

Atomicity, reentrancy, thread safety, and idempotence are not established.

## Dependencies

**F-aed51d3fa722** is the only supplied opaque function ID. No separately identified dependency is available.

All intermediate indirect targets and the final target therefore remain unresolved dependencies without invented IDs or semantic roles. Call-site separation does not establish target separation.

Their calling conventions, implicit inputs, global effects, ownership behavior, failures, and return behavior are **documented_only**.

## Acceptance examples

These are conditional static acceptance criteria, not runtime results.

### Equal opening byte values

If the two different source expressions for the opening offset-nine stores both yield low byte zero, both stores remain required. Acceptance must not require their values to differ.

### Unsigned expansion and widths

If an unsigned byte read yields 0xFE, its corresponding zero-extended word store contains 254, not −2. The 0x3A halfword store still affects two bytes; replacing it with a byte store is unacceptable.

### Publication-source ordering

In an environment where the absolute word store at 0x60 changes the publication-source global, publication uses the value read before that store. Reloading afterward and using the changed value would violate the represented ordering.

### Final capture boundary

If the 0x3A byte store overlaps a byte of either subsequent capture, that capture observes memory after the store, subject to the environment’s memory semantics. The 0xA9 word read and computed offset-0x74 read occur before that store.

### Changing 0x67 observations

If the three 0x67 reads return 0x11, 0x22, and 0x33 respectively, the 0x21 store receives 0x11, and the two corresponding adjusted-four-times-base offset-nine stores receive 0x22 and 0x33. Replacing all three reads with the originally captured low byte is unacceptable.

### Intermediate visibility

A trace-capable environment observes the offset-0x20 word publications of 0x20, 0x2F, the byte-derived word, and 0x20 in order. A single final store does not preserve that trace.

### Failure or non-return

An inaccessible reached destination carries no clean-error or rollback guarantee. If the final dependency does not return, the enclosing routine need not return; earlier completed effects remain subject to the memory environment.

## Uncertainties

The following remain **documented_only**:

- Application purpose, object roles, and intended invariants.
- Completeness and execution fidelity of the supplied representation.
- ABI, implicit register inputs, and result conventions.
- Dependency identities, effects, ownership behavior, and return behavior.
- Address-space width, byte order, alignment, and low-address mapping.
- Signed overflow and variable-shift semantics.
- Aliasing, volatility, device semantics, concurrency, and intra-expression read ordering.
- Allocation lifetimes and release responsibilities.
- Exact multiplicity of repeated publication.

These unknowns do not justify moving separately ordered global reads across writes or replacing fresh reads with retained values.

## Evidence

This contract is based solely on static inspection of the supplied representation. Prior prose, comments, and reviewer feedback are not independent proof of behavior.

Established evidence includes the represented interface, indirect calls, numerical transformations, explicit access widths, repeated mutations, global-read ordering around publication, the final-stage 0xA9 and 0x67 accesses, and the four-argument final transfer.

Truncation and compact-base range statements are conditional 32-bit arithmetic deductions. Application intent and dependency semantics remain unresolved.

No runtime tests, execution traces, target-resolution checks, or memory-map validation have been performed.
