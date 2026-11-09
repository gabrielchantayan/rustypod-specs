<!-- Trusted provenance: F-b043bf3d56cd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b043bf3d56cd

## Purpose

F-b043bf3d56cd performs a sequence of byte writes and indirect dependency calls, then enters a repeating dispatch sequence with no visible normal exit.

The established behavior is a low-level interaction protocol, not a recognizable application operation. It:

- Stores the low byte of the first input at an address derived from the fourth input.
- Clears bytes at addresses selected through other byte reads.
- Passes shifted inputs and transformed dependency results to further calls.
- Repeats a ten-call cycle, carrying information from the final result of one iteration into the first call of the next.

No reliable higher-level purpose, such as initialization, device control, decoding, or memory management, is established. Those interpretations must not be used as acceptance criteria.

This contract describes the supplied static representation. It does not establish that all represented control flow is executable or complete.

## Interface

### Explicit inputs

The represented interface has four word-sized inputs:

| Input | Represented width | Established use |
|---|---:|---|
| First | 32-bit unsigned | Supplies an initial stored byte, an unsigned right-shifted call argument, and a scaled address used for later selector reads. |
| Second | 32-bit opaque word | No explicit use appears in the supplied representation. |
| Third | 32-bit opaque word | No explicit use appears in the supplied representation. |
| Fourth | 32-bit unsigned | Supplies an unsigned right-shifted address component and the first dependency argument. |

The absence of explicit uses of the second and third inputs does not prove that their incoming machine-register contents are unobservable to dependencies. Dependency signatures and the calling convention are not established.

### Ambient inputs

Two additional incoming values are referenced without initialization inside the represented function:

- An **ambient selector base**, used to read a selector byte at an offset of one byte.
- An **ambient call-origin value**, used to derive the first indirect call target by adding `0xFEC`.

These are operational prerequisites, not documented additional public arguments. Their source, validity, lifetime, and relationship to the explicit inputs are unknown.

### Output and completion

The represented return type has no result value. More importantly, there is no visible normal return path: after the prefix sequence, execution enters an unconditional loop.

If all memory accesses complete and every dependency returns normally, the represented function continues indefinitely. Dependency failure, traps, exceptions, nonlocal transfers, or nonreturning behavior may prevent later operations; their handling is not shown.

### Width and address interpretation

The supplied operations represent 32-bit unsigned words and 8-bit memory accesses. Unsigned right shifts therefore insert zero bits, and unsigned address scaling uses 32-bit arithmetic.

Some dependency results are represented as signed 32-bit values before being left-shifted. Where this contract describes their shifted bit patterns, that description is conditional on ordinary 32-bit machine-shift semantics. The representation does not resolve signed-overflow semantics.

Pointer width, memory mapping, alignment rules for instruction targets, and the complete ABI are not independently established.

## Behavior

### 1. Initial byte store

The first observable operation writes one byte:

- Destination: the fourth input shifted right by seven bits, plus `0x1C`.
- Value: the low eight bits of the first input.

The unsigned shifted component ranges from zero through `0x01FFFFFF`. Consequently, this destination ranges from `0x1C` through `0x0200001B`; the addition does not overflow a 32-bit unsigned word.

There is no visible range check, ownership check, or null check.

### 2. First ambient-selector clear

After the initial store:

1. Read one byte at the ambient selector base plus one.
2. Interpret that byte as an unsigned value.
3. Write zero to the byte at that value plus `0x11`.

The selected destination is therefore between `0x11` and `0x110`, inclusive.

The selector read follows the initial store. If those addresses alias, the selector read observes the state after that store, subject to the actual memory system.

### 3. Initial dependency sequence

The following calls occur in order. Dependency identifiers are defined in **Dependencies**.

| Order | Dependency | Explicit argument information | Result use |
|---:|---|---|---|
| 1 | F-D01 | Fourth input shifted right by seven | No explicit result use |
| 2 | F-D02 | No explicit arguments shown | No explicit result use |
| 3 | F-D03 | First input shifted right by eight | No explicit result use |
| 4 | F-D04 | No explicit arguments shown | No explicit result use |
| 5 | F-D05 | No explicit arguments shown | No explicit result use |
| 6 | F-D06 | No explicit arguments shown | No explicit result use |
| 7 | F-D07 | No explicit arguments shown | Retained for a later left shift |

“No explicit arguments shown” is not a verified zero-argument ABI contract. Likewise, an unused result does not mean that a call has no effects.

### 4. Second ambient-selector clear and continuation

Immediately after F-D07 returns, the function repeats the ambient-selector clear:

1. Reread the byte at the ambient selector base plus one.
2. Write zero at the resulting unsigned byte value plus `0x11`.

