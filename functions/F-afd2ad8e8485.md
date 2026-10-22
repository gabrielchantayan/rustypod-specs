<!-- Trusted provenance: F-afd2ad8e8485; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-afd2ad8e8485

## Purpose

F-afd2ad8e8485 classifies two unsigned integer inputs independently against the same threshold, 28, and returns a four-byte value encoding the two classification results.

The first input controls the contribution `0x1000`; the second input controls the contribution `0x0800`. An input contributes its associated value only when it is strictly less than 28. Equality with 28 does not qualify.

The established result set is exactly:

- `0x00001800`: both inputs are below 28.
- `0x00001000`: only the first input is below 28.
- `0x00000800`: only the second input is below 28.
- `0x00000000`: neither input is below 28.

The interpretation of these values as independent flags is supported by their bit pattern. Their application-specific meanings are unknown.

## Interface

### Inputs

The function accepts two positional unsigned integer values:

| Position | Meaning in this contract | Classification |
|---|---|---|
| First | First unsigned input | Qualifies when strictly less than 28 |
| Second | Second unsigned input | Qualifies when strictly less than 28 |

The supplied representation identifies both inputs as unsigned integers. This contract describes their behavior over the 32-bit unsigned domain, from 0 through 4,294,967,295 inclusive. The target’s actual calling convention and input storage width are not independently established; the width qualification is recorded under **Uncertainties**.

The inputs are values, not identified pointers, references, buffers, handles, or ownership-bearing objects. No relationship between the inputs is required. They may be equal or different, and either may be above or below the threshold independently.

There is no established requirement that the inputs represent coordinates, indices, dimensions, or any other application-specific quantity.

### Output

The function returns a four-byte result. Every established result is a nonnegative value representable in 32 bits. Its signed interpretation, if any, does not change the numerical interpretation of the four possible results.

The output has the following bit-level meaning:

| Result bit | Mask | Set exactly when |
|---|---|---|
| 12 | `0x00001000` | First input is below 28 |
| 11 | `0x00000800` | Second input is below 28 |
| All other bits | All remaining positions | Never set by the supplied behavior |

Bit numbering starts at zero for the least significant bit.

The result is a returned value, not a serialized byte sequence. No byte order or external storage layout is established.

### Preconditions and errors

No additional precondition is visible for values within the unsigned input domain. In particular, values greater than or equal to 28 are accepted and classified; they are not rejected.

There is no separate error return, exception path, diagnostic output, or status update in the supplied behavior. Zero is an ordinary classification result, not an established failure indication.

## Behavior

### Classification rule

The complete input-to-output relationship is:

| First input | Second input | Required return |
|---|---|---|
| Less than 28 | Less than 28 | `0x00001800` |
| Less than 28 | At least 28 | `0x00001000` |
| At least 28 | Less than 28 | `0x00000800` |
| At least 28 | At least 28 | `0x00000000` |

The threshold is strict. The values 0 through 27 qualify; 28 and all larger unsigned values do not.

Within either classification range, the exact magnitude does not affect the result. For example, replacing a qualifying input of 0 with 27 leaves its contribution unchanged. Replacing a nonqualifying input of 28 with the largest representable unsigned value also leaves its contribution unchanged.

The positions are not interchangeable. Exchanging the inputs exchanges which contribution is present when exactly one input qualifies. When both inputs qualify, or neither qualifies, exchanging them leaves the result unchanged.

### Evaluation order

The supplied control flow first classifies the first input. It then classifies the second input within the selected branch and returns the corresponding constant.

Every visible path therefore includes classification of both inputs. There is no path that returns a result based solely on the first input.

These comparisons use values already supplied to the function. The representation does not establish externally observable reads from caller memory during classification. In particular, the order in which a caller obtains or computes the argument values is outside this function’s contract.

### Arithmetic and boundaries

The supplied behavior performs comparisons and selects a constant. It does not add, subtract, multiply, shift, or otherwise transform either input.

Consequently:

- No input-dependent arithmetic overflow or underflow is present.
- Large unsigned values are compared against 28 without being reduced to a smaller range.
- There is no visible masking, saturation, normalization, or clamping of either input.
- The result remains one of the four listed constants for every accepted input pair.

