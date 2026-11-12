<!-- Trusted provenance: F-72bbeb9af70c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-72bbeb9af70c

## Purpose

F-72bbeb9af70c publishes rectangle coordinates, target dimensions, placement offsets, and a packed pair of scale values. It accepts either caller-supplied geometry or a request to derive geometry from extents, targets, an optional ratio, orientation, and an object policy byte.

**Established static behavior:** the routine performs 32-bit arithmetic, invokes arithmetic dependencies, reads shared references and configuration fields, writes fixed-offset words, and invokes four configuration dependencies in a fixed order.

**Inferred intent:** the configuration describes a scaled viewport or image. One derived-geometry policy retains the requested rectangle while changing a target dimension; the other computes a centered rectangle. The physical axes, units, and consumer of these fields are not established.

This contract distinguishes captured references from separate evaluations of shared references. Those distinctions matter because no synchronization or dependency guarantee establishes that successive evaluations return the same address.

## Interface

### Arguments

All thirteen arguments occupy 32 bits.

| Position | Name | Interpretation |
|---|---|---|
| 1 | Object | Signed, address-like value passed to configuration dependencies and conditionally used for a byte read at offset `0x1E6`. |
| 2 | Mode | Signed selector: zero means direct mode; every nonzero value means derived mode. |
| 3 | High origin | Signed high-position component of the direct rectangle origin. |
| 4 | Low origin | Unsigned low-position component of that origin. |
| 5 | High extent | Unsigned requested extent associated with the high packed coordinate. |
| 6 | Low extent | Unsigned requested extent associated with the low packed coordinate. |
| 7 | High placement | Signed high-position component of direct placement. |
| 8 | Low placement | Unsigned low-position component of direct placement. |
| 9 | High target | Unsigned initial target associated with the high coordinate. |
| 10 | Low target | Unsigned initial target associated with the low coordinate. |
| 11 | Ratio enable | Signed selector; zero disables ratio adjustment. |
| 12 | Ratio numerator | Unsigned ratio component, subject to reduction. |
| 13 | Ratio denominator | Unsigned ratio component, subject to reduction. |

“High” and “low” identify packed positions, not established physical axes.

### Results and accessible state

There is no return value or explicit status result. Observable results are dependency calls and memory accesses.

The routine requires:

- A readable shared main-base reference.
- Appropriately accessible 32-bit fields at main-base offsets `0x30`, `0x34`, `0x4C`, `0x50`, `0x54`, and `0x3AC`, as used on the selected path.
- A readable shared value for the first configuration dependency.
- A readable shared secondary-base reference.
- A readable byte at Object plus `0x1E6` only when derived mode reaches the unequal-scale decision.
- Whatever additional validity conditions the dependencies require.

There are no explicit address checks, extent checks, failure reports, or recovery paths.

### Arithmetic and layout notation

The following are contract notation, not additional dependencies:

- **Q(n, d):** F-870c4fcf7295 called with two 32-bit patterns.
- **R(n, d):** F-d7be9cfb4000 called with two 32-bit patterns.
- **Pack(high, low):** the 32-bit bitwise OR of the low operand and the high operand shifted left by 16. Neither operand is independently masked.
- **Targets(high, low):** bits 0–9 contain the low target’s low ten bits; bits 16–25 contain the high target’s low ten bits; all other bits are zero.
- **Fixed-product(product):** interpret the wrapped 32-bit product as signed, then divide by 4096 with truncation toward zero.

Q and R have division-like uses, but their arithmetic semantics are unavailable.

Products, unsigned sums, and shifts retain their low 32 bits. Signed interpretations are identified below. Packing can overlap fields: high bits in the low operand remain present and can affect the upper half.

The supplied representation includes signed shifts and signed address-like arithmetic. The description follows apparent 32-bit machine behavior; language-level portability is unresolved.

## Behavior

### 1. Entry capture and direct preliminary writes

At entry, before the mode branch, the routine captures the main-base reference. This capture occurs in both modes.

In direct mode, it performs four word stores in this order:

| Offset | Value | Address selection |
|---|---|---|
| `0x30` | Pack(High origin, Low origin) | A separate evaluation of the shared main-base reference, not the entry capture. |
| `0x34` | Pack(High origin + High extent, Low origin + Low extent) | Entry capture. |
| `0x54` | Targets(High target, Low target) | Entry capture. |
| `0x50` | Pack(High placement, Low placement) | Entry capture. |

