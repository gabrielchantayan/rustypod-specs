<!-- Trusted provenance: F-3ca9dea8fab9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3ca9dea8fab9

## Purpose

F-3ca9dea8fab9 describes a recovered sequence of indirect calls, coprocessor-directed operations, fixed-width bit transformations, and a final 16-bit store. The supplied representation ends at an invalid-instruction boundary, not a normal return.

No application-level purpose is established. The evidence does not support labeling the sequence as initialization, decoding, validation, allocation, or device configuration. Coprocessor-directed operations are present, but their architectural effects are unknown.

This contract separates three evidence levels:

- **Established static evidence:** represented input storage widths, operation order, call arguments, consumed results, and the explicit store.
- **Conditional arithmetic deductions:** consequences of interpreting words as 32-bit bit vectors and wide results as 64-bit bit vectors.
- **documented_only unknowns:** ABI details, dependency behavior, architectural semantics, and recovery accuracy not established by the supplied material.

No runtime tests have been performed.

## Interface

### Entry state

The displayed interface declares no parameters and no return value. Its body nevertheless consumes incoming machine state.

| Input role | Represented storage width | Use |
|---|---:|---|
| First incoming data word | 32 bits | Supplies left shifts by 22 and 11. |
| Second incoming data word | 32 bits | Supplies left shifts by 7 and 29. |
| Third incoming data word | 32 bits, explicitly unsigned | Supplies a logical right shift by 5. |
| Incoming link value | 32-bit integer representation | Supplies the first indirect target through addition of `0xC3C`. |
| First incoming coprocessor-selector value | 32 bits | Used with the fourth selector in the first coprocessor operation. |
| Second incoming coprocessor-selector value | 32 bits | Used with the fourth selector in three coprocessor operations. |
| Third incoming coprocessor-selector value | 32 bits | Used with the fourth selector in the primary-result operation. |
| Fourth incoming coprocessor-selector value | 32 bits | Used in every coprocessor operation, including both trailing positions of one operation. |

Each of the four coprocessor-selector inputs has an established **recovered 32-bit storage width**. That fact does not establish its architectural meaning, how many bits an instruction effectively uses, or whether the recovery accurately identifies the original operands. The values must not be described as having an unknown represented width; their **effective selector widths and interpretation** remain unknown.

The data words and selectors are implicit machine-state inputs. Their presence does not establish a conventional public calling convention or a parameter list.

### Results and termination

No normal return or caller-visible return value is represented.

If every preceding operation completes, the sequence attempts a 16-bit store of `0x00CD` and then reaches an invalid-instruction termination marker. The marker’s runtime effect is **documented_only**. It is not established as an orderly shutdown, recoverable error, architectural halt, or particular exception.

### Preconditions for reaching the end

The complete path requires:

- Reached indirect targets to be executable under the applicable execution mode and calling convention.
- Every dependency to return before execution can continue.
- Coprocessor-directed operations to complete under the applicable support and privilege rules.
- The final destination to permit the 16-bit write, including applicable alignment requirements.
- The recovered representation to correspond sufficiently closely to actual executable instructions.

No explicit checks enforce these conditions.

## Behavior

### Width and arithmetic model

A **word** denotes 32 bits. A **wide result** denotes 64 bits. A wide result’s low half is bits 0–31; its high half is bits 32–63.

For conditional bit-vector deductions:

- Word left shifts discard bits beyond bit 31.
- Explicitly unsigned right shifts insert zeros.
- Word arguments preserve the resulting 32-bit patterns; a dependency’s interpretation of those patterns is unknown.
- Splitting a wide result into halves is a numerical operation, not a memory-layout or endianness claim.

Some displayed left shifts have signed operands. The deductions below use fixed-width machine arithmetic, not an assertion that all such expressions have defined semantics in a high-level language.

The ordered phases below describe the represented path. If an operation faults, transfers control elsewhere, or fails to return, later phases are not thereby guaranteed.

### Initial dispatch and first returned payload

The first observable operation is coprocessor-directed. Its fixed numeric tuple is `(0, 5, 7)`. Its data payload is the second input word shifted left by 7, followed by the first and fourth incoming selector values.

F-000000000001 is then invoked at a target formed from the incoming link value plus `0xC3C`. Its sole explicit argument is the same shifted input word.

Next, F-000000000002 is invoked with no explicit arguments, followed by F-000000000003 with no explicit arguments. The latter supplies a word result.

That result becomes the data payload of a coprocessor operation with tuple `(0, 6, 7)`, followed by the second and fourth incoming selectors. F-000000000004 is then invoked with no explicit arguments.

“No explicit arguments” describes only the displayed call expression. It does not prove that a dependency ignores registers, memory, global state, or other machine state.

### Primary-result processing

F-000000000005 receives three word arguments, in this order:

1. First input word shifted left by 22.
2. First input word shifted left by 11.
3. Second input word shifted left by 29.

Its returned word is the **primary result**.

The next coprocessor operation has tuple `(0, 5, 7)`. Its data payload is the primary result shifted left by 1, followed by the third and fourth incoming selectors.

