<!-- Trusted provenance: F-b4f2a03d18ae; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b4f2a03d18ae

## Purpose

F-b4f2a03d18ae computes an unsigned bit-mask result from two integer inputs. Its complete observable behavior is a deterministic arithmetic return value; the supplied body contains no memory access, external call, or persistent state change.

For a second input that is a positive power of two, the result is the least nonnegative increment that would make the first input divisible by that power of two. This alignment interpretation is **inferred intent**, not an established restriction on accepted inputs.

The arithmetic is also defined for zero, negative second inputs, and positive second inputs that are not powers of two. Those cases must follow the mask calculation below rather than a generalized alignment or remainder operation.

## Interface

The function takes two integer values and returns one unsigned integer:

| Item | Recovered type interpretation | Role |
|---|---|---|
| First input, **A** | 32-bit unsigned integer | Value used in the inner bitwise mask |
| Second input, **B** | 32-bit signed integer | Supplies both the subtraction value and the mask boundary |
| Return value, **R** | 32-bit unsigned integer | Final masked arithmetic result |

The numerical contract below uses the conventional interpretation of the supplied recovered type notation: unsigned integer values are 32 bits, and signed integer values are 32 bits. The supplied material does not independently establish the target ABI or integer widths. If that type interpretation is not authoritative, the same bit-vector rule applies at the actual unsigned-integer width, but the numerical boundaries and examples involving the top bit would change.

The function receives values, not pointers. It exposes no output buffer, object handle, length field, record layout, or ownership-bearing argument. There is no memory layout requirement beyond the scalar integer types.

All bit patterns of the two input types are covered by the arithmetic under the 32-bit interpretation. In particular, the body contains no requirement that **B** be positive or a power of two.

Caller-side conversion of an argument into either parameter type is outside this function’s contract. Once the parameters have been received, their values determine the result.

## Behavior

### Exact arithmetic contract

Let:

- **W** be 2³².
- **U** be the unsigned representation of **B**, obtained by reduction modulo **W**.
- **M** be (**U** − 1) modulo **W**.
- **AND** denote bitwise conjunction over 32 bits.

The returned value is:

**R = ((U − (A AND M)) modulo W) AND M**

This equation is the behavioral definition. It is not conditional on an alignment interpretation.

Both occurrences of “second input minus one” use unsigned arithmetic because the subtraction includes an unsigned constant. The inner conjunction consequently produces an unsigned value. In the remaining subtraction, the signed second input is converted to unsigned before subtraction. Negative values therefore do not introduce signed-overflow behavior into this calculation.

Every subtraction wraps modulo 2³². No intermediate is mathematically unbounded, saturating, or clamped. The final conjunction yields a value representable by the unsigned return type without an additional narrowing step.

### Mask semantics

The mask is the unsigned predecessor of the second input’s bit pattern. It is not necessarily a contiguous sequence of low-order one bits.

Every set bit in the result must also be set in **M**. Equivalently:

**R AND M = R**

Any result bit corresponding to a zero bit in **M** is zero, regardless of the other arithmetic.

The first input affects the result only through **A AND M**. Consequently, for a fixed **B**, two first inputs produce the same result whenever their masked values are equal. Bits excluded by **M** cannot affect the result through a carry, borrow, or any other operation because they are removed before subtraction.

The second input affects both the mask and the unsigned subtraction value.

### Positive power-of-two second input

When **B** is a positive power of two, **M** consists of the low-order bits below that power of two. The result then equals the unsigned negation of **A**, restricted to those bits.

Under this condition:

- The result lies between zero and **B** − 1.
- An already aligned first input returns zero.
- Otherwise, the result is the increment to the next multiple of **B**.
- For **B** equal to one, every first input returns zero.

The function returns the increment only. It does not add that increment to the first input or return an aligned address or value.

If a caller subsequently adds the result to **A** using 32-bit unsigned arithmetic, that separate addition may wrap at 2³². This does not change the function’s returned increment.

### Zero second input

When **B** is zero, unsigned subtraction produces an all-ones mask. The final mask therefore removes no bits, and the result is the unsigned negation of **A**:

**R = (−A) modulo 2³²**

Thus a zero second input is not rejected and does not uniformly produce zero. There is no division, so this case does not create a divide-by-zero condition.

### Other second inputs

For positive non-power-of-two inputs, the exact mask rule remains controlling. It is not generally equivalent to the increment needed to reach a multiple of **B**.

For negative inputs, the function uses the unsigned representation of the signed value. No negative-input branch, absolute-value conversion, or signed remainder operation is present.

For the minimum signed 32-bit value, **U** is 2³¹ and **M** is 2³¹ − 1. The result therefore behaves as a low-31-bit unsigned negation even though the supplied second input is signed and negative.

