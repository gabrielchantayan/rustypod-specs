<!-- Trusted provenance: F-8c5d4c47b2fd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8c5d4c47b2fd

## Purpose

F-8c5d4c47b2fd exposes a long sequence of indirect calls, byte writes, word writes, and a small number of explicit memory reads. Its application-level purpose is not established.

The supplied representation ends at an invalid-instruction boundary rather than an ordinary return. It also depends on incoming machine state that is absent from its two-argument interface. Consequently, this contract describes the visible execution prefix and its ordering, not a verified, normally terminating service.

The calculations resemble manipulation of packed values and address-like indices. That resemblance does not establish an object model, allocation protocol, device interface, or ownership relationship.

## Interface

### Explicit inputs

| Input | Represented width | Visible use |
|---|---:|---|
| First argument | 32 bits | No explicit use in the visible body |
| Second argument | 32 bits | Written in full once and narrowed to its low byte for two other writes |

The first argument is not proven irrelevant to the complete execution: the indirect-call convention and machine state passed to callees are unknown.

### Additional incoming state

Execution also consumes:

- An incoming base used for the initial word write at offset `0x14`.
- An incoming base used for a byte write at offset `1`.
- An incoming 32-bit unsigned value used for the initial write, logical right shifts, and a later byte value.
- An incoming control/address value used to construct the first indirect-call target.

These are requirements of the visible representation, not additional parameters established by a recovered calling convention.

### Output and termination

No ordinary return value is established. The visible declaration has no return value, and the visible path ends at an invalid-instruction boundary after its final indirect call.

Values returned by dependencies are used as 32-bit quantities or as two 32-bit halves of a represented 64-bit result. The actual ABI behind those representations is unknown.

### Width and arithmetic conventions

This contract uses:

- **Byte:** 8 bits.
- **Word:** 32 bits.
- **Low byte:** the least significant eight bits.
- **Low/high halves:** the least/most significant 32 bits of a represented 64-bit result.
- **Logical right shift:** zero-filling right shift on an unsigned quantity.

Explicit unsigned left shifts and unsigned arithmetic retain their low 32 bits. Some other address calculations and shifts are represented with signed intermediates. Their intended machine behavior appears to be finite-width arithmetic, but overflow and negative-shift-operand behavior cannot be established from this representation alone. Examples involving those operations are conditional on valid operands or a separately established machine-arithmetic model.

No byte order, alignment policy, address-space mapping, or fault model is established.

## Behavior

### Notation and dependency order

The descriptive names below identify values for this contract only; they do not assert source-level types or object identities.

Dependency IDs of the form **F-Q01-1** are opaque labels assigned to individual visible call occurrences. Different labels do not prove different target functions. Calls described as taking “no explicit arguments” may still receive or inspect unspecified machine state.

Each numbered phase follows the preceding phase if execution continues. Within a phase, operations occur in the stated order. Calls can introduce additional reads, writes, faults, or termination.

### 1. Initial state write and first result

1. Write the incoming unsigned value as a word at the initial base plus `0x14`.
2. Retain its logical right shift by four for later use.
3. Invoke F-Q01-1 with the incoming value logically shifted right by two.
4. Invoke F-Q01-2 through F-Q01-5, in order, with no explicit arguments.

The word result of F-Q01-5 is the **first index**.

Write the incoming value logically shifted right by two at the address formed from four times the first index plus `0x14`.

### 2. First scaled index and packed result

Invoke F-Q02-1 with four times the first index, followed by F-Q02-2 through F-Q02-5 with no explicit arguments. The final word result is the **first packed result**.

Form the **eightfold index** as eight times the first index. Then:

1. Write its low byte at the second incoming base plus `1`.
2. Write the first packed result shifted left by two at the eightfold index plus `0x34`.
3. Invoke F-Q03-1 with that shifted packed result.
4. Invoke F-Q03-2 with no explicit arguments, retaining its word result as a **forwarded word**.

Define the **masked base** as the first packed result AND `0x3fffffff`.

### 3. First two-word result