This is a fresh read. Calls F-D01 through F-D07 may have changed the selector or affected the selected memory; preservation is not assumed.

The function then performs:

| Order | Dependency | Explicit argument information | Result use |
|---:|---|---|---|
| 8 | F-D08 | F-D07 result shifted left by two bits | No explicit result use |
| 9 | F-D09 | No explicit arguments shown | No explicit result use |
| 10 | F-D10 | No explicit arguments shown | No explicit result use |
| 11 | F-D11 | No explicit arguments shown | No explicit result use |
| 12 | F-D12 | No explicit arguments shown | Used as an unsigned 32-bit result |

From the F-D12 result, the function selects bits 25 through 29: it masks with `0x3FFFFFFF`, then shifts right by 25. The resulting field is an unsigned integer from zero through 31.

### 5. Scaled-selector clear

The function next derives a selector address from the first input:

- Multiply the first input by `0x400` using 32-bit unsigned arithmetic.
- Add one.
- Read the byte at that address.
- Write zero at the unsigned selector byte plus `0x11`.

The selector address depends only on the low 22 bits of the first input. It is always congruent to one modulo `0x400`; the multiplication may wrap, but adding one cannot overflow because the product’s low ten bits are zero.

This read occurs after F-D12 returns and before F-D13 is called.

### 6. Loop initialization

The function calls F-D13 with twice the extracted five-bit field. Its argument is therefore an even number from zero through 62.

It then calls F-D14, whose result becomes the first argument of the first loop iteration’s opening call.

The first iteration’s second argument is the extracted five-bit field shifted right by eight. Since that field is at most 31, this argument is always zero.

### 7. Repeating cycle

Each iteration has the following order:

1. Make the opening call with two explicit arguments.
   - The first iteration calls F-D15.
   - Every subsequent iteration calls F-D25.
2. Call F-D16.
3. Call F-D17.
4. Call F-D18.
5. Call F-D19 and retain its result.
6. Repeat the scaled-selector clear from section 5, including a fresh selector-byte read.
7. Call F-D20 with the F-D19 result shifted left by two bits.
8. Call F-D21.
9. Call F-D22.
10. Call F-D23.
11. Call F-D24 and retain its result.
12. Derive the next iteration’s opening-call arguments and repeat.

Thus, each iteration contains ten calls and one selector read followed by one zero-byte write. The distinction between F-D15 and F-D25 is essential: the opening target changes after the first iteration.

Under a 32-bit bit-pattern interpretation, the next opening-call arguments are:

- First argument: the preceding F-D24 result shifted left by two bits, retaining the low 32 bits.
- Second argument: the preceding F-D24 result shifted left by 26 bits, retaining the low 32 bits, then shifted right logically by eight.

The latter transformation depends only on the low six bits of the preceding result. Equivalently under that interpretation, it places those six bits at bit positions 18 through 23, producing a value from zero through `0x00FC0000` in multiples of `0x00040000`.

There is no represented loop counter, termination predicate, completion callback, or normal exit.

## Side effects

### Direct memory effects

Before entering the loop, the function directly performs:

- One byte write containing input data.
- Two selector-byte reads through the ambient selector base, each followed by a zero-byte write.
- One selector-byte read through the scaled first-input address, followed by a zero-byte write.

Every completed loop iteration adds:

- One further selector-byte read through the scaled first-input address.
- One further zero-byte write selected by that read.

All selector-based write destinations lie in the inclusive address range `0x11`–`0x110`. Selector values are bytes, not pointer-sized values loaded from memory.

Repeated reads must not be treated as cached values. A dependency, an overlapping earlier store, or external activity may change the selector between reads.

### Dependency effects

Dependency calls occur between direct memory operations and may introduce arbitrary additional observable effects. Their effects, failure modes, and synchronization behavior are unknown.

The representation does not establish that any dependency:

- Preserves globals or reachable memory.
- Leaves selector bytes unchanged.
- Allocates or releases memory.
- Preserves resources referenced by the caller.
- Returns normally.

### Ownership and errors

No explicit allocation, release, ownership transfer, or resource acquisition appears in the represented function itself. This does not establish ownership behavior for its dependencies.

No explicit error result, validation branch, recovery action, or rollback is shown. An invalid access or invalid call target may prevent all subsequent operations. The handling of such failures is `documented_only`.

## Dependencies

The following opaque identifiers are local labels for represented call targets. They do not claim recovered names, verified signatures, or semantic identities. Distinct labels denote distinct target expressions in the supplied representation, not proof of distinct underlying implementations.

