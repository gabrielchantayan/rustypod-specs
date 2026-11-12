<!-- Trusted provenance: F-dfe64070bba9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dfe64070bba9

## Purpose

F-dfe64070bba9 maps six effective 32-bit input words to one packed 64-bit result. Four words define two unsigned 64-bit operands. The other two contribute only their most significant bits, which determine an independent one-bit sign selector.

The arithmetic portion of the result is a truncated 32-bit window of the operands’ exact product. The selected window depends on the product’s highest bit. There is at most one position of adjustment; there is no repeated normalization or rounding.

**Established behavior:** The output combines this product-derived field with the sign selector and thirty-one zero bits.

**Inferred intent:** The combination resembles a numerical helper with separately represented sign information. No particular numerical format, exponent convention, or surrounding operation is established.

## Interface

### Effective inputs

The following labels describe mathematical roles, not implementation identifiers or physical locations.

| Label | Width | Role |
|---|---:|---|
| \(S_A\) | 32 bits | First sign-bearing word |
| \(H_A\) | 32 bits | High word of the first arithmetic operand |
| \(L_A\) | 32 bits | Low word of the first arithmetic operand |
| \(S_B\) | 32 bits | Second sign-bearing word |
| \(H_B\) | 32 bits | High word of the second arithmetic operand |
| \(L_B\) | 32 bits | Low word of the second arithmetic operand |

Each word is an unsigned bit pattern in the range \(0\) through \(2^{32}-1\).

The supplied representation declares four arguments, corresponding in order to \(S_A\), \(H_A\), \(L_A\), and \(S_B\). It also consumes the two additional incoming words \(H_B\) and \(L_B\). Their delivery mechanism is **documented_only**. Supplying only the four declared arguments does not establish values for all effective inputs.

Define

\[
A=2^{32}H_A+L_A,\qquad
B=2^{32}H_B+L_B.
\]

Both \(A\) and \(B\) are unsigned 64-bit integers. These definitions establish numerical word significance, not memory byte order.

The sign selector \(s\) is one if exactly one of \(S_A\) and \(S_B\) has bit 31 set; otherwise it is zero. Bits 0–30 of both sign-bearing inputs are ignored.

### Return value

The function returns one 64-bit bit pattern \(R\), divided as follows:

| Bits | Contents |
|---|---|
| 63–32 | Product-derived 32-bit field \(T\) |
| 31 | Sign selector \(s\) |
| 30–0 | Zero |

Thus

\[
R=2^{32}T+2^{31}s.
\]

The sign selector does not negate an operand or the result. In particular, result bit 31 is not the sign bit of a signed 64-bit interpretation. Result bit 63 belongs to the arithmetic field and may be set independently of \(s\).

There is no separately returned exponent, status, inexact indicator, or error code. Physical return locations and external serialization are **documented_only**.

### Preconditions and ownership

All six effective input words must have defined incoming values.

No additional arithmetic restriction is established. The domain includes zero, maximum unsigned values, operands with zero high words, operands with zero low words, and operands whose highest bits are set. There is no requirement that either operand already be normalized.

No input is used as an address. The established interface has no buffer-size, alignment, pointer-validity, or allocation precondition.

The return is a scalar value. No ownership transfer, retained storage, or release obligation is established.

The equations below use exact mathematical integers. A replacement must preserve the specified unsigned bit-level result, including carry contributions. Narrowing intermediate arithmetic prematurely or relying on overflowing signed arithmetic is not an equivalent contract.

## Behavior

### Product window

Let the exact product be

\[
P=AB.
\]

It satisfies

\[
0\le P\le(2^{64}-1)^2<2^{128}.
\]

The returned arithmetic field is

\[
T=
\begin{cases}
\left\lfloor P/2^{95}\right\rfloor,&P<2^{127},\\[4pt]
\left\lfloor P/2^{96}\right\rfloor,&P\ge2^{127}.
\end{cases}
\]

Each alternative fits in 32 bits:

- Below \(2^{127}\), division by \(2^{95}\) produces a value strictly below \(2^{32}\).
- At or above \(2^{127}\), division by \(2^{96}\) produces a value from \(2^{31}\) through \(2^{32}-1\).

Equivalently, the function retains product bits 126–95 when product bit 127 is clear, and product bits 127–96 when product bit 127 is set. Bit positions refer to the completed unsigned product, with bit 0 least significant.

The highest bits of the individual input words are not arithmetic sign bits. They participate in ordinary unsigned multiplication.

### Truncation and limited adjustment

