<!-- Trusted provenance: F-af60ab6f2bde; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-af60ab6f2bde

## Purpose

F-af60ab6f2bde reports a scaled signed separation between two adjacent integer values in caller-provided storage. The later element supplies the endpoint; the earlier element supplies the starting value. The scaling discards two low-order bit positions from their signed difference.

For a nonnegative difference within the signed integer range, the result counts complete groups of four. Negative differences require the execution environment’s signed-right-shift rule. An arithmetic right shift gives division by four rounded toward negative infinity, not toward zero.

The input values have no established application-level meaning. They are not established as addresses, byte counts, array bounds, or measurements. Interpreting the scaling as conversion between particular units would be inferred intent, not a contractual fact.

## Interface

**Input.** One pointer designating the first of two consecutive signed integer elements.

**Output.** One signed integer returned directly to the caller. There is no separate output buffer, status value, or error channel.

**Storage layout.** The first input is the element designated by the pointer. The second is the immediately following element of the same integer sequence. Their starting addresses are separated by one element’s storage size. The supplied evidence does not independently establish that size as four bytes.

The elements must belong to storage that permits both integer accesses under the applicable alignment, object-lifetime, initialization, and object-access rules. Merely having readable bytes after the first element does not by itself establish that the second integer access is valid.

**Widths and representation.** Both inputs and the result are represented as signed integers of the same declared kind. No explicit wider arithmetic type or unsigned conversion is present. The evidence does not fix:

- Integer bit width or storage size.
- Pointer width.
- Integer byte order.
- Signed integer representation.
- Argument-passing or return-value ABI details.

A signed 32-bit model can be used for conditional examples, but it is not an unconditional interface guarantee.

**Ownership.** The caller retains the input storage. This function neither allocates nor releases it, stores the pointer for later use, nor transfers ownership elsewhere. The storage must remain valid for the duration of the accesses.

**Supported input conditions.** Both elements must be accessible and initialized, and their mathematical difference must fit in the represented signed arithmetic type for a defined source-level arithmetic result. Concurrent activity must not invalidate the accesses or create a data race. These conditions are not checked by the function.

A negative representable difference is not inherently an overflow case. Its result instead depends on the environment’s treatment of signed right shift.

## Behavior

The observable numerical relationship is directional: the first element is the starting value and the second element is the endpoint. Reversing them can change the sign and result.

The scaling applies to their signed difference. It does not independently scale the two endpoints and then compare the scaled values. Fractional contributions at the endpoints therefore cannot be discarded separately.

For a nonnegative, representable difference, the result is fully determined:

- Equal values produce zero.
- A positive separation smaller than four also produces zero.
- Separations from four through seven produce one.
- Separations from eight through eleven produce two.
- More generally, only complete groups of four contribute to the returned value.

Neither endpoint must itself be nonnegative. Two negative inputs can have a positive difference, and inputs of opposite signs can also be supported when the difference remains representable.

For a negative difference, the supplied evidence establishes signed right shifting by two positions, but does not establish the target-specific result. Under arithmetic right shift, the result is the mathematical difference divided by four and rounded downward. Thus a difference of negative one produces negative one; a difference of negative five produces negative two. Replacing this behavior with division rounded toward zero would change those results.

Overflow must be considered before scaling. A mathematical difference outside the signed range has no defined source-level result, even if one quarter of that mathematical difference would fit in the return type. The function contains no explicit widening, saturation, overflow detection, or unsigned conversion that would supply an alternative rule.

The evidence does not justify treating an overflowing subtraction as modular wrapping. Nor does it establish a particular trap or recovery response.

**Access ordering.** Both elements contribute to the result, but neither input access has an established precedence over the other. A caller must not depend on the earlier-addressed element being read first, or on the later-addressed element being read first.

The pair is not established as an atomic snapshot. Exact machine access widths, instruction counts, and physical scheduling are outside the available evidence. The logical requirement to obtain two integer values does not establish two particular load instructions.

## Side effects

For valid, ordinary input storage, the represented function has no explicit writes. It does not alter either element or update any represented global state.

There are no represented allocations, releases, callbacks, logging operations, external I/O operations, synchronization operations, or persistent state changes. No input-dependent branch selects an alternate processing or error path.

The absence of writes is not a synchronization guarantee. In particular, unsynchronized modification of the input storage is not made safe by this function’s read-only use of that storage.

