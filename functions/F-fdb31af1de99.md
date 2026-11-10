<!-- Trusted provenance: F-fdb31af1de99; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fdb31af1de99

## Purpose

F-fdb31af1de99 accepts two significant 32-bit operand words and ordinarily returns a 32-bit result word. Its operand fields, significand multiplication, and exceptional classifications support an **inferred intent** of software multiplication for a binary32-like floating-point format.

The established behavior is a word-level contract, not a guarantee of IEEE 754 multiplication:

- Exponent-zero operands do not undergo general subnormal multiplication.
- In exceptional processing without an exponent of 255, true zeros retain their sign contributions; exponent-zero operands with nonzero fractions lose theirs.
- NaN-like operands select an unresolved indirect control transfer.
- An infinity-like operand paired with an exponent-zero operand selects an undeclared data value.
- Normal-path range handling includes an address-derived comparison whose complete semantics are unresolved.

All behavior below is derived from static evidence. No runtime tests have been performed.

## Interface

### Explicit inputs and output

| Position | Width | Established use |
|---|---:|---|
| First | 32 bits | First encoded operand |
| Second | 32 bits | Second encoded operand |
| Third | 32 bits | No visible use |
| Fourth | 32 bits | Incoming value is not read; its parameter storage is reused on paths requiring a second-exponent intermediate |

An ordinary return supplies an unsigned 32-bit word.

The first two operands have this logical layout:

| Field | Bits | Width |
|---|---|---:|
| Sign | 31 | 1 bit |
| Exponent | 30–23 | 8 bits |
| Fraction | 22–0 | 23 bits |

The following classifications describe encodings, without imposing standard floating-point semantics:

- **True zero:** exponent zero and fraction zero, with either sign.
- **Subnormal-like:** exponent zero and fraction nonzero.
- **Normal:** exponent from 1 through 254.
- **Infinity-like:** exponent 255 and fraction zero.
- **NaN-like:** exponent 255 and fraction nonzero.

“Exponent-zero” includes both true zeros and subnormal-like operands.

### Implicit input and preconditions

The NaN-like path uses an incoming return-link value. Its low two bits are cleared to form an indirect transfer target. This value is not an explicit argument.

The visible body performs no explicit validation. Successful completion depends on the selected path:

- Ordinary arithmetic requires the 32-bit word, signed-shift, and 64-bit product interpretations described below.
- The undeclared-data path requires evaluation of an unresolved value expression and conversion to the return type.
- The indirect-transfer path requires a usable target and any prerequisites imposed by that target.
- Final range selection depends on incompletely specified address arithmetic.

There are no explicit pointer arguments, buffers, lengths, or ownership-bearing parameters. The calling convention, implicit argument state, and validity of the return-link-derived target remain **documented_only**.

## Behavior

### Arithmetic interpretation

The established word-level interpretation uses:

- Unsigned 32-bit arithmetic with reduction modulo \(2^{32}\).
- Logical right shifts for unsigned words.
- Two’s-complement signed interpretation of 32-bit words.
- Arithmetic right shift for the signed exponent-adjustment intermediate.
- A full unsigned 64-bit significand product.

Unsigned comparisons must remain distinct from signed reinterpretations. In particular, the range decisions must not be replaced with conventional floating-point overflow or underflow rules.

### Classification and precedence

Initial classification examines the first exponent for zero. Only if it is nonzero does it examine the second exponent for zero. If neither is zero, it checks the first exponent for 255 and then, if necessary, the second exponent for 255.

Only two normal operands enter significand multiplication.

Every other combination enters exceptional processing, which independently obtains the second exponent. Thus, selecting exceptional processing because the first exponent is zero does not make the second operand irrelevant.

Exceptional processing distinguishes:

1. Neither exponent is 255.
2. At least one exponent is 255, with a NaN-like operand taking precedence.
3. No NaN-like operand, but an infinity-like operand paired with an exponent-zero operand.
4. Remaining infinity-like combinations.

### Exceptional processing without exponent 255

At least one operand has exponent zero on this branch. The returned fraction and exponent are zero; only the returned sign may be set.

For each original operand, define its **sign contribution**:

| Original operand class | Sign contribution |
|---|---|
| Normal | Its original sign |
| True zero | Its original sign |
| Subnormal-like | Zero, regardless of its original sign |

