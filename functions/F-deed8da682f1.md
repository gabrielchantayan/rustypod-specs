<!-- Trusted provenance: F-deed8da682f1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-deed8da682f1

## Purpose

The supplied static representation describes an operation that interleaves dependency calls with byte reads, byte writes, and four-byte reads and writes. Incoming machine state, dependency results, and a global word determine addresses and stored values.

No application-level purpose is established. The repeated storage patterns do not establish allocation, serialization, device control, or a particular data structure.

The represented path has no ordinary return. It ends at an invalid-instruction boundary. This establishes a limit of the available representation, not a deliberate application-level halt or a known complete execution.

This contract distinguishes:

- **Established static evidence:** widths, accesses, arguments, and ordering explicitly represented.
- **Inferred intent:** possible interpretation, not a behavioral requirement.
- **documented_only:** behavior that cannot be resolved from the supplied material.

All behavior below is conditional on execution reaching the relevant operation.

## Interface

### Explicit inputs

| Input position | Represented width and interpretation | Visible use |
|---|---|---|
| First | 32-bit quantity, interpretation unspecified | Argument to the first indirect dependency. |
| Second | 32-bit quantity, interpretation unspecified | No explicit use on the represented path. |
| Third | Signed 32-bit integer | Shifted left by 26 for an early dependency argument. |
| Fourth | Signed 32-bit integer | Shifted left by 10 for an early dependency argument. |

The second input’s lack of explicit use does not establish irrelevance under the unresolved calling convention.

### Additional incoming state

The operation also requires:

- An incoming byte used by the first store.
- An incoming address used for the first selector read.
- An incoming unsigned 32-bit quantity used for another selector address.
- Incoming return-link-like state used to calculate the first indirect target.
- Secondary machine-result state consumed after several calls.

The four explicit inputs alone are insufficient to reproduce the represented behavior.

### Outputs and preconditions

No explicit external return value is established. Outputs include memory accesses, dependency invocations, dependency effects, and possible faults or abnormal termination.

A late dependency supplies a represented 64-bit result. Its low and high 32-bit halves are subsequently consumed internally.

Successful progress requires accessible call targets and suitable readable or writable memory at every computed address. The representation contains no explicit validation of those requirements. Alignment requirements, memory permissions, and complete calling-convention requirements are **documented_only**.

### Widths and address notation

A **selector update** reads an unsigned byte from a specified selector address, then writes one byte at address 17 plus that byte. Destinations range from 17 through 272 inclusive.

A **word slot at an index** denotes the four-byte location at address 20 plus four times that index. This notation does not imply an array, allocation, or valid bounds.

Byte stores retain the low eight bits. Unsigned 32-bit right shifts are logical. Unsigned 32-bit arithmetic wraps modulo \(2^{32}\).

Some shifts, multiplications, and address calculations have signed 32-bit operands. Their exact execution semantics are **documented_only**. Conditional wrapping examples below do not resolve signed-overflow or signed-left-shift behavior. Pointer width, byte order, atomicity, and concurrency guarantees are likewise unresolved.

## Behavior

### Entry and early dependency results

The first represented memory access reads a selector byte at the incoming address plus one. The incoming byte is then stored at address 17 plus that selector. Both operations precede the first dependency call.

The first indirect target depends on incoming return-link-like state. Its explicit argument is the first input.

After intervening calls, an indirect dependency receives the third input shifted left by 26, followed by the initial selector widened without sign extension.

A later call supplies **first result**, an unsigned 32-bit word. The following argument-bearing call receives first result twice and the fourth input shifted left by 10. After another call, a selector update clears a byte. Its selector address is the incoming unsigned 32-bit quantity shifted right by nine, plus one.

The next argument-bearing call receives an associated secondary-result word twice, followed by zero. After intervening calls:

- A selector is read at first result shifted right by 12, plus one.
- Its destination receives the low byte of the associated secondary-result word shifted right by 22.
- A newly call-produced signed 32-bit value, shifted left by two, is passed onward.
- A later call receives the captured selector followed by zero.

A subsequent dependency supplies **second result**, an unsigned 32-bit word. The next argument-bearing call receives second result twice and first result shifted left by 10.

