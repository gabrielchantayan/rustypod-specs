<!-- Trusted provenance: F-fa614b4adba2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fa614b4adba2

## Purpose

F-fa614b4adba2 describes a partially recovered routine that interleaves indirect calls with four word-sized memory writes and four byte-sized memory writes. Values returned by selected calls determine stored data, subsequent call arguments, and memory addresses.

No application-level purpose is established. In particular, the available evidence does not establish that these accesses represent an object update, serialization operation, allocation protocol, or device transaction.

The visible path ends at an invalid-instruction boundary rather than a demonstrated normal return. This contract therefore describes the recovered execution prefix, conditional on each preceding call returning and each memory access completing.

## Interface

### Explicit and implicit inputs

The recovered interface has no explicit parameters and no established return value. It nevertheless depends on incoming machine state:

| Symbol used in this contract | Meaning |
|---|---|
| B | Incoming base value used for an initial word write and word read. |
| P | Incoming 32-bit payload later written to a computed address. |
| L | Incoming link-related value used to compute the first indirect-call target. |
| Memory | Addressable storage read or written by the routine and its dependencies. |

These symbols are descriptive contract notation, not names of source-level parameters.

The first call target is computed from L by adding 4,094. Subsequent visible call targets are derived from fixed code-position values. Their numeric addresses are not needed to express the data behavior and are not reproduced here.

### Widths and arithmetic

The recovered operations expose:

- 32-bit word values and word accesses.
- 8-bit byte reads and writes.
- Two call results represented as 64-bit values, each split into lower and upper 32-bit halves.
- Logical right shifts of unsigned 32-bit values.
- Left shifts and multiplications used to form word values and addresses.

For the calculations below, `low32(x)` means the low 32 bits of x, and `low8(x)` means the low eight bits. These express the recovered fixed-width dataflow without assuming that a high-level signed shift has defined language semantics.

A 32-bit machine-arithmetic interpretation is strongly indicated. Exact address wrapping, alignment restrictions, execution-state conventions for indirect branches, and fault behavior remain documented_only. Examples avoid address overflow unless explicitly discussing bit truncation.

### Preconditions

Progress through the complete visible prefix requires:

1. Every indirect-call target to be executable under the actual calling convention.
2. Every dependency reached along the path to return control.
3. All reached memory addresses to permit the required access width.
4. The incoming state and call results to make those accesses meaningful in the execution environment.

No visible check enforces these conditions. There is no established null-pointer check, bounds check, alignment check, or error return.

The contract does not impose disjointness between memory regions. Addresses may alias one another or dependency-managed state.

## Behavior

### Dependency notation

F-D01 through F-D45 are opaque, document-local handles for successive call occurrences. They are not published identities of recovered functions, and different handles do not prove different callees. A repeated target or alias between targets remains possible.

A call described as having “no explicit arguments” has no arguments represented in the supplied recovery. This does not establish that the machine call receives no meaningful register or stack state.

### 1. Establish the first stored values

The routine calls F-D01, F-D02, F-D03, and F-D04, in that order, with no explicit arguments. Their results are not explicitly consumed by the surrounding visible operations.

It then calls F-D05 with no explicit arguments and captures a 32-bit result, R.

The next visible events are:

1. Write `low32(R × 4)` as a word at B + 36.
2. Read a word from B + 116; retain it as A.
3. Form T = `low32(R × 32)`.
4. Call F-D06 with four explicit arguments, in order:
   - A;
   - `low32(A × 2)`;
   - T;
   - `(A AND 0x7fffffff)` logically shifted right by two.

The fourth argument clears bit 31 before shifting. It is not generally equivalent to an unrestricted unsigned shift of A by two.

The routine next calls F-D07, F-D08, F-D09, and F-D10, in order, with no explicit arguments.

It then writes T as a word at A + 36.

A is the value captured by the earlier word read. There is no visible reload from B + 116 before this write. However, the dependencies between these events may have their own memory effects.

### 2. Read through the captured value and obtain a split result

After the write at A + 36:

1. Read a word at A + 116.
2. Pass that read value as the sole explicit argument to F-D11.
3. Call F-D12, F-D13, and F-D14, in order, with no explicit arguments.
4. Call F-D15 with no explicit arguments and capture its represented 64-bit result.

Let Q be the lower 32 bits of this result and H its upper 32 bits. Define S = `low32(Q × 4)`.

The routine then writes P as a word at the address formed by H × 4 + 20. This address is computed directly from H; the evidence does not show a separate base pointer added to it.

