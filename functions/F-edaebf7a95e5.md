<!-- Trusted provenance: F-edaebf7a95e5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-edaebf7a95e5

## Purpose

F-edaebf7a95e5 produces a signed measure of the separation between two stored integers, using sixteen-unit groups.

The first stored integer is the **reference**; the next is the **comparison**. Direction matters: the mathematical separation is positive when the comparison exceeds the reference and negative when it is smaller.

For a nonnegative separation that fits the signed arithmetic type, the result counts complete groups of sixteen. An incomplete group contributes nothing. The grouping applies to the separation, not to the two input values independently.

For a representable negative separation, the exact result depends on the environment’s rule for signed right shifts. The supplied evidence does not establish that rule.

“Reference,” “comparison,” and “separation” describe arithmetic roles. Interpreting the values as addresses, byte counts, positions, or object boundaries would be **inferred intent**, not an established application-level contract.

This specification is based only on the supplied static evidence. No runtime tests have been performed.

## Interface

### Argument and layout

The function accepts one pointer to storage containing two consecutive elements of the native signed-integer type used by the function.

| Element position | Arithmetic role |
|---|---|
| First | Reference |
| Immediately following | Comparison |

The comparison occupies the next element position, not the next byte position. Its byte displacement depends on the element type’s storage size.

The evidence establishes neither a particular integer width nor a particular byte layout. In particular, it does not independently justify a four-byte element size or an eight-byte accessible region.

Valid use requires:

- Both element positions to be readable through the supplied pointer.
- Storage and alignment appropriate for the element type.
- Valid object lifetimes and valid stored integer representations.
- Any synchronization required by the environment’s concurrency rules.

A non-null pointer is insufficient. Storage valid for only one element is also insufficient. Physical adjacency alone does not establish that two separately situated objects may validly be accessed as consecutive elements.

There is no length argument, optional second operand, adjustable grouping size, or separate error-output location.

### Return value and arithmetic domain

The return value has the same declared signed-integer type as the stored elements. It is a numeric result, not a success code.

The separation is calculated in that signed type. The supplied body establishes no conversion to a wider arithmetic domain before the separation is formed. Both inputs may therefore be representable while their mathematical separation is not.

The fixed grouping operation is a signed right shift by four positions. Its nonnegative behavior is established; its negative behavior requires an environmental rule.

The exact signed range, storage size, alignment, representation, byte order, and physical calling convention remain **documented_only**.

### Ownership and lifetime

The caller retains ownership of the input storage. The function does not allocate, free, retain, or transfer it.

The function requires read access, not write access. Both elements must remain valid when observed. The supplied body establishes no deferred access or use of the pointer after return.

## Behavior

### Separation precedes grouping

The comparison is measured relative to the reference before grouping takes place. Grouping each input independently and then comparing the grouped values is not an equivalent implementation of this contract.

For example, a reference of 7 and a comparison of 22 are fifteen units apart. Their separation contains no complete sixteen-unit group, so the result is zero, even though the two values lie in different origin-based sixteen-unit intervals.

The signs of the input values do not independently determine the sign of the separation. Two negative inputs can produce a positive separation; two nonnegative inputs can produce a negative one.

### Nonnegative separation

For valid, stable input whose nonnegative separation is representable, the result is the number of complete sixteen-unit groups:

| Separation | Required result |
|---:|---:|
| 0 through 15 | 0 |
| 16 through 31 | 1 |
| 32 through 47 | 2 |
| 48 through 63 | 3 |

This behavior continues through the representable nonnegative range.

Equal inputs return zero. Unequal inputs with a positive separation smaller than sixteen also return zero. Consequently, zero does not establish equality.

There is no fractional return, remainder output, or separate indication that an incomplete group was discarded.

### Negative separation

A representable negative separation reaches a signed right-shift operation. The evidence does not determine the environment’s treatment of that operation for negative operands.

If the environment defines the operation as division by sixteen rounded toward negative infinity, these results follow:

