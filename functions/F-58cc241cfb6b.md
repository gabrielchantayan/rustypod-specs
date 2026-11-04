<!-- Trusted provenance: F-58cc241cfb6b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-58cc241cfb6b

## Purpose

F-58cc241cfb6b deterministically derives a 32-bit unsigned result from two effective 32-bit inputs. Two additional 32-bit arguments are present in the interface but do not affect the result.

The established behavior is fixed-width arithmetic and bit mixing. The effective inputs are combined into a 64-bit quantity, transformed through several mixing operations, and reduced to a 32-bit result.

A hashing or fingerprinting purpose is plausible, but not established. This contract does not assign cryptographic properties, collision resistance, statistical quality, or suitability for any particular application.

## Interface

The interface has four arguments, in this order:

| Position | Width and interpretation | Effect |
|---|---|---|
| First | 32-bit value; semantic type unspecified | Unused |
| Second | 32-bit value; semantic type unspecified | Unused |
| Third | Unsigned 32-bit integer | Effective input, denoted \(a\) |
| Fourth | Signed 32-bit integer | Its 32-bit representation contributes to the arithmetic; denoted \(b\) after unsigned interpretation |

The return value is an unsigned 32-bit integer.

For the fourth argument, unsigned interpretation means reduction modulo \(2^{32}\). Thus a signed value of \(-1\) contributes \(b=4{,}294{,}967{,}295\), not a negative operand in the subsequent mixing operations.

The first two arguments are not dereferenced, interpreted as addresses, validated, or incorporated into the calculation. Their presence does not establish that they are pointers, handles, or ownership-bearing values.

There are no output parameters, buffers, length arguments, callbacks, or explicit state objects.

### Preconditions

Every bit pattern of each 32-bit argument is accepted by the visible arithmetic. In particular:

- The third argument may be zero or its maximum unsigned value.
- The fourth argument may be negative, zero, or positive.
- Intermediate arithmetic overflow is part of the defined fixed-width calculation, not an error condition.
- No alignment, accessible-memory, allocation, or object-lifetime precondition is established by the function body.

Calling-convention details beyond these argument and result widths are not established.

## Behavior

### Arithmetic model

The following mathematical description specifies the result without requiring any particular implementation strategy.

Let:

- \(M_{32}=2^{32}\).
- \(M_{64}=2^{64}\).
- \([z]_{32}\) mean the residue of integer \(z\) in the range \(0\) through \(M_{32}-1\).
- \([z]_{64}\) mean the residue of integer \(z\) in the range \(0\) through \(M_{64}-1\).
- \(\oplus\) denote bitwise exclusive OR.
- \(x\mathbin{\gg}k\), for an unsigned 64-bit quantity, denote a logical right shift with zero fill.

All complements have a specified width. A 64-bit complement of \(x\) is \(M_{64}-1-x\). Left shifts and arithmetic used to form a 64-bit state retain only the low 64 bits.

No arithmetic right shift or sign extension participates in the mixing.

### Input combination

The initial 64-bit state is:

\[
S=\left[2^{32}(b-a)+a-1\right]_{64}.
\]

Equivalently, its low and high 32-bit halves are:

\[
L=[a-1]_{32},
\]

\[
H=\left[b-a-\mathbf{1}_{a=0}\right]_{32},
\]

with:

\[
S=2^{32}H+L.
\]

Here \(\mathbf{1}_{a=0}\) is one when \(a\) is zero and zero otherwise.

This boundary condition matters: when the third argument is zero, forming the low half produces \(2^{32}-1\), and the high half includes the corresponding subtraction of one. The zero case is not equivalent to simply subtracting \(a\) from \(b\) independently of the low half.

The seed is not a direct concatenation of the two effective inputs. Both effective inputs influence the high half, while the low half depends on the third argument.

### Mixing contract

Define two families of 64-bit transformations:

\[
R_k(x)=x\oplus(x\mathbin{\gg}k),
\]

\[
D_k(x)=\left[x-2^k x-1\right]_{64}.
\]

The subtraction transformation \(D_k\) can equivalently be understood as adding \(x\) to the 64-bit complement of its truncated left shift by \(k\). The minus-one term is essential.

The mixed state is the following composition, evaluated from the innermost expression outward:

\[
T=
D_{27}\left(
R_{15}\left(
\left[
9\cdot
R_8\left(
D_{13}\left(R_{22}(S)\right)
\right)
\right]_{64}
\right)
\right).
\]

The returned value is:

\[
\boxed{
\left[T\oplus(T\mathbin{\gg}31)\right]_{32}
}
\]

This establishes the transformation order:

1. Right-shift XOR by 22.
2. Complemented-left-shift addition corresponding to \(D_{13}\).
3. Right-shift XOR by 8.
4. Multiplication by nine, modulo \(2^{64}\).
5. Right-shift XOR by 15.
6. Complemented-left-shift addition corresponding to \(D_{27}\).
7. Right-shift XOR by 31, retaining only the low 32 bits.

The 64-bit model includes the propagation of carries between the two 32-bit halves. An implementation that processes the halves independently without those carries does not satisfy this contract.

### Final reduction

If the final mixed state has high half \(H_T\) and low half \(L_T\), the returned value can also be expressed as:

\[
L_T\oplus
\left(
[2H_T]_{32}
\mathbin{\mathrm{OR}}
(L_T\mathbin{\gg}31)
\right).
\]

Thus the result is not merely the low half of the final mixed state. The final reduction incorporates the low half’s top bit and the low 31 bits of the high half.

### Determinism and input distinctions

The result depends only on the third argument and the 32-bit representation of the fourth argument. Altering either unused argument while leaving those effective inputs unchanged must leave the result unchanged.

The initial seed uniquely represents each pair of effective 32-bit input patterns: its low half recovers \(a\), and its high half then recovers \(b\). Nevertheless, the returned value contains only 32 bits. Distinct effective input pairs therefore necessarily share results somewhere in the input domain. No uniqueness guarantee applies to the return value.

There is no input-dependent loop, retry, early return, or error branch in the supplied body.

## Side effects

The supplied body establishes no externally visible side effects other than returning its result.

Specifically, it contains:

- No loads through argument-derived addresses.
- No stores through arguments.
- No global or static-state access.
- No allocation or release operation.
- No input/output operation.
- No calls to other functions.
- No exception or error-reporting operation.

The calculation uses transient arithmetic state only. It does not retain an argument value for later use.

No ownership is acquired or transferred. The caller does not receive allocated storage and is not required by this function to release anything. The function does not consume or invalidate an object supplied through either unused argument.

There is no observable ordering of external reads or writes to preserve because none are present in the supplied body. The transformation order specified above is a numerical dependency, not a sequence of externally visible memory effects.

These statements describe the supplied body. They do not establish properties of an unknown caller, wrapper, instrumentation layer, or execution environment.

## Dependencies

No external function dependency is present in the supplied body.

The carry operations represented in the source data express unsigned overflow from 32-bit addition. They are arithmetic semantics, not evidence of a callable dependency, allocator, runtime service, or stateful helper.

The only function identity required by this contract is F-58cc241cfb6b itself. No assumptions about unknown callees preserving globals or owning memory are needed.

## Acceptance examples

These are statically derived acceptance conditions, not reports of executed tests.

### Unused arguments do not affect the result

For any fixed third and fourth arguments, two calls that differ only in the first argument, second argument, or both must return identical 32-bit values.

This remains true when those unused arguments contain zero, all-one bits, or values that would not be valid addresses if interpreted as pointers. No pointer interpretation is part of this interface contract.

### Zero initial seed

When the third argument is \(1\) and the fourth argument is \(1\), the initial state is zero.

Under the specified transformations:

- The state after multiplication by nine is `0xF700000000000000`.
- The state after the right-shift XOR by 15 is `0xF701EE0000000000`.
- The final mixed state is `0xF701EDFFFFFFFFFF`.
- The returned value is `0x11FC2400`.

The first two arguments may have any 32-bit values without changing this result.

### Zero third argument

When both effective input values are zero, the initial state must be:

`0xFFFFFFFFFFFFFFFF`

A candidate implementation that instead starts from `0x00000000FFFFFFFF` has omitted the high-half adjustment associated with the low-half subtraction.

The subsequent result must follow the complete mixing contract; this example specifies the seed boundary, not a separately asserted output vector.

### Maximum unsigned third argument

When the third argument is `0xFFFFFFFF` and the fourth argument is zero, the initial state must be:

`0x00000001FFFFFFFE`

This condition exercises wraparound in the high-half arithmetic and the decrement used to form the low half.

### Negative fourth argument

When the third argument is \(1\) and the fourth argument is signed \(-1\), the initial state must be:

`0xFFFFFFFE00000000`

The negative argument contributes its unsigned 32-bit representation. Rejecting it, clamping it, or performing an unbounded signed calculation instead would violate the contract.

### Carry-sensitive implementations

If an implementation uses two 32-bit halves, its result must agree with the full 64-bit formulas even when:

- A complemented-shift addition produces a low-half carry.
- Multiplication by nine produces a low-half carry.
- Intermediate values wrap past \(2^{64}-1\).

These cases are required arithmetic behavior, not overflow failures.

## Uncertainties

### Inferred intent

The transformation resembles a fixed-width integer mixing function. Using the result as a hash, table index component, or fingerprint is plausible intent, not established behavior.

### documented_only

The following information is unavailable from the supplied body:

- The semantic meaning of either effective input.
- Why the interface retains two unused arguments.
- The intended callers and valid application-level input domain.
- Whether any surrounding interface reserves particular input or output values.
- The machine calling convention and external symbol binding.
- Performance, timing, instruction selection, and generated-code properties.
- Any cryptographic or statistical guarantees.

The mathematical contract does not depend on resolving those unknowns. No runtime tests have been performed.

## Evidence

The behavioral claims derive from static examination of the supplied function body.

Established evidence includes four 32-bit arguments, two unused arguments, unsigned 32-bit intermediate arithmetic, explicit carry propagation, fixed logical shifts, complements, exclusive OR operations, and a single unsigned 32-bit return.

The 64-bit formulation is an algebraic consolidation of those half-width operations. It preserves the seed’s zero-input adjustment, cross-half shift contributions, low-half carries, modular overflow, and final 32-bit reduction.

The hexadecimal acceptance values are mathematical derivations from that formulation. They are not observed execution output. No external documentation, dependency implementation, or runtime observation was available or used.
