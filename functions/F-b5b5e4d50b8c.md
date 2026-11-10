<!-- Trusted provenance: F-b5b5e4d50b8c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b5b5e4d50b8c

## Purpose

F-b5b5e4d50b8c conditionally updates a dependent object using four sampled values, two context dimensions, and a mode byte. It checks the sampled values against stored state through dependency calls. If any check returns nonzero, it computes geometry and an 8-bit control value, updates the dependent object, invokes a final association operation, and stores the four sampled values.

The apparent intent is a change-driven visual update involving scaling, positioning, clipping, and opacity. Those semantic labels are inferred. The established behavior is the ordered arithmetic, calls, reads, and writes described below; the supplied evidence does not establish what the dependencies render or otherwise implement.

The function returns the 32-bit value **1** on every observed normal-return path. This is not evidence that every dependency operation succeeds.

## Interface

### Input

The sole argument identifies a context record. The supplied representation uses 32-bit address-bearing fields and 32-bit integer arithmetic.

The following context locations are accessed. Offsets are byte offsets from the context address.

| Offset | Accessed representation | Observed role |
|---|---|---|
| `0x18` | 32-bit value | Second argument to the final association dependency |
| `0x24` | 32-bit value | First argument to each dependent-object operation |
| `0x28` | Signed 32-bit integer | First dimension |
| `0x2C` | Signed 32-bit integer | Second dimension |
| `0x48` | One byte | Zero/nonzero selector for complementing the control value |
| `0x50` | Address-bearing 32-bit value | Reference to the first sampled-value record |
| `0x54` | Address-bearing 32-bit value | Reference to the second sampled-value record |
| `0x58` | Address-bearing 32-bit value | Reference to the third sampled-value record |
| `0x5C` | Address-bearing 32-bit value | Reference to the fourth sampled-value record |
| `0x60` | 32-bit storage | First comparison-state slot |
| `0x64` | 32-bit storage | Second comparison-state slot |
| `0x68` | 32-bit storage | Third comparison-state slot |
| `0x6C` | 32-bit storage | Fourth comparison-state slot |

Each sampled-value reference is dereferenced at byte offset `+4` to obtain a 32-bit value. No other portion of those referenced records is inspected directly.

For this contract:

- **A** denotes the first sampled value.
- **B** denotes the second sampled value.
- **X** denotes the third sampled value.
- **Y** denotes the fourth sampled value.
- **W** and **H** denote the initial dimension reads from `0x28` and `0x2C`.

These are explanatory names, not additional interface fields.

### Output and preconditions

The normal return value is always 1. Observable outputs also include dependency calls, possible dependency effects, and direct writes to the four comparison-state slots.

The function performs no explicit null checks, bounds checks, alignment checks, or validation of dimension ranges. Normal execution requires all actual accesses to be valid and all invoked dependencies to return normally.

No ownership transfer, allocation, or release is explicit in this function. Passing a value or address to a dependency does not establish that dependency’s ownership or retention policy.

## Behavior

### 1. Sample all four source records

The function first reads A, B, X, and Y, in that order. Each sample requires reading its context reference and then reading the referenced record at `+4`.

All four samples occur before the first comparison call. Consequently, skipping later comparisons does not skip their initial sampling.

These samples are kept in local 32-bit storage. The comparison dependency receives addresses of that storage and may modify it; subsequent calculations and final state writes use the resulting local values.

### 2. Perform short-circuited comparison calls

F-1d1b465375c3 is called with:

1. The address of A’s local storage and the address of context slot `0x60`.
2. If the first result is zero, the corresponding addresses for B and `0x64`.
3. If both preceding results are zero, the corresponding addresses for X and `0x68`.
4. If all three preceding results are zero, the corresponding addresses for Y and `0x6C`.

Any nonzero result selects the update path immediately. Later comparison calls are omitted.

If every executed comparison returns zero and all four comparisons are reached, the function returns 1 without the geometry calculations, dependent-object operations, final association operation, or four direct state writes described below.

That path is not proven side-effect-free: the comparison calls receive writable addresses, including context-state addresses.

### 3. Derive and constrain nominal dimensions

On the update path, the function derives a horizontal target by:

- Arithmetic-right-shifting A by 16 bits.
- Clearing the lowest two bits of that result.

Thus the horizontal target is a signed 32-bit value divisible by four. For nonnegative values, this discards the fractional portion and rounds the remaining integer down to a multiple of four.

The vertical target is formed by multiplying the horizontal target by three in 32-bit arithmetic, interpreting that product as signed, and arithmetic-right-shifting by two.

The function then reads W from context offset `0x28` and selects a nominal horizontal extent:

- If four times W is signed-less-than the horizontal target, select four times W.
- Otherwise, calculate W divided by four with truncation toward zero, and select the greater of that quotient and the horizontal target.

It next reads H from `0x2C` and applies the same selection rule to H and the vertical target, producing the nominal vertical extent.

