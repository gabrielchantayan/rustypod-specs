<!-- Trusted provenance: F-ef5e44e7d81f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ef5e44e7d81f

## Purpose

F-ef5e44e7d81f conditionally publishes an attribute and a six-field record using values obtained from a provider.

A negative activation value disables publication. Otherwise, the function selects record storage, requests working data from an indirect dependency, derives an attribute, and publishes five components plus a scale. Depending on the acquired scale’s magnitude, the components either pass through unchanged or undergo five ordered calls to a numerical dependency.

Every established normal return is a 32-bit zero. This return value does not distinguish disabled operation from completed publication, nor does it certify that either dependency succeeded.

**Inferred intent:** the scale constant 65,536 and the final 16-bit shifts are consistent with fixed-point processing. No application-specific units, numerical formula, or normalization guarantee follows from that interpretation.

## Interface

### Inputs and result

The boundary exposes two 32-bit address-bearing inputs:

- **Destination:** identifies storage containing an attribute and a base address for records.
- **Provider:** identifies storage containing an activation/selection field and an indirect acquisition target.

These roles do not establish complete object types, allocation sizes, ownership, or non-overlap.

Normal completion produces the 32-bit value zero. No alternative error result is established.

### Externally addressed fields

Offsets below are byte offsets from the corresponding input address.

| Storage | Offset | Width | Role |
|---|---:|---:|---|
| Provider | 0x60 | 32 bits | Signed activation value; independently read again for record selection |
| Provider | 0x34 | 32 bits | Indirect acquisition target |
| Destination | 0x130 | 32 bits | Record-storage base address |
| Destination | 0x44 | 16 bits | Published attribute |

Each selection unit contributes 252 bytes to the displacement from the record-storage base. This spacing does not establish an allocation extent or a permissible index range.

The working data consists of five components and a scale. “Component 1” through “component 5” are descriptive labels only; they do not specify the acquisition buffer’s physical layout.

The selected record receives:

| Record-relative offset | Width | Content |
|---|---:|---|
| 0xD8 | 32 bits | Component 1 |
| 0xDC | 32 bits | Component 3 |
| 0xE0 | 32 bits | Component 2 |
| 0xE4 | 32 bits | Scale |
| 0xE8 | 32 bits | Component 4 after a signed right shift by 16 |
| 0xEC | 32 bits | Component 5 after a signed right shift by 16 |

The scale and the two components subject to shifting have signed 32-bit interpretations at those operations. The numerical signedness of components 1–3 is not established.

### Conditions for defined completion

A disabled invocation needs only a valid activation-field read. It does not require usable destination storage.

An enabled invocation additionally requires:

- Valid storage for every reached read and write.
- A callable acquisition target.
- Defined working values before each consumption.
- Supported address arithmetic and scale-magnitude arithmetic.
- Dependencies that return normally.
- Continued validity of the selected record storage through publication.

The function does not explicitly initialize the working data before acquisition. Passing an address associated with component 1 does not prove that acquisition initializes all five components and the scale. It also does not establish their adjacency, writable extent, or correspondence to positions in a buffer.

There are no explicit null, bounds, alignment, initialization, or dependency-status checks.

## Behavior

### Disabled and enabled outcomes

Activation uses a signed 32-bit observation of the provider field at offset 0x60.

Every negative value disables the operation, including −2,147,483,648. A disabled invocation returns zero without accessing the destination, reading the acquisition target, calling either dependency, or explicitly writing output.

Zero and positive activation values enable the operation. Selection then uses a separate observation of the same provider field. The activation observation is not a retained selection snapshot.

Consequently, selection can differ from activation. A negative selection observed after nonnegative activation is not rejected by a second sign test.

### Record selection and acquisition

Before acquisition, the function determines the selected record address from the observed destination base and the independently observed selection, using 252 bytes per selection unit. The relative order of those two reads is not established.

The resulting address is retained for later publication. Changes to the destination’s stored base or to the provider’s selection field during acquisition do not themselves redirect this invocation’s record writes. Such changes can nevertheless affect validity, ownership, or overlapping storage.

The address calculation uses represented 32-bit integer values. Wider intermediates, checked arithmetic, saturation, and defined wrapping are not established. Valid selection bounds and outcomes for overflowing calculations remain **documented_only**.

Acquisition is performed through F-indirect-1 after record selection. Its return value is ignored. The subsequent computation therefore depends on the working values being defined, not on a checked acquisition status.

### Scale policy

The acquired scale determines a retained factor:

- For a nonnegative scale, the factor has the same value.
- For a negative scale whose positive magnitude is representable in signed 32 bits, the factor is that magnitude.

Deriving the factor does not itself replace the scale. A negative scale can therefore remain negative while the factor is positive.

For −2,147,483,648, the positive magnitude is not representable in signed 32 bits. Behavior at that negation boundary remains **documented_only**; no wrapping result or continued execution is required by this contract.

For supported factor derivation, there are two modes:

- **Bypass:** factor exactly 65,536. No component-conversion calls occur, and the function does not explicitly replace the scale.
- **Conversion:** every other factor, including zero. All five components undergo conversion, after which the scale is explicitly replaced with 65,536.

Both positive and negative acquired scales of magnitude 65,536 select bypass. Bypass does not eliminate attribute computation or the final shifts of components 4 and 5.

The factor is retained for later use. A subsequent change to the working scale alone does not trigger factor recomputation.

### Attribute production

Attribute production uses two calls to F-73d099a86adb. The first receives the retained factor followed by 1000. The second receives 65,536 followed by the full 32-bit result of the first call.

Only the low 16 bits of the second result are written to the destination attribute. This is truncation of the returned bit pattern, not saturation or range validation. The intermediate result is not first narrowed to 16 bits.

Both numerical calls occur after acquisition. The attribute write occurs before any component-conversion call and before any explicit record write.

No multiplication, division, reciprocal, rounding, or error interpretation is established for these calls.

### Component treatment and publication

In conversion mode, F-73d099a86adb processes components 1 through 5 in that order. Each call receives the corresponding component’s then-current 32-bit value as its first argument and the retained factor as its second argument. Its returned 32-bit value replaces that component before the next conversion call.

These inputs are not an established simultaneous snapshot. Nor is a converted result guaranteed to remain unchanged until publication if a later dependency can reach its storage.

After the fifth conversion, the function explicitly sets the scale to 65,536. No dependency call intervenes between this replacement and record publication.

In bypass mode, neither these five conversions nor the explicit scale replacement occurs.

Record writes have the following required order:

1. Component 1 at 0xD8.
2. Component 2 at 0xE0.
3. Component 3 at 0xDC.
4. Scale at 0xE4.
5. Shifted component 4 at 0xE8.
6. Shifted component 5 at 0xEC.

Thus the write at 0xE0 precedes the write at 0xDC despite their address order.

Each publication consumes its working value individually. There is no established six-value publication snapshot. If permitted overlap lets an earlier write alter a later-consumed value, that later publication can reflect the alteration. This qualification includes the scale after its explicit replacement in conversion mode.

Components 4 and 5 are each interpreted as signed 32-bit values and shifted right by 16 before publication. For nonnegative values, this discards the remainder from division by 65,536. Results for negative values depend on the environment’s signed-right-shift convention and remain **documented_only**.

After all six record writes, normal completion returns zero.

## Side effects

A normally completed enabled invocation explicitly publishes one 16-bit attribute and six 32-bit record fields. Conversion mode also replaces the five working components and the scale.

Dependencies may have additional effects. Neither dependency is established to preserve globals, provider storage, destination storage, working data, selected record storage, or allocation lifetimes. Lack of an explicit pointer argument to a region is not evidence that the dependency cannot reach it.

No direct allocation, release, or ownership transfer is established for F-ef5e44e7d81f itself. Acquisition receives an invocation-local address, but that does not establish permission to retain or use it after the invocation. Dependency retention and release behavior remain **documented_only**.

The storage regions are not established to be disjoint. Where overlap is permitted, an attribute write can affect later inputs, and a record write can affect a later-consumed working value. Such effects do not change the required explicit access order.

Publication is not transactional:

- Before the attribute write, dependency effects may already have occurred.
- After the attribute write but before record publication, the destination may contain a new attribute with unchanged record fields.
- Interruption during record publication may leave only a completed prefix of the six writes.

No rollback, recovery path, completion marker, or interpretation of a dependency result as failure is established. Ordered writes do not imply atomicity, synchronization, or cross-thread visibility.

## Dependencies

### F-indirect-1

F-indirect-1 denotes the opaque acquisition target obtained from provider offset 0x34. Different provider contents may identify different implementations.

It receives four arguments, in order:

1. Provider address.
2. Integer 6.
3. An invocation-local address associated with component 1.
4. Integer 3.

The return value is ignored.

The constants 6 and 3 do not independently establish an element count, operation selector, data format, or writable extent. Initialization coverage, declared argument types, status conventions, ownership, and external effects remain **documented_only**.

