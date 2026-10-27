<!-- Trusted provenance: F-cca11044abb5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cca11044abb5

## Purpose

F-cca11044abb5 translates a writable four-integer record relative to a second four-integer record. It changes both horizontal coordinates by the same displacement and both vertical coordinates by the same displacement, preserving the destination’s signed horizontal and vertical extents when arithmetic is representable.

The coordinate interpretation is inferred intent. The established behavior is arithmetic on four consecutive signed integer elements. There is no validation that either record represents a rectangle, that coordinates are ordered, or that extents are nonnegative.

The displacement is consistent with approximately aligning the records’ centers, with a specific rounding rule: divide the **difference of extents** by two, rounding toward zero. This is not generally interchangeable with independently rounding each record’s center.

## Interface

**Inputs**

- **Destination:** a pointer to at least four consecutive writable signed integer elements. The function also reads all four elements.
- **Reference:** a pointer to at least four consecutive readable signed integer elements.

Use these semantic names for the elements, in increasing element-index order:

| Element index | Destination | Reference |
|---|---|---|
| 0 | horizontal start, \(a_0\) | horizontal start, \(b_0\) |
| 1 | vertical start, \(a_1\) | vertical start, \(b_1\) |
| 2 | horizontal end, \(a_2\) | horizontal end, \(b_2\) |
| 3 | vertical end, \(a_3\) | vertical end, \(b_3\) |

These names describe their arithmetic roles, not a verified external data type or geometric convention.

**Layout and widths**

The supplied evidence uses signed `int` elements and ordinary consecutive-element indexing. Element \(k\) is located \(k\) integer-element widths after the pointer’s base address. There are no intervening record fields in the accesses shown.

The exact integer width, byte width, alignment requirements, byte order, and pointer width are not established by the supplied material. In particular, a 32-bit integer representation must not be treated as confirmed. If the execution environment establishes 32-bit signed integers, the four accessed elements occupy four consecutive 32-bit slots; arithmetic still requires the overflow qualifications below.

**Output**

There is no return value. The result is communicated through four writes to the destination.

No length argument, status result, diagnostic channel, or ownership-transfer indicator is present.

**Preconditions for defined arithmetic behavior**

- Both pointers must support the indicated reads, and the destination must support the indicated writes.
- Storage must have the required alignment and lifetime for signed integer accesses.
- Values must remain stable against concurrent modification during the call.
- Every evaluated signed subtraction and addition, including intermediate extent calculations and final coordinate updates, must be representable in the implementation’s signed integer type.

Invalid memory and arithmetic outside these conditions have no specified recovery behavior in this contract.

## Behavior

### Displacement calculation

Let all \(a\) and \(b\) values denote the values available during displacement calculation, before this function performs any destination write.

The horizontal signed extents are:

\[
w_a = a_2-a_0,\qquad w_b = b_2-b_0.
\]

The vertical signed extents are:

\[
h_a = a_3-a_1,\qquad h_b = b_3-b_1.
\]

The two displacements are:

\[
d_x=(b_0-a_0)+\operatorname{trunc}_0\!\left(\frac{w_b-w_a}{2}\right)
\]

\[
d_y=(b_1-a_1)+\operatorname{trunc}_0\!\left(\frac{h_b-h_a}{2}\right).
\]

Here \(\operatorname{trunc}_0\) means integer division rounded toward zero. A positive odd extent difference discards a positive half-unit; a negative odd extent difference discards a negative half-unit.

These equations describe the established grouping of arithmetic. They do not authorize rearrangement into a formula with different intermediate overflow behavior or rounding points.

The horizontal displacement is calculated first. The vertical displacement is calculated next. Both calculations finish before the first destination write.

### Destination updates

The function then updates the destination in this order:

1. Horizontal start receives its current value plus \(d_x\).
2. Vertical start receives its current value plus \(d_y\).
3. Horizontal end receives its current value plus \(d_x\).
4. Vertical end receives its current value plus \(d_y\).

Under the stable-memory precondition, the resulting destination is therefore:

\[
(a_0+d_x,\ a_1+d_y,\ a_2+d_x,\ a_3+d_y).
\]

Each update reads the destination element being updated before writing that element. The reference is not explicitly read again after destination writes begin.

There is no conditional early exit, including when both displacements are zero. The four destination assignments remain part of the behavior.

### Rounding and geometric consequences

When arithmetic is representable:

- The destination’s signed horizontal extent is preserved.
- The destination’s signed vertical extent is preserved.
- Coordinate order is preserved within each axis; reversed endpoints are not normalized.
- Equal corresponding extents produce displacement equal to the difference between the corresponding start coordinates.
- Even differences of extents permit exact alignment of the corresponding mathematical centers.
- Odd differences leave a half-unit center discrepancy caused by truncating the extent-difference adjustment toward zero.

The function does not resize, clamp, intersect, or normalize either record.

### Aliasing

Exact aliasing of destination and reference is supported by the visible arithmetic: with stable valid storage, both displacements are zero and the four assignments preserve the values.

Partial overlap is not explicitly rejected. Both displacement calculations occur before any destination write, so this function’s writes do not feed back into those calculations. Subsequent updates use the already calculated displacements.

