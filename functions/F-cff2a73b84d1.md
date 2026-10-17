<!-- Trusted provenance: F-cff2a73b84d1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cff2a73b84d1

## Purpose

F-cff2a73b84d1 returns an integer derived from two ordered evaluations of an externally referenced scalar.

For a valid, stable, nonnegative scalar, and provided the initial calculation and assignment complete normally, the result is:

- Zero produces one.
- One produces zero.
- Any value greater than one produces zero.

This supports an inferred zero-value-predicate intent. The complete behavior is more specific: the first observation determines a candidate result, and the second observation determines whether that candidate is replaced with zero.

The first observation is explicitly converted to the type named `uint`. That conversion does **not**, by itself, establish that the subsequent subtraction uses unsigned arithmetic. The definition of `uint` is absent. Even if it denotes an unsigned integer type, integer promotions can promote a sufficiently narrow unsigned type to signed `int` before subtraction.

The scalar’s declaration and access semantics are also absent. Consequently, the function is not unconditionally equivalent to a single Boolean test or a single physical memory read.

## Interface

### Inputs

There are no explicit parameters.

The implicit input is a scalar accessed through an externally stored reference. This contract calls the target the **observed scalar**. That phrase identifies its role, not an additional interface symbol.

The body contains two evaluations of the observed scalar:

1. An evaluation used to form the candidate result.
2. A later evaluation compared with one.

The observations may differ if the environment permits the scalar or its reference to change. The supplied material does not establish stability, volatility, atomicity, or synchronization.

Neither the reference’s storage layout nor the target’s layout is supplied. No field offsets, array indexing, or buffer lengths are established.

### Output

The declared return type is signed `int`. Its width and representation are not independently established.

Under the stable, nonnegative conditions described above, the return is exactly zero or one. Other conditions can expose a different result or prevent normal completion.

There is no separate status result, error indicator, or returned address.

### Conversion and arithmetic widths

Three types must be distinguished:

- The observed scalar’s own type.
- The type named `uint`, which is the target of the explicit conversion.
- The arithmetic type selected for subtraction after integer promotions and the usual arithmetic conversions.

These types need not have the same width or signedness.

If `uint` denotes an unsigned integer type with \(W\) value bits, conversion of an integer observation to that type produces the corresponding value modulo \(2^W\). What happens next depends on promotions:

- If that unsigned type is subject to integer promotion and all its values fit in signed `int`, the converted value promotes to signed `int`. The subtraction then uses signed `int`, because the literal one also has that type.
- If the unsigned type instead promotes to unsigned `int`, the subtraction uses unsigned `int`.
- If it is an unsigned type not changed by integer promotion, the usual arithmetic conversions determine the common type with signed `int`. For an unsigned type of at least the rank of `int`, that common type is the unsigned type.

For the common, but unproven, interpretation that `uint` is unsigned `int`, subtraction occurs modulo \(2^W\), where \(W\) is the width of unsigned `int`. Its result is then converted to signed `int` for storage. An unsigned result outside the signed range requires the implementation’s applicable conversion treatment; this contract does not select a particular returned value or assume that conversion always completes normally.

For an unsigned `uint` that promotes to signed `int`, subtraction is not modular unsigned subtraction. The converted operand is between zero and the maximum value of that unsigned type, all of which fit in signed `int`. Subtracting such a value from one is representable in signed `int`.

The definition of `uint` remains a prerequisite for choosing between these cases. Its spelling alone does not establish the arithmetic width.

### Preconditions

Normal, defined execution requires:

- A valid reference and readable target for each evaluation.
- Appropriate alignment, object lifetime, and access permissions.
- Accesses consistent with the applicable concurrency and memory model.
- Defined treatment of the conversions and arithmetic encountered.

There is no visible check enforcing these conditions.

The simpler zero-predicate interpretation additionally requires stable, nonnegative observations and normal completion of the initial calculation and assignment.

## Behavior

### Ordered observations and result formation

The established source-level sequence is:

**First observation.** The function evaluates the externally referenced scalar and explicitly converts that value to `uint`.