F-000000000006 then receives four arguments:

1. Primary result shifted left by 1.
2. Third input word logically shifted right by 5.
3. Primary result shifted left by 4.
4. Zero.

F-000000000007 and F-000000000008 follow, each with no explicit arguments.

The next coprocessor operation has tuple `(0, 7, 7)`, zero data payload, and the second and fourth incoming selectors.

An intermediate word is then formed by retaining the low two bits of the primary result shifted left by 4, and shifting that masked value left by 16. Under the fixed-width interpretation, this intermediate is always zero: shifting left by 4 already clears both bits selected by the mask.

A further coprocessor operation follows with tuple `(0, 0, 7)`, zero data payload, and the fourth incoming selector in both trailing positions.

F-000000000009 receives one explicit zero argument and supplies the **secondary result**, a word.

The sixth and final coprocessor operation has tuple `(0, 2, 7)`. Its payload is the primary result shifted left by 12, followed by the second and fourth incoming selectors.

F-00000000000a then receives:

1. Primary result shifted left by 12.
2. Primary result shifted left by 16.
3. Secondary result shifted left by 9.

The shifts by 12 and 16 are bit-vector simplifications of additional shifts applied to the already shifted primary value.

### Wide-result processing and word-result chain

F-00000000000b is invoked with no explicit arguments and supplies the **first wide result**.

F-00000000000c receives its low half, its high half, and its low half shifted left by 1, in that order. No result from this call is consumed by the displayed body.

F-00000000000d receives the masked intermediate shifted left by 8. Under the fixed-width interpretation, this argument is zero. Its word result is passed to F-00000000000e, whose word result is passed to F-00000000000f.

F-000000000010 follows with no explicit arguments. F-000000000011 then supplies the **second wide result**, also through a call with no explicit arguments.

F-000000000012 receives:

1. The second wide result’s low half.
2. Its high half.
3. Its high half shifted left by 11.

This call supplies the **final wide result**.

### Final calls and store

Before the remaining calls, two values are derived:

- The final wide result logically shifted right by 41, retained as a word.
- The masked intermediate shifted left by 25, which remains zero under the fixed-width interpretation.

F-000000000013 receives the final wide result’s low half.

F-000000000014 is then invoked with no explicit arguments and returns a word. F-000000000015 receives that word first and the shifted intermediate second. Its second argument is consequently zero under the stated interpretation.

F-000000000016 follows with no explicit arguments. Only after it returns does the explicit 16-bit store occur.

Let \(n\) be bits 52–63 of the final wide result, interpreted as an unsigned integer. The represented destination calculation reduces to:

\[
\operatorname{address}=n+\left\lfloor n/4\right\rfloor.
\]

The derivation first logically shifts the final wide result right by 41. This produces at most 23 significant bits, so converting the resulting word to a signed interpretation does not set its sign bit. The subsequent signed right shift by 11 therefore yields \(n\), without sign extension affecting the result. The second summand is an unsigned right shift of that value by 2.

Consequences:

- \(0\leq n\leq4095\).
- The destination lies between `0x00000000` and `0x000013FE`, inclusive.
- There is no separate base-address addition.
- Destinations can be zero or odd.
- Not every address within that interval is produced.
- The stored quantity is exactly 16 bits with value `0x00CD`.

There is no visible null check, bounds check, alignment correction, or permission check. After the store, the representation reaches the invalid-instruction boundary.

## Side effects

### Explicitly represented effects

The body contains, in order across the phases above:

- Six coprocessor-directed operations.
- Twenty-two indirect dependency calls.
- One explicit 16-bit data-memory store.
- A terminal invalid-instruction boundary.

No ordinary data-memory load is explicitly represented in the body. This does not exclude dependency accesses, instruction fetches, implicit architectural effects, or accesses absent from an inaccurate recovery.

The store’s byte order is unknown. On a byte-addressed little-endian target, the two bytes would be `CD 00`; on a byte-addressed big-endian target, they would be `00 CD`. Neither arrangement is established for the actual target.

### Dependency effects and state preservation

Dependencies may read or modify memory, globals, devices, or machine state. They may fault or not return. Their effects are not bounded by this contract.

The represented dataflow carries values across calls, but no independent calling-convention evidence establishes how those values are preserved in actual execution. In particular, unknown callees must not be assumed to preserve globals or other shared state.

The coprocessor tuples do not establish cache, barrier, translation, synchronization, or device-configuration semantics.

### Ownership

No allocation, release, reference-count change, or ownership transfer is established.

Arguments are raw words or numerical halves of wide results. They are not established as pointers or owned resources. No dependency is assumed to acquire, preserve, or release ownership.

The final destination’s ownership and memory classification are unknown. It could identify ordinary memory, a device mapping, protected memory, or an unmapped location.

## Dependencies

These opaque IDs identify call sites for this contract. Distinct IDs do not prove distinct runtime targets.

