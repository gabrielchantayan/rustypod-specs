<!-- Trusted provenance: F-92471cfdccbb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-92471cfdccbb

## Purpose

F-92471cfdccbb fills a caller-provided 16-byte record containing two encoded endpoints, two caller-selected byte values, a starting position, and an effective length.

Endpoint encoding depends on geometry obtained through external calls. Each endpoint is independently compared with a global threshold. A threshold-exceeding endpoint receives a sentinel instead of dependency-driven encoding. A requested length of −1 selects an automatic-length calculation.

**Established static behavior:** record writes, call order, visible arguments, arithmetic conversions, conditional branches, and temporary-state accesses described below.

**Inferred intent:** the record represents a geometry-addressed range, and one dependency supplies quotient-like values. Neither a particular external record format nor mathematical division is established.

**Unknown:** dependency contracts, the global threshold’s declaration, and several target-specific conversion details. These remain documented_only rather than being supplied by assumption.

## Interface

The function has seven positional inputs and returns no value in the supplied representation.

| Input | Width and interpretation | Role |
|---|---|---|
| A | Opaque 32-bit value | Saved in temporary callback-facing storage. |
| B | Destination pointer | Receives the record’s direct writes. |
| C | Opaque 32-bit value | Saved in temporary storage; its original low character-width value supplies destination byte 0. |
| D | Opaque 32-bit value | Initially saved in temporary storage; that saved location is read later to supply destination byte 4. |
| E | Unsigned 32-bit integer | Starting position. |
| F | Signed 32-bit integer | Requested length; exactly −1 selects automatic calculation. |
| G | Signed 32-bit integer | Subtracted in automatic calculation; otherwise unused by the visible arithmetic. |

The original C input and its saved copy are distinct sources: the destination write uses the original input. By contrast, the byte derived from D is read from its saved temporary location near completion. This function alone does not establish that callbacks leave that location unchanged.

### Destination layout

| Offset | Width | Content |
|---|---:|---|
| 0 | 1 byte | Character-width narrowing of C |
| 1 | 1 byte | Starting endpoint component |
| 2 | 1 byte | Starting endpoint packed component |
| 3 | 1 byte | Starting endpoint low-order component |
| 4 | 1 byte | Character-width narrowing of the then-current saved D value |
| 5 | 1 byte | Ending endpoint component |
| 6 | 1 byte | Ending endpoint packed component |
| 7 | 1 byte | Ending endpoint low-order component |
| 8 | 16 bits | Low half of E |
| 10 | 16 bits | High half of E |
| 12 | 16 bits | Low half of the effective length |
| 14 | 16 bits | High half of the effective length’s unsigned 32-bit representation |

The final four accesses are 16-bit stores, not separately specified byte stores. Their byte order is native to the target and is not independently established by the fragment.

B must designate at least 16 writable bytes, with alignment adequate for the target’s 16-bit accesses. There is no destination-size argument, pointer validation, allocation, or explicit error result. Narrowing through character and signed-halfword types also depends on the target’s conversion conventions where values are not representable.

## Behavior

### Provider dispatch and temporary state

Before any direct destination write, the function saves A, B, C, and D in temporary storage following a leading 32-bit slot. The displayed context arrangement is:

| Relative offset | Initial content |
|---|---|
| 0 | Leading 32-bit slot; not visibly initialized |
| 4 | A |
| 8 | B, represented as a pointer-sized slot in the displayed model |
| 12 | C |
| 16 | D |

F-fbcf824e7197 is called without visible arguments. Its result supplies the base for two indirect callback lookups:

1. The callback located at base offset 12 receives the address of the leading context slot.
2. After that callback returns, the callback located at base offset 8 receives the address of a separate four-byte temporary buffer.

Both lookups use the same retained provider base. The second callback target is obtained after the first callback, so a change to the provider structure can affect the second dispatch. Neither callback’s return value is visibly consumed.

