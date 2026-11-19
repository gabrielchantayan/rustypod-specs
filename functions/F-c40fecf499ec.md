<!-- Trusted provenance: F-c40fecf499ec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c40fecf499ec

## Purpose

F-c40fecf499ec performs an ordered sequence of indirect dependency invocations, constructs shifted and masked quantities from their results, reads and modifies fixed and computed memory locations, and ends with another indirect invocation.

The supplied static representation does not establish an application-level purpose. Address-like values, repeated field offsets, and printable constants do not establish allocation, serialization, formatting, container maintenance, or device initialization.

This contract distinguishes:

- **Established static evidence:** represented operations, operand widths, ordering, and value-capture boundaries.
- **[INFERENCE]:** conclusions requiring an explicitly stated arithmetic or execution model.
- **documented_only:** unresolved behavior for which no executable guarantee is supplied.

No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit use appears in the body. |
| Second argument | 32 bits | No explicit use appears in the body. |
| Third argument | Signed 32 bits | Shifted left by two positions in two early invocation argument lists. |

The first two arguments are not established as addresses, sizes, flags, or handles. Their absence from represented expressions does not establish that dependencies cannot observe incoming machine-register contents under unresolved calling conventions.

There is no visible range, sign, overflow, or alignment check on the third argument.

### Implicit inputs

Execution depends on incoming control-link state, indirect-call targets, dependency results, additional result-register values, and memory contents.

Memory inputs include:

- A byte at absolute address 0x05.
- A global 32-bit value captured as a base before the repeated write region.
- A 32-bit word beginning at absolute address 0xA9.
- Three separate late byte reads at absolute address 0x67.
- Bytes and words at computed addresses, including values subsequently used as memory bases.

These inputs are sampled at their represented read points, not collectively at entry. Dependencies can affect memory before those reads; no preservation contract is available.

### Outputs and completion

The represented interface has no explicit return value. Observable outputs are dependency interactions and memory effects.

A return follows the final invocation. Normal completion is therefore conditional on reaching that invocation and its returning normally. No success flag, error code, output size, or ownership token is established.

### Preconditions and errors

Every reached indirect target must be callable under its actual convention. Every reached memory access must be valid under the applicable memory model, including low absolute addresses and potentially unaligned word accesses.

The body does not validate these conditions. No explicit graceful handling of invalid targets, inaccessible storage, arithmetic anomalies, dependency errors, or nontermination is represented.

No explicit rollback or recovery mechanism is represented; preservation or reversal of earlier effects by dependencies is unknown.

In particular, the final dependency is not established to preserve the state produced by the direct-memory region, whether it returns, fails, or transfers control elsewhere.

## Behavior

### Ordering and execution structure

The represented body consists of a long dependency-interaction region, a direct-memory mutation region containing explicit repeated groups, and a late mutation region ending in an unresolved invocation.

No explicit conditional branch or input-controlled loop appears. Dependencies may nevertheless branch, block, fail, modify state, or never return.

Ordering is part of the contract. An invocation cannot be discarded merely because its represented result is unused. Repeated stores cannot be merged solely because their final values might agree. Computed accesses may overlap each other, fixed-address storage, or storage represented as global.

### Dependency interactions

For discussion, the early indirect sites are collectively designated F-dep-001; the final site is designated F-dep-002. These are opaque site labels, not resolved implementation identities.

The initial target depends on incoming control-link state. Subsequent interaction targets are constructed from embedded control locations with a displacement of 0xFFE.

F-dep-001 sites receive zero through four represented arguments. Argument construction includes explicit zeros, duplicate captured values, left-shifted results, masked fields, and values retained across intervening invocations.

Results are represented as 32-bit quantities, 64-bit quantities with separately consumed low and high halves, and additional result-register values with unresolved producer contracts.

The third explicit argument contributes to two distinct early interactions:

- One receives a dependency result shifted left by eight together with the third argument shifted left by two.
- A subsequent interaction receives the preceding 64-bit result’s low half shifted left by eight, its high half unchanged, and the third argument shifted left by two again.

These uses establish propagation, not application meaning.

Some high halves are forwarded unchanged; others are independently shifted or masked. Retained results may be used after several further dependency interactions.

### Widths and arithmetic

The representation distinguishes 8-bit bytes, 16-bit halfwords, 32-bit words, and 64-bit dependency results.

Byte stores retain the low eight bits. Halfword stores retain the low sixteen bits. Unsigned byte reads used as word values are zero-extended.

