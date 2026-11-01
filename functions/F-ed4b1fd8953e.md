<!-- Trusted provenance: F-ed4b1fd8953e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ed4b1fd8953e

## Purpose

F-ed4b1fd8953e maps a selector to a positive numeric adjustment and applies that adjustment to a supplied base. An unsupported selector returns zero.

Under the conditional 32-bit integer model defined below, the supported selectors are **32,796 through 32,809**, inclusive. Their adjustments range from **12 through 168**, increasing by 12 for each successive selector.

This is a numeric contract. Neither the inputs nor the result are established as addresses, handles, object references, sizes, or storage offsets.

**Inferred intent:** The regularly spaced adjustments suggest selection relative to a base with a fixed stride. This does not establish units, storage layout, backing objects, or permission to access memory.

## Interface

The function accepts two signed integer values and returns one signed integer value.

| Component | Meaning |
|---|---|
| Base | Starting value to which a supported selector’s adjustment applies. |
| Selector | Determines whether an adjustment is available and, if so, its magnitude. |
| Result | Adjusted base within the guaranteed arithmetic domain, or zero for an unsupported selector. |

There are no explicit buffers, lengths, callbacks, resource references, output parameters, or separate status values.

### Widths and representation

The supplied declarations establish signed integer inputs and a signed integer result. They do not independently establish a particular executable’s integer widths or calling convention.

The concrete values in this contract assume:

- A signed input and result range of **−2,147,483,648 through 2,147,483,647**.
- The same signed range for the relevant arithmetic intermediates.
- A **32-bit unsigned domain** for selector classification.
- Ordinary signed-arithmetic guarantees only where every evaluated signed intermediate is representable.

These assumptions are referred to as the **conditional 32-bit model**.

Unsigned classification does not imply unsigned result arithmetic or establish wrapping behavior for signed arithmetic. Physical argument placement, result placement, signed representation, and executable overflow behavior remain **documented_only**.

### Preconditions

For an unsupported selector, any representable base is admissible under the conditional model.

For a supported selector, the ordinary result guarantee requires the base to fall within the selector-specific domain listed below. These arithmetic restrictions are not validation rules: exceeding a supported selector’s base ceiling does not make that selector unsupported.

No accessible-memory, alignment, initialization, or resource-lifetime precondition follows from the explicit interface. Both arguments are values rather than references that the function dereferences.

## Behavior

### Selection and result

Under the conditional 32-bit model, the supported set is exactly **32,796–32,809**. Every other representable selector returns zero, independently of the base.

For a supported selector within its arithmetic domain, the result is the mathematical sum of the base and the corresponding adjustment:

| Selector | Adjustment | Largest base with the ordinary arithmetic guarantee |
|---:|---:|---:|
| 32,796 | 12 | 2,147,090,095 |
| 32,797 | 24 | 2,147,090,083 |
| 32,798 | 36 | 2,147,090,071 |
| 32,799 | 48 | 2,147,090,059 |
| 32,800 | 60 | 2,147,090,047 |
| 32,801 | 72 | 2,147,090,035 |
| 32,802 | 84 | 2,147,090,023 |
| 32,803 | 96 | 2,147,090,011 |
| 32,804 | 108 | 2,147,089,999 |
| 32,805 | 120 | 2,147,089,987 |
| 32,806 | 132 | 2,147,089,975 |
| 32,807 | 144 | 2,147,089,963 |
| 32,808 | 156 | 2,147,089,951 |
| 32,809 | 168 | 2,147,089,939 |

Each guaranteed base interval is inclusive and begins at **−2,147,483,648**.

Consequences within these domains:

- Every representable base at or below **2,147,089,939** is admissible for every supported selector.
- The minimum signed base is admissible for every supported selector.
- At a fixed base admissible for both calls, increasing the selector by one increases the result by 12.
- For a fixed supported selector, changing the base changes the result by the same amount, provided both bases remain admissible.

There is no additional supported interval among negative selectors. In particular, zero, every negative selector, both signed input extremes, and the immediate neighbors **32,795** and **32,810** are unsupported.

### Why the arithmetic domain is narrower than the final-result range

The ordinary guarantee depends on intermediate representability, not just the final mathematical sum.

Static arithmetic analysis identifies a positive intermediate contribution between **393,552 and 393,708**, depending on the supported selector. An intermediate involving that contribution must fit before a later reduction produces the smaller net adjustment. The resulting base ceilings are therefore stricter than ceilings based only on the final adjustment.

