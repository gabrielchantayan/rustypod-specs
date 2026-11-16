<!-- Trusted provenance: F-cc87f81ca81f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cc87f81ca81f

## Purpose

F-cc87f81ca81f performs an ordered sequence of unresolved indirect invocations, transforms values obtained from those invocations, and then modifies interconnected storage using mixed-width accesses. Its last displayed action is an unresolved indirect transfer with four supplied arguments.

No application-level purpose is established. The evidence does not identify formatting, initialization, decoding, allocation, or container management. Character-like constants and address-shaped values do not establish those interpretations.

This contract distinguishes:

- **Established static evidence:** operations, widths, and ordering visible in the supplied representation.
- **Inferred intent:** explanatory interpretations, not additional behavioral guarantees.
- **documented_only:** unresolved behavior for which this contract supplies no invented implementation rule.

The representation contains interface and control-flow uncertainty. Obligations below are conditional on execution reaching the corresponding operations with meaningful target semantics.

## Interface

### Entry and result

The displayed interface has no explicit parameters and no explicit result. Nevertheless, it consumes implicit execution state:

- An entry-carried control value that contributes to the first indirect destination.
- An entry-carried 32-bit value shifted left by 12 and supplied twice to an early invocation.
- Explicit and secondary call-produced values.
- Existing contents of fixed-address and dynamically addressed storage.

The function therefore is not established as independent of caller state.

Displayed invocation results include 32-bit values and 64-bit aggregates. Aggregate values are numerically separated into low and high 32-bit words. This establishes numerical extraction, not byte order or a verified calling convention.

A return appears after the terminal indirect transfer. Whether that transfer returns, and whether the displayed entry/result interface accurately describes the original target, remain **documented_only**.

### Widths and layout

| Quantity or access | Width | Established interpretation |
|---|---:|---|
| Byte read | 8 bits | Unsigned where displayed as a byte |
| Byte write | 8 bits | Low eight bits of the supplied value |
| Halfword write | 16 bits | Includes constants 0x003a and 0x0065 |
| Word read/write | 32 bits | Numeric payload or address carrier |
| Aggregate result | 64 bits | Source of numerical word and field extraction |

Address calculations predominantly use 32-bit quantities. Mapping those quantities to storage, required alignment, and permitted access widths are **documented_only**.

Observed relative offsets include 1, 4, 5, 8, 9, 0x0d, 0x10, 0x11, 0x14, 0x15, 0x19, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74. These identify access locations, not a recovered structure layout. A word at offset 0x14 overlaps a byte at offset 0x15; a word at offset 0x10 overlaps a byte at offset 0x11.

### Preconditions for normal traversal

Every reached indirect destination must support the effective invocation convention. Every reached memory location must permit its observed access. Entry-carried and secondary call-produced values must be meaningful when consumed.

Normal traversal also requires any necessary low-address mappings and support for the displayed overlapping, mixed-width accesses. No visible checks enforce these requirements. Null rejection, bounds validation, alignment repair, and fallback behavior are not established.

## Behavior

### Call-driven phase

Execution first follows a long straight-line sequence of indirect invocations. The initial destination depends on entry state. Many subsequent destinations combine embedded control-location values with displacement 0xffe. Their dependency identities are unresolved.

Result-dependent argument preparation includes:

- Shifting a 32-bit result left by eight.
- Supplying a shifted aggregate low word followed by its unchanged high word.
- Supplying additional shifts, masked fields, unsigned extractions, zeros, or secondary call-produced values.
- Repeating the same prepared value in multiple argument positions.
- Retaining values across intervening invocations for later use.

Displayed left-shift counts range from **1 through 31**. In particular, an argument preparation masks a value to 19 bits and shifts that masked value left by one. The minimum displayed count is therefore not two. This observation does not establish that every count within the range occurs.

Masks select fields of several widths, including 11, 12, 13, 14, 15, 17, 18, 19, 20, 21, 22, and 24 bits. Their application meaning is unresolved.

Invocation ordering is observable. Calls with unused explicit results, no displayed arguments, or all-zero arguments must not be treated as effect-free. Unknown callees may modify shared state, retain arguments, affect external resources, fail, or never return.

