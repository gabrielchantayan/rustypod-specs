<!-- Trusted provenance: F-cded3bb79772; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cded3bb79772

## Purpose

F-cded3bb79772 transforms a 32-bit input through an external arithmetic dependency, uses part of the dependency’s result to select and interpolate byte-valued table entries, scales the interpolated value, and writes one 32-bit output. On the visible normal-return path, its direct return value is zero.

The table interpolation and scaling are established by the supplied static evidence. The interpretation as a calibrated curve or normalization operation is **inferred intent**, not an established application-level contract. In particular, the arithmetic dependency’s meaning is unknown; it must not be described as division without additional evidence.

No runtime tests have been performed.

## Interface

The function accepts three arguments:

| Argument | Role | Required access |
|---|---|---|
| Context address | Base address for a 32-bit dependency argument, a byte table, and a 32-bit scale value | Read access at the offsets described below |
| Output address | Destination for the computed result | Writable 32-bit storage |
| Input address | Source of the input bit pattern | Readable 32-bit storage; read twice before the dependency call |

The direct return is a 32-bit zero on normal completion. The computed result is delivered through the output address.

### Context layout

Offsets are byte offsets from the context address.

| Offset | Width | Established use |
|---|---:|---|
| 8 | 32 bits | Passed as the third argument to F-7071f254f784 |
| 94 through 193, inclusive | 100 bytes | Byte-valued interpolation table |
| 1536 | 32 bits | Scale storage, read twice after interpolation |

No other context fields are explicitly accessed by this function. This does not constrain memory accessed by F-7071f254f784.

Table entries are interpreted as unsigned bytes, with values from 0 through 255. Other explicit memory accesses have 32-bit width. Byte order and alignment requirements for those accesses are **documented_only**: the supplied evidence does not independently establish them.

### Preconditions and ownership

For defined normal operation:

- Each address actually accessed must be valid for the corresponding read or write.
- Address calculations must resolve to the intended storage under the target execution model.
- F-7071f254f784 must return normally.
- The context fields and table bytes needed after that call must remain accessible after it returns.

No explicit null, bounds, alignment, or lifetime checks are visible.

The function contains no explicit allocation or release operation. It does not visibly transfer ownership of any argument. However, the dependency’s ownership and memory effects are unknown; its behavior must not be assumed to preserve allocations, globals, or referenced storage.

Aliasing between the arguments is not explicitly rejected. Its consequences follow the access order below, rather than a separate non-aliasing guarantee.

## Behavior

### 1. Construct the dependency arguments

The function first reads the 32-bit word at the input address as an unsigned value and multiplies it by 419,430,400, hexadecimal 0x19000000. This multiplication is represented at 64-bit width.

The low 32 bits of that product become the first dependency argument.

The second dependency argument is formed from:

- The high 32 bits of that product.
- A sign adjustment derived from a **second** read of the input word.

The second input read is interpreted as signed for the sign adjustment. Its arithmetic right shift by 31 supplies either zero or negative one. Multiplication by 419,430,400 therefore contributes either zero or negative 419,430,400 to the high word.

When both input reads observe the same bit pattern, these first two dependency arguments are the low and high words of the signed 64-bit product:

\[
\operatorname{signed32}(\text{input}) \times 419{,}430{,}400.
\]

This equivalence is conditional on consistent input reads. If the two reads differ, the low product word and its sign adjustment can correspond to different observations.

The remaining dependency arguments are:

- Third argument: the 32-bit word read at context offset 8.
- Fourth argument: zero.

F-7071f254f784 is called exactly once on the visible normal path.

The first input read and its multiplication precede construction of the sign adjustment. The evidence does not establish a portable ordering between the second input read and the context-offset-8 read during argument evaluation.

### 2. Interpret the dependency result

The dependency supplies a 64-bit result. Only its low 32 bits affect the eventual output.

Let \(R\) denote that low word.

Two quantities are derived from it:

- Fraction \(f = R \mathbin{\&} 0x003FFFFF\), in the range 0 through 4,194,303.
- Selection value: interpret \(R\) as signed 32-bit, arithmetic-right-shift it by 22, and then interpret the shifted result as unsigned 32-bit.

