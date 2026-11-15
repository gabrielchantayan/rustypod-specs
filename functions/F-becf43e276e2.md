<!-- Trusted provenance: F-becf43e276e2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-becf43e276e2

## Purpose

F-becf43e276e2 specifies a repeating sequence of indirect calls, computed-address memory accesses, and carried-state updates. The supplied representation contains an unconditional back edge and no normal return.

No application-level purpose is established. Structured addressing or encoding is plausible **inferred intent**, not a contract identifying an allocator, serializer, initializer, or constructor.

This document distinguishes:

- **Established static evidence:** operations, widths, sequencing, and target relationships visible in the supplied representation.
- **Conditional arithmetic interpretation:** calculations under a wrapping 32-bit machine-word model.
- **documented_only:** behavior that cannot be resolved from the supplied evidence.

No runtime tests have been performed.

## Interface

### Arguments and inherited state

| Input | Represented width | Visible role |
|---|---:|---|
| First argument | 32 bits | Input to the iteration’s first call; replaced near the back edge. |
| Second argument | 32 bits | Neither explicitly consumed nor assigned in the visible body. |
| Third argument | 32 bits | Supplies shifted call arguments and a word-store payload; replaced by the last call. |
| Fourth argument | 32 bits | Input to an early call; replaced before the final store, final read, and final two calls. |

The unused second argument is not proven irrelevant to the actual ABI or unresolved dependencies.

Additional entry state is required:

- An address-like selector base.
- A 32-bit initial payload.
- A continuation-like value determining the first call target.
- An accessible global 32-bit word.
- An additional represented 32-bit result-register value consumed by a later call.

The first three are carried or replaced between iterations. The additional result-register value has unresolved provenance; it is not established as an ordinary caller argument.

There is no represented return value. If all calls return and all accesses complete, another iteration begins. Successful infinite execution is not guaranteed: calls may not return, and accesses may fault.

### Widths, addressing, and preconditions

Selector reads are unsigned 8-bit reads. Byte stores retain the low eight bits of their payload. Word accesses are 32 bits. Certain call results are represented as 64 bits and split into low and high 32-bit halves.

Arithmetic descriptions below conditionally retain the low 32 bits after scaling, shifting, and address calculation. This is a machine-word interpretation, not a claim that displayed signed expressions have portable overflow behavior.

A **selector update at a base** means:

- Read one unsigned byte at that base plus one.
- Write a byte at the selector’s numeric value plus `0x11`.

The selector’s source base is not added to the destination. Thus selector-directed byte destinations lie between `0x11` and `0x110`, inclusive.

Successful progress requires valid instruction targets, a compatible actual calling convention, and valid memory accesses. The body does not check these requirements. Alignment, byte order, address-space permissions, atomicity, and fault delivery are **documented_only**.

## Behavior

### Call notation and observable granularity

Call numbers below identify encounter order within one iteration. Unless a target relationship is stated otherwise, call number 001 refers to opaque dependency reference **F-000000000001**, and similarly through 146.

A five-call group contains five separate calls. Its first call receives the listed arguments; its remaining four calls have no explicitly represented arguments. Its last call supplies the consumed result, if any. No explicit arguments does not imply an empty machine-level input state.

| Calls | First-call arguments | Consumed final result |
|---|---|---|
| 001–005 | First argument | Index |
| 006–010 | Four times result 005 | 32-bit value |
| 013–017 | Result 012; result 010 shifted left 30 | Index |
| 018–022 | Four times result 017 | None |
| 023–027 | Fourth argument | 64-bit result |
| 030–034 | Result 029; first scaled value shifted left 28 | Index |
| 035–039 | Four times result 034 | Index |
| 040–044 | Four times result 039, supplied twice | Index |
| 045–049 | First scaled value shifted left 14 | Index |
| 050–054 | Four times result 049 | 64-bit result |
| 062–066 | Four times result 061 | Retained index |
| 069–073 | Result 068; third argument shifted left 28 | Index |
| 074–078 | Four times result 073 | None |
| 081–085 | Propagated payload shifted left 28; additional result-register value | Index |
| 086–090 | Four times result 085 | Index |
| 098–102 | Four times result 097 | Index |
| 103–107 | Four arguments detailed below | Index |
| 108–112 | Four times result 107 | Index |
| 113–117 | Four times result 112, supplied three times | Index |
| 118–122 | Four times result 117 | Index |
| 123–127 | Four times result 122, supplied three times | Index |
| 128–132 | Four times result 127 | 32-bit value |
| 135–139 | Result 134 | Index |
| 140–144 | Four times result 139 | Index |

