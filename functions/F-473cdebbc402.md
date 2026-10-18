<!-- Trusted provenance: F-473cdebbc402; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-473cdebbc402

## Purpose

F-473cdebbc402 fills a caller-owned, two-element integer result and returns a pointer identifying that result storage.

With separate input and output storage, the result consists of midpoint-like values for two ordered pairs drawn from four input elements. The pair using input positions 1 and 3 supplies the first output element; the pair using positions 0 and 2 supplies the second. Positions are numbered from zero.

The calculation uses a signed difference, division by two, and an adjustment from the pair’s starting endpoint. Consequently, integer rounding depends on endpoint order, and an unrepresentable difference prevents a defined source-level result even when the mathematical midpoint is representable.

**Inferred intent:** The operation may calculate a center from two coordinate pairs while exchanging coordinate order. The supplied representation does not establish coordinate meanings, geometric constraints, or a required ordering between endpoints.

## Interface

### Arguments and result

| Interface item | Required storage or meaning |
|---|---|
| Input pointer | Identifies readable integer elements at positions 0 through 3. |
| Output pointer | Identifies writable integer elements at positions 0 and 1. |
| Returned pointer | Equals the supplied output pointer; identifies the first output element. |

The output is not separately allocated. The caller supplies and retains responsibility for its storage.

The input and output regions need not be distinct under the visible interface. Their overlap affects results because the first output write precedes the second calculation. A separate-input interpretation must therefore not be imposed on overlapping calls.

### Widths and layout

The supplied representation identifies the elements as signed integers but does not independently establish their bit width, object size, representation, or alignment.

Let \(S\) be the size in bytes of one such integer object. The accessed locations are:

| Region | Element positions | Byte offsets from that region’s pointer |
|---|---|---|
| Input | 0, 1, 2, 3 | \(0,\ S,\ 2S,\ 3S\) |
| Output | 0, 1 | \(0,\ S\) |

The function does not expose a length argument, stride argument, or result-size field.

If the target uses four-byte integer objects, the input offsets are 0, 4, 8, and 12 bytes, and the output offsets are 0 and 4 bytes. This is a conditional layout example, not established target metadata. Pointer width and byte order are also unresolved.

### Preconditions

A defined source-level invocation requires:

- Valid integer objects and live storage for every location accessed.
- Appropriate alignment and permission for the respective reads and writes.
- Readable integer values at the time each input location is used.
- Representable signed arithmetic intermediates.
- No concurrent access that makes execution invalid under the applicable memory model.

Initial output contents are irrelevant unless output storage also serves as input. Overlap does not remove any validity, lifetime, alignment, or range requirement.

No validation of pointers, storage extent, or arithmetic range appears in the supplied body.

## Behavior

### Pair selection

The two result positions have distinct endpoint assignments:

| Result position | Starting endpoint | Other endpoint |
|---|---|---|
| 0 | Input position 1 | Input position 3 |
| 1 | Input position 0 | Input position 2 |

“Starting endpoint” denotes arithmetic precedence, not numerical order. It may be smaller than, equal to, or greater than the other endpoint.

For nonoverlapping, stable storage, each result depends only on its designated pair. No sorting, normalization, clamping, or endpoint validation is established.

### Integer calculation and rounding

For each pair, the calculation first forms the signed difference from the starting endpoint to the other endpoint. It divides that difference by two using integer division, then adds the resulting adjustment to the starting endpoint.

This intermediate-difference requirement is part of the behavior. Replacing the operation with an endpoint sum, wider arithmetic, saturation, or an otherwise overflow-resistant midpoint calculation is not established as equivalent.

Under truncation-toward-zero division:

- An even difference produces the exact mathematical midpoint.
- An odd difference produces the nearer integer on the starting endpoint’s side of the mathematical midpoint.
- Equal endpoints produce that endpoint.
- Exchanging unequal endpoints can change the result by one when their difference is odd.

For example, the ordered pair starting at 2 and ending at 7 produces 4. Reversing the endpoints produces 5. For a pair starting at −7 and ending at −2, the result is −5, not −4.

These rounding statements and the numerical acceptance examples are conditional on truncation toward zero. That is the ordinary modern signed-integer interpretation of the supplied division, but the material does not independently verify the executed target’s division behavior.

The signed difference must be representable before division. A representable mathematical midpoint alone is insufficient. Under the stated truncation convention, a representable difference yields a result between the endpoints, so the final adjustment does not create an additional range problem.

### Visibility of the first result

The first output element is written before the input values for the second result are evaluated.

This ordering is externally significant when storage overlaps. The second result is based on the input values available after the first write, not on a guaranteed snapshot taken at function entry.

Three overlap arrangements illustrate the distinction:

- **Identical input and output bases:** The first result replaces input position 0. That replacement becomes the starting endpoint for the second result.
- **Output begins at input position 2:** The first result replaces input position 2. That replacement becomes the other endpoint for the second result.
- **Output begins at input position 1:** The first write changes an input position not used by the second calculation. The second write changes input position 2 only after its value has contributed to the result.

Other valid overlap arrangements follow the same storage-dependence rule. No copying or nonaliasing restriction is visible.

### Read-order limits

The supplied statement structure establishes a completed first output write before the second calculation and a completed second output write before return.

It does not establish a left-to-right order among operand reads within an individual calculation. Nor does repeated use of an input element establish a fixed number of physical loads.

For ordinary stable integer storage, these limits do not change the values described here. The function has no established device-memory, volatile-access, or asynchronous-observation protocol.

### Completion and errors

