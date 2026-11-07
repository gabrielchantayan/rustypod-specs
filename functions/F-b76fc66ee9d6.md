<!-- Trusted provenance: F-b76fc66ee9d6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b76fc66ee9d6

## Purpose

F-b76fc66ee9d6 performs a byte write, invokes two initialization-stage dependencies, and then repeatedly executes a fixed sequence of dependency calls and another byte write. The visible loop has no termination condition or normal return.

The supplied representation establishes an ordering and data-transformation contract, but not a domain-level purpose. In particular, it does not establish whether the byte writes access ordinary memory, memory-mapped hardware, or another address space.

**Evidence qualification:** the supplied representation reports invalid instruction data and removed unreachable regions. This document describes the visible recovered path, not a verified reconstruction of every possible machine-level behavior. No runtime tests have been performed.

## Interface

### Entry state

The recovered interface has no declared parameters and no declared return value. Nevertheless, its behavior depends on incoming machine state:

| Logical input | Role |
|---|---|
| **Write seed** | Unsigned word supplying the first two explicitly written byte values. |
| **Selector base** | Address-bearing value used to locate a one-byte destination selector at offset 1. |
| **Shift seed** | Unsigned word supplying arguments to initialization and the first loop iteration. |
| **Dispatch base** | Incoming link-related value used to derive the first indirect call target. |

These names describe roles, not source identifiers or a conventional calling convention. How a caller supplies these values is unknown.

### Widths and interpretation

The recovered operations use byte accesses and integer shifts consistent with a 32-bit word model. Accordingly, the arithmetic contracts below use:

- An unsigned 8-bit selector.
- An 8-bit destination store.
- Unsigned 32-bit write and shift seeds.
- 32-bit result bit patterns for dependency results that feed subsequent shifts.
- Logical right shifts for the unsigned seeds.
- Low-32-bit retention for left-shift results when describing machine-word behavior.

The exact target ABI and executable instruction semantics are not independently available. In particular, some dependency results are represented as signed integers before being left-shifted. Applying strict source-language signed-shift rules would not establish a reliable contract for all possible results. Low-32-bit left-shift behavior is therefore a **conditional machine-word interpretation**, not an independently verified runtime fact.

### Memory layout

Each explicit byte write resolves its destination through this layout:

| Location | Access | Meaning |
|---|---|---|
| Selector base plus 1 | Read one unsigned byte | Destination selector. |
| Numeric address equal to selector plus 17 | Write one byte | Destination of the explicit store. |

The destination is **not** selector base plus selector plus 17. Only the selector read is relative to the selector base.

Because the selector is zero-extended before adding 17, the destination address ranges from 17 through 272 inclusive. No larger pointer or multi-byte address is loaded from the selector location.

### Preconditions and output

To proceed along the visible path:

- The selector byte must be readable whenever it is accessed.
- Each selected destination must permit the byte write.
- The resolved indirect call targets must be callable under the actual machine ABI.
- Dependencies must return normally for subsequent operations to occur.

There are no visible validation checks for these conditions.

There is no visible normal output value. Observable outputs consist of memory accesses, calls and their explicit arguments, dependency effects, and continued execution.

## Behavior

### Dependency naming

The dependency IDs below are document-local opaque labels. They distinguish visible call sites or target expressions; they do not assert independently verified function identities. Distinct labels do not prove distinct implementations.

### Entry-stage actions

The following order is established by the recovered path:

1. Read the selector byte at selector base plus 1.
2. Write the low eight bits of the write seed to numeric address selector plus 17.
3. Invoke **F-unknown-01**, whose target is derived from the dispatch base plus displacement `0xF74`. Its explicit argument is the shift seed shifted left by one bit.
4. Invoke **F-unknown-02**, a fixed-target dependency with no explicit arguments. Retain its returned word as the first loop’s carried value.
5. Enter the loop.

Under the 32-bit interpretation, the argument to F-unknown-01 is twice the initial shift seed modulo \(2^{32}\). Its return value is not explicitly consumed.

The first byte write occurs before either dependency call. No explicit selector read occurs between these two calls.

### First loop iteration

The first iteration performs these actions in order:

1. Invoke **F-unknown-03** with two explicit arguments:
   - The result retained from F-unknown-02.
   - The initial shift seed logically shifted right by eight bits.
