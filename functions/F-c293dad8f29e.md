<!-- Trusted provenance: F-c293dad8f29e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c293dad8f29e

## Purpose

F-c293dad8f29e performs an ordered sequence of mixed-width memory accesses, derives further addresses from memory contents and entry-state values, and transfers control to an unidentified indirect destination.

The supplied static representation establishes memory manipulation, not an application-level purpose. It does not establish text formatting, object initialization, collection traversal, or a recognizable dispatch protocol. Character-like constants do not establish textual intent.

This contract describes the visible access relationships and their ordering. It does not certify the accuracy of the represented function boundary, arithmetic semantics, or indirect control flow.

## Interface

### Explicit inputs and output

The visible interface contains four 32-bit input slots.

| Input | Established use |
|---|---|
| First | Address base for writes at offsets 0x01, 0x15, and 0x54. |
| Second | Address base for reads at offset 0x74 and writes at offset 0x14; its low byte is also stored through the first input. |
| Third | No explicit use in the visible body. |
| Fourth | Uninterpreted 32-bit payload written to two early destinations. |

No return value is specified. The terminal transfer is presented as an indirect call followed by return, but the destination’s actual calling convention and return behavior are unresolved.

### Additional required state

Four uninitialized 32-bit entry-state values are consumed. This contract calls them **entry bases A, B, C, and D**. These are descriptive labels, not established public parameters.

A separate global word supplies another address base. The indirect destination also requires state whose origin is not established.

For compact description, the following names denote captured values or arithmetic relationships, not additional memory objects:

| Name | Meaning |
|---|---|
| Scaled base | Four times B. |
| Doubled-entry base | Twice D. |
| Doubled-derived base | Twice the word read from doubled-entry-base offset 0x74. |
| Lookup source | Word read from scaled-base offset 0x04. |
| Late lookup value | Word obtained by the second byte-indexed lookup through the lookup source. |
| Doubled-lookup base | Twice the late lookup value. |
| Target base | Four times the late lookup value. |
| Adjusted base | Scaled base minus 0x96. |
| Publication base | Four times the adjusted base. |
| Global base | Captured value of the separate global word. |

These names do not imply valid pointers, disjoint storage, allocation boundaries, or ownership.

### Widths, layouts, and arithmetic

Offsets are byte offsets. Accesses distinguish 8-bit bytes, 16-bit quantities, and 32-bit words. Byte lookup values are zero-extended; byte stores retain the low eight bits. A 16-bit store occupies two bytes, not two copies of its low byte.

No complete object layout, alignment guarantee, or endianness is supplied. A word access beginning at an offset can overlap several byte fields described elsewhere.

Address arithmetic is represented using 32-bit quantities. The scaling terminology identifies the represented dependency; it does not establish portable signed-overflow behavior. Modulo-\(2^{32}\) address arithmetic requires an explicit machine-bitvector interpretation. The signed right shift near the end has a potentially out-of-range count; no masking rule is established.

A complete execution requires every accessed address to support its access width and direction, including low absolute addresses and addresses derived directly from bytes. No visible bounds, null, alignment, or error checks establish these conditions.

## Behavior

### Observation model

The contract concerns an ordered access trace, not merely final memory contents.

Earlier writes may alter later reads through aliasing. Separately described destinations need not be distinct. Captured values remain captured unless an explicit later read replaces them. Repeated writes and reads cannot be collapsed when their multiplicity is observable.

The ordering below follows the supplied representation. It is not a claim about hardware transactions, concurrent visibility, or compiler treatment of ordinary memory.

### Early linking and scaling

The initial operation reads the word at second-input offset 0x74 to obtain a destination base and writes a word at that base’s offset 0x54. The stored value comes from another read of second-input offset 0x74. The representation does not establish which of those two source reads occurs first; unequal observations could therefore affect the destination and value independently.

The early linking sequence then has these ordered relationships:

- B is written to B’s offset 0x24, its low byte to A’s offset 0x09, A to first-input offset 0x54, and the fourth input to A’s offset 0x54.
- The B self-reference and byte write through A are repeated.
- Second-input offset 0x14 receives twice C. First-input offset 0x54 then receives **twice B**, replacing its earlier A value if storage is unchanged.
- Second-input offset 0x14 receives B, first-input offset 0x01 receives the second input’s low byte, and second-input offset 0x14 receives B again.

There is no later explicit write of B itself to first-input offset 0x54.

Two consecutive byte stores put the scaled base’s low byte at doubled-entry-base offset 0x15. First-input offset 0x15 then receives the doubled-entry base’s low byte.

Scaled-base offset 0x54 receives the fourth input. Doubled-entry-base offset 0x34 receives the scaled base. The word at doubled-entry-base offset 0x74 is then read and doubled, producing the doubled-derived base. After that read, scaled-base offset 0x54 receives the scaled base itself, scaled-base offset 0x15 receives the low byte of eight times B, and B’s offset 0x44 receives the doubled-derived base.

These intermediate writes are observable effects, not an atomic initialization.

### Byte-indexed lookup and linked storage

A byte read at scaled-base offset 0x05 supplies a zero-extended address component. A word is read at that byte value plus 0x74. There is no independent table base and no four-byte index scaling. Call the resulting word the **early lookup base**.

Early-lookup-base offset 0x44 receives the doubled-derived base. Scaled-base offset 0x15 then receives the scaled base’s low byte. The early lookup base’s low byte is stored at scaled-base offset 0x0D, followed by scaled-base self-references at offsets 0x14 and 0x54.

A word read at scaled-base offset 0x64 supplies a separate linked base. Its offset 0x0D receives the early lookup base’s low byte, and its offset 0x14 receives that linked base itself. Only afterward is scaled-base offset 0x04 read to capture the lookup source.

The doubled-derived base receives the early lookup base’s low byte at offset 0x09 and eight times B at offset 0x44. A byte is then read from lookup-source offset 0x05. Before that byte is stored as a zero-extended word at early-lookup-base offset 0x44, scaled-base offset 0x09 receives the doubled-derived base’s low byte.

Subsequent updates put eight times B at scaled-base offset 0x14, the lookup source’s low byte at doubled-derived-base offset 0x15, and the scaled base at doubled-derived-base offset 0x44.

Lookup-source offset 0x05 is read again. Scaled-base offset 0x15 receives the low byte of eight times B before the word lookup at the newly read byte plus 0x74 captures the late lookup value. The second byte observation need not match the first.

Before the constant-write phase, scaled-base offsets 0x15, 0x54, 0x09, and 0x14 respectively receive the lookup source’s low byte, the scaled base, the doubled-lookup base’s low byte, and eight times B.

### Constants and repeated publication

The late lookup value determines three constant destinations, in this order:

| Destination | Width | Value |
|---|---:|---:|
| Target base | 8 bits | 0x3A |
| Eight times the late lookup value | 16 bits | 0x003A |
| Target-base offset 0x20 | 32 bits | 0x0000003A |

Publication-base offset 0x20 subsequently receives four word writes: 0x20, 0x2F, the zero-extended byte read from publication-base offset 0x04, and 0x20 again. The byte read occurs after the 0x2F write and before its corresponding word write.

The global base is captured next. A word value of 0x20 is then written to absolute address 0x60. Even if that store changes the global word’s storage, later uses of the global base use the already captured value.

The repeated publication region has an initial group using eight times B, followed by repeated groups using eight times the adjusted base. Each group orders four writes:

- A word containing the target base at the group-specific base’s offset 0x54.
- A byte containing the publication base’s low byte at global-base offset 0x19.
- A byte containing the global base’s low byte at publication-base offset 0x11.
- A word containing the group-specific base at global-base offset 0x34.

The byte payloads and bases are captured values, not repeated loads of the global word. The repetition does not establish traversal: the subsequent groups reuse fixed computed addresses. Exact repetition-count certification remains **documented_only** in this contract; observable repetitions must not be replaced by a single group.

### Post-publication captures

The boundary after the repeated region has an alias-sensitive order:

1. Capture the word at absolute address 0xA9.
2. Capture the word at target-base offset 0x74.
3. Write the low byte of eight times the adjusted base plus 0x30 to absolute address 0x3A.
4. Capture the word at doubled-lookup-base offset 0x14.
5. Capture the word at publication-base offset 0x04.

The fourth capture later supplies the third indirect argument. The fifth supplies a later global-base-offset-0x74 write. Neither may be moved before the byte store to 0x3A: that store can overlap either word.

### Final propagation and absolute-byte observations

Doubled-lookup-base offset 0x0D first receives the publication base’s low byte. Publication-base offset 0x08 receives 0x6C; offsets 0x0D and 0x09 receive the global base’s low byte. Doubled-lookup-base offset 0x54 then receives a self-reference.

Absolute address 0x67 receives the low byte of the word captured from 0xA9. The word at doubled-lookup-base offset 0x44 is read **before** 0x67 is read to supply the byte write at absolute address 0x21. Call that captured word the **shift source**.

Publication-base offset 0x10 receives the shift source’s low byte. Global-base offset 0x54 receives the full shift source. Doubled-lookup-base offset 0x01 receives the low byte of its signed right-shift result, using the publication base’s low byte as the shift count. Global-base offset 0x54 then receives the publication base.

The next propagation preserves these dependencies:

- Publication-base offset 0x40 is read as a word.
- The previously captured publication-base-offset-0x04 word is stored at global-base offset 0x74.
- The word captured from publication-base offset 0x40 is stored at publication-base offset 0x10, replacing four bytes beginning there.
- Global-base offset 0x44 receives the doubled-lookup base.

There are then **two additional, distinct byte reads of absolute address 0x67**, each supplying a separate store to publication-base offset 0x09.

Between those two observations, doubled-lookup-base offset 0x09 receives the publication base’s low byte, publication-base offset 0x44 receives the word captured from 0xA9, and publication-base offset 0x54 is read to obtain the **subsequent destination base**. Thus, neither later 0x67 observation may be replaced with the first observation used for 0x21, or with the other later observation.

After the second of these publication-byte stores:

- Subsequent-destination offset 0x0D receives the low byte of the word captured from target-base offset 0x74.
- Doubled-lookup-base offset 0x54 receives the word previously captured from its offset 0x14, replacing the self-reference.
- Publication-base offset 0x14 is read and captured.
- Doubled-lookup-base offset 0x11 receives that same captured target word’s low byte.
- Publication-base offset 0x09 receives the subsequent destination’s low byte, and offset 0x44 receives the full subsequent destination.

The target word’s low byte comes from its earlier word capture, not a fresh target-memory read.

### Terminal propagation and transfer

A word read from doubled-lookup-base offset 0x04 supplies the **terminal base**. The remaining interleaving is significant:

- A 16-bit value 0x0065 is stored at twice the terminal base **before** terminal-base offset 0x64 is read. That read captures the second indirect argument.
- Subsequent-destination offset 0x11 receives the low byte of the captured publication-base-offset-0x14 word. Subsequent-destination offset 0x54 then receives the 32-bit value 100.
- Only after those stores is terminal-base offset 0x74 read, capturing the fourth indirect argument.
- Doubled-lookup-base offset 0x64 is read afterward, producing a relay base.
- Doubled-lookup-base offset 0x09 receives the terminal base’s low byte. Relay-base offset 0x54 receives the captured second argument. A word containing the relay base is then stored at twice the captured second argument.

The terminal-base word reads cannot be grouped before the intervening writes. Aliasing can change either captured argument.

The indirect transfer presents these four 32-bit argument slots:

| Slot | Value |
|---|---|
| 1 | 0x00000072 |
| 2 | Captured word from terminal-base offset 0x64. |
| 3 | Word captured from doubled-lookup-base offset 0x14 after the absolute-0x3A store. |
| 4 | Captured word from terminal-base offset 0x74, after the subsequent-destination stores. |

No result from the transfer is visibly consumed.

## Side effects

The visible body modifies explicit-input-addressed storage, hidden-state-derived storage, memory-derived storage, global-derived storage, and low absolute storage.