No displayed conditional branch or success-code gate separates this phase from direct memory mutation. Progress requires preceding invocations to return normally.

### Arithmetic interpretation

Under a **conditional 32-bit bit-vector interpretation**, arithmetic results used as words retain only their low 32 bits. Left shifts discard higher bits; unsigned right shifts insert zeros. Address addition and scaling must not silently become arbitrary-precision calculations.

Consequences under that interpretation include:

- Multiplication by 0x02000000 preserves only the input’s low seven bits in the result.
- A value shifted left by 25 has a zero low byte.
- Successive shifts can remove all information even when each individual count is below 32.
- Unsigned extraction from a 64-bit aggregate at bit positions 34, 36, and 44 corresponds to its high word shifted right by 2, 4, and 12 respectively.

These are conditional arithmetic properties, not assertions of portable signed-language semantics.

A final-phase byte uses a signed word shifted right by a count obtained from another value’s low byte. Counts at least 32 are not excluded by the displayed expression alone. Signed-right-shift behavior and excessive-count handling remain **documented_only**.

### Initial direct-memory publication

The following role names are explanatory labels, not recovered identifiers:

- **Shifted carrier:** an earlier result shifted left by 25.
- **Scaled base:** another earlier result multiplied by 0x02000000.
- **Compact index:** a result shifted left by eight, masked to 15 bits, then shifted unsigned right by six.
- **Compact base:** twice the compact index.
- **Final pair:** the low and high words of the last preparatory aggregate.

Direct mutation begins by storing the shifted carrier as a word at final-pair-high plus 0x14. Next, a byte at shifted-carrier plus 9 receives the low byte of final-pair-high. A word at final-pair-low multiplied by 256 plus 0x54 then receives the scaled base. The byte at shifted-carrier plus 9 is subsequently overwritten with the shifted carrier’s low byte.

The scaled-base word at offset 0x14 is cleared before a byte is sampled at offset 5. The compact-base word at offset 0x14 is then cleared, followed by publication of the sampled byte as a zero-extended word at scaled-base plus 0x34. Only after that publication is the scaled-base word at offset 0x14 read.

Another compact-base word clear at offset 0x14 precedes two byte writes at compact-base plus 0x15, both using the low byte of the sampled word.

The sampled word cannot unconditionally be substituted with zero: intervening destinations may overlap its storage. Duplicate byte writes are separately displayed effects.

### Graph mutation and low-address storage

Subsequent effects combine compact-base fields, scaled addresses, saved words, and values reached through offsets 0x54, 0x64, and 0x74.

The visible behavior includes saving an offset-0x54 word before replacing it; publishing address-derived values through offsets 0x14, 0x24, 0x34, 0x44, and 0x54; and storing low-byte fragments through offsets 9, 0x0d, 0x11, and 0x15. Some destinations receive their own address-derived value. Such stores establish no ownership relationship.

An offset-5 byte is used directly to select a word at byte-value plus 0x74. No additional table base is established. Later offset-5 sampling is fresh: previous samples cannot generally replace it across intervening stores.

Fixed-address effects include a byte write at 1, a byte write at 0x15, a word write at 0x54, and a later word write of 0x20 at 0x60. Storage associated with address 5 also supplies an address-contributing value.

Fixed-address and dynamic storage are not established as disjoint. A dynamic byte store can affect a later word read, including a read interpreted as an address.

### Constant publication and repeated cross-linking

A graph-derived value supplies three destinations receiving 0x3a:

- A byte at its fourfold-scaled address.
- A halfword at its eightfold-scaled address.
- A word at its fourfold-scaled address plus 0x20.

A different fourfold-scaled base plus 0x20 receives successive word values 0x20, 0x2f, a zero-extended byte freshly read from that base plus 4, and 0x20 again. The read occurs after the 0x2f store and before the sampled-byte publication. These effects are not equivalent to a single final store without additional memory assumptions.

For the remaining description:

- **Saved link** is the word saved before an earlier offset-0x54 replacement.
- **Adjusted value** is four times the saved link, minus 0x96.
- **Related base** is four times the adjusted value.
- **Selected value** is the later word obtained through byte-value plus 0x74.
- **Graph node** is twice the selected value.
- **Current scaled address** is four times the selected value.
- **Global-carried address** is sampled from global address-carrier storage before repeated cross-linking.

