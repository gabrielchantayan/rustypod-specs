<!-- Trusted provenance: F-5adc5e81c3dd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5adc5e81c3dd

## Purpose

F-5adc5e81c3dd executes a long ordered chain of unresolved indirect invocations, transforms selected results, performs fixed-address and computed-address memory mutations, and finishes with an unresolved four-argument indirect transfer.

**Established static evidence:** the supplied representation identifies argument and result widths, arithmetic transformations, access widths, saved-value dependencies, and significant read/write ordering. It does not establish an application-level purpose.

**[INFERENCE] Intent:** repeated offset accesses and reciprocal-looking address publications may represent state construction or linkage maintenance. No particular structure, protocol, allocator, formatter, or device interface is established.

This contract describes static behavioral constraints, not guaranteed successful execution for arbitrary inputs.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit body use. |
| Second argument | 32 bits | No explicit body use. |
| Third argument | Signed 32 bits | Shifted left by six; the resulting 32-bit quantity supplies an early invocation and a later masked extraction. |

The lack of explicit uses of the first two arguments does not establish independence from incoming execution state. Hidden dependency inputs and the actual calling convention remain unresolved.

### Implicit inputs

Execution also depends on incoming execution-link state, execution-position-derived invocation targets, primary and secondary dependency-result channels, existing memory, a later-loaded global word, and the terminal transfer target.

The explicit arguments alone therefore do not determine the complete behavior.

### Outputs and completion

No explicit return value is established. Observable outputs consist of dependency effects, direct memory writes, and the terminal invocation.

A return is represented after the terminal invocation. Whether the actual transfer returns normally or requires a special convention is **documented_only**. No success flag, error code, or recovery interface is represented.

### Preconditions and ownership

Every reached invocation requires a callable target with a compatible actual convention. Every reached memory operation requires accessible storage at its computed address, with support for its width and alignment. This includes low fixed addresses and an unaligned-looking word access at 0xA9.

No corresponding validity checks are visible.

Allocation, release, ownership transfer, reference counting, and lifetime extension are not established. Publishing an address does not establish ownership. Unknown dependencies must not be assumed to preserve globals or allocate, retain, or release memory.

## Behavior

### Invocation preparation and arithmetic

The preparation region contains a sequential chain of indirect invocations without visible loops or conditional branches. The first target derives from incoming execution-link state plus 0xFFE; subsequent targets generally derive from execution-position-related values plus that displacement.

Invocation order matters. Omitted explicit arguments are not equivalent to explicit zeros, and an invocation with no represented arguments is not established as pure or input-free. Results may be consumed immediately or retained across intervening invocations. Represented 64-bit results supply independently consumed low and high 32-bit portions. Secondary result channels also supply arguments.

An early four-argument invocation receives zero, an unresolved secondary-channel value, the third argument shifted left by six, and zero. A later invocation receives an extraction made by retaining the shifted quantity’s low 20 bits, logically shifting right by ten, and shifting left by six. Under truncating 32-bit arithmetic, this selects original argument bits 4–13 and places them at output bits 6–15.

Later transformations mostly concern dependency results, not the third argument. They include 32-bit left shifts, logical right shifts, masks ranging from 11 to 24 bits, successive width-limited transformations, and power-of-two scaling. Unsigned 64-bit right shifts by 34, 36, and 44 are also represented; they must not be interpreted as 32-bit shifts.

Word-sized arithmetic is represented at 32 bits. Byte and halfword stores narrow to eight and sixteen bits respectively. Actual signed-overflow, signed-shift, address-overflow, and excessive-shift-count behavior remains **documented_only**. Numerical examples below state their arithmetic assumptions.

Dependencies may mutate memory, fault, diverge, or produce external effects before any direct data-memory store occurs.

### Independent dataflow roles

The following names describe values, not object types or ownership:

| Role | Definition |
|---|---|
| **Opening base** | Low 32-bit portion of an earlier dependency-result pair, shifted left by 25 and retained across later invocations. |
| **Shifted-base precursor** | A separate, later primary dependency result. |
| **Shifted base** | Shifted-base precursor multiplied by 0x02000000. |
| **Compact extraction** | A later primary result shifted left by eight at represented 32-bit width, masked to 15 bits, then logically shifted right by six. |
| **Compact base** | Twice the compact extraction. |
| **Saved shifted word** | Another earlier primary result shifted left by eight. |
| **Final result pair** | Low and high 32-bit portions of the last represented 64-bit result before direct mutation. |
| **Retained state word** | A word read through a zero-extended fixed-storage byte before that location is replaced. |

