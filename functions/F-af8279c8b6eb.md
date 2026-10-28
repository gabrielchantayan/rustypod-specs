<!-- Trusted provenance: F-af8279c8b6eb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-af8279c8b6eb

## Purpose

F-af8279c8b6eb computes the exact difference of two signed products formed from four signed 32-bit inputs. The first input is multiplied by the fourth; the third input is multiplied by the second. The second product is subtracted from the first.

Writing the inputs as \(A,B,C,D\), in argument order, the mathematical result is:

\[
R=(A\times D)-(C\times B).
\]

The returned value is the 64-bit representation of \(R\), assembled from its upper and lower 32-bit words. For every combination of signed 32-bit inputs, the mathematical result fits in a signed 64-bit integer. Consequently, this is an exact arithmetic operation, not a saturating operation or an approximation.

The expression has the form of a two-by-two determinant. Its use for geometry, orientation, matrix arithmetic, or any other application is inferred intent only; no such application is established by the supplied evidence.

## Interface

### Inputs

The function takes four scalar arguments:

| Argument position | Contract name | Interpretation |
|---|---|---|
| First | \(A\) | Signed 32-bit integer |
| Second | \(B\) | Signed 32-bit integer |
| Third | \(C\) | Signed 32-bit integer |
| Fourth | \(D\) | Signed 32-bit integer |

Each input has the inclusive range:

\[
-2^{31}\leq A,B,C,D\leq 2^{31}-1.
\]

These names are specification notation, not additional parameters or memory locations.

All four arguments are values. None is interpreted as an address, buffer, handle, length, or ownership-bearing object. There is no input structure or caller-provided output area.

### Output

The function returns one 64-bit scalar bit pattern. Its signed two’s-complement interpretation is \(R\).

The supplied evidence establishes the result width and word construction but does not establish a caller-facing signed return declaration. A caller interpreting the returned bits as unsigned observes:

\[
R_{\mathrm{unsigned}}=
\begin{cases}
R,&R\geq 0,\\
R+2^{64},&R<0.
\end{cases}
\]

The numeric word layout is:

- Bits 31 through 0: the low 32 bits of \(R\).
- Bits 63 through 32: the high 32 bits of \(R\).

This describes significance, not byte order in memory. The evidence does not establish an ABI return-register assignment or a serialized byte layout.

### Preconditions and errors

There are no value-dependent preconditions within the signed 32-bit input domain. Zero, negative values, and both signed endpoints are supported.

The visible operation has no error return, status output, exceptional branch, or failure sentinel. A result of zero is an ordinary arithmetic result. Behavior for arguments supplied through an incompatible calling convention or with a different width is outside this contract.

## Behavior

### Signed multiplication

Both products use the signed interpretations of their operands. Each operand is widened to signed 64-bit before multiplication.

Define:

\[
P=A\times D,\qquad Q=C\times B.
\]

The function therefore retains the complete signed products. It does not first perform a 32-bit multiplication and then widen a truncated result.

For example, \(65{,}536\times65{,}536\) contributes \(2^{32}\), not zero.

For any product of two signed 32-bit inputs:

\[
-2^{62}+2^{31}\leq P,Q\leq 2^{62}.
\]

The upper endpoint is achieved by multiplying two instances of \(-2^{31}\). The lower endpoint is achieved by multiplying \(-2^{31}\) by \(2^{31}-1\). Each product is exactly representable in signed 64-bit arithmetic.

### Word extraction and subtraction

The visible construction separates the products into 32-bit words.

Let \(P_L\) and \(Q_L\) be their low words, interpreted as unsigned values from zero through \(2^{32}-1\). Let \(P_H\) and \(Q_H\) be their upper 32-bit bit patterns.

The low output word is:

\[
L=(P_L-Q_L)\bmod 2^{32}.
\]

The borrow is determined by an **unsigned** comparison:

\[
b=
\begin{cases}
1,&P_L<Q_L,\\
0,&P_L\geq Q_L.
\end{cases}
\]

The upper output word is the 32-bit representation of:

\[
H=P_H-Q_H-b.
\]

Here, interpreting the upper words as signed values gives the arithmetic meaning of the upper-word subtraction. The returned 64-bit pattern places \(H\) above \(L\).

The borrow depends on the unsigned low-word comparison, not on either complete product’s sign and not on the signed interpretation of the low words. Omitting the borrow, or replacing its comparison with a signed comparison, can change the result.

These word operations establish the same result as exact signed subtraction \(P-Q\). They are not separate outputs.

### Result bounds

The complete result satisfies:

\[
-2^{63}+2^{31}\leq R\leq 2^{63}-2^{31}.
\]

The extreme product values can be selected independently because they use disjoint argument pairs. These bounds lie inside the signed 64-bit range.

Thus, for valid inputs:

- Neither multiplication overflows signed 64-bit arithmetic.
- The mathematical subtraction does not overflow signed 64-bit arithmetic.
- A low-word wrap is handled by the upper-word borrow.
- No saturation, clamping, or arithmetic error indication is required.

