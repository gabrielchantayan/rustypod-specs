<!-- Trusted provenance: F-c0b2b2741cab; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c0b2b2741cab

## Purpose

F-c0b2b2741cab combines a scaled product of its first two inputs with its third input, selects a value according to two signed thresholds, and returns that selection after conversion to a signed 16-bit value.

The threshold interval is −32768 through 32767, inclusive. Within that interval, the calculated value is selected directly. Above the interval, a value from one external storage location is selected. Below the interval, a value from another external storage location is selected.

This resembles a fixed-point multiply-and-add with saturation, but **saturation to the interval endpoints is not established**. The external values are not supplied. The observable contract is therefore threshold-based selection followed by signed narrowing, not an unconditional clamp to −32768 or 32767.

## Interface

The function accepts three integer inputs:

| Input | Role |
|---|---|
| First input | One multiplicand |
| Second input | The other multiplicand |
| Third input | Additive base, combined with the scaled product |

It returns an integer representing the selected value after conversion to a signed short.

There are no explicit pointer, buffer, length, callback, or ownership parameters. No output parameter is present.

### Widths and representation

The source-facing interpretation is a 32-bit signed integer interface with a 16-bit signed intermediate return conversion. Under that interpretation:

- Each input is a signed 32-bit integer.
- Multiplication occurs at signed integer width; there is no visible widening to a 64-bit product.
- The product is shifted right by 15 bit positions.
- The shifted result is added to the third input at signed integer width.
- The selected value is narrowed to signed 16 bits.
- That signed 16-bit value is widened to the integer return type, preserving its sign.

The supplied representation uses integer and short types but does not independently establish their machine widths or all target-specific conversion rules. Consequently, the 32-bit/16-bit model is a stated interpretation, not evidence of a complete platform ABI.

In particular, the final conversion must not be mistaken for returning an unsigned 16-bit quantity. Under the stated model, a narrowed negative value returns as a negative integer.

### Implicit inputs

Two external integer-valued storage locations participate:

- **Upper-selection storage:** read regardless of the calculated value.
- **Lower-selection storage:** read only when the calculated value is strictly less than −32768.

These names describe their selection roles, not their contents. Neither location is established to contain a threshold endpoint, a constant, or a value already representable as signed 16 bits.

No external function dependencies are visible.

## Behavior

### Arithmetic transformation

The first two inputs are multiplied before scaling. The product is then shifted right by 15 positions, and the shifted result is added to the third input.

The operation order is significant:

- Scaling applies to the product, not separately to either multiplicand.
- The third input is added after the shift.
- Threshold selection occurs after the addition.
- Narrowing to signed 16 bits occurs after threshold selection.

The visible expression does not provide a wider multiplication temporary. A mathematical product that fits in 64 bits but not in the working signed integer type is therefore not automatically a valid input case.

For nonnegative, representable products, shifting right by 15 discards the low 15 bits. For negative products, the exact rounding behavior depends on the signed right-shift semantics. If the target uses arithmetic right shift, the result corresponds to division by 32768 rounded toward negative infinity. It is not generally equivalent to signed division rounded toward zero.

For example, under arithmetic right shift, a product of −1 contributes −1 after scaling, rather than zero. This distinction can change both the returned value and which threshold branch is selected.

### Threshold selection

Call the result of the arithmetic transformation the **combined value**. Selection uses signed comparisons:

| Combined value | Selected value |
|---|---|
| Less than −32768 | Value read from lower-selection storage |
| Exactly −32768 | Combined value |
| Greater than −32768 and less than 32768 | Combined value |
| Exactly 32768 | Value read from upper-selection storage |
| Greater than 32768 | Value read from upper-selection storage |

Thus −32768 is included in the directly returned interval, whereas 32768 is excluded. The greatest directly selected value is 32767.

The two external selections are asymmetric in their access behavior. Upper-selection storage is read before the threshold decision even when its value will not be selected. Lower-selection storage is read only for the below-range case.

### Return conversion

After selection, the selected value is converted to signed short and then to the integer return type.

For a selected value already within −32768 through 32767 under the 16-bit model, conversion preserves its numerical value.

For an external selection outside that interval, the result depends on the target’s out-of-range conversion rules. The visible operation is a narrowing conversion, not another comparison or clamp. A contract cannot substitute an endpoint for such a value without additional platform or storage-content evidence.

Where the target implements signed narrowing through low-16-bit retention and two’s-complement interpretation, an externally selected value of 65535 would return −1. That is a conditional platform example, not a platform-independent guarantee.

### Preconditions and failures

There is no explicit input validation, error code, exception path, or retry.

A defined source-level execution requires:

- The multiplication to be valid under the applicable signed arithmetic rules.
- The addition of the shifted product and third input to be valid under those rules.
- Upper-selection storage to be readable.
- Lower-selection storage to be readable if the combined value is below −32768.
- The target’s signed shift and narrowing semantics to be known when a case depends on them.

Under ordinary source-language signed arithmetic rules, multiplication or addition overflow is not established to wrap. The later threshold checks do not make an earlier overflow safe.

If the representation instead describes machine arithmetic that wraps at 32 bits, that would be a separate execution model requiring confirmation. It must not be silently substituted for the source-level model.

