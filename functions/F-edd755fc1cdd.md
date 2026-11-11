<!-- Trusted provenance: F-edd755fc1cdd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-edd755fc1cdd

## Purpose

F-edd755fc1cdd derives two 32-bit result fields from the wrapped difference between two adjacent 32-bit input words. The arithmetic uses a fixed divisor of 12. Normal completion packs the quotient field into the low 32 bits of a 64-bit return value and the remainder field into the high 32 bits.

For a working difference whose top bit is clear, the function establishes ordinary unsigned quotient-and-remainder division by 12.

For a working difference whose top bit is set, the function divides its unsigned magnitude by 12 and conditionally negates the two result fields. The supplied representation derives the sign controls using a signed 32-bit right shift by 32 positions. Its effective target semantics are unresolved. Signed division with truncation toward zero is plausible intent, but is not an unconditional contract for this path.

This specification describes static evidence and arithmetic conclusions from that evidence. No runtime tests have been performed.

## Interface

### Input layout

The function accepts one pointer providing access to two consecutive 32-bit words:

| Byte offset | Width | Role |
|---|---:|---|
| 0 | 32 bits | First word; subtracted from the second word |
| 4 | 32 bits | Second word; the starting operand of the subtraction |

The function reads both words. It does not explicitly write either word or any other part of the input region.

The layout describes word positions, not a byte-level encoding. The supplied evidence does not independently establish target endianness.

### Preconditions and access requirements

- Both words must be readable when accessed.
- The address must satisfy the target’s requirements for 32-bit integer access.
- There is no visible null-pointer, accessibility, alignment, or region-length validation.
- The two accesses do not establish an atomic or coherent snapshot. Synchronization needed for concurrently mutable storage must be provided outside this function.
- Readability does not establish ownership. The storage may be borrowed from another owner; neither caller ownership nor an ownership requirement is established.

The two reads appear as operands of a single subtraction expression. Their relative order is not established by the supplied representation. No other explicit memory read intervenes in the visible arithmetic body.

### Return layout

On normal arithmetic completion, the function returns a 64-bit value:

| Result bits | Width | Meaning |
|---|---:|---|
| 31–0 | 32 bits | Quotient field |
| 63–32 | 32 bits | Remainder field |

This specifies the numerical packing of the return value, not its physical return registers or byte order when stored in memory.

Where sign correction produces a negative field, that field contains the corresponding 32-bit two’s-complement bit pattern. Consumers must interpret the fields separately; the entire 64-bit value is not a signed quotient.

There is no separate normal-path error flag. A conditional exceptional branch can return part of a dependency result instead of an arithmetic result, as described below.

## Behavior

### Wrapped difference and path selection

Let \(a\) and \(b\) be the unsigned values of the first and second input-word bit patterns. The working difference is:

\[
d=(b-a)\bmod 2^{32}.
\]

All subsequent path selection uses this 32-bit result, rather than an unbounded subtraction.

The branch test combines the working difference with the constant 12 before examining its signed interpretation. That combination does not change bit 31. Consequently:

- \(0\le d<2^{31}\) selects the nonnegative path.
- \(2^{31}\le d<2^{32}\) selects the magnitude-and-sign path.

The selected path can differ from one chosen using wider arithmetic. In particular, neither the sign of an unbounded unsigned-operand difference nor overflow-sensitive signed subtraction is a substitute for the wrapped bit pattern.

### Nonnegative path

For a working difference with bit 31 clear, the returned fields satisfy:

\[
q=\left\lfloor\frac{d}{12}\right\rfloor,\qquad
r=d-12q.
\]

The low field contains \(q\); the high field contains \(r\).

Established invariants are:

- \(12q+r=d\).
- \(0\le r<12\).
- Both fields are nonnegative.
- The largest quotient on this path is 178,956,970.
- No dependency call is made on this path.

