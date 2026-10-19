<!-- Trusted provenance: F-4cbd6328b1ab; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4cbd6328b1ab

## Purpose

F-4cbd6328b1ab selects one of three fixed 32-bit results using only whether each of two integer inputs is zero.

The first input has precedence. When it is zero, the result is always `0x0000003B`, regardless of the second input. When the first input is nonzero, the second input determines whether the result is `0x0000002C` or `0x00000020`.

The selection behavior is established by the supplied static source. The meanings of the returned values are not established. They must be treated as numeric results, not assigned character, status, formatting, or protocol semantics without additional evidence.

## Interface

**Inputs**

- **First input:** an integer value, classified solely as zero or nonzero.
- **Second input:** an integer value, classified solely as zero or nonzero when the first input is nonzero.

Both inputs are passed by value in the supplied function representation. Neither is used as an address, size, offset, index, or ownership-bearing reference.

The source declares both inputs using an integer type, but does not independently establish that type’s bit width or the calling convention. The behavioral contract therefore applies to values representable by the actual input type; it does not prescribe how a wider external value must be converted before the call.

**Output**

One 32-bit value is returned:

| First input | Second input | Return value | Decimal value |
|---|---|---:|---:|
| Zero | Any representable value | `0x0000003B` | 59 |
| Nonzero | Zero | `0x0000002C` | 44 |
| Nonzero | Nonzero | `0x00000020` | 32 |

These cases are exhaustive for integer inputs. No other result is produced by the supplied body.

The output representation is identified as four bytes by the supplied source type. Its signedness is not established. All three results are positive and fit in both signed and unsigned 32-bit integer representations, so that uncertainty does not change their numeric values.

No output buffer, returned allocation, aggregate layout, or serialized byte sequence is involved. Endianness does not affect the numeric contract; the source does not specify a memory representation for callers that later store or serialize the result.

## Behavior

### Selection and precedence

The function first determines whether the first input is zero.

- If it is zero, the function returns `0x0000003B`. The second input does not participate in the selection.
- If it is nonzero, the function determines whether the second input is zero:
  - A zero second input selects `0x0000002C`.
  - A nonzero second input selects `0x00000020`.

The first input’s zero case overrides every possible value of the second input. In particular, two zero inputs produce `0x0000003B`, not `0x0000002C`.

The two inputs are not interchangeable. Exchanging them can change the result because they have different roles in the decision.

### Value boundaries

Only equality with zero matters. There is no distinction among different nonzero magnitudes.

Where the actual input type supports negative values, negative values belong to the same nonzero class as positive values. For example, replacing a representable positive first input with a representable negative first input leaves the result unchanged when the second input remains in the same zero/nonzero class.

The function performs no addition, subtraction, multiplication, division, shifting, narrowing arithmetic, or range-dependent calculation. Consequently, no arithmetic overflow, underflow, division-by-zero, or shift-boundary behavior is introduced by this body.

### Evaluation order and observability

The established logical order is:

1. Classify the first input as zero or nonzero.
2. Classify the second input only on the first-input-nonzero path.
3. Return the selected constant.

This order describes the supplied function body. It does not establish which registers or stack locations a compiled implementation accesses, nor does it constrain a caller’s evaluation of expressions used to supply the arguments. The second input remains part of the interface even when its value cannot affect the result.

There are no observable memory reads or writes expressed in the body beyond receiving input values and returning the result. No stronger machine-instruction ordering guarantee is established.

### Preconditions and errors

No value-level precondition is visible beyond supplying two values valid for the actual integer parameter type and using the correct calling convention.

Every pair of valid integer inputs selects a result. The body contains no error return distinct from the three listed results, no exception mechanism, no retry, and no failure-dependent branch.

This does not establish that an invalid ABI-level call is safe. Incorrect argument passing, incompatible declarations, or execution-environment faults fall outside the supplied behavioral evidence.

## Side effects

The supplied body shows no externally visible state change other than its return value.

Specifically, it contains no:

- Global or static state access.
- Pointer dereference.
- Caller-owned memory modification.
- Allocation, deallocation, or ownership transfer.
- I/O, logging, callback, or synchronization operation.
- Dependency invocation.

There is therefore no object lifetime or cleanup obligation established by this function. The numeric result is not shown to represent an owned resource.

Repeated calls with the same input classifications select the same result in the supplied body. This is a static property of the shown selection logic, not a claim about unexamined wrappers, instrumentation, or the surrounding execution environment.

## Dependencies

No function dependencies are present in the supplied body.

All established behavior of F-4cbd6328b1ab follows from its two zero comparisons and constant return values. No unknown callee needs to preserve global state, validate inputs, allocate memory, or release resources.

The caller’s use of the returned value is outside this contract. No downstream interpretation or consumer behavior is supplied.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

| Inputs | Required result | Property illustrated |
|---|---:|---|
| First input zero; second input zero | `0x0000003B` | The first-input-zero case takes precedence. |
| First input zero; second input positive and representable | `0x0000003B` | A nonzero second input cannot override the first-input-zero case. |
| First input zero; second input negative and representable | `0x0000003B` | If negatives are supported, they do not affect this path. |
| First input positive and representable; second input zero | `0x0000002C` | A nonzero first input enables the second-input decision. |
| First input negative and representable; second input zero | `0x0000002C` | The first input’s sign does not distinguish nonzero cases. |
| Both inputs nonzero and representable | `0x00000020` | Nonzero magnitudes and signs do not alter this result. |

Additional acceptance boundaries:

- Holding the first input at zero and varying the second across all representable values must leave the result at `0x0000003B`.
- Holding the first input nonzero and changing the second from zero to nonzero must change the result from `0x0000002C` to `0x00000020`.
- For any representable nonzero value, inputs consisting of that value followed by zero produce `0x0000002C`; reversing those inputs produces `0x0000003B`.
- If the actual input type has representable minimum and maximum values, either boundary must behave according to its zero/nonzero classification, without an additional boundary-specific result.

## Uncertainties

**Established static evidence**

- There are two integer inputs.
- The first input’s zero/nonzero classification controls whether the second input matters.
- The complete result set is 59, 44, and 32.
- The return representation is four bytes.
- The body contains no calls or explicit external state accesses.

**Inferred intent**

The inputs may be intended as flags because each is tested only against zero. This interpretation is not required by the behavior and does not justify restricting accepted nonzero values to one particular flag encoding.

**Unknowns — documented_only**

- The exact input integer width, ABI, and calling convention.
- The semantic meaning of the three numeric results.
- The caller’s interpretation of those results.
- Any wider interface restrictions imposed outside the supplied body.
- The compiled instruction sequence and physical argument-access order.

None of these unknowns changes the established selection table for valid integer inputs.

## Evidence

This contract is based solely on static inspection of the supplied body for F-4cbd6328b1ab. The evidence consists of an outer zero comparison on the first input, a conditional zero comparison on the second input, and three fixed return values.

No runtime tests have been performed. The acceptance examples are derived requirements, not observed execution results.