Masks and shifts are width-sensitive. Masking before shifting is not generally interchangeable with masking afterward. Logical right shifts of unsigned operands must not be replaced with arithmetic right shifts.

Some right shifts apply to the full 64-bit result before narrowing. A full-result shift right by 36 extracts the high 32-bit half shifted logically right by four; it is not a 36-position shift of a 32-bit operand. Full-result shifts by 34 and 44 likewise retain their 64-bit interpretation.

**[INFERENCE]** Under 32-bit wrapping arithmetic, word-valued additions, subtractions, products, and left shifts discard bits above bit 31. Consecutive left shifts can erase information rather than merely rescale a value. Signed notation alone does not establish portable signed-overflow semantics.

A late signed right shift uses the low byte of a computed word as its count. Negative-operand behavior and counts outside 0–31 remain **documented_only**. No modulo-32 count reduction is established.

### Initial direct-memory effects

Values captured from the last interactions provide:

- A low result half shifted left by 25.
- A separate result multiplied by \(2^{25}\).
- Another result shifted left by eight.
- An index formed by shifting a result left by eight, retaining fifteen low bits, then shifting logically right by six.
- The low and high halves of the final represented 64-bit interaction result.

**[INFERENCE]** With 32-bit wrapping arithmetic, the index is four times its producer’s low seven bits: 0–508 in steps of four. Twice that index is 0–1016 in steps of eight.

The initial effects retain several important order relationships:

- A word store through the last result’s high half at offset 0x14 precedes a byte store through the 25-position-shifted base at offset 9.
- A word store through the last result’s low half multiplied by 256, at offset 0x54, separates two byte stores to that same offset-9 location.
- A zero word store at offset 0x14 of the \(2^{25}\)-scaled base precedes a byte read at offset 5 of that base.
- A zero word store through the doubled-index base intervenes before the captured byte is stored, zero-extended, at offset 0x34 of the scaled base.
- A later word read at offset 0x14 supplies a retained quantity used in subsequent addresses and stores.

There is no represented null check on these computed bases. Stores following a zero store do not establish that a later read returns zero: intervening accesses can alias its storage.

### Access widths and field relationships

Offsets describe access locations, not a recovered object layout.

| Offset | Represented access widths |
|---:|---|
| 1 | Byte |
| 4 | Byte and 32-bit reads |
| 5 | Byte |
| 8 | Byte |
| 9 | Byte |
| 0x0D | Byte |
| 0x10 | Byte and 32-bit |
| 0x11 | Byte |
| 0x14 | 32-bit |
| 0x15 | Byte |
| 0x19 | Byte |
| 0x20 | 32-bit |
| 0x24 | 32-bit |
| 0x34 | 32-bit |
| 0x40 | 32-bit |
| 0x44 | 32-bit |
| 0x54 | 32-bit |
| 0x64 | 32-bit |
| 0x74 | 32-bit |

The byte read at offset 4 supplies a zero-extended value for an ordered series of offset-0x20 stores. Other offset-4 reads capture full words used in later address relationships. These widths are not interchangeable.

Words read from fields become subsequent bases. Some zero-extended byte values also become bases for word reads at offset 0x74. No bounds or pointer validation is represented.

Several stores place a base value into a field addressed through that base. **[INFERENCE]** These resemble self-references, but establish no list, tree, ownership, or object-lifetime invariant.

### Capture at absolute address 0x05

The byte read at absolute address 0x05 is distinct from the earlier computed offset-5 byte read.

It occurs after initial computed mutations, including two byte stores at offset 0x15 of the doubled-index base, and before the byte write to absolute address 0x01.

The captured byte is zero-extended. Its retained value supplies:

- A byte-store address at offset 0x15.
- A word-read address at offset 0x54.
- A subsequent word-store address at that same offset 0x54.

The offset-0x54 word is read before its overwrite. That captured word then supplies additional bases and scaled quantities.

Earlier computed writes may alias 0x05. Later writes may change it again. Neither possibility permits moving the read to function entry or replacing its capture with a later reload.

### Mutations before repetition

The following region combines retained values, scaled bases, fresh field reads, and mixed-width stores. Offsets 0x14, 0x24, 0x34, 0x44, 0x54, 0x64, and 0x74 participate.

Repeated field reads remain distinct. Computed offset-5 byte reads and offset-0x74 word relationships are revisited between mutations. A previous read is not a substitute for a subsequent one without an independently established non-aliasing guarantee.

