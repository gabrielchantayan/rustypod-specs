<!-- Trusted provenance: F-6a1cae7aa2bf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6a1cae7aa2bf

## Purpose

F-6a1cae7aa2bf conditionally updates two opaque objects using four sampled control values, cached comparison state, stored dimensions, and a polarity flag.

The established behavior is a change-gated sequence of arithmetic, dependency calls, and cache writes. The function always returns the 32-bit value 1 when it reaches its normal return. That value does not distinguish “nothing changed,” “update performed,” or “geometry suppressed.”

The apparent intent is to update a paired visual composition: one control influences size, one supplies complementary 8-bit values, and two influence placement or clipping. This interpretation is inferred. The meanings of the opaque objects, control units, and dependency arguments are not established.

## Interface

### Input and return

The function receives one 32-bit address identifying a writable state record. It performs no explicit null, bounds, alignment, or lifetime checks.

Normal return is a 32-bit value equal to 1. Dependency failures, exceptions, traps, or nonreturning behavior are not handled in the supplied evidence.

### State layout

Offsets are byte offsets from the supplied record address. Fields described as words are 32 bits.

| Offset | Access and established role |
|---|---|
| 0x18 | Word passed as the second argument of the final call concerning the second object. |
| 0x1c | Word passed as the second argument of the final call concerning the first object. |
| 0x20 | Opaque first-object value, read at multiple call sites. |
| 0x24 | Opaque second-object value, read at multiple call sites. |
| 0x28 | Signed dimension used in horizontal size selection; read again later for the geometry gate and final arguments. |
| 0x2c | Signed dimension used in vertical size selection; conditionally read again later for the geometry gate and final arguments. |
| 0x48 | One-byte flag; zero and nonzero select opposite complementary-value assignments. |
| 0x50 | Address of the first control record. |
| 0x54 | Address of the second control record. |
| 0x58 | Address of the third control record. |
| 0x5c | Address of the fourth control record. |
| 0x60 | Writable first comparison/cache word. |
| 0x64 | Writable second comparison/cache word. |
| 0x68 | Writable third comparison/cache word. |
| 0x6c | Writable fourth comparison/cache word. |

Each control record must permit a 32-bit read at offset 4. No other control-record layout is established.

The object fields and the fields at 0x18 and 0x1c are passed as opaque words. Their pointer status, ownership, and interpretation are unknown.

### Arithmetic conventions

The visible operations use 32-bit quantities, arithmetic right shifts for signed values, logical shifts for explicitly unsigned values, and signed comparisons where specified below.

For ordinary, nonoverflowing inputs, the arithmetic descriptions below are exact. Bit masks and explicitly unsigned additions also have direct 32-bit interpretations. Overflow-sensitive signed expressions require the qualification in **Uncertainties**; the evidence does not provide a portable source-language overflow contract.

Signed division by 2 and the signed quarter-dimension calculation truncate toward zero. This matters for negative values.

## Behavior

### 1. Sample all four controls

Before calling any dependency, the function reads the four control values in this order:

1. Read the address at 0x50, then read its word at offset 4.
2. Repeat through the address at 0x54.
3. Repeat through the address at 0x58.
4. Repeat through the address at 0x5c.

These produce four local 32-bit samples. All four samples are taken even if the first comparison subsequently requests an update.

No later direct reread of the control records is shown.

### 2. Evaluate the change gate

The function calls F-1d1b465375c3 with the address of the first local sample and the address of the cache word at 0x60.

If its return is zero, it makes the corresponding call for the second sample and 0x64. It continues similarly through the third and fourth samples only while previous returns are zero.

Consequently:

- A nonzero result from any comparison enters the update path immediately.
- Comparisons after the first nonzero result are not called.
- Four zero results skip the entire update path and return 1.
- Skipping the update path does not imply absence of effects: comparison dependencies have already run and receive writable addresses.

The dependency is not established to be a pure comparison. A called dependency can modify the sample or cache word whose address it receives. Subsequent behavior uses the local sample values that remain after these calls, not an independently preserved original copy.

### 3. Select provisional dimensions

The first local sample supplies a size-related quantity:

- Arithmetic-shift it right by 16.
- Clear the lowest two bits using the 32-bit mask 0xfffffffc.

Call the resulting signed bit pattern the horizontal target. The vertical target is obtained by multiplying that bit pattern by 3 and then arithmetic-shifting the 32-bit result right by 2.

The first stored dimension, read from 0x28, selects the provisional horizontal size:

- If four times the stored dimension is signed-less-than the horizontal target, use four times the stored dimension.
- Otherwise, compute the stored dimension divided by 4 with truncation toward zero, and choose the greater of that result and the horizontal target under signed comparison.

The second stored dimension, read from 0x2c, selects the provisional vertical size by the same rule using the vertical target.

