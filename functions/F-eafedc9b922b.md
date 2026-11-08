<!-- Trusted provenance: F-eafedc9b922b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eafedc9b922b

## Purpose

F-eafedc9b922b finalizes a buffered, word-oriented transformation. It inserts externally supplied padding bytes into a block, processes one or two blocks through F-5af6fb9bdd85, and serializes five resulting state words into 20 output bytes.

Established behavior includes OR-based padding insertion, conditional block clearing, placement of two state words into the final block, big-endian output serialization, and a final reset of the buffered-position field.

A digest-finalization interpretation is inferred intent. The supplied evidence does not establish a particular algorithm, standardized padding values, or the transformation performed by F-5af6fb9bdd85.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

The function accepts two addresses:

| Input | Storage required for the ordinary bounded case | Role |
|---|---|---|
| Output address | At least 20 writable bytes | Receives five serialized 32-bit words. |
| State address | At least 24 accessible 32-bit words, with the described destinations writable | Holds transformation state, an embedded block, and a buffered-position field. |

Every visible path that returns normally returns the 32-bit value 1. This is neither a byte count nor a success result derived from inspecting a dependency return value.

### State layout

A word is 32 bits, occupying four bytes. Offsets are relative to the state address.

| State word indices | Byte offsets | Established use |
|---|---|---|
| 0–4 | 0–19 | Result words read after final processing. |
| 5 | 20–23 | Source copied into final block word 15. |
| 6 | 24–27 | Source copied into final block word 14. |
| 7–22 | 28–91 | Embedded sixteen-word block. |
| 23 | 92–95 | Buffered-position field, read near entry and cleared after serialization. |

Block word 0 is state word 7. Block words 14 and 15 are state words 21 and 22.

State accesses operate on numeric 32-bit words. Their underlying storage byte order is not established here. Output byte order is explicitly most-significant byte first.

### External padding reference

The function evaluates a global reference to padding bytes. The number, timing, and use of these evaluations depend on the buffered position’s low two bits.

A reference evaluation and a byte read are different events. An evaluated address can be unused, and several byte reads can use one previously captured address without evaluating the global reference again.

The ordering specified below is **decompiled-source evaluation order**, not a guarantee of distinct hardware loads or a concurrent-memory synchronization contract.

Only when the reference remains stable across its evaluations can all padding-byte reads be described as consuming a single contiguous prefix of one sequence. Under that condition, the function reads between one and four bytes, starting at the referenced address.

The actual padding bytes, ownership, initialization, and mutability are unknown.

### Preconditions and supported interpretation

For the ordinary buffered-block interpretation:

- The position field is stable and lies in the inclusive range 0–63.
- State and output storage support the described accesses.
- The global padding reference can be evaluated, and every address actually dereferenced supplies the required readable bytes.
- Existing data in a partially occupied word is suitable for OR-based insertion.
- F-5af6fb9bdd85 accepts the state address, embedded block address, and numeric third argument 1.

The function checks none of these conditions.

Non-overlapping state and output storage avoids serialization-induced changes to later result-word reads. Overlap is not rejected and has observable consequences.

## Behavior

### Position interpretation

The buffered-position field is evaluated twice near entry:

1. Its low two bits determine a byte position from 0 through 3.
2. A separate read is interpreted as signed 32-bit and shifted arithmetically right by two to determine the selected block-word index.

These are separate source-level reads, not an established atomic snapshot. If their values differ, the byte position and word index need not describe one coherent buffered position.

With stable input in 0–63, the selected word index is 0–15. Values with the high bit set produce a negative signed index rather than a large unsigned index.

### Padding construction and reference evaluation order

The selected byte position determines which existing word bits contribute and which padding bytes are read.

| Byte position | Existing word contributes? | Padding-byte count | Destination bit ranges, in byte-read order |
|---|---|---|---|
| 0 | No | 4 | 31–24, 23–16, 15–8, 7–0 |
| 1 | Yes | 3 | 23–16, 15–8, 7–0 |
| 2 | Yes | 2 | 15–8, 7–0 |
| 3 | Yes | 1 | 7–0 |

Each padding byte is widened without sign extension before shifting or combination. The resulting word is 32 bits.

The reference evaluations and byte reads differ by path:

**Byte position 0**

- The global reference is evaluated, and the byte at that address is read for bits 31–24.
- After that byte read, the global reference is evaluated again.
- The remaining bytes are read at offsets 1, 2, and 3 from the second evaluated address, supplying bits 23–16, 15–8, and 7–0 respectively.
- The existing selected block word is not read.

Consequently, a changing reference can produce a word whose first byte comes from one location and whose remaining bytes come from another.

**Byte position 1**

- The existing selected block word is read first.
- The global reference is then evaluated once and captured.
- Bytes at offsets 0, 1, and 2 from that captured address are read in order and ORed into bits 23–16, 15–8, and 7–0 respectively.

