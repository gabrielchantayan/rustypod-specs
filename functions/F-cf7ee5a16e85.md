<!-- Trusted provenance: F-cf7ee5a16e85; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cf7ee5a16e85

## Purpose

F-cf7ee5a16e85 performs an ordered sequence of unresolved indirect invocations, transforms their results and ambient machine inputs, updates memory through fixed and computed addresses, and ends with a four-argument indirect transfer.

No application-level purpose is established. Recurring offsets, self-valued stores, and character-like constants do not establish a container operation, formatter, decoder, allocator, or initialization routine.

This contract distinguishes three evidence classes:

- **Established static evidence:** represented access widths, constants, data dependencies, and ordering.
- **Inferred machine interpretation:** arithmetic consequences conditional on a 32-bit machine-word model.
- **documented_only:** unresolved ABI, dependency, memory-map, or execution semantics.

The contract describes represented behavior, not a validated executable replacement.

## Interface

### Entry and inputs

The represented interface has no explicit parameters. Execution nevertheless consumes ambient inputs:

- A continuation-like incoming value used to derive the first indirect target.
- An incoming 32-bit quantity shifted left by seven and supplied twice to an early invocation.
- An incoming unsigned 32-bit quantity whose upper ten bits are extracted and shifted left by three.
- Primary and secondary result words from indirect invocations, including results represented as 64-bit pairs.
- Memory reached through result-derived addresses, later memory-derived addresses, and fixed storage.

A parameterless portable interface cannot be reconstructed from this evidence. The entry calling convention, required register state, valid execution contexts, and permitted indirect targets are **documented_only**.

### Outputs and completion

No explicit return value is established. Observable outputs are dependency interactions, memory accesses and updates, and the terminal transfer to F-unknown-03.

Its four represented arguments are:

| Position | Value |
|---|---|
| First | 0x72 |
| Second | A late-loaded 32-bit word from offset 0x64 of a freshly obtained base |
| Third | A saved 32-bit word read from a computed base at offset 0x14, after the final-phase write to fixed address 0x3a |
| Fourth | A later 32-bit word read at offset 0x74 of the same fresh base used for the second argument |

The second argument remains the saved loaded value despite intervening memory writes. The third argument is likewise not reloaded before dispatch.

An ordinary completion is represented after the transfer. Whether that transfer returns, acts as a terminal dispatch, or follows another control-flow convention is **documented_only**.

### Widths, layouts, and preconditions

| Access or value | Width | Treatment |
|---|---:|---|
| Byte | 8 bits | Stores retain the low eight bits. Explicit unsigned byte reads are zero-extended. |
| Halfword | 16 bits | Used for stores of 0x3a and 0x65. |
| Word | 32 bits | Used for most scalar values, computed bases, reads, and stores. |
| Paired result | 64 bits | Low and high portions are extracted; some extraction begins above bit 32. |

Offsets are byte offsets. No complete record layout, allocation size, array bound, or non-aliasing guarantee is established.

Execution requires every reached target and memory access to be valid under the actual execution environment. Byte order, alignment requirements, address wrapping, low-address mappings, and fault handling are **documented_only**. These requirements are not enforced by represented validation branches.

## Behavior

### Dependency-driven preparation

A long straight-line prefix invokes F-unknown-01 and the unresolved sites grouped under F-unknown-02 before the first explicit data-memory store.

The first target is derived from the incoming continuation-like value using displacement 0xffe. Later targets use embedded code-location values with the same recurring displacement. This does not establish that those sites resolve to one function or to a shared public dispatch table.

The prefix has no represented conditional branch or loop. Reaching its end still requires every preceding invocation to return.

Argument relationships include:

- Duplicate arguments derived from one value by shifts of seven or fifteen bits.
- Frequent left shifts of primary results by eight.
- Secondary result words passed unchanged, shifted, masked, or alongside primary results.
- Explicit zero arguments, including zeros retained through later transformations.
- Masked fields of several widths, including 11 through 24 bits.
- Unsigned right shifts followed by left shifts.
- Extraction from 64-bit paired results at bit positions 34, 36, and 44.

A site displaying no explicit arguments must not be treated as pure or independent of ambient machine state. Actual dependency signatures remain unresolved.

Some represented values survive across later invocations. That dataflow does not establish preservation of globals, pointed-to memory, object lifetimes, or every machine register.

### Arithmetic interpretation