Consequently, the first store need not reach the same backing object as the following three.

Derived mode skips these stores. Caller-supplied origins and placement do not determine its rectangle or final placement.

### 2. Ratio activation and reduction

Ratio adjustment is active only when:

- Ratio enable is nonzero.
- The original numerator and denominator differ.
- The original denominator is nonzero.

Otherwise, the effective numerator and denominator both become 1, and the adjusted extent becomes the original High extent.

For an active ratio:

- If both components are below `0x01000000` and either exceeds `0xFFFF`, both are shifted right by 8.
- If either component is at least `0x01000000`, both are shifted right by 16.
- Otherwise, neither changes.

The adjusted extent is R(effective numerator × High extent, effective denominator), with the multiplication wrapping to 32 bits.

The activation decision is not repeated after reduction. Reduction may turn a nonzero denominator into zero, and a zero numerator is allowed. No local check prevents the resulting R call.

### 3. Initial scales and post-call base capture

For the initial scale calls only, each zero target is replaced by 1. Original target values remain available for subsequent geometry and publication.

The routine reads bit 0 at offset `0x3AC` through a fresh evaluation of the shared main-base reference. This one orientation observation determines all three initial Q calls:

| Orientation | First call: High scale | Second call: Low scale | Third call: Adjusted scale |
|---|---|---|---|
| Clear | Q(High extent shifted left 12, guarded High target) | Q(Low extent shifted left 12, guarded Low target) | Q(adjusted extent shifted left 12, guarded High target) |
| Set | Q(High extent shifted left 12, guarded Low target) | Q(Low extent shifted left 12, guarded High target) | Q(adjusted extent shifted left 12, guarded Low target) |

Calls occur left to right. They do not reread orientation between calls.

Immediately after the third Q returns, the routine captures the main-base reference again. This capture later supplies the rectangle-end address in the equal-scale derived branch.

If ratio adjustment is inactive, the third Q result is discarded and the adjusted scale is replaced by the first Q result. The third call nevertheless occurs.

Each zero scale is then independently replaced by `0x1000`. In the remaining description:

- **H** means normalized High scale.
- **L** means normalized Low scale.
- **A** means normalized Adjusted scale.
- **N** and **D** mean the effective ratio numerator and denominator.

These labels describe values, not shared storage.

In direct mode, the final scale pair is high H, low L. No derived geometry is performed.

### 4. Derived geometry

Derived mode initially retains both original targets and has zero placement offsets.

#### Equal adjusted and low scales

When A equals L:

1. Store zero at offset `0x30` through a separate evaluation of the shared main-base reference.
2. Store Pack(High extent, Low extent) at offset `0x34` through the capture made immediately after the third initial Q call.

The final scale pair is high A, low L, which are equal. Both original targets remain unchanged. The object policy byte is not read.

These two rectangle stores need not use the same base.

#### Unequal scales with policy byte exactly 1

When A differs from L, the routine reads one byte at Object plus `0x1E6`. A value of exactly 1 selects this policy; all other byte values select the next policy.

The comparison between A and L is signed. After selecting its comparison branch, the routine rereads orientation through a fresh evaluation of the shared main-base reference.

This policy retains the original requested rectangle but changes one target:

| Signed comparison | Orientation | Final high scale | Final low scale | Target replacement |
|---|---|---|---|---|
| A < L | Clear | R(D × L, N) | L | High target = Q(High extent shifted left 12, final high scale) |
| A < L | Set | L | R(D × L, N) | Low target = Q(High extent shifted left 12, final low scale) |
| A ≥ L | Clear | H | A | Low target = Q(Low extent shifted left 12, A) |
| A ≥ L | Set | A | H | High target = Q(Low extent shifted left 12, A) |

Any R call precedes the target Q call. The other target remains unchanged. No later zero-scale normalization is performed.

After the target Q returns, the routine captures the main-base reference. It then:

1. Stores zero at offset `0x30` through a separate evaluation of that shared reference.
2. Stores Pack(High extent, Low extent) at offset `0x34` through the new capture.