These arithmetic definitions are subject to the width qualifications above.

After sampling the global-carried address, the fixed-address word at 0x60 receives 0x20. An initial cross-link publication stores the current scaled address at eight times the saved link plus 0x54. Byte publications then connect the related base and global-carried address through offsets 0x19 and 0x11, followed by a word publication of eight times the saved link at global-carried address plus 0x34.

The recurring form instead uses eight times the adjusted value for both the destination base at offset 0x54 and the value published at global-carried address plus 0x34. Its four-store order is word, byte, byte, word.

The representation shows static repetition, not a demonstrated data-dependent loop. Exact repetition multiplicity is **documented_only** in this contract; the repeated effects must not be collapsed into one publication.

### Final snapshots interleaved with mutation

The final region does **not** take all snapshots before further writes. Its opening observable order is:

1. Read a word at fixed address 0xa9.
2. Read a word at current-scaled-address plus 0x74.
3. Write a byte at fixed address 0x3a.
4. Read a word at graph-node plus 0x14.
5. Read a word at related-base plus 4.

The byte at 0x3a is derived from the adjusted value: under the conditional bit-vector interpretation, it is the low byte of eight times that value plus 0x30.

Call the four saved words the **fixed snapshot**, **scaled snapshot**, **saved graph word**, and **saved related word**, respectively. The 0x3a write precedes both latter reads. If that byte destination overlaps either sampled word, the saved value must reflect the write. In particular, the saved graph word later becomes the terminal transfer’s third argument.

### Fixed-byte publication and fresh reads

After those snapshots, byte publications affect graph-node plus 0x0d and related-base plus 8, 0x0d, and 9. They publish, respectively, the related base’s low byte, constant 0x6c, and two copies of the global-carried address’s low byte. A word at graph-node plus 0x54 then receives the graph-node address.

Fixed byte storage at **0x67** receives the fixed snapshot’s low byte. The function next reads the graph-node word at offset 0x44. It then **freshly reads byte storage at 0x67** for publication at fixed address 0x21.

The offset-0x44 word drives a byte publication at related-base plus 0x10, a word publication at global-carried-address plus 0x54, and the signed-shift byte at graph-node plus 1. The global-carried-address word at offset 0x54 is subsequently overwritten with the related base.

A fresh related-base word read at offset 0x40 supplies a word publication at related-base plus 0x10. Between that read and publication, global-carried-address plus 0x74 receives the saved related word. Global-carried-address plus 0x44 then receives the graph-node address.

**Only after these stores**, byte storage at 0x67 is read again for the first later byte publication at related-base plus 9. That read cannot be replaced with the original fixed snapshot’s low byte: intervening dynamic stores may have changed 0x67.

Following that publication, graph-node plus 9 receives the related base’s low byte, and related-base plus 0x44 receives the complete fixed snapshot. A word is then freshly read from related-base plus 0x54; call it the **late link**. **After these effects**, 0x67 is read once more for a second byte publication at related-base plus 9.

Thus the three post-publication reads of 0x67 are distinct observations: one supplies 0x21, and two supply later writes at related-base plus 9. No equality among their values is guaranteed.

### Final linkage and transfer

The scaled snapshot’s low byte is published at late-link plus 0x0d. Graph-node plus 0x54 receives the saved graph word. A word is then sampled at related-base plus 0x14, before the scaled snapshot’s low byte is published at graph-node plus 0x11.

Related-base plus 9 and plus 0x44 subsequently receive the late link’s low byte and full word. A fresh word at graph-node plus 4 supplies another address carrier. A halfword 0x0065 is written at twice that carrier before its offset-0x64 word is read.

The sampled related-base offset-0x14 word contributes a byte at late-link plus 0x11. Late-link plus 0x54 receives word value 100. A word is then read from the new carrier’s offset 0x74, followed by a fresh graph-node offset-0x64 read.

Final linkage publishes the new carrier’s low byte at graph-node plus 9, the carrier’s offset-0x64 word at the node selected by graph-node offset 0x64 plus 0x54, and that selected node’s address at twice the carrier’s offset-0x64 word.

