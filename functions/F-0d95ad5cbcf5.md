<!-- Trusted provenance: F-0d95ad5cbcf5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0d95ad5cbcf5

## Purpose

The established behavior is an ordered sequence of indirect calls and byte-sized memory writes, followed by an unconditional repeating sequence. The visible control flow has no normal return.

No application-level purpose is established. In particular, the evidence does not justify describing this function as a serializer, device driver, decoder, allocator, or protocol implementation. The narrow memory accesses and repeated calls could support several interpretations, none of which is part of this contract.

This contract describes the supplied static representation. Its control-flow recovery carries explicit reliability warnings, so correspondence with an executable implementation remains **documented_only**.

## Interface

### Explicit inputs

The represented interface has four positional inputs:

| Input | Represented width | Visible use |
|---|---:|---|
| First | 32-bit unsigned | Supplies two byte writes and the three explicit arguments of the first call. |
| Second | 32 bits, otherwise untyped | No visible direct use. |
| Third | 32 bits, otherwise untyped | No visible direct use. |
| Fourth | 32-bit unsigned | Supplies a byte-write address and an explicit argument to a later call. |

The absence of direct uses of the second and third inputs does not establish that called functions cannot observe their incoming machine state. The calling convention and register preservation rules are unavailable.

### Hidden entry state

The function also depends on two incoming machine-state values not represented as explicit parameters:

- A **state base**, used to locate a byte at offset 1.
- A **link-derived value**, used in calculating the first indirect call target.

No validity, ownership, alignment, or lifetime guarantees for these values are supplied.

### Outputs and termination

The represented return type has no value. The visible control flow nevertheless does not reach an ordinary return: after a finite prefix, it repeats indefinitely.

Observable outputs consist of:

- Explicit byte writes.
- Indirect calls and their explicit arguments.
- Any effects produced by those callees.
- Any fault or nonlocal termination caused by memory access or a callee.

There is no represented success result, error result, timeout, cancellation check, or finite loop bound.

### Width conventions

The expressions use 32-bit values and one-byte memory accesses. Byte stores retain the low eight bits of their source value. Byte loads used to form addresses are unsigned and therefore range from 0 through 255.

Unsigned right shifts are logical. Several left shifts operate on values represented as signed integers. Their executable semantics are not independently established by this representation. Where this contract gives modular arithmetic identities, those identities are conditional on ordinary 32-bit bit-vector execution; they are not claims about portable language-level signed arithmetic.

## Behavior

### Memory destinations

Three destination constructions govern the direct writes.

**State-selected destination.** Read one unsigned byte at state base plus 1. Add decimal 17 to that byte to obtain the write address. The resulting address lies between decimal 17 and 272 inclusive.

Each occurrence performs a fresh selector-byte read. A previous selector value is not reused across intervening calls.

**Fourth-input destination.** Logically shift the fourth input right by 7, then add decimal 28. The resulting address lies between decimal 28 and 33,554,459 inclusive. Store one byte there.

**Derived-selector destination.** Take the result retained from dependency F-D12, shift it left by 2, multiply the resulting represented integer by 1,024, then add 1. Read one unsigned byte at that address. Add decimal 17 to the byte and store at the resulting address.

Under 32-bit modular arithmetic, the derived selector-read address equals the low 20 bits of the F-D12 result shifted left by 12, plus 1. The visible expression involves signed intermediates, so this simplification is conditional rather than an unconditional language-level guarantee.

The selector-read address and selected write address are distinct concepts. A high or wrapped selector-read address can still select a write address between 17 and 272.

### Finite prefix

The following order is established by the supplied representation. Dependency IDs are defined in the Dependencies section.

1. Read the state selector and write the low byte of the first input to its selected destination.
2. Invoke F-D01 with three explicit arguments, each equal to the first input.
3. Invoke F-D02 and retain its 32-bit result.
4. Invoke F-D03 with that result.
5. Invoke F-D04, F-D05, and F-D06, in that order.
6. Invoke F-D07 and retain its result.
7. Read the state selector again. Write the low byte of the first input logically shifted right by 22. This byte contains original input bits 22 through 29; bits 30 and 31 do not survive the byte store.
8. Invoke F-D08 with the F-D07 result shifted left by 2.
9. Invoke F-D09, F-D10, and F-D11, in that order.
10. Invoke F-D12 and retain its 32-bit result.
11. Write the low byte of the F-D12 result shifted left by 2 to the fourth-input destination.
12. Read the state selector again and write zero to its selected destination.
13. Invoke F-D13 with the fourth input logically shifted right by 7.
14. Invoke F-D14.
15. Invoke F-D15 with the F-D12 result masked to its low 30 bits and then logically shifted right by 6.
16. Invoke F-D16, F-D17, and F-D18, in that order.
17. Invoke F-D19 and retain its result.
18. Read the state selector again and write zero to its selected destination.
19. Invoke F-D20 with the F-D19 result shifted left by 2.
20. Invoke F-D21, F-D22, and F-D23, in that order.
21. Invoke F-D24 and retain its 32-bit result.
22. Mask the F-D24 result to its low 30 bits, then logically shift right by 25. Retain this five-bit quantity.
23. Read the derived selector using the retained F-D12 result, and write zero to the selected destination.
24. Invoke F-D25 with the retained five-bit quantity shifted left by 1.
25. Invoke F-D26 and retain its result as the first loop value.