| Separation | Conditional result |
|---:|---:|
| −1 through −16 | −1 |
| −17 through −32 | −2 |
| −33 through −48 | −3 |

Under that condition, exact negative multiples of sixteen produce their exact quotients. Other negative separations produce the next lower integer relative to the mathematical quotient.

This condition must not be silently replaced with division rounded toward zero. For a separation of −1, rounding toward zero would give zero, whereas the stated condition gives −1.

Without an established negative-shift rule, exact negative outputs remain **documented_only**. The supplied evidence does not even support an unconditional requirement that every negative separation produce a negative return.

Reversing the inputs reverses their mathematical separation when both directions fit the arithmetic type. It need not negate the returned value. Under the negative-infinity condition, a separation of 15 returns zero, while the reversed separation returns −1.

### Arithmetic limits

Representability is required when the separation is formed, before grouping. A representable grouped answer does not repair an earlier out-of-range separation.

For example, measuring from the minimum signed value to zero can require a positive separation larger than the type’s maximum value. Its mathematical quotient after grouping may fit, but no widened calculation is established.

The body provides no explicit overflow check, saturation, recovery, or reserved result. Guaranteed modular wraparound is not established either.

Under ordinary source-level signed-arithmetic rules, an unrepresentable separation is outside the defined arithmetic domain. No specific executable result, diagnostic, or fault is promised for that case.

### Observation order and consistency

Both stored values contribute to the result. Their relative observation order is not established. “Reference” identifies an arithmetic role, not a required first read.

The function establishes no atomic access, volatile access, lock, retry, or coherent-pair snapshot. Concurrent modification must satisfy the environment’s access rules; this function supplies no additional coordination.

The contract does not prescribe physical load count, machine access width, or instruction order. For stable valid storage, these implementation details do not change the required numeric result.

### Invalid inputs and errors

There is no explicit validation or rejection path for null, expired, misaligned, insufficient, unreadable, or otherwise incompatible storage. There is also no explicit check of the stored representations.

Such inputs have no specified numeric result or failure response. A fault is not guaranteed, and an ordinary-looking return does not establish that the access was valid.

No return value is reserved for errors. Zero must not be interpreted as successful validation, and negative returns must not be interpreted as error codes.

## Side effects

Within the valid memory-access and arithmetic domain, the returned integer is the only explicit output.

The supplied body establishes:

- No modification of either input element.
- No writes to other storage.
- No global-state access.
- No allocation or deallocation.
- No external input or output.
- No synchronization.
- No pointer retention.

The function itself leaves the input storage unchanged. This does not constrain modifications by other participants.

These statements describe the established operations in the supplied body. They do not constrain consequences outside the defined access and arithmetic domain, nor do they prescribe physical memory transactions.

## Dependencies

There are no function-call dependencies in the supplied body. No additional opaque function IDs are required.

Accordingly, the contract does not rely on an unknown callee preserving globals, retaining arguments, or managing memory.

The remaining dependencies are environmental:

| Environmental property | Contract consequence |
|---|---|
| Signed-integer range | Determines valid inputs and representable separations |
| Element size and alignment | Determines accessible storage and layout requirements |
| Integer representation and byte order | Determines how stored bytes denote values |
| Object and pointer-access rules | Determines whether both observations are valid |
| Signed shifting of negative values | Determines exact negative results |
| Signed-arithmetic execution rules | Governs out-of-range separation |
| Concurrency rules | Governs simultaneous modification |
| Calling convention | Determines physical argument and return placement |

A common platform convention is not evidence that the supplied function uses that convention.

## Acceptance examples

These examples are static deductions, not observed executions. Unless an example states otherwise, assume valid consecutive elements, stable readable values, and a representable separation.

### Established nonnegative behavior

