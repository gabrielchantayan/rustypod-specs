<!-- Trusted provenance: F-bed5d5d6f858; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bed5d5d6f858

## Purpose

F-bed5d5d6f858 derives a numerical difference from two 32-bit fields and returns a quotient/remainder pair packed into a 64-bit result.

The established nonnegative behavior is division of the 32-bit difference by 40: the quotient occupies the low 32 bits and the remainder occupies the high 32 bits.

The negative-difference path computes division of the difference’s unsigned magnitude by 40, then conditionally negates the quotient and remainder. The conditions depend on an ambiguous signed shift by 32 in the supplied representation. Signed division with truncation toward zero is the inferred intent, not an unconditional guarantee for negative inputs.

This is a static behavioral contract. No runtime tests have been performed.

## Interface

### Input

The function receives one pointer to storage addressed as an array of 32-bit integer fields. The relevant layout is:

| Byte offset | Width | Role |
|---|---:|---|
| 0 | 32 bits | Subtrahend |
| 4 | 32 bits | Not explicitly accessed |
| 8 | 32 bits | Minuend |

The caller must provide storage permitting valid reads of the fields at offsets 0 and 8. No null-pointer check, bounds check, or explicit alignment check is visible. Invalid storage therefore has no defined recovery behavior in this contract.

The arithmetic input is the bit pattern obtained by subtracting the field at offset 0 from the field at offset 8, retaining 32 bits. Call this difference \(D\).

The signed interpretation of each original field does not independently determine the branch. The branch depends on bit 31 of the resulting difference. Consequently, a mathematical subtraction outside the signed 32-bit range must not be treated as an unbounded signed subtraction.

### Output

A normal return is a 64-bit value with this numerical layout:

| Bit range | Width | Meaning |
|---|---:|---|
| 0–31 | 32 bits | Quotient bit pattern |
| 32–63 | 32 bits | Remainder bit pattern |

This is a bit-position contract, not an in-memory byte-order contract. The supplied evidence does not establish how a caller stores or transports the returned value.

For nonnegative differences:

\[
Q=\left\lfloor D/40\right\rfloor,\qquad R=D-40Q
\]

and the returned value is:

\[
(R\ll32)\;|\;Q.
\]

On paths that produce negative components, each component is represented independently in its 32-bit field. The entire 64-bit return must not be interpreted as a single signed quotient.

### Ownership

The function receives no explicit allocation size, ownership flag, or output buffer. It contains no visible allocation, release, ownership transfer, or write through the input pointer on its ordinary arithmetic paths.

The exceptional dependency path is described separately. Its ownership and memory effects are unknown.

## Behavior

### Difference formation and branch selection

The function explicitly accesses the first and third 32-bit fields and forms their 32-bit difference. The middle field does not contribute to the visible arithmetic.

The supplied expression does not establish which input field is read first. This contract therefore requires both reads but does not impose an order between them. Both reads precede the subsequent arithmetic and any possible dependency invocation.

Setting bit 31 in the difference selects the negative-handling path. Otherwise, the nonnegative path is selected. The additional constant combined into the branch condition does not change this sign-bit test.

### Nonnegative differences

For \(0\leq D\leq2{,}147{,}483{,}647\), the function computes the quotient and remainder of division by 40.

The following invariants describe the normal result:

- \(Q\) and \(R\) are nonnegative.
- \(0\leq R<40\).
- \(D=40Q+R\).
- The quotient fits in 32 bits.
- The remainder is placed in the high 32-bit field even when the quotient is zero.

The arithmetic uses shifted comparisons and conditional subtraction. Its internal paths differ according to the magnitude of the difference; those paths do not establish separate public modes.

For smaller values, the function subtracts selected multiples of 40 directly. For larger values, it uses a shared scaled-division path. The change to that shared path begins at a difference of 10,240. This is an implementation boundary to exercise during validation, not a discontinuity in the intended result.

No dependency call occurs on the established ordinary division paths.

### Negative-bit differences

When bit 31 of \(D\) is set, the function forms a 32-bit unsigned magnitude by negating the difference modulo \(2^{32}\). Call this magnitude \(M\).

Thus:

\[
1\leq M\leq2{,}147{,}483{,}648.
\]

