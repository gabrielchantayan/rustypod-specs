<!-- Trusted provenance: F-5ecc94b34ace; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5ecc94b34ace

## Purpose

F-5ecc94b34ace computes the unsigned integer quotient of a 64-bit value divided by ten. The input value is supplied as two 32-bit words; the result is a single 64-bit unsigned value.

For a low input word \(L\) and a high input word \(H\), define:

\[
N = L + 2^{32}H.
\]

The behavioral contract is:

\[
Q = \left\lfloor \frac{N}{10} \right\rfloor.
\]

No remainder is returned. The mathematical remainder \(N-10Q\) is necessarily between zero and nine.

This quotient relationship is established by static arithmetic analysis of the supplied representation. It is not a claim based on the function’s name, a comment, or an executed test.

## Interface

### Inputs

The function accepts two unsigned 32-bit values:

| Position | Meaning | Range |
|---|---|---|
| First | Low word \(L\) of the dividend | \(0\) through \(2^{32}-1\) |
| Second | High word \(H\) of the dividend | \(0\) through \(2^{32}-1\) |

The first input supplies bits 0–31 of \(N\); the second supplies bits 32–63. Argument order is therefore low word first, high word second.

Every pair of valid 32-bit input words is supported. There is no requirement that the represented dividend fit in a signed 64-bit integer, and the high input word’s most significant bit does not indicate a negative dividend.

### Output

The result is an unsigned 64-bit integer \(Q\), with range:

\[
0 \le Q \le 1{,}844{,}674{,}407{,}370{,}955{,}161.
\]

When described as two words:

\[
Q_{\mathrm{low}} = Q \bmod 2^{32},
\qquad
Q_{\mathrm{high}} = \left\lfloor Q/2^{32}\right\rfloor.
\]

The low result word occupies bits 0–31, and the high result word occupies bits 32–63. This is a bit-significance contract, not a specification of byte order in memory or physical return registers.

### Preconditions and ownership

The only established input precondition is that each argument be interpreted as an unsigned 32-bit word.

There are no pointer parameters, buffers, handles, or supplied storage objects. No allocation, ownership transfer, retention, or release is involved in the established behavior.

The machine-level calling convention and any language binding’s conversion rules for wider or signed arguments are not established by the supplied material.

## Behavior

### Required result

For every supported input, the returned value must satisfy both:

\[
10Q \le N
\]

and

\[
N < 10(Q+1).
\]

These are mathematical inequalities, not instructions to evaluate potentially overflowing products in a caller’s fixed-width type.

The result rounds toward zero, which for this unsigned domain is the same as rounding downward. It does not round to the nearest integer and does not return a fractional result.

The divisor is always ten. Neither input selects a divisor or operating mode.

### Arithmetic model

The supplied representation uses 32-bit unsigned operations, explicit carry and borrow information, and a signed interpretation of one final 32-bit comparison value.

For interpreting that evidence:

- Unsigned 32-bit arithmetic is reduced modulo \(2^{32}\).
- Right shifts of unsigned words insert zero bits.
- A carry reports whether an unsigned word addition exceeds \(2^{32}-1\).
- A borrow reports whether an unsigned word subtraction requires borrowing from the next word.
- Cross-word shifts combine adjacent portions of the low and high words.
- The final signed comparison examines a residual’s sign; it does not change the unsigned interpretation of the inputs.

Carries and borrows are essential to the established result. Treating the low and high inputs as independently divided words would not satisfy this contract.

### Approximation and correction

The static evidence establishes an arithmetic approximation followed by a correction of either zero or one. These are internal details; no intermediate value is exposed to a caller.

For analysis, let \(T\) denote the full-width approximation immediately before its final division by eight. It starts from the dividend minus its unsigned quarter, then incorporates successively smaller shifted contributions at distances of 4, 8, 16, and 32 bits. Each contribution uses the accumulated value at that stage.

Let:

\[
A = \left\lfloor T/8 \right\rfloor.
\]

The arithmetic bounds establish that \(A\) is either the required quotient or one below it:

\[
A \in
\left\{
\left\lfloor N/10 \right\rfloor-1,\,
\left\lfloor N/10 \right\rfloor
\right\},
\]

with negative candidates excluded because \(A\) is unsigned.

Define the mathematical residual:

\[
R = N-10A.
\]

It follows that:

\[
0 \le R \le 19.
\]

The final decision returns \(A+1\) exactly when \(R\ge10\); otherwise it returns \(A\). Thus the correction produces the required quotient across the entire input domain.

### Correction comparison and wraparound

The correction is represented through subtraction and a signed test, rather than a direct comparison of a separately returned remainder.

Its decision value corresponds to:

\[
D = N-10-10A = R-10.
\]

Since \(R\) lies between zero and nineteen:

\[
-10 \le D \le 9.
\]

Within this narrow range, testing whether the reconstructed residual is nonnegative distinguishes the correction cases exactly:

- \(D<0\): retain \(A\).
- \(D\ge0\): return \(A+1\).

Negative residuals are represented through unsigned word subtraction before their sign is examined. That intermediate wraparound is intentional; it is not an error result.

If correction increments a low result word of \(2^{32}-1\), the carry must increment the high result word. Otherwise the high result word remains unchanged. This is ordinary full-width increment behavior, not saturation.

### Boundary behavior