This region includes a byte store of 0x3A through a fourfold base, a halfword store of 0x3A through the corresponding eightfold base, and a word store of 0x3A at offset 0x20 of the fourfold base.

Another captured quantity is reduced by 0x96. Four times that adjusted quantity supplies the base for four ordered word stores at offset 0x20: first 0x20, then 0x2F, then the zero-extended byte freshly read at offset 4 of that base, and finally 0x20 again.

The byte read occurs after the 0x2F store and before its derived word store. It is neither an entry-state read nor a full-word read.

### Global-base capture and repeated writes

After those offset-0x20 stores, a separate global word is captured as a base. Its capture precedes the word store of 0x20 at absolute address 0x60.

That retained base is reused throughout the repeated region without a represented reload. Aliasing writes to the original global storage do not redirect subsequent groups.

Each group has four ordered effects:

1. A word store at offset 0x54 of a scaled base.
2. A byte store at offset 0x19 of the captured global base, using the low byte of the adjusted fourfold quantity.
3. A byte store at offset 0x11 of that adjusted fourfold quantity, using the low byte of the captured global base.
4. A word store at offset 0x34 of the captured global base.

The initial group uses an unadjusted scaled base and corresponding offset-0x34 value. Subsequent groups use the adjusted scaled quantity. The word value written at offset 0x54 is retained throughout.

The groups are explicit repetition, not a represented convergence operation or “repeat until unchanged” loop. Their exact count is **documented_only** in this contract. Final-state equivalence alone does not justify reducing them to one group.

### Capture at absolute address 0xA9

Immediately after repetition, the function reads and retains the full word beginning at 0xA9. This precedes both a computed offset-0x74 word read through a retained scaled base and the byte write to absolute address 0x3A.

The captured 0xA9 word survives intervening mutations. Its low byte later supplies the assignment to absolute address 0x67; its full width later supplies a computed offset-0x44 word store.

These uses consume the capture, not a reload. Computed writes may alter the four bytes beginning at 0xA9 without changing the already retained value.

The byte written at 0x3A is derived from the adjusted quantity multiplied by eight and increased by 0x30, truncated to byte width. It is not established as a character or formatting result.

### Three fresh reads at absolute address 0x67

The assignment to 0x67 and its three subsequent reads are separate events:

- The low byte of the captured 0xA9 word is assigned to 0x67.
- A computed offset-0x44 word is read.
- A fresh byte read at 0x67 supplies the write to absolute address 0x21.
- Intervening computed mutations occur, including mixed-width stores.
- Another fresh byte read at 0x67 supplies a byte store at offset 9 of the adjusted fourfold base.
- Further mutations occur, including another computed byte store, the full captured 0xA9 word’s offset-0x44 store, and an offset-0x54 word read.
- A third fresh byte read at 0x67 supplies another offset-9 byte store through the adjusted fourfold base.

Intervening computed writes can alias 0x67. The three reads are not established to agree, and neither computed offset-9 store may automatically reuse the assignment value or a previous read.

### Final captures and invocation

A word read at offset 0x14 of another retained base is saved before intervening mutations. It later supplies an offset-0x54 store and F-dep-002’s third argument. That store uses the saved value; it does not guarantee recovery of a whole earlier memory state.

Other late captures use offsets 4, 0x40, 0x54, 0x64, and 0x74. The late region includes a halfword store of 0x65 through a doubled base and a word store of decimal 100 at a computed offset 0x54.

F-dep-002 receives four represented arguments:

| Position | Value |
|---:|---|
| First | Constant 0x72 |
| Second | A late word captured through an offset-0x64 relationship |
| Third | The retained offset-0x14 word |
| Fourth | A late offset-0x74 word read through the base obtained by the final offset-4 relationship |

No result check follows this invocation. Its arguments are not simply copies of the earlier 0xA9 capture.

## Side effects

Direct fixed-address effects include:

- A byte read at 0x05.
- Byte writes at 0x01, 0x15, 0x21, 0x3A, and 0x67.
- Three subsequent byte reads at 0x67.
- Word writes at 0x54 and 0x60.
- A word read beginning at 0xA9.

Computed accesses can overlap these locations. No independent object boundaries or disjointness guarantees are supplied.

The 0xA9 read makes unaligned-word behavior relevant. Endianness, alignment requirements, memory permissions, volatile behavior, and external observation rules remain **documented_only**.