Passing the context address does not establish the callback’s authorized access extent, initialization guarantees, or ownership rights. The adjacent saved values may constitute callback context, but their external contract is unavailable.

### Geometry acquisition and early capture

F-b0bad7f383ca then receives four visible arguments:

- The current leading context value.
- A pointer to a four-byte region represented as two unsigned 16-bit elements.
- A pointer to another such region.
- A pointer to a separate four-byte buffer.

Only element zero of each geometry region is visibly read by this function. Denote those elements as the **first geometry component** and **second geometry component**.

Immediately after this call, the first geometry component is captured for later starting-endpoint packing. This capture precedes destination byte 0.

The dependency’s return is temporarily retained. For automatic length, it is replaced by a fresh read of the leading context slot. For explicit length, it does not influence the subsequent record contents. Its apparent nonuse does not eliminate the call’s output or side effects.

The buffer given to the second provider callback and the buffer given to F-b0bad7f383ca are distinct. Neither buffer’s contents are visibly read afterward.

### Effective length and first write

If F equals −1, the current leading context value is captured after the geometry call and before the first direct destination write. Otherwise, F remains the effective length.

Destination byte 0 is then written from C.

Only after that write does the automatic case calculate:

**effective length = captured context value − E − G**

Both subtractions use unsigned 32-bit arithmetic: E forces the first subtraction into that domain, and G is converted for the second. The unsigned result is then assigned to the signed 32-bit length location. An out-of-range unsigned-to-signed conversion is target-dependent; the fragment does not independently establish its result.

There is no rejection of negative explicit lengths, zero, subtraction wraparound, or an automatically calculated value outside a natural range. Only −1 receives special treatment; other negative F values remain explicit lengths.

### Geometry product and stride

After byte 0 and any automatic calculation, the first geometry component is read again and widened explicitly to unsigned 32 bits. The second geometry component is multiplied by that unsigned value, so it too participates as unsigned 32-bit data.

Thus:

**geometry product = first component × second component, in unsigned 32-bit arithmetic**

For unsigned 16-bit components, the maximum product is:

**65,535 × 65,535 = 4,294,836,225**

This fits within unsigned 32 bits. The multiplication is not signed multiplication and does not introduce signed-multiplication overflow.

The product is then assigned to a signed 32-bit stride location. Products above 2,147,483,647 cannot be represented there. The unresolved operation is this unsigned-to-signed conversion, not the multiplication. No two’s-complement wrapping rule is assumed for that conversion.

The computed stride is used for both endpoints. Geometry is not multiplied again for the ending endpoint.

### Threshold comparisons

The global threshold is read separately for the starting and ending decisions. Its declaration is absent: storage width and declared signedness are unknown.

Each visible comparison places the threshold against an unsigned 32-bit position. That alone does not establish the comparison’s conversion domain. A wider signed threshold type, for example, could produce different conversions from a 32-bit signed or unsigned type.

The branch condition is therefore specified as **threshold less than position under the target’s applicable comparison conversions**, not unconditionally as an unsigned comparison. Equality of the compared values takes the dependency-driven branch.

### Starting endpoint

If the first threshold comparison selects saturation, bytes 1–3 receive:

**00 C1 FF**

No F-d7be9cfb4000 call is made for this endpoint.

Otherwise, the function calls F-d7be9cfb4000 with E and the signed stride. Its result is retained as an unsigned 32-bit value. Let **Q** be that result masked to its low 16 bits.

The residual expression is:

**E − stride × Q**

Here Q is unsigned because it comes from an unsigned result and mask. Consequently, the stride is converted to unsigned 32 bits for the multiplication, and the subtraction is also unsigned 32-bit arithmetic. These operations use modulo 2³² arithmetic; they are not signed multiplication or signed subtraction.

The unsigned residual is assigned to a signed 32-bit temporary. As with the stride conversion, an out-of-range result introduces a target-dependent unsigned-to-signed conversion.

