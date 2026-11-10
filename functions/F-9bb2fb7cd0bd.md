<!-- Trusted provenance: F-9bb2fb7cd0bd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9bb2fb7cd0bd

## Purpose

F-9bb2fb7cd0bd performs a finite sequence of byte writes and indirect calls, then enters an unconditional repeating sequence of calls and byte writes. The represented control flow has no normal return.

Its established contract is an ordering and data-dependence contract. Input bits and dependency results determine stored bytes, selector-read addresses, call arguments, and one call site’s first-versus-later dispatch target. No application-level purpose, protocol, or successful completion condition is established.

This description is limited to the supplied static representation. That representation includes warnings about bad instruction data and removed unreachable regions. Whether it accurately describes executable behavior remains unresolved.

## Interface

### Explicit inputs

The represented interface has four positional, 32-bit inputs:

| Position | Representation | Established use |
|---|---|---|
| 1 | Unsigned 32-bit word | Supplies two stored bytes and all three explicit arguments to F-U01. |
| 2 | Opaque 32-bit value | No explicit use in the represented body. |
| 3 | Opaque 32-bit value | No explicit use in the represented body. |
| 4 | Unsigned 32-bit word | Supplies a write address and the argument to F-U13 after a logical right shift by 7. |

Unused explicit inputs are not established as unobservable to dependencies. Actual calling conventions and residual machine state are unknown.

### Hidden inputs and access preconditions

Two incoming machine-context values are consumed without initialization in the represented body:

- A context-register value supplies the base address for selector-byte reads at an offset of one byte.
- A link-register value supplies the base for F-U01’s computed call target.

Their origins, validity, and public-interface status are unknown. They are required context for the represented execution, not additional established public parameters.

Progress through the described events requires valid memory accesses, valid call targets, and returning dependencies. The function contains no explicit checks establishing those conditions.

### Outputs and termination

The represented return type is void. No returned status, error value, or ownership-bearing result is shown.

The finite prefix leads directly into an unconditional loop. The function itself supplies no exit condition. A fault, nonreturning dependency, external interruption, or other transfer of control could prevent further events, but none is handled locally.

### Widths and arithmetic

Explicit memory accesses are one byte wide. Loaded selector bytes are unsigned and therefore range from 0 through 255. A byte store retains only the lowest eight bits of its value.

Unsigned inputs and unsigned captured results are 32 bits wide. Their right shifts are logical. Unsigned left shifts retain the low 32 bits.

Several dependency results are instead represented as signed 32-bit values before left shifts. The selector-address calculation also uses signed multiplication and addition. Exact overflow behavior for these signed operations is unresolved.

Where a result below is qualified by **low-32-bit arithmetic**, it assumes fixed-width machine operations retaining the low 32 bits. That qualification does not establish portable signed-expression semantics.

## Behavior

### Selector-based byte writes

Two distinct selector mechanisms are used.

**Context-selected write.** Each occurrence reads one unsigned byte at the incoming context base plus one. It adds 17 to the byte’s numeric value and writes one byte at that resulting numeric address.

The destination is therefore between 17 and 272 inclusive, or 0x11 through 0x110. It is not the context base plus the selector value.

**Result-selected write.** Each occurrence reads one unsigned byte at an address derived from an earlier dependency result. It likewise adds 17 to the loaded byte and writes at that numeric destination.

Each occurrence performs a fresh selector read. Neither the selector byte nor the selected destination is cached across occurrences.

No non-aliasing property is established. A selected destination may overlap selector storage, another destination, or other accessible memory. Dependencies may also change selector storage between reads.

### Dependency notation

F-U01 through F-U36 label call sites in execution order through the first traversal. They are opaque contract-local identifiers, not independently established function identities.

F-U27 has two dispatch variants:

- **F-U27-A:** the first invocation at that call site.
- **F-U27-B:** every subsequent invocation at that call site.

These variants have independently unresolved targets and behavior. A single call-site label must not be read as evidence of a stable target.

“No explicit arguments” below means that the representation supplies none. It does not establish the complete machine-level interface.

### Initial writes and result capture

The first observable direct memory operation is a context-selected write of input 1’s low byte. This precedes every represented dependency call.

The next calls occur in this order:

| Site | Explicit arguments | Result consumed by this function |
|---|---|---|
| F-U01 | Input 1 in all three positions | None |
| F-U02 | None | Opaque 32-bit result |
| F-U03 | F-U02’s captured result | None |
| F-U04 | None | None |
| F-U05 | None | None |
| F-U06 | None | None |
| F-U07 | None | Signed 32-bit result |

After F-U07 returns, a fresh context-selected write stores the low byte of input 1 logically shifted right by 22. The stored byte consists of original input bits 22 through 29. Bits 30 and 31 do not survive byte truncation.

The following calls then occur:

| Site | Explicit arguments | Result consumed by this function |
|---|---|---|
| F-U08 | F-U07’s result shifted left by 2 | None |
| F-U09 | None | None |
| F-U10 | None | None |
| F-U11 | None | None |
| F-U12 | None | Unsigned 32-bit result |