| Dependency | Explicit arguments | Result consumed by this body |
|---|---|---|
| F-000000000001 | One shifted input word; link-relative target | None |
| F-000000000002 | None | None |
| F-000000000003 | None | Word |
| F-000000000004 | None | None |
| F-000000000005 | Three shifted input words | Primary word |
| F-000000000006 | Three derived words and zero | None |
| F-000000000007 | None | None |
| F-000000000008 | None | None |
| F-000000000009 | Zero | Secondary word |
| F-00000000000a | Three derived words | None |
| F-00000000000b | None | First wide result |
| F-00000000000c | Low half, high half, shifted low half | None |
| F-00000000000d | Shifted masked intermediate; conditionally zero | Word |
| F-00000000000e | Previous word result | Word |
| F-00000000000f | Previous word result | None |
| F-000000000010 | None | None |
| F-000000000011 | None | Second wide result |
| F-000000000012 | Low half, high half, shifted high half | Final wide result |
| F-000000000013 | Final wide result’s low half | None |
| F-000000000014 | None | Word |
| F-000000000015 | Previous word result and shifted intermediate | None |
| F-000000000016 | None | None |

“None” in the result column means that no result is consumed, not that the dependency has no return value.

All targets after F-000000000001 are represented through code-location-relative address construction. Their identities, executable validity, calling conventions, implementations, and possible aliasing remain **documented_only**.

## Acceptance examples

These are conditional specification examples, not executed tests. They assume the stated bit-vector model and completion of preceding operations.

### Input truncation

If the second input word is `0x02000001`, shifting it left by 7 produces `0x00000080` after 32-bit truncation. That is the payload of the first coprocessor operation and the explicit argument to F-000000000001.

Passing an untruncated wider product would not match this interpretation.

### Unsigned right shift

If the third input word is `0xFFFFFFFF`, F-000000000006 receives `0x07FFFFFF` as its second argument.

An arithmetic right shift yielding `0xFFFFFFFF` would not match the represented unsigned operation.

### Primary-result propagation

If F-000000000005 returns `0x00000001`:

- The following coprocessor payload is `0x00000002`.
- F-000000000006 receives `0x00000002` first and `0x00000010` third.
- The final coprocessor payload is `0x00001000`.
- F-00000000000a receives `0x00001000` first and `0x00010000` second.
- F-00000000000d receives zero.
- F-000000000015 receives zero as its second argument.

The zero arguments follow from masking, not from any assumption about dependency behavior.

### Wide-result ordering

If F-00000000000b returns `0x1122334480000001`, F-00000000000c receives:

1. `0x80000001`.
2. `0x11223344`.
3. `0x00000002`.

If F-000000000011 returns `0x8000000100000002`, F-000000000012 receives:

1. `0x00000002`.
2. `0x80000001`.
3. `0x00000800`.

These examples distinguish half ordering and 32-bit truncation.

### Final destinations

If the destination permits the access:

| Final wide result’s bits 52–63 | Computed destination | Store |
|---:|---:|---|
| `0x000` | `0x00000000` | 16-bit `0x00CD` |
| `0x001` | `0x00000001` | 16-bit `0x00CD` |
| `0x004` | `0x00000005` | 16-bit `0x00CD` |
| `0xFFF` | `0x000013FE` | 16-bit `0x00CD` |

Lower bits of the final wide result do not affect this destination. They can still affect the argument previously passed to F-000000000013.

Odd destinations are computed outcomes, not guarantees that an unaligned write succeeds.

### Interrupted execution

If F-000000000012 does not return, the later calls, store, and terminal boundary are not reached through the represented path. No cleanup or recovery branch is shown.

## Uncertainties

The following remain **documented_only**:

- Whether the recovered sequence accurately represents valid executable instructions.
- The public ABI, physical locations of implicit inputs, and actual execution mode.
- Architectural address width, address-space interpretation, endianness, and alignment behavior.
- The architectural meaning and effective bit width of each coprocessor selector, despite each having known recovered 32-bit storage.
- Coprocessor operation semantics, privileges, and fault behavior.
- Dependency identities, contracts, side effects, preservation rules, and ownership behavior.
- Whether apparent wide returns are scalar returns or a recovery convention combining machine results.
- The actual mechanism preserving represented values across calls.
- The runtime consequence of the invalid-instruction boundary.
- The intended meaning and validity of the low computed destination.

No normal-return contract, error code, retry, rollback, or cleanup action is established. Absence of explicit handling does not establish absence of faults.

## Evidence

The sole evidence is the supplied recovered representation, including its bad-instruction warnings.

It statically establishes the displayed operation order, argument expressions, consumed result widths, six coprocessor-directed operations, twenty-two calls, and final 16-bit store. The four incoming coprocessor-selector values each have an explicitly represented four-byte storage type, establishing 32-bit recovered storage independently of their unknown architectural interpretation.

The zero-intermediate deductions and final-address formula are conditional arithmetic results under 32-bit word and 64-bit wide-result semantics. They are not runtime observations.

No dependency implementations, trusted ABI specification, verified instruction stream, or memory-map documentation were supplied. No runtime tests have been performed.
