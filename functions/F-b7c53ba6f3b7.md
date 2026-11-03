<!-- Trusted provenance: F-b7c53ba6f3b7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b7c53ba6f3b7

## Purpose

F-b7c53ba6f3b7 adds a scalar-weighted contribution from each of three source components to the corresponding destination component. It modifies caller-provided storage and returns no value.

The three destination replacements occur in component order. They are not a simultaneous update from an entry-time snapshot: overlapping regions can allow an earlier replacement to change a source value used by a later component.

**Inferred intent:** accumulation using a fixed-point scale of \(2^{16}\). This interpretation explains the arithmetic within the domain established below. Component meaning, units, and intended operating limits are unknown.

This contract distinguishes:

- **Established static evidence:** operations, access dependencies, and sequencing visible in the supplied body.
- **Conditional analytical guarantees:** results derived under explicitly stated integer and storage assumptions.
- **documented_only:** unresolved behavior for which the supplied material does not establish a requirement.

No runtime tests have been performed.

## Interface

### Inputs and output

| Input | Role |
|---|---|
| Destination address | Identifies three consecutive signed-integer components, each read and replaced. |
| Source address | Identifies three consecutive signed-integer components supplying the contributions. |
| Scalar | A signed-integer value used for all three components. |

There is no length, stride, capacity, mode, status result, or separate error output. Normal completion is observable through destination storage.

The scalar is passed by value. Subsequent changes to caller storage from which it originated do not change that invocation’s scalar.

The source address is not directly used as a write destination. Source-designated objects can nevertheless change when they also belong to the destination region.

### Widths and layout

The arithmetic guarantees in this contract assume:

- Components and scalar are signed 32-bit integers, with range \([-2^{31},2^{31}-1]\).
- Widened signed and unsigned arithmetic uses 64-bit values.
- The narrower unsigned arithmetic uses 32-bit values.

These widths are consistent with the supplied representation but are not independently verified target properties.

Components are consecutive integer objects, without a caller-selected stride. With four-byte integers and eight-bit bytes, their starting offsets are 0, 4, and 8 bytes; each region spans 12 bytes. Byte order and alignment requirements are **documented_only**.

### Preconditions and ownership

A defined invocation requires:

- Three live, readable, writable destination objects.
- Three live, readable source objects.
- Valid alignment, permissions, lifetimes, and access compatibility.
- Defined arithmetic and destination conversions for all encountered values.
- No prohibited concurrent interference.

No visible validation enforces these requirements.

The regions may share valid component objects. This does not establish support for arbitrary bytewise overlap, misaligned addresses, incompatible object access, or access outside live storage.

Storage remains caller-provided. No allocation, release, address retention, or ownership transfer is visible.

## Behavior

### Component ordering and observations

Destination replacements occur first component, then second component, then third component. Each replacement precedes the next component’s processing.

Each component depends on its existing destination value, the scalar, and two evaluations of its corresponding source component. All observations needed for a component precede that component’s replacement. Their relative order within the component is not established.

The two source evaluations are expression-level dependencies. They do not require two physical memory transactions when one observation can be reused without changing applicable behavior.

There is no complete-source snapshot before destination modification begins. Later source observations therefore reflect any applicable changes caused by earlier destination replacements.

The numerical guarantees below assume that both source evaluations for a component agree. The supplied material does not establish a supported situation in which they differ, or a portable result in that situation.

### Guaranteed numerical relationship

For one component, let:

- \(d\) be its destination value before replacement.
- \(s\) be the value shared by its two source observations.
- \(k\) be the scalar.
- \(p=ks\) be the mathematical product.

Under the stated integer model, a sufficient domain for defined contribution formation is:

\[
0 \le p < 2^{47}.
\]

Within this domain, the contribution is:

\[
q=\left\lfloor\frac{p}{65536}\right\rfloor,
\qquad 0\le q\le 2^{31}-1.
\]

Positive fractional contributions are discarded toward zero, equivalently downward for these nonnegative products. No rounding-to-nearest adjustment is present.

Accumulation has unsigned 32-bit semantics. The value presented for conversion into signed destination storage is:

\[
u=(d+q)\bmod 2^{32},
\qquad 0\le u<2^{32}.
\]

A negative destination value therefore participates through its unsigned conversion; this is not signed saturating addition.

If \(u\le 2^{31}-1\), the stored value is exactly \(u\). If \(u\ge 2^{31}\), the signed result, or any implementation-defined conversion signal, depends on the environment. Unsigned modular addition does not by itself establish a negative two’s-complement storage result.