### F-73d099a86adb

F-73d099a86adb accepts two 32-bit values and supplies a 32-bit result. Its arithmetic semantics and numerical signedness are unresolved.

For normally returning invocations with supported factor derivation:

| Outcome | Acquisition calls | Numerical calls |
|---|---:|---:|
| Disabled | 0 | 0 |
| Enabled bypass | 1 | 2 |
| Enabled conversion | 1 | 7 |

The first two numerical calls produce the attribute. The five additional calls in conversion mode occur after the attribute has been written.

No overflow rule, rounding rule, zero-argument behavior, error encoding, or memory-preservation guarantee is established. Returned values are consumed without an explicit success check.

## Acceptance examples

These examples are conditional obligations from static evidence, not observed executions. Unless stated otherwise, enabled cases assume valid reached storage, defined consumed values, supported arithmetic, and normally returning dependencies.

### Disabled operation with an unusable destination

If activation observes −1, the function returns zero without destination access or dependency calls. An unusable destination alone does not invalidate this path. The same rule applies to every negative signed 32-bit activation value.

### Selection changes after activation

If activation observes zero but the selection observation yields one, the chosen record is 252 bytes beyond the observed base, assuming valid arithmetic and storage.

Changing the stored base during acquisition does not redirect publication. Releasing the already selected allocation instead violates the requirement that reached storage remain valid.

A negative selection following nonnegative activation receives no additional sign-based rejection. Whether the resulting address is valid is a separate question.

### Equal magnitudes, different retained scales

Assume dependencies preserve the working values and output writes do not alter later-consumed storage.

An acquired scale of 65,536 requires bypass and publication of scale 65,536. A scale of −65,536 also requires bypass, but its published scale remains −65,536.

Each case requires two numerical calls, no component-conversion calls, and the final shifts of components 4 and 5.

### Conversion from a negative scale

For a scale of −131,072 at factor derivation, the retained factor is 131,072.

The attribute is published before the five conversions begin. Every conversion receives 131,072 as its second argument; first arguments use the then-current components in component order.

The scale is explicitly replaced with 65,536 before record publication. Its published value is 65,536 provided intervening overlapping writes do not change the consumed storage. Exact converted component values cannot be required without the numerical dependency’s contract.

### Width and ordering boundaries

If the final numerical result for the attribute has bit pattern 0x12345678, the attribute receives 0x5678. This specifies retained bits, not byte order.

If component 4 is 131,071 when consumed for publication, its shifted output is 1.

If that component is −65,537, an output of −2 is required only under a sign-propagating right-shift convention.

Regardless of component values, the component-2 write at 0xE0 precedes the component-3 write at 0xDC.

### Zero, unsupported magnitude, and interruption

A zero scale selects conversion. Each component-conversion call receives zero as its second argument. Numerical success is not established.

A scale of −2,147,483,648 reaches the unresolved signed-negation boundary. This contract specifies no resulting factor or subsequent publication for that case.

If the attribute write completes but the first component-conversion call does not return, explicit record publication has not begun. Restoration of the attribute is not required.

## Uncertainties

The following remain **documented_only**:

- Application meaning, units, and complete object types.
- Acquisition layout, writable extent, and initialization coverage.
- Dependency arithmetic, declared signedness, error conventions, and preservation behavior.
- Minimum-scale negation and overflowing address calculations.
- Negative signed-right-shift results.
- Byte order, alignment requirements, allocation extents, and valid selection bounds.
- Actual overlap and dependency reachability.
- Retention, release, ownership, and allocation-lifetime effects.
- Reentrancy, concurrent mutation safety, synchronization, and observer visibility.

A zero return does not resolve these uncertainties or independently establish valid numerical results.

## Evidence

**Established static evidence:** the supplied representation supports the input and output widths, addressed offsets, signed activation gate, separate selection observation, 252-byte selection spacing, and selection before acquisition. It also supports the acquisition arguments, ignored acquisition result, lack of explicit working-data initialization, magnitude-dependent mode selection, attribute narrowing, conversion order, scale replacement, publication order, and signed final shifts.

**Inferred intent:** fixed-point normalization is consistent with the constants and shifts. This interpretation adds no arithmetic, layout, ownership, initialization, or preservation guarantee.

**Evidence limits:** dependency implementations, complete declarations, and runtime conditions are unavailable. The boundary alone does not establish acquisition coverage, valid memory extents, dependency side effects, overflow outcomes, or negative-shift results.

**Verification status:** static analysis only. No runtime tests have been performed.