Invalid inputs are not converted into a defined sentinel:

- A null pointer has no special accepted meaning.
- Storage containing only the first element is insufficient.
- Misaligned, inaccessible, expired, or otherwise invalid storage has no documented recovery behavior.
- Uninitialized values and unsupported concurrent access are outside the supported contract.
- Signed subtraction overflow has no documented error return.

The precise manifestation of an invalid access is unresolved. A trap, termination, or other particular failure must not be promised from this evidence alone.

## Dependencies

No function calls are present. There are therefore no callable dependencies requiring additional opaque function IDs, and no callee ownership or global-preservation assumptions are needed.

The result still depends on execution-environment rules rather than callable services:

- The signed integer type’s range and representation.
- The layout and validity of the two input elements.
- The rule for right shifting a negative signed value.
- The source-level arithmetic rules governing out-of-range subtraction.
- The memory model governing access to caller storage.

These dependencies constrain which numerical claims are portable. They do not imply hidden validation, allocation, or error handling.

## Acceptance examples

These are static, conditional acceptance criteria, not runtime observations. Every example assumes valid, initialized storage for both consecutive elements and an absence of invalid concurrent modification.

| First element | Second element | Required result | Additional condition |
|---:|---:|---:|---|
| 0 | 0 | 0 | None beyond the shared conditions. |
| 10 | 13 | 0 | The positive difference is three. |
| 10 | 14 | 1 | The positive difference is four. |
| 10 | 17 | 1 | The positive difference is seven. |
| 10 | 18 | 2 | The positive difference is eight. |
| −8 | −4 | 1 | Negative endpoints do not invalidate a positive difference. |
| 3 | 4 | 0 | Scaling applies to the difference, not separately to the endpoints. |
| 9 | 8 | −1 | Requires arithmetic right shift. |
| 9 | 4 | −2 | Requires arithmetic right shift. |

The last example distinguishes downward rounding from rounding toward zero. Under the stated arithmetic-right-shift condition, negative one is not an acceptable result for that pair.

Additional boundaries are conditional on a signed 32-bit model:

| First element | Second element | Required outcome |
|---:|---:|---|
| 0 | 2,147,483,647 | Return 536,870,911. |
| 0 | −2,147,483,648 | Return −536,870,912 if negative values use arithmetic right shift. |
| −2,147,483,648 | 2,147,483,647 | No defined source-level numerical result: the difference exceeds the signed range. |
| 2,147,483,647 | −2,147,483,648 | No defined source-level numerical result: the difference is below the signed range. |

Neither overflow example may be assigned a wrapping result without additional evidence. Both remain outside the defined source-level domain despite the later scaling operation.

For all defined examples, the function itself must leave caller storage unchanged. Acceptance criteria must not impose an input-read order or atomic-pair guarantee absent from the evidence.

## Uncertainties

**documented_only — Concrete widths and ABI.** The supplied representation does not establish integer size, pointer size, byte order, or calling convention. Four-byte element spacing remains conditional.

**documented_only — Negative results.** The exact result of shifting a negative signed difference requires an execution-environment rule. Arithmetic right shift is used only where explicitly stated as a condition.

**documented_only — Machine behavior after overflow.** No wrapping, saturation, trap, or stable returned value is established for out-of-range signed subtraction.

**documented_only — Physical accesses.** Exact instructions, access granularity, physical ordering, and indivisibility are unknown. No synchronized snapshot is established.

**documented_only — Invalid-input failure.** Validation is absent, but the specific failure caused by invalid storage is not established.

**documented_only — Application intent.** The reason for using adjacent integers and a fourfold scale is unknown. No unit, monotonicity requirement, nonnegative-input requirement, or enclosing record layout is established.

## Evidence

The supplied static representation establishes a single pointer input, access to two consecutive signed integer elements, a directional signed difference, two-bit signed right scaling of that difference, and a direct signed integer return. It contains no calls, explicit writes, validation branches, or alternate return paths.

The nonnegative acceptance results follow from representable signed arithmetic and right shifting. Negative numerical examples additionally assume arithmetic right shift. Boundary examples additionally assume a signed 32-bit model.

The functional description is established by static evidence. Any application-level interpretation remains inference, and unresolved environment properties remain documented_only.

No implementation execution or runtime tests have been performed.