The sufficient product domain is an analytical condition, not a checked input restriction. For overlapping regions, it applies to the source values actually encountered during processing, including values changed by earlier replacements.

### Arithmetic limits

Every product of two signed 32-bit values fits in signed 64-bit range. The largest positive product is \(2^{62}\), obtained when both operands equal \(-2^{31}\).

That multiplication bound does not establish valid contribution formation over the entire product range. The supplied arithmetic also includes a signed intermediate shift. Its validity depends on more than whether the widened product fits.

Accordingly:

- Negative products are outside the sufficient domain stated above.
- Products at or above \(2^{47}\) are outside that domain.
- Negative operands are not individually excluded: two negative operands can produce a supported positive product.
- Two’s-complement representation alone does not resolve otherwise invalid signed arithmetic.
- A mathematically small or zero low-width result does not establish that preceding operations were valid.

The stated domain is sufficient, not an independently proven characterization of every value supported by every possible environment.

A floor-scaled interpretation for negative products is plausible intent, but it is not an unconditional requirement. Similarly, a wrapping interpretation for larger products requires additional environmental evidence. These extensions remain **documented_only**.

No saturation, clamping, checked-overflow result, or alternate arithmetic-error path is visible.

### Overlap

For disjoint regions, destination writes do not alter source components through these regions.

For identical regions, each component uses its own value before its replacement. Earlier replacements affect different component objects and do not, through this overlap alone, change later source components.

For partial overlap, an earlier destination replacement can alter a source object needed later. The later contribution then depends on the altered value. A subsequent write cannot retroactively change an already consumed source observation.

Consequently, an implementation that always snapshots all source components before making any destination replacement would not preserve the contract for every valid overlap.

### Zero scalar

There is no explicit zero-scalar bypass. Source observations, destination observations, and destination replacements remain part of the expressed operation.

With valid storage and directly representable final conversions, a zero scalar leaves destination values numerically unchanged. Invalid source storage does not become acceptable merely because the mathematical contribution is zero.

This requirement concerns applicable program behavior, not a fixed number of physical memory accesses.

### Completion and errors

Normal, defined completion replaces all three destination components.

There is no returned error status, validation failure result, rollback, or transactional commit. Invalid storage or arithmetic is not translated into a function-level diagnostic.

The ordered replacements do not establish a recoverable partial-completion protocol. Undefined execution provides no guarantee that earlier writes remain observable. Storage state after a fault, conversion signal, interruption, or other exceptional execution is **documented_only**.

## Side effects

The explicit state changes are the three ordered destination replacements. Any valid alias to those objects can observe the resulting changes, including an alias through the source region.

No named-global access is visible. This does not imply that global state is unchanged: destination storage may itself be global storage.

No callable dependency, allocation, release, synchronization operation, or address retention is visible.

Neither an individual replacement nor the complete operation is established as atomic. Component ordering does not establish inter-thread synchronization, hardware ordering, or fault atomicity.

Physical access counts and widths, asynchronous visibility, and suitability for device memory are **documented_only**. This contract does not authorize unsynchronized concurrent access or infer volatile-access guarantees.

## Dependencies

No callable dependencies are visible, so there are no additional opaque function IDs to enumerate. No unknown callee behavior is needed to explain the visible operation.

The environmental dependencies are:

| Dependency | Required or unresolved property |
|---|---|
| Integer model | Actual widths, ranges, storage dimensions, and representation. |
| Arithmetic rules | Validity of signed intermediate operations, particularly outside the sufficient product domain. |
| Destination conversion | Result or signal when the accumulated unsigned value exceeds the signed maximum. |
| Storage model | Object lifetimes, alignment, permissions, compatible access, and valid overlap. |
| Execution environment | Concurrency restrictions, exceptional execution, and physical-memory behavior. |

These dependencies are not permission to assume wrapping arithmetic, preserved global state, or additional ownership behavior.

## Acceptance examples

These examples are conditional analytical deductions, not observed executions. Unless stated otherwise, they assume the integer model above, valid disjoint regions, matching source observations within each component, and no external interference.

### Ordinary accumulation

Destination \([10,20,30]\), source \([4,6,8]\), and scalar \(32768\) produce contributions \([2,3,4]\).

Required destination:

\[
[12,23,34].
\]

All products are within the sufficient domain, and all final conversions are directly representable.