Memory events described below occur between these groups. They must not be moved across calls: dependency effects and memory aliasing are unresolved.

### 1. Initial payload and retained selector

After calls 001–005, store the inherited 32-bit payload at four times result 005 plus `0x14`.

Calls 006–010 produce a 32-bit value. Read the selector at the inherited base plus one, then store the low byte of four times result 010 at its selector destination.

Next:

- Call 011 receives four times result 010.
- Call 012 has no explicit arguments and produces a 32-bit token.
- Calls 013–017 consume that token and the shifted value listed above.

Read the selector at the inherited base plus one again. Retain this particular selector byte. At its selector destination, store the low byte of result 010 masked with `0x3FFFFFFF` and logically shifted right 20.

After calls 018–022, store the third argument shifted left two as a 32-bit word at the retained selector’s numeric value plus `0x14`. This store does not reread the selector, even if intervening calls changed its source byte.

### 2. Captured global and first wide result

Read and retain the global 32-bit word before calls 023–027.

Define the **first scaled value** as the low 32 bits of result 027 shifted left two. Freshly read the selector at the inherited base plus one. Its destination receives the low byte of the captured global logically shifted right 22. A dependency changing the global after capture does not change this retained payload.

Call 028 receives the high half of result 027. Call 029 has no explicit arguments and produces a token. After calls 030–034, freshly read the inherited-base selector and clear its destination.

Calls 035–039 produce an index. Calls 040–044 receive four times that index twice and produce another index. Define the **first propagated payload** as four times result 044.

Store this payload as a word at the first scaled value multiplied by `0x1000`, plus `0x14`.

After calls 045–049, store the same retained payload at four times result 049 plus `0x14`. These two stores do not consume independently returned payloads.

### 3. Second wide result and derived selector sources

Calls 050–054 produce a 64-bit result. Define the **second scaled value** as its low half shifted left two.

Read a selector at the first scaled value multiplied by `0x4000`, plus one. Its destination receives the low byte of the first propagated payload masked with `0x3FFFFFFF` and logically shifted right 20.

The following calls occur individually:

| Call | Explicit arguments or consumed result |
|---|---|
| 055 | Receives the high half of result 054 |
| 056 | No explicit arguments; produces a token |
| 057 | Receives result 056 |
| 058 | No explicit arguments; produces a 64-bit result |
| 059 | Receives result 058’s low half, then high half |
| 060 | No explicit arguments |
| 061 | No explicit arguments; produces an index |

Read a selector at the second scaled value multiplied by `0x10000000`, plus one, and clear its destination.

Calls 062–066 produce the **retained index**. Retain both that index and its four-times value.

Define the **working selector base** as four times the second scaled value. Read its selector and clear the destination.

### 4. Shifted payload propagation and intervening clear

Call 067 receives the second scaled value shifted left 10. Call 068 has no explicit arguments and produces a token.

Calls 069–073 receive that token and the third argument shifted left 28. **Immediately after call 073, freshly read the working-base selector and clear its destination. Only then perform calls 074–078**, beginning with four times result 073. Their final result is not explicitly consumed.

Define the **shifted payload** as the retained index, result 066, shifted left 14. Store it as a word at the retained index’s four-times value multiplied by `0x1000`, plus `0x14`.

Call 079 receives the retained index’s four-times value shifted left 14. Call 080 has no explicit arguments.

Calls 081–085 receive the shifted payload shifted left 28 and an additional represented 32-bit result-register value. The latter’s producer and meaning remain **documented_only**. It must not be replaced with zero or assumed to equal an ordinary return value.