For ordinary positive dimensions without overflow, this behaves like constraining each target to the interval from one quarter to four times its corresponding context dimension. That interval interpretation is inferred and does not replace the actual signed comparisons for negative or overflowing inputs.

### 4. Derive anchors and clipping values

The horizontal anchor is obtained by adding `0xFFFF` to X in 32-bit arithmetic and arithmetic-right-shifting by 16. The vertical anchor is obtained identically from Y. For non-overflowing signed fixed-point inputs, these expressions act like rounding upward to an integer.

The addition for the vertical anchor is evaluated before the addition for the horizontal anchor.

The function initializes two placement offsets and two source-adjustment values to zero.

#### Horizontal calculation

Let the horizontal half-extent be the nominal horizontal extent divided by two, truncating toward zero.

- If the horizontal anchor is at least this half-extent, the horizontal placement offset becomes the anchor minus the half-extent. The horizontal source adjustment remains zero, and the horizontal extent is unchanged.
- If the horizontal anchor is smaller, the horizontal placement offset remains zero. The deficit is the half-extent minus the anchor. F-870c4fcf7295 receives the 32-bit product of that deficit and W as its first argument, and the original nominal horizontal extent as its second argument. Its returned value becomes the horizontal source adjustment. The horizontal extent is then reduced by the deficit.

The numerator multiplication is a 32-bit operation. The dependency is not established to perform division, despite the apparent scaling relationship.

#### Vertical calculation

The vertical half-extent is the nominal vertical extent divided by two, truncating toward zero.

- If the vertical anchor is at least the vertical half-extent, the vertical placement offset becomes their difference. The vertical source adjustment remains zero, and the vertical extent is unchanged.
- If the vertical anchor is smaller, F-870c4fcf7295 receives the 32-bit product of the deficit and H, followed by the original nominal vertical extent. Its returned value becomes the vertical source adjustment. The vertical extent is reduced by the deficit, and the vertical placement offset remains zero.

The horizontal dependency call, when needed, occurs before the vertical one.

No direct guard prevents zero or negative extents, negative products, or an argument combination that the arithmetic dependency might reject.

### 5. Compute and submit the control value

The function derives a value from B by:

1. Adding `0x7FFF` using 32-bit unsigned arithmetic.
2. Masking the result with `0x00FFFFFF`.
3. Shifting right by 16 bits.

The result is between 0 and 255 inclusive. The mask discards bits above bit 23; this is not saturation or clamping.

The mode byte at context offset `0x48` is then read:

- Nonzero leaves the value unchanged.
- Zero replaces it with 255 minus the value.

The function reads the object value at `0x24` and calls F-460b2abbbbbe with that value and the resulting control value.

This call occurs before the later dimension-validity decision. Even when geometry is subsequently rejected, the computed control value has already been submitted.

### 6. Re-read dimensions and select the geometry operation

After the initial control-value call, the first dimension is read again from `0x28`. Call this value **W′**.

The validity check deliberately uses a crossed pairing:

- If the **vertical source adjustment is greater than or equal to W′**, geometry is rejected. The second dimension is not read for this check.
- Otherwise, the second dimension is read from `0x2C`; call it **H′**. Geometry is rejected if the **horizontal source adjustment is greater than or equal to H′**.

These are signed comparisons. The observed subtraction/overflow checks implement signed less-than-or-equal decisions rather than merely testing the sign bit of a wrapped difference.

If geometry is rejected, the function calls F-460b2abbbbbe again with a freshly read object value from `0x24` and control value zero. F-a05963f65233 is not called.

If geometry is accepted, F-a05963f65233 is called with nine arguments, in this order:

1. A freshly read object value from `0x24`.
2. Horizontal placement offset.
3. Vertical placement offset.
4. Vertical source adjustment.
5. Horizontal source adjustment.
6. Possibly reduced horizontal extent.
7. Possibly reduced vertical extent.
8. W′ minus the horizontal source adjustment.
9. H′ minus the vertical source adjustment.

The crossed validity checks do **not** prove that the final two arguments are positive. The checked pairings and the subtraction pairings differ. This distinction is part of the established behavior and must not be silently normalized.

### 7. Associate and store state

After either the accepted or rejected geometry path, the function calls F-1d891874186d with the values read from context offsets `0x24` and `0x18`.

The available evidence does not establish the relative evaluation order of these two argument reads. Both occur for this call after the preceding geometry or zero-control operation returns.

After the association call returns, the function directly stores the current local values in this order:

1. A to `0x60`.
2. B to `0x64`.
3. X to `0x68`.
4. Y to `0x6C`.

It then returns 1.

There is no direct re-sampling of the source records before these stores. Changes made to those records by dependencies therefore do not automatically replace the already captured values.

## Side effects

Established direct effects are:

- Ordered reads of four referenced source values.
- Conditional reads of context dimensions and mode.
- Up to four comparison calls, with writable local and context addresses.
- Zero, one, or two calls to the arithmetic dependency.
- One initial control-value call on every update path.
- Either a second, zero-valued control call or one geometry call.
- One final association call on every update path.
- Four ordered 32-bit state writes after the association call.

There is no explicit allocation, deallocation, synchronization, rollback, or error-status propagation.

Dependency effects are not constrained by this body. In particular, the later reads of dimensions and object fields must not be replaced by assumptions that earlier calls preserve those fields. A dependency may make the later observed dimensions differ from W and H.

If a dependency fails to return normally, subsequent calls and state writes are not established to occur. The function contains no visible recovery mechanism.

## Dependencies

| Opaque ID | Established use | Not established |
|---|---|---|
| F-1d1b465375c3 | Receives two addresses; zero permits the next comparison, nonzero triggers updating | Equality semantics, mutations, ownership, retained addresses |
| F-870c4fcf7295 | Receives a 32-bit product and nominal extent; returns a source-adjustment value | Division semantics, zero-denominator behavior, signedness guarantees, side effects |
| F-460b2abbbbbe | Receives an object value and a control value in the range 0–255 | Whether this means opacity, visibility, or another property |
| F-a05963f65233 | Receives the nine geometry-related arguments listed above | Coordinate conventions, clipping guarantees, validation, rendering effects |
| F-1d891874186d | Receives the object value and the context value at `0x18` | Whether it attaches, commits, refreshes, transfers ownership, or performs another operation |

No dependency’s return status is explicitly used except the comparison results and the arithmetic results.

## Acceptance examples

These are conditional static examples, not executed tests. Unless stated otherwise, assume valid memory, normal dependency returns, no relevant dependency mutations, and no arithmetic overflow.

### All comparisons report zero

If all four comparison calls return zero:

- All four source values have already been sampled.
- All four comparison calls occur in order.
- No control, arithmetic, geometry, or association call occurs.
- No direct final state stores occur.
- The return value is 1.

### First comparison reports nonzero

If the first comparison returns nonzero:

- The other three comparisons are skipped.
- Their source values nevertheless remain available because sampling preceded comparison.
- The complete update path executes using the current local sample values.
- All four state slots are directly written after the association call.

### Centered positive geometry

Suppose:

- A represents 80 in signed 16.16 form.
- B is zero.
- X represents 40 and Y represents 30.
- W is 100 and H is 80.
- The mode byte is nonzero.

The selected extents are 80 by 60. Both placement offsets and both source adjustments are zero. No arithmetic dependency call is needed.

The initial control value is zero. Geometry is accepted, and its trailing four arguments are 80, 60, 100, and 80.

If only the mode byte changes to zero, the initial control value becomes 255.

### Both anchors require clipping

Keep the same dimensions and A, but let X represent 30 and Y represent 20.

Both deficits are 10. The arithmetic calls receive:

- First: 1000 and 80.
- Second: 800 and 60.

The resulting extents are 70 and 50. If those calls return 12 and 13 respectively, both placement offsets remain zero. The geometry call receives source adjustments in the order 13, 12, and its final two arguments are 88 and 67.

### Geometry rejection still submits control first

If the vertical source adjustment is at least W′:

- The computed control value is submitted first.
- The second dimension is not read for the validity check.
- A second control call submits zero.
- No geometry call occurs.
- Association and the four state stores still occur.

### The crossed check can accept a negative trailing value

Suppose the initial control call leaves W′ equal to 10 and H′ equal to 100, while the horizontal and vertical source adjustments are 20 and 5.

The checks accept geometry because 5 is less than 10 and 20 is less than 100. The final two geometry arguments are nevertheless −10 and 95. Replacing this behavior with same-axis validation would change the contract.

## Uncertainties

The following are **documented_only**:

- Exact public types, calling convention, and broader record layout.
- Semantic names and units for all sampled values and context fields.
- Dependency implementations, failure behavior, ownership, retention, and mutations.
- Required valid ranges for dimensions and arithmetic inputs.
- Whether aliasing between referenced records, state slots, and other context storage is permitted.
- Concurrency and reentrancy expectations.
- Intended behavior for overflowing signed arithmetic and negative dimensions.

The supplied representation exposes 32-bit operations, unsigned masking, arithmetic shifts, and signed comparisons. Ordinary-range examples are unambiguous. Extreme signed-overflow cases should not be promoted into a portable arithmetic guarantee without instruction-level evidence. In particular, the apparent scaling and fixed-point interpretations do not justify widening intermediate arithmetic, saturating results, or introducing validation.

## Evidence

This contract is based solely on the supplied static function body and its opaque dependency mapping. Established details include access offsets and widths, sampling and short-circuit order, arithmetic expressions, conditional calls, argument ordering, final stores, and the constant normal return.

Visual-update terminology, fixed-point interpretation, and the apparent three-to-four aspect relationship are inferred intent, not independently verified semantics.

No runtime tests have been performed. No dependency implementation or external interface documentation was available.