For a nonnegative signed \(R\), the selection value is the ordinary upper portion above the 22 fractional bits. For a negative signed \(R\), the arithmetic shift produces a negative number whose unsigned interpretation is large. Consequently, every negative signed \(R\) takes the same branch as a selection value above 99.

The high 32 bits returned by the dependency do not affect table selection, interpolation, scaling, or the stored output.

### 3. Select interpolation endpoints

The function chooses two endpoint values.

**Selection below 99**

For selection value \(k\) from 0 through 98:

1. Read the unsigned byte at context offset \(94+k\) as the lower endpoint.
2. Read the unsigned byte at context offset \(95+k\) as the upper endpoint.

These reads are separate and ordered as listed. They are not an atomic snapshot of the table.

**Selection equal to 99**

- Read the byte at context offset 193 as the lower endpoint.
- Use the constant 256 as the upper endpoint.

No byte beyond offset 193 is read for this case.

**Every other selection value**

Use 256 for both endpoints. No table byte is read.

This last case includes:

- Nonnegative signed results with selection values from 100 through 511.
- Every dependency low word whose signed interpretation is negative.

This is the established selection behavior. It is not a symmetric clamp that treats negative results as the first table entry.

### 4. Interpolate at 22 fractional bits

Let \(a\) and \(b\) be the chosen lower and upper endpoints, and let:

\[
Q = 2^{22} = 4{,}194{,}304.
\]

The function computes the following 32-bit interpolation representation:

\[
T = aQ + f(b-a).
\]

The visible arithmetic uses unsigned 32-bit operations, including unsigned subtraction for \(b-a\). A decreasing table segment therefore causes intermediate wraparound. That wraparound does not invalidate the resulting interpolation: modulo \(2^{32}\), the calculation equals the displayed expression.

Because \(0 \leq f < Q\) and both selected endpoints lie between 0 and 256, the resulting integer is always within:

\[
0 \leq T \leq 2^{30}.
\]

Thus the final interpolation value is representable without a negative signed interpretation, even when the endpoint difference was formed through unsigned wraparound.

For an ordinary table segment, \(T/Q\) is a linear interpolation between its two bytes. For selection 99, it interpolates from the last byte toward the implicit endpoint 256. For all other selections, \(T\) is exactly \(2^{30}\).

### 5. Scale and store

After interpolation, the function:

1. Reads the 32-bit unsigned scale at context offset 1536.
2. Forms the unsigned 64-bit product of that scale and \(T\).
3. Reads the scale at offset 1536 again while constructing the final output.
4. Writes one 32-bit word to the output address.
5. Returns zero.

The second scale read participates in a signed-high-word correction multiplied by the sign of \(T\). Since the established interpolation range makes that sign zero, the second scale value does not affect the numerical result. **The read itself remains part of the visible access sequence.**

If \(S\) denotes the value observed by the first scale read, the stored result is:

\[
\boxed{\left\lfloor \frac{T S}{2^{30}} \right\rfloor}.
\]

There is no round-to-nearest adjustment. The fractional part is discarded.

Because \(T \leq 2^{30}\) and \(S\) is an unsigned 32-bit value, the result is between zero and \(S\), inclusive, and fits in 32 bits.

## Side effects

### Explicit effects

Apart from the dependency call, the function explicitly performs:

- Two 32-bit input reads.
- One 32-bit context read at offset 8.
- Zero, one, or two table-byte reads.
- Two 32-bit scale reads at context offset 1536.
- One 32-bit output write.

All table and scale reads occur after the dependency returns. Changes made by the dependency to those locations can therefore affect the calculation.

The function itself contains no explicit context or input write other than the final output write if addresses alias.

### Aliasing consequences

- If output and input storage overlap, the final write occurs after both explicit input reads.
- If output overlaps the scale field, both scale reads occur before that storage is overwritten.
- If output overlaps a selected table location, the selected bytes are read before the final write.
- If the dependency modifies input storage, both explicit input reads have already occurred.
- If the dependency modifies table or scale storage, subsequent explicit reads observe whatever values the execution environment supplies at those accesses.

These statements describe access order, not permission for otherwise invalid or unsupported overlapping accesses.