Thus the rectangle stores occur after this branch’s arithmetic calls, and their bases can differ.

#### Unequal scales with any other policy byte

This policy computes a rectangle and centering offsets. The signed comparison and fresh orientation read occur as above.

| Signed comparison | Orientation | New high extent | New low extent | Final high scale | Final low scale |
|---|---|---|---|---|---|
| A < L | Clear | Original High extent | Fixed-product(original Low target × A) | H | A |
| A < L | Set | Original High extent | R(N × Fixed-product(H × original High target), D) | A | H |
| A ≥ L | Clear | Fixed-product(original High target × R(D × L, N)) | Original Low extent | R(D × L, N) | L |
| A ≥ L | Set | Fixed-product(original Low target × R(D × L, N)) | Original Low extent | L | R(D × L, N) |

Repeated R expressions within a row refer to one result, not repeated calls. Every multiplication wraps to 32 bits before its subsequent operation.

For the changed extent, its rectangle offset is the signed interpretation of the wrapped original-minus-new difference, divided by 2 with truncation toward zero. The other rectangle offset is zero.

After these computations, the routine captures the main-base reference and performs:

1. Offset `0x30`, through a separate evaluation of the shared reference: Pack(high rectangle offset, low rectangle offset).
2. Offset `0x34`, through the capture: Pack(high rectangle offset + new high extent, low rectangle offset + new low extent).
3. An orientation read at offset `0x3AC` through that same capture.

The rectangle is not checked for containment. Negative offsets, wrapped differences, and overlapping packed bits remain possible.

The last orientation read selects target recomputation:

| Orientation | First target call | Second target call |
|---|---|---|
| Clear | High target = Q(new high extent shifted left 12, final high scale) | Low target = Q(new low extent shifted left 12, final low scale) |
| Set | High target = Q(new low extent shifted left 12, final high scale) | Low target = Q(new high extent shifted left 12, final low scale) |

The second call follows completion of the first. No orientation reread occurs between them.

### 5. Configuration calls and scale publication

Both modes next invoke four configuration dependencies:

1. F-7e8bf137318f receives Object and the shared first configuration value evaluated for this call.
2. After that call returns, the routine captures the shared secondary-base reference.
3. F-203c8240fbaa receives Object and a **separate evaluation** of the shared secondary-base reference plus `0x1F8`.
4. F-65da30b5f79a receives Object and the captured secondary base plus `0x1B0`.
5. F-f4a84f390a23 receives Object and the captured secondary base plus `0x1F8`.

Only the third and fourth configuration calls explicitly share the captured secondary base. The second call is not guaranteed to use that capture or the same underlying object.

After all four calls return:

1. The routine captures the shared main-base reference.
2. Through a separate evaluation of that shared reference, it writes the final scale word at offset `0x4C`.

The scale word contains the final low scale’s low 16 bits in bits 0–15 and the final high scale’s low 16 bits in bits 16–31. Other scale bits are discarded.

Direct mode returns after this store. The final main-base capture is not used for any later direct-mode access.

### 6. Final derived target and placement publication

Derived mode reads orientation at offset `0x3AC` through the main-base capture made after the four configuration calls—not necessarily through the base used for the scale store.

The placement frame is:

| Orientation | High target limit | Low target limit |
|---|---:|---:|
| Clear | 320 | 240 |
| Set | 240 | 320 |

The low placement offset is computed first, then the high placement offset. For each dimension:

- Interpret the target as signed 32-bit.
- If it is below its limit, subtract its 32-bit pattern from the limit, interpret the wrapped difference as signed, and divide by 2 with truncation toward zero.
- Otherwise, retain offset zero.

Finally, using the same captured main base as the final orientation read, write:

1. Offset `0x54`: Targets(current High target, current Low target).
2. Offset `0x50`: Pack(high placement offset, low placement offset).

These two stores and the final orientation read share one captured base. The preceding `0x4C` store can address another object.

Targets exceeding the frame are not cropped or rejected. Targets with bit 31 set compare as negative.

## Side effects