The opening base and shifted base have independent dependency-result origins. Their common scale of 2²⁵ does not make them interchangeable.

Byte offsets include 0x01, 0x05, 0x08, 0x09, 0x0D, 0x11, 0x15, and 0x19. Word offsets include 0x04, 0x14, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74. Offset 0x10 receives both byte and word writes. These accesses do not establish one common structure or a minimum object extent.

### Opening mutation and early fixed storage

The first direct word store publishes the **opening base** at the final result pair’s high half plus 0x14. Its source is the retained low half of the earlier result pair shifted left by 25—not the later shifted-base precursor.

The same opening base supplies the address for two byte stores at offset nine. The first receives the low byte of the final pair’s high half; the second receives the low byte of the opening base. Between those byte stores, a word store publishes the separate **shifted base** at the final pair’s low half multiplied by 256, plus 0x54.

The shifted base’s word at 0x14 is then cleared. Its byte at offset five is read and zero-extended. A zero word is written at compact base plus 0x14 before the byte-derived word is published at shifted base plus 0x34. The word at shifted base plus 0x14 is subsequently reread.

**Ordinary-memory consequence:** in stable byte-addressed memory, after successful accesses, this reread is zero. The only intervening stores are the compact-base zero-word store and the byte-derived word store 0x20 bytes beyond the cleared word. The former cannot introduce nonzero bytes, even if it overlaps the cleared word; the latter does not overlap it. The represented intervening stores therefore do not justify a nonzero reread. Device semantics, external mutation, or a nonordinary address model remain separate unresolved possibilities.

After the reread, another compact-base word clear precedes two identical byte stores at compact base plus 0x15, each using the reread’s low byte.

Only then is fixed byte 0x05 read and zero-extended into an address base. This read precedes the write of the compact base’s low byte to fixed address 0x01. The saved byte-derived base is used after that fixed write; it is not freshly read for each subsequent access.

The ensuing computed mutations publish a byte at byte-derived base plus 0x15, briefly publish the saved shifted word’s low byte at shifted base plus 0x11, publish the saved shifted word through the reread value plus 0x54, and clear shifted base plus 0x11.

The word at byte-derived base plus 0x54 is then captured as the retained state word before replacement with the shifted-base-plus-0x14 reread value. Earlier mutations may already have affected fixed byte 0x05 before its designated read.

### Middle mutation and fixed-storage interleaving

The middle region follows words at offset 0x74, performs self-derived word stores, and uses doubled, quadrupled, and octupled retained-state values as addresses or payloads. Repeated reads and repeated writes remain distinct accesses.

Compact base plus 0x34 first receives the compact extraction. A word reached through shifted base plus 0x74 receives the shifted-base precursor shifted left by 26, and later the saved shifted word. Other stores publish a reached word through its own plus-0x54 address and write the retained state word through its own plus-0x24 address.

Fixed word 0x54 is written with twice the retained state word **between** a compact-base-plus-0x14 store of the precursor shifted left by 26 and two compact-base-plus-0x14 stores of the retained state word.

Two compact-base-plus-0x15 byte stores then receive the low byte of four times the retained state word. Fixed byte 0x15 next receives the low byte of the compact base, before subsequent scaled-address updates.

Later operations include:

- Publishing scaled retained-state values at offsets 0x14, 0x34, 0x44, and 0x54.
- Doubling a word obtained through compact base plus 0x74.
- Reading bytes at offset five and using their unsigned values as bases for offset-0x74 word reads.
- Publishing narrowed reached-word values at offsets nine, 0x0D, and 0x15.
- Reading offset 0x64 and subsequently using the reached value as another base.
- Repeating offset-five byte reads after intervening mutations rather than reusing an earlier byte.

These are address and dataflow relationships, not evidence of a valid list, pointer provenance, or disjoint objects.

### Constants and repeated publication

A later region writes numeric 0x3A at three widths: a byte at a fourfold-scaled reached-word address, a halfword at the corresponding eightfold-scaled address, and a word at the fourfold-scaled address plus 0x20.

For the publication region:

