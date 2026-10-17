<!-- Trusted provenance: F-22fec3f2a1c8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-22fec3f2a1c8

## Purpose

F-22fec3f2a1c8 performs a local bit-clearing computation on an unsigned integer supplied by value. Each transformation of a nonzero working value removes its least significant set bit. The computation finishes when that working value is zero.

The function returns no value and does not publish the final zero or any intermediate result. It does not count set bits in an exposed result, despite using a transformation commonly associated with population-count algorithms.

The established contract concerns the supplied function body. Its source-level iteration count depends on the input’s number of set bits, but neither execution time nor retention of a loop in generated machine code is established.

## Interface

### Input

The function accepts one unsigned integer argument by value.

The definition of the declared unsigned type is not supplied. Its exact width, rank, and relationship to the implementation’s ordinary signed integer type remain **documented_only**.

For this contract, let **W** denote the number of value bits in that unsigned type. Its representable values are therefore:

\[
0 \leq x \leq 2^W-1.
\]

This notation does not assert a particular storage size, padding arrangement, or arithmetic evaluation width.

A 32-bit interpretation is conditional, not established. If the declared type has 32 value bits, its range is 0 through 4,294,967,295.

Every representable argument value is accepted by the supplied body. There is no nonzero-input requirement, reserved bit pattern, alignment requirement, or address-validity requirement.

### Output

The function returns normally without a return value.

It has no output parameter, result structure, status value, or exposed counter. The working value is zero at completion, but that zero remains local to the invocation.

### Passing and ownership

The argument is an integer value, not a pointer or resource handle. The function does not interpret it as an address.

Updating the local working value does not update a caller-owned object from which the argument was obtained. Argument evaluation and any conversion into the declared parameter type occur outside the supplied body and are not specified here.

No allocation, release, borrowing arrangement, or ownership transfer is present. There is no buffer layout, element stride, byte-order requirement, or caller-provided storage to describe. Calling convention, register assignment, and physical parameter layout are not established.

## Behavior

### Result of each transformation

For every nonzero working value, one transformation clears exactly its least significant set bit and preserves all other bits in the declared unsigned value.

Consequently:

- The number of set bits decreases by exactly one.
- No previously clear bit becomes set in the stored result.
- The resulting value is smaller than the preceding nonzero value.
- A value containing exactly one set bit becomes zero immediately.

For a zero working value, the transformation produces zero.

A width-independent mathematical model of the stored result is:

\[
T(x)=x\mathbin{\land}\bigl((x-1)\bmod 2^W\bigr),
\]

where the conjunction operates on W-bit representations and the remainder is chosen in the range 0 through \(2^W-1\).

This equation describes the resulting local unsigned value. It does **not** assert that the subtraction in the supplied expression is necessarily evaluated using W-bit unsigned arithmetic.

### Arithmetic evaluation and integer promotions

The arithmetic evaluation type cannot be fixed without the missing type definition and compilation environment.

Two relevant cases explain why the stored result is established even though the intermediate type is not:

- **An unsigned type narrower than the ordinary signed integer type, with every value representable by that signed type:** Integer promotion can make the subtraction signed. For zero input, the subtraction then produces signed negative one, not a W-bit unsigned all-ones intermediate. Combining zero with that intermediate still produces zero.
- **An applicable unsigned evaluation type:** Subtracting one from zero wraps to the maximum value of that evaluation type. Its width is determined by the applicable promotions and conversions; it must not automatically be identified with W.

For a nonzero input in the signed-promotion case, subtracting one remains nonnegative and representable. The bitwise operation clears the least significant set bit. For unsigned evaluation, the same stored bit-clearing result follows.

Thus, the contract does not depend on choosing one of these intermediate evaluation types. The established results are that zero remains zero and every nonzero transformation removes exactly one set bit. It would be incorrect to turn the unresolved promotion rules into either an unconditional W-bit-intermediate claim or an invented arithmetic error.

### Repetition and termination

The repetition is post-tested: a transformation occurs before the updated working value is checked for zero.

Zero input therefore receives one transformation. It is not skipped by an initial zero check and is not rejected.

For an input containing **k** set bits:

- If \(k>0\), the body executes exactly k times.
- If \(k=0\), the body executes exactly once.

A nonzero input can contain at most W set bits, so it requires at most W body executions. Zero requires one. Under the stated unsigned-value model, no representable input causes an unbounded loop.

The continuation decision uses the value after transformation. In particular, a single-set-bit input finishes after its first transformation, without an additional body execution.

### State on completion

The working value is zero when the function returns. There is no separately retained original value, exposed iteration count, or published sequence of intermediate values in the supplied body.

The caller’s stored argument remains unchanged by this body because the parameter is passed by value. This does not make a claim about unrelated caller activity or concurrent changes outside the function.

### Ordering and observability

The local transformation completes before the continuation check uses its result. A further transformation occurs only if that check finds a nonzero updated value.

The source does not establish an order between the separate operand evaluations within the arithmetic expression. Such an order is unnecessary to determine the result: the operands use the current local value, and there is no intervening call or externally observable access in the expression.

There are no pointer-based reads or writes, volatile accesses, global accesses, or dependency calls in the supplied body. Accordingly, the source-level progression is not an externally observable memory-access trace.