### Fractional contributions

With destination \([0,0,0]\), source \([1,65535,65536]\), and scalar \(1\), the required destination is:

\[
[0,0,1].
\]

The first two contributions do not round upward.

For source \([-1,1,-65537]\) with the same destination and scalar, the negative products fall outside the sufficient domain. A result of \([-1,0,-2]\) is acceptable only under additional environmental rules establishing negative floor-scaled contributions and the necessary signed conversions. It is not an unconditional expected result.

### Supported products from negative operands

Destination \([0,0,0]\), source \([-2,-3,-4]\), and scalar \(-65536\) require:

\[
[2,3,4].
\]

The products are positive and within the sufficient domain. Operand sign alone is not the acceptance boundary.

### Zero scalar

Destination \([7,8,9]\), any valid source region, and scalar \(0\) require:

\[
[7,8,9].
\]

An invalid source region does not satisfy the preconditions.

For an initially negative destination value, numerical preservation additionally depends on the environment’s conversion of the corresponding high unsigned value back into signed storage.

### Identical regions

If both addresses designate the same three objects, initially \([2,3,4]\), scalar \(65536\) requires:

\[
[4,6,8].
\]

Each component uses its own pre-replacement value.

### Overlap carrying changes forward

Four consecutive backing objects initially contain:

\[
[1,2,3,4].
\]

If the source covers the first three objects and the destination covers the last three, scalar \(65536\) requires final backing contents:

\[
[1,3,6,10].
\]

The first replacement changes the source value used for the second component; the second replacement changes the source value used for the third. The snapshot-based outcome \([1,3,5,7]\) does not satisfy this case.

With the placement reversed—destination covering the first three objects and source covering the last three—the required backing contents are:

\[
[3,5,7,4].
\]

In this direction, earlier replacements do not change source objects still needed later.

### Destination-conversion boundaries

Destination \([-1,-1,-1]\), source \([1,1,1]\), and scalar \(65536\) require:

\[
[0,0,0].
\]

Each unsigned accumulation wraps to zero, whose signed conversion is directly representable.

A destination component of \(2147483647\) with contribution one instead produces unsigned value \(2147483648\). Requiring storage of \(-2147483648\) needs an environmental conversion rule establishing that result.

### Contribution-domain boundaries

Destination \([0,0,0]\), source \([65535,65535,65535]\), and scalar \(2147483647\) require:

\[
[2147450879,2147450879,2147450879].
\]

Each product is below \(2^{47}\).

A source component of \(-65536\) and scalar \(-2147483648\) produce a mathematical product exactly equal to \(2^{47}\). The sufficient-domain guarantee does not apply.

If both operands equal \(-2147483648\), their product is \(2^{62}\). A mathematically zero low-width residue does not establish a zero contribution without independently valid intermediate arithmetic.

## Uncertainties

- **documented_only — target model:** independently verified widths, storage dimensions, signed representation, byte order, alignment, and applicable language rules.
- **documented_only — extended arithmetic:** contribution formation for negative products and products at or above \(2^{47}\).
- **documented_only — signed storage conversion:** results or signals for accumulated unsigned values above the signed maximum.
- **documented_only — differing source observations:** supported circumstances and results when a component’s two evaluations disagree.
- **documented_only — physical effects:** access counts and widths, atomicity, hardware ordering, asynchronous visibility, and device-memory suitability.
- **documented_only — exceptional execution:** completion and storage state after faults, conversion signals, interruption, invalid arithmetic, or prohibited interference.
- **Inferred intent:** fixed-point accumulation with 16 fractional bits. Application meaning and intended operating limits remain unknown.

These uncertainties do not support an unrestricted wrapping, saturating, or transactional contract.

## Evidence

**Established static evidence:** three ordered component replacements; dependence on existing destination values; two source evaluations per component; widened signed multiplication; signed intermediate arithmetic; unsigned accumulation; and conversion into signed destination storage.

The supplied body contains no complete-source snapshot, explicit zero-scalar bypass, validation, saturation, callable dependency, allocation, release, address retention, ownership transfer, or rollback.

**Conditional analytical deductions:** the sufficient product domain, floor-scaled contributions within that domain, unsigned accumulation relationship, destination-conversion boundaries, and overlap examples follow under the stated integer and storage assumptions.

**Inferred intent:** the fixed-point interpretation follows from the numerical scaling, not from an independently established application requirement.

No runtime tests have been performed.