Word-sized transformations must preserve intermediate widths. In particular, a left shift followed by a right shift cannot generally be cancelled.

Under a truncating 32-bit machine-word interpretation:

- Shifting left by eight discards the original upper eight bits.
- Retaining twenty bits and then shifting right by ten leaves at most ten meaningful bits.
- Retaining twenty-four bits and then shifting right by twenty leaves at most four meaningful bits.
- Successive left shifts of twenty-five and seven produce zero.
- Successive left shifts of twenty-eight and twenty-eight also produce zero.

These are **inferred machine interpretations**, not assertions of portable semantics for signed-looking arithmetic. Multiplication, addition, and address scaling can also overflow the represented word width.

A late right shift uses the low eight bits of a computed base as its count. The expression permits counts from 0 through 255. Handling of counts at least 32, and sign extension of the shifted value, are **documented_only**. The count must not silently be reduced modulo 32.

### Initial explicit memory updates

Direct memory mutation starts after the last preparation invocation returns a paired result.

The opening updates establish several ordering constraints:

- A word store at offset 0x14 through the high result precedes a byte store at offset 9 through a retained computed base.
- A word store at offset 0x54 through the low result scaled by 256 occurs before that offset-9 byte is overwritten.
- A computed word at offset 0x14 is cleared before a byte is read at offset 5 of that base.
- Another offset-0x14 destination is cleared between that byte read and a word store of its unsigned extension at offset 0x34.
- A subsequent offset-0x14 word read precedes another clear and two identical byte stores at offset 0x15.

Readback concerns memory at the time of the read. Earlier apparent clears do not justify substituting zero without assumptions about aliasing and memory semantics.

### Computed-address mutation and storage views

The next region mixes word values, byte narrowing, and addresses scaled by two, four, or eight. Values that appear scalar are also used directly as memory bases.

Observed access views include:

| Offsets | Access forms |
|---|---|
| 1, 8, 9, 0x0d, 0x11, 0x15, 0x19 | Byte stores |
| 4, 5 | Byte reads at some bases; offset 4 also has word reads |
| 0x10 | Byte and word stores |
| 0x14, 0x44, 0x54, 0x74 | Word reads and writes |
| 0x20, 0x24, 0x34 | Word stores |
| 0x40, 0x64 | Word reads |

These views may overlap. For example, an offset-0x15 byte can modify the representation of an offset-0x14 word. They are not independent fields.

The region also includes:

- An unsigned byte read from fixed address 0x05, with its numeric value subsequently used as a base.
- Byte writes to fixed addresses 0x01 and 0x15.
- A word write to fixed address 0x54.
- Repeated stores of a computed base into its own offset 0x24.
- Two represented reads at one offset-0x74 location within an update that uses the loaded value both to select a destination and as stored data.
- Fresh pointer-like word reads after earlier writes.

The relative evaluation order of the two reads within that single update is not independently established. Both represented reads must remain distinguished from a guaranteed single stable lookup.

No list, tree, ownership graph, or reference-count meaning is established.

### Constant stores and repeated region

A later region stores 0x3a as a byte, as a halfword, and as a word at an offset destination.

Another word destination receives 0x20, then 0x2f, then an unsigned byte loaded from offset 4 of its computed base, then 0x20 again. The read occurs after the 0x2f store and before the final 0x20 store. Aliasing can therefore affect the loaded byte.

A fixed-storage word supplying a later base is read before the word store of 0x20 to fixed address 0x60.

An explicitly repeated region then combines:

- A word store at offset 0x54 through a scaled base.
- A byte store at offset 0x19 through the fixed-storage-derived base.
- A byte store at offset 0x11 through another computed base.
- A word store at offset 0x34 through the fixed-storage-derived base.

The opening group uses a different scaled base from subsequent groups. The repetitions are explicit straight-line stores, not a demonstrated data-dependent loop. Their exact multiplicity is not enumerated in this contract; that is a documentation limitation, not evidence of a variable count.

Repeated stores remain part of the represented access sequence even when ordinary memory would end with the same value.

### Final-phase roles and read boundary

For the final phase, the following descriptive roles identify retained values without assigning object types:

- **Working base:** the arithmetic-derived base used for accesses at offsets 4, 8, 9, 0x10, 0x14, 0x40, 0x44, and 0x54.
- **Secondary base:** the retained doubled value used for accesses at offsets 1, 4, 9, 0x0d, 0x11, 0x14, 0x44, 0x54, and 0x64.
- **Fixed-storage-derived base:** the base loaded before the repeated region.
- **Repeated-value base:** the retained value stored repeatedly through scaled destinations.