In particular, the bit pattern representing the most negative signed 32-bit value has magnitude 2,147,483,648. This magnitude remains representable as an unsigned 32-bit value; it must not be rejected solely because it exceeds the maximum positive signed 32-bit value.

The division stage computes magnitude components:

\[
Q_M=\left\lfloor M/40\right\rfloor,\qquad R_M=M-40Q_M,
\]

with \(0\leq R_M<40\).

The final quotient and remainder signs are controlled separately:

- The quotient is negated if the sign-control value is negative.
- The remainder is negated if bit 30 of the sign-control value is set.

The sign-control value is derived by a signed right shift of the original difference by 32 positions. That operation is not safely interpretable from ordinary 32-bit language semantics alone. The supplied representation does not establish its effective target behavior.

Two conditional interpretations are therefore relevant:

- If the operation produces all one bits for every negative difference, both components are negated. This yields signed division by 40 with truncation toward zero and a remainder having the dividend’s sign.
- If the operation instead retains the original negative difference, the quotient is still negated, but the remainder sign depends on the original difference’s bit 30. That can produce a positive nonzero remainder for some negative differences.

Neither interpretation may be silently substituted for the other. Negative-result acceptance requires resolution of this ambiguity or an explicitly stated conditional assumption.

### Widths and arithmetic boundaries

The observable arithmetic is expressed through 32-bit components. Negation, subtraction, sign tests, and packing must preserve their bit-pattern significance.

Important boundaries include:

- A difference wrapping from mathematical \(2^{31}\) to bit pattern `0x80000000` selects the negative path.
- A mathematical difference of \(-2^{32}+1\) becomes the positive 32-bit difference 1.
- Negating `0x80000000` modulo \(2^{32}\) leaves that bit pattern unchanged; its unsigned magnitude is nevertheless 2,147,483,648.
- Component negation affects only its own 32-bit field. It does not borrow into or carry out of the adjacent return field.

The scaled division stage uses carry-sensitive quotient construction. Its intermediate signed-looking values must not be treated as evidence that mathematical signed overflow raises an exception. No such exception path is shown.

### Exceptional dependency branch

A branch in the supplied representation tests whether an internal divisor is zero. Every visible assignment capable of supplying that divisor uses a nonzero constant or a nonzero scaled value at that test. No normal input selecting the zero-divisor branch is established.

If that branch is reached, the visible behavior is:

1. Invoke F-dd3f3ec55291 with two arguments, both numerically 2.
2. Inspect the low 32 bits of its returned value.
3. If those bits are zero, return a 64-bit value whose low 32 bits are zero and whose high 32 bits are retained from the dependency’s result.
4. If those bits are nonzero, issue the software-interrupt operation identified by `0x123456`.
5. If control resumes after that operation, remain in an infinite loop.

This branch is documented_only as an apparent unreachable exceptional path. It is not evidence that ordinary division by 40 can report a recoverable error.

## Side effects

On ordinary arithmetic paths:

- The two specified input fields are read.
- No explicit write to input storage is present.
- No explicit global read or write is present.
- No allocation, deallocation, I/O operation, or dependency invocation is present.
- A packed 64-bit value is returned.

There is no visible synchronization or snapshot mechanism. If another actor modifies the input fields concurrently, the evidence does not establish that the two reads observe a coherent pair. Language-level concurrency validity is also outside the supplied evidence.

On the exceptional branch, F-dd3f3ec55291 may have side effects not described here. Its name and call signature do not establish whether it modifies globals, touches input-related memory, allocates, releases resources, or returns normally. The interrupt may also alter control flow or machine state in ways not specified by the supplied representation.

## Dependencies

### F-dd3f3ec55291

**Invocation:** Two arguments, each equal to 2.

**Visible result use:** The caller treats the result as containing at least two 32-bit fields. It tests the low field and may retain the high field in its own return.

**Visible control dependency:** A nonzero low field selects the interrupt-and-nonreturn path; a zero low field selects a return with zero in the low field.

**Unknowns:** Argument meaning, calling convention, exact declared return type, memory effects, global effects, ownership effects, failure behavior, and termination guarantees.