F-U08’s signed left shift has unresolved overflow semantics.

Call F-U12’s captured result the **address-driving word**. This is a descriptive label, not evidence that the dependency returns an address.

### Input-derived destination and additional calls

The address-driving word is shifted left by two as an unsigned 32-bit operation. Its low 32-bit result is retained in a signed 32-bit intermediate. The shift itself therefore has defined low-32-bit behavior in the representation; the interpretation of an out-of-range unsigned-to-signed conversion remains unresolved.

The function stores the low byte of that intermediate at the numeric address:

- input 4 logically shifted right by seven;
- then increased by 28.

This destination ranges from 28 through 33,554,459 inclusive. The addition cannot overflow an unsigned 32-bit value. The stored byte is the address-driving word’s low six bits shifted left by two, so its lowest two bits are zero.

Immediately afterward, with no intervening call, a context-selected write stores zero. Its selector read occurs after the input-derived write. If the preceding write aliases that selector storage, ordinary sequential memory semantics make the new byte available to this read.

The next calls occur in this order:

| Site | Explicit arguments | Result consumed by this function |
|---|---|---|
| F-U13 | Input 4 logically shifted right by 7 | None |
| F-U14 | None | None |
| F-U15 | Address-driving word masked to its low 30 bits, then logically shifted right by 6 | None |
| F-U16 | None | None |
| F-U17 | None | None |
| F-U18 | None | None |
| F-U19 | None | Signed 32-bit result |

F-U15 receives a value between 0 and 16,777,215 inclusive. Its argument consists of original address-driving bits 6 through 29. Original bits 30 and 31 are discarded before the shift.

After F-U19 returns, another fresh context-selected write stores zero.

The next calls are:

| Site | Explicit arguments | Result consumed by this function |
|---|---|---|
| F-U20 | F-U19’s result shifted left by 2 | None |
| F-U21 | None | None |
| F-U22 | None | None |
| F-U23 | None | None |
| F-U24 | None | Unsigned 32-bit result |

F-U20’s signed shift has unresolved overflow semantics.

### Transition into repetition

F-U24’s result is masked to its low 30 bits and logically shifted right by 25. The resulting **five-bit selector** ranges from 0 through 31 and contains original result bits 25 through 29.

A result-selected write then stores zero. Its selector-read address is derived from the signed intermediate retained after F-U12: multiply that intermediate by 1,024, then add one.

Under low-32-bit arithmetic, this address equals the address-driving word shifted left by 12, truncated to 32 bits, plus one. The represented signed conversion, multiplication, and addition prevent treating this as an unconditional signed-language equivalence.

The following calls occur:

| Site | Explicit arguments | Result consumed by this function |
|---|---|---|
| F-U25 | Five-bit selector shifted left by 1 | None |
| F-U26 | None | Signed 32-bit result |

F-U25 receives an even value from 0 through 62.

After F-U25 and before F-U26, the function establishes the target-base state for F-U27-A. That state remains the represented target base when the first loop iteration begins.

### Repeating phase: dispatch and arguments

Each iteration begins with the F-U27 call site, but its dispatch target changes after the first iteration:

| Invocation | Dispatch variant | Computed numeric target |
|---|---|---|
| First | F-U27-A | 0x08833787 |
| Every later invocation | F-U27-B | 0x088337C5 |

These are target values from the represented address arithmetic. They do not establish instruction-set interpretation, executable validity, callable signatures, or dependency behavior.

The first target derives from the base established before F-U26. During each iteration, the function replaces the target-base state with the later-target base immediately before calling F-U36. The loop back edge does not restore the initial base. Consequently, the next F-U27 invocation uses F-U27-B.

The target transition is independent of F-U36’s returned value. Its result controls argument feedback, not the selection between these two targets.

F-U27’s explicit arguments are:

| Invocation | First argument | Second argument |
|---|---|---|
| First | F-U26’s captured result | Zero |
| Later | Previous iteration’s F-U36 result shifted left by 2 | Current unsigned loop word logically shifted right by 8 |

The initial loop word is the five-bit selector. Its right shift by eight is necessarily zero.

For later iterations, the loop word is formed from the preceding F-U36 result shifted left by 26, then held as unsigned 32 bits. Under low-32-bit arithmetic, F-U27-B’s second argument is therefore the previous F-U36 result’s low six bits shifted left by 18.

### Repeating phase: remaining event order

After F-U27-A or F-U27-B returns, each iteration continues in this order:

1. F-U28, F-U29, and F-U30 are called, with no explicit arguments.
2. F-U31 is called with no explicit arguments; its signed 32-bit result is captured.
3. A result-selected zero write occurs, using the same retained address-driving intermediate as before the loop. The selector byte is reread.
4. F-U32 receives F-U31’s result shifted left by two.
5. F-U33, F-U34, and F-U35 are called, with no explicit arguments.
6. The target-base state is set for F-U27-B.
7. F-U36 is called with no explicit arguments; its signed 32-bit result is captured.
8. That result supplies the next iteration’s first argument through a left shift by two, and its unsigned loop word through a left shift by 26.