For nonnegative dimensions without overflow, each target is therefore constrained between one quarter and four times its associated stored dimension. This concise interpretation is not a substitute for the signed rules on negative or overflow-sensitive inputs.

The two dimension reads occur at different points: the first precedes horizontal selection, and the second follows it.

### 4. Compute placement and clipping

The fourth sample supplies the vertical anchor; the third supplies the horizontal anchor. Each anchor is formed by adding 0xffff to its sample and arithmetic-shifting the result right by 16.

For nonoverflowing values, this is ceiling conversion from signed 16.16-style units to integer units. The fixed-point interpretation is inferred; the arithmetic is established.

The function prepares horizontal and vertical placement values and two clipping amounts, initially zero.

For the horizontal axis:

- Compute half the provisional horizontal size, truncating toward zero.
- If the horizontal anchor is at least this half-size, horizontal placement is the anchor minus the half-size. Horizontal clipping remains zero.
- Otherwise, horizontal placement remains zero.
- In that latter case, compute the positive gap between the half-size and anchor, call F-870c4fcf7295 with the gap multiplied by the initially read first stored dimension and the provisional horizontal size, and retain its return as the horizontal clipping amount.
- Reduce the provisional horizontal size by the gap, not by the dependency’s return value.

For the vertical axis:

- Compute half the provisional vertical size, truncating toward zero.
- If the vertical anchor is at least this half-size, vertical placement is the anchor minus the half-size. Vertical clipping remains zero.
- Otherwise, vertical placement remains zero.
- Compute the gap, call F-870c4fcf7295 with the gap multiplied by the initially read second stored dimension and the provisional vertical size, and retain its return as the vertical clipping amount.
- Reduce the provisional vertical size by that gap.

Horizontal clipping, if needed, is calculated before vertical clipping. There is no visible zero-denominator guard before either dependency call. The dependency’s arithmetic and failure behavior are unknown; it must not be assumed to perform division merely because the arguments resemble a scaling ratio.

### 5. Apply complementary 8-bit values

From the second local sample, the function forms an unsigned quantity by:

- Adding 0x7fff using unsigned 32-bit arithmetic.
- Masking the result with 0x00ffffff.
- Logically shifting right by 16.

The result is always between 0 and 255. This is masking and extraction, not saturation. Values outside an apparent nominal input range can wrap into that interval.

The flag at 0x48 is then read:

- If zero, call F-460b2abbbbbe for the first object with the extracted value, followed by a call for the second object with 255 minus that value.
- If nonzero, call the first object with 255 minus the extracted value, followed by the second object with the extracted value.

The first-object value is read before its call. The second-object value is read after that call. The implementation therefore does not establish that both calls use object values captured before either dependency runs.

### 6. Gate the geometry operation

After both complementary-value calls, the function rereads the signed dimension at 0x28.

It tests the **vertical clipping amount** against this first dimension:

- If vertical clipping is greater than or equal to the first dimension, suppress the geometry operation.
- Otherwise, read the signed dimension at 0x2c and test the **horizontal clipping amount** against that second dimension.
- If horizontal clipping is greater than or equal to the second dimension, suppress the geometry operation.

These are signed comparisons. The visible overflow-flag expression implements signed less-than-or-equal; it must not be replaced with a test based only on the sign of a potentially overflowing subtraction.

On suppression, the function calls F-460b2abbbbbe for the second object with zero. This occurs after that object has already received its complementary value.

Otherwise, F-a05963f65233 receives nine arguments, in order:

1. The second-object value, freshly read from 0x24.
2. Horizontal placement.
3. Vertical placement.
4. Vertical clipping amount.
5. Horizontal clipping amount.
6. Adjusted horizontal size.
7. Adjusted vertical size.
8. The reread first dimension minus horizontal clipping.
9. The reread second dimension minus vertical clipping.

The cross-axis ordering is established: the gate compares vertical clipping with the first dimension and horizontal clipping with the second, while the final two differences subtract horizontal clipping from the first dimension and vertical clipping from the second. A contract must preserve this distinction rather than normalize it to a conventional rectangle model.

### 7. Perform final object calls and cache writes

Whether geometry was applied or suppressed, the update path next calls:

1. F-1d891874186d with the first-object value and the word at 0x1c.
2. F-1d891874186d with the second-object value and the word at 0x18.

The second call’s fields are read after the first call. Evaluation order between the two field reads within an individual call is not independently established.

Finally, the four local samples are written to 0x60, 0x64, 0x68, and 0x6c, in that order. These writes occur after all update-path dependency calls.

The function then returns 1.

## Side effects

Direct record writes by this function are the four final cache writes on the update path. They are not performed on the all-zero comparison path.

Dependencies can have additional effects. In particular:

- F-1d1b465375c3 receives writable addresses into both local storage and the state record.
- Other dependencies receive opaque values whose reachable state and aliasing relationships are unknown.
- No dependency is established to preserve global state or record fields.
- Later field reads can therefore observe changes made during earlier calls.