However, reference-addressed storage that overlaps the destination can change as a consequence of destination writes. “Reference” describes its use in the calculation, not a guarantee that the underlying storage remains unchanged.

The source does not establish volatile or atomic access semantics. Operand evaluation order within an individual arithmetic expression is not fully specified. Consequently, this contract does not promise a total order for every individual input read.

## Side effects

The explicit side effects are four signed-integer writes through the destination pointer, in element order 0, 1, 2, 3.

For nonoverlapping records, there are no explicit writes to reference storage. With overlap, destination writes can also modify storage reachable through the reference pointer.

There are no visible calls, allocations, deallocations, ownership transfers, global-variable accesses, I/O operations, or explicit error reports. The function neither acquires nor releases the supplied storage.

The evidence does not establish exception handling, memory-fault recovery, transactional updates, or rollback. In particular, the ordered writes are not an atomic four-element replacement. Other observers must not assume that all four new values become visible simultaneously.

Signed overflow is not specified as wrapping, saturation, or an error return. A mathematically representable final result is insufficient if an earlier intermediate operation overflows.

## Dependencies

No external function dependencies are visible. F-cca11044abb5 makes no calls in the supplied body.

Its behavior depends on the execution environment’s signed integer representation and range, integer-division semantics, pointer validity, and memory-access rules. No callee preservation or memory-ownership assumptions are needed.

## Acceptance examples

These examples are conditional expectations for valid, stable storage and representable arithmetic. They are not runtime observations.

### Equal extents

Destination: \((2,4,8,10)\)  
Reference: \((20,30,26,36)\)

Both records have horizontal and vertical extents of 6. The displacements are \(18\) and \(26\).

Expected destination: **\((20,30,26,36)\)**.

### Larger reference with even extent differences

Destination: \((0,0,4,2)\)  
Reference: \((10,20,18,26)\)

The horizontal extent difference is 4 and the vertical extent difference is 4. The displacements are \(12\) and \(22\).

Expected destination: **\((12,22,16,24)\)**.

The destination retains extents 4 and 2; it does not expand to match the reference.

### Positive odd extent difference

Destination: \((0,0,2,2)\)  
Reference: \((10,0,15,2)\)

The horizontal extent difference is 3. Its halved integer adjustment is 1, so the horizontal displacement is 11.

Expected destination: **\((11,0,13,2)\)**.

The resulting horizontal center is 12, while the reference center is 12.5.

### Negative odd extent difference

Destination: \((0,0,5,2)\)  
Reference: \((10,0,12,2)\)

The horizontal extent difference is \(-3\). Division toward zero gives \(-1\), not \(-2\). The horizontal displacement is therefore 9.

Expected destination: **\((9,0,14,2)\)**.

This distinguishes the behavior from floor division.

### Reversed endpoints

Destination: \((8,6,2,2)\)  
Reference: \((20,30,14,26)\)

The corresponding signed extents match: \(-6\) horizontally and \(-4\) vertically.

Expected destination: **\((20,30,14,26)\)**.

The function does not reject or normalize the reversed endpoints.

### Partial overlap

Suppose five consecutive integer elements initially contain:

\[
(0,0,4,4,8).
\]

The destination begins at the first element; the reference begins at the second. Thus the initial destination is \((0,0,4,4)\), and the initial reference is \((0,4,4,8)\).

The calculated displacements are 0 horizontally and 4 vertically.

Expected complete storage afterward:

\[
\mathbf{(0,4,4,8,8)}.
\]

The reference-addressed record has changed because its storage overlaps the destination, not because of separate writes through the reference pointer.

### Unrepresentable intermediates

If subtracting a start coordinate from an end coordinate exceeds the signed integer range, no normal result is required by this contract—even if a rearranged mathematical calculation would produce in-range final coordinates.

No wrapping result or recoverable error should be assumed.

## Uncertainties

**Inferred intent**

- The four elements likely describe two-dimensional bounds.
- The operation likely intends to center one set of bounds relative to another.

Neither interpretation establishes a coordinate-system origin, axis direction, inclusive/exclusive endpoint convention, or valid geometric domain.

**documented_only**

- Exact integer and pointer widths.
- Target-specific signed arithmetic and exceptional execution behavior beyond the stated representability conditions.
- Required alignment and actual caller-provided storage sizes.
- Whether real callers use exact or partial aliasing.
- Whether callers impose additional coordinate constraints.
- Machine-level access sizes or instruction ordering beyond what the supplied source establishes.
- Behavior under concurrent mutation, memory-mapped I/O, or invalid storage.

These unknowns do not justify assuming validation, ownership transfer, wrapping arithmetic, or stronger synchronization.

## Evidence

The supplied static body establishes:

- Reads of four consecutive signed integer elements from each input record.
- Horizontal displacement calculation followed by vertical displacement calculation.
- Division of each signed extent difference by two.
- Completion of both displacement calculations before destination writes.
- Four ordered destination updates using one displacement per axis.
- No visible calls, validation branches, explicit global accesses, allocation, deallocation, or status return.

The geometric interpretation and center-alignment description are deductions from that arithmetic, not independently verified application requirements.

No runtime tests have been performed. The acceptance examples are calculated consequences of the supplied static evidence under their stated preconditions.