Next, F-D16 receives these four explicit arguments:

1. S;
2. S;
3. S;
4. H logically shifted right by three.

The repeated arguments are established dataflow. They do not establish the callee’s interpretation of those positions.

### 3. Consume a second split result and write through an index

F-D17 is called with no explicit arguments. Its result is again represented as 64 bits.

Let J be its lower 32 bits and K its upper 32 bits. F-D18 receives:

1. J;
2. K;
3. J;
4. K logically shifted right by three.

The routine then calls F-D19, F-D20, and F-D21, in order, with no explicit arguments.

F-D22 is called with no explicit arguments and supplies a 32-bit result, I. The routine writes S as a word at the address formed by I × 4 + 20.

S here is still the scaled lower half of the earlier F-D15 result. It is not derived from F-D17.

F-D23 is then called with `low32(I × 4)` as its sole explicit argument.

### 4. Perform two byte writes through independently read selectors

The routine calls F-D24, F-D25, and F-D26, in order, with no explicit arguments. F-D27 is then called with no explicit arguments and supplies a 32-bit result, U.

Define V = `low32(U × 4)`.

The routine:

1. Reads one unsigned byte from A + 1.
2. Adds 17 to the byte’s numeric value to form a destination address.
3. Writes `low8(V)` to that address.

This is a two-stage access: a byte read selects the address of a separate byte write. The selected destination is between 17 and 272 inclusive. It is not A plus a field offset, and it is not a write through a word-sized pointer loaded from A + 1.

The routine calls F-D28 and F-D29 with no explicit arguments, followed by F-D30 with no explicit arguments. It captures a 32-bit result, W, from F-D30.

It then independently reads the byte at A + 1 again and writes a byte at that newly selected byte value plus 17. The stored value is:

`low8((U AND 0x3fffffff) logically shifted right by 20)`.

Thus the stored byte contains original bits 20 through 27 of U. The masked-and-shifted intermediate can contain ten bits, but the byte write retains only eight.

The second selector read must not be replaced by reuse of the first selector. Intervening dependencies, or an earlier aliased write, can change the byte at A + 1.

F-D31 is subsequently called with `low32(W × 4)` as its sole explicit argument.

### 5. Perform two separately selected zero writes

The routine calls F-D32, F-D33, and F-D34, in order, with no explicit arguments. F-D35 is called with no explicit arguments and supplies a 32-bit result, X.

Using V from the F-D27 result, the routine:

1. Computes a selector address from V × 1,024 + 1.
2. Reads one unsigned byte at that address.
3. Writes a zero byte at the selector’s numeric value plus 17.

It then calls F-D36 with these four explicit arguments:

1. `low32(V × 8)`;
2. `low32(V × 8)`;
3. V;
4. `low32(X × 4)`.

Next come F-D37, F-D38, and F-D39, each with no explicit arguments. F-D40 is called with no explicit arguments and supplies a 32-bit result, Y.

The routine again reads a byte at the address formed from V × 1,024 + 1 and writes a zero byte at the newly read selector plus 17.

This second zero write is a separate observable operation. Even if its computed destination equals the first zero-write destination, the evidence does not authorize eliminating it or the intervening selector read.

F-D41 receives `low32(Y × 4)` as its sole explicit argument.

### 6. Reach the recovered endpoint

Finally, the routine calls F-D42, F-D43, F-D44, and F-D45, in order, with no explicit arguments.

The recovered path then reaches an invalid-instruction termination marker. No normal return, success result, cleanup sequence, or recovery branch is established beyond that point.

## Side effects

### Direct memory accesses

Excluding accesses inside dependencies, the visible path contains these ordered direct accesses:

| Order | Access |
|---:|---|
| 1 | Word write at B + 36: scaled F-D05 result. |
| 2 | Word read at B + 116: capture A. |
| 3 | Word write at A + 36: differently scaled F-D05 result. |
| 4 | Word read at A + 116: argument to F-D11. |
| 5 | Word write at scaled upper half of F-D15 plus 20: P. |
| 6 | Word write at scaled F-D22 result plus 20: S. |
| 7 | Byte read at A + 1. |
| 8 | Byte write at selected address plus 17: low byte of V. |
| 9 | Fresh byte read at A + 1. |
| 10 | Byte write at selected address plus 17: selected high bits of U. |
| 11 | Byte read at scaled V plus 1. |
| 12 | Byte write at selected address plus 17: zero. |
| 13 | Fresh byte read at scaled V plus 1. |
| 14 | Byte write at selected address plus 17: zero. |

