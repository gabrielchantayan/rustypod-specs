<!-- Trusted provenance: F-f2b8ff34d9f0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f2b8ff34d9f0

## Purpose

F-f2b8ff34d9f0 transforms a 32-bit word using a second 32-bit word as an exponent adjustment. Its visible behavior is consistent with an operation on a binary32-style floating-point encoding, but the established contract is a transformation of integer bit patterns, not a floating-point arithmetic operation.

For an input with an exponent field between 1 and 254, inclusive, the function either:

- Adjusts the exponent while preserving the sign and fraction fields.
- Produces signed zero when the adjustment is too negative.
- Produces signed infinity when the adjustment is too positive.

Inputs whose exponent field is zero receive special treatment: zero retains its sign, while any nonzero fraction becomes positive zero. Inputs whose exponent field is 255 pass through unchanged when their fraction is zero; otherwise, execution invokes an unresolved indirect dependency.

**Evidence status:** These outcomes are derived from the supplied static representation. No runtime tests have been performed. Floating-point terminology describes the apparent encoding and inferred intent; it does not establish compliance with a floating-point standard.

## Interface

### Explicit inputs

The visible interface has four positional inputs, each occupying 32 bits:

| Position | Contract role |
|---|---|
| First | Input bit pattern to transform. |
| Second | Adjustment word. Signed comparisons interpret its bit pattern as a two’s-complement 32-bit integer. |
| Third | No explicit use in the visible transformation. Its meaning, if any, at the indirect dependency boundary is unknown. |
| Fourth | Does not affect the visible branch selection or directly computed results. For nonzero exponent inputs, the visible logic replaces its working value before using it. Its significance outside that logic is unknown. |

The first word has the following observed field layout:

| Bits | Width | Role |
|---|---:|---|
| 31 | 1 bit | Sign field |
| 30–23 | 8 bits | Exponent field |
| 22–0 | 23 bits | Fraction field |

The exponent field is therefore an unsigned integer from 0 through 255. No host floating-point interpretation is required to extract or transform these fields.

### Result

Every directly resolved path returns one 32-bit word.

For the unresolved indirect path, the visible continuation returns the word obtained from the indirect dependency, if that dependency returns normally. The supplied material does not establish that the dependency always returns.

There is no separate visible error code, status object, or out-parameter.

### Implicit execution input

One path depends on an incoming link/control value. Its low two bits are cleared to obtain an indirect call target. Thus, the destination is numerically aligned to a four-byte boundary.

This implicit value is not one of the four explicit inputs. Its origin, validity, and relationship to the surrounding calling convention are not established.

The static representation also contains an incoming overflow condition. Its initial value does not determine any directly resolved result: paths that need arithmetic overflow information establish it through their own comparisons or addition, while the special-value paths bypass the relevant decision.

### Preconditions

For directly resolved paths, the visible operations require only 32-bit input words and the represented integer arithmetic semantics. No readable or writable memory region is explicitly required.

For the indirect path, successful completion additionally depends on a valid execution target and satisfaction of that dependency’s unknown calling requirements. Those requirements are **documented_only**.

## Behavior

### Numeric interpretation and arithmetic widths

Let:

- \(W\) denote the first input word.
- \(E\) denote its eight-bit exponent field.
- \(Q\) denote its 23-bit fraction field.
- \(S\) denote its sign mask, either `0x00000000` or `0x80000000`.
- \(A\) denote the signed, two’s-complement interpretation of the second input word.

The second input retains all 32 bits. It is not narrowed to eight bits merely because the destination field is eight bits.

The visible arithmetic operates at 32-bit width. Integer multiplication and addition used to form the returned word are modulo \(2^{32}\). Signed comparison decisions account for arithmetic overflow; they must not be replaced with comparisons against the sign bit of a wrapped intermediate alone.

### Inputs with exponent zero

When \(E=0\), the fraction determines the result:

- If \(Q=0\), return \(S\). Positive zero remains positive zero, and negative zero remains negative zero.
- If \(Q\ne0\), return `0x00000000`, regardless of the original sign.

The adjustment does not affect these results. In particular, this path does not normalize or scale a nonzero fraction into a normal encoding.

The sign distinction matters: a negative input with exponent zero and a nonzero fraction produces positive zero, not negative zero.

### Inputs with exponent 255

When \(E=255\):

- If \(Q=0\), return \(W\) unchanged.
- If \(Q\ne0\), invoke the unresolved indirect dependency described below.

For zero-fraction inputs, both sign variants pass through unchanged, and the adjustment has no effect.