**Candidate formation.** After the applicable promotions and arithmetic conversions, it subtracts the converted value from one. It assigns the result to signed integer result storage, applying any required conversion to signed `int`.

**Second observation.** It evaluates the externally referenced scalar again and determines whether that second value is greater than one.

**Conditional replacement.** If the comparison is true, it replaces the candidate result with signed integer zero. Otherwise, it retains the candidate.

**Return.** It returns the resulting signed integer.

The comparison uses the second observation’s own type together with the integer literal one, under the applicable promotion and comparison rules. The first observation’s explicit conversion to `uint` does not apply to the second observation.

The initial calculation and assignment precede the conditional replacement. A later comparison that would select zero does not bypass an invalid first access, an arithmetic problem, or a conversion that prevents normal completion.

### Stable nonnegative observations

If both observations are zero, conversion preserves zero and the candidate is one. The second comparison is false, so the return is one.

If both observations are one, conversion preserves one and the candidate is zero. The second comparison is false, so the return is zero.

If both observations are the same nonnegative value greater than one, the second comparison selects zero. This final result is conditional on normal completion of the earlier calculation and assignment; the replacement is not a guard around those operations.

These outcomes do not require the first-stage arithmetic to be unsigned. They hold in the narrow-unsigned promotion case as well as in the unsigned-`int` case, subject to the stated completion condition.

### Changing observations

The two observations have different roles. The second does not cause the candidate to be recalculated.

Consequently:

- First zero, then one: return one.
- First one, then zero: return zero.
- First zero, then two: replace the candidate and return zero.
- First two, then zero or one: retain the candidate formed from two.

For a first observation of two, the retained candidate depends on the arithmetic and assignment rules. With unsigned `int` arithmetic, subtraction first produces the unsigned value congruent to negative one; conversion to signed `int` then determines the stored candidate. With a narrower unsigned type that promotes to signed `int`, subtraction directly produces signed negative one.

These are source-level distinctions. Whether a particular execution actually performs two separate physical target reads depends on the omitted access declarations and the implementation.

### Negative observations

Negative observations are possible only if the target’s actual type and value domain permit them.

If `uint` is unsigned, a negative first observation is converted to that unsigned type before subtraction. This does not establish that subtraction itself is unsigned: a narrow unsigned conversion result can subsequently promote to signed `int`.

The second comparison is independent. A genuinely negative signed second observation compares as less than one, so it does not select replacement with zero. Whether that description applies depends on the target’s actual type and the comparison conversions.

Two conditional examples show the distinction:

- If the target is signed `int` and `uint` is the corresponding unsigned `int`, a first observation of negative one converts to the unsigned maximum. Modular subtraction from one produces two. A stable second observation of negative one does not trigger replacement, so the function returns two.
- If `uint` is an unsigned type whose entire range fits in signed `int`, a first observation of negative one converts to that unsigned type’s maximum. It then promotes to signed `int`, and the candidate is one minus that maximum. With a signed negative second observation, that candidate is retained.

An unconditional Boolean result for negative observations is therefore unsupported.

### Errors and abnormal conditions

The body contains no explicit validation, diagnostic, retry, exception handling, or failure return.

Invalid references, unreadable storage, misalignment, expired object lifetime, and unsynchronized conflicting accesses have no function-specific recovery contract here. Their consequences belong to the applicable language and execution environment.

## Side effects

### Reads

There are two ordered source-level evaluations of the observed scalar. The arithmetic evaluation precedes the comparison evaluation.

The excerpt does not establish the number or width of physical memory transactions:

- Ordinary accesses may permit reuse or combination under the implementation’s rules.
- Access semantics requiring distinct observations must be respected.
- Device or read-sensitive behavior is unresolved.
- Evaluation of the external reference may itself require reading external storage; its declaration is absent.

No target-access width can be inferred solely from the conversion to `uint`.

### Writes and state changes

The body assigns to its result storage and may replace that stored candidate with zero.

No explicit write through the external reference, or to another externally visible object, appears. This is not a guarantee that reading the target has no external effect; a read-sensitive target could have effects not established by the supplied material.

### Ownership and resources

No allocation, release, acquisition, retention, or ownership transfer is visible. The function accepts no explicit resource and returns no resource handle.