The five-bit quantity in step 22 ranges from 0 through 31. Consequently, the explicit F-D25 argument ranges over the even values from 0 through 62.

The F-D15 argument ranges from 0 through 16,777,215. Masking before shifting matters: it excludes original bits 30 and 31 rather than allowing them to contribute.

### Repeating sequence

The repeating sequence has two retained values:

- A **loop value**, initially the result of F-D26.
- A **shift-state value**, initially the five-bit quantity derived from F-D24.

Every iteration performs the following ordered actions:

1. Invoke a loop-entry dependency with the loop value as its first explicit argument and the shift-state value logically shifted right by 8 as its second.
2. Invoke F-D28, F-D29, and F-D30, in that order.
3. Invoke F-D31 and retain its result.
4. Read the derived selector using the retained F-D12 result, then write zero to the selected destination.
5. Invoke F-D32 with the F-D31 result shifted left by 2.
6. Invoke F-D33, F-D34, and F-D35, in that order.
7. Invoke F-D36 and retain its result.
8. Set the next loop value to the F-D36 result shifted left by 2.
9. Set the next shift-state value to the F-D36 result shifted left by 26.
10. Repeat without a visible condition.

The loop-entry dependency changes after the first iteration:

- First iteration: F-D27.
- Every subsequent iteration: F-D37.

This distinction is observable and must not be replaced by a single assumed invariant call target.

On the first iteration, the second loop-entry argument is necessarily zero, because the initial shift-state value is at most 31.

Under 32-bit bit-vector semantics, subsequent second arguments equal the low six bits of the preceding F-D36 result shifted left by 18. Bits above those six do not survive the left shift by 26.

The derived selector is reread during every iteration. Its read address is based on the retained F-D12 result, but its byte contents may change.

## Side effects

### Direct memory accesses

Before entering the loop, the function directly performs:

- Four reads of the byte at state base plus 1.
- Four corresponding state-selected byte writes.
- One byte write to the fourth-input destination.
- One derived-selector byte read and corresponding zero write.

Each loop iteration adds one derived-selector byte read and one selected zero write.

These counts exclude every access made by dependencies.

The function does not visibly perform direct multi-byte stores, allocation, deallocation, or bulk copying.

### Ordering and aliasing

The direct writes and calls occur in the order described above. No intervening call may be treated as effect-free.

Selectors are loaded before their corresponding stores. If a selected destination aliases the selector byte itself, the store may change which destination a later access selects. Dependencies may also modify selector memory between accesses.

The fourth-input destination may overlap a selector location or another selected destination. No disjointness precondition is established. Consequently, the contract does not guarantee that all state-selected stores target the same address, or that an earlier stored value remains observable after later operations.

The final visible write in each loop iteration is a zero byte, but later dependencies may overwrite that location.

### Ownership and errors

No ownership transfer is established for any input, hidden-state value, or memory region. No dependency is known to allocate, retain, release, or preserve memory.

Successful continued execution requires that each actual read, write, and call target be valid in its execution environment. This is a conditional execution requirement, not evidence that the function validates those accesses.

No local recovery behavior is shown. An invalid access, invalid target, callee fault, or callee non-return can prevent the remaining sequence from occurring.

## Dependencies

The identifiers below are local opaque dependency labels. They distinguish represented call sites or target constructions; they do not assert resolved implementation identities.

