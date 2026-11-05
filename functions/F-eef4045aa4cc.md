<!-- Trusted provenance: F-eef4045aa4cc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eef4045aa4cc

## Purpose

F-eef4045aa4cc maps two unsigned 32-bit input words to a 64-bit result bit pattern.

**Inferred intent:** encode the unsigned 64-bit integer represented by the inputs as IEEE 754 binary64, rounding to nearest with ties to even. The first input supplies the least significant 32 bits; the second supplies the most significant 32 bits.

**Established static evidence:** the operation uses integer normalization, discarded-bit inspection, unsigned addition with carry, and assembly of two result words. No floating-point operation is visible.

The conversion interpretation is conditional. Reachable logical shifts have counts equal to or greater than the 32-bit operand width, and the supplied material does not establish their execution semantics. The numerical contract below applies when such shifts produce zero. Other shift rules must not be treated as equivalent.

## Interface

### Inputs

| Argument | Width | Meaning |
|---|---:|---|
| First word | 32 bits | Unsigned low-order portion |
| Second word | 32 bits | Unsigned high-order portion |

The represented mathematical integer is

\[
N=(\text{second word})\,2^{32}+\text{first word}.
\]

Its range is \(0\) through \(2^{64}-1\), inclusive. Every combination of input bits is admitted by the visible control flow. Neither argument is a pointer or an ownership-bearing reference.

This argument order is part of the contract. Exchanging the words generally changes the represented integer.

### Result

The result is a 64-bit bit pattern formed from a high 32-bit word and a low 32-bit word. The high word occupies result bits 63–32; the low word occupies bits 31–0.

This layout does not establish:

- Memory byte order.
- Register placement.
- A floating-point return register convention.
- Whether a caller’s declared return type is an integer, floating-point value, or another 64-bit representation.

Under the conditional conversion interpretation, the result bits encode a binary64 value. Numerically converting the returned bit pattern as an integer is not equivalent to interpreting it as binary64.

### Arithmetic requirements

The visible word arithmetic is unsigned and 32 bits wide. Ordinary word results retain their low 32 bits; an explicitly detected carry records overflow beyond that width. Byte narrowing retains eight bits.

These width rules matter for normalization positions below zero: they are represented through unsigned wraparound, not rejected, saturated, or treated as large positive mathematical positions.

Logical right shifts require unsigned behavior. Counts reduced to eight bits are not thereby reduced modulo 32.

## Behavior

### Zero

When both inputs are zero, the result is exactly `0000000000000000`.

This result follows from the explicit zero path and does not depend on oversized-shift semantics. No leading-zero count on zero is required.

Interpreted as binary64, the result is positive zero.

### Nonzero normalization

The most significant nonzero input word determines the normalization basis:

- A nonzero second word takes precedence, regardless of the first word.
- Otherwise, the first word supplies the basis.

The leading-zero count is therefore applied only to a nonzero 32-bit word and lies between 0 and 31.

For describing the alignment, let \(k\) be the bit length of \(N\): the position of its highest set bit, counting positions from one. Thus \(1\leq k\leq64\). The normalization position relative to the boundary between the input words is \(k-32\). Negative positions are represented modulo \(2^{32}\) in the word arithmetic.

The visible alignment has two word-sized portions. The upper portion combines the normalized leading input word with a right-shifted contribution from the original first word. The lower portion is a left-shifted contribution from that same original first word. The upper combination is bitwise OR, not addition.

The first-word contributions use these effective counts after the visible eight-bit reductions:

| Input class | Right-shift count | Left-shift count |
|---|---:|---:|
| Second word nonzero | 1–32 | 0–31 |
| Second word zero; first word has its top bit set | 0 | 32 |
| Second word zero; leading-zero count is 1–31 | 225–255 | 33–63 |

Both contributions depend on the original first input, not on one another’s shifted result.

**Conditional interpretation:** if a logical shift of a 32-bit operand by 32 or more produces zero, the two aligned portions together represent

\[
N\,2^{64-k}.
\]

This is exact left normalization: the highest set bit occupies bit 63, and no significant input bits are lost.

The eight-bit count reductions alone do not establish this interpretation. In particular, a machine that masks counts to five bits can produce different aligned contents.

### Precision and rounding