The function uses shorter comparison sequences for smaller values and a shared scaled-divisor mechanism for larger values. These choices do not change the quotient-and-remainder relation.

Relevant boundaries include:

- Differences from 0 through 11 return quotient zero and remainder equal to the difference.
- Difference 12 returns quotient one and remainder zero.
- Difference 23 returns quotient one and remainder 11.
- Difference 24 enters additional comparisons and returns quotient two and remainder zero.
- Differences 192 and 3,072 cross further internal comparison boundaries without changing the arithmetic contract.

### Magnitude calculation for top-bit-set differences

For a working difference with bit 31 set, the function forms its 32-bit modular negation:

\[
m=(-d)\bmod 2^{32}=2^{32}-d.
\]

The magnitude therefore satisfies:

\[
1\le m\le 2^{31}.
\]

The boundary pattern 0x80000000 is unchanged by modular negation. It represents the unsigned magnitude 2,147,483,648; no positive signed 32-bit representation is required for this magnitude.

The division mechanism obtains:

\[
q_m=\left\lfloor\frac{m}{12}\right\rfloor,\qquad
r_m=m-12q_m,
\]

with:

\[
12q_m+r_m=m,\qquad 0\le r_m<12.
\]

These magnitude relations are established independently of the unresolved sign-control operation.

### Conditional sign correction

Before magnitude conversion, the function derives a sign-control quantity from the original working difference. The supplied representation expresses this as a signed 32-bit right shift by 32 positions.

The effective semantics of that operation are **documented_only**. The representation alone does not justify treating it as a sign extension, treating the shift count as masked, or assigning it some other target-specific result.

Given a resulting 32-bit sign-control pattern, the subsequent corrections are distinct:

- Bit 31 controls modular negation of the quotient magnitude.
- Bit 30 controls modular negation of the remainder magnitude.

Thus, conditional on those bits:

| Bit 31 | Bit 30 | Quotient field | Remainder field |
|---|---|---|---|
| Clear | Clear | \(q_m\) | \(r_m\) |
| Set | Clear | \((-q_m)\bmod 2^{32}\) | \(r_m\) |
| Clear | Set | \(q_m\) | \((-r_m)\bmod 2^{32}\) |
| Set | Set | \((-q_m)\bmod 2^{32}\) | \((-r_m)\bmod 2^{32}\) |

This table describes the consequences of the controls, not four independently selectable or proven-reachable modes.

If both bits are set, the fields match signed division of the wrapped signed difference by positive 12, with truncation toward zero and a remainder having the dividend’s sign. Other control combinations need not satisfy that signed quotient-and-remainder identity.

Negating a zero field leaves it zero. Consequently, an unresolved sign control does not affect a field whose magnitude is zero.

### Arithmetic widths and division mechanism

The visible arithmetic uses 32-bit working patterns, unsigned comparisons, modular updates, and explicit 32-bit carry observations. The quotient construction includes negative-looking intermediate patterns used to control continuation. These are internal control values, not error codes.

The scaled division uses magnitudes selected from:

| Scale |
|---:|
| 12 |
| 768 |
| 49,152 |
| 3,145,728 |
| 201,326,592 |

Within the mechanism, a comparison against a shifted remaining magnitude precedes its corresponding conditional subtraction. Continuing iterations reduce the divisor scale by a factor of 64.

These operations support the quotient-and-remainder relations above. They do not require an independent implementation to reproduce the internal accumulator encoding or individual comparison sequence.

Width preservation is essential at the input boundary and during sign correction. Replacing the initial subtraction with a wider subtraction can select a different path. Replacing 32-bit negation with unrestricted mathematical negation can likewise change the represented fields.

The supplied representation is not sufficient to specify portable language semantics for every signed-looking intermediate expression. The contract concerns the represented fixed-width behavior, with the oversized sign-control shift retained as an explicit uncertainty.

### Conditional exceptional branch

The body includes a check for a zero selected divisor scale.