Validity of the external reference remains an environmental prerequisite. The body does not establish or extend the target’s lifetime.

## Dependencies

No function calls appear in the supplied body. There are no callable dependencies to identify with additional opaque function IDs.

Execution depends on:

- The externally stored reference and the object it identifies.
- The target’s declared type and access semantics.
- The definition of `uint`.
- Integer promotions, usual arithmetic conversions, and assignment conversion.
- The implementation’s integer widths and out-of-range conversion behavior.
- Applicable concurrency and memory-access rules.

No preservation, mutation, allocation, or release behavior is attributed to unknown callees; none are visible.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

| Conditions | Required outcome |
|---|---|
| Both observations validly produce zero | Return one. |
| Both observations validly produce one | Return zero. |
| Both observations produce the same nonnegative value greater than one, and initial calculation and assignment complete normally | Return zero. |
| First observation zero; second observation one | Return one. |
| First observation one; second observation zero | Return zero. |
| First observation zero; second observation two | Return zero. |
| First observation one; second observation two | Return zero. |

Additional criteria require explicit type assumptions:

**Unsigned-`int` arithmetic with a retained candidate.** If `uint` is unsigned `int`, the first observation is two, and the second is zero, subtraction produces the unsigned value congruent to negative one. If its conversion to signed `int` completes normally, return the resulting signed value. Do not require a particular signed result without the implementation’s conversion rule.

**Narrow unsigned type promoted to signed `int`.** If `uint` is an unsigned integer type whose entire range fits in signed `int` and which promotes to that type, first observing two and then zero returns negative one. The explicit conversion to an unsigned type does not make this subtraction unsigned.

**Stable negative one with corresponding integer types.** If the target is signed `int` and `uint` is the corresponding unsigned `int`, observing negative one both times returns two.

**Stable negative one with an eight-bit unsigned conversion type.** If `uint` has the range zero through 255 and promotes to signed `int`, and the target is signed `int`, observing negative one both times returns negative 254. Conversion first produces 255; signed subtraction produces negative 254; the second observation does not trigger replacement.

**Unreadable target.** No sentinel return or graceful recovery is required by this body. There is no visible validation path.

**Read-sensitive target.** Acceptance must respect the target’s actual access semantics. Neither a guaranteed single physical read nor a universally required pair of physical reads follows from the excerpt alone.

## Uncertainties

The following remain **documented_only**:

- The external reference’s declaration, initialization, and stability.
- The target’s type, width, signedness, alignment, lifetime, and valid value domain.
- The definition, width, rank, and signedness of `uint`.
- Whether the explicit conversion result undergoes integer promotion.
- The subtraction’s effective arithmetic type when those declarations are unavailable.
- The width and representation of signed `int`.
- Treatment of unsigned-to-signed values outside the signed range.
- Volatile, atomic, device-memory, or synchronization properties.
- The number and widths of physical reads, including reference-storage reads.
- Whether the reference or target can change between observations.
- The stored value’s operational meaning and caller expectations.

The apparent zero-test intent does not resolve these uncertainties. In particular, it does not justify substituting one observation for two in every environment or treating every explicit unsigned conversion as proof of unsigned arithmetic.

## Evidence

**Established static evidence:** F-cff2a73b84d1 has no explicit parameters and returns signed `int`. It converts the first external scalar observation to `uint`, subtracts that converted value from one under the applicable arithmetic rules, and assigns the result to signed integer storage. It then evaluates the external scalar again, replaces the candidate with zero if the second value is greater than one, and returns the stored result. No function calls or explicit external writes are visible.

**Derived under stated type assumptions:** unsigned `int` gives modular unsigned subtraction; a sufficiently narrow unsigned type can promote to signed `int` and give signed subtraction. The acceptance examples distinguish these cases rather than selecting an unavailable type definition.

**Inferred intent:** for stable nonnegative observations and normal completion, the function acts as a zero-value predicate.

**Evidence limits:** external declarations, type definitions, platform details, callers, and runtime access behavior are not supplied. No runtime tests have been performed. All examples are static consequences under their stated conditions.