Store the shifted payload at four times result 085 plus `0x14`.

Calls 086–090 produce another index. Retain its four-times value. Freshly read the working-base selector and store the low byte of the shifted payload masked with `0x3FFFFFFF` and logically shifted right 20 at its destination.

### 5. Further payload propagation

Call 091 receives the retained index, result 066, shifted left 26. Call 092 produces a token without explicit arguments. Call 093 receives that token and the four-times value of result 090 shifted left 28.

Call 094 produces a 64-bit result without explicit arguments. Call 095 receives its low half followed by its high half. Call 096 has no explicit arguments. Call 097 has no explicit arguments and produces an index.

Freshly read the working-base selector and clear its destination.

Calls 098–102 produce another index. Define the **new payload** as four times result 102. Store it at the four-times value of result 090 multiplied by `0x1000`, plus `0x14`.

Calls 103–107 begin with four arguments, in this order:

1. The four-times value of result 090 multiplied by `0x400`.
2. The same value again.
3. Result 090 shifted left 14.
4. The new payload.

Store the same new payload at four times result 107 plus `0x14`.

### 6. Self-derived stores and direct clears

After calls 108–112, store four times result 112 at sixteen times result 112 plus `0x14`.

After calls 113–117, store that same four-times payload at four times result 117 plus `0x14`.

After calls 118–122, store four times result 122 at sixteen times result 122 plus `0x14`. Then replace the inherited selector base with result 122 multiplied by `0x40000000`.

After calls 123–127, store four times result 122 at four times result 127 plus `0x14`. Retain result 127 separately: it supplies the final word-store payload.

Calls 128–132 produce a retained 32-bit value. Perform two direct byte stores:

1. Zero at the new inherited base plus `0x11`.
2. Zero at that base plus one.

These stores do not read or follow a selector.

Call 133 receives result 132 shifted left two. Call 134 has no explicit arguments and produces a token. Calls 135–139 consume that token.

Repeat the same direct zero stores, again writing the `+0x11` byte before the `+1` byte.

### 7. Final accesses and state-update order

Calls 140–144 produce the final index.

The remaining order is:

1. Replace the fourth argument with four times result 144.
2. Store result 127 shifted left 31 at sixteen times result 144 plus `0x14`.
3. Read a 32-bit word at `0x14` plus result 132 masked with `0x3FFFFFFF` and logically shifted right 20.
4. Call 145 with the already-replaced fourth argument.
5. Replace the continuation-like state with a fixed value.
6. Call 146 without explicit arguments; its result replaces the third argument.
7. Replace the inherited payload with zero.
8. Replace the first argument with the final word-read value logically shifted right 22, then shifted left 28.
9. Take the back edge.

The second argument has no explicit assignment. The inherited selector base remains the value established after result 122.

Under wrapping 32-bit arithmetic, the final stored payload depends only on result 127’s low bit. The final read’s starting address lies between `0x14` and `0x413`, inclusive, and need not be aligned.

## Side effects

Visible effects include selector reads, byte writes, 32-bit stores, a global word read, and a final computed-address word read. Calls may introduce additional effects between any of them.

Required distinctions include:

- Fresh selector reads are separate accesses, not cached reads.
- The early retained selector survives calls before its word store.
- The global-derived payload uses the captured word, not a later global reread.
- The working-base clear after call 073 precedes calls 074–078.
- Both direct-clear pairs write `+0x11` before `+1`.
- Fourth-argument replacement precedes the final store, final read, and calls 145–146.
- Third-argument replacement occurs through call 146; first-argument replacement follows it.

No visible bounds checks, error-status branches, rollback, synchronization, release operation, or cleanup appear.

Ownership is not established. Address-like arguments do not prove borrowing, retention, allocation, or release. Unknown callees must not be assumed to preserve globals or manage memory.

Memory regions may alias. In particular, stores may affect later selector or word reads. No disjointness is established among selector sources, global storage, low-address destinations, and computed word destinations.

## Dependencies

### Target identity versus unknown behavior

Indirect invocation does not make every target address unknown. The representation contains inherited-state-dependent, symbol-dependent, and explicitly constant-derived targets.