The low-word subtraction is modular, but the complete signed result remains exact.

### Ordering and observability

The visible expression obtains the low word of \(P\), then the low word of \(Q\), before constructing the return value using the corresponding upper words and the borrow.

This establishes the arithmetic dependencies, not a required machine instruction schedule. All operands are scalar arguments, and the visible body contains no memory access or external call whose ordering would expose these intermediate calculations.

A replacement implementation may compute the result with a single full-width subtraction, provided it preserves the specified 64-bit result for every valid input. It need not reproduce intermediate product calculations or a particular register-level sequence.

## Side effects

The supplied body contains no visible:

- Reads from pointed-to memory or global state.
- Writes to caller-owned memory or global state.
- Allocation or release of memory.
- Resource acquisition or ownership transfer.
- I/O, callbacks, or external function calls.

The only caller-visible output established by the evidence is the returned scalar.

The operation is stateless: its specified result depends only on the four argument values. Repeated calls with the same inputs have the same specified return bits.

Machine-level condition flags, temporary registers, stack bookkeeping, and calling-convention preservation rules are not specified by the arithmetic body. Their treatment remains an ABI matter, not an additional output of this contract.

## Dependencies

No function dependencies are visible in the supplied body. F-af8279c8b6eb performs its work using scalar arithmetic, comparisons, conversions, word extraction, and word concatenation.

There are therefore no dependency function IDs to list and no unknown callee effects to assume. In particular, this contract does not rely on a callee preserving globals, allocating storage, or releasing memory.

The evidence’s word-concatenation notation describes the returned bit pattern; it does not establish an external helper call.

## Acceptance examples

The following are conditional acceptance criteria derived mathematically from the static evidence. They are not reports of executed tests.

| Inputs \((A,B,C,D)\) | Required signed result | Required 64-bit pattern |
|---|---:|---|
| \((0,0,0,0)\) | \(0\) | `0x0000000000000000` |
| \((2,3,4,5)\) | \(10-12=-2\) | `0xFFFFFFFFFFFFFFFE` |
| \((2,4,3,6)\) | \(12-12=0\) | `0x0000000000000000` |
| \((-2,3,4,5)\) | \(-10-12=-22\) | `0xFFFFFFFFFFFFFFEA` |
| \((1,1,1,0)\) | \(-1\) | `0xFFFFFFFFFFFFFFFF` |
| \((65536,0,0,65536)\) | \(2^{32}\) | `0x0000000100000000` |
| \((65536,1,1,65536)\) | \(2^{32}-1\) | `0x00000000FFFFFFFF` |
| \((65535,0,0,65537)\) | \(2^{32}-1\) | `0x00000000FFFFFFFF` |

The final two examples distinguish different low-word conditions. The penultimate example requires borrowing from the high word because the product low words are zero and one. The final example has no borrow and a low word whose top bit is set; it must not be mistaken for a negative complete result.

Boundary acceptance conditions:

- With \(A=D=-2^{31}\), \(B=2^{31}-1\), and \(C=-2^{31}\), the result must be \(2^{63}-2^{31}\), with pattern `0x7FFFFFFF80000000`.
- With \(A=C=D=-2^{31}\) and \(B=2^{31}-1\), the same positive bound applies.
- With \(A=B=C=-2^{31}\) and \(D=2^{31}-1\), the result must be \(-2^{63}+2^{31}\), with pattern `0x8000000080000000`.

More generally, exchanging the first and third arguments while also exchanging the second and fourth negates the result. Equal products must return zero regardless of their signs or magnitudes.

## Uncertainties

The following are **documented_only**:

- **Public return signedness:** The evidence establishes 64 returned bits, not the declaration used by callers.
- **ABI details:** Register assignment, stack layout, argument marshalling, and byte order are not established.
- **Application meaning:** Determinant-like intent is inferred from the expression; no domain semantics are established.
- **Instruction sequence:** Arithmetic equivalence is established, but the exact emitted instructions and timing are unknown.
- **Runtime integration:** No callers, surrounding execution state, or platform integration behavior were supplied.

These unknowns do not prevent specifying the arithmetic mapping for four signed 32-bit arguments. They must not be resolved by assuming a particular application or calling convention.

## Evidence

**Established static evidence:** Four signed 32-bit arguments are widened for signed 64-bit multiplication in the pairings first-with-fourth and third-with-second. The product low words are subtracted as 32-bit unsigned values. Their unsigned comparison supplies a borrow to the upper-word subtraction. The upper and lower results form one 64-bit return value. No external calls or memory accesses are visible.

**Mathematical deductions:** This construction returns \((A\times D)-(C\times B)\) exactly. The stated product and result bounds, unsigned interpretation of negative returns, and acceptance values follow from that construction.

**Inferred intent:** The expression resembles a determinant. No broader purpose is asserted.

**Verification status:** No runtime tests have been performed. The acceptance examples are derived expectations, not observed execution results.