**Byte position 2**

- The existing selected block word is read first.
- The global reference is evaluated and captured.
- The global reference is evaluated a second time.
- The padding bytes come from offsets 0 and 1 of the **second** captured address, supplying bits 15–8 and 7–0.
- The first captured address supplies no byte read.

**Byte position 3**

- The existing selected block word is read first.
- The global reference is evaluated and captured.
- The global reference is evaluated a second time.
- The only padding byte is read from offset 0 of the **first** captured address and ORed into bits 7–0.
- The second captured address supplies no byte read.

For partial-word insertion, all existing set bits survive. The destination lanes are not cleared before OR operations. A zero padding byte therefore cannot clear stale data, and a nonzero byte can set bits but cannot replace a lane wholesale.

For byte position 0, the word is formed entirely from the four padding-byte reads; its previous contents do not contribute.

The completed word is written to the selected block location after the required padding-byte reads.

### Block selection and clearing

The selected word index is advanced by one. For stable ordinary positions, behavior is:

| Buffered position | Padding-containing block word | Before any intermediate call | After intermediate call |
|---|---|---|---|
| 0–55 | 0–13 | Clear subsequent words through word 13, in increasing order | No intermediate call |
| 56–59 | 14 | Clear block word 15 | Clear block words 0–13, in increasing order |
| 60–63 | 15 | No additional block-word clear | Clear block words 0–13, in increasing order |

For positions 0–55, clearing excludes the padding-containing word. If that word is block word 13, no subsequent word needs clearing.

For positions 56–63, F-5af6fb9bdd85 is called after padding insertion and any specified pre-call clear. It receives:

1. The state address.
2. The address of block word 0.
3. The numeric value 1.

After this intermediate call returns, block words 0–13 are cleared. These stores occur after any effects of that call.

The intermediate-call threshold is position 56, not position 60.

### Final-block population

After the applicable clearing:

1. State word 6 is read and copied into block word 14.
2. State word 5 is read and copied into block word 15.

The first destination write precedes the second source read.

When an intermediate dependency call occurs, these copies use the state as it exists after that call. Neither source value is saved at entry for this purpose.

Interpreting these words as components of a count or length is inferred intent. Their units and significance order are not established.

### Final processing

F-5af6fb9bdd85 is then called with the state address, embedded block address, and numeric value 1.

Every normally progressing invocation reaches this final call. Ordinary positions 0–55 produce one dependency call; positions 56–63 produce two.

The block address points into the same state object passed as the first argument. This function does not create a separate block copy.

### Result serialization

After the final dependency call returns, state words 0–4 are serialized in increasing word order.

For each word, the function reads the word once, then writes four consecutive output bytes in descending significance: bits 31–24, 23–16, 15–8, and 7–0. All four writes complete before the next state word is read.

Thus numeric word 0x12345678 produces bytes 12, 34, 56, 78.

Exactly 20 output bytes are written by this serialization. No terminator is appended.

The five words are not collectively snapshotted. If earlier output writes overlap a later state word, that later word is read after the overlapping writes. Within one four-byte group, however, all bytes derive from the word value already read.

### Completion

After the 20 output writes, state word 23 is set to zero. The function then returns 1.

There is no explicit final erasure of the block or reset of state words 0–6. Dependency-produced changes remain unless overwritten by the stated operations.

## Side effects

The source-level ordered effects are:

1. Read the position field for its low two bits.
2. Read it again for signed word-index derivation.
3. For partial-word insertion, read the selected block word.
4. Evaluate the padding reference and read padding bytes according to the path-specific ordering above.
5. Write the completed padding-containing word.
6. Perform applicable clearing and, when required, the intermediate dependency call followed by clearing.
7. Copy state word 6 into block word 14.
8. Copy state word 5 into block word 15.
9. Invoke the final dependency call.
10. Read each result word and write its four output bytes before proceeding to the next word.
11. Clear the position field.

In particular, byte position 0 has a padding-byte read between its two reference evaluations. Byte positions 2 and 3 perform both reference evaluations before their first padding-byte read, but use different captured addresses.

These source-level distinctions do not establish physical load counts, atomicity, or supported concurrent modification.

### Ownership and lifetime

This function contains no visible allocation, deallocation, ownership transfer, or retention of supplied addresses.

That observation does not establish corresponding guarantees for F-5af6fb9bdd85. Its allocation, release, retention, and global-memory behavior require a separate contract.

### Errors and interruption

No explicit validation covers null addresses, storage size, alignment, invalid positions, unreadable padding data, or dependency failure. No alternative error return is visible, and no dependency return value is inspected.

A fault or non-returning dependency prevents subsequent effects. There is no rollback. In particular, the position field is cleared only after serialization finishes.

## Dependencies

### F-5af6fb9bdd85

Established calling behavior:

- Receives the state address.
- Receives the address of the embedded sixteen-word block.
- Receives the numeric third argument 1.
- Is called once or twice for ordinary bounded positions.
- Has no return value inspected by this function.