For nonzero-fraction inputs, the visible body does not itself define a replacement fraction, a quieting rule, a canonical result, or an error value. Any such behavior would have to come from the unresolved dependency.

### Inputs with exponent from 1 through 254

For these inputs, a directly adjusted result is returned exactly when:

\[
-E < A < 255-E.
\]

Equivalently, the mathematical adjusted exponent must satisfy:

\[
1 \le E+A \le 254.
\]

Within this range, the returned word has:

- The original sign bit.
- Exponent \(E+A\).
- The original fraction bits.

The visible result is formed by adding the adjustment word multiplied by `0x00800000` to the original word, using 32-bit modular arithmetic. Under the accepted range above, this changes the exponent field without changing the sign or fraction.

The strict inequalities are part of the contract:

- \(A=-E\) is not accepted as an ordinary adjusted result.
- \(A=255-E\) is not accepted as an ordinary adjusted result.

No rounding of the fraction occurs on this path.

### Out-of-range adjustments for exponents 1 through 254

If the adjusted exponent is outside the accepted range, the fraction is discarded.

For an adjustment at or below the lower boundary,

\[
A \le -E,
\]

the result is \(S\): signed zero retaining the original sign.

For an adjustment at or above the upper boundary,

\[
A \ge 255-E,
\]

the result is \(S\) combined with `0x7F800000`: signed infinity retaining the original sign.

These cases cover every rejected adjustment for a normal exponent. Since \(E\) lies between 1 and 254:

- Every rejected negative adjustment lies on the lower side.
- Every rejected nonnegative adjustment lies on the upper side.
- An adjustment of zero is always accepted.

The function does not produce a nonzero exponent-zero encoding when a normal input crosses the lower boundary. It also does not retain fraction bits when the upper boundary is crossed.

### Observable ordering

The visible control flow establishes the following ordering:

1. The exponent is obtained from the first input.
2. For a nonzero exponent, the ordinary-adjustment decision uses the distance to exponent 255 and, when appropriate, the adjusted exponent.
3. A successful ordinary adjustment returns before the later special-case handling.
4. Exponent-zero inputs are resolved by inspecting the fraction.
5. Exponent-255 inputs inspect the fraction before either returning unchanged or invoking the indirect dependency.
6. Remaining normal-exponent inputs return signed zero or signed infinity.

There are no explicit memory reads or writes in this sequence. Field inspections are operations on input values, not established accesses through pointers.

The representation does not establish machine-level read timing, register allocation, or an ordering contract for hidden state beyond the control dependency on the indirect target.

## Side effects

### Directly resolved paths

The visible body contains no explicit:

- Memory allocation or release.
- Pointer dereference.
- Global-variable access.
- Persistent state update.
- I/O.
- Floating-point status update.
- Error reporting.

Its directly resolved observable output is the returned word.

These observations do not establish a complete machine-state preservation contract. Register clobbers, processor status after return, and ABI-specific effects are not specified by the supplied representation.

### Indirect path

For an exponent-255 input with a nonzero fraction, control is transferred through an unresolved target after clearing the low two bits of the incoming link/control value.

The dependency may have observable effects that are not represented here. In particular, there is no basis to assert that it:

- Preserves globals or processor state.
- Avoids reading or writing memory.
- Avoids allocation or release.
- Accepts the four original explicit inputs unchanged.
- Returns normally.
- Produces a particular floating-point encoding.

If it returns normally, its returned 32-bit word becomes this function’s result without a visible subsequent transformation.

### Ownership

No explicit input is treated as a memory reference by the directly resolved logic, and no ownership transfer is visible there.

Ownership at the indirect boundary is unknown. The lack of an explicit pointer parameter does not prove that the dependency cannot access or release memory through implicit state.

## Dependencies

### F-f2b8ff34d9f0

This is the specified function. No recursive invocation is established by the supplied material.

### Unresolved indirect dependency

**Opaque function ID:** unavailable in the supplied dependency mapping.

The dependency is reached only when the first input has exponent 255 and a nonzero fraction. Its target is derived from the incoming link/control value with the low two bits cleared.

The missing identity is not replaced with an invented stable function ID. The dependency’s identity, interface, state effects, ownership rules, and failure behavior are **documented_only**.

The visible representation shows no explicit arguments at the call site. That is not sufficient evidence that the actual callee receives no arguments: register-carried values, hidden context, and calling-convention effects remain unresolved.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not observed test results. Hexadecimal values denote complete 32-bit words.

Unless an example reaches the indirect dependency, changing the third or fourth explicit input must not change the directly computed result.

### Ordinary adjustment