Constants including 0x3A, 0x20, 0x2F, 0x6C, 0x65, and decimal 100 do not establish strings, encodings, terminators, or buffer layouts.

No direct allocation, release, lifetime extension, or ownership transfer is established. Address-like stores do not confer ownership. Dependencies may allocate, retain, mutate, release, or restore storage; their contracts are unknown.

No locking, atomicity, transactionality, reentrancy, or thread-safety guarantee is established. The absence of a local recovery path does not establish that dependencies leave earlier effects intact.

## Dependencies

| Opaque ID | Scope |
|---|---|
| F-dep-001 | Collection of indirect interaction sites preceding direct memory mutation |
| F-dep-002 | Final unresolved indirect invocation |

These labels do not establish actual callee identities. Different target expressions do not prove distinct implementations.

Actual targets, complete calling conventions, additional result-register provenance, error handling, and control-transfer behavior remain **documented_only**.

Neither dependency label guarantees memory preservation, global preservation, ownership behavior, normal return, or noninterference with earlier effects. A dependency may modify or reverse previously written storage; the body supplies no contract resolving that possibility.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Argument propagation and index boundaries

If the third argument is 3 and the relevant invocations are reached, each represented two-position-shifted contribution is 12. Rejection, saturation, and defined behavior for negative or overflowing inputs are not established.

Under 32-bit wrapping arithmetic, an index producer whose low seven bits equal 127 produces index 508 and doubled index 1016. Changing only higher bits preserves that index, not necessarily overall execution.

### Byte extension and ordered stores

If a computed offset-5 byte read obtains 0xFE, its derived unsigned word value is 254, not −2.

If the offset-4 byte read among the ordered offset-0x20 stores obtains 0xFE, the third word store writes 254. A full-word read is not equivalent. The fourth store still writes 0x20.

### Captured memory inputs

If preceding computed writes change 0x05 from 0x10 to 0x24 before its read, subsequent addresses derived from that capture use 0x24. Later changes to 0x05 do not replace the capture.

If a repeated-group store overwrites the original global-base storage, later groups still use the retained base.

If writes change the storage beginning at 0xA9 after its word capture, the later low-byte assignment to 0x67 and full-word offset-0x44 store still consume the retained word.

### Fresh reads and repetition

If 0x67 remains unchanged after its assignment, all three later reads may agree. If an intervening aliasing write changes it, each reached read consumes the byte present at its own read point.

An observer distinguishing individual stores must retain the repeated-group ordering. One final-state update is insufficient.

### Failure, recovery, and completion

If a reached address is inaccessible, no graceful error result or permission to skip the access is established.

If F-dep-002 is reached, its first argument is 0x72. Normal completion remains conditional on its return.

If a dependency modifies or restores previously written storage before failing, that possibility is not excluded by this contract. Conversely, dependency rollback is not promised. The established fact is the absence of an explicit local rollback or recovery mechanism, not a guarantee about the surviving memory state.

## Uncertainties

The following remain **documented_only**:

- Application purpose and domain meanings of arguments and fields.
- Actual indirect targets, complete signatures, and additional-register contracts.
- Instruction-level overflow, address wrapping, and variable-shift semantics.
- Endianness, alignment behavior, mappings, and access permissions.
- Object boundaries, aliasing restrictions, and external observation rules.
- Exact repeated-group count in this contract.
- Dependency allocation, release, retention, mutation, preservation, and restoration behavior.
- State remaining after dependency failure or nonlocal control transfer.
- Whether anomalous accesses and extensive repetition accurately represent executable behavior.
- Final-target errors, control flow, and return behavior.

These uncertainties do not justify replacing the represented interactions with a plausible higher-level algorithm.

## Evidence

The supplied static function representation is the sole evidence. Prior prose, reviewer feedback, and embedded annotations are not independent confirmation.

The body supports the explicit argument widths, shifted propagation, indirect interactions, mixed-width accesses, and named capture boundaries: 0x05 after early computed mutations and before 0x01; the global base before 0x60; 0xA9 after repetition and before the computed offset-0x74 read and 0x3A write; and three distinct late reads of 0x67.

It also shows a final unresolved invocation after the direct stores, with no represented local rollback or recovery path. Because that invocation’s implementation is unavailable, neither preservation nor reversal of prior effects is established.

Arithmetic equivalences are conditional where stated. No instruction bytes, dependency implementations, caller contracts, memory maps, or runtime traces were available. No runtime tests have been performed.