2. Invoke **F-unknown-04**, with no explicit arguments.
3. Invoke **F-unknown-05**, with no explicit arguments.
4. Invoke **F-unknown-06**, with no explicit arguments.
5. Invoke **F-unknown-07**, with no explicit arguments, and retain its returned word.
6. Read the selector byte again from selector base plus 1.
7. Write the low eight bits of the write seed logically shifted right by 22 bits to numeric address the newly read selector plus 17.
8. Invoke **F-unknown-08** with the retained result of F-unknown-07 shifted left by two bits.
9. Invoke **F-unknown-09**, with no explicit arguments.
10. Invoke **F-unknown-10**, with no explicit arguments.
11. Invoke **F-unknown-11**, with no explicit arguments.
12. Invoke **F-unknown-12**, with no explicit arguments, and retain its returned word.
13. Prepare the next iteration:
    - The carried value becomes the result of F-unknown-12 shifted left by two bits.
    - The shift seed becomes that same result shifted left by 26 bits.
    - The write seed becomes zero.
14. Continue directly to the next iteration.

The first loop write extracts bits 22 through 29 of the original write seed. Bits 30 and 31 do not contribute to that stored byte.

The second selector read is independent of the entry-stage selector read. Dependencies executed between them may change the selector or related memory. The two writes therefore need not target the same address.

### Subsequent iterations

Every later iteration follows the same sequence, with these differences:

- The first dependency is **F-unknown-13**, not F-unknown-03.
- Its first argument is the carried value prepared from the preceding F-unknown-12 result.
- Its second argument is the current shift seed logically shifted right by eight bits.
- The explicit byte written during the iteration is zero, because the write seed was reset to zero at the preceding iteration’s end.

The remaining dependencies retain the order F-unknown-04 through F-unknown-12, with the selector read and byte write between F-unknown-07 and F-unknown-08.

Under the conditional 32-bit interpretation, if the preceding F-unknown-12 result has bit pattern \(t\), F-unknown-13 receives:

- First argument: \(4t \bmod 2^{32}\).
- Second argument: \((t \mathbin{\&} 63) \times 2^{18}\).

The second argument depends only on the low six bits of the preceding result. It is not equivalent to shifting the full result left by 18 without first discarding higher bits.

### Continuation and interruption

No visible condition selects a normal exit. If all accesses succeed and all dependencies return normally, execution repeats indefinitely.

That does not establish that the real execution cannot stop. A dependency may fail to return, a memory access may fault, or external control may interrupt execution. The supplied material defines no recovery behavior for those cases.

## Side effects

### Explicit memory effects

The function visibly performs:

- One selector read and one byte write before initialization calls.
- One further selector read and one byte write per loop iteration.

No explicit allocation, deallocation, bulk copy, wider store, or ownership transfer is shown.

Only eight bits are explicitly stored at each selected destination. The representation does not establish the access ordering guarantees of the underlying hardware, nor whether the accesses have device-specific effects.

### Aliasing

The selector location and a selected destination are not established to be disjoint.

If an explicit write aliases the selector byte, that write can change the selector used by a later iteration. Independently, dependencies may modify the selector. An implementation conforming to the recovered ordering must not replace the repeated selector reads with a permanently cached value.

### Dependency effects and retained state

Dependency side effects are unknown. A return value that is not consumed by the visible expression sequence does not make the corresponding call removable.

The recovered path retains the logical inputs and intermediate results where described. This does not independently establish a register-preservation convention or prove that callees preserve globals, addressable memory, or external state.

No ownership contract can be inferred merely from the absence of an explicit allocation or release. Dependencies may have their own resource effects, but those effects are not documented here.

### Errors

The visible path contains no:

- Argument validation.
- Destination-range validation beyond the arithmetic implied by the one-byte selector.
- Null-target check.
- Error-code branch.
- Retry policy.
- Cleanup or exception handler.

It is not established how execution behaves after an invalid access or invalid call target. Such outcomes must not be represented as a defined success, failure return, or recovery sequence.

## Dependencies

“No explicit arguments” means the recovered call expression supplies none; it does not establish that the callee observes no ambient machine state.

