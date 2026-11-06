<!-- Trusted provenance: F-8d7acd353fe6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8d7acd353fe6

## Purpose

F-8d7acd353fe6 transforms a value supplied as two 32-bit words into a 64-bit result.

**Inferred numerical purpose:** conversion of a signed, two’s-complement 64-bit integer into an IEEE 754 binary64 encoding, using round-to-nearest, ties-to-even.

This interpretation is supported by the sign handling, magnitude normalization, retained precision, rounding decision, and result layout. It is not an independently established external type declaration.

**Material qualification:** some input values reach word shifts with counts of 32 or greater. The supplied representation establishes those counts but does not establish their execution semantics. The numerical conversion contract is therefore conditional for those inputs. In particular, low-word-only magnitudes require oversized logical shifts to contribute zero for the inferred conversion to hold.

This specification distinguishes the statically established operations from their numerical interpretation. No runtime tests have been performed.

## Interface

### Input representation

There are two by-value arguments, each 32 bits wide:

| Argument position | Meaning |
|---|---|
| First | Low word: input bits 31–0 |
| Second | High word: input bits 63–32 |

Let \(L\) and \(H\) denote these words as unsigned mathematical integers. The inferred signed input is

\[
I=
\begin{cases}
H\,2^{32}+L,&H<2^{31},\\
H\,2^{32}+L-2^{64},&H\geq2^{31}.
\end{cases}
\]

Bit 31 of the high word determines whether magnitude formation uses the negative-input path.

No input pair is explicitly rejected. There are no pointer, alignment, buffer-capacity, or ownership preconditions. However, the absence of validation does not resolve the execution semantics of oversized shifts.

### Result representation

The return is a 64-bit bit vector assembled from two 32-bit words. High and low describe numerical bit positions, not memory byte order.

Under the binary64 interpretation, the layout is:

| Result bits | Interpretation |
|---|---|
| 63 | Sign |
| 62–52 | Eleven-bit biased exponent |
| 51–0 | Fifty-two-bit fraction |

The external return type and calling convention are not established. A consumer interpreting the result as binary64 must preserve its bits; numerically converting the returned integer encoding would be a different operation.

### Arithmetic widths

The established arithmetic model uses 32-bit words and a 64-bit assembled result:

- Word negation, addition, subtraction, multiplication, and left-shift results retain their low 32 bits.
- Carry detection identifies overflow beyond a 32-bit unsigned sum.
- Right shifts used for bit extraction are logical.
- Leading-zero counting is performed on a nonzero 32-bit operand.
- An intermediate eight-bit extraction contributes only its least significant bit to the final rounding increment. Its other bits do not affect the returned result.

The shift-count uncertainty is separate from ordinary modulo-\(2^{32}\) word arithmetic. It must not be resolved by silently choosing a host language’s shift rules.

## Behavior

### Input interpretation and magnitude

The sign decision uses the original high word’s top bit.

For a nonnegative input, the input words already represent the unsigned magnitude. For a negative input, the operation forms the two-word negation. The adjustment across the word boundary depends on whether the original low word is zero.

Consequently, magnitude formation must distinguish:

- A negative input whose low word is zero.
- A negative input whose low word is nonzero and therefore requires the corresponding cross-word adjustment.

This is significant for values such as \(-1\) and \(-2^{32}\). Treating both words as independently negated quantities would not preserve the required magnitude.

The minimum signed input, \(-2^{63}\), does not require a positive signed 64-bit intermediate. Its unsigned magnitude, \(2^{63}\), fits in the two-word representation.

### Zero

An all-zero magnitude returns the all-zero 64-bit result.

This path does not perform leading-zero counting or normalization shifts. Its output therefore does not depend on the unresolved oversized-shift semantics.

There is no distinct negative-zero input in the inferred two’s-complement domain.

### Normalized magnitude

For a nonzero magnitude, leading-zero information comes from its high word when that word is nonzero; otherwise it comes from its low word. The selected operand is therefore nonzero.

Under zero-producing oversized logical shifts, the normalized magnitude has its highest bit at position 63. If \(M=|I|>0\) and

\[
e=\lfloor\log_2 M\rfloor,
\]

then the normalized 64-bit quantity is

\[
N=M\,2^{63-e}.
\]

This mathematical relationship describes the normalization result; it does not specify an additional externally visible intermediate.

The distinction between high-word and low-word magnitudes affects which shift counts occur:

| Magnitude class | Relevant shift-count boundary |
|---|---|
| High word nonzero | Cross-word right shift ranges from 1 through 32; lower-word left shift ranges from 0 through 31 |
| High word zero, low word’s top bit set | A lower-word left shift has count 32 |
| High word zero, low word has leading zeros | A lower-word left shift has count 33–63, and a cross-word right shift has count 225–255 |

The large right-shift counts arise because the represented count retains eight bits. They are not established as five-bit, modulo-32 counts.

For the inferred conversion, a 32-bit logical shift by at least 32 must contribute zero. Reducing such a count modulo 32 can retain bits that should have been discarded.

This uncertainty is input-dependent:

- For magnitudes from \(2^{32}\) through \(2^{63}-1\), the normalization shifts stay within 0–31. Their conversion results do not require an oversized-shift assumption.
- Low-word-only nonzero magnitudes reach oversized shifts.
- The magnitude \(2^{63}\) reaches a right shift by 32 with a zero operand. Zero-fill and modulo-32 conventions give the same contribution for that operand, but the supplied material does not establish whether the operation itself is defined in the execution environment.

### Precision and rounding

When normalization has the stated mathematical meaning, 53 significant bits are retained, including the leading bit that is implicit in binary64. The remaining eleven low bits determine rounding.

The rounding decision depends on three properties:

1. Whether the highest discarded bit is set.
2. Whether any lower discarded bit is set.
3. Whether the least significant retained bit is set.

The resulting rule is:

| Discarded amount relative to one retained unit | Result |
|---|---|
| Less than one half | Retained significand unchanged |
| Greater than one half | Retained significand incremented |
| Exactly one half, retained significand even | Unchanged |
| Exactly one half, retained significand odd | Incremented |

Here “even” means the least significant retained bit is zero.

Rounding operates on magnitude. The sign is incorporated separately. Thus negative halfway cases select the even magnitude, not an unconditional direction toward or away from zero.

Under this interpretation, integers with magnitude at most \(2^{53}\) are exactly representable. Larger integers may also be exact when their low bits satisfy the applicable binary64 spacing.

### Result assembly and carry

The retained bits occupy both result words. Assembly accounts for overflow from the low word into the high word, including overflow caused by the rounding increment.

This carry is part of the returned value. In particular, rounding an all-ones retained significand can advance the exponent to the next power of two.

Under the numerical interpretation:

- The exponent before any rounding overflow corresponds to \(e+1023\).
- The normalized leading bit becomes implicit rather than remaining an explicit fraction bit.
- Every nonzero result is finite and normal.
- No subnormal, infinity, or NaN is needed for this input domain.
- The largest positive signed input can round to \(2^{63}\), despite that value being outside the positive signed input range.

### Errors and precision reporting

There is no explicit error result, validation failure, retry, or diagnostic path.

The body does not return a separate indicator for lost precision. Inexact conversion is expressed through the rounded encoding alone.

Unresolved shift semantics are a limitation of the established execution contract, not a visible error-reporting mechanism.

## Side effects

The supplied body contains no explicit data-memory access, global-state access, allocation, release, I/O, or external function call.

The arguments are values, not storage references. Their internal transformation does not establish a write to caller-owned memory. No resource ownership is accepted, transferred, retained, or released.

There is therefore no visible ordering requirement between external memory reads and writes. The established ordering constraints are computational: the original sign is selected before magnitude handling, the magnitude determines zero handling and normalization, and the normalized bits determine rounding and assembly. These dependencies do not imply a required processor instruction schedule.

The representation does not establish post-return processor flags, scratch-register contents, or floating-point status. Absence of an explicit status update is not an ABI-level preservation guarantee.

## Dependencies

No separate callable dependency is identified. No dependency identifier other than F-8d7acd353fe6 is supplied.

The visible primitive requirements are:

- Leading-zero counting for a nonzero 32-bit word.
- Logical shifts of word operands.
- Truncated 32-bit arithmetic.
- Unsigned 32-bit carry detection.
- Assembly of two words into a 64-bit result.

These are operation requirements, not additional function identities.

The unresolved environmental dependency is the meaning of oversized word shifts. Establishing that meaning is necessary before claiming the inferred conversion for every input pair.

No assumption about an unknown callee preserving globals or releasing memory is needed: no such callee appears in the supplied body.

## Acceptance examples

These are statically derived expectations, not observed executions. Hexadecimal results show the complete 64-bit encoding, high word first.

The qualification column distinguishes ordinary-width normalization from cases requiring additional shift semantics.