- **Adjusted index** is four times the retained state word minus 0x96.
- **Publication base** is four times the adjusted index.
- **Loaded base** is a global word loaded after the following constant sequence.
- **Published payload** is four times the word reached by the later byte-derived offset-0x74 read.
- **Terminal linked base** is twice that reached word, retained before publication.

Publication base plus 0x20 receives word values in this order: 0x20, 0x2F, a freshly read byte from publication base plus four widened to 32 bits, and 0x20.

Only after that sequence is the loaded-base global read. Fixed word 0x60 is then written with 0x20 before repeated publication begins. Later overlap with the global’s storage does not imply a reload of the saved loaded base.

The first publication group has four ordered effects:

1. Publish the payload as a word at eight times the retained state word plus 0x54.
2. Publish the publication base’s low byte at loaded base plus 0x19.
3. Publish the loaded base’s low byte at publication base plus 0x11.
4. Publish eight times the retained state word as a word at loaded base plus 0x34.

Subsequent groups preserve this four-write order but substitute eight times the adjusted index for both the scaled destination base in the first effect and the final word payload.

This is explicit finite repetition, not a data-dependent loop. Exact multiplicity is **documented_only** in this contract. The repeated accesses must not be treated as a single final-state assignment.

### Terminal snapshots and fresh fixed-byte reads

The terminal region first captures the word at fixed address 0xA9 and then the word at published payload plus 0x74. It next writes fixed byte 0x3A with the low-eight-bit result of eight times the adjusted index plus 0x30.

Only after that byte write are terminal linked base plus 0x14 and publication base plus four read and saved. Any overlap with fixed byte 0x3A affects these later snapshots under the actual byte order.

Computed byte stores then update terminal linked base plus 0x0D and publication base plus eight, 0x0D, and nine. The plus-eight byte receives 0x6C. A word store publishes the terminal linked base through its own plus-0x54 address.

The saved 0xA9 word’s low byte is written to fixed address 0x67. A word is then read at terminal linked base plus 0x44, **before** a fresh read of fixed byte 0x67 supplies the write to fixed byte 0x21.

The newly read offset-0x44 word supplies a byte at publication base plus 0x10, a word at loaded base plus 0x54, and a shifted byte at terminal linked base plus one. The right-shift count is the publication base’s low eight bits; signed and excessive-count behavior is unresolved. Loaded base plus 0x54 is subsequently overwritten with the publication base.

A word captured from publication base plus 0x40 is later written at publication base plus 0x10. Between that read and write, loaded base plus 0x74 receives the earlier publication-base-plus-four snapshot. Loaded base plus 0x44 then receives the terminal linked base.

Two later byte stores to publication base plus nine each freshly read fixed byte 0x67. The first follows the loaded-base-plus-0x44 store. Before the second, terminal linked base plus nine is updated, the saved 0xA9 word is published at publication base plus 0x44, and publication base plus 0x54 is read.

There are therefore three distinct fresh reads of 0x67 in this terminal region. Conversely, the later word publication of the 0xA9 value uses its saved snapshot, not a fresh word read.

### Final mutation and transfer

The saved published-payload-plus-0x74 word supplies narrowed byte fields. The saved terminal-linked-base-plus-0x14 word is written at terminal linked base plus 0x54.

Publication base plus 0x14 is freshly read before further byte and word updates. Its low byte later supplies a field through the word obtained from publication base plus 0x54.

A word read at terminal linked base plus four supplies a new base. Halfword constant 0x65 is written at twice that base before its offsets 0x64 and 0x74 are read. Further mutations include word constant 100 at an offset-0x54 destination and linkage-like stores involving a later terminal-linked-base-plus-0x64 read.

The terminal invocation receives four represented 32-bit arguments:

1. Constant 0x72.
2. The word read at offset 0x64 from the base obtained through terminal linked base plus four.
3. The saved terminal-linked-base-plus-0x14 word.
4. The word read at offset 0x74 from that same later base.

The fourth argument is not the earlier published-payload-plus-0x74 snapshot.

## Side effects

Direct writes use byte, halfword, and word widths. Computed addresses may overlap each other or fixed storage; mixed-width fields may overlap. Duplicate and overwritten writes remain distinct represented effects.

Overlap must be evaluated against the actual intervening operations. In particular, general alias uncertainty does not invalidate the ordinary-memory zero result of the cleared shifted-base-plus-0x14 reread.