The function visibly performs no allocation, deallocation, ownership transfer, or explicit retention. That does not establish the absence of such behavior inside dependencies.

Pointers to local samples are valid during the function invocation. Whether F-1d1b465375c3 retains them is unknown; no safe post-return lifetime is established.

Updates are sequential, not transactional. There is no rollback if a dependency fails to return after earlier effects have occurred.

## Dependencies

| Opaque ID | Established use | Not established |
|---|---|---|
| F-1d1b465375c3 | Up to four ordered, short-circuited calls; receives a local-sample address and cache-word address; zero continues comparison, nonzero triggers updating. | Comparison semantics, mutation rules, retention, error conventions. |
| F-870c4fcf7295 | Up to two calls for clipping calculations; two 32-bit arithmetic arguments; return becomes a clipping amount. | Division semantics, rounding, zero handling, overflow handling, side effects. |
| F-460b2abbbbbe | Two complementary-value calls on every update; an optional additional zero-valued call for the second object. | Meaning of the scalar, object ownership, failure behavior. |
| F-a05963f65233 | One nine-argument geometry-like call when both signed gates permit it. | Argument units, validation, rendering effects, ownership. |
| F-1d891874186d | Two ordered final calls pairing each object with another stored word. | Whether it binds, copies, submits, retains, or releases anything. |

The visible function does not inspect returns from the object-update dependencies.

## Acceptance examples

These examples are conditional static expectations, not runtime results.

### No change reported

If all four comparison calls return zero and return normally:

- All four control samples have already been read.
- The comparisons occur in sample order.
- No size, clipping, complementary-value, geometry, or final object call follows.
- No direct final cache writes occur.
- The return is 1.

Comparison-induced writes remain possible.

### First comparison requests an update

If the first comparison returns nonzero:

- The remaining three comparisons are omitted.
- Their control samples still exist because sampling preceded the first comparison.
- On normal completion, all four samples are written to the caches, including those whose comparisons were omitted.

### Centered, unclipped case

Assume dependencies do not mutate relevant fields or samples, and the comparison sequence requests an update. Let the first and second stored dimensions be 100 and 80. Let the first sample represent 80 whole units, the second be zero, and the third and fourth represent 50 and 40 whole units.

The provisional sizes are 80 and 60. Half-sizes are 40 and 30, so both placements are 10 and both clipping amounts are zero. F-870c4fcf7295 is not called.

With a zero flag, complementary values are 0 for the first object and 255 for the second. The geometry gate passes. F-a05963f65233 receives the second object followed by:

10, 10, 0, 0, 80, 60, 100, 80.

Final object calls and all four cache writes follow.

### Both clipping calculations required

Under the same nonmutation assumptions, retain dimensions 100 and 80 and a first sample representing 80 units, but use zero for both anchor samples.

The horizontal clipping dependency receives 4000 and 80. The vertical clipping dependency receives 2400 and 60. The adjusted sizes are 40 and 30.

If the dependency returns 50 and 40 respectively, the signed gate passes. The geometry arguments after the object are:

0, 0, 40, 50, 40, 30, 50, 40.

This example exposes the established clipping-argument order.

### Geometry suppressed at equality

If vertical clipping equals the later first dimension, the geometry operation is suppressed without reading the second dimension for the second gate. The second object receives an additional zero-valued call. Final object calls and cache writes still occur.

### Nonzero polarity flag

Any nonzero flag byte selects the same branch. For an extracted value of 64, the first object receives 191 and the second receives 64, before any possible suppression call.

## Uncertainties

- **documented_only — domain:** Valid ranges for dimensions, anchors, and samples are not supplied. Negative values are not rejected.
- **documented_only — signed overflow:** Several additions, subtractions, multiplications, and left shifts are overflow-sensitive. The static representation exposes 32-bit operations but does not establish portable language semantics for every out-of-range signed expression.
- **documented_only — dependency behavior:** No dependency implementation or contract is supplied. Arithmetic, mutation, ownership, retention, and error semantics remain unknown except where the caller’s use establishes them.
- **documented_only — geometry meaning:** Axis names describe dataflow relationships, not a proven external coordinate system.
- **documented_only — concurrency and aliasing:** No synchronization or alias restrictions are visible. Stable snapshots of the entire record are not provided.
- **Inferred intent:** Size scaling, anchor placement, clipping, and complementary visual weighting are plausible interpretations, not verified API meanings.
- No runtime tests have been performed.

## Evidence

Established behavior is derived solely from the supplied static representation of F-6a1cae7aa2bf: field accesses, 32-bit arithmetic, signed comparisons, short-circuit dependency calls, argument order, direct writes, and the constant normal return.

No implementation beyond that representation, dependency body, runtime trace, external documentation, or test result was available. The acceptance examples are arithmetic consequences under their stated conditions, not observed executions.