| IDs | Explicit arguments and result use |
|---|---|
| F-D01 | Three copies of the first input; result not explicitly consumed. Target depends on incoming link-derived state. |
| F-D02, F-D03 | F-D02 has no displayed explicit arguments; its result is supplied to F-D03. |
| F-D04–F-D06 | No displayed explicit arguments; results not explicitly consumed. |
| F-D07, F-D08 | F-D07 result, shifted left by 2, is supplied to F-D08. |
| F-D09–F-D11 | No displayed explicit arguments; results not explicitly consumed. |
| F-D12 | Produces the retained value used for a byte store, F-D15’s argument, and derived-selector addressing. |
| F-D13 | Receives the fourth input shifted right by 7. |
| F-D14 | No displayed explicit arguments; result not explicitly consumed. |
| F-D15 | Receives the masked and shifted F-D12 result. |
| F-D16–F-D18 | No displayed explicit arguments; results not explicitly consumed. |
| F-D19, F-D20 | F-D19 result, shifted left by 2, is supplied to F-D20. |
| F-D21–F-D23 | No displayed explicit arguments; results not explicitly consumed. |
| F-D24, F-D25 | A masked five-bit portion of F-D24’s result, doubled, is supplied to F-D25. |
| F-D26 | Produces the initial loop value. |
| F-D27 | First-iteration loop entry; receives two explicit arguments. |
| F-D28–F-D30 | No displayed explicit arguments; results not explicitly consumed. |
| F-D31, F-D32 | F-D31 result, shifted left by 2, is supplied to F-D32. |
| F-D33–F-D35 | No displayed explicit arguments; results not explicitly consumed. |
| F-D36 | Produces both next-iteration retained values. |
| F-D37 | Subsequent-iteration loop entry; receives two explicit arguments. |

“No displayed explicit arguments” does not establish an argument-free machine interface. Likewise, an unconsumed result does not imply an effect-free call.

All calls are indirect in the supplied representation. Except for the first target’s dependence on incoming state, their target constructions use embedded address-like values. Their validity, calling conventions, preservation behavior, and implementation identities are **documented_only**.

## Acceptance examples

These are conditional contract examples, not executed tests. They assume valid memory, returning dependencies, and the stated arithmetic interpretation.

1. **First-input byte extraction.** With the first input equal to hexadecimal `FFFFFFFF`, the first state-selected byte is hexadecimal `FF`. The later first-input-derived state-selected byte is also hexadecimal `FF`, because the shifted value is narrowed to one byte.

2. **Selector changes across calls.** If the first state-selector read returns 3, the first write targets address 20. If intervening dependencies change that selector to 9, the next state-selected write targets address 26. Reusing the first destination would violate the represented behavior.

3. **Fourth-input address formation.** For a fourth input of 128, the fourth-input destination is address 29, and F-D13 receives 1. For a fourth input of 127, the destination is address 28, and F-D13 receives 0.

4. **F-D12 masking and narrowing.** If F-D12 returns hexadecimal `FFFFFFFF`, the fourth-input byte write stores hexadecimal `FC`, and F-D15 receives hexadecimal `00FFFFFF`.

5. **Initial loop argument.** If F-D24 returns hexadecimal `3E000000`, the retained five-bit quantity is 31 and F-D25 receives 62. F-D27 still receives zero as its second argument.

6. **Later loop argument.** Under 32-bit bit-vector semantics, if F-D36 returns 63, the next loop value is 252 and F-D37’s second argument is hexadecimal `00FC0000`. If it returns 64, that second argument is zero.

7. **Fresh derived selector reads.** If consecutive loop iterations read selector bytes 2 and 5 from the same derived selector-read address, their direct zero stores target addresses 19 and 22 respectively.

8. **Non-returning dependency.** If F-D19 does not return, the subsequent state-selected zero write and all loop activity are absent. The function supplies no visible recovery path.

## Uncertainties

The following remain **documented_only**:

- The real entry calling convention and interpretation of hidden state.
- Executable correspondence of the recovered control flow.
- Validity and meaning of the indirect call targets.
- Signed-shift and overflowing signed-multiplication semantics.
- Whether low numeric addresses represent ordinary memory, mapped hardware, or invalid locations.
- Dependency effects on memory, globals, register state, ownership, and control flow.
- Whether the actual executable has exits or alternate paths absent from the supplied representation.
- Behavior under concurrent mutation, interrupts, or asynchronous memory changes.

The infinite loop is established within the visible representation. It does not prove that an actual execution runs forever: faults and non-returning callees can interrupt it.

## Evidence

The static evidence consists solely of the supplied function representation and its warnings about invalid instruction data and removed unreachable regions.

That representation establishes explicit byte accesses, argument transformations, call ordering, retained values, the changing loop-entry target, and an unconditional back edge.

It does not establish an application-level purpose, a complete machine-level calling convention, dependency contracts, or reliable executable behavior across the warned-about regions.

No runtime tests have been performed. No acceptance example above is a claim of observed execution.