### Evaluation and observable ordering

The supplied expression has no short-circuiting, conditional branch, or early return. The arithmetic definition applies to every invocation.

There are no observable memory reads or writes whose ordering must be preserved. The input values are already scalar parameters. Although the source expression refers to the second parameter more than once, it does not establish a meaningful external read order or require repeated reads from caller-owned storage.

The mathematical dependency order is fixed: the masked first-input value determines the subtraction result, and the final mask determines the returned bits. An implementation may use another evaluation arrangement if it preserves the same return value and introduces no additional observable effects.

## Side effects

Within the supplied body:

- No memory is allocated, freed, acquired, or transferred.
- No caller-owned storage is modified.
- No global or persistent state is read or written.
- No external function is called.
- No input/output operation, synchronization operation, or error-reporting operation occurs.
- No error status distinct from the unsigned return value is produced.

There are no arithmetic-error branches. Under the stated unsigned-width interpretation, wraparound is part of the normal result rather than an error.

The source does not establish an execution-time guarantee, instruction sequence, or constant-time property. Such properties must not be inferred merely from the absence of source-level branches.

## Dependencies

No dependency calls appear in the supplied body. No additional opaque function IDs are needed to describe its behavior.

The contract depends only on scalar integer conversion, unsigned subtraction, and bitwise conjunction. There is no callee whose memory ownership or global-state behavior must be assumed.

Platform calling convention, register placement, and binary symbol metadata are not supplied. They are not additional behavioral guarantees of this specification.

## Acceptance examples

These are **conditional acceptance examples derived statically**, not results of executed tests. All numerical values assume 32-bit integers as described above.

| A | B | Expected R | Required interpretation |
|---:|---:|---:|---|
| 0 | 8 | 0 | Zero is already aligned to a positive power of two. |
| 13 | 8 | 3 | The low three bits of 13 are five; the masked difference is three. |
| 16 | 8 | 0 | An exact multiple returns no increment. |
| 4,294,967,295 | 8 | 1 | A maximum unsigned first input still produces the normal low-bit increment. |
| 123 | 1 | 0 | A zero mask forces the result to zero. |
| 0 | 0 | 0 | Unsigned negation of zero is zero. |
| 1 | 0 | 4,294,967,295 | A zero second input produces unsigned negation, not rejection. |
| 4,294,967,295 | 0 | 1 | The same zero-input rule wraps modulo 2³². |
| 3 | 6 | 5 | The mask is five; the result is not the ordinary increment of three needed to reach six. |
| 0 | 6 | 4 | Even a zero first input need not return zero for a non-power-of-two second input. |
| 0 | −1 | 4,294,967,294 | The final mask clears the low bit of the unsigned second-input representation. |
| 1 | −2,147,483,648 | 2,147,483,647 | The mask retains the low 31 bits. |

Additional conditional properties:

- With **B** equal to eight, first inputs five and thirteen must both return three because their masked values are identical.
- For every first input, **B** equal to one must return zero.
- For any fixed **B**, changing only first-input bits excluded by **M** must not change the return value.
- For every input pair, the result must contain no set bit outside **M**.
- An implementation that rejects zero or negative **B**, or substitutes ordinary modulo-based alignment for non-power-of-two **B**, would not preserve the supplied behavior.

## Uncertainties

### Established static evidence

The supplied body establishes one arithmetic return expression, unsigned subtraction involving a one-valued unsigned constant, two bitwise conjunctions, and no external calls or explicit memory effects.

The mask calculation, wraparound behavior, and examples follow from that expression under the stated integer-width interpretation.

### Inferred intent

Power-of-two alignment padding is a plausible purpose because that interpretation exactly matches the arithmetic for positive power-of-two second inputs. No caller, documentation, or validation establishes that those are the only intended inputs.

The specification therefore does not impose such a precondition.

### documented_only

The following remain unverified:

- The target ABI and independent confirmation of 32-bit integer widths.
- Caller-side restrictions on the second input.
- Whether callers interpret the first input as an address, offset, size, or another scalar quantity.
- How callers use the returned value.
- Binary-level timing, instruction selection, and exception behavior unrelated to the specified arithmetic.
- Runtime agreement between an executable implementation and the supplied recovered body.

These unknowns do not justify adding validation, changing edge-case results, or assigning ownership semantics.

## Evidence

The sole evidence is the supplied recovered function body and scalar signature associated with F-b4f2a03d18ae. No surrounding callers, dependency implementations, executable observations, or independent ABI metadata were provided.

This contract was derived by applying expression grouping, integer-conversion rules, unsigned modular arithmetic, and bitwise-mask semantics to that material. The acceptance examples are calculated consequences of those rules.

No runtime tests have been performed.