F-d7be9cfb4000 is called again with:

- That signed residual.
- The unsigned first geometry component captured during stride preparation.

Its result is narrowed to a plain character value, which supplies byte 1.

Byte 2 combines:

- Q bits 8 and 9, placed in output bits 6 and 7.
- A one-based residual expression using the character conversions of the signed residual, the **earlier** first geometry capture made immediately after F-b0bad7f383ca, and the character result supplying byte 1.

The residual expression subtracts the product of the two character component values from the character-converted residual, then adds one. The addition is unsigned because its constant is unsigned. This result is combined by bitwise OR with the Q contribution and narrowed to one byte.

There is no six-bit mask on the residual contribution before the OR. Unexpected component values can therefore set bits also used by the Q contribution. Plain-character signedness affects the promoted arithmetic.

Byte 3 receives the character-width narrowing of the first dependency result.

### Ending endpoint

The ending position is:

**E + effective length − 1, modulo 2³²**

E makes the addition unsigned 32-bit arithmetic. There is no empty-range exception and no check that the result is at or after E.

The threshold is read again after starting-endpoint processing. Dependencies invoked for the starting endpoint may have changed it.

If the second comparison selects saturation, bytes 5–7 receive:

**FF FF FF**

Otherwise, F-d7be9cfb4000 is called with the ending position and the previously computed signed stride. Its low 16 result bits form Q. The ending residual uses the same unsigned multiplication and subtraction rules described for the starting residual, followed by assignment to a signed 32-bit location.

The next F-d7be9cfb4000 call has **only one visible argument**, that signed residual. No second argument or divisor is established.

Its character-narrowed result supplies byte 5. Byte 6 uses the same packing structure as byte 2, except that its first geometry component is read afresh **after this dependency call and after the byte-5 write**. It does not use the early saved geometry value.

Byte 7 receives the character-width narrowing of the first ending dependency result.

### Completion

After both endpoint decisions, byte 4 is written from the then-current saved D location. The four 16-bit numeric stores follow in increasing field order: E’s low half, E’s high half, effective length’s low half, and its unsigned representation’s high half.

The function returns without a status value.

## Side effects

The direct destination write order on completed execution is:

**0; 1; 2; 3; 5; 6; 7; 4; 8–9; 10–11; 12–13; 14–15.**

This is not an atomic update. An observer can encounter newly written endpoint bytes while byte 4 and numeric fields still contain previous contents.

All provider and geometry calls precede the first direct destination write. Endpoint dependency calls are conditional and occur between destination writes. Their own reads, writes, and external effects are unknown.

The function contains no visible heap allocation, release, ownership transfer, or direct global assignment. This does not establish that dependencies avoid those operations or preserve globals.

Temporary addresses escape to callbacks and F-b0bad7f383ca. Their storage is invocation-local; no valid lifetime after return or permission to release it is established. Dependency retention, release, and initialization behavior remain unknown.

There is no rollback, retry, explicit failure handling, or cleanup protocol. A failure before byte 0 leaves no direct destination writes from this function, but dependencies may already have produced side effects. Later failure can leave a partial record. Invalid pointers, invalid dispatch targets, inaccessible memory, and unsupported alignment have no visible recovery path.

## Dependencies

| Opaque dependency | Established use | Unresolved contract |
|---|---|---|
| F-fbcf824e7197 | Supplies the base for two indirect dispatches | Validity, lifetime, failure behavior, side effects |
| Callback at provider offset 12 | Receives the leading context address | Accessible extent, initialization, mutation, retention, ownership |
| Callback at provider offset 8 | Receives a four-byte buffer address | Output, purpose, side effects, retention |
| F-b0bad7f383ca | Receives a context value and three temporary-region pointers | Output guarantees, geometry meaning, failure behavior, preservation |
| F-d7be9cfb4000 | Supplies endpoint component values through two-argument and one-visible-argument calls | Actual operation, argument convention, signedness, zero-input handling, side effects |