| First input | Second input | Required result | Reason |
|---|---|---|---|
| `0x3F800000` | `0x00000001` | `0x40000000` | Exponent 127 becomes 128. |
| `0xBF800000` | `0xFFFFFFFF` | `0xBF000000` | Adjustment is −1; sign is preserved and exponent becomes 126. |
| `0x3FC12345` | `0x00000002` | `0x40C12345` | Exponent increases by two; the fraction remains unchanged. |
| `0x00800000` | `0x00000000` | `0x00800000` | The minimum normal exponent is unchanged. |

For every input whose exponent is from 1 through 254, a zero adjustment must return the original word exactly.

### Lower boundary

| First input | Second input | Required result | Reason |
|---|---|---|---|
| `0x01000001` | `0xFFFFFFFF` | `0x00800001` | Exponent 2 becomes 1; this is still in range. |
| `0x00800001` | `0xFFFFFFFF` | `0x00000000` | The adjusted exponent reaches zero; the fraction is discarded. |
| `0x80800001` | `0xFFFFFFFF` | `0x80000000` | The same boundary preserves a negative sign. |
| `0x3F800000` | `0x80000000` | `0x00000000` | The most negative signed adjustment is below the lower boundary. |

These examples distinguish flushing to signed zero from producing a subnormal result.

### Upper boundary

| First input | Second input | Required result | Reason |
|---|---|---|---|
| `0x7E800001` | `0x00000001` | `0x7F000001` | Exponent 253 becomes 254; the fraction is preserved. |
| `0x7F000001` | `0x00000001` | `0x7F800000` | Reaching exponent 255 takes the saturation path. |
| `0xFF000001` | `0x00000001` | `0xFF800000` | Upper saturation preserves the negative sign. |
| `0x3F800000` | `0x7FFFFFFF` | `0x7F800000` | The largest positive signed adjustment is above the upper boundary. |

A conforming interpretation must not allow wrapped addition to turn the last case into a negative-adjustment outcome.

### Exponent-zero inputs

For any adjustment word:

- `0x00000000` returns `0x00000000`.
- `0x80000000` returns `0x80000000`.
- `0x00000001` returns `0x00000000`.
- `0x80000001` returns `0x00000000`.

The last case specifically requires loss of the negative sign when the fraction is nonzero.

### Exponent-255 inputs and indirect completion

For any adjustment word:

- `0x7F800000` returns unchanged.
- `0xFF800000` returns unchanged.
- `0x7FC00001` reaches the unresolved indirect dependency.

If the last case’s incoming link/control value is `0x1234567B`, the derived target is `0x12345678`. This example establishes only the address transformation, not that the destination is executable or valid.

If that dependency returns the word `0x13579BDF`, F-f2b8ff34d9f0 returns `0x13579BDF`. No stronger result condition is established.

## Uncertainties

### Inferred intent

The field layout and exponent adjustment strongly suggest a binary32 scaling helper with flush-to-zero behavior. This interpretation explains the ordinary transformation, signed saturation, and special-value handling.

It does not establish a named library API, a particular mathematical operation for all bit patterns, floating-point exception semantics, or standards compliance.

### Documented-only unknowns

The following remain unresolved:

- The identity and full contract of the indirect dependency.
- The provenance and validity of the incoming link/control value.
- The actual argument state at the indirect call.
- Hidden memory, ownership, and global-state effects on that path.
- Behavior when the indirect destination is invalid or does not return.
- Complete ABI and processor-state preservation rules.
- Whether omitted control-flow regions affect behavior outside the supplied representation.

The supplied material includes warnings that some regions were removed as unreachable and that an indirect transfer was not fully recovered. Those warnings limit confidence in completeness. They do not justify inventing additional paths or callee behavior.

## Evidence

The established contract is based solely on static inspection of the supplied restricted representation:

- An eight-bit field is extracted from bits 30–23.
- Bits 22–0 distinguish zero-fraction from nonzero-fraction cases.
- Bit 31 supplies the retained sign.
- The normal-input comparison sequence yields the strict signed bounds \(-E < A < 255-E\).
- Accepted adjustments add a multiple of `0x00800000` at 32-bit width.
- Rejected normal adjustments retain only the sign and, for the upper case, the infinity exponent mask.
- Exponent-zero nonzero-fraction inputs return positive zero.
- Exponent-255 zero-fraction inputs return unchanged.
- Exponent-255 nonzero-fraction inputs invoke an unresolved aligned target and forward its returned word.

No runtime execution, dependency inspection, or external validation has been performed. The acceptance examples are static consequences and conditional requirements, not test evidence.