The visible retained-bit layout corresponds to keeping the upper 53 bits of the aligned 64-bit quantity and discarding the lower 11 bits.

Within the lower aligned word:

| Bit position | Role |
|---|---|
| 0–9 | Lower discarded bits |
| 10 | Guard bit: the highest discarded bit |
| 11 | Least significant retained bit |

The rounding increment is either zero or one:

- A clear guard bit produces no increment.
- A set guard bit with any nonzero lower discarded bit produces an increment.
- A set guard bit with all lower discarded bits clear produces an increment only when the least significant retained bit is one.

Thus the rounding decision distinguishes below-halfway, above-halfway, and exact-halfway cases. At an exact tie, the retained result becomes even.

Only a single bit contributes the increment. Although an intermediate value is narrowed to a byte, its other bits do not increase the rounding amount.

The low result word contains the retained portion of the lower aligned word and the lowest eleven bits of the upper aligned word. These occupy disjoint bit ranges before rounding. Their combination and the rounding increment use 32-bit arithmetic. Overflow is propagated to the high result word rather than discarded.

The high result word incorporates the remaining retained significant bits, the normalization-dependent exponent contribution, and the carry from the low result word. Its construction is additive: a significand carry can advance the exponent. The leading normalized bit participates in this construction; it must not be counted again as a separately stored fraction bit.

### Conditional numerical contract

Assuming zero-producing oversized logical shifts and the stated word-width and carry semantics, the returned pattern is the binary64 representation of \(N\), rounded to nearest with ties to even.

For nonzero input, the exponent before a possible rounding carry corresponds to the highest set-bit position of \(N\). The stored biased exponent is therefore \(1022+k\). Rounding can increase it by one.

Consequences include:

- All integers from zero through \(2^{53}\) are represented exactly.
- Above \(2^{53}\), exactness depends on the integer’s low bits.
- A nonrepresentable integer selects its nearest binary64 neighbor.
- An exact midpoint selects the neighbor with an even retained significand.
- Rounding across a power-of-two boundary increases the exponent.
- The maximum input, \(2^{64}-1\), rounds to \(2^{64}\).
- Every nonzero result is finite, positive, and normal.
- No input requires a negative result, subnormal result, infinity, or NaN.

For a nonzero integer in the interval from \(2^e\) through values below \(2^{e+1}\), binary64 spacing is \(2^{e-52}\). Where that spacing exceeds one, it determines the integer granularity and halfway cases relevant to this conversion.

The rounding decision is expressed entirely through integer bits. No floating-point rounding-mode setting is visibly consulted.

## Side effects

### Observable access and ordering

The visible operation consumes scalar arguments and returns a scalar bit pattern. It contains no explicit:

- Memory dereference.
- Global-state access.
- Allocation or release.
- Input/output operation.
- Call to another identified function.

There are therefore no visible memory reads or writes whose relative order a caller must preserve.

The high input determines the normalization basis. Zero detection prevents normalization of an all-zero input. For nonzero input, rounding depends on the aligned bits, and the final high word depends on carry from the low-word calculation. These are data dependencies, not guarantees about machine instruction scheduling.

No timing or constant-time guarantee is established.

### Ownership and persistent state

Neither input conveys ownership, and the result does not identify an allocated object. No ownership transfer, release obligation, or persistent state update appears in the supplied operation.

This conclusion is limited to the visible function representation. It does not assert properties of an unspecified wrapper or caller.

### Errors

There is no explicit error result, validation failure, exception branch, or recovery path.

Zero is a valid input, not an error sentinel. All nonzero input patterns enter the same general normalization and rounding behavior.

Unresolved oversized-shift behavior is an execution-semantics prerequisite, not a recoverable input error handled by this function. A replacement implementation must define those shifts deliberately rather than relying on a host language’s potentially different rules.

## Dependencies

No external dependency with an opaque function ID is shown. There is no identified callee whose ownership or global-state behavior must be assumed.

The visible primitive requirements are:

| Primitive | Required property |
|---|---|
| Leading-zero count | Counts leading zeros in a nonzero 32-bit word |
| Logical word shifts | Operate on unsigned 32-bit operands; oversized-count behavior remains unresolved |
| Unsigned carry detection | Detects overflow from 32-bit addition |
| Word assembly | Places the high and low result words in their specified 64-bit positions |