Invoke F-Q04-1 with the forwarded word, then F-Q04-2 through F-Q04-5 with no explicit arguments. Retain the final represented 64-bit result as the **first pair**.

Then:

1. Write the low byte of the eightfold index at masked base plus `1`.
2. Write the high half of the first pair at eightfold index plus `0x34`, replacing the earlier word at that address if it still denotes the same storage.
3. Invoke F-Q05-1 with the low half of the first pair shifted left by two.
4. Invoke F-Q05-2 through F-Q05-5 with no explicit arguments.

Retain the final word result as the **second index**.

### 4. Sixteenfold-index writes

Form the **sixteenfold index** as sixteen times the first index.

Perform these writes in order:

1. At address `((masked base logical-right-shift 20) + 0x11)`, write the low byte of the sixteenfold index.
2. At sixteenfold index plus `0x0d`, write the low byte of the second explicit argument.

Invoke F-Q06-1 with two identical explicit arguments, each the second index shifted left by two. Invoke F-Q06-2 with no explicit arguments, retaining its word result as the **third result**.

Invoke F-Q07-1 with the third result and its logical right shift by three. Invoke F-Q07-2 through F-Q07-5 with no explicit arguments. Retain the final word result as the **third index**.

Then write:

1. The low byte of the sixteenfold index at absolute address `0x11`.
2. The low byte of the second argument again at sixteenfold index plus `0x0d`.

These are separate writes, with intervening dependency calls; they are not safely collapsible.

### 5. Masked-base updates and another packed result

Invoke F-Q08-1 with the third index shifted left by two, followed by F-Q08-2 through F-Q08-5 with no explicit arguments. Retain the final word as the **second packed result**.

Write:

1. The complete second argument at masked base plus `0x14`.
2. The low byte of masked base shifted left by one at masked base plus `0x0d`.

From the second packed result derive:

- **Record base:** thirty-two times that result.
- **Scaled payload:** that result shifted left by seven.
- **Small index:** that result AND `0x07ffffff`, then logically shifted right by seventeen. Its range is `0` through `1023`.

Invoke F-Q09-1 through F-Q09-3 with no explicit arguments. Retain the final word result as the **fourth index**.

Write the scaled payload at record base plus `0x14`, then write the low byte of the fourth index shifted left by two at record base plus `0x0d`.

### 6. Small-index and byte-selected writes

Invoke F-Q10-1 with the fourth index shifted left by two. Invoke F-Q10-2 through F-Q10-5 with no explicit arguments.

Then:

1. At small index plus `7`, write the low byte of twice the incoming unsigned value’s logical right shift by four.
2. Read a byte at record base plus `1`.
3. Zero-extend that byte, add `0x11`, and write the low byte of the small index at that address.

The first destination is in `0x07`–`0x406`. The second is in `0x11`–`0x110`.

Invoke F-Q11-1 with three identical arguments, each the fourth index shifted left by twenty-one. Invoke F-Q11-2 with no explicit arguments, retaining its word result as the **retained result**.

### 7. Clear a selected byte and retain its selector

Invoke F-Q12-1 with the retained result, followed by F-Q12-2 through F-Q12-5 with no explicit arguments. Retain the final word as the **fifth index**.

Read a byte at record base plus `1`. This is a fresh read, not reuse of the earlier selector. Retain it as the **saved selector** and write zero at saved selector plus `0x11`.

Invoke F-Q13-1 with the fifth index shifted left by two, followed by F-Q13-2 through F-Q13-5 with no explicit arguments.

Define:

- **Doubled selector:** twice the saved selector, ranging from `0` through `510`.
- **Saved byte:** the low byte of the retained result logically shifted right by three.

Write the saved byte at record base plus `0x0d`, then write the saved selector at absolute address `0x0d`.

### 8. Auxiliary result and fixed-byte write

Invoke F-Q14-1 with doubled selector, `0x98`, and `0x260`, in that order. Invoke F-Q14-2 with no explicit arguments.