The choice between the two windows is the only normalization-like adjustment established here. The function does not continue shifting when the selected field has leading zeros.

Discarded product bits do not round or increment the retained field. There is no saturation, sticky-bit output, or report that information was discarded.

Consequently:

- Every product below \(2^{95}\) produces \(T=0\).
- Nonzero operands can produce a zero arithmetic field.
- Product bit 95 contributes when product bit 127 is clear.
- Product bit 95 is discarded when product bit 127 is set.
- Completed product bits 0–94 do not appear in the return and do not cause a rounding increment.

These statements concern bits of the completed product. They do not permit ignoring low operand words before multiplication. Those words can contribute carries and other terms that reach the retained window.

The field is not globally monotonic in the product. At the window-selection boundary, a product just below \(2^{127}\) can yield a field near `0xFFFFFFFF`, while a larger product at or just above that boundary yields a field near `0x80000000`. No exponent or adjustment flag accompanies that change.

### Independence of sign information

The arithmetic field depends only on \(H_A,L_A,H_B,L_B\). The sign selector depends only on bit 31 of \(S_A\) and \(S_B\).

For fixed arithmetic inputs:

- Changing bits 0–30 of either sign-bearing word changes nothing.
- Toggling bit 31 of exactly one sign-bearing word toggles only result bit 31.
- Toggling bit 31 of both sign-bearing words leaves the complete return unchanged.

A zero arithmetic field does not clear the sign selector. A return with only bit 31 set is therefore permitted.

### Operand relationships

The same product-window rule applies whether zero, one, or both low arithmetic words are zero. These input shapes do not establish different public operations or error policies.

A zero high word does not make an operand zero if its low word is nonzero. Conversely, either complete operand being zero forces \(T=0\), regardless of the other operand.

Equality between an operand’s high and low words is permitted. Neither ordering between those words is a precondition.

Exchanging the complete operands \(A\) and \(B\) preserves \(T\). The complete return is also preserved if the sign selector remains unchanged. This symmetry does not extend to exchanging the high and low words within one operand.

## Side effects

### Observable reads and writes

The represented body contains no explicit data-memory access, named global access, external call, callback, allocation, deallocation, I/O, or synchronization operation.

Its established external data result is the returned scalar. No persistent-state update is established.

The effective inputs are incoming values, not specified memory transactions. The contract therefore does not require six memory reads or impose a physical order for obtaining them. No externally observable sequence of data-memory reads or writes is demonstrated.

This conclusion applies to the represented body. It does not extend to an unseen caller, adapter, or execution environment.

### Machine-state limitations

Absence of explicit memory effects does not establish preservation of all machine state.

Physical input locations, scratch-state changes, condition-code effects, and preservation of unspecified registers are **documented_only**. A consumer must not infer those guarantees from the scalar arithmetic contract.

### Errors

No explicit rejection, arithmetic-error indication, exceptional return, or alternate error encoding is present.

Defined zero inputs and maximum unsigned inputs receive the ordinary result. They do not invoke a separate saturation or error policy.

Failure to supply defined values for the additional incoming words violates the effective-input precondition. No detection or reporting mechanism for that condition is established.

Fault behavior caused by external machine conditions is **documented_only**.

## Dependencies

No external function dependency appears in the supplied body. There are no dependency function IDs to enumerate.

The contract does not rely on an unknown callee preserving globals, retaining storage, or releasing memory.

An unseen surrounding component may provide the additional incoming words. Its identity, calling convention, ownership rules, and state effects are **documented_only**.

## Acceptance examples

The following are conditional acceptance requirements derived from static analysis, not observed executions. Hexadecimal quantities denote unsigned bit patterns. Unless stated otherwise, the sign-bearing inputs have matching bit 31 values, so \(s=0\).

### Zero product retains independent sign information

If either complete arithmetic operand is zero, the required return is:

| Sign relationship | Return |
|---|---|
| Matching sign bits | `0x0000000000000000` |
| Different sign bits | `0x0000000080000000` |

This includes the case where all four arithmetic words are zero. Canonicalizing the second result to all-zero bits would violate the contract.

### Ignored sign payload

For any fixed arithmetic operands, changing \(S_A\) from `0x00000000` to `0x7FFFFFFF` must preserve the complete return.

Changing it instead to `0x80000000` must toggle only result bit 31. Toggling bit 31 of \(S_B\) as well must restore the original result.

### One-position adjustment

For

\[
H_A=H_B=\texttt{0x80000000},\qquad L_A=L_B=0,
\]