The independent-bit description specifies the result relationship; it does not require an implementation to perform a bitwise combination operation.

Negative mathematical values are outside the unsigned input domain. If a caller supplies a value originating from a signed negative integer, the behavior of its conversion and argument passing belongs to the surrounding language and ABI, not to this function. Once an unsigned argument value has been established, that value is classified normally.

## Side effects

No externally visible state mutation appears in the supplied body.

Specifically, it contains no identified:

- Reads or writes of global state.
- Dereferences of input-derived addresses.
- Writes through caller-provided pointers.
- Allocation or release of memory.
- Resource acquisition, ownership transfer, or resource release.
- Input/output operations.
- Calls to another function.

The only established output is the returned four-byte value. No caller-owned object is modified by the visible behavior.

For a fixed pair of input values, the supplied body selects the same result regardless of prior calls. No persistent state or cross-call dependency is visible.

These statements concern the supplied function body. They do not establish properties of surrounding instrumentation, calling code, or execution machinery.

## Dependencies

No callee dependency is visible. F-afd2ad8e8485 completes its classification without invoking another opaque function.

There is therefore no unknown callee whose behavior must be assumed to preserve global state, retain arguments, or release memory.

Environmental details such as the exact calling convention, register assignments, stack layout, and machine representation of argument passing are not supplied. They are not necessary to describe the established classification behavior, but they would be necessary to specify a binary-compatible interface fully.

## Acceptance examples

These examples are conditional acceptance criteria derived from the supplied static representation. They are not runtime observations.

| First input | Second input | Required result | Criterion |
|---:|---:|---:|---|
| 0 | 0 | `0x00001800` | Zero qualifies in both positions |
| 27 | 27 | `0x00001800` | Both inputs immediately below the threshold qualify |
| 27 | 28 | `0x00001000` | Equality excludes the second contribution |
| 28 | 27 | `0x00000800` | Equality excludes the first contribution |
| 28 | 28 | `0x00000000` | Neither equality case qualifies |
| 0 | 29 | `0x00001000` | Only the first input qualifies |
| 29 | 0 | `0x00000800` | Only the second input qualifies |
| 29 | 29 | `0x00000000` | Values above the threshold do not qualify |
| 4,294,967,295 | 0 | `0x00000800` | Under the 32-bit input model, the largest first input does not qualify |
| 0 | 4,294,967,295 | `0x00001000` | Under the 32-bit input model, the largest second input does not qualify |
| 4,294,967,295 | 4,294,967,295 | `0x00000000` | Neither maximum-width input qualifies |

Additional relational criteria:

- Changing one input while keeping it below 28 must not change the result.
- Changing one input while keeping it at least 28 must not change the result.
- Moving the first input from 27 to 28 must clear only `0x1000`.
- Moving the second input from 27 to 28 must clear only `0x0800`.
- No accepted input pair may produce a result outside the four-value result set.

## Uncertainties

### Established static evidence

The supplied representation directly supports two unsigned comparisons against 28, the branch order, the four return constants, and a four-byte result. It also shows no calls or external state access.

### Inferred intent

The output appears to encode two independent predicates as flags. This interpretation explains the result layout, but no application-level meaning for either flag is established.

### documented_only

The following remain unspecified rather than assumed:

- Independent confirmation that the input type occupies exactly 32 bits on the original target. The behavioral table remains valid for any unsigned input width capable of representing 28.
- Exact binary calling convention and argument placement.
- Whether downstream consumers treat the result as flags, an enumeration, or another category of value.
- The semantic meaning or units of either input.
- Any caller-side validation or conversion before invocation.
- Runtime behavior of an executable implementation, including timing.

## Evidence

This contract is based solely on the supplied static function representation.

That representation exposes a complete four-way classification: first-input comparison, second-input comparison, and selection of one of four constants. It contains no visible dependency calls, pointer access, state mutation, or input arithmetic.

No runtime tests, executable inspection, or independent ABI validation have been performed. The acceptance examples are deductions from static evidence, not reported test results.