On defined completion, both output elements have been assigned, and the returned pointer equals the supplied output pointer. The return does not identify the end of the result region and does not encode arithmetic success or failure.

No explicit error path or diagnostic mechanism appears. Invalid access and unrepresentable signed arithmetic therefore have no specified recovery behavior in this contract. In particular, the source does not promise wrapping, saturation, a particular signal, an exception, or continued execution.

The operation is not transactional. Its structure does not provide rollback, but invalid execution also does not justify a portable promise about which partial effects remain observable.

## Side effects

The directly expressed state changes are two integer writes to caller-provided output storage, in result-position order.

Input storage is not directly designated as a write destination. Nevertheless, any portion shared with the output can change. Input preservation is guaranteed by neither the argument roles nor the function’s visible structure when the regions overlap.

The supplied body contains no explicit:

- Global-state access.
- Allocation or deallocation.
- External call.
- I/O.
- Pointer-retention mechanism.

No ownership transfer is shown. The function neither creates an independently owned result nor extends the lifetime of the supplied storage. The returned pointer is useful only for as long as the caller’s output storage remains valid.

These statements describe visible direct effects. They do not add guarantees for invalid memory accesses, arithmetic outside the defined range, or environmental behavior not represented in the supplied material.

## Dependencies

No callable dependency is present in the supplied body.

The operation relies on environmental rules for signed integer arithmetic, division, object layout, pointer validity, alignment, lifetime, and concurrent access. Those rules are not identified helper functions and cannot be assigned additional behavior from this evidence.

There is no unknown callee whose global-state preservation, memory ownership, or release behavior must be assumed.

## Acceptance examples

These are conditional acceptance criteria, not execution results. Unless a case states otherwise, assume:

- Valid, stable, nonoverlapping input and output storage.
- Truncation-toward-zero integer division.
- Representable intermediate arithmetic.

### Pair selection and rounding

| Input positions 0–3 | Required output positions 0–1 | Property |
|---|---|---|
| \([0,10,8,18]\) | \([14,4]\) | Input positions 1 and 3 supply the first result. |
| \([2,3,7,8]\) | \([5,4]\) | Positive odd differences round toward each starting endpoint. |
| \([7,8,2,3]\) | \([6,5]\) | Reversing those pairs changes the odd-difference results. |
| \([-5,-9,-5,-9]\) | \([-9,-5]\) | Equal endpoints remain unchanged. |
| \([-8,-7,-3,-2]\) | \([-5,-6]\) | Negative endpoint values do not imply floor rounding. |

For every defined case, the returned pointer must equal the supplied output pointer, independently of the numerical result.

### Identical input and output bases

Assume a four-element backing region initially contains \([0,10,8,18]\), and both arguments identify its first element.

The first output value is 14 and replaces backing position 0. The second pair therefore starts at 14 and ends at 8, producing 11. The final backing contents are:

\[
[14,11,8,18]
\]

Producing \([14,4,8,18]\) by snapshotting the initial input would not preserve the established overlap behavior.

### Output begins at input position 2

Assume the same initial backing contents, but the output pointer identifies backing position 2.

The first result replaces position 2 with 14. The second pair then starts at 0 and ends at that replacement value, producing 7. The final backing contents are:

\[
[0,10,14,7]
\]

The returned pointer identifies backing position 2, not the beginning of the backing region.

### Output begins at input position 1

With initial backing contents \([0,10,8,18]\) and the output pointer identifying position 1, the first result replaces position 1 with 14. The second calculation still uses the unchanged values 0 and 8, producing 4.

The final backing contents are:

\[
[0,14,4,18]
\]

This demonstrates that overlap alone does not necessarily alter the numerical output; the particular locations overwritten matter.

### Conditional range boundary

If the target’s signed integer range is −2,147,483,648 through 2,147,483,647, a pair starting at the minimum and ending at the maximum requires a positive difference of 4,294,967,295.

That difference is outside the stated range. No ordinary defined-arithmetic result is required, even though the mathematical midpoint lies within the range. An implementation must not be characterized as wrapping, saturating, or using a wider temporary without further evidence.

## Uncertainties

The following remain **documented_only**:

- Actual integer width, object size, signed representation, alignment, pointer width, and byte order.
- Executed target division semantics beyond the conditional interpretation used here.
- Exact instructions, load counts, and operand-read order within a calculation.
- Concrete consequences of invalid addresses, insufficient storage, invalid lifetimes, races, or unrepresentable arithmetic.
- Behavior required for special-purpose or externally changing memory.
- Any caller-side restrictions on overlap not present in the supplied body.
- Whether coordinate geometry is the intended application.

The absence of explicit checks does not establish a particular failure mode. Likewise, the absence of visible global accesses or calls does not resolve behavior introduced by invalid execution or an unspecified execution environment.

## Evidence

**Established static evidence:** The supplied representation selects two ordered input pairs, writes the result for positions 1 and 3 first, writes the result for positions 0 and 2 second, and returns the output pointer. Each result uses a signed endpoint difference, division by two, and an adjustment from the starting endpoint. No call, allocation, release, validation branch, or explicit global access is present.

**Derived consequences:** The first write can influence the second calculation through overlap. Intermediate subtraction range matters independently of mathematical midpoint range. Under truncation-toward-zero division, odd differences round toward the starting endpoint, making endpoint order significant.

**Inferred intent:** A midpoint or center-coordinate operation. That interpretation supplies no additional validation, geometry, ownership, or layout guarantees.

**Verification status:** Static analysis of the supplied representation only. No implementation access, runtime execution, tests, or independent target-width verification has been performed.