The visible assignments reaching that check supply nonzero scales. Later scale reductions do not return execution to the check. Accordingly, the shown arithmetic control flow provides no ordinary input that activates this branch.

Conditional on reaching it, the visible behavior is:

1. Invoke F-dd3f3ec55291 with two integer arguments, both equal to 2.
2. If the dependency returns, inspect the low 32 bits of its 64-bit result.
3. If that low field is zero, return the dependency’s high 32 bits unchanged, with the low 32 bits cleared.
4. If that low field is nonzero, execute a software-interrupt operation identified by 0x123456.
5. If the interrupt returns to the following control flow, execute an unconditional nonterminating loop.

A result returned through this branch is not established to contain an arithmetic quotient and remainder.

The interrupt’s broader effect is unknown. It is not established to be a process exit, assertion service, particular exception, or operating-system call.

## Side effects

### Visible normal-path effects

The normal arithmetic path:

- Reads the two input words.
- Computes using local working state.
- Returns the packed 64-bit result.
- Performs no explicit input-memory write.
- Contains no explicit allocation, deallocation, or ownership-transfer operation.
- Contains no explicit global-state access.
- Makes no dependency call.

The visible normal path does not change ownership. The input’s actual owner is not established: the caller may own the storage, or may merely have permission to read storage owned elsewhere. The body establishes neither caller ownership nor a requirement for exclusive ownership.

The absence of input writes does not make concurrent mutation safe or guarantee that both reads observe the same logical state.

### Conditional exceptional effects

If the zero-scale branch were reached, the dependency could perform effects not visible in this body. Its memory accesses, ownership behavior, global-state changes, I/O, and termination behavior are unknown.

No preservation of globals or memory ownership may be inferred merely from the absence of explicit accesses in F-edd755fc1cdd.

The software interrupt and, if execution continues, the nonterminating loop are additional conditional effects.

## Dependencies

| Opaque function ID | Visible interaction | Limits of established behavior |
|---|---|---|
| F-dd3f3ec55291 | Called with integer arguments 2 and 2 behind the zero-scale check; a 64-bit return is inspected | Its implementation and broader effects are unavailable; the shown scale assignments do not make the call reachable for ordinary inputs |

If the dependency returns normally, only the following use of its return value is established:

- Low 32 bits zero: preserve the high field and return a zero low field.
- Low 32 bits nonzero: proceed to the software interrupt.

The call site does not establish whether the dependency returns, preserves globals, acquires or releases memory, transfers ownership, performs I/O, or reports errors through another mechanism.

## Acceptance examples

These examples are arithmetic consequences of the supplied static representation, not observed executions. Hexadecimal packed results are complete 64-bit values.

### Established nonnegative results

| First word | Second word | Wrapped difference | Quotient | Remainder | Packed result |
|---:|---:|---:|---:|---:|---|
| 0 | 0 | 0 | 0 | 0 | 0x0000000000000000 |
| 100 | 111 | 11 | 0 | 11 | 0x0000000B00000000 |
| 100 | 112 | 12 | 1 | 0 | 0x0000000000000001 |
| 100 | 123 | 23 | 1 | 11 | 0x0000000B00000001 |
| 0 | 24 | 24 | 2 | 0 | 0x0000000000000002 |
| 0 | 192 | 192 | 16 | 0 | 0x0000000000000010 |
| 0 | 3,072 | 3,072 | 256 | 0 | 0x0000000000000100 |
| 0 | 2,147,483,647 | 2,147,483,647 | 178,956,970 | 7 | 0x000000070AAAAAAA |

### Wrapped subtraction

For first word 0xFFFFFFFF and second word zero, the working difference is 1. The required nonnegative-path result is:

**0x0000000100000000**

Using the unbounded difference −4,294,967,295 instead would not preserve this behavior.

For first word 0x80000000 and second word 0x7FFFFFFF, the working difference is 0xFFFFFFFF, despite the unsigned second word being smaller by only one. This selects the magnitude-and-sign path with magnitude one.

