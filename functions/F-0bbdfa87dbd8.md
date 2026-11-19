<!-- Trusted provenance: F-0bbdfa87dbd8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0bbdfa87dbd8

## Purpose

F-0bbdfa87dbd8 performs a long sequence of indirect calls, transforms primary and secondary results, and then performs ordered byte, halfword, and word accesses through computed addresses and shared storage. Its final represented action is an indirect transfer with four arguments.

**Established static evidence:** the supplied representation describes these calls, transformations, memory accesses, and their caller-side ordering.

**Inferred intent:** recurring offsets suggest record-like storage; some constants resemble character values. Neither observation establishes an object model, text format, output stream, or application-level purpose.

**documented_only:** the function’s higher-level purpose, complete dependency contracts, and executable behavior where the static representation is incomplete or arithmetic semantics are unresolved.

This contract describes behavior conditional on the supplied representation accurately reflecting the executable and execution reaching the relevant operation. No runtime tests have been performed.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters and no explicit return value. No success flag, error code, returned allocation, or output length is established.

Effective inputs nevertheless include:

- Incoming machine context used to construct the first indirect target.
- Results of earlier indirect calls, including represented 64-bit results.
- Secondary machine-result words whose producing interfaces are incomplete.
- Shared memory and values subsequently interpreted as addresses.
- The execution environment’s address arithmetic, memory mapping, alignment, and fault rules.

The final transfer receives, in argument order:

1. The constant 0x72.
2. A word loaded late in the function from offset 0x64 of a loaded base.
3. A word saved earlier from offset 0x14 of another computed base.
4. A later word loaded from offset 0x74 of the same loaded base used for the second argument.

A represented return follows this transfer. Whether that boundary is an ordinary returning call in the executable remains documented_only.

### Widths and arithmetic

The representation distinguishes:

| Quantity | Width | Required distinction |
|---|---:|---|
| Byte | 8 bits | A byte store retains the low eight bits |
| Halfword | 16 bits | Occupies two bytes |
| Word | 32 bits | Occupies four bytes |
| Wide call result | 64 bits | Low and high 32-bit portions may be consumed independently |

Explicit unsigned-byte reads widen to values from 0 through 255. Numeric high-word extraction from a 64-bit result does not establish memory endianness.

Address calculations use represented 32-bit quantities. Many shifts and products have signed-looking intermediate types. Exact executable overflow and signed-shift semantics are not established by that notation. Any acceptance example relying on modulo-2³² arithmetic states that condition explicitly.

The dynamic right shift in the final region uses the low eight bits of a computed address value as its count. That gives a represented count range of 0–255, not necessarily 0–31. Count masking, oversized-count behavior, and treatment of negative operands remain documented_only.

### Preconditions and errors

Successful completion requires every reached indirect target and memory access to be valid under the actual execution environment:

- Call targets must support the actual calling convention.
- Reads and writes must have accessible storage of the required width.
- Applicable alignment requirements must be satisfied.
- Values later used as addresses must support those later accesses.

No explicit checks enforce these conditions. Zero or small address values are not rejected before use. There is no established graceful error result, exception translation, rollback, or cleanup path. A fault or nonreturning dependency can leave earlier effects in place.

## Behavior

### Indirect-call phase

The opening caller-side order is a call carrying zero, two calls without represented explicit arguments, another call carrying zero, another call without represented explicit arguments, and a call whose result feeds subsequent transformations.

The first target depends on incoming machine context. Later targets repeatedly use a fixed-displacement construction. Similar construction does not prove identical target identity.

The remaining call phase is straight-line in the supplied representation. Its argument construction includes:

- Left shifts by eight bits, and additional shifts up to 31 bits.
- Independent use of high and low portions of wide results.
- Masking before shifting, masking after shifting, and unsigned right shifts.
- Repeated arguments carrying the same transformed value.
- Explicit zeros and transformations of previously established zeros.
- Reuse of saved results after intervening calls.
- Use of secondary result words without complete callee interfaces.

Calls without represented explicit arguments are not established as side-effect-free or as having complete zero-argument signatures. Dependencies may change globals, mappings, or storage before later caller accesses.