These are six direct reads and eight direct writes. Dependency calls intervene as specified in Behavior.

If addresses alias, earlier writes may affect later reads. Byte and word operations may also overlap. The evidence provides no basis for treating these operations as independent field updates.

### Ownership and state

The visible routine establishes no allocation, release, ownership transfer, reference-counting operation, or lifetime guarantee.

Dependencies may modify shared state, access memory, acquire or release resources, fail, or terminate execution. Their bodies and contracts are unavailable. Their effects must not be assumed absent merely because the visible caller ignores a return value.

There is no visible rollback. If execution stops after a write, prior visible effects are not shown being undone.

## Dependencies

The explicit result-consuming calls are:

| Opaque call handle | Visible role |
|---|---|
| F-D05 | Supplies R for two word values and one F-D06 argument. |
| F-D15 | Supplies Q and H for S, an address, and F-D16 arguments. |
| F-D17 | Supplies J and K for F-D18 arguments. |
| F-D22 | Supplies I for a word-write address and F-D23 argument. |
| F-D27 | Supplies U for byte data, selector-address arithmetic, and F-D36 arguments. |
| F-D30 | Supplies W for F-D31. |
| F-D35 | Supplies X for F-D36. |
| F-D40 | Supplies Y for F-D41. |

Other explicit argument-bearing calls are F-D06, F-D11, F-D16, F-D18, F-D23, F-D31, F-D36, and F-D41. Their argument order is specified above.

All remaining calls have no represented explicit arguments and no explicitly consumed result. That classification does not imply purity, irrelevance, or removability.

Dependency identities, full signatures, calling conventions, ownership rules, and error behavior are documented_only.

## Acceptance examples

These are conditional static acceptance examples, not runtime observations. Each assumes required calls return and required accesses are valid.

1. **Different scaling of one result.** If F-D05 returns 3, the first word write stores 12 at B + 36. The later word write stores 96 at A + 36. F-D06 also receives 96 as its third argument.

2. **Clear the top bit before shifting.** If A is `0x80000008`, F-D06’s second argument is `0x00000010` under 32-bit arithmetic, while its fourth argument is 2.

3. **Preserve the earlier split result.** If F-D15 returns lower half 7 and upper half 8, S is 28. The subsequent direct word write stores P at address 52, and F-D16 receives 28, 28, 28, and 1. A later F-D17 result does not replace S for the indexed word write.

4. **Read byte selectors afresh.** Suppose U is `0x12345678`. The first byte data is `0xe0`; the second is `0x23`. If the two reads at A + 1 return 5 and 9 respectively, the writes target addresses 22 and 26.

5. **Truncate the shifted byte value.** If U is `0x3ff00000`, masking and shifting produces `0x3ff`, but the second byte write stores only `0xff`.

6. **Retain separate zero-write events.** If the two reads at the V-derived selector address return 12 and 13, zero is written at addresses 29 and 30 respectively. Equal selector results would still describe two writes, separated by dependency calls.

7. **Do not invent completion on failure.** If F-D22 does not return, its following indexed word write and every later event are not established as occurring. Earlier effects remain possible; rollback is not demonstrated.

## Uncertainties

The following remain documented_only:

- The application purpose and validity of the recovered function boundary.
- Whether all apparent calls and argument boundaries faithfully represent executable instructions.
- Whether the final invalid-instruction boundary is intentional, unreachable in practice, or caused by incomplete recovery.
- Exact dependency identities and whether different call occurrences share a target.
- The ABI mechanism for the represented 64-bit results.
- Address-space width, address overflow behavior, endianness, and alignment requirements.
- Whether byte-selected low addresses are ordinary memory, mapped hardware, or invalid in the actual environment.
- Concurrency, volatile-access requirements, and external memory observers.
- Memory ownership and dependency effects on global state or resource lifetimes.
- Any behavior beyond the recovered endpoint.

The repeated scaling and selector accesses may suggest structured data manipulation, but assigning a format or data-model meaning would exceed the evidence.

## Evidence

The supplied static recovery establishes a single visible sequential path with 45 indirect call occurrences, six direct memory reads, and eight direct memory writes. It exposes the argument expressions, selected return-value uses, byte truncations, and access ordering described here.

The same recovery reports invalid instruction data and truncates control flow at the endpoint. Consequently, this is a contract for the established visible prefix, not proof of a complete executable function.

No runtime tests have been performed. No dependency implementation, memory map, calling-convention specification, or independent execution trace was supplied. All acceptance examples are conditional deductions from the displayed operations.