After another call, a word slot indexed by signed secondary-result state receives second result.

The secondary-result roles used here and below describe dependencies represented in machine state. They do not establish conventional return types or preservation guarantees.

### Global-dependent updates

Immediately after that word-slot store, two represented reads access the same global word:

1. The first supplies a captured unsigned 32-bit value.
2. The second supplies a value multiplied by four to form the **global-derived base**, represented as signed 32-bit state.

These reads are not established as one atomic snapshot.

An indirect dependency receives the preceding secondary-result index. After intervening calls, a newly returned signed index selects a word-slot store of second result shifted right by 12. The next call receives four times that index.

A subsequent dependency supplies **third result**, an unsigned 32-bit word. A selector update reads at global-derived base plus one and stores the low byte of third result shifted left by two.

The following argument-bearing dependency receives the captured first global value masked with `0x3fffffff`, then shifted right by 10.

After intervening calls, the selector at global-derived base plus one is read again. Its destination receives the low byte of third result masked with `0x3fffffff`, then shifted right by 20. This is a fresh selector read, not reuse of the earlier byte.

Subsequent calls consume a newly produced signed word shifted left by two, and later another shifted call result followed by the newly captured selector.

The global-derived base is also shifted left by 26 to form a **shifted global base**. After another call, two separate four-byte reads at shifted global base plus 28 supply a dependency’s two arguments. The representation does not establish which of these two reads occurs first.

The shifted global base is multiplied by 1024 to form the **multiplied global-derived base**. After the dependency receiving the two memory values and a further call, a selector update reads at multiplied global-derived base plus one and clears its destination.

Under wrapping 32-bit arithmetic, the shift by 26 followed by multiplication by 1024 produces zero. That reduction is conditional: signed intermediates prevent this contract from asserting it unconditionally.

### Pre-direct-call arguments and linked stores

The next argument-bearing dependency receives:

- Zero.
- Associated unsigned secondary-result state.
- The multiplied global-derived base.

After intervening calls, a fresh selector read at multiplied global-derived base plus one selects a byte store containing the low byte of that secondary-result state shifted right by 22.

A call then receives a newly produced signed value shifted left by two. Following further calls, another argument-bearing indirect dependency receives, in order:

1. **The multiplied global-derived base shifted left by 26.**
2. **A newly call-produced signed value shifted left by two.**

The first argument retains its global-read dependency. It is not derived from the newly produced value. Its signed arithmetic remains unresolved; it must not be specified as unconditionally zero.

The immediately following indirect call supplies **fourth result**, a signed 32-bit value. **F-674e75a953e8** is then invoked with fourth result twice.

After another indirect call, a word slot indexed by associated unsigned secondary-result state receives fourth result. The next argument-bearing dependency receives that index and the index masked with `0x3fffffff`, then shifted right by 10.

After intervening calls, a newly returned signed index selects a word-slot store of the preceding secondary-result index. The next call receives four times the new index.

A subsequent call supplies **fifth result**, an unsigned 32-bit word. A selector update reads at eight times fourth result plus one and stores the low byte of four times fifth result.

A dependency receives four times fifth result. After intervening calls, another selector update reads at 64 times fifth result plus one. It stores the low byte of fifth result masked with `0x3fffffff`, then shifted right by 20.

Subsequent argument-bearing calls receive:

- A newly produced signed value shifted left by two.
- Later, another shifted call-produced value followed by the captured selector.
- Later, fourth result shifted left by 10, supplied twice.

After another call, a selector read at address 1 selects a byte to clear. The zero base for this particular access is explicit, rather than dependent on reducing the global-derived arithmetic.

### Secondary-result updates and the split result

The next argument-bearing call receives associated unsigned secondary-result state twice, followed by 16 times that state.

After intervening calls, a selector update reads at 16 times that secondary-result state plus one and stores its low byte after a logical right shift by 22.

Later calls receive a newly produced signed value shifted left by two, and then the captured selector followed by another newly produced signed value shifted left by two.

A subsequent dependency receives two zeros. The next indirect call supplies the represented 64-bit result:

- **Low half:** the low 32 bits, used as unsigned state.
- **High half:** the high 32-bit pattern, subsequently represented as signed state.

The operation stores zero in the word slot indexed by high half, then calls a dependency with high half. After intervening calls, a newly returned signed index selects a word-slot store containing high half. The following call receives four times that new index.

The signed interpretation of high half matters to unresolved signed address arithmetic. It does not change which 32-bit pattern is stored.

### Low-half-derived stores

After further calls, a four-byte store writes low half multiplied by `0x04000000` at address low half multiplied by `0x10000000`, plus 20. Both products are unsigned 32-bit expressions.

A dependency receives low half multiplied by `0x04000000`. After intervening calls, a returned signed index selects a word-slot store containing low half masked with `0x1fffffff`, then shifted right by nine. The next call receives four times that index.

### Subsequent word-storage sequence

The following stages are ordered. Each newly returned index is distinct as a data role; equality between indices is neither required nor excluded.

1. A returned signed index is multiplied by four. That scaled value is stored at address 20 plus 16 times the index. The next argument-bearing call receives two separate four-byte reads from address 20 plus the scaled value: one logically shifted right by 12 and the other shifted left by two. The scaled value is the middle argument. Relative order of the two reads is unspecified.
2. After intervening calls, another word slot receives that scaled value. A dependency receives four times the destination index.
3. A new returned signed index is scaled by four and stored at address 20 plus 16 times itself. A dependency receives the scaled value twice, followed by low half shifted left by 26.
4. After intervening calls, another word slot receives this scaled value. A dependency receives four times its destination index.
5. Another returned signed index is scaled by four and stored at address 20 plus 16 times itself. A dependency receives the scaled value. After intervening calls, a new word slot receives that same scaled value, followed by a call with four times its destination index.
6. Another returned signed index is scaled by four and stored at address 20 plus 16 times itself. A four-byte unsigned value is then read at address 20 plus this scaled value and retained across subsequent calls. A dependency receives the scaled value.
7. After intervening calls, a newly returned signed index, the **retained index**, selects a word-slot store of the retained memory value logically shifted right by 12. Four times retained index is captured as an unsigned 32-bit **retained scaled index** and passed to a dependency.
8. Another returned signed index is scaled by four and stored at address 20 plus 16 times itself. The next argument-bearing call receives, in order: retained scaled index masked to six bits and shifted left by 14; the newest scaled value; and retained index shifted left by 30.
9. After intervening calls, another word slot receives the newest scaled value. A dependency receives four times that destination index.

The six-bit mask in stage 8 applies to the **retained scaled index**, not directly to retained index.

Addresses using a factor of 16 are not interchangeable with word-slot addresses using a factor of four. All these accesses may alias.

### Final byte updates and endpoint

After further calls, an indirect dependency supplies **final byte-source word**, an unsigned 32-bit value.

A selector update reads at retained index multiplied by `0x10000000`, plus one. Its destination receives the low byte of final byte-source word shifted left by two. The address multiplication has a signed operand; its execution semantics remain unresolved.

The following argument-bearing dependency receives the shifted-left-by-two value twice, followed by final byte-source word shifted left by four. This call has a different represented target displacement from the surrounding indirect-call pattern.

A later dependency supplies a 32-bit quantity that is passed to another indirect dependency. After additional calls, a fresh selector read uses the same retained-index-derived address. Its destination receives the low byte of final byte-source word masked with `0x3fffffff`, then shifted right by 20.

Further calls consume newly produced signed values shifted left by two, including a later call that also receives the captured selector. One final indirect call follows. The representation then ends at the invalid-instruction boundary.

## Side effects

Directly represented memory operations are unguarded. No explicit null checks, bounds checks, error-status handling, rollback, or synchronization appears.

Important constraints:

- Selector bytes are unsigned; selector 255 chooses address 272.
- Four-byte writes are not byte writes.
- Repeated selector reads can choose different destinations.
- Repeated four-byte reads remain separate represented accesses.
- Captured values and fresh memory reads are not interchangeable.
- Dependencies may alter globals, selector storage, word storage, or other state.

