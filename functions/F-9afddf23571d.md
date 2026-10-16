<!-- Trusted provenance: F-9afddf23571d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9afddf23571d

## Purpose

F-9afddf23571d computes a scaled product of two signed integer inputs and returns a signed integer result. The represented calculation multiplies the second input by the first, doubles that product, and then right-shifts the doubled value by 16 bit positions.

For nonnegative values whose multiplications are representable, this is equivalent to dividing the mathematical input product by 32,768 and discarding the fractional part. That equivalence is conditional: intermediate overflow and the treatment of negative right shifts prevent it from being an unconditional specification for every input.

**Established static evidence:** two integer inputs, one integer return, two multiplications, and a final right shift with a count of 16.

**Inferred intent:** scaled integer multiplication, possibly for fixed-point arithmetic. No fixed-point format, units, or interpretation of the inputs is established.

## Interface

The function accepts two positional inputs:

| Position | Meaning established by the source | Declared type |
|---|---|---|
| First | One multiplicative operand | Signed `int` |
| Second | The other multiplicative operand | Signed `int` |

The result is also declared as signed `int`. There are no explicit output parameters, pointer parameters, buffers, handles, or callback arguments.

### Width and representation

The supplied material does not independently establish the width of `int`, its physical representation, or an ABI. A 32-bit interpretation is useful for conditional acceptance examples, but is not an unconditional fact about the interface.

Under a 32-bit signed-integer model:

- Each input and the result occupy 32 bits.
- The signed range is −2,147,483,648 through 2,147,483,647.
- The displayed expression does not explicitly widen either multiplication to 64 bits.
- The right-shift count is 16, not a mask or a request to return a 16-bit object.
- The result remains a signed 32-bit integer even when its value fits within a narrower range.

Under ordinary source-language integer rules, the multiplications occur in the `int` arithmetic domain. The small integer constants do not themselves request wider arithmetic.

The shift also requires an operand width greater than 16 for ordinary defined shift semantics. An implementation with a 16-bit `int` would therefore need separate treatment; the supplied material does not resolve that case.

### Ownership and lifetime

Inputs are passed by value. No caller-owned memory is explicitly accessed, retained, allocated, transferred, or released. The returned integer has no associated ownership or lifetime obligation.

## Behavior

### Arithmetic stages

The represented calculation has three dependent stages:

1. Form the product of the second input and the first input.
2. Multiply that intermediate product by two.
3. Right-shift the doubled intermediate by 16 positions and return the result.

This specifies the grouping of the arithmetic. It does not establish a temporal order for evaluating the two input operands of the first multiplication. Because those operands are already integer values and contain no explicit side-effecting expressions, no caller-visible memory-read ordering follows from their textual order.

There is no explicit branch, loop, validation step, saturation, error result, or recovery path.

### Representable, nonnegative calculations

When both multiplications are representable and the doubled intermediate is nonnegative, the shift produces the integer quotient of that intermediate divided by 65,536.

Equivalently, the result is the floor of the mathematical input product divided by 32,768. The multiplication must still be valid before this equivalence is used; a small mathematical quotient does not excuse an overflowing intermediate.

For these cases:

- Product values from 0 through 32,767 return zero.
- A product of 32,768 returns one.
- Fractional portions are discarded without rounding to nearest.
- Both negative inputs can produce this nonnegative case when their product and its doubling are representable.

### Negative calculations

When the doubled intermediate is negative, the supplied expression does not independently establish a portable numerical result.

Under an arithmetic right-shift model, the result is the mathematical input product divided by 32,768, rounded toward negative infinity. A small negative product therefore returns −1 rather than zero.

That behavior must not be silently replaced by signed division that truncates toward zero. Conversely, arithmetic right shift must not be asserted as an established platform fact without supporting implementation evidence.

### Overflow boundaries

The source contains no explicit overflow detection, saturation, wrapping conversion, or wider intermediate type.

Under ordinary C signed-arithmetic semantics, overflow in either multiplication prevents a defined result from being promised. The final shift does not repair overflow that has already occurred.

