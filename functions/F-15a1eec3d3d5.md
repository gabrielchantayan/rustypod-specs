<!-- Trusted provenance: F-15a1eec3d3d5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-15a1eec3d3d5

## Purpose

F-15a1eec3d3d5 constructs a 64-bit result from selected bits of three 32-bit inputs. A fourth 32-bit input is accepted but does not contribute to the result.

The result consists of:

- A low-order 32-bit half taken unchanged from the first input.
- A high-order 32-bit half whose lower 31 bits come from the second input and whose highest bit comes from the fourth input.

The function therefore replaces the highest bit of the second input with the highest bit of the fourth input, then places that modified word above the first input.

This is an established bit-level operation. Interpreting the replaced bit as a sign bit, or interpreting the assembled result as a floating-point value, would require context not present in the supplied evidence.

## Interface

### Inputs

The interface has four inputs, in this order:

| Position | Width | Contract role |
|---|---:|---|
| First | 32 bits | Supplies all bits of the result’s low-order half. |
| Second | 32 bits | Supplies bits 0 through 30 of the result’s high-order half. Its bit 31 is discarded. |
| Third | 32 bits | Accepted but unused in the supplied body. |
| Fourth | 32 bits | Supplies bit 31 of the result’s high-order half. Its bits 0 through 30 are discarded. |

The second and fourth inputs are represented as unsigned 32-bit quantities in the supplied evidence. The first and third inputs have a known width of 32 bits but no established higher-level type.

For this contract, all inputs are described as bit patterns. No signed conversion is needed to explain the behavior.

### Output

The function returns one 64-bit bit pattern. Its bit layout is:

| Result bits | Source |
|---|---|
| 0–31 | First input bits 0–31 |
| 32–62 | Second input bits 0–30 |
| 63 | Fourth input bit 31 |

Bit 0 denotes the least significant bit.

Equivalently, if the inputs in interface order are \(A\), \(B\), \(C\), and \(D\), the unsigned numerical interpretation of the result is:

\[
R =
A +
2^{32}\left((B \mathbin{\&} \text{0x7FFFFFFF})
\mathbin{|}
(D \mathbin{\&} \text{0x80000000})\right).
\]

Here, \(A\) is interpreted as an unsigned 32-bit value solely to express the concatenation numerically. This does not establish a signedness requirement for the caller’s original representation.

### Preconditions and validity

No input-dependent precondition is visible. Every combination of four 32-bit input patterns has a defined result under the supplied operation.

The evidence does not establish how arguments wider than 32 bits would be converted by a caller or foreign-function interface. Such conversions are outside this contract.

There are no pointer, buffer, length, alignment, or nullability requirements in the visible operation.

## Behavior

### Result construction

The function preserves the entire first input as the low-order word.

For the high-order word, it retains the second input’s lower 31 bits and removes that input’s highest bit. Independently, it retains only the fourth input’s highest bit. Those retained portions occupy disjoint bit positions and are combined into one 32-bit word.

The resulting high-order word is concatenated above the first input. Concatenation produces exactly 64 result bits; it does not sign-extend either word.

The third input is not consulted by the supplied body.

### Observable relationships

For any fixed first, second, and fourth inputs, changing the third input leaves the returned bits unchanged.

Changing only the second input’s bit 31 also leaves the returned bits unchanged. That bit is discarded regardless of the fourth input.

Changing only the fourth input’s lower 31 bits leaves the returned bits unchanged. Those bits do not participate in the result.

Changing a bit of the first input changes exactly the corresponding bit among result bits 0–31. Changing one of the second input’s lower 31 bits changes exactly the corresponding bit among result bits 32–62. Changing the fourth input’s highest bit changes exactly result bit 63.

These relationships follow directly from the nonoverlapping result fields. They do not depend on interpreting any input as a signed integer.

### Width and arithmetic boundaries

The masks operate on 32-bit inputs. The modified high-order word remains a 32-bit pattern, including when its highest bit is set.

The final operation combines two 32-bit words into a 64-bit result. There is no visible addition, multiplication, division, arithmetic overflow check, saturation, or rounding in the supplied body. The equation above is a mathematical description of that bit assembly, not evidence of additional arithmetic instructions.