These roles do not imply distinct addresses or non-overlapping storage.

The start of this phase has a strict read/write boundary:

1. The word at fixed address 0xa9 is read and saved.
2. A word at offset 0x74 of the repeated-value base is read and saved.
3. A byte is written to fixed address 0x3a.
4. The word supplying the third dispatch argument is read at offset 0x14 of the secondary base.
5. Another word is saved from offset 4 of the working base.

The fixed byte is the low eight bits of eight times the retained adjusted scale value plus 0x30, subject to the unresolved arithmetic execution model.

The first two saved words precede the 0x3a write. The next two reads follow it and can observe it if their accessed storage overlaps address 0x3a. Neither later read may be moved before that write.

### Fixed byte storage at 0x67 and 0x21

The final phase first performs byte updates through the working and secondary bases and stores the secondary base as a word at its own offset 0x54. Only then does fixed address 0x67 receive the low byte of the saved word from address 0xa9.

There are three subsequent, distinct byte reads from address 0x67:

| Sample | Ordering and consumer |
|---|---|
| First | After the write to 0x67, a word is read from secondary-base offset 0x44. The byte at 0x67 is then read to supply the byte written to fixed address 0x21. |
| Second | After intervening computed-address writes and a word read at working-base offset 0x40, address 0x67 is read again to supply a byte store at working-base offset 9. |
| Third | After a byte store at secondary-base offset 9, a word store at working-base offset 0x44, and a word read at working-base offset 0x54, address 0x67 is read again for another byte store at working-base offset 9. |

Between the first and second samples, the represented writes include a byte at working-base offset 0x10, two successive words at fixed-storage-derived-base offset 0x54, a byte at secondary-base offset 1, a word at fixed-storage-derived-base offset 0x74, a word at working-base offset 0x10, and a word at fixed-storage-derived-base offset 0x44.

Consequently:

- The 0x21 write is sourced from the first fresh read of 0x67, not directly from the saved 0xa9 word.
- The two computed-address byte stores use their respective fresh reads.
- Intervening computed writes can overlap 0x67 and change later samples.
- The first computed byte store can itself overlap 0x67.
- A device-backed read can have independent effects or yield changing values.

Equality of all three samples requires additional memory assumptions. It is not unconditional forwarding of the originally saved low byte.

### Remaining mutation and dispatch

The word read at working-base offset 0x40 supplies the later word store at working-base offset 0x10. A saved earlier working-base offset-4 word is written through the fixed-storage-derived base.

After the three 0x67 samples, the function uses the low byte of the saved repeated-value-base offset-0x74 word for computed byte stores. It restores the saved third dispatch argument into secondary-base offset 0x54, then reads a word at working-base offset 0x14.

Further writes overwrite working-base offset 9 with the low byte of the word obtained from working-base offset 0x54. Thus, neither earlier byte store there is necessarily its final value.

A fresh base is loaded from secondary-base offset 4. A halfword 0x65 is stored through twice that base, before the read at its offset 0x64 that supplies the second dispatch argument. Later writes include word 100 at offset 0x54 of another computed base.

The fourth dispatch argument is read from fresh-base offset 0x74 after those writes. Another base is then loaded from secondary-base offset 0x64. Final updates include a word at that base’s offset 0x54 and a word through twice the saved second dispatch argument.

F-unknown-03 follows these writes. No rollback precedes it.

## Side effects

### Memory and ordering

The function accesses fixed low addresses and arbitrary computed addresses. No null, range, alignment, overflow, or address-validity checks are represented.

Accesses describe an ordered mutation, not a simultaneous final-state assignment. Earlier writes can alter later loaded values, including later bases and dispatch arguments. Saved values remain distinct from subsequent reads of their original locations.

No atomicity, synchronization, transaction boundary, or concurrency protection is established. Whether repeated or device-backed accesses have effects beyond ordinary storage is **documented_only**.

### Ownership and lifetime

No explicit allocation, release, or ownership-transfer operation is identifiable in the direct memory phase.

Dependencies may allocate, release, retain references, or mutate shared state. Their contracts are unavailable. Pointer-like arguments and self-valued stores do not establish ownership or continued object lifetime.