If a later operation faults, earlier stores precede it on the represented path, and no rollback is explicitly represented. This does **not** guarantee persistence or external observability: unresolved dependencies may overwrite or restore memory before faulting, and visibility guarantees are unknown.

No explicit allocation, release, ownership transfer, or lifetime extension is established. Dependency behavior does not justify assuming any such operation, or assuming its absence.

## Dependencies

### F-674e75a953e8

This dependency is called once on the represented path with fourth result supplied twice. The call precedes a further indirect call and the word-slot store of fourth result.

Its return convention, memory effects, ownership behavior, failure behavior, and preservation of machine state are **documented_only**.

### Unresolved indirect dependencies

The supplied mapping provides no opaque function IDs for the indirect targets. They remain unidentified and must not be equated with one another or with F-674e75a953e8.

The first target depends on incoming machine state. Later targets are predominantly expressed relative to fixed code-position values, with several different displacements.

Calls displaying no explicit arguments do not establish a zero-argument ABI. Primary results, secondary machine-result state, and a represented 64-bit result are consumed.

Intervening calls are part of the represented sequence, including calls without listed arguments. The behavioral grouping above does not authorize removing them, combining them, or assuming they preserve memory or machine state.

## Acceptance examples

These are conditional static obligations, not executed tests. Each assumes the relevant operation is reached and its required accesses succeed.

1. **Initial selector widening.** If the first selector is 255 and the incoming byte is `0xab`, the first store writes `0xab` at address 272 before the first dependency call.

2. **Byte truncation.** If third result is `0x123456ff`, its shifted-left-by-two selector update stores `0xfc`.

3. **Fresh selectors.** If the two reads at global-derived base plus one yield 4 and 9, the respective byte destinations are 21 and 26. Reusing the first selector is incorrect.

4. **Global-derived pre-direct-call argument.** The two-argument dependency immediately before the call producing fourth result receives multiplied global-derived base shifted left by 26 first, and the new call-produced value shifted left by two second. Conditional on wrapping 32-bit semantics throughout the relevant base arithmetic, the first argument is zero. If the new value is `0x12`, the second argument is `0x48`. Without that arithmetic condition, zero is not established for the first argument.

5. **Split result.** If the represented 64-bit result is `0x0000000300000007`, the first following word-slot store writes zero at address 32. The subsequent separately indexed store writes 3, not 7.

6. **Unsigned address wrap.** If low half is `0x10`, its `0x10000000` address product wraps to zero. The corresponding four-byte store targets address 20 and contains `0x40000000`.

7. **Scaled-index mask.** If retained index is 5 and its multiplication by four produces 20, the later three-argument call’s first argument is `0x50000`: the six-bit mask applies to 20, not 5.

8. **Failure after a store.** If a dependency faults after a preceding store, the represented order includes that store before the faulting call and contains no explicit rollback. The contract does not require the stored bytes to remain unchanged or externally visible.

## Uncertainties

The following remain **documented_only**:

- Validity and completeness of the represented instruction stream.
- Full signature, calling convention, and incoming-state requirements.
- Origins and preservation rules for secondary results.
- Identities and contracts of indirect dependencies.
- Signed arithmetic and address-conversion semantics.
- Memory mapping, alignment, permissions, endianness, and atomicity.
- Global-word meaning and whether its separate reads differ.
- Ordinary, volatile, or device-backed memory behavior.
- Ownership, lifetime, concurrency, and external visibility.
- Actual execution beyond the invalid-instruction boundary.

Structured-state manipulation is a possible interpretation of the repeated patterns, not an established purpose.

## Evidence

This contract derives from the supplied static representation and opaque dependency mapping. Prior prose and reviewer feedback are not independent behavioral evidence.

The supplied representation warns about invalid instruction data and overlapping global symbols. Those warnings limit confidence in complete executable behavior; they do not justify repairing or simplifying the represented path.

Established details include access widths, masks, distinct reads, data origins, argument order, and represented sequencing. In particular, the pre-fourth-result argument retains its global-derived origin, and the absence of explicit rollback establishes no persistence guarantee.

No runtime tests have been performed. Successful execution, endpoint reachability, and application-level intent are not established.