Its transformation, state modifications, block modifications, other memory effects, ownership behavior, and termination conditions are unknown.

Preservation of globals, including the padding reference, must not be assumed. All padding construction precedes the first dependency call in this function, but later state reads can observe dependency changes.

The serialized result depends on state words 0–4 after the final call and cannot be calculated from the supplied function alone.

### Padding-data reference

This is a global data dependency, not a function dependency.

With reference stability, the required readable prefix contains four, three, two, or one bytes for byte positions 0, 1, 2, or 3 respectively. Without stability, accessibility requirements follow the individual captured addresses and offsets specified above.

An evaluated but unused address does not itself supply a padding-byte read. No standardized byte values are established.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless stated otherwise, the position and padding reference are stable, addresses are valid, state and output do not overlap, and dependencies return normally.

### Word-aligned insertion

Given position 0 and a stable padding reference whose first four bytes are 80, 00, 00, 00:

- Block word 0 becomes 0x80000000 regardless of its previous value.
- Block words 1–13 are cleared.
- Block words 14 and 15 receive state words 6 and 5.
- Exactly one dependency call occurs.

The padding reference is nevertheless evaluated twice. Reference stability is what makes the four reads a single-prefix read.

### Partial-word insertion

Given position 1, selected word 0x12000001, and padding bytes 80, 00, 00:

- The existing word is read before the padding reference is evaluated.
- The selected word becomes 0x12800001.
- Its low byte remains 01; the zero padding byte does not replace it.

### Different reference values

The following describe conditional source-level evaluation traces, not a guarantee that concurrent changes can produce them on a particular runtime:

- At byte position 0, suppose the first evaluated address supplies byte 12, while offsets 1–3 of the second evaluated address supply 34, 56, 78. The constructed word is 0x12345678. Offset 0 of the second address is not read.
- At byte position 2, suppose the existing word is 0xAABB0000 and the second captured address supplies bytes 12, 34. The result is 0xAABB1234. The first captured address supplies no padding byte.
- At byte position 3, suppose the existing word is 0xAABBCC00 and the first captured address supplies byte 56. The result is 0xAABBCC56, regardless of the unused second captured address’s byte contents.

### Intermediate-call boundary

Given position 55 and first padding byte 80:

- Block word 13 is ORed with 0x00000080.
- No following word through block word 13 is cleared.
- Exactly one dependency call occurs.

Given position 56 and stable padding prefix 80, 00, 00, 00:

- Block word 14 becomes 0x80000000.
- Block word 15 is cleared before the intermediate call.
- Block words 0–13 are cleared after that call.
- Final block words 14 and 15 receive the then-current state words 6 and 5.
- A second dependency call occurs.

If the intermediate call changes state word 6, the changed value is copied.

Given position 63, block word 15 receives the OR-based insertion. The intermediate call occurs without an additional pre-call block clear.

### Output and overlap

If the final call leaves state words 0–4 as 0x01234567, 0x89ABCDEF, 0x00000000, 0xFFFFFFFF, and 0x10203040, the output is:

01 23 45 67 89 AB CD EF 00 00 00 00 FF FF FF FF 10 20 30 40

The position field is then zeroed and the function returns 1.

With overlapping storage, an earlier output group can change a later result word before its read. Acceptance must not require five-word snapshot semantics. If output overlaps the position field, the final zero store can overwrite serialized bytes.

## Uncertainties

The following remain **documented_only**:

- Algorithm identity and the meaning of state words 5 and 6.
- Padding contents, ownership, initialization, and mutability.
- Whether separate source-level reference evaluations correspond to distinct hardware loads.
- Supported synchronization or concurrent mutation behavior.
- F-5af6fb9bdd85’s computation, memory effects, ownership behavior, and failure modes.
- Native state-storage byte order, address width, alignment requirements, and surrounding allocation layout.
- Whether overlapping state, output, or padding storage is intended.

Out-of-range positions are not rejected. A signed 32-bit value shifted right by two yields an index from −536870912 through 536870911; adding one remains within signed 32-bit range.

The resulting indexed accesses can fall outside the embedded block. Subsequent clearing can also reach storage outside that block. Whether those accesses remain accessible, corrupt other state, or fault depends on unavailable layout and address-arithmetic details. The ordinary 24-word storage requirement is not sufficient to justify arbitrary out-of-range inputs.

## Evidence

Established static evidence consists of the supplied function’s field accesses, signed index derivation, unsigned byte widening, path-specific padding-reference evaluations, byte-read ordering, OR operations, block stores, clearing bounds, dependency calls, result serialization, final position reset, and constant return.

Digest-like finalization and count-like field meanings are inferred intent only. They do not supply missing padding bytes or dependency semantics.

No runtime execution, dependency inspection, external algorithm comparison, or acceptance-test execution has been performed.