both operands equal \(2^{63}\), so \(P=2^{126}\).

The required return is `0x8000000000000000`. Its highest set bit belongs to \(T\), not to the sign selector.

### Product bit 95 is retained below the selection boundary

For

\[
H_A=\texttt{0x80000000},\quad
H_B=\texttt{0xFFFFFFFF},\quad
L_A=L_B=0,
\]

the product is

\[
P=2^{127}-2^{95}.
\]

The required return is `0xFFFFFFFF00000000`. Discarding product bit 95 in this case would be incorrect.

### A set highest product bit selects the higher window

For

\[
H_A=H_B=\texttt{0xFFFFFFFF},\qquad L_A=L_B=0,
\]

the product is

\[
P=2^{128}-2^{97}+2^{64}.
\]

The required return is `0xFFFFFFFE00000000`. Neither another shift nor a rounding increment is permitted.

### Smallest retained product position

For

\[
H_A=\texttt{0x80000000},\quad H_B=1,\quad L_A=L_B=0,
\]

the product is \(2^{95}\), and the required return is `0x0000000100000000`.

Every product strictly below \(2^{95}\) has a zero arithmetic field.

### A low word changes retained output

For

\[
H_A=\texttt{0xFFFFFFFF},\quad L_A=0,\quad
H_B=0,\quad L_B=\texttt{0xFFFFFFFF},
\]

the product is

\[
P=2^{96}-2^{65}+2^{32}.
\]

The required return is `0x0000000100000000`. Changing only \(L_B\) to zero changes the return to zero. Exchanging the complete operands preserves each corresponding result.

### Nonzero operands need not produce a nonzero field

For

\[
H_A=L_A=H_B=L_B=1,
\]

the product is

\[
P=(2^{32}+1)^2=2^{64}+2^{33}+1<2^{95}.
\]

The required return is zero for matching sign bits, or `0x0000000080000000` for different sign bits. Repeated normalization that exposes lower product bits would violate the contract.

### Both low words affect the retained field

For

\[
H_A=H_B=\texttt{0x80000000},\qquad
L_A=L_B=\texttt{0xFFFFFFFF},
\]

the product is

\[
P=2^{126}+2^{96}-2^{33}+1.
\]

The required return is `0x8000000100000000`, rather than the `0x8000000000000000` obtained when both low words are zero.

### Window selection can decrease the field

Hold \(A=2^{64}-1\).

- With \(B=2^{63}\), the product is \(2^{127}-2^{63}\), and the required return is `0xFFFFFFFF00000000`.
- With \(B=2^{63}+1\), the product is \(2^{127}+2^{63}-1\), and the required return is `0x8000000000000000`.

The larger operand produces a smaller packed arithmetic field because the selected window changes.

### Maximum operands

When all four arithmetic words are `0xFFFFFFFF`,

\[
P=(2^{64}-1)^2=2^{128}-2^{65}+1.
\]

The required return is `0xFFFFFFFF00000000` for matching sign bits, or `0xFFFFFFFF80000000` for different sign bits. These results follow from truncation, not saturation.

## Uncertainties

The following remain **documented_only**:

- How the two additional incoming words are supplied.
- Whether the four-argument declaration describes the complete calling convention.
- Physical input and return locations.
- Unspecified register, scratch-state, and condition-code effects.
- Byte order for any external serialization.
- Restrictions imposed by unseen callers.
- Any numerical format represented by the words.
- Exponent handling, rounding, or further normalization outside this function.
- Timing, interrupt, fault, and concurrency behavior beyond the visible absence of synchronization.

These unknowns do not justify assuming a floating-point format, inventing an error policy, or promising preservation of unspecified state.

The arithmetic contract is conditional on defined effective inputs and the represented 32-bit word semantics. It does not assert that every overflowing signed expression in a textual representation has a portable language-level meaning.

## Evidence

**Established static evidence:** The supplied representation declares four 32-bit arguments, consumes two additional incoming 32-bit values, and returns a 64-bit quantity. The sign-bearing inputs determine only result bit 31. Every return clears bits 0–30. No external call or explicit data-memory effect appears.

**Derived static behavior:** Width-aware arithmetic analysis, accounting for carry contributions and modular word arithmetic, yields the exact unsigned-product window specified above. The result covers zero and nonzero low words, equal-word cases, and either ordering of the arithmetic words. The sign selector remains independent of this arithmetic.

**Inferred intent:** A numerical-helper interpretation is plausible but is not an acceptance requirement.

**Runtime evidence:** None. No runtime tests have been performed.