The contract does not require:

- Particular register or stack updates.
- A fixed instruction count.
- Observable storage of intermediate values.
- A measurable timing distinction between inputs.
- Retention of the computation after optimization.

Eliminating the local computation, where the execution environment permits it, would not change the function’s exposed return value or state effects.

## Side effects

The only state changes established by the supplied body are updates to its local working value.

The body contains no:

- Access to global or static state.
- Dereference of an address.
- Allocation or deallocation.
- Input/output operation.
- External function invocation or callback.
- Synchronization operation.
- Explicit error report, exception, trap, or abort.

The argument’s bit pattern affects the number of source-level transformations, not an exposed result.

There is no resource acquisition, cleanup obligation, partial output, or failure-recovery path. No special error behavior is defined for zero, for the maximum representable value, or for values with their highest bit set.

These statements describe explicit behavior of the supplied body. They do not specify incidental machine activity associated with function entry and return, runtime instrumentation, or an unspecified ABI.

## Dependencies

No dependency function is invoked.

The function relies on the language-level rules for unsigned integer values, integer promotions, usual arithmetic conversions, subtraction, bitwise conjunction, assignment conversion, comparison with zero, and post-tested repetition.

The missing type definition is a specification gap rather than a callable dependency. No assumption about an unknown callee’s global-state preservation, allocation behavior, or memory ownership is needed.

## Acceptance examples

These examples are conditional deductions from the supplied source, not runtime test results. Binary patterns require a declared unsigned type wide enough to represent them. Leading zero bits do not affect the results.

| Input | Local value progression | Body executions | Exposed outcome |
|---|---|---:|---|
| Zero | Zero remains zero | 1 | Normal return without a value |
| One | One becomes zero | 1 | Normal return without a value |
| Binary 1000 | 1000 becomes zero | 1 | Normal return without a value |
| Binary 1010 | 1010 becomes 1000, then zero | 2 | Normal return without a value |
| Binary 1111 | 1111 becomes 1110, then 1100, then 1000, then zero | 4 | Normal return without a value |

Additional acceptance conditions:

- **Zero with signed promotion:** If the declared unsigned type promotes to a signed integer type that represents every argument value, subtraction at zero produces signed negative one. The stored result is nevertheless zero, and the function finishes after one body execution. Acceptance must not require an unsigned all-ones subtraction intermediate.
- **Zero with unsigned evaluation:** If the subtraction is evaluated in an unsigned type, subtracting one from zero produces that evaluation type’s maximum value. The stored result remains zero, and the function finishes after one body execution. Acceptance must not infer the intermediate width from W alone.
- **Highest bit only:** An input with only its highest representable bit set reaches zero in one body execution. Numeric magnitude does not determine the iteration count.
- **All bits set:** An input with all W value bits set requires exactly W body executions.
- **Conditional 32-bit maximum:** If the declared type has 32 value bits, input 4,294,967,295 requires 32 source-level body executions.
- **Caller storage:** If a caller supplies a stored value by value, this body does not replace that stored value with zero.
- **No count result:** An input with four set bits causes four body executions but does not return or otherwise publish the number four.
- **No observable-loop requirement:** Generated code that returns without retaining the local loop is not contradicted by the exposed behavior described here.

A verification method must distinguish the local source model from observable runtime behavior. Requiring a particular physical instruction sequence or externally visible intermediate value would add a requirement unsupported by the supplied body.

## Uncertainties

### Established static evidence

The supplied body establishes:

- One unsigned integer parameter passed by value.
- No return value.
- A post-tested local transformation.
- Clearing of one least significant set bit per nonzero transformation.
- One body execution for zero.
- A zero working value at termination.
- No dependency calls or explicit nonlocal accesses.

The stored-value behavior is established without fixing whether the subtraction is evaluated in a signed or unsigned type.

### Inferred intent

The transformation resembles the central step of a population-count algorithm. That resemblance does not establish an intended count result, a missing counter, or an omitted output.

The computation might also be intentionally expendable. Neither an optimization objective nor an application-level role is established. These possibilities are not contractual requirements.

### Documented-only unknowns

The following remain **documented_only**:

- The unsigned type’s exact definition, value width, storage size, and rank.
- The precise promoted types and intermediate arithmetic width.
- The target ABI, compilation settings, and generated instructions.
- Whether an executable retains the local repetition.
- Execution time and input-dependent timing distinguishability.
- The function’s role in a larger program.
- Caller-side conversions and surrounding behavior.

These unknowns prevent asserting a fixed-width ABI or a particular subtraction intermediate. They do not prevent specifying the local stored result, termination, or absence of explicit external effects.

## Evidence

This contract is based solely on static inspection of the supplied function body.

The available evidence consists of an unsigned scalar parameter, a post-tested loop, a local subtraction-and-bitwise transformation, a zero comparison on the updated value, and a return without a value. The supplied body contains no dependency calls, pointer dereferences, or explicit nonlocal state accesses.

The mathematical stored-value model and acceptance examples are deductions from those operations. The distinction between stored width and arithmetic evaluation width follows from the unresolved type definition and applicable integer-promotion rules.

No runtime tests, binary inspection, performance measurements, or independent checks of the missing type definition have been performed.
