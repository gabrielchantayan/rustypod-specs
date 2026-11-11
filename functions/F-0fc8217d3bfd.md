<!-- Trusted provenance: F-0fc8217d3bfd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0fc8217d3bfd

## Purpose

The supplied static representation describes a stateful sequence of indirect calls, memory reads, and memory writes. A finite initialization phase is followed by an unconditional repeating phase with no displayed normal return.

The repeating phase carries both argument state and dispatch state. In particular, its first indirect call uses one target expression on the first iteration and a different target expression on subsequent iterations. This transition follows from explicit carrier assignments; it does not require dependency side effects.

No application-level purpose is established. Field extraction from call results suggests packed data, but no format, protocol, device operation, allocation scheme, or conventional API can be identified reliably.

This contract is conditional on execution reaching the displayed operations, dependencies returning where required, and memory accesses succeeding. Decoding and reachability warnings limit confidence that the supplied representation completely and accurately describes executable behavior.

## Interface

### Explicit inputs

| Input | Displayed width and interpretation | Explicit use |
|---|---|---|
| First argument | Signed 32-bit integer | Multiplied by four; used in a word-write address, stored in memory, and passed in scaled form to F-U01. |
| Second argument | Uninterpreted 32-bit quantity | No explicit use. |
| Third argument | Uninterpreted 32-bit quantity | No explicit use. |
| Fourth argument | Unsigned 32-bit integer | Logically shifted right by seven; used to address a byte write and as an argument to F-U23. |

The absence of explicit uses of the second and third arguments does not establish that indirect dependencies cannot observe their incoming machine state.

### Additional entry state

Two additional incoming values affect the displayed behavior:

- An unsigned 32-bit register value, logically shifted right by eight to obtain a saved selector base.
- A link-state value used to derive the target of F-U01.

Their origin and calling convention are unknown. The four explicit arguments alone do not specify all required entry state.

### Output, termination, and failure

No result value or normal return path is displayed. After initialization, the function repeats indefinitely unless execution is interrupted by a dependency that does not return, a failed memory access, another external event, or behavior absent from the static representation.

There is no displayed status result, error branch, exception handler, rollback, or cleanup protocol.

### Arithmetic and memory preconditions

Reached targets must be callable, and reached memory accesses must be valid at their displayed widths. No checks establish these conditions.

The representation uses 32-bit quantities and address arithmetic. Logical right shifts and unsigned masks have explicit bit-level meanings. Unsigned 32-bit left shifts retain the low 32 bits. However:

- Multiplication of the first signed argument by four can overflow.
- Several dependency results undergo signed left shifts.
- Some unsigned results are assigned to signed 32-bit intermediates.
- The loop-selector address uses multiplication of a signed intermediate.
- Address conversion, alignment, and overflow behavior are not established.

Consequently, formulas that assume wrapping 32-bit machine arithmetic are identified as conditional. They do not establish defined signed-arithmetic behavior for every input.

Byte-write descriptions below specify the retained eight-bit patterns, rather than a signed interpretation of a character value.

## Behavior

### Derived values and call notation

The following descriptive terms are used:

- **Scaled first input:** four times the first argument.
- **Initial word address:** scaled first input plus decimal 20.
- **Saved selector address:** the incoming unsigned register value shifted right by eight, plus one.
- **Fourth-input base:** the fourth argument logically shifted right by seven.
- **Saved-selector write:** read one unsigned byte at the saved selector address, add decimal 17 to the zero-extended value, and write one byte at that resulting address.
- **Loop-selector write:** perform the same read-and-address operation using the loop selector address instead.

Each selector read is a separate observable memory operation. A previous selector value or destination is not reused.

F-U01 through F-U46 are opaque call-site IDs. They do not imply 46 distinct target functions. “None” in the tables means no explicit arguments are displayed, not that the actual dependency has a proven zero-argument interface.

### 1. Initial word writes

The first displayed memory operation writes the original first argument as a 32-bit word at the initial word address.

The saved selector base is then derived, before F-U01. The following calls occur in order:

| Call site | Explicit arguments | Result retained by this function |
|---|---|---|
| F-U01 | Scaled first input | No |
| F-U02 | None | No |
| F-U03 | None | No |
| F-U04 | None | No |
| F-U05 | None | Yes |

After F-U05 returns, a second 32-bit write stores the scaled first input at the same initial word address.

Dependencies therefore execute between the two writes. They may observe or modify the location; the second write is not conditional on its current contents.

### 2. First packed result and two selected-byte writes

The next calls are:

| Call site | Explicit arguments | Result retained |
|---|---|---|
| F-U06 | F-U05’s result shifted left by two | No |
| F-U07 | None | No |
| F-U08 | None | No |
| F-U09 | None | No |
| F-U10 | None | Yes |

Call F-U10’s unsigned 32-bit result the **first packed result**.

A saved-selector write stores its low eight bits after shifting left by two. Original bits 0 through 5 become stored bits 2 through 7; stored bits 0 and 1 are zero.

The next calls occur in this order:

| Call site | Explicit arguments | Result retained |
|---|---|---|
| F-U11 | Three copies of the first packed result shifted left by two | No |
| F-U12 | None | Yes |
| F-U13 | F-U12’s 32-bit result | No |
| F-U14 | None | No |
| F-U15 | None | No |
| F-U16 | None | No |
| F-U17 | None | Yes |

A second saved-selector write then stores the byte obtained by masking the first packed result to its low 30 bits, logically shifting right by 20, and retaining eight bits.

The stored byte consists of original bits 20 through 27. Original bits 28 and 29 survive the intermediate shift but not the final byte conversion.

The selector byte is reread after F-U17. The destination can differ from the first selected-byte destination because intervening dependencies, or an earlier aliasing write, may have changed the selector.

### 3. Second packed result and additional writes

The next sequence is:

| Call site | Explicit arguments | Result retained |
|---|---|---|
| F-U18 | F-U17’s result shifted left by two | No |
| F-U19 | None | No |
| F-U20 | None | No |
| F-U21 | None | No |
| F-U22 | None | Yes |

Call F-U22’s unsigned 32-bit result the **second packed result**.

Two byte writes follow, without an intervening dependency call:

1. At fourth-input base plus decimal 28, store the low eight bits of the second packed result shifted left by two.
2. Perform a saved-selector write storing zero.

The first write has a directly calculated destination; it does not read a selector. Its byte contains original bits 0 through 5 in output bits 2 through 7.

The subsequent calls are:

| Call site | Explicit arguments | Result retained |
|---|---|---|
| F-U23 | Fourth-input base | No |
| F-U24 | None | No |
| F-U25 | Second packed result masked to 30 bits, then logically shifted right by six | No |
| F-U26 | None | No |
| F-U27 | None | No |
| F-U28 | None | No |
| F-U29 | None | Yes |

F-U25 receives an unsigned value from 0 through 16,777,215 representing original bits 6 through 29 of the second packed result.

After F-U29 returns, another saved-selector write stores zero.

### 4. Loop preparation

The following calls occur next:

| Call site | Explicit arguments | Result retained |
|---|---|---|
| F-U30 | F-U29’s result shifted left by two | No |
| F-U31 | None | No |
| F-U32 | None | No |
| F-U33 | None | No |
| F-U34 | None | Yes |

The function masks F-U34’s unsigned result to its low 30 bits and logically shifts right by 25. The resulting **initial loop field** contains original bits 25 through 29 and ranges from 0 through 31.

The loop selector address is derived separately from the second packed result:

- Its previously computed left-by-two value supplies a signed 32-bit intermediate.
- That intermediate is multiplied by 1,024.
- One is added to obtain the selector-read address.

Under a wrapping 32-bit bitvector interpretation, this address equals the second packed result shifted left by 12, plus one. Signed multiplication and conversion uncertainties prevent making that an unconditional language-level identity.

A loop-selector write now stores zero.

The final preparation calls are:

| Call site | Explicit arguments | Result retained |
|---|---|---|
| F-U35 | Initial loop field shifted left by one | No |
| F-U36 | None | Yes |

F-U35 receives an even value from 0 through 62. F-U36’s result becomes the initial **loop-carried argument**.

Before F-U36 is invoked, the representation assigns the carrier that will determine F-U37’s first target. Define opaque dispatch labels:

- **D-INITIAL:** the initial carrier plus decimal 4,094.
- **D-RECURRING:** the recurring carrier plus decimal 4,094.

The carriers are distinct displayed constants; the recurring carrier exceeds the initial carrier by decimal 62. These labels describe target expressions, not recovered function identities.

### 5. Repeating phase: calls, writes, and dispatch transition

Each iteration begins with:

| Call site | Explicit arguments | Result retained |
|---|---|---|
| F-U37 | Current loop-carried argument; current loop field logically shifted right by eight | No |
| F-U38 | None | No |
| F-U39 | None | No |
| F-U40 | None | No |
| F-U41 | None | Yes |

F-U37 uses **D-INITIAL on the first iteration**.

A loop-selector write then stores zero. The selector byte is reread on every iteration.

The remaining calls are:

| Call site | Explicit arguments | Result retained |
|---|---|---|
| F-U42 | F-U41’s result shifted left by two | No |
| F-U43 | None | No |
| F-U44 | None | No |
| F-U45 | None | No |
| F-U46 | None | Yes |

Immediately before F-U46 is invoked, the recurring carrier is assigned. That carrier remains the displayed dispatch state across the loop boundary. Consequently, **F-U37 uses D-RECURRING on the second and every later iteration**.

This transition is independent of F-U46’s returned value. The return value updates argument state, not the target carrier. No assignment at the next iteration’s entry restores the initial carrier.

F-U46’s signed 32-bit result supplies the next argument state:

- The next loop-carried argument is that result shifted left by two.
- The next loop field is that result shifted left by 26 and assigned to an unsigned 32-bit quantity.

These shifts have the signed-arithmetic limitations described above. Under a 32-bit bitvector interpretation, only the result’s low six bits survive the latter shift.

On the first iteration, F-U37’s second argument is zero because the initial loop field is at most 31. Under that same bitvector interpretation, later iterations pass the preceding F-U46 result’s low six bits shifted left by 18.

The next iteration begins unconditionally. Neither a zero result nor an unchanged selector supplies an exit condition.

## Side effects

### Direct memory accesses

Before the repeating phase, the displayed body performs:

- Two 32-bit writes to the initial word address.
- Four saved-selector byte reads, each followed by a selected byte write.
- One directly addressed byte write based on the fourth argument.
- One loop-selector byte read followed by a selected zero write.

Each iteration adds one loop-selector byte read and one selected zero write.

Every selector-mediated destination lies between decimal addresses 17 and 272 inclusive, because the selector is an unsigned byte. These low addresses must not be assumed to represent valid heap fields or conventional application memory.

### Ordering and aliasing

The ordering of calls, reads, and writes is part of the displayed behavior.

A dependency may change a selector before its next read. A selected destination may itself equal a selector-read address, allowing a write to affect a later selection.

The directly addressed write following F-U22 precedes the saved-selector read for the following zero write. If its destination aliases that selector address, it can change the zero-write destination without any intervening dependency call.

The loop selector address remains derived from the saved second-packed-result intermediate. Its contents are not fixed: dependencies and aliasing writes may change the byte read there.

These observations do not establish volatile access semantics, atomicity, synchronization, or visibility to concurrent observers.

### Ownership and external state

No explicit allocation, release, ownership transfer, or object-retention protocol is shown.

Dependencies may read or change globals, memory, resources, or other external state. Nothing establishes that they preserve globals or refrain from allocating or releasing memory. The contract describes the explicit state flow without inventing dependency guarantees.

## Dependencies

F-U01 through F-U36 occur in the finite prefix. F-U37 through F-U46 identify repeating call sites.

F-U01’s target depends on incoming link state. Other target expressions use displayed address-like carriers and offsets. Their validity, linkage, and ABI remain unknown.