Fixed-address effects include a word store at 0x60, a word read at 0xA9, byte stores at 0x3A, 0x67, and 0x21, and three distinct byte reads at 0x67. Byte-indexed word reads begin within the range 0x74 through 0x173.

The supplied warning about overlapping global storage means distinct symbolic views do not establish separate allocations.

No allocation, release, ownership transfer, synchronization, transaction, or rollback is visible. Address use does not establish ownership. Invalid accesses can prevent later effects; no visible recovery or status result specifies what happens afterward. Earlier writes are not shown being undone.

The indirect destination may have additional effects, including changes to global state or memory ownership.

## Dependencies

No resolved direct function dependency is shown.

The terminal destination is an **unidentified indirect dependency**. No supplied opaque function ID identifies it; F-c293dad8f29e identifies only the function specified here.

The destination’s calling convention, memory effects, global preservation, ownership behavior, and return behavior remain **documented_only**. No preservation or release obligation may be inferred.

Execution also depends on suitable memory mappings, hidden entry state, global contents, and machine arithmetic and access semantics.

## Acceptance examples

These are conditional review criteria, not executed tests.

1. **Early replacement.** With valid non-overlapping storage, first-input offset 0x54 receives A and later twice B. A contract requiring a later store of B there would not match the supplied representation.

2. **Unscaled byte lookup.** If a lookup-source byte is 0xFF, the associated word lookup begins at 0x173. It does not use 0xFF multiplied by four.

3. **Publication history.** Observable publication-base-offset-0x20 stores include 0x20, 0x2F, a zero-extended byte observation, and 0x20 in order. Keeping only the final value loses the specified trace.

4. **Capture affected by 0x3A.** If doubled-lookup-base offset 0x14 equals 0x3A and accesses complete, its captured word includes the intervening byte store’s effect according to the environment’s endianness. The third indirect argument uses that post-store capture.

5. **Global propagation.** If destinations are otherwise disjoint, global-base offset 0x74 receives the word captured from publication-base offset 0x04, and global-base offset 0x44 receives the doubled-lookup base.

6. **Distinct 0x67 observations.** If an intervening aliased store changes 0x67, a later publication-base-offset-0x09 store uses the new byte. Reusing the byte previously copied to 0x21 is invalid.

7. **Target-byte propagation.** Subsequent-destination offset 0x0D and doubled-lookup-base offset 0x11 receive the low byte of the earlier target-base-offset-0x74 word capture, even if that source storage later changes.

8. **Terminal halfword alias.** If twice the terminal base overlaps terminal-base offset 0x64, the second argument is captured after the 0x0065 halfword store.

9. **Terminal word alias.** If subsequent-destination offset 0x54 overlaps terminal-base offset 0x74, the fourth argument observes memory after the word store of 100. Capturing it before that store is invalid.

10. **Unused explicit input.** Changing only the third explicit input introduces no explicit dependency in the visible body. This does not constrain the unidentified continuation.

## Uncertainties

The following remain **documented_only**:

- Application-level intent and actual public calling convention.
- Origin and validity of hidden entry-state values.
- Accuracy of the represented function boundary and repeated region.
- Exact repeated-publication count.
- Ordinary-memory, device-memory, or invalid-address status of accessed locations.
- Endianness, alignment requirements, and fault behavior.
- Signed-overflow, left-shift, and out-of-range right-shift semantics.
- Concurrent observation and synchronization rules.
- Indirect destination identity, effects, and actual call-versus-jump behavior.

These uncertainties do not erase established access dependencies, capture order, or propagation effects.

## Evidence

This contract uses the supplied static representation for F-c293dad8f29e. Prior prose and reviewer feedback are not independent evidence.

Established static evidence includes explicit access widths, constants, address relationships, separate reads, capture-before-use dependencies, visible ordering, unused third-input status, hidden-state consumption, and unresolved indirect control flow.

Application intent and executable validity are not established. No runtime execution or tests have been performed. Acceptance examples are conditional specifications, not observed results.