| Reference | Comparison | Separation | Required return |
|---:|---:|---:|---:|
| 0 | 0 | 0 | 0 |
| 7 | 22 | 15 | 0 |
| 7 | 23 | 16 | 1 |
| 7 | 38 | 31 | 1 |
| 7 | 39 | 32 | 2 |
| −40 | −8 | 32 | 2 |
| −32 | 0 | 32 | 2 |

The boundary cases distinguish complete-group counting from rounding to the nearest group. The 7-to-22 case also distinguishes grouping the separation from grouping each input separately.

The negative-valued input cases require no special negative-shift assumption because their separations are nonnegative.

All these cases require the function to leave the input values unchanged.

### Conditional negative behavior

The following returns apply only if negative signed shifts implement division rounded toward negative infinity:

| Reference | Comparison | Separation | Conditional return |
|---:|---:|---:|---:|
| 1 | 0 | −1 | −1 |
| 15 | 0 | −15 | −1 |
| 16 | 0 | −16 | −1 |
| 17 | 0 | −17 | −2 |
| 0 | −32 | −32 | −2 |

The −1 and −17 cases distinguish this rule from rounding toward zero. The −16 and −32 cases cover exact multiples.

Without evidence for the stated environmental condition, acceptance must not require these exact returns.

### Conditional range boundaries

For this subsection only, assume the signed range is −2,147,483,648 through 2,147,483,647.

| Reference | Comparison | Required conclusion |
|---:|---:|---|
| 0 | 2,147,483,647 | Representable separation; return 134,217,727 |
| −2,147,483,648 | 0 | Separation exceeds the assumed maximum; no defined source-level result is promised |
| 0 | −2,147,483,648 | Representable separation; return −134,217,728 under the negative-infinity shift condition |
| 2,147,483,647 | −2,147,483,648 | Separation is below the assumed minimum; no defined source-level result is promised |

This range is illustrative, not established target information. The out-of-range cases remain outside the defined arithmetic domain even though grouping their mathematical separations would produce representable quantities.

### Storage and ordering boundaries

A pointer valid for only the first element fails the access precondition. Acceptance must not require a particular result, diagnostic, or fault.

Observing the comparison before the reference is permitted. Acceptance must not require reference-first observation or exactly two physical loads.

A zero return cannot certify equal inputs or valid storage.

## Uncertainties

The following remain **documented_only**:

- Exact integer width, range, size, alignment, representation, and byte order.
- Exact results for negative separations without an established signed-shift rule.
- Executable-specific behavior after unrepresentable signed subtraction.
- Physical observation order, load count, access widths, and instruction selection.
- Physical argument and return placement.
- Application meaning and units of the stored values.
- External caller restrictions, including ordering, alignment to sixteen-unit boundaries, or narrower numeric bounds.

The inferred intent is a directed separation expressed in sixteen-unit groups. No evidence establishes whether those units represent bytes, positions, objects, or another quantity.

The absence of validation does not prove that callers always supply valid storage, ordered values, or bounded separations. Those properties remain preconditions or unknown caller constraints, not established external invariants.

## Evidence

**Established static evidence:** The supplied body accepts one pointer to a native signed-integer type and returns that same signed type. It uses the first two consecutive elements, forms the comparison’s separation from the reference, and applies fixed four-position signed right-shift scaling to the separation. No widening step, validation, writes, global accesses, calls, allocation, release, or retention appears.

**Established deductions:** Representable nonnegative separations yield complete sixteen-unit groups. Grouping occurs after the separation is formed. Equal inputs return zero, but zero does not imply equality. Relative observation order is not specified.

**Conditional deductions:** Exact negative examples require the expressly stated signed-shift condition. Numeric range examples require their expressly assumed range. Defined access and representable signed subtraction delimit the ordinary defined domain.

**Inferred intent:** Directed separation measured in sixteen-unit quantities, without an established application-specific interpretation.

**Unresolved evidence:** Target layout, negative-shift policy, executable overflow behavior, calling convention, and physical memory transactions are not supplied.

No runtime tests have been performed.