The result sign is the XOR of the two contributions. Equivalently, an operand contributes its original sign exactly when its original exponent is nonzero **or** its original fraction is zero.

This distinction follows from a local transformation that sets bit 23 whenever an operand’s fraction is zero. For a true zero, that bit changes the temporary exponent from zero to nonzero. The later exponent checks therefore retain the true zero’s sign contribution. They discard the sign contribution only when the original operand has exponent zero and a nonzero fraction.

The transformations affect local operand copies, not caller storage.

The complete class-level result is:

| First class | Second class | Returned zero’s sign |
|---|---|---|
| True zero | True zero | XOR of both original signs |
| True zero | Subnormal-like | First operand’s sign |
| Subnormal-like | True zero | Second operand’s sign |
| Subnormal-like | Subnormal-like | Zero |
| True zero | Normal | XOR of both original signs |
| Normal | True zero | XOR of both original signs |
| Subnormal-like | Normal | Normal operand’s sign |
| Normal | Subnormal-like | Normal operand’s sign |

Two normal operands do not reach this branch.

Consequently, negative zero multiplied by a positive normal operand returns negative zero. Two negative true zeros return positive zero, but mixed-sign true zeros return negative zero. Two subnormal-like operands return positive zero regardless of their signs.

No fraction multiplication occurs on this branch.

### Exceptional processing with exponent 255

#### NaN-like precedence

NaN-like detection uses doubled operand words with 32-bit wrapping arithmetic. Doubling removes the sign contribution. The doubled value exceeds `0xff000000` exactly when the original operand is NaN-like.

The first operand is tested first. The second operand is tested only if the first is not NaN-like.

If either operand is NaN-like, control transfers through the incoming return-link value with its low two bits cleared. This occurs before selecting the undeclared exceptional-result value or assembling an infinity result.

The supplied representation models this transfer as a call and forwards its returned 32-bit word. Whether the underlying transfer actually behaves as an ordinary call is unresolved. The contract therefore does not promise:

- Normal return.
- A particular NaN payload or quieting behavior.
- Floating-point exception-status changes or preservation.
- Preservation of globals or caller-accessible memory.

If the transfer returns as represented, its returned word becomes this function’s result.

#### Infinity-like paired with exponent-zero

After excluding NaN-like operands, an infinity-like operand paired with any exponent-zero operand selects an undeclared data value as the return expression.

This includes both true zeros and subnormal-like operands. A nonzero fraction in the exponent-zero operand does not avoid this selection.

The established output is the unsigned 32-bit word obtained from evaluating that expression and applying the actual return conversion. The evidence does not establish the data entity’s declaration, storage width, signedness, value, or access mechanism.

In particular, a 32-bit return type does not establish a 32-bit underlying object or a 32-bit memory read. No specific result encoding, including a canonical NaN, is guaranteed.

#### Other infinity-like combinations

With no NaN-like operand and neither exponent zero, the presence of an infinity-like operand produces:

| Operand signs | Result |
|---|---|
| Equal | `0x7f800000` |
| Different | `0xff800000` |

This includes infinity-like paired with normal and infinity-like paired with infinity-like.

### Normal significand product

For each normal operand, the significand word consists of its 23 fraction bits shifted left eight places, with bit 31 set. The encoded sign and exponent do not survive this construction.

The two unsigned significand words are multiplied to produce a full 64-bit product.

Let \(H\) initially denote the product’s high 32 bits. If its low 32 bits are nonzero, bit 0 of \(H\) is set. This is a sticky-bit merge, not an increment.

### Exponent adjustment and normalization

For the normal path, define:

- \(e_1,e_2\): the original unsigned exponent fields.
- \(d\): 1 when the operand signs differ, otherwise 0.
- \(n\): 1 when bit 31 of the sticky-merged \(H\) is set, otherwise 0.

The signed intermediate is:

\[
I=2^{16}(e_1+e_2-128)+256d.
\]

For normal operands, this fits in a signed 32-bit value.

The adjustment word is:

\[
T=\left(I+\left\lfloor I/2^{16}\right\rfloor+n\right)\bmod 2^{32}.
\]

The floor expresses arithmetic right shift, including for negative \(I\).