An inaccessible upper-selection location can prevent completion even for an otherwise in-range result, because its read is unconditional. An inaccessible lower-selection location matters only on the below-range path. No recovery behavior for either failure is visible.

## Side effects

The visible external effects are reads, in this order:

1. After computing the combined value, read upper-selection storage once.
2. If the combined value is less than −32768, read lower-selection storage once.
3. Return the converted selected value.

The first read remains part of the shown behavior even when its value is subsequently displaced by the combined value or by the lower external selection.

No external writes, allocation, deallocation, object construction, ownership transfer, or resource release are visible. Input values are passed by value; the arithmetic does not modify caller-owned input storage.

The representation does not establish that either external location is volatile, atomic, memory-mapped, immutable, or protected against concurrent modification. It also does not establish compiler-level or hardware-level ordering guarantees beyond the access sequence shown.

If reading these locations itself has externally observable effects, the unconditional upper read is material. For example, the in-range path must not be described as having no external access merely because its return value is independent of the stored upper-selection value.

## Dependencies

No call to another function is visible. There are therefore no opaque callee IDs to list beyond the subject, F-c0b2b2741cab.

The function depends on:

- The two external storage values and their availability.
- The integer widths used by the execution environment.
- Signed multiplication and addition semantics.
- Signed right-shift semantics for negative products.
- Signed-short narrowing semantics for values outside the short range.
- The signed extension behavior when converting the narrowed value to the return type.

No initialization mechanism, writer, synchronization scheme, or ownership rule for the external storage is supplied. Their values must not be assumed stable across invocations.

## Acceptance examples

These are static, conditional acceptance cases, not reports of executed tests. Unless otherwise stated, they assume 32-bit signed inputs, 16-bit signed short, readable external storage, and no arithmetic overflow.

### Directly selected values

| Inputs: first, second, third | Expected result | External reads |
|---|---:|---|
| 0, 123, 0 | 0 | Upper only |
| 0, 123, −32768 | −32768 | Upper only |
| 0, 123, 32767 | 32767 | Upper only |
| 32768, 1, 10 | 11 | Upper only |
| 32767, 1, 10 | 10 | Upper only |

The last two cases distinguish scaling the product from adding an unscaled multiplication result. They also show that positive products below 32768 contribute zero after the shift.

### Boundary selection

If upper-selection storage contains 123 and lower-selection storage contains −456:

| Inputs: first, second, third | Expected result | External reads |
|---|---:|---|
| 0, 0, 32768 | 123 | Upper only |
| 0, 0, −32769 | −456 | Upper, then lower |
| 0, 0, −32768 | −32768 | Upper only |

These cases intentionally use external values that are not threshold endpoints. They establish that threshold crossing selects storage contents rather than inherently returning a saturated endpoint.

If the two locations instead contain 32767 and −32768 respectively, the boundary cases behave like signed 16-bit saturation. That behavior is conditional on those contents.

### Negative scaling

With arithmetic signed right shift:

- Inputs −1, 1, 0 produce −1.
- Inputs −32768, 1, 0 produce −1.
- Inputs −1, 1, −32768 produce a combined value of −32769 and select lower-selection storage.

The first and third cases depend on negative-shift semantics. They are not unconditional requirements for an unspecified execution environment.

### Out-of-range stored selections

If upper-selection storage contains 65535 and inputs are 0, 0, 32768, the upper value is selected and then narrowed.

- Under low-16-bit, two’s-complement narrowing, the return is −1.
- Without that conversion rule, the exact result remains documented_only.

The function does not visibly replace this stored value with 32767.

### Overflow-sensitive inputs

Inputs 65536, 65536, 0 have a mathematical product outside signed 32-bit range. The later shift would make the mathematical scaled product smaller, but that does not establish validity of the preceding multiplication.

No unconditional return value is specified for this case under source-level signed arithmetic semantics.

## Uncertainties

The following remain **documented_only**:

- Exact machine widths and ABI details beyond the stated 32-bit/16-bit interpretation.
- Whether the representation should be governed by source-language overflow rules or confirmed machine-level wrapping behavior.
- Signed right-shift behavior for negative products.
- Conversion behavior when a selected external value is outside signed-short range.
- The contents, initialization, mutability, and intended meaning of both external storage locations.
- Whether either storage access has device, volatile, atomic, or concurrency semantics.
- The external mechanism of failure if a required read cannot complete.

**Inferred intent:** the scale factor and thresholds are consistent with fixed-point arithmetic feeding a signed 16-bit result, potentially with saturation supplied by stored endpoint values.

**Not established:** a particular fixed-point format for the inputs, guaranteed saturation, protection from overflow, purity, thread safety, or stable configuration between calls.

## Evidence

The supplied static representation establishes:

- Three integer inputs and an integer return.
- Multiplication followed by a right shift of 15, followed by addition.
- An unconditional read from upper-selection storage after the arithmetic.
- Signed threshold comparisons at 32768 and −32768.
- A conditional lower-selection read only below −32768.
- Direct selection of combined values in the inclusive interval −32768 through 32767.
- A signed-short conversion before the integer return.
- No visible function calls or external writes.

No runtime tests have been performed. The acceptance examples are deductions from the supplied representation under their stated assumptions, not observed execution results.