F-U37 requires a distinction beyond its call-site ID:

| Invocation | Target-expression label | Carrier establishment |
|---|---|---|
| First | D-INITIAL | Assigned immediately before F-U36 |
| Second and later | D-RECURRING | Assigned immediately before the preceding F-U46 |

Both target expressions add decimal 4,094 to their respective carriers. Their displayed values differ by decimal 62. This proves a dispatch-expression transition in the representation, but not that the destinations implement different logical functions: entry aliases, shared implementations, and decoding errors remain unresolved.

F-U36’s result initializes F-U37’s first argument. F-U46’s result determines later arguments. Neither result is displayed as the source of F-U37’s target.

No explicit argument list or result-use annotation establishes a complete dependency contract.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Initial word replacement.** With first argument 3 and successful accesses, the first word write stores 3 at decimal address 32. After F-U05 returns, the second stores 12 at the same address. Intervening dependency effects remain permitted.

2. **Low-byte extraction.** If the first packed result is 63, the first saved-selector write stores 252. If it is 64, that write stores zero. This is truncation, not saturation.

3. **Upper-byte extraction.** If the first packed result is hexadecimal 0x3AB00000, the second saved-selector write stores hexadecimal 0xAB. The intermediate shifted value includes bits discarded by the byte write.

4. **Fourth-input addressing.** With fourth argument 128, the direct byte-write address is decimal 29. A second packed result of 3 causes that write to store 12.

5. **Selector rereading.** If successive saved-selector reads yield 2 and 9, their associated writes target decimal addresses 19 and 26. Reusing the first destination would not match the representation.

6. **Initial loop boundary.** If F-U34 returns hexadecimal 0xFFFFFFFF, the initial loop field is 31, F-U35 receives 62, and the first F-U37 invocation receives zero as its second argument.

7. **Dispatch changes despite zero data.** If F-U36 and all reached loop dependencies return zero, the first F-U37 invocation still uses D-INITIAL, while the second and later invocations use D-RECURRING. Both explicit arguments may remain zero; identical argument values do not eliminate the target transition.

8. **Later argument state.** Under 32-bit bitvector shift semantics, if F-U46 returns 63, the next loop-carried argument is 252 and the next F-U37 second argument is hexadecimal 0x00FC0000. That invocation uses D-RECURRING. This example does not assert defined signed-left-shift behavior for those values.

9. **No zero-result exit.** Returning zero throughout the repeating phase does not terminate it. Each completed iteration still includes its selector read, selected zero write, and remaining calls.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and any packed-result format.
- The true ABI and origin of hidden entry state.
- Dependency identities, contracts, termination behavior, and resource effects.
- Whether D-INITIAL and D-RECURRING identify distinct logical implementations.
- Whether all displayed instructions and reachable paths were recovered correctly.
- Signed overflow, negative left shifts, and out-of-range unsigned-to-signed conversion behavior.
- Address overflow, alignment, endianness, memory mapping, and device-memory interpretation.
- Fault behavior, externally imposed termination, and dependency-induced invalidation of later accesses.

An inferred intent is that call results contain fields consumed through masking and shifting. That interpretation supplies no semantic names for those fields.

The dispatch transition is not merely inferred intent: distinct initial and recurring carrier assignments are explicit in the supplied representation. What those targets do remains unknown.

## Evidence

Established static evidence consists solely of the supplied representation: displayed input widths, arithmetic, masks, memory-access widths, call order, result forwarding, carrier assignments, and unconditional loop structure.

The initial F-U37 target carrier is established before F-U36. The recurring carrier is established before F-U46 and reaches the next F-U37 without restoration of the initial value. This supports the first-versus-later dispatch distinction independently of assumptions about dependency side effects.

The material reports invalid instruction data and removed unreachable regions. These warnings limit confidence in decoding and completeness; they do not justify replacement behavior.

No implementation access, dependency inspection, runtime execution, or runtime tests have been performed. All acceptance examples are conditional deductions from static evidence.