Dependency references identify target expressions, not verified semantic function boundaries.

- **F-000000000001:** initial target depends on inherited continuation-like state.
- **F-000000000147:** fixed target used for the first call on subsequent iterations, after the continuation replacement.
- Symbol-dependent references: **002–005, 007–010, 014–017, 019–022, 039, 041–042, 046–047, 051–052, 058–059, 063–064, 107–108, and 134–135**, each expanded using the opaque-ID convention above.
- All other references from **006–146**, excluding those symbol-dependent entries, have targets determined by explicit constant arithmetic.

Static arithmetic establishes two equal-target relationships:

- **F-000000000129 = F-000000000133** as numeric call targets.
- **F-000000000141 = F-000000000145** as numeric call targets.

Among the explicit constant-derived targets, including F-000000000147, all remaining pairs are numerically distinct under the stated arithmetic interpretation.

No target equality or inequality is established between the unresolved symbol-dependent targets and these fixed targets, or between the initial inherited target and any other target. Different symbol names alone do not supply trusted address relationships.

Numeric equality does not guarantee equivalent effects: repeated calls can receive different arguments and encounter changed memory or machine state.

### Unresolved contracts

For every dependency, semantic identity, calling convention, register preservation, memory and global effects, ownership behavior, errors, non-return, and reentrancy remain **documented_only**.

The represented wide results may be logical 64-bit returns or recovered register pairs. No error result is visibly checked before subsequent work.

## Acceptance examples

These examples are conditional static consequences, not runtime observations.

1. **Selector addressing:** A selector of `0x7F` selects byte address `0x90`. The early retained-selector word store would use `0x93`, independently of the selector’s source base.

2. **Truncation:** If result 010 is `0x12345678`, its four-times byte payload is `0xE0`; its masked-and-right-shifted byte payload is `0x23`.

3. **Captured global:** Capturing `0xFFC00000` produces a later byte payload of `0xFF`, even if a dependency changes the global before that write.

4. **Retained selector:** An early retained selector of `0x20` selects word address `0x34`. A subsequent change of its source byte to `0x40` does not retarget that store.

5. **Clear ordering:** If call 073 completes and the fresh working-base selector is `0x20`, address `0x31` is cleared before call 074 begins. A side-effecting call 074 must therefore encounter the post-clear state, absent another intervening effect.

6. **Word wrapping:** A third argument of `0x40000001` produces a left-two word payload of `0x00000004`.

7. **Direct clears:** If the newly established inherited base is zero, each direct pair writes zero to `0x11` and then `0x01`; it does not first read address `0x01`.

8. **Early fourth-argument update:** If result 144 is three, the fourth argument becomes 12 before the final word store and read. Call 145 receives 12.

9. **Final payload:** If result 127 is odd, the final stored word is `0x80000000`; if even, it is zero.

10. **Next first argument:** A final word-read value of `0x00C00000` produces next-iteration first argument `0x30000000`.

11. **Target reuse:** Calls 129 and 133 use the same fixed numeric target despite different argument presentation. Calls 141 and 145 likewise share a target.

12. **Progress:** If every call returns and every access succeeds, the final updates lead to another iteration, not a normal return.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and reliable executable function boundaries.
- Initial inherited values and the actual ABI.
- Symbol-dependent target addresses and initial continuation target.
- Dependency semantics, global mutation, and memory ownership.
- Additional result-register provenance.
- Signed-overflow interpretation outside the conditional machine-word model.
- Memory validity, layout, alignment, byte order, and hardware-access semantics.
- Whether the represented unconditional loop faithfully captures executable control flow.

Warnings concerning removed unreachable material and overlapping global symbols limit confidence in completeness and global layout. They do not establish missing validation, recovery, or exit behavior.

## Evidence

The contract is based solely on static inspection of the supplied representation: represented arguments and inherited state, ordered calls and accesses, explicit widths and arithmetic, constant-derived target relationships, and assignments around the unconditional back edge.

No independent dependency contracts, trusted memory layout, executable observations, or runtime results were supplied. No runtime tests have been performed.