### Errors and partial progress

No explicit error return, exception handler, retry, or recovery path is represented.

Consequences of an invalid target or inaccessible address are **documented_only**. Earlier invocations and stores may already have taken effect. Neither clean failure nor unchanged memory is promised.

## Dependencies

Only the subject has a resolved opaque identity. The other IDs are document-local boundary labels:

| Opaque ID | Role | Established boundary |
|---|---|---|
| F-unknown-01 | Initial indirect invocation | Target derived from ambient continuation-like state |
| F-unknown-02 | Remaining preparation sites | Distinct unresolved sites with zero to four explicit word arguments and word or paired-result consumption |
| F-unknown-03 | Terminal indirect transfer | Four represented arguments after the direct memory phase |

F-unknown-02 is a grouping, not one known callee. Target equality is not established.

For every unresolved dependency, calling convention, memory effects, global preservation, ownership, failure behavior, and return guarantees are **documented_only**.

## Acceptance examples

These are conditional static examples, not executed tests.

### Unsigned promotion

If an unsigned byte read obtains 0xfe, its word-sized numeric extension is 254, not −2. Byte arrangement remains dependent on target byte order.

### Incoming field extraction

For incoming unsigned word 0xffc00000, upper-ten-bit extraction followed by a three-bit left shift yields 0x1ff8. For 0x003fffff, it yields zero. Dependency results remain unconstrained.

### The 0x3a read boundary

If secondary-base offset 0x14 begins at address 0x3a, the third dispatch argument is read after the fixed byte write and must reflect that byte in ordinary memory. Its numeric contribution depends on byte order.

The saved word from repeated-value-base offset 0x74 was read before that write. Even if its original storage overlaps 0x3a, its retained value is not retroactively changed.

### Stable 0x67 storage

If ordinary storage at 0x67 receives 0xe7 and no intervening write overlaps it, all three subsequent reads obtain 0xe7. The represented sequence still contains three reads, the intervening computed word read before the 0x21 write, and the intervening memory updates.

### Aliasing between 0x67 samples

Suppose the secondary base is 0x5e, so its offset-9 byte store addresses 0x67. Suppose earlier relevant writes do not overlap 0x67.

If the saved low byte is 0xe7, the 0x21 write and first working-base offset-9 store can receive 0xe7. The intervening secondary-base offset-9 store then changes 0x67 to the low byte of the working base. The final sample must obtain that new byte under stable ordinary-memory assumptions.

Later overwriting of working-base offset 9 does not erase the required earlier stores from the access sequence.

### Overwritten word destination

For non-aliased ordinary memory, the destination receiving 0x20, 0x2f, a promoted byte, and 0x20 ends at 0x20. Final-state equality alone does not prove the required read and store order.

### Invalid access or terminal transfer

An unmapped computed destination has no established clean-error result. If all preceding operations complete, F-unknown-03 receives 0x72 first and the specified saved or late-loaded words afterward. Its return behavior remains unresolved.

## Uncertainties

- **documented_only — Purpose:** no application-domain meaning is established.
- **documented_only — Entry ABI:** ambient inputs and machine-state lifetimes are unresolved.
- **documented_only — Dependencies:** identities, preservation rules, ownership effects, and return guarantees are unavailable.
- **documented_only — Arithmetic:** signed overflow, oversized shifts, address wrapping, and exact instruction semantics are unresolved.
- **documented_only — Memory:** byte order, alignment, mapping, device behavior, and validity of fixed low addresses are unknown.
- **documented_only — Layout and aliasing:** recurring offsets do not establish types, extents, or disjoint storage.
- **documented_only — Completion:** terminal target and subsequent control flow are unresolved.
- **Documentation limitation:** exact repeated-store multiplicity is not enumerated.
- **Inferred intent:** masks suggest field manipulation and offsets suggest structured access; neither establishes a format or data structure.

## Evidence

The supplied static representation establishes the access widths, constants, data dependencies, and ordering described here. In particular, it distinguishes the three reads of address 0x67 and places the 0x3a byte write between the two early final-phase reads and the later saved-value reads.

Prior prose and reviewer feedback are not independent execution evidence. No dependency inspection, implementation access, runtime execution, or runtime tests have been performed.

The acceptance examples express consequences under explicit assumptions. A uniquely executable replacement would additionally require the entry ABI, dependency contracts, target arithmetic semantics, and memory-map information.
