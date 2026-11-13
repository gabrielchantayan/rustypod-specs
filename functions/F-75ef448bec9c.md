<!-- Trusted provenance: F-75ef448bec9c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-75ef448bec9c

## Purpose

F-75ef448bec9c specifies a sequence of indirect calls, calculated memory accesses, and 32-bit field transformations. The supplied representation terminates at a reported invalid-instruction boundary; it does not establish a normal return.

No application-level purpose is established. Repeated address calculations with a 20-byte displacement suggest structured state manipulation, but they do not establish an object layout, array, serialization format, or ownership protocol.

This contract describes the visible behavior conditionally: each operation is required only if execution reaches it. An invalid access, non-returning dependency, or other dependency effect may prevent subsequent operations.

## Interface

### Explicit inputs

The apparent interface has two arguments:

- A 32-bit first argument that the visible body does not explicitly consume.
- An unsigned 32-bit **input word**.

The first argument’s apparent non-use does not prove machine-level irrelevance. Indirect dependencies may observe ambient registers or other execution state not expressed by their displayed arguments.

### Additional entry-state inputs

The visible body also consumes:

- An unsigned 32-bit **scale source**.
- An address-like **initial read base**, used for a byte read at displacement 1.
- A **link-state value**, used to construct the first indirect destination by adding 4094.

Their initialization, validity, and calling-convention roles are unresolved.

### Outputs and execution conditions

No ordinary return value or normal completion path is established. Observable outputs are indirect calls, explicit memory reads and writes, dependency effects, and possible arrival at the reported invalid-instruction boundary.

Complete traversal requires all reached destinations and accesses to be valid, all necessary dependencies to return, and all required entry state to exist. The body contains no explicit validation of these conditions.

The supplied representation treats ordinary captured call results as 32-bit values, except for one 64-bit result. These widths describe the visible data flow; they do not independently establish a machine return convention.

## Behavior

### Arithmetic and address conventions

A **word** is 32 bits. A **scaled index** is an index multiplied by 4. Two recurring address relationships are:

- **Word slot:** index multiplied by 4, then increased by 20.
- **Wide slot:** index multiplied by 16, then increased by 20.

These names describe arithmetic only, not allocated storage or a data schema.

Explicitly unsigned 32-bit multiplication, addition, and left shifts retain the low 32 bits. Right shifts of unsigned words are logical. A byte write retains the low 8 bits.

Several dependency results and address calculations are represented as signed 32-bit integers. Signed multiplication, left shifts, overflow, and conversions must not silently be assigned unsigned wraparound semantics. Where these distinctions matter, machine behavior remains unresolved. Examples using signed calculations assume representable arithmetic or an independently established machine interpretation.

Each word write occupies four bytes. Endianness, alignment requirements, atomicity, and address-space interpretation are not established.

### Call occurrence identifiers

F-d001 through F-d098 identify indirect **call occurrences**, in execution order. They do not assert 98 distinct destination functions.

A call described as having “no explicit arguments” has none displayed in the supplied representation. This does not prove that it consumes no ambient machine state. Results are ignored unless a use is described below.

Ranges of call IDs denote individual calls in increasing order.

### Initial selector and scale publication

The scale source is logically shifted right by 12 before the first call. The resulting field later supplies the **initial scale**, obtained by shifting that field left by 10. The initial scale ranges from zero through `0x3FFFFC00`.

The initial interactions have this order:

1. F-d001 receives the input word.
2. F-d002–F-d005 receive no explicit arguments; the result of F-d005 is retained.
3. A byte is read at initial read base plus 1 and zero-extended as the **initial selector**.
4. One byte is written at initial selector plus 17. Its value is input bits 22–29. Input bits 30–31 do not survive byte truncation.
5. F-d006 receives the retained F-d005 result shifted left by 2.
6. F-d007–F-d010 receive no explicit arguments; the result of F-d010 is retained.
7. F-d011 receives the initial selector and the retained F-d010 result shifted left by 2.
8. F-d012 receives no explicit arguments.
9. F-d013 receives two copies of the initial scale.
10. F-d014 receives no explicit arguments and produces a represented 64-bit result.