No dependency is assumed to preserve the threshold, previously exposed temporary storage, or provider contents. A division interpretation of F-d7be9cfb4000 is inferred from result use, not established.

## Acceptance examples

These are conditional static expectations, not executed tests.

### Both endpoints saturate

Given E = 101, F = 1, and a stable threshold whose comparison value is 100:

- Bytes 1–3 become **00 C1 FF**.
- Bytes 5–7 become **FF FF FF**.
- No endpoint call to F-d7be9cfb4000 occurs.
- Provider and geometry calls still occur.
- The numeric fields represent starting position 101 and length 1, subject to the stated target storage conventions.

Assume all preceding calls and conversions complete normally.

### Equality remains dependency-driven

Given E = 100 and a threshold comparing as 100, starting saturation is not selected. The starting dependency calls occur.

The same equality rule applies independently to the ending decision. It does not imply that the threshold retains its first value.

### Automatic length ignores the geometry return

Given F = −1, a leading context value of 1000 when captured after the geometry call, E = 100, and G = 20:

- Effective length is 880.
- Ending position is 979.

The relevant value is the context read at capture time, not necessarily the first callback’s original output and not F-b0bad7f383ca’s return.

### Zero and wrapped lengths

Given E = 0 and F = 0, the ending position is **FFFFFFFF**. With a stable comparison value of 100, the ending sentinel is selected; zero length is not rejected.

Given E = **FFFFFFFE** and F = 4, the ending position is 1. With a stable comparison value of 100, the starting endpoint saturates while the ending endpoint uses dependency-driven encoding.

### Conditional ordinary encoding

Assume stable geometry components 10 and 4, E = 47, a threshold comparison that does not saturate, and ordinary integer-quotient results from the two starting calls:

- Product and signed stride are 40.
- First result is 1.
- Residual is 7.
- Second result is 0.
- Bytes 1–3 become **00 08 01**.

This example does not resolve the ending call’s missing visible argument.

### High geometry product

With both geometry components equal to 65,535, unsigned multiplication yields 4,294,836,225 exactly. It does not overflow unsigned 32 bits.

No unconditional stride or endpoint bytes are specified: assignment of that product to signed 32 bits requires the target’s out-of-range conversion rule. Under a specifically assumed bit-preserving conversion, the signed stride would be −131,071; subsequent masked-result residual arithmetic would convert it back to unsigned for multiplication.

## Uncertainties

- **documented_only — Threshold declaration:** Width and signedness are absent. Comparison conversions cannot be fixed solely from the unsigned positions.
- **documented_only — Out-of-range signed conversions:** Geometry-product and residual assignments, and automatic-length assignment, can require unsigned-to-signed conversion outside the signed range. This is distinct from arithmetic overflow.
- **documented_only — Dependency behavior:** Initialization, failures, memory ownership, retention, global effects, and preservation guarantees are unavailable.
- **documented_only — Ending call interface:** Only one argument is visible in the final component call. A hidden divisor cannot be supplied by assumption.
- **documented_only — Initialization:** The leading context slot and output regions are not visibly initialized before dependency use. Their later reads require suitable dependency behavior.
- **documented_only — Target representation:** Native byte order, plain-character signedness, and out-of-range character or halfword conversions are not independently established.
- **documented_only — Format identity:** Geometry-style packing suggests intent but does not establish a named format or consumer.

## Evidence

The supplied static representation establishes the call sequence, visible argument widths, temporary accesses, masks, explicit unsigned widening before geometry multiplication, signed destination locations, direct stores, and branch structure.

It does not include the threshold declaration or dependency implementations. Consequently, threshold storage type, full comparison conversion rules, dependency semantics, and target-specific out-of-range conversions remain unresolved.

No runtime tests, execution traces, dependency inspection, or external-format validation have been performed.