- Direct preliminary stores precede ratio and scale calls.
- Derived rectangle stores precede the four configuration calls.
- The final scale store follows all four configuration calls.
- Derived target and placement stores occur last.
- Capturing a reference does not freeze the referenced memory. Dependencies or external activity may change fields subsequently read through that capture.
- Fresh reference evaluations are not interchangeable with captures. Distinct evaluations may select different backing objects.
- No transaction, rollback, locking, or explicit synchronization is shown.
- The routine itself shows no allocation, deallocation, or explicit ownership transfer.
- Dependency ownership, retention, mutation, and callback behavior are unknown.
- There is no local error handling if a dependency fails to return normally. Earlier stores are not undone.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-d7be9cfb4000 | Two-argument producer of ratio-adjusted extents or scales. |
| F-870c4fcf7295 | Two-argument producer of scales or target dimensions. |
| F-7e8bf137318f | First configuration call, receiving Object and a shared configuration value. |
| F-203c8240fbaa | Second configuration call, receiving Object and a separately evaluated secondary base plus `0x1F8`. |
| F-65da30b5f79a | Third configuration call, receiving Object and captured secondary base plus `0x1B0`. |
| F-f4a84f390a23 | Fourth configuration call, receiving Object and that captured secondary base plus `0x1F8`. |

Exact Q/R arithmetic, signedness, rounding, exceptional behavior, and zero-divisor behavior are unavailable. Configuration-call results are not used as status values. No dependency is assumed to preserve globals or retain, release, or leave memory untouched.

## Acceptance examples

These are conditional static cases, not executed tests. Completion examples assume the relevant dependencies return normally and required memory remains accessible.

1. **Direct packing.** With origins high 2 and low 3, extents high 4 and low 5, and targets high `0x456` and low `0x789`, preliminary rectangle words are `0x00020003` and `0x00060008`; the target word is `0x00560389`. The first store uses the separately evaluated base, while the next stores use the entry capture.

2. **Inactive ratio still calls Q three times.** With ratio adjustment disabled, a third initial Q result differing from the first is discarded. The adjusted scale is taken from the first result before zero normalization.

3. **Equality avoids the policy read.** If all three initial Q calls return zero, all normalized scales become `0x1000`. Derived mode writes the original rectangle and does not read Object plus `0x1E6`. Its start and end stores use the distinct address-selection rules above.

4. **Reduction permits a zero divisor.** With enabled ratio, numerator `0x01000000`, and denominator 1, reduction produces numerator `0x100` and denominator zero. R receives zero as its second argument; the routine supplies no local recovery.

5. **Signed scale ordering.** An adjusted scale of `0x80000000` compares less than a low scale of 1. The signed-less-than branch applies.

6. **Distinct secondary evaluations.** If the secondary-base capture yields one address and its subsequent separate evaluation yields another, F-203c8240fbaa receives the latter plus `0x1F8`. The next two configuration calls use the former plus their respective offsets.

7. **Split final publication.** If the final main-base capture and the separate scale-store evaluation differ, `0x4C` is written through the latter. Derived orientation, target, and placement accesses still use the former.

8. **Final centering.** With final orientation clear and final targets high 200 and low 100, placement offsets are high 60 and low 70. The final placement word is `0x003C0046`.

9. **Late orientation changes matter.** If a configuration dependency changes orientation before returning, final placement uses the subsequent orientation observation through the final captured main base, not an earlier scale-calculation observation.

## Uncertainties

- **documented_only:** Dependency implementations and exceptional behavior are unavailable.
- **documented_only:** Memory type, volatility, endianness, physical units, and object lifetime guarantees are not established.
- **documented_only:** No synchronization guarantee establishes stability between separate shared-reference evaluations.
- **documented_only:** No dependency guarantee establishes stability of fields accessed through captured references.
- **documented_only:** Exact language-level behavior of extreme signed shifts and address arithmetic is unresolved; this contract describes apparent 32-bit operations.
- **Inferred intent:** `0x1000` represents unity with twelve fractional bits.
- **Inferred intent:** The orientation-dependent limits describe 320-by-240 and 240-by-320 placement frames.

## Evidence

Established claims come from the supplied static function representation: argument widths, predicates, explicit masks, shifts, signed comparisons, dependency-call order, fixed-offset accesses, and the distinction between saved references and separate shared-reference evaluations.

The behavioral conclusions do not rely on the prior draft’s assertions. Dependency implementations, runtime traces, and external interface documentation were not supplied. No runtime tests have been performed.