The 64-bit result is separated into a **paired low word** and a **paired high word**. The high half is represented as a signed 32-bit value for subsequent index arithmetic; the low half is unsigned.

The initial scale is written to the word slot selected by the paired high word. F-d015 then receives the paired high word.

The first byte-write destination is derived from an unsigned byte, not a full-width pointer read. Its numerical address is therefore between 17 and 272 inclusive, before platform-specific address interpretation.

### Word publication and link construction

The following stages occur in order. Each row’s explicit memory operations occur after its leading calls and before its trailing call. Captured indices are independent dependency results unless the row states otherwise.

| Stage | Required behavior |
|---|---|
| Paired-high publication | F-d016–F-d019 have no explicit arguments. The F-d019 result selects a word slot. The paired high word is written there, preserving its 32-bit pattern. F-d020 receives the selecting index multiplied by 4. |
| Paired-low transformation | F-d021–F-d024 have no explicit arguments. The paired low word multiplied by `0x04000000` is written at paired low word multiplied by `0x10000000`, plus 20. These are unsigned 32-bit calculations. The low word masked with `0x1FFFFFFF` is retained. F-d025 receives the value just written. |
| Masked-field publication | F-d026–F-d029 have no explicit arguments. The F-d029 result selects a word slot. The retained masked word shifted right by 9 is written there. F-d030 receives the selecting index multiplied by 4. |
| First wide-slot construction | F-d031–F-d034 have no explicit arguments. The F-d034 result selects a wide slot, which receives that index multiplied by 4. F-d035 receives, in argument-position order, a word-slot read shifted right by 12, the scaled index, and a word-slot read shifted left by 2. Both read expressions address the word slot selected by F-d034. |
| First link publication | F-d036–F-d039 have no explicit arguments. The F-d039 result selects a word slot, which receives the scaled index from the first wide-slot construction. F-d040 receives the new selecting index multiplied by 4. |
| Second wide-slot construction | F-d041–F-d044 have no explicit arguments. The F-d044 result selects a wide slot, which receives that index multiplied by 4. F-d045 receives two copies of this scaled index, followed by the paired low word shifted left by 26. |
| Second link publication | F-d046–F-d049 have no explicit arguments. The F-d049 result selects a word slot, which receives the scaled index from the second wide-slot construction. F-d050 receives the new selecting index multiplied by 4. |
| Third wide-slot construction | F-d051–F-d054 have no explicit arguments. The F-d054 result selects a wide slot, which receives that index multiplied by 4. F-d055 receives this scaled index. |
| Third link publication | F-d056–F-d059 have no explicit arguments. The F-d059 result selects a word slot, which receives the scaled index from the third wide-slot construction. F-d060 receives the new selecting index multiplied by 4. |
| Fourth wide-slot construction and capture | F-d061–F-d064 have no explicit arguments. The F-d064 result selects a wide slot, which receives that index multiplied by 4. A word is then read from the corresponding word slot and retained as the unsigned **captured word**. F-d065 receives the scaled index. |
| Captured-field publication | F-d066–F-d069 have no explicit arguments. The F-d069 result becomes the signed **retained index**. Its product with 4 is retained as the unsigned **retained offset**. The captured word shifted right by 12 is written at retained offset plus 20. F-d070 receives the retained offset. |
| Fifth wide-slot construction | F-d071–F-d074 have no explicit arguments. The F-d074 result selects a wide slot, which receives that index multiplied by 4. F-d075 receives the retained offset masked with `0x3F` and shifted left by 14, the new scaled index, and the retained index shifted left by 30. |
| Fifth link publication | F-d076–F-d079 have no explicit arguments. The F-d079 result selects a word slot, which receives the scaled index from the fifth wide-slot construction. F-d080 receives the new selecting index multiplied by 4. |