The normalized significand word \(N\) is \(H\) when \(n=1\), and the low 32 bits of \(2H\) when \(n=0\).

The sign difference is embedded in \(T\). There is no separate final operation that always restores the input-derived sign.

### Candidate and correction selection

Define:

\[
Q=N\mathbin{\text{ logical-right-shift }}8,
\qquad
g=\text{bit 7 of }N.
\]

The initial candidate is:

\[
C=(Q+T\cdot2^{23}+g)\bmod2^{32}.
\]

The correction decision uses a quantity \(R\):

| Condition | \(R\) |
|---|---|
| \(g=1\) | Low seven bits of \(N\) |
| \(g=0\) | \(Q\) |

Thus \(R\) is not uniformly a discarded-bit remainder.

Further processing is selected according to:

| Condition | Further processing selected when |
|---|---|
| \(R\ne0\) | Unsigned \(T>\texttt{0x00fbffff}\) |
| \(R=0\) | \(g=1\) |

If further processing is not selected, the result is the initial candidate.

When it is selected, candidate bit 0 is cleared if either \(R=0\) or \(T=\texttt{0x00fc0000}\). All subsequent decisions use this possibly modified candidate.

### Range handling

Range handling applies only when the correction decision selects further processing.

| Condition | Outcome |
|---|---|
| Unsigned \(T<\texttt{0x00fc0000}\) | Return the candidate |
| Otherwise, signed interpretation of wrapping \(T-\texttt{0x00fc0000}\) is negative | Apply the under-range rule |
| Otherwise | Apply the final range rule |

**Under-range rule:** interpret \(T\) XOR `0x000000ff` as signed. If this value is less than −65,535, return `0x00000000`; otherwise return the candidate.

This replacement is positive zero. No separate sign restoration occurs.

**Final range rule:** form the low 32 bits of twice the candidate plus `0x00800000`, with the addition performed before doubling. Interpret that word as signed.

- A negative value returns the candidate.
- A nonnegative value selects one of the two infinity encodings through an address-derived signed comparison.

The comparison combines the candidate with an address base numerically represented as `0xa0000000`. Its address width, scaling, and complete conversion semantics are unresolved.

Under a byte-addressed, wrapping 32-bit interpretation, a negative signed sum selects `0xff800000`, and a nonnegative sum selects `0x7f800000`. This is a conditional interpretation, not an unconditional platform-independent result.

## Side effects

The ordinary arithmetic and direct exceptional-result paths show no explicit caller-memory writes. Operand changes and intermediate calculations are local.

The undeclared exceptional-result expression is evaluated only after:

1. Exceptional classification.
2. Exclusion of NaN-like operands.
3. Recognition of an infinity-like operand paired with an exponent-zero operand.

There is no visible assignment to that data entity. The evidence does not establish whether evaluating it performs a memory access, nor the number or width of any accesses.

The final range comparison contains an address expression but no visible dereference of the resulting address.

The indirect target may read or modify state beyond this body. Its allocation, release, blocking, trapping, exception, and return behavior are unknown. Therefore, the entire function cannot unconditionally be classified as pure, nonallocating, or normally returning.

No explicit ownership transfer appears in the ordinary paths. Ownership behavior of the indirect dependency remains unresolved.

There is no visible common error-code channel, diagnostic output, or validation failure. Exceptional operand encodings select result words, unresolved data evaluation, or indirect control transfer rather than a separately reported error.

## Dependencies

Only F-fdb31af1de99 has a supplied opaque function ID. No opaque ID is available for the indirect target; assigning one would invent an identity.

| Dependency | Selection | Established contract |
|---|---|---|
| Unresolved indirect function dependency; opaque ID unavailable | Either operand is NaN-like | Target derives from the incoming return-link value with its low two bits cleared; a returned word is forwarded if the transfer returns as represented |
| Unresolved exceptional-result data entity | Infinity-like paired with exponent-zero, after excluding NaN-like operands | Its evaluated value, after the actual return conversion, supplies the 32-bit result |
| Address-derived comparison | Final normal-path range selection | Selects an infinity encoding; exact address arithmetic remains **documented_only** |

No dependency’s identity, declaration, preservation guarantees, or ownership policy follows merely from these uses.

## Acceptance examples

These are static deductions, not executed tests. Normal-path results assume the stated arithmetic interpretation.