The signed shifts involving F-U31 and F-U36 have unresolved overflow semantics. There is no represented exit check between iterations.

## Side effects

The explicit direct memory effects are:

- Four context-selected byte writes, storing input 1’s low byte, input 1’s bits 22–29, zero, and zero, respectively.
- One byte write at the input-4-derived address.
- One result-selected zero write before repetition.
- One result-selected zero write after F-U31 in each iteration that reaches that point.

Every selector-based write includes a preceding byte read. Reaching the loop write does not imply that the iteration subsequently completes.

No explicit allocation, release, ownership transfer, bulk memory access, synchronization, or local error handling appears. This absence does not constrain dependency behavior.

Dependencies may modify memory or globals, affect resources, fail to return, or transfer control elsewhere. No preservation or ownership guarantee is established. The specified state feedback describes assignments in the represented body, not an independently verified ABI guarantee that arbitrary callees preserve machine state.

## Dependencies

F-U01’s target is formed from the incoming link-register value plus 4,076. Its validity, arithmetic behavior at problematic address boundaries, and interface are unresolved.

Other targets are computed from fixed location values or a code reference. F-U27 specifically dispatches to two different computed targets across its first and later invocations. F-U27-A and F-U27-B must remain independently unresolved dependencies; neither target’s behavior can be inferred from the other.

All listed calls are observable ordering obligations when execution reaches them. An ignored result does not make a call optional or side-effect-free.

The supplied dependency mapping establishes only F-9bb2fb7cd0bd’s identity. It supplies no dependency contracts, target validity guarantees, resource rules, or assurances that similarly represented calls reach the same function.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. They require valid accesses and targets, returning dependencies, and ordinary sequential memory where memory observations are described.

1. **Initial truncation and ordering.** If input 1 is 0x123456AB and the first context selector is 3, the first write stores 0xAB at address 20. It occurs before F-U01.

2. **Upper input-byte extraction.** If input 1 is 0xC0400000, the context-selected write after F-U07 stores 1. Input bits 30 and 31 do not contribute.

3. **Input-derived address.** If input 4 is 0x100 and F-U12 returns 0x43, the input-derived write stores 0x0C at address 30. The subsequent context selector is read only afterward.

4. **Fresh context selector.** If the first context selector is 1 and F-U07 changes it to 2, the initial write targets address 18 and the write after F-U07 targets address 19, assuming no other relevant changes.

5. **Masking precedes shifting.** If F-U12 returns 0xC0000040, F-U15 receives 1, not the result of an unmasked right shift.

6. **Initial loop arguments.** If F-U24 returns 0xFE000000, the five-bit selector is 31 and F-U25 receives 62. The first F-U27 invocation uses F-U27-A at computed target 0x08833787, receives F-U26’s result first, and receives zero second.

7. **Dispatch and feedback together.** Under low-32-bit arithmetic, if the first F-U36 returns 0x41, the second F-U27 invocation uses F-U27-B at computed target 0x088337C5 and receives 0x104 and 0x40000. It does not reuse the first invocation’s target.

8. **No target reset.** If several iterations complete, the third and subsequent F-U27 invocations also use F-U27-B. F-U36’s returned values may change their arguments but do not restore F-U27-A.

9. **Selector changes affect destinations.** If successive result-selector reads yield 7 and 9, the corresponding zero writes target addresses 24 and 26, even though their selector-read address is derived from the same retained word.

## Uncertainties

The following remain **documented_only**:

- Executable fidelity of the supplied representation, given its instruction-data and reachability warnings.
- Actual calling conventions, hidden-context initialization, and dependency interfaces.
- Validity and instruction-set interpretation of every computed target, including both F-U27 variants.
- Whether addressed storage is ordinary memory, mapped hardware, or another facility.
- Unsigned-to-signed conversion behavior outside the signed range and overflow behavior of represented signed arithmetic.
- Dependency effects on globals, memory, machine context, ownership, and resources.
- Fault behavior, external interruption, and any termination path absent from the representation.

The limited inferred intent is manipulation of small values through calls and byte-selected destinations. It does not establish a device operation, data format, protocol, or success criterion.

## Evidence

Established static evidence consists of the supplied function representation and its accompanying warnings. It supports the explicit input widths, byte accesses, event ordering, argument transformations, retained-result dependencies, unconditional loop, and F-U27’s first-versus-later target transition.

The dispatch distinction follows from target-base initialization before F-U26, replacement before F-U36, and the absence of a reset on the loop back edge. Both target behaviors remain unresolved.

No runtime tests have been performed. No dependency implementation, executable validation, external interface documentation, ownership contract, or error-handling contract was supplied. Conditional arithmetic examples are interpretations under their stated model, not observed executions.