The two read expressions feeding F-d035 follow its wide-slot write and precede entry to the dependency. Their relative evaluation order and physical load count are not established. They must not be assumed to observe equal values under device-memory, concurrent-access, or other externally changing-memory conditions.

The captured word is read before F-d065. A later dependency mutation of its source address does not replace the already captured value in the visible data flow.

The retained offset’s unsigned representation does not resolve overflow in the signed multiplication that produces it. Its subsequent addition of 20 is represented as unsigned arithmetic. The retained index’s later left shift by 30 also requires a separately established signed-shift interpretation where the signed operation is not representable.

### Final selector updates and terminal boundary

F-d081–F-d084 receive no explicit arguments. The unsigned 32-bit result of F-d084 is retained as the **final word**.

A **selector address** is calculated from the retained index multiplied by `0x10000000`, plus 1. This address calculation is represented using signed arithmetic; wraparound is not independently established.

A byte is read from the selector address. One byte is then written at that zero-extended byte value plus 17. The stored byte is the low byte of the final word shifted left by 2.

F-d085 receives two copies of the final word shifted left by 2, followed by the final word shifted left by 4. Its target construction uses an offset of 4076, unlike the 4094 offset in the surrounding call pattern.

F-d086 receives no explicit arguments. Its captured 32-bit result is passed to F-d087.

F-d088–F-d091 receive no explicit arguments, and the result of F-d091 is retained. The selector address is read again as a byte. This new value becomes the **final selector**.

One byte is written at final selector plus 17. Its value is the low byte of the final word masked with `0x3FFFFFFF` and shifted right by 20. Consequently, the stored byte contains original final-word bits 20–27; bits 28–29 are discarded by byte truncation.

F-d092 receives the retained F-d091 result shifted left by 2. F-d093–F-d096 then receive no explicit arguments. F-d097 receives the F-d096 result shifted left by 2 and the zero-extended final selector. F-d098 receives no explicit arguments.

The representation then reaches its reported invalid-instruction boundary. No normal return, subsequent instruction behavior, or specific machine exception is established.

The two final selector reads are distinct accesses separated by calls. Their values and resulting write destinations may differ.

## Side effects

### Explicit memory footprint

If every visible operation is reached, the body contains:

- Three byte-read expressions.
- Three byte writes.
- **Fourteen four-byte writes.**
- Three word-read expressions: two feeding F-d035 and one capturing the word before F-d065.

The fourteen word writes comprise one initial-scale write and thirteen writes across the subsequent stages: paired-high publication, paired-low transformation, masked-field publication, five wide-slot constructions, and five associated publications.

These counts exclude all dependency effects. The word-read expression count does not establish two physical loads for F-d035.

### Aliasing and persistence

No accessed region is established to be disjoint from another. In particular:

- A word slot and wide slot selected by zero both address 20.
- Independent dependency results may select the same address.
- A later store may overwrite an earlier store.
- Byte destinations may overlap word storage or each other.
- Dependencies may alter, invalidate, or otherwise affect memory used by later operations.

There is no visible rollback or compensating action. Completed effects are not visibly undone if a later operation faults or fails to return.

### Ownership and resources

No explicit allocation, release, ownership transfer, or lifetime-management operation is established. Calculated indexing does not prove that storage exists or belongs to this function.

Dependencies may allocate, release, retain, or invalidate resources. No ownership or preservation guarantee is assigned to them.

## Dependencies

F-d001–F-d098 are unresolved indirect dependencies labeled by call occurrence.

The first destination depends on the incoming link-state value plus 4094. Later destination constructions use internally supplied bases, ordinarily with offset 4094; F-d085 uses 4076. These constructions do not establish dependency identities or calling conventions.

For every dependency, the following remain unknown:

- Its implementation and accepted argument ranges.
- Its machine-level argument and return convention.
- Whether it returns normally, faults, or transfers control elsewhere.
- Its effects on globals, memory, resources, and ambient execution state.
- Whether another occurrence resolves to the same destination.

No dependency is assumed to preserve globals or to own, retain, or release passed or indirectly reachable storage. None is established to be recursive.