**Reachability:** No ordinary input reaching this call is established by the supplied arithmetic.

No other opaque function dependency is visible.

## Acceptance examples

These are static acceptance expectations, not executed test results. Field values below are given as 32-bit bit patterns or ordinary integers where unambiguous.

### Established nonnegative cases

| Offset 0 | Offset 8 | Difference | Low field: quotient | High field: remainder |
|---:|---:|---:|---:|---:|
| 100 | 100 | 0 | 0 | 0 |
| 100 | 139 | 39 | 0 | 39 |
| 100 | 140 | 40 | 1 | 0 |
| 100 | 141 | 41 | 1 | 1 |
| 0 | 10,239 | 10,239 | 255 | 39 |
| 0 | 10,240 | 10,240 | 256 | 0 |
| 0 | 2,147,483,647 | 2,147,483,647 | 53,687,091 | 7 |

For the difference 41, the packed result is `0x0000000100000001`. This distinguishes the component layout from a reversed quotient/remainder layout.

Changing the field at offset 4, while keeping the accessed fields and their storage valid, must not change the visible ordinary arithmetic result.

### Modular subtraction

With `0xFFFFFFFF` at offset 0 and zero at offset 8, the 32-bit difference is 1. The expected result has quotient 0 and remainder 1. Treating this as an unbounded subtraction of 4,294,967,295 would select the wrong behavior.

With `0x80000000` at offset 0 and zero at offset 8, the difference bit pattern is `0x80000000`. This selects the negative path, despite the mathematical unsigned difference having positive magnitude \(2^{31}\).

### Conditional negative cases

Under the condition that the ambiguous sign-control operation produces all one bits for negative differences:

| Signed difference | Low field interpreted as signed quotient | High field interpreted as signed remainder |
|---:|---:|---:|
| −1 | 0 | −1 |
| −39 | 0 | −39 |
| −40 | −1 | 0 |
| −41 | −1 | −1 |
| −2,147,483,648 | −53,687,091 | −8 |

For −41 under that condition, the packed result is `0xFFFFFFFFFFFFFFFF`: both fields independently contain −1.

The minimum signed difference is a discriminating case. If the sign-control operation instead preserves the original difference, its bit 30 is clear, so the visible sign-adjustment logic would produce quotient −53,687,091 and remainder +8. Acceptance must not require −8 without resolving or explicitly conditioning on the sign-control semantics.

## Uncertainties

- **documented_only — Negative sign control:** The effective meaning of the signed shift by 32 is unresolved. This prevents an unconditional signed quotient/remainder contract for negative differences.
- **documented_only — Exceptional reachability:** The zero-divisor branch is present, but no valid ordinary input reaching it is established.
- **documented_only — Dependency effects:** F-dd3f3ec55291 has no supplied behavioral contract. Global preservation, memory preservation, and resource handling cannot be assumed.
- **documented_only — Interrupt behavior:** The interrupt identifier is visible; its platform meaning and whether execution resumes are unknown.
- **documented_only — Access mechanics:** Exact machine-load ordering, alignment tolerance, and fault behavior are not established.
- **documented_only — ABI:** The numerical packing is visible, but physical return registers, storage byte order, and caller interpretation are not supplied.

The likely intent is a fixed-divisor signed quotient/remainder operation over a difference between two fields. No higher-level meaning—such as an element count, storage capacity, or record size—is established merely by the divisor 40 or the field spacing.

## Evidence

The supplied static representation establishes:

- Explicit input accesses at offsets 0 and 8.
- A 32-bit difference and sign-bit branch.
- Repeated comparison and subtraction against 40 and scaled multiples of 40.
- A shared magnitude-division path.
- Independent quotient and remainder sign adjustments.
- A 64-bit return formed from two 32-bit components.
- One exceptional dependency call followed by conditional return or interrupt and nontermination.

The nonnegative quotient/remainder contract follows from the arithmetic. Signed truncation toward zero is inferred intent, conditional on resolving the negative sign-control operation.

The supplied representation also includes warnings about removed unreachable material and overlapping global symbols. Those warnings do not independently establish additional accesses, effects, or callable behavior.

No implementation execution, runtime observation, dependency inspection, or acceptance test has been performed.