### Failure behavior

No explicit error return, fallback for invalid addresses, or dependency-error interpretation is visible. Zero is the normal direct return, not evidence of validation.

Invalid memory, unsupported alignment, a dependency failure, or a dependency that does not return has no separately specified recovery path here. Such behavior is **documented_only**, not converted into an assumed zero result or no-op.

## Dependencies

### F-7071f254f784

**Established call contract at this site:**

- One call.
- Four 32-bit arguments.
- First two arguments contain the constructed product words.
- Third argument comes from context offset 8.
- Fourth argument is zero.
- A 64-bit result is consumed.
- Only its low word affects the output.

**Unknowns:**

- Whether the operation is division, normalization, or another transformation.
- Signedness and semantic meaning of its arguments.
- Whether zero or particular context values are invalid.
- Memory, global-state, allocation, release, synchronization, and exception effects.
- Whether all input combinations return normally.

No dependency guarantees beyond the visible call and result consumption are established.

## Acceptance examples

These are conditional consequences of the static behavior, not executed tests. Unless stated otherwise, assume valid storage, normal dependency return, and stable relevant values.

### Consistent signed-input construction

If both input reads observe 0xFFFFFFFF, the first two dependency arguments encode signed negative 419,430,400 as a 64-bit two-word value. They do not encode positive \(4{,}294{,}967{,}295 \times 419{,}430{,}400\).

This example establishes argument construction only; it makes no prediction about the dependency’s result.

### Exact table knot

If the dependency low word is \(7Q\), table byte 7 is 64, and the first scale read is 1,000:

- Selection is 7.
- Fraction is zero.
- Table bytes 7 and 8 are both read, despite the zero fraction.
- \(T=64Q\).
- The stored output is 250.

The upper endpoint is numerically irrelevant in this particular case, but its read still occurs.

### Decreasing segment

If selection is 12, the fraction is \(Q/2\), the selected endpoints are 200 and 100, and the first scale read is 1,024:

- The interpolated level is 150.
- The stored output is 600.

Unsigned subtraction of the endpoint difference must not turn this into saturation or an error.

### Final table segment

If selection is 99, the fraction is \(Q/2\), the last table byte is 128, and the first scale read is 1,024:

- Only offset 193 is read from the table.
- The implicit upper endpoint is 256.
- The interpolated level is 192.
- The stored output is 768.

### Above-table and negative-result cases

If the dependency low word is either \(100Q\) or 0xFFFFFFFF:

- No table byte is read.
- \(T=2^{30}\).
- The stored output equals the first scale value exactly.

The negative-result example follows the unsigned selection comparison; it does not select the first table entry.

### Changed second scale observation

If \(T=2^{29}\), the first scale read is 1,001, and the second scale read is 7, the output is 500. The second read still occurs but does not alter the numerical result.

### Unused dependency high word

For identical low dependency-result words and identical subsequent memory observations, changing only the dependency result’s high word does not change the stored output.

## Uncertainties

The following remain **documented_only**:

- The application meaning and valid range of the input.
- The meaning of the context word at offset 8.
- The dependency’s operation and side effects.
- Context allocation size, construction rules, and table monotonicity.
- Target byte order, alignment constraints, and exceptional memory-access behavior.
- Whether external synchronization guarantees stable repeated reads.
- Whether the addresses may refer to device or otherwise side-effectful memory.
- The surrounding caller’s interpretation of the constant zero return.

Arithmetic right shift and 32-/64-bit widths are used as represented by the supplied static evidence. No independent execution or target-level validation has been performed.

## Evidence

**Established static evidence:** The supplied function body supports the argument construction, single dependency call, selection branches, table offsets, unsigned interpolation, repeated scale access, final output write, and constant normal return.

**Derived arithmetic consequences:** The interpolation range, irrelevance of the dependency’s high result word, numerical irrelevance of the second scale value, and floor-scaled output formula follow from the visible operations and endpoint bounds.

**Inferred intent:** The byte table appears to describe a piecewise-linear response that approaches an implicit endpoint of 256. This interpretation is explanatory only.

**Verification status:** No runtime tests, implementation execution, or external dependency inspection have been performed.