- A zero dividend returns zero.
- Dividends below ten return zero.
- Exact multiples of ten return their exact integer quotient.
- A nonzero remainder does not itself cause upward rounding.
- Crossing a 32-bit boundary in either the dividend or quotient does not reset, truncate, or independently divide either word.
- Values with bit 63 set remain unsigned dividends.
- The maximum 64-bit unsigned dividend is supported.
- No valid input can overflow the returned quotient.

### Observable ordering

The established interface has no caller-visible intermediate writes, callback invocations, or externally readable intermediate state.

The inputs are consumed as by-value words, and the result becomes observable on return. No observable sequencing contract exists between internal arithmetic steps beyond their contribution to the final result.

A replacement implementation may use another arithmetic strategy if it preserves the specified input interpretation, exact quotient, and absence of additional observable effects.

## Side effects

The supplied representation establishes no:

- Reads from or writes to global state.
- Reads through caller-supplied addresses.
- Writes to caller-owned storage.
- Allocation or deallocation.
- Input/output operations.
- Callback invocation.
- Mutation of either input object in the caller.

The function is therefore deterministic with respect to its two input words in the supplied model: identical word pairs produce identical result bits.

No error code, exceptional return value, or failure branch is established. All \(2^{64}\) possible input word pairs represent supported dividends.

These statements describe the represented function. They do not specify physical register clobbers, stack organization, instrumentation, or other properties outside the supplied representation.

## Dependencies

No outgoing function dependency is present in the supplied representation.

The carry, concatenation, shift, and integer operations are arithmetic notation, not identified external callees. Accordingly, there is no dependency whose behavior must be assumed to preserve globals or acquire, retain, or release memory.

F-5ecc94b34ace is the sole opaque function identifier required for this contract.

Runtime support routines introduced by a compiler, if any, are outside the supplied evidence.

## Acceptance examples

The following are conditional acceptance cases derived from the contract. They are not reports of executed tests.

Hexadecimal word values below denote unsigned 32-bit quantities. Input columns are ordered low word, then high word.

| Low input | High input | Represented dividend | Required quotient |
|---|---|---:|---:|
| `00000000` | `00000000` | 0 | 0 |
| `00000009` | `00000000` | 9 | 0 |
| `0000000A` | `00000000` | 10 | 1 |
| `00000013` | `00000000` | 19 | 1 |
| `00000014` | `00000000` | 20 | 2 |
| `FFFFFFFF` | `00000000` | 4,294,967,295 | 429,496,729 |
| `00000000` | `00000001` | 4,294,967,296 | 429,496,729 |
| `00000000` | `0000000A` | 42,949,672,960 | 4,294,967,296 |
| `FFFFFFFF` | `FFFFFFFF` | 18,446,744,073,709,551,615 | 1,844,674,407,370,955,161 |

Additional acceptance conditions:

- If the inputs are low `00000000`, high `00000001`, the result must be 429,496,729, not zero. This distinguishes full-width division from division of the low word alone.
- If the inputs are low `00000000`, high `0000000A`, the returned low word must be `00000000` and the high word `00000001`. This checks that the result is not truncated to 32 bits.
- For the maximum dividend, the returned high word must be `19999999` and the low word `99999999`. The mathematical remainder is five.
- For any dividend \(10k\) within the supported range, the result must be \(k\).
- For \(10k+r\), where \(0\le r\le9\) and the dividend remains representable, the result must remain \(k\).
- Increasing a dividend from \(10k+9\) to \(10k+10\), when both are representable, must increase the result from \(k\) to \(k+1\).

## Uncertainties

### Established static behavior

The word widths, input significance, carry and borrow propagation, result composition, lack of outgoing calls, and quotient relationship are established from the supplied arithmetic representation.

### Inferred intent

The arithmetic structure is consistent with implementing unsigned division by ten using shifts, additions, and a final correction rather than a general division operation.

That likely implementation intent is not needed to define the required result.

### documented_only

The following remain outside the established contract:

- Physical argument and return registers.
- Stack layout and register-preservation requirements.
- Memory byte order used by an external serializer or binding.
- Source-language declaration and conversion rules at call sites.
- Compiler-added runtime helpers or instrumentation.
- Execution time, instruction count, and constant-time behavior.
- Hardware-specific fault behavior or execution-environment guarantees.

No timing or side-channel guarantee follows from the absence of memory operations in the supplied representation.

No runtime tests have been performed.

## Evidence

This contract uses static analysis of the supplied representation only.

The principal arithmetic justification is:

1. The two input words form a single unsigned 64-bit dividend through explicit cross-word arithmetic.
2. The approximation stages admit the conservative bounds:

   \[
   \frac45N-6 < T < \frac45N+\frac45.
   \]

   These account for the discarded fractions in the shifted contributions. The upper bound also places \(T\), and the preceding nondecreasing accumulation stages, below \(2^{64}\).

3. Dividing these bounds by eight gives:

   \[
   \frac{N}{10}-\frac34
   <
   \frac{T}{8}
   <
   \frac{N}{10}+\frac1{10}.
   \]

   Since \(N\) is an integer, the fractional part of \(N/10\) is one of \(0,0.1,\ldots,0.9\). The strict upper bound prevents the approximation from reaching the next quotient integer. The lower bound limits any underestimate to one.

4. The remaining arithmetic reconstructs the high-word sign of \(N-10-10A\), including multiplication carries and subtraction borrows.
5. That residual lies between negative ten and positive nine. The signed decision therefore applies exactly the correction needed for unsigned division by ten.
6. The return composition preserves both result words, including a carry from the optional increment.

The acceptance examples are consequences of these static findings. They provide no execution evidence.