These are primitive semantic requirements, not evidence of external runtime calls.

The leading-zero count on the selected word appears more than once in the supplied representation. Nothing establishes separate observable side effects for those occurrences; the contract requires the ordinary deterministic meaning of that primitive.

## Acceptance examples

Only the zero case below is unconditional on the unresolved shift rule. All nonzero cases are **conditional mathematical expectations under the numerical contract**, not observed execution results.

Hexadecimal entries denote complete word or result bit patterns.

| Second word | First word | Integer | Expected result bits | Acceptance property |
|---|---|---:|---|---|
| `00000000` | `00000000` | \(0\) | `0000000000000000` | Explicit zero path |
| `00000000` | `00000001` | \(1\) | `3FF0000000000000` | Smallest positive input |
| `00000000` | `80000000` | \(2^{31}\) | `41E0000000000000` | Alignment includes a shift count of 32 |
| `00000001` | `00000000` | \(2^{32}\) | `41F0000000000000` | Second word supplies the leading bit |
| `00200000` | `00000000` | \(2^{53}\) | `4340000000000000` | Exact precision boundary |
| `00200000` | `00000001` | \(2^{53}+1\) | `4340000000000000` | Exact tie selects the lower even significand |
| `00200000` | `00000002` | \(2^{53}+2\) | `4340000000000001` | Exactly representable above the boundary |
| `00200000` | `00000003` | \(2^{53}+3\) | `4340000000000002` | Exact tie selects the upper even significand |
| `80000000` | `00000000` | \(2^{63}\) | `43E0000000000000` | Right alignment includes a count of 32 |
| `FFFFFFFF` | `FFFFFFFF` | \(2^{64}-1\) | `43F0000000000000` | Rounding propagates into the exponent |

Around \(2^{54}\), adjacent representable integers are four apart:

| Integer | Expected numerical value | Acceptance property |
|---:|---:|---|
| \(2^{54}+1\) | \(2^{54}\) | Below halfway |
| \(2^{54}+2\) | \(2^{54}\) | Exact tie with an even lower significand |
| \(2^{54}+3\) | \(2^{54}+4\) | Above halfway |

These examples distinguish ordinary truncation from nearest rounding and distinguish ties-to-even from unconditional upward rounding.

A shift implementation that merely masks all counts modulo 32 is not accepted as equivalent on the strength of these numerical expectations. Equivalence requires authoritative target semantics or independent evidence.

## Uncertainties

### documented_only: oversized logical shifts

The supplied representation establishes the operand widths, count expressions, and reachable count ranges. It does not establish what execution does for counts of 32 or more.

Zero-producing oversized shifts make the normalization and numerical conversion coherent. That is evidence of likely intent, not proof of the target’s semantics.

Until this prerequisite is resolved, only the zero result and the described static bit-processing relationships are unconditional. The nonzero numerical mapping must remain conditional.

### documented_only: external return convention

The returned bit positions are established. Native return type, register assignment, memory byte order, and caller interpretation are not.

No floating-point ABI promise can be derived solely from the 64-bit result assembly.

### documented_only: machine-level effects

Instruction selection, processor flags, timing, and target-specific trapping behavior are not established. No explicit request to modify floating-point exception state appears, but the supplied material is not a machine-level effects audit.

### Inferred intent

Unsigned-integer-to-binary64 conversion with ties-to-even is inferred from the alignment, precision, exponent construction, and rounding behavior. No separate public API declaration or caller contract was supplied to establish that intent independently.

## Evidence

This contract uses static examination of the supplied function representation only.

Established observations are:

- Two unsigned 32-bit arguments and a 64-bit assembled result.
- An explicit all-zero path.
- Selection of the most significant nonzero input word for normalization.
- Leading-zero counting on nonzero operands.
- Wrapped 32-bit position arithmetic and eight-bit shift-count reductions.
- Reachable shift counts at and above the operand width.
- A 53-bit retained-significand layout with eleven discarded bits.
- Guard, lower-discarded-bit, and retained-parity rounding decisions.
- Unsigned carry propagation into the high result word.
- No explicit memory access, external call, or persistent-state update.

The numerical interpretation and nonzero acceptance examples are conditional deductions, not runtime observations.

**No runtime tests have been performed.**