At every selector-specific ceiling in the table, the required result is **2,147,090,107**. Raising that base by one leaves the ordinary arithmetic domain even though the mathematical final result still fits in the signed result range.

For example:

- Base **2,147,090,095**, selector **32,796**: the required result is **2,147,090,107**.
- Base **2,147,090,096**, selector **32,796**: the ordinary arithmetic guarantee does not apply.

No extra lower-bound restriction is needed. For every supported selector, the relevant intermediate and final values remain representable when the base is the minimum signed input.

These limits characterize the supplied function’s guaranteed domain. They do not require another implementation of this contract to reproduce its internal arithmetic.

### Zero is not a distinct status

Zero has two possible meanings:

1. The selector was unsupported.
2. The selector was supported and its adjustment canceled the base.

For example, all of the following supported calls require zero:

| Base | Selector |
|---:|---:|
| −12 | 32,796 |
| −60 | 32,800 |
| −168 | 32,809 |

A caller cannot determine whether selection succeeded from the result alone unless an independent caller invariant excludes supported zero-result cases. No such invariant is established here.

### Calls outside the ordinary arithmetic domain

A supported selector remains supported even when its base exceeds the listed ceiling. Such a call is not specified to return zero.

The function contains no explicit overflow detection, saturation, error reporting, or recovery behavior. If the applicable execution model gives no defined behavior to the overflowing signed arithmetic, the contract guarantees neither a numeric result nor normal, effect-free completion for that call.

An independently established modular-arithmetic model could provide an additional guarantee: the supported result would have the residue of the mathematical adjusted base modulo **2³²**. That is a conditional alternative, not an established execution property.

A signed interpretation of such a residue additionally requires an established signed representation.

## Side effects

Within the ordinary arithmetic domain, the returned integer is the only explicit externally visible output.

The supplied body contains no explicit:

- Reads or writes through either argument.
- Global-state reads or writes.
- Calls to other functions.
- Allocation, release, retention, or ownership transfer.
- Persistent-state updates.
- Diagnostics or separate error-state updates.

The result creates no established ownership obligation. Treating it as an address or identifier would require a separate contract establishing that interpretation and its validity.

### Observable ordering

Selector classification determines whether supported-result arithmetic applies. Unsupported selections return zero without subjecting the base to the arithmetic restrictions of supported selections.

Consequently, the maximum signed base is admissible with an unsupported selector even though it lies outside every supported selector’s ordinary arithmetic domain.

There are no explicit shared-memory accesses, device operations, callbacks, or dependency calls whose relative ordering needs specification. The order in which the caller prepares the two arguments is outside this function’s boundary.

The arithmetic-domain restrictions concern evaluated intermediate values; they do not establish a physical instruction schedule or externally observable sequence of arithmetic operations.

### Layout and ownership limits

No buffer, aggregate, record, object, or allocation layout is established. The adjustment spacing does not prove byte units, element sizes, field offsets, contiguous storage, or allocation boundaries.

The absence of explicit memory access does not characterize generated stack traffic, instrumentation, or calling-convention machinery. For calls outside the ordinary arithmetic domain, the side-effect description is not an independent guarantee of safe execution.

## Dependencies

There are **no callable dependencies** in the supplied body. No dependency function IDs are therefore needed.

No assumptions about unknown callees preserving globals, retaining arguments, or releasing memory are used.

The contract does depend on environmental properties:

| Property | Contract significance |
|---|---|
| Signed widths and ranges | Determine representable inputs, arithmetic-domain boundaries, and results. |
| Unsigned width and conversion rules | Determine the exact selector classification. |
| Signed overflow semantics | Determine whether any additional behavior is guaranteed outside the ordinary domain. |
| Signed representation | Determines how an independently established modular residue is interpreted as signed. |
| Calling convention | Determines physical argument and result passage, not established here. |

These properties are not independently resolved for a particular executable environment by the supplied evidence.

## Acceptance examples

The following are **conditional acceptance criteria derived from static analysis**, not observed executions. Concrete values assume the conditional 32-bit model. Ordinary required results also require representable signed intermediates.

### Supported selections