For the conditional 32-bit model, let the mathematical product of the inputs be \(P\). Both multiplication stages are representable exactly when:

\[
-1{,}073{,}741{,}824 \leq P \leq 1{,}073{,}741{,}823.
\]

This interval ensures that both \(P\) and \(2P\) fit in the signed 32-bit range. Its asymmetry matters: −1,073,741,824 can be doubled representably, while +1,073,741,824 cannot.

If a separate execution model establishes fixed-width wrapping multiplication, overflow cases require a different contract. Wrapping is not established by the supplied material.

## Side effects

The represented body has no explicit:

- Global or static state access.
- Memory dereference or memory write.
- Allocation or deallocation.
- Input/output operation.
- Dependency invocation.
- Persistent state change.

The only explicit externally delivered value is the return value.

There is consequently no explicit sequence of observable memory reads or writes to specify. Input values are consumed by arithmetic, and the resulting value is returned.

These observations do not establish the absence of compiler instrumentation, execution-environment traps, or other behavior outside the represented body. In particular, an overflowing operation must not be described as reliably returning an error, raising an exception, or continuing with a wrapped value.

## Dependencies

No dependency invocation is present. F-9afddf23571d has no identified callee IDs.

Its unresolved environmental requirements are arithmetic rather than function dependencies:

- The width and range of `int`.
- The treatment of negative signed right shifts.
- The applicable semantics for signed overflow.
- The calling convention used to convey the two arguments and return value.

No assumptions about callee ownership, global preservation, or memory release are needed because no callee is shown.

## Acceptance examples

The following examples assume signed 32-bit inputs, intermediates, and return values. Negative-result examples additionally require the stated shift policy. These are conditional acceptance criteria, not observed executions.

| First input | Second input | Expected result or restriction |
|---:|---:|---|
| 0 | 2,147,483,647 | Returns 0; both multiplication stages remain representable. |
| 1 | 32,767 | Returns 0. |
| 1 | 32,768 | Returns 1. |
| 1 | 65,535 | Returns 1, demonstrating discarded fractional precision. |
| 1 | 65,536 | Returns 2. |
| −1 | −32,768 | Returns 1. |
| 32,767 | 32,769 | Returns 32,767; their product is 1,073,741,823, the positive admissible boundary. |
| −32,768 | 32,768 | Returns −32,768 under arithmetic right shift; the doubled product is exactly −2,147,483,648. |
| −1 | 1 | Returns −1 under arithmetic right shift, not zero. Without a negative-shift policy, no portable numerical expectation is assigned here. |
| 32,768 | 32,768 | Doubling the representable product overflows signed 32-bit arithmetic; no defined result is promised under ordinary C semantics. |
| −2,147,483,648 | −1 | The first multiplication overflows signed 32-bit arithmetic; no defined result is promised. |

Exchanging the two inputs preserves the mathematical product. Thus, for representable calculations under the same shift policy, exchanged inputs must return the same value. This is a numerical symmetry, not evidence of an operand-read ordering.

## Uncertainties

The following remain **documented_only**:

- **Actual integer width and ABI:** neither is established by the supplied function text.
- **Negative-shift policy:** arithmetic right shift is a conditional model, not a confirmed property.
- **Overflow execution behavior:** wrapping, trapping, or another machine-level outcome is not established.
- **Input domain restrictions:** no caller evidence establishes that inputs always remain within the representable-product interval.
- **Numeric format:** a fixed-point interpretation is plausible, but signedness conventions beyond the declared type, fractional-bit assignments, and application units are unknown.
- **Error handling outside this body:** no surrounding caller or runtime behavior is supplied.

These unknowns do not justify adding checks, saturation, rounding, wider intermediates, or fallback results to the contract.

## Evidence

This contract is based solely on static inspection of the supplied function representation. That evidence establishes the two signed-integer parameters, signed-integer return, arithmetic grouping, doubling operation, final shift count, and absence of explicit calls or memory operations.

The numerical boundaries and examples are deductions under explicitly stated arithmetic assumptions. Fixed-point purpose is inferred intent only.

No runtime tests, platform inspection, caller inspection, or independent verification have been performed.