### Conditional negative-pattern results

For first word 13 and second word zero:

- Working difference: 0xFFFFFFF3.
- Magnitude: 13.
- Quotient magnitude: 1.
- Remainder magnitude: 1.

Conditional packed results are:

| Sign-control bit 31 | Sign-control bit 30 | Packed result |
|---|---|---|
| Set | Set | 0xFFFFFFFFFFFFFFFF |
| Set | Clear | 0x00000001FFFFFFFF |
| Clear | Set | 0xFFFFFFFF00000001 |
| Clear | Clear | 0x0000000100000001 |

Selecting one row unconditionally requires resolving the oversized-shift semantics.

For first word 12 and second word zero, the remainder magnitude is zero. Its field remains zero regardless of bit 30. The quotient field is either 1 or 0xFFFFFFFF, according to bit 31.

### Minimum signed-pattern boundary

For first word zero and second word 0x80000000:

- Working difference and modularly negated pattern: 0x80000000.
- Unsigned magnitude: 2,147,483,648.
- Quotient magnitude: 178,956,970.
- Remainder magnitude: 8.

Conditional packed results are:

| Sign-control bit 31 | Sign-control bit 30 | Packed result |
|---|---|---|
| Set | Set | 0xFFFFFFF8F5555556 |
| Set | Clear | 0x00000008F5555556 |
| Clear | Set | 0xFFFFFFF80AAAAAAA |
| Clear | Clear | 0x000000080AAAAAAA |

### Ownership-neutral input

Readable storage borrowed from a third-party owner satisfies the visible access requirement just as caller-owned storage does, provided the other access preconditions hold. The visible normal path neither takes ownership nor releases the storage. Acceptance must not require caller ownership.

## Uncertainties

### Documented_only: oversized sign-control shift

The effective semantics of the signed 32-bit right shift by 32 are unresolved. This affects sign selection for top-bit-set working differences.

The conditional correction rules remain visible, but neither a single unconditional negative-result rule nor reachability of every control combination is established.

### Documented_only: invalid or unstable input storage

The body does not validate null, inaccessible, misaligned, or insufficient storage. Platform responses to invalid accesses are not established.

For concurrently changing words, the body provides no coherent-snapshot guarantee. The resulting pair of observed values and any language- or platform-level concurrency consequences are not established.

### Documented_only: ownership and exceptional effects

The input’s actual owner is unknown. Normal-path ownership remains unchanged in the visible body.

The dependency’s effects and the software interrupt’s semantics are unavailable. Their ownership and global-state behavior must not be guessed.

### Inferred intent

Subtracting adjacent words and dividing by 12 suggests converting a span into a count of fixed-size units. The evidence does not establish an element type, container layout, relationship between the input values as addresses, or divisibility requirement.

Signed quotient-and-remainder division is plausible intent for the top-bit-set path. That interpretation does not resolve the shift ambiguity.

## Evidence

The supplied static representation establishes:

- Two adjacent 32-bit input accesses, with their relative read order unspecified.
- No explicit writes to input storage.
- A 32-bit wrapped subtraction controlling path selection.
- Division by magnitude 12 through comparisons, scaled subtractions, and carry-controlled quotient construction.
- A packed 64-bit return with low quotient and high remainder fields on arithmetic completion.
- Separate quotient and remainder sign controls.
- A sign-control expression containing an oversized 32-bit shift.
- A dependency call behind a zero-scale check not satisfied by the shown reaching assignments.
- A conditional software interrupt followed by nontermination if execution continues.
- No visible normal-path ownership change, and no evidence identifying the input’s owner.

The quotient-and-remainder identities and acceptance values are static arithmetic conclusions. Span conversion and ordinary signed-division intent remain inferences.

No dependency implementation, runtime trace, or empirical acceptance result is available. No runtime tests have been performed.
