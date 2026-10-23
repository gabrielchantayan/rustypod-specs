<!-- Trusted provenance: F-2abbdd5d9a43; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2abbdd5d9a43

## Purpose

F-2abbdd5d9a43 classifies one integer value against a fixed, four-member set. It returns exactly **1** for values **9, 10, 13, and 32**, and exactly **0** for every other value representable by its input type.

The accepted values correspond to horizontal tab, line feed, carriage return, and space in ASCII. This correspondence supports an inferred whitespace-related purpose, but the established behavior is narrower: an exact numeric membership test. The function does not establish a general whitespace, locale, Unicode, or character-decoding contract.

In particular, vertical tab and form feed are not accepted. A consumer needing a broader whitespace definition cannot treat this function as interchangeable with a general-purpose whitespace classifier.

## Interface

**Input:** One integer passed by value.

**Output:** One integer with the same declared integer type as the input. The result is normalized to one of two values:

| Result | Meaning |
|---|---|
| 1 | The supplied integer is 9, 10, 13, or 32. |
| 0 | The supplied integer is not one of those four values. |

The supplied representation declares the interface using a signed integer type. It does not independently establish that type’s binary width, calling convention, register placement, or stack layout. A 32-bit interface may be plausible in the surrounding environment, but it is **not established by the supplied material** and is not a requirement of this contract.

All comparison constants and both possible results fit within the declared integer type. The classification does not depend on overflow, arithmetic wrapping, shifts, or the input’s byte order.

There are no pointer arguments, buffer arguments, length arguments, aggregate layouts, or optional arguments. The function does not consume a string or inspect neighboring characters.

**Precondition:** The caller supplies a valid value through the actual integer interface. No additional restriction to nonnegative values, byte-sized values, or encoded characters is visible.

If the caller starts with a wider integer or another type, any conversion performed before entry is outside this function’s established behavior. Classification applies to the integer value actually received, not necessarily the caller’s original value.

## Behavior

### Classification rule

The result is 1 if and only if the received integer equals at least one member of this set:

**{9, 10, 13, 32}**

The result is 0 otherwise.

Each comparison concerns the complete received integer value. Nothing in the supplied body masks the input to eight bits, extracts a character from a larger value, or converts an encoded character into a numeric code point.

Consequently, a representable integer whose low byte happens to equal an accepted value is still rejected unless the entire integer equals that accepted value. For example, 288 is rejected even though its low eight bits equal 32.

### Coverage and normalization

Every valid input belongs to exactly one of two classes: membership or nonmembership in the four-value set. The conditions shown in the supplied body are complementary, so every input receives a normalized result.

There is no input for which the function intentionally returns the original integer merely because no condition matched. In particular:

- Input 0 returns 0.
- Input 1 returns 0.
- Each accepted value returns 1 rather than returning itself.
- A negative input returns 0.
- Any representable positive input outside the accepted set returns 0.

This is an exact Boolean-valued integer contract. Returning some other nonzero value for acceptance would not satisfy it.

### Ordering and observability

The supplied body establishes its classification facts from the original received value before replacing the function-local input value with the normalized result. Later decisions therefore do not reinterpret a normalized 1 as though it had been the original input.

The source-level checks are presented in the order 32, 9, 13, and 10. This order introduces no externally observable read or write requirement: the comparisons use only the passed integer and have no visible side effects. The established result does not depend on a particular machine-instruction ordering.

The only caller-visible value produced by the body is the returned integer. There is no partial output, intermediate classification report, or externally visible transition between acceptance and rejection.

## Side effects

The supplied body contains no explicit access to caller-owned memory, global state, files, devices, environment variables, or locale state.

Its writes are confined to function-local classification state and the function-local value used to form the return result. Changing that local value does not change the caller’s original integer object.

No allocation, deallocation, ownership transfer, resource acquisition, or resource release is present. The caller retains ownership of any object from which it obtained the argument.

The body provides no error-reporting channel distinct from its normal result. A return value of 0 means nonmembership in the accepted set, not a reported execution failure. There is no visible exception path, retry, diagnostic message, or sentinel error result.

These statements describe the supplied function body. They do not establish guarantees about external instrumentation, invocation machinery, or behavior caused by violating the actual calling convention.

## Dependencies

**No dependency calls are present in the supplied body.**

The function’s visible result is determined by integer equality or inequality, Boolean combination, local assignment, and return. No additional opaque function IDs are needed to describe its behavior.

There is no visible dependency on:

- A character-classification library.
- A current locale.
- An encoding table.
- A decoder.
- An allocator.
- Shared mutable classification state.

Because no callee appears, this contract does not assume that an unknown callee preserves globals or manages memory.

## Acceptance examples

The following are static contract examples, not reported test results. Each assumes that the listed value is received unchanged through the actual integer interface.

| Received input | Required result | Contract reason |
|---|---:|---|
| 9 | 1 | Exact accepted value; ASCII horizontal tab correspondence. |
| 10 | 1 | Exact accepted value; ASCII line feed correspondence. |
| 13 | 1 | Exact accepted value; ASCII carriage return correspondence. |
| 32 | 1 | Exact accepted value; ASCII space correspondence. |
| 0 | 0 | Not a member of the accepted set. |
| 1 | 0 | A possible result value is not thereby an accepted input. |
| 8 | 0 | Adjacent to 9, but not accepted. |
| 11 | 0 | Vertical tab is outside the fixed set. |
| 12 | 0 | Form feed is outside the fixed set. |
| 14 | 0 | Adjacent to 13, but not accepted. |
| 31 | 0 | Adjacent to 32, but not accepted. |
| 33 | 0 | Adjacent to 32, but not accepted. |
| −1 | 0 | Negative values cannot equal any accepted value. |
| 288 | 0 | The full integer is compared; low-byte equivalence is insufficient. |

Additional conditional acceptance requirements:

- **If the argument represents a character in a non-ASCII encoding,** the same numeric rule applies. Character meaning does not expand or alter the accepted set.
- **If a locale treats additional characters as whitespace,** those values still return 0 unless numerically equal to one of the four accepted values.
- **If a caller converts a larger value before entry,** the required result follows the converted value received by this function. The original wider value is not available to the function.
- **If the minimum or maximum representable input integer is supplied,** the result is 0 unless that value equals an accepted constant. No arithmetic-boundary operation occurs.

## Uncertainties

### Established static behavior

The supplied body establishes the four accepted integer values, exact 0-or-1 results, complete classification of the input domain, and absence of explicit external calls or memory accesses.

### Inferred intent

The accepted constants suggest recognition of selected ASCII whitespace characters. This is an interpretation of the values, not evidence that the function accepts all whitespace or requires an ASCII-encoded input stream.

### documented_only unknowns

The supplied material does not establish:

- The integer type’s exact bit width or binary ABI.
- How callers obtain, validate, decode, or convert arguments.
- Whether callers assign special meaning to negative values.
- The larger parser, protocol, or application purpose.
- Machine-level instruction order or timing characteristics.

None of these unknowns is needed to determine the numeric result once a valid integer has entered the function.

## Evidence

This contract is based exclusively on static inspection of the supplied function body. That body compares the received integer with four constants, uses complementary classification conditions, assigns either 1 or 0, and returns the resulting integer. It contains no calls or explicit accesses to external storage.

**No runtime tests have been performed.** The acceptance examples express required outcomes derived from the visible classification rule; they are not claims of executed verification.