The stated sequencing follows the supplied representation. It is not an independent proof that unresolved machine calls preserve every value required by that representation.

## Acceptance examples

These are conditional static examples, not executed tests. Unless a case concerns interruption, assume valid accesses, returning dependencies, and the stated arithmetic interpretation.

1. **Initial extraction.** An input word of `0xFFFFFFFF` produces an initial byte-write value of `0xFF`. An input word of `0xC0000000` produces zero: its highest two bits do not survive the conversion.

2. **Byte-sized selection.** If the initial selector read yields `0xFF`, the initial byte-write destination is numerical address 272. Treating that read as a full-width pointer would not satisfy the contract.

3. **Scale boundaries.** Scale sources `0x00000FFF`, `0x00001000`, and `0xFFFFFFFF` produce initial scales zero, `0x400`, and `0x3FFFFC00`, respectively.

4. **Separated result halves.** If the high half of F-d014’s result is 3, the initial-scale write targets address 32. The low half does not select that destination.

5. **Unsigned multiplication.** A paired low word of 16 makes the transformed-word destination address 20 under unsigned 32-bit arithmetic. The written value is `0x40000000`. This does not establish that address 20 is accessible.

6. **Masked-field limit.** A paired low word of `0xFFFFFFFF` produces `0x000FFFFF` for the write before F-d030.

7. **Wide-slot aliasing.** If F-d034 returns zero, its wide-slot write and both following word-slot read expressions address 20. With ordinary stable memory and no intervening modification, those reads observe the just-written zero.

8. **Captured-value timing.** If the word read before F-d065 is `0xABCDEF12`, the later write before F-d070 contains `0x000ABCDE`. A subsequent change to the original location does not alter that captured-value relationship.

9. **Distinct selectors.** If the first final selector read yields 4 and the second yields 9, the final byte-write destinations are 21 and 26. Caching the first selector for both writes would not match the visible accesses.

10. **Final-byte boundaries.** A final word of `0xFFFFFFFF` produces final byte values `0xFC` and `0xFF`. A final word of `0x30000000` produces zero for both writes.

11. **Partial completion.** If a dependency does not return after an earlier store completes, later stores and arrival at the terminal boundary are not required. No rollback is specified.

12. **Signed arithmetic limit.** A retained index for which multiplication by `0x10000000` exceeds signed representability does not, by this contract alone, establish a wrapped selector address. That case requires additional machine evidence.

## Uncertainties

The following remain **documented_only**:

- Application purpose, data structures, and intended invariants.
- Complete machine interface and hidden entry-state requirements.
- Accuracy of the represented function boundary, call signatures, and control flow.
- Whether the invalid-instruction boundary denotes reachable invalid execution, embedded data, or an inaccurate interpretation.
- Signed overflow, negative-index behavior, nonrepresentable signed shifts, and relevant signed conversions.
- Endianness, alignment, mapping, permissions, atomicity, and device-memory semantics.
- Physical load count and relative evaluation order for F-d035’s two read expressions.
- Dependency identities, preservation rules, exceptions, non-return behavior, global mutations, and ownership effects.
- Validity of calculated destinations.
- Behavior beyond the reported terminal boundary.

These unknowns are not resolved by repeated arithmetic patterns or by the conditional examples.

## Evidence

**Established static evidence:** The supplied body shows the apparent argument widths, additional entry-state uses, 98 indirect call occurrences, captured results, masks and shifts, access expressions, and terminal invalid-instruction indication. Its explicit memory effects include fourteen four-byte writes: one initial-scale write and thirteen subsequent stage writes.

**Inferred intent:** Repeated scaled-index and wide-slot operations may participate in structured state construction. This inference adds no layout, allocation, validity, or ownership guarantee.

**Unknown:** No dependency implementation, independently established target ABI, memory map, executable trace, or validated continuation beyond the terminal boundary is available.

No runtime tests have been performed. Acceptance examples express static arithmetic and ordering consequences, subject to the execution conditions and unresolved machine semantics stated above.