The complete call-by-call argument schedule and target identities are documented_only in this prose contract. The observed phase must not be replaced with a generic value-processing operation on the assumption that its calls are interchangeable.

### Computed bases and initial memory effects

The following descriptive roles identify relationships, not recovered structures or ownership:

- **Wide base:** a preceding call result multiplied by 2²⁵.
- **Compact field:** the low 15 bits of another result after an eight-bit left shift, then logically shifted right by six.
- **Compact base:** twice the compact field.
- **Saved shared word:** a word subsequently read at offset 0x54 from a base obtained by widening the byte at absolute address 0x05.

Under 32-bit truncation, the compact field is at most 508 and is a multiple of four, because the masked shifted value has eight zero low bits. The compact base is consequently at most 1016 and is a multiple of eight.

The first direct accesses establish this ordering:

- A word derived from an earlier result is stored at offset 0x14 from the high-word-derived address of the final preparatory wide result.
- A byte at offset 9 from another saved computed base is written, followed by a word write elsewhere, followed by another byte write to that same offset-9 location with a different represented source.
- The wide-base word at offset 0x14 is cleared before the byte at wide base + 5 is read.
- The compact-base word at offset 0x14 is cleared before that byte’s unsigned value is stored as a word at wide base + 0x34.
- The wide-base word at offset 0x14 is then read.
- The compact-base word at offset 0x14 is cleared again, followed by two consecutive byte writes at compact base + 0x15, both derived from the loaded word.

The loaded wide-base word must not automatically be replaced with zero: intervening accesses can overlap through other computed bases or have environment-specific effects.

### Shared-state and linked-address region

The byte at absolute address 0x05 is read and widened to supply a base. A byte derived from the compact base is then written at absolute address 0x01.

The word at byte-derived base + 0x54 is saved before being overwritten. This saved shared word supplies several later address calculations.

The access pattern includes these established relationships:

- A word loaded from wide base + 0x74 supplies a destination base for offset-0x54 writes.
- Another update has two represented reads at compact base + 0x74: one contributes the destination address, the other the stored word. Their relative operand-evaluation order is not established.
- Offset 0x24 from the saved shared word receives that same saved word twice, with other writes between them.
- Compact base + 0x14 receives a shifted value and is subsequently overwritten twice with the saved shared word.
- Absolute address 0x54 receives a word derived by doubling the saved shared word.
- Absolute address 0x15 receives a byte derived from the compact base.

Further linked-address updates use the saved shared word scaled by four and eight, words read at offsets 0x04, 0x64, and 0x74, and unsigned-byte reads at offset 5. A byte-derived immediate base is limited to 0–255 before its subsequent offset addition; the word loaded through it is not similarly limited.

Repeated offsets include byte locations 0x09, 0x0D, 0x11, 0x15, and 0x19, and word locations 0x14, 0x24, 0x34, 0x44, 0x54, 0x64, and 0x74. These do not establish record sizes, field meanings, or non-overlap between bases.

The complete intermediate access ledger is documented_only here; the relationships above do not authorize coalescing repeated stores or caching repeated reads.

### Constants and repeated publication

For the remaining regions, additional descriptive roles are:

- **Linked word:** a word obtained through a byte-derived offset-0x74 lookup.
- **Doubled linked base:** twice that linked word.
- **Fourfold linked base:** four times that linked word.
- **Adjusted value:** four times the saved shared word, minus 0x96.
- **Workspace base:** four times the adjusted value.
- **Publication base:** the value read from a separately located shared word.

A byte at the fourfold linked base, a halfword at eight times the linked word, and a word at fourfold linked base + 0x20 each receive numeric value 0x3A.

Workspace base + 0x20 subsequently receives four word writes in this order: 0x20, 0x2F, the unsigned value read from workspace base + 0x04, and 0x20 again. The byte read occurs after the first two writes.

These accesses use the same unchanged workspace base. The four-byte destination spans offsets 0x20–0x23; the byte source is at offset 0x04. They do not overlap through ordinary address arithmetic, including modulo-2³² wraparound. Any claim of aliasing between them would require an additional memory-mapping alias or environment behavior, neither of which is established.

The publication-base shared word is read after those writes and before the word 0x20 is written to absolute address 0x60.