| Base | Selector | Required result | Coverage |
|---:|---:|---:|---|
| 0 | 32,796 | 12 | First supported selector. |
| 0 | 32,797 | 24 | Adjacent supported selector. |
| 0 | 32,809 | 168 | Last supported selector. |
| 1,000 | 32,800 | 1,060 | Interior selector and nonzero base. |
| −100 | 32,796 | −88 | Negative result. |
| −12 | 32,796 | 0 | Supported zero result. |
| −168 | 32,809 | 0 | Supported zero at the upper endpoint. |
| −2,147,483,648 | 32,796 | −2,147,483,636 | Minimum base, smallest adjustment. |
| −2,147,483,648 | 32,809 | −2,147,483,480 | Minimum base, largest adjustment. |

### Unsupported selections

| Base | Selector | Required result |
|---:|---:|---:|
| 1,000 | 32,795 | 0 |
| 1,000 | 32,810 | 0 |
| 1,000 | 0 | 0 |
| 1,000 | −1 | 0 |
| 1,000 | −2,147,483,648 | 0 |
| 1,000 | 2,147,483,647 | 0 |
| 2,147,483,647 | 32,810 | 0 |

The final case distinguishes unsupported selection from supported-result arithmetic: the supported base ceilings do not restrict an unsupported call.

### Arithmetic-domain boundaries

| Base | Selector | Acceptance condition |
|---:|---:|---|
| 2,147,090,095 | 32,796 | Must return 2,147,090,107. |
| 2,147,090,096 | 32,796 | Outside the ordinary guarantee; no concrete result is required by that guarantee. |
| 2,147,089,939 | 32,809 | Must return 2,147,090,107. |
| 2,147,089,940 | 32,809 | Outside the ordinary guarantee; no concrete result is required by that guarantee. |

Both excluded calls have representable mathematical final values. Final-result representability alone is insufficient to bring them into the guaranteed domain. Neither excluded call is reclassified as unsupported.

### Independently established wrapping

These criteria apply only if wrapping modulo **2³²** is independently guaranteed for all relevant arithmetic:

| Base | Selector | Required unsigned residue |
|---:|---:|---:|
| 2,147,483,635 | 32,796 | 2,147,483,647 |
| 2,147,483,647 | 32,796 | 2,147,483,659 |

With a separately established two’s-complement signed interpretation, the second residue represents **−2,147,483,637**.

Both calls lie outside the ordinary signed-intermediate domain. These examples do not establish wrapping support or predict an observed execution.

## Uncertainties

The following remain **documented_only**:

- **Executable integer model:** The conditional 32-bit widths and ranges are not independently established for a concrete executable.
- **Other integer models:** The concrete selector interval, domain ceilings, and acceptance values are not asserted for different widths or conversion rules.
- **Overflow realization:** No execution evidence establishes wrapping, trapping, or any other outcome outside the ordinary arithmetic domain.
- **Physical interface:** Argument locations, result location, and calling convention are unresolved.
- **Caller invariants:** No restrictions on bases, selectors, or supported zero-result cases are established beyond the conditional arithmetic requirements.
- **Application meaning:** The selectors and adjustments have no established domain-specific interpretation.
- **Units and layout:** Byte units, record sizes, field relationships, allocation boundaries, and backing storage are unknown.
- **Address validity:** Neither input nor the result is established as a pointer or valid address.
- **Generated environmental effects:** Instrumentation and execution machinery outside the explicit body remain unexamined.

The fixed-stride interpretation is inferred intent only. It resolves none of these uncertainties.

## Evidence

**Established static evidence:** The function accepts two signed integer values and returns a signed integer value. Selection uses unsigned arithmetic. Unsupported selections produce zero without the supported-result arithmetic. The supported path uses signed arithmetic. The body contains no explicit calls, dereferences, global accesses, ownership operations, or diagnostics.

**Derived under the conditional 32-bit model:** The supported selector interval, adjustment mapping, selector-specific base ceilings, minimum-base admissibility, and acceptance values follow from integer conversion and arithmetic representability analysis. Supported zero results establish that zero alone cannot distinguish an unsupported selector.

**Inferred intent:** Regularly spaced adjustments suggest fixed-stride numeric selection relative to a base. This is not evidence of storage, units, or ownership semantics.

**Not established:** Executable widths, physical interface, overflow realization, caller invariants, application meaning, storage layout, address validity, and generated environmental effects.

No runtime tests have been performed.