Every 64-bit pattern is representable as an output for some inputs: the first input can provide any low-order word, the second input can provide any lower 31 bits of the high-order word, and the fourth input can provide either value of the remaining bit.

### Evaluation and memory order

No data-memory reads or writes are explicit in the supplied body. The function operates on its input values and returns a value.

The evidence establishes the resulting bit dependencies, not a physical instruction schedule. In particular, it does not establish an observable order for obtaining the individual argument values. Evaluation of caller-side argument expressions occurs outside this function’s visible behavior.

## Side effects

The supplied body contains no explicit:

- Global-state access or mutation.
- Pointer dereference or buffer access.
- Allocation, deallocation, or ownership transfer.
- Input/output operation.
- Synchronization operation.
- Dependency invocation.
- Error-state update.

The visible externally relevant effect is the returned 64-bit value. No caller-provided object is explicitly modified.

There is no visible error branch, status return separate from the result, or distinguished failure pattern. A result of zero, all ones, or any other 64-bit pattern is an ordinary result of the bit-selection rule, not an established error indication.

Platform-level calling mechanics, including register use and any stack activity, are not specified by the supplied body.

## Dependencies

No dependency calls appear in the supplied body. No additional opaque function IDs are required to explain the transformation.

The behavior does not visibly rely on a callee preserving globals, supplying storage, or releasing memory. There is no basis here for assigning any such responsibility to another function.

## Acceptance examples

The following are conditional acceptance examples derived from static evidence. They have not been executed. Hexadecimal inputs denote exactly 32 bits; hexadecimal results denote exactly 64 bits.

| First input | Second input | Third input | Fourth input | Required result |
|---|---|---|---|---|
| `0x00000000` | `0x00000000` | Any | `0x00000000` | `0x0000000000000000` |
| `0x89ABCDEF` | `0x12345678` | Any | `0x00000000` | `0x1234567889ABCDEF` |
| `0x89ABCDEF` | `0x12345678` | Any | `0x80000000` | `0x9234567889ABCDEF` |
| `0x89ABCDEF` | `0x92345678` | Any | `0x00000000` | `0x1234567889ABCDEF` |
| `0x00000000` | `0xFFFFFFFF` | Any | `0x00000000` | `0x7FFFFFFF00000000` |
| `0xFFFFFFFF` | `0x00000000` | Any | `0xFFFFFFFF` | `0x80000000FFFFFFFF` |
| `0xFFFFFFFF` | `0xFFFFFFFF` | Any | `0xFFFFFFFF` | `0xFFFFFFFFFFFFFFFF` |

Additional acceptance conditions:

- Replacing the third input with any other 32-bit pattern must preserve the result.
- Replacing the second input with a value differing only in bit 31 must preserve the result.
- Replacing the fourth input with a value having the same bit 31 but different lower bits must preserve the result.
- If the fourth input’s highest bit changes and all other contributing bits remain fixed, only the result’s highest bit must change.
- The first input’s highest bit must remain result bit 31; it must not be extended into the high-order word.

These conditions distinguish exact field selection from alternatives such as sign extension, copying the entire fourth input, or retaining the second input’s original highest bit.

## Uncertainties

The following matters are **documented_only** because they are not established by the supplied evidence:

- The semantic meaning of the assembled 64-bit pattern.
- Whether callers interpret the result as an integer, floating-point representation, packed record, or another object.
- The intended purpose of the unused third input.
- The machine calling convention and physical return-value placement.
- The byte order used if a caller subsequently stores the result in memory.
- Caller-side conversions, argument evaluation, validation, or subsequent state changes.

Bit positions and high/low word placement are established independently of memory byte order. This contract does not prescribe a byte sequence for serialization.

A possible intent is to transplant a sign-like bit while retaining other fields. That interpretation is inferred only; the established behavior is the bit transformation, without a numeric-format interpretation.

## Evidence

The supplied static body establishes a single return operation that combines two 32-bit halves. The low half comes directly from the first input. The high half combines the second input masked by `0x7FFFFFFF` with the fourth input masked by `0x80000000`.

The third input has no visible use. The body contains no explicit calls, branches, loops, memory accesses, or state updates.

All behavioral requirements and acceptance examples above are derived from that static operation. No runtime tests have been performed.