| Opaque ID | Position and explicit inputs | Explicitly consumed result |
|---|---|---|
| **F-unknown-01** | Entry stage; shift seed shifted left by 1; target derived from dispatch base | None |
| **F-unknown-02** | Entry stage; no explicit arguments | First iteration’s carried value |
| **F-unknown-03** | First iteration only; carried value and initial shift seed shifted right by 8 | None |
| **F-unknown-04** | Each iteration; no explicit arguments | None |
| **F-unknown-05** | Each iteration; no explicit arguments | None |
| **F-unknown-06** | Each iteration; no explicit arguments | None |
| **F-unknown-07** | Each iteration; no explicit arguments | Value shifted left by 2 for F-unknown-08 |
| **F-unknown-08** | Each iteration, after the byte write; shifted F-unknown-07 result | None |
| **F-unknown-09** | Each iteration; no explicit arguments | None |
| **F-unknown-10** | Each iteration; no explicit arguments | None |
| **F-unknown-11** | Each iteration; no explicit arguments | None |
| **F-unknown-12** | End of each iteration; no explicit arguments | Source of both next-iteration word transformations |
| **F-unknown-13** | Every iteration after the first; transformed prior result and current shift seed shifted right by 8 | None |

Dependency implementations, ownership rules, external effects, failure modes, and preservation guarantees are **documented_only** unknowns.

## Acceptance examples

These are conditional acceptance criteria derived from the representation, not executed tests.

### Byte selection and truncation

Given a readable selector of `0x20`, write seed `0xABCDEF12`, and dependencies that return normally:

- The entry-stage destination is numeric address `0x31`.
- The entry-stage stored byte is `0x12`.
- If the selector remains `0x20`, the first loop write also targets `0x31`.
- That first loop write stores `0xAF`, from bits 22 through 29 of the write seed.
- Each later loop write stores `0x00`, regardless of the original seed.

Dependency effects may interleave other writes; this example specifies only the function’s explicit stores.

### Selector changes between writes

Given an entry selector of `0x02`, the initial destination is `0x13`.

If a dependency changes the selector to `0xFF` before the first loop’s selector read, that loop write targets `0x110`. Reusing the initial destination would not satisfy the visible contract.

### Initialization argument width

Under the 32-bit interpretation, an initial shift seed of `0x80000001` produces:

- F-unknown-01 argument `0x00000002`.
- F-unknown-03 second argument `0x00800000`.

These arise from different shifts of the original seed. The first call’s argument does not replace that seed.

### Loop-carried truncation

If F-unknown-12 returns bit pattern `0x00000041`, the next F-unknown-13 call receives:

- First argument `0x00000104`.
- Second argument `0x00040000`.

Only the result’s low six bits contribute to the second argument.

If it returns `0x40000000`, both next-iteration arguments are zero under the low-32-bit shift interpretation.

### Required call and write order

If all dependencies return normally, the first loop’s byte write occurs after F-unknown-07 and before F-unknown-08. It cannot be moved before the preceding calls without changing the contract.

The first iteration invokes F-unknown-03; the second invokes F-unknown-13 at the corresponding position. Treating those targets as interchangeable requires evidence not supplied here.

### Interrupted initialization

If F-unknown-01 does not return normally, the entry-stage byte write has already occurred. The visible path does not reach F-unknown-02 or any loop action in that execution.

## Uncertainties

- **documented_only — Reconstruction completeness:** invalid instruction data and removed regions prevent treating the recovered path as a complete executable specification.
- **documented_only — Calling convention:** the origin and ABI placement of incoming state are unknown.
- **documented_only — Exact word semantics:** the 32-bit model is consistent with the recovered operations; signed-result left shifts require machine-level confirmation.
- **documented_only — Address arithmetic:** pointer width, address-space mapping, and exceptional address behavior are not independently established.
- **documented_only — Dependency identity:** document-local labels distinguish visible target expressions, not proven unique callees.
- **documented_only — Dependency behavior:** memory effects, global changes, resource ownership, and nonlocal control transfers are unknown.
- **Inferred intent:** repeated low-address byte writes interleaved with calls could serve a device-facing protocol. No domain-specific protocol is established, so none is required by this contract.

## Evidence

The supplied static representation supports the following facts:

- Incoming machine state influences execution despite the parameterless recovered interface.
- Each explicit store uses a fresh unsigned-byte selector read and adds 17 to form its numeric destination.
- The first stored byte is the write seed’s low byte; the first loop store uses bits 22 through 29; later loop stores are zero.
- Two calls precede the loop.
- The first loop-entry target differs from subsequent loop-entry targets.
- Each iteration contains ten dependency calls and one explicit byte write.
- One dependency result feeds a same-iteration left shift; another feeds two next-iteration shifts.
- The visible loop has no normal exit.

The representation also explicitly reports recovery problems. No runtime observations, dependency bodies, ABI documentation, or independent instruction-level verification were supplied. Acceptance examples therefore remain static, conditional criteria rather than test results.