| Low input word | High input word | Signed interpretation | Expected result | Qualification |
|---|---|---:|---|---|
| `0x00000000` | `0x00000000` | \(0\) | `0x0000000000000000` | Zero path; no normalization assumption |
| `0x00000001` | `0x00000000` | \(1\) | `0x3ff0000000000000` | Oversized logical shifts contribute zero |
| `0xffffffff` | `0xffffffff` | \(-1\) | `0xbff0000000000000` | Oversized logical shifts contribute zero |
| `0x80000000` | `0x00000000` | \(2^{31}\) | `0x41e0000000000000` | Oversized logical shifts contribute zero |
| `0x00000000` | `0x00000001` | \(2^{32}\) | `0x41f0000000000000` | No oversized normalization shift |
| `0x00000000` | `0xffffffff` | \(-2^{32}\) | `0xc1f0000000000000` | No oversized normalization shift |
| `0xffffffff` | `0x001fffff` | \(2^{53}-1\) | `0x433fffffffffffff` | No oversized normalization shift |
| `0x00000001` | `0x00200000` | \(2^{53}+1\) | `0x4340000000000000` | No oversized normalization shift |
| `0x00000003` | `0x00200000` | \(2^{53}+3\) | `0x4340000000000002` | No oversized normalization shift |
| `0xffffffff` | `0x7fffffff` | \(2^{63}-1\) | `0x43e0000000000000` | No oversized normalization shift |
| `0x00000000` | `0x80000000` | \(-2^{63}\) | `0xc3e0000000000000` | Shift by 32 of zero is defined and contributes zero |

The examples distinguish these behavioral requirements:

- **Negative magnitude formation:** \(-1\) and \(-2^{32}\) exercise different cross-word adjustments.
- **Exact precision boundary:** \(2^{53}-1\) remains exact.
- **Halfway rounding downward to even:** \(2^{53}+1\) becomes \(2^{53}\).
- **Halfway rounding upward to even:** \(2^{53}+3\) becomes \(2^{53}+4\).
- **Exponent-boundary carry:** \(2^{63}-1\) becomes \(2^{63}\).
- **Unsigned magnitude capacity:** the minimum signed input is exactly representable under the stated shift qualification.

A contrasting conditional example exposes the shift issue directly. If every word shift instead reduces its count modulo 32, the supplied operations for \(2^{31}\) derive `0x41e0000000100000`, not `0x41e0000000000000`. This is a static consequence of that alternative assumption, not a runtime observation.

An implementation cannot claim the full conversion contract merely by matching the rounding rule; its normalization semantics must also match.

## Uncertainties

### Established static evidence

The supplied representation establishes:

- Two 32-bit input words and a 64-bit assembled return.
- Sign selection from the original high word.
- Conditional two-word magnitude formation.
- A separate all-zero return.
- Leading-zero counting on a nonzero selected word.
- Shift counts that can exceed the operand width.
- A rounding increment with an exact-half parity distinction.
- Carry propagation from the low result word.
- No explicit memory ownership operations or external calls.

### Inferred intent

The following interpretation is strongly supported but remains inferred:

- The input words represent a signed two’s-complement integer.
- The result bits encode IEEE 754 binary64.
- The intended numerical operation is round-to-nearest, ties-to-even conversion.

For normalization paths using only ordinary-width shifts, the arithmetic supports that interpretation without the oversized-shift qualification.

### documented_only

The supplied material does not resolve:

- Execution semantics for shifts whose counts are at least 32.
- Whether an oversized shift of zero is defined.
- The external return type and calling convention.
- Memory byte order when the returned bits are stored.
- Processor-status effects absent from the representation.
- Any caller-level requirements beyond the two word values.

These points remain documented unknowns. They are not filled in by assuming a particular processor, compiler, language, or caller.

## Evidence

This contract is based solely on static inspection of the supplied representation for F-8d7acd353fe6.

The conversion interpretation is supported jointly by sign extraction, two-word magnitude handling, leading-bit normalization, 53-bit precision, eleven-bit discarded-bit analysis, binary64-shaped exponent assembly, and explicit carry handling.

The shift qualification follows from the represented eight-bit count treatment and the resulting counts of 32–63 and 225–255. Its effect is material to returned bits. Conversely, magnitudes from \(2^{32}\) through \(2^{63}-1\) avoid oversized normalization shifts, so their acceptance examples are not unnecessarily conditioned on that unresolved behavior.

No runtime execution, caller inspection, ABI documentation, or independent execution-semantics evidence was supplied. All examples are derived expectations rather than test reports.