The terminal indirect transfer receives four arguments, in order:

1. Constant 0x72.
2. The word read from the new carrier’s offset 0x64.
3. The saved graph word sampled after the fixed-address 0x3a write.
4. The later word sampled from the new carrier’s offset 0x74.

No rollback is displayed before this transfer.

## Side effects

Direct effects include shared-memory reads and writes, repeated stores, overlapping-width mutation, and low-address access. Every indirect invocation may add effects not characterized here.

No allocation, release, reference-count change, ownership transfer, or lifetime extension is established. Unknown callees are not assumed to preserve globals, retain memory, or release memory.

Reentrancy, thread safety, atomicity, and idempotence are not established. Repeated execution may consume previously modified fields.

No explicit error result, recovery handler, retry, or rollback path is displayed. Invalid accesses, dependency failure, or nontermination can prevent completion after partial effects. Their precise mechanisms are **documented_only**.

## Dependencies

The only supplied opaque function identity is **F-cc87f81ca81f**. No separate opaque function IDs resolve the indirect destinations; this contract does not invent target identities.

Unresolved dependency roles comprise the entry-derived initial destination, preparatory invocation destinations, producers of secondary call values, and the terminal destination.

Their effective signatures, shared-state effects, ownership rules, failure behavior, and termination properties are **documented_only**.

## Acceptance examples

These are conditional specification examples, not executed tests.

### Width-preserving forwarding

For an aggregate with low word 0x12345678 and high word 0x9abcdef0, forwarding the low word shifted left by eight and the unchanged high word yields 0x34567800 and 0x9abcdef0 under 32-bit bit-vector semantics. An untruncated first argument of 0x1234567800 is unacceptable.

### Compact-index boundary

The compact-index transformation produces multiples of four from 0 through 508. The compact base therefore ranges from 0 through 1016 in multiples of eight. An index of 511 does not satisfy the transformation.

### Unsigned sampling and overlap

A sampled byte of 0xe7 becomes word 0x000000e7 when zero-extended, not 0xffffffe7. A subsequent byte update at offset 0x15 overlaps a word at offset 0x14; the resulting numerical word depends on byte order.

### Snapshot ordering

Suppose fixed address 0x3a overlaps a byte of the graph-node word at offset 0x14, and the computed byte changes that storage. The saved graph word must include the changed byte. Sampling that word before the write, or moving the write after the snapshot, violates the specified ordering and can change terminal argument three.

### Fresh fixed-byte observations

Suppose the initial 0x67 publication is 0x12, a dynamic store before the first later related-base-plus-9 publication changes 0x67 to 0x34, and the related-base-plus-0x44 word publication subsequently changes it to 0x56. If no other relevant writes intervene, those two byte publications must consume 0x34 and 0x56 respectively, not two copies of 0x12.

### Preserving intermediate effects

An observer of the four word publications at the constant-publication destination must observe 0x20, 0x2f, the sampled byte extended to a word, and 0x20, in that order. Retaining only the final value loses established effects.

## Uncertainties

The following remain **documented_only**:

- Application purpose and valid entry-state domain.
- Accuracy of the apparent entry/result interface and execution path.
- Indirect target identities and invocation conventions.
- Origins and lifetimes of secondary call values.
- Byte order, address mapping, alignment, and signed arithmetic semantics.
- Validity and meaning of low-address storage.
- Structure boundaries and alias relationships.
- Exact multiplicity of repeated cross-link publications.
- Terminal target, return behavior, and continuation.
- Fault, exception, dependency-failure, and nontermination behavior.

Exact intermediate values cannot generally be reduced using disjoint-storage assumptions. The fixed-address 0x3a write and fresh 0x67 reads are established ordering requirements, not unresolved details.

## Evidence

This contract uses only the supplied static representation. No tools, independent implementation access, or runtime tests were used.

Static evidence supports the displayed invocation sequence, numerical transformations, access widths, ordered snapshots and publications, repeated stores, fresh fixed-byte reads, and terminal argument ordering. It does not establish application semantics or a complete executable target model.

Role names are explanatory only. Embedded comments, prior prose, and reviewer feedback are not treated as independent proof. Provenance is left to the publishing system.