| First operand | Second operand | Required result or observation |
|---|---|---|
| `0x3f800000` | `0x3f800000` | `0x3f800000` |
| `0x40000000` | `0x40400000` | `0x40c00000` |
| `0xbf800000` | `0x3f800000` | `0xbf800000` |
| `0x80000000` | `0x3f800000` | `0x80000000`; the true zero retains its negative sign contribution |
| `0x00000000` | `0xbf800000` | `0x80000000`; both retained signs participate |
| `0x80000000` | `0xbf800000` | `0x00000000`; the two negative sign contributions cancel |
| `0x80000000` | `0x80000000` | `0x00000000` |
| `0x80000000` | `0x00000000` | `0x80000000`; mixed-sign true zeros produce negative zero |
| `0x00000000` | `0x80000000` | `0x80000000` |
| `0x80000001` | `0x3f800000` | `0x00000000`; the subnormal-like operand’s negative sign is discarded |
| `0x00000001` | `0xbf800000` | `0x80000000`; only the normal operand contributes a sign |
| `0x80000001` | `0x80000000` | `0x80000000`; only the true zero contributes a sign |
| `0x80000001` | `0x00000001` | `0x00000000`; neither subnormal-like operand contributes a sign |
| `0x00800000` | `0x3f800000` | `0x00800000` |
| `0x00800000` | `0x3f000000` | `0x00000000`, rather than the mathematical subnormal product |
| `0x7f800000` | `0xbf800000` | `0xff800000` |
| `0xff800000` | `0xbf800000` | `0x7f800000` |
| `0x7f800000` | `0x00000001` | Select the unresolved exceptional-result expression |
| `0x7fc00000` | `0x3f800000` | Select the return-link-derived indirect transfer |
| `0x00000000` | `0xffc00000` | Select the indirect transfer before exceptional-result data evaluation |

For the unresolved-data example, if evaluation and return conversion yield a word \(W\), the function returns \(W\). This condition establishes neither the data entity’s storage width nor a load width.

Changing only the third or incoming fourth input does not change the visible direct calculations or classification. This does not constrain what an unresolved indirect target might observe through its actual calling convention.

## Uncertainties

**Inferred intent:** software binary32-like multiplication with compact normalization, rounding, and range handling.

Full IEEE 754 conformance is not established. The evidence does not establish gradual underflow, configurable rounding modes, standard NaN propagation, signaling-NaN handling, or floating-point exception flags.

The following remain **documented_only**:

- The exceptional-result data entity’s declaration, value, signedness, storage width, mutability, and representation.
- Whether its evaluation entails memory access, and the width and number of any accesses.
- Conversion from its value to the unsigned 32-bit return type.
- The indirect target’s identity, calling convention, arguments, effects, ownership behavior, and return behavior.
- Whether the represented indirect call accurately describes the underlying transfer.
- Address scaling, address width, and conversion semantics in the final range comparison.
- Guarantees imposed by surrounding callers or platform conventions.
- Reachability and behavior of regions omitted from the supplied representation.

These gaps do not justify substituting conventional floating-point-library behavior.

## Evidence

**Established static evidence:** the supplied body exposes operand-field extraction, ordered exceptional classification, local bit-23 insertion before exceptional sign selection, a full 64-bit significand product, sticky-bit incorporation, normalization, candidate correction, mixed signed and unsigned range decisions, an unresolved data return expression, and a return-link-derived indirect transfer.

**Exceptional-sign evidence:** the exponent checks used to discard sign contributions occur after bit 23 is set for zero-fraction operands. True zeros therefore acquire a nonzero temporary exponent and retain their original signs. Exponent-zero operands with nonzero fractions remain exponent-zero and lose their sign contributions. This establishes the contribution rule and mixed-sign-zero examples above.

**Representation boundary:** the unsigned 32-bit return establishes the returned word’s width, not the unresolved data entity’s declaration or an underlying memory-access width.

**Evidence limitations:** no declaration for the unresolved data entity, indirect-target implementation, caller context, or authoritative address-expression semantics is supplied. Reports that regions were omitted as unreachable do not independently establish their reachability.

**Runtime status:** no runtime tests have been performed. Acceptance examples are static deductions subject to their stated conditions.