| Function ID | Established role |
|---|---|
| F-D01 | Initial target derived from the ambient call-origin value; receives the shifted fourth input. |
| F-D02 | Second prefix call. |
| F-D03 | Receives the first input shifted right by eight. |
| F-D04 | Prefix call following F-D03. |
| F-D05 | Prefix call following F-D04. |
| F-D06 | Prefix call following F-D05. |
| F-D07 | Supplies a result later shifted left by two. |
| F-D08 | Receives the shifted F-D07 result. |
| F-D09 | Continuation call following F-D08. |
| F-D10 | Continuation call following F-D09. |
| F-D11 | Continuation call following F-D10. |
| F-D12 | Supplies the word from which bits 25–29 are extracted. |
| F-D13 | Receives twice the extracted five-bit field. |
| F-D14 | Supplies the first iteration’s initial first argument. |
| F-D15 | Opening call of the first loop iteration only. |
| F-D16 | Loop call following the opening call. |
| F-D17 | Loop call following F-D16. |
| F-D18 | Loop call following F-D17. |
| F-D19 | Supplies the result shifted for F-D20. |
| F-D20 | Receives the shifted F-D19 result after the selector clear. |
| F-D21 | Loop call following F-D20. |
| F-D22 | Loop call following F-D21. |
| F-D23 | Loop call following F-D22. |
| F-D24 | Supplies both derived arguments for the next iteration. |
| F-D25 | Opening call of every iteration after the first. |

All targets other than F-D01 are represented through fixed address arithmetic. No dependency body or behavioral contract is supplied. No self-call to F-b043bf3d56cd is established.

## Acceptance examples

These are conditional contract examples, not executed tests. They assume valid accesses, returning dependencies, and no unspecified interference unless stated otherwise.

### Initial store and shifted arguments

With first input `0x123456AB` and fourth input `0x00000100`:

- The first byte write stores `0xAB` at address `0x1E`.
- F-D01 receives `2`.
- F-D03 receives `0x00123456`.

The store precedes the first ambient-selector read and all dependency calls.

### Fresh ambient selector

Suppose the first ambient selector read produces `5`. The corresponding write stores zero at `0x16`.

If dependencies change that selector byte to `9` before the second ambient-selector read, the second write targets `0x1A`, not `0x16`. Reusing the first selector value would violate the represented behavior.

### Five-bit extraction

If F-D12 returns `0xFFFFFFFF`:

- The extracted field is `31`.
- F-D13 receives `62`.
- F-D15’s second argument is zero.

Bits above bit 29 do not contribute to the extracted field.

### Unsigned address wrapping

If the first input is `0x00400000`, multiplying by `0x400` wraps to zero in 32-bit unsigned arithmetic. The scaled selector is therefore read at address `1`.

This example establishes the address calculation only. It does not establish that address `1` is readable.

### Loop-carried arguments and target change

Suppose F-D14 returns `7`, F-D19 returns `3`, and the first iteration’s F-D24 returns `1`.

- F-D15 receives first argument `7` and second argument zero.
- After the iteration’s selector clear, F-D20 receives `12`.
- The next iteration opens with F-D25, not F-D15.
- F-D25 receives first argument `4` and second argument `0x00040000`.

These chosen shifts fit within signed 32-bit positive range, avoiding the signed-shift ambiguity.

### No normal completion

If every call returns and every access succeeds indefinitely, the function keeps executing the repeated cycle. Returning normally after a finite number of iterations would not match the visible control flow.

## Uncertainties

The following are `documented_only`:

- **Control-flow completeness:** supplied analysis warnings report invalid instruction data and removed unreachable regions. Missing or misrepresented behavior cannot be excluded.
- **Actual ABI:** complete parameter passing, return-value conventions, register preservation, and hidden arguments are unknown.
- **Ambient prerequisites:** the origins and required validity of both ambient values are unknown.
- **Signed shifts:** several left shifts operate on values represented as signed. Negative operands or nonrepresentable signed results are not assigned portable language-level behavior here. The modulo-bit-pattern formulas describe a conditional machine interpretation.
- **Memory meaning:** the low addresses may identify ordinary memory, mapped hardware, or invalid locations. No interpretation is established.
- **Dependency behavior:** effects on memory, globals, ownership, exceptions, and control flow are unknown.
- **Concurrency and ordering guarantees:** the listed sequence is the order represented by the supplied operations, not a demonstrated hardware memory-ordering or synchronization guarantee.
- **Application intent:** no domain-specific purpose is established.

## Evidence

Established static evidence consists of the supplied representation of F-b043bf3d56cd: its four-input interface, byte accesses, integer expressions, indirect calls, and unconditional repeating control flow.

Arithmetic ranges, the first iteration’s zero second argument, and the change from F-D15 to F-D25 follow from that representation. The dependency identifiers in this document are opaque labels assigned for explanation, not recovered symbols.

No dependency implementation, machine execution trace, or independent ABI evidence was supplied. No runtime tests have been performed.