An auxiliary result represented in a secondary machine-result location is then logically shifted right by three and passed to F-Q15-1. Its exact producing instruction and ABI relationship are not established. Invoke F-Q15-2 through F-Q15-5 with no explicit arguments; retain the final word as the **sixth index**.

Invoke F-Q16-1 with the sixth index shifted left by two, retaining its word result as the **seventh index**.

Write:

1. The saved byte at doubled selector plus `0x0d`.
2. Byte `0xc0` at absolute address `0x0d`.

### 9. Linked-word reads and writes

Invoke F-Q17-1 with the seventh index shifted left by two, followed by F-Q17-2 through F-Q17-5 with no explicit arguments. Retain the final word as the **eighth index**.

Then, in order:

1. Write the eighth index shifted left by two at doubled selector plus `0x24`.
2. Read a word at doubled selector plus `0x74`; retain it as the **linked base**.
3. Form the **linked payload** as the eighth index shifted left by five.

Invoke F-Q18-1 with:

1. Linked base.
2. Linked base shifted left by one.
3. Linked payload.
4. Linked base AND `0x7fffffff`, logically shifted right by two.

Invoke F-Q18-2 through F-Q18-5 with no explicit arguments.

Write the linked payload at linked base plus `0x24`. Then read a word at linked base plus `0x74` and pass that freshly read word to F-Q19-1. Invoke F-Q19-2 through F-Q19-5 with no explicit arguments. Retain the final represented 64-bit result as the **second pair**.

### 10. Paired results and derived-address word writes

Form the **pair payload** as the low half of the second pair shifted left by two.

Write the retained result logically shifted right by three at four times the high half of the second pair plus `0x14`.

Invoke F-Q20-1 with the pair payload three times, followed by the high half of the second pair logically shifted right by three.

Invoke F-Q21-1 with no explicit arguments and retain its represented 64-bit result as the **third pair**.

Invoke F-Q22-1 with these four arguments:

1. Low half of the third pair.
2. High half of the third pair.
3. Low half of the third pair again.
4. High half logically shifted right by three.

Invoke F-Q22-2 through F-Q22-5 with no explicit arguments. Retain the final word as the **ninth index**.

Write the pair payload at four times the ninth index plus `0x14`.

### 11. Repeated selector reads and terminal clears

Invoke F-Q23-1 with four times the ninth index, followed by F-Q23-2 through F-Q23-5 with no explicit arguments. Retain the final word as the **final packed result**. Form the **final scaled value** by shifting it left by two.

Read a byte at linked base plus `1`, then write the low byte of the final scaled value at that byte’s zero-extended value plus `0x11`.

Invoke F-Q24-1 through F-Q24-3 with no explicit arguments, retaining the final word as the **tenth index**.

Read a byte again at linked base plus `1`. At the selected address plus `0x11`, write the low byte of the final packed result AND `0x3fffffff`, logically shifted right by twenty.

Invoke F-Q25-1 with the tenth index shifted left by two, followed by F-Q25-2 through F-Q25-5 with no explicit arguments. Retain the final word as the **eleventh index**.

Read a byte at 1024 times the final scaled value plus `1`. Write zero at that byte’s zero-extended value plus `0x11`.

Invoke F-Q26-1 with:

1. Final scaled value shifted left by three.
2. The same value again.
3. Final scaled value.
4. Eleventh index shifted left by two.

Invoke F-Q27-1 through F-Q27-4 with no explicit arguments. Retain the final word as the **twelfth index**.

Read a byte again at 1024 times the final scaled value plus `1`, then write zero at its selected address plus `0x11`.

Invoke F-Q28-1 with the twelfth index shifted left by two, followed by F-Q28-2 through F-Q28-5 with no explicit arguments. The visible path then reaches the invalid-instruction boundary.

## Side effects

All listed writes are externally relevant unless the actual memory model proves otherwise. They must not be treated as private object updates merely because they use recurring offsets.

Particular ordering constraints include:

- Absolute address `0x0d` receives the saved selector before later receiving `0xc0`.
- Absolute address `0x11` receives a byte independently of the dynamically selected writes.
- Several selector bytes are reread after calls. The selected destination may therefore change.
- Dependencies occur between writes to nominally identical addresses. Intermediate values may be observed.
- Different address expressions may alias. The visible sequence, rather than an assumed structure layout, determines the resulting memory state.

No allocation, release, ownership transfer, synchronization, rollback, or cleanup operation is explicitly established. Dependencies may perform such operations; their absence from this body is not proof of their absence during execution.

No validation, bounds check, null check, or recoverable error result is visible. A bad address, unsuitable alignment, invalid call target, dependency failure, or invalid instruction may prevent later effects.

## Dependencies

F-Q01-1 through F-Q28-5, as individually enumerated in the behavior section, label only the call occurrences actually specified there; numbering does not imply that every phase contains five calls.

The first target depends on incoming control state plus a fixed displacement. Later targets are constructed from embedded address-like values and fixed displacements. The supplied dependency map does not establish identities or behavioral contracts for these targets.

For every dependency, the following remain unknown:

- Actual signature and calling convention.
- Additional implicit inputs and outputs.
- Global-state changes.
- Memory ownership and lifetime effects.
- Whether it returns, faults, or transfers control elsewhere.
- Whether separate call occurrences reach the same implementation.

No dependency is assumed pure, memory-preserving, allocation-free, or ownership-preserving.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Second-argument narrowing.** If the second argument is `0x123456ab` and execution reaches the relevant writes, the two sixteenfold-index-plus-`0x0d` writes each carry byte `0xab`. The masked-base-plus-`0x14` write carries the complete word `0x123456ab`.

2. **Maximum small index.** If the second packed result has all low twenty-seven bits set, the small index is `1023`. The phase-6 direct byte destination is therefore `0x406`; its byte-selected destination still depends on a separate memory read.

3. **Saved versus fresh selectors.** If the phase-7 selector read returns `0xff`, its clear targets `0x110` and the doubled selector is `510`. Later dependency changes to the selector’s original storage do not change that saved value. Other explicitly repeated selector reads must still observe their own current memory values.

4. **Unsigned truncation.** If the first packed result is `0xc0000001`, its unsigned left shift by two is `0x00000004`, while the masked base is `1`. These are distinct derived quantities.

5. **Repeated clears are observable operations.** If the two final selector reads return `2` and `3`, the clears target `0x13` and `0x14`, respectively. An implementation that retains the first selector for both clears does not match the visible sequence.

6. **Interrupted execution.** If a dependency does not return, later listed writes are not guaranteed. No restoration of earlier writes is established.

## Uncertainties

- **documented_only — Entry validity:** The body consumes unexplained incoming state and is accompanied by an invalid-control-flow warning. A conventional callable entry is not established.
- **documented_only — Termination:** The final invalid-instruction marker establishes a representation boundary, not a proven hardware exception or intentional termination mechanism.
- **documented_only — Arithmetic:** Signed intermediate overflow and actual address-width semantics require independent instruction-level evidence.
- **documented_only — Memory layout:** Offsets such as `1`, `0x0d`, `0x14`, `0x24`, `0x34`, and `0x74` do not establish structures or field types.
- **documented_only — Auxiliary result:** The secondary-result value used in phase 8 lacks an established producer/consumer ABI.
- **documented_only — Dependencies and ownership:** Target identities, side effects, object lifetimes, and release responsibilities are unresolved.
- **Inferred intent:** Packed-field extraction and index scaling are plausible interpretations only; no application-level claim relies on them.

## Evidence

The established static evidence is the supplied function representation: explicit memory accesses, represented operand widths, masks, shifts, call ordering, result consumption, and the terminal invalid-instruction boundary.

Comments and recovered type annotations are not independent proof of instruction semantics, calling conventions, or application intent. This contract therefore distinguishes the visible represented sequence from unresolved execution details.

No implementation access, instruction-level validation, runtime tests, or dependency execution has been performed. No snapshot hash or independent provenance is asserted.