The following repeated publication uses four access positions:

| Position | Destination and width | Published value |
|---|---|---|
| First | Offset 0x54 from an eightfold-scaled base; word | Fourfold linked base |
| Second | Publication base + 0x19; byte | Low byte of workspace base |
| Third | Workspace base + 0x11; byte | Low byte of publication base |
| Fourth | Publication base + 0x34; word | Corresponding eightfold-scaled base |

The initial group uses eight times the saved shared word. Subsequent groups use eight times the adjusted value. Their access order is preserved within each group. There is no represented data-dependent termination condition or intervening dependency call. Exact repetition cardinality remains documented_only in this contract.

### Final-region reads and saved values

After the publication sequence, the function reads the word at absolute address 0xA9. It then reads the word at fourfold linked base + 0x74. Only after both reads does it write the byte at absolute address 0x3A, using the low-byte result of eight times the adjusted value plus 0x30.

After that absolute byte write, it saves:

- The word at doubled linked base + 0x14.
- The word at workspace base + 0x04.

These reads are not entry-state snapshots and must not be moved before the write at 0x3A.

The next computed stores include a byte at doubled linked base + 0x0D, byte 0x6C at workspace base + 8, publication-base-derived bytes at workspace base + 0x0D and + 9, and the doubled linked base itself as a word at doubled linked base + 0x54.

The byte at absolute address 0x67 then receives the low byte of the saved 0xA9 word.

### Fresh reads at 0x67 and same-base copying

The function next reads a word at doubled linked base + 0x44. It subsequently reads the byte at 0x67 for the byte write at absolute address 0x21. The represented source is the current byte at 0x67, not a direct reuse of the saved 0xA9 word.

Further stores publish the loaded offset-0x44 word as:

- A byte at workspace base + 0x10.
- A word at publication base + 0x54.
- A shifted byte at doubled linked base + 1.

Publication base + 0x54 is then overwritten with the workspace base.

A word is subsequently read from workspace base + 0x40. After the saved workspace-offset-0x04 word is stored at publication base + 0x74, that offset-0x40 value is stored as a word at workspace base + 0x10. **The source and destination share the same unchanged workspace base.** This is not a transfer between different bases. The word destination also covers the earlier byte destination at workspace base + 0x10.

After another word store at publication base + 0x44, the byte at 0x67 is freshly read for a byte write at workspace base + 9.

Later, after a byte store at doubled linked base + 9, a word store of the saved 0xA9 value at workspace base + 0x44, and a word read at workspace base + 0x54, the byte at 0x67 is freshly read again for another byte write at workspace base + 9.

Thus, following the write at 0x67, there are three distinct represented byte reads there: one for 0x21 and two for later computed-address writes. Intervening computed-address stores may overlap 0x67. The later reads cannot be replaced with the original low byte of the saved 0xA9 word.

### Final linked updates and transfer

The word read at workspace base + 0x54 supplies another destination base. The function writes bytes derived from the earlier fourfold-linked-offset-0x74 word and restores the saved doubled-linked-offset-0x14 word at doubled linked base + 0x54.

It then reads workspace base + 0x14, performs further byte and word updates, and loads a new base from doubled linked base + 0x04.

The final linked effects include:

- A halfword value 0x65 at twice the newly loaded base.
- A word read at that loaded base + 0x64.
- A byte derived from the saved workspace-offset-0x14 word at another base + 0x11.
- A word value of decimal 100 at that base + 0x54.
- A word read at the newly loaded base + 0x74.
- A word read at doubled linked base + 0x64.
- A byte at doubled linked base + 9.
- A word write at offset 0x54 from the last loaded linked address.
- A word write at twice the value loaded from the newly loaded base + 0x64.

The final indirect transfer follows these mutations. Its second and fourth arguments are the saved offset-0x64 and offset-0x74 words from the newly loaded base; its third argument is the earlier saved doubled-linked-offset-0x14 word.

## Side effects

Direct absolute accesses include:

| Address | Established access |
|---|---|
| 0x01 | Byte write |
| 0x05 | Unsigned-byte read |
| 0x15 | Byte write |
| 0x21 | Byte write sourced from a read at 0x67 |
| 0x3A | Byte write |
| 0x54 | Word write |
| 0x60 | Word write |
| 0x67 | Byte write followed by three distinct byte reads |
| 0xA9 | Word read |