Saved bases and snapshots remain saved values despite later overlapping writes. Fresh reads—including fixed byte 0x05 and the three terminal reads of 0x67—observe memory at their own trace positions.

Byte order, alignment requirements, device behavior, atomicity, and hardware ordering guarantees are not established.

No explicit bounds checks, rollback, cleanup, synchronization, barriers, or error reporting appear. A fault or nonreturning dependency may leave completed effects visible. Transactionality, reentrancy, and thread safety are not established.

## Dependencies

F-5adc5e81c3dd is the only supplied opaque function identity. No opaque identities are supplied for the indirect dependencies or terminal target; none are invented.

Their complete contracts remain **documented_only**, including actual calling conventions, hidden inputs, result-channel interpretation, global preservation, ownership effects, external I/O, exceptional completion, and normal-return guarantees.

A represented 64-bit result establishes low/high dataflow in the supplied representation, not necessarily a higher-level numeric return type.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

### Independent opening sources

Assume truncating 32-bit arithmetic, an earlier result-pair low half of 1, and a separate shifted-base precursor of 2. The opening base is 0x02000000; the shifted base is 0x04000000.

The first direct word store must publish 0x02000000 through the final high half plus 0x14. The intervening scaled-low-half word store must publish 0x04000000. Both opening offset-nine byte stores address 0x02000009. Substituting the later precursor’s scaled value for the first word-store source fails this criterion.

### Cleared-word reread

Assume stable ordinary byte-addressed memory, successful accesses, and no external mutation. The shifted-base-plus-0x14 reread must yield zero even if the intervening compact-base zero-word store overlaps it. The word store at shifted base plus 0x34 does not justify a nonzero result.

This criterion does not prescribe read results for device memory or externally modified storage.

### Arithmetic and narrowing

Under truncating 32-bit arithmetic, third argument 16 produces initial shifted value 1024 and early extracted value 64. Third argument 0x4000 produces initial value 0x100000 and extracted value zero.

A byte-store source with pattern 0x123456AB publishes 0xAB; a halfword store narrows to sixteen bits.

### Fixed-storage ordering

If a computed write after fixed byte 0x01 changes fixed byte 0x05, subsequent byte-derived addressing still uses the earlier 0x05 read.

If compact base plus 0x14 overlaps fixed word 0x54, the fixed write must remain between the specified compact-base word updates.

### Saved bases and terminal snapshots

If fixed word 0x60 overlaps the loaded-base global, publication uses the value captured before that fixed write.

If fixed byte 0x3A overlaps the later terminal-linked-base-plus-0x14 snapshot, that snapshot reflects the preceding byte write under the applicable byte order. The earlier 0xA9 snapshot remains unchanged as a saved value.

### Fresh reads and partial completion

If intervening computed writes modify fixed byte 0x67, each later fresh read uses the byte then present. Reusing the initial saved 0xA9 low byte is not equivalent.

A fault after completed writes requires no invented rollback or error return. Reaching the terminal target requires the stated saved-versus-later-read argument distinctions.

## Uncertainties

The following remain **documented_only**:

- Application purpose, intended structures, and state invariants.
- Indirect dependency identities and complete invocation contracts.
- Full invocation-trace enumeration and exact publication multiplicity in this contract.
- Actual calling convention, hidden inputs, and result representation.
- Signed arithmetic, overflow, address conversion, and problematic shift behavior.
- Byte order, alignment, permissions, device semantics, and external mutation.
- Computed storage validity, extent, lifetime, and ownership.
- Concurrency guarantees and dependency effects.
- Terminal transfer convention and guaranteed return.
- Fidelity of the supplied static representation to executable behavior.

These limitations do not erase established dataflow distinctions or ordering constraints.

## Evidence

The sole behavioral evidence is the supplied static representation for F-5adc5e81c3dd. Embedded comments, the prior document, and reviewer feedback are not independent proof.

The opening-source distinction follows from retaining an earlier result pair’s low half shifted by 25 while separately deriving the shifted base from a later primary result. The cleared-word conclusion follows from the widths, values, and destinations of the only two intervening stores.

Other established claims concern represented widths, constants, offsets, saved-value relationships, and relative access ordering. Application intent is labeled as inference; unresolved environmental and dependency behavior remains documented.

No implementation access, dependency inspection, runtime execution, or runtime tests were performed.