Computed accesses may overlap absolute storage or one another. Width and ordering therefore matter even where a destination is later overwritten.

No direct allocation or release operation is identified. Storing address-like words and passing words to dependencies does not establish ownership transfer, retention, or release. Dependency ownership behavior is unknown.

Atomicity, volatility, synchronization, thread safety, reentrancy, memory-mapped device behavior, and storage lifetimes are documented_only. No transaction boundary or rollback is represented.

## Dependencies

F-0bbdfa87dbd8 is the only supplied opaque function identity. No independent opaque IDs are available for indirect targets; this contract does not invent identities or assign semantic dependency names.

| Boundary | Established caller behavior | documented_only |
|---|---|---|
| Initial indirect call | Target derived from incoming context; represented zero argument | Identity, signature, effects, results |
| Intermediate calls | Ordered calls with transformed primary or secondary results | Complete signatures, global preservation, ownership, errors |
| Final indirect transfer | Four arguments beginning with 0x72, after memory mutations | Identity, return behavior, result meaning, additional effects |

No dependency is established as an allocator, validator, formatter, output routine, or cleanup function.

## Acceptance examples

These examples are conditional static requirements, not executed tests.

### Width and unsigned widening

If the wide-base-offset-5 byte read yields 0xFE, its later word publication at wide base + 0x34 carries numeric value 254. A byte destination receiving a wider value ending in 0xAB stores only the eight-bit pattern 0xAB.

### Same-base copy

If the workspace-offset-0x40 read yields 0x12345678, the later word store at workspace base + 0x10 receives that saved word. The destination uses the same base as the source. A later change to the source does not change the already saved value.

### Final read ordering

If a computed address for the saved doubled-linked-offset-0x14 word overlaps 0x3A, its read occurs after the write at 0x3A. Acceptance must use memory at that later point, not an entry-state value.

### Fresh global-byte reads

If an intervening computed store changes 0x67 between the write at 0x21 and a later workspace-offset-9 write, that later write uses the newly read byte. It need not match the byte previously written at 0x21 or the low byte of the saved 0xA9 word.

### Non-overlapping workspace offsets

Assuming ordinary modulo-2³² addressing without mapping aliases, the word writes at workspace base + 0x20 cannot modify the byte at workspace base + 0x04. Their byte offsets differ by 0x1C through 0x1F, none congruent to zero modulo 2³².

### Observable overwritten stores

If workspace base + 0x20 denotes observable device storage, the required word-write history is 0x20, 0x2F, widened byte-read result, 0x20. Checking only the final value is insufficient.

### Scaling and interruption

Under 32-bit truncation, multiplication by 2²⁵ preserves only the low seven source bits in the resulting pattern. If a dependency does not return, later caller accesses need not occur; prior effects are not specified to be undone.

## Uncertainties

The following remain documented_only:

- Accuracy of recovered function boundaries and indirect control flow.
- Dependency identities, full argument schedules, signatures, and secondary-result conventions.
- Signed overflow, multiplication, and shift semantics not fixed by the representation.
- Dynamic right-shift behavior for negative operands or oversized counts.
- Endianness, alignment, faults, and memory-mapping aliases.
- Object layouts, ownership, allocation boundaries, lifetimes, and synchronization.
- Exact publication repetition count and intermediate access details not exhaustively enumerated here.
- Application-level purpose and any consumer-facing success criterion.

The same-base offset-0x40-to-offset-0x10 transfer, final global-read ordering, fresh reads at 0x67, and arithmetic non-overlap of the unchanged-workspace offsets 0x20 and 0x04 are established relationships, not unresolved alternatives.

## Evidence

This contract is based solely on the supplied static representation and opaque identity mapping. The prior draft, reviewer feedback, and source comments are not independent behavioral authority.

Established evidence consists of represented call boundaries, operand widths, saved-value relationships, explicit reads and writes, and caller-side ordering. No dependency bodies or runtime observations were supplied. No runtime tests have been performed. No snapshot hash or independently verified provenance is asserted.
