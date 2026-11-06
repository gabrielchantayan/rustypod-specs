<!-- Trusted provenance: F-4866fbcc8b6b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4866fbcc8b6b

## Purpose

F-4866fbcc8b6b conditionally produces four signed coordinate extrema from a sequence of two-coordinate records.

The destination is first cleared to four zero values. Further processing requires equality between a 32-bit field in each of two supplied objects. If that comparison succeeds and a subsequent address guard permits processing, the function either retains zero extrema for an empty sequence or computes minimum and maximum values across the records it visits.

The output order is:

1. Minimum first coordinate.
2. Minimum second coordinate.
3. Maximum first coordinate.
4. Maximum second coordinate.

Interpreting these values as an axis-aligned bounding rectangle is inferred intent. The established behavior is signed extrema calculation; no coordinate units, geometric validity rules, or object-type meanings are supplied.

## Interface

The function has three inputs and no returned value:

| Input | Role |
|---|---|
| Reference-object address | Supplies a 32-bit comparison field at byte offset 0x18. |
| Sequence-object address | Supplies a 32-bit comparison field at byte offset 0x48, a signed 16-bit record count at byte offset 0x6E, and a record-array address at byte offset 0x70. |
| Destination address | Receives four consecutive signed 32-bit values, occupying 16 bytes. |

The supplied representation uses integer-valued object addresses and 32-bit integer record elements. This contract describes the apparent 32-bit target layout. A portable ABI, pointer representation, byte order, and alignment policy are not independently established.

### Data layout

Each record consists of two consecutive signed 32-bit coordinates:

| Record-relative byte offset | Meaning |
|---|---|
| 0 | First coordinate |
| 4 | Second coordinate |

The record stride is eight bytes. No per-record metadata or separator is read.

The destination layout is:

| Destination-relative byte offset | Meaning |
|---|---|
| 0 | Minimum first coordinate |
| 4 | Minimum second coordinate |
| 8 | Maximum first coordinate |
| 12 | Maximum second coordinate |

The count is signed, not unsigned. Its representable range is −32,768 through 32,767. Coordinate comparisons are also signed.

### Preconditions for ordinary successful use

The destination must permit four consecutive 32-bit writes immediately upon entry. A later null-address check does not relax this requirement.

Both comparison fields must be readable after those initial writes. If the fields match and processing is not suppressed by the address guard, the count must be readable. A nonzero count additionally requires a readable array-address field and at least one readable record.

For a stable positive count, the array must contain that many accessible records. The array and its calculated endpoint must support meaningful pointer arithmetic and ordering without address wrapping.

Disjoint, stable input and output storage gives the straightforward extrema behavior described below. Aliasing and concurrent changes require additional care because the destination is cleared before input fields are examined, and the count is read repeatedly.

## Behavior

### Initial destination writes

The function writes zero to all four destination slots, in increasing slot order.

These writes occur before:

- Either comparison field is evaluated.
- The sequence-object address guard.
- The destination null-address check.
- Any count or record-array access.

Consequently, even a comparison mismatch does not leave the destination unchanged. Its normal result is four zeros.

If the destination overlaps either input object, these initial writes may change fields subsequently read by the function. The contract must therefore be evaluated against the memory visible after the clearing writes, not necessarily the memory present at entry.

### Comparison gate

The function compares the sequence object's 32-bit field at offset 0x48 with the reference object's 32-bit field at offset 0x18.

If they differ, the function returns immediately. No count, array-address, or coordinate read is performed on that path.

Both comparison reads follow the destination clearing. The supplied expression does not establish an evaluation order between the two comparison operands; a contract should not require one particular field to be read first.

The comparison establishes only equality of these values. It does not establish that either address denotes a valid object of some particular external type.

### Address guard

If the comparison fields match, processing continues only when:

- The sequence-object address is not the 32-bit integer address value −108, equivalently 0xFFFFFF94.
- The destination address is nonzero.

Failure of this guard causes a return without final result writes beyond the initial clearing.

The special sequence-object value would make adding 0x6C reach zero under ordinary 32-bit modular address interpretation. That relationship may explain the guard, but its intended meaning is unknown.

This guard is not an early safety check. The sequence comparison field has already been accessed, and the destination has already been written. Neither a null destination nor the special sequence-object value can be assumed safe merely because it is tested here.

### Zero count

After passing the comparison and address gates, the function reads the signed 16-bit count.

If the count is zero:

- It does not read the array-address field.
- It does not read any coordinate.
- It selects zero for all four extrema.
- It writes those four zero results to the destination, in output-slot order.

Thus this path performs both the initial clearing writes and a second sequence of four zero writes.

A null or otherwise unusable array address does not matter on this path, provided the array-address field does not need to be accessed for some external reason not shown here.

### Nonzero count

Any nonzero count, including a negative count, enters the record-processing path.

The function reads the array address once, then reads the first record's first and second coordinates, in that order. These seed both minima and both maxima.

The first record is therefore accessed before the function establishes whether a second record lies within the iteration bound. A negative count does not suppress this first-record read.

For each subsequent candidate record, the function checks whether its address is below a calculated endpoint. The endpoint is the saved array address advanced by twice the current signed count in 32-bit elements, equivalent to eight bytes per counted record.

The count field is read again for each continuation check. It is not a single entry-time snapshot used throughout the scan.

When a candidate record passes the bound check:

1. Its first coordinate is read.
2. The first-coordinate minimum and maximum are updated by signed comparison.
3. Its second coordinate is read.
4. The second-coordinate minimum and maximum are updated by signed comparison.
5. The scan advances to the next record.

Equal coordinates do not change an extremum's value. There is no subtraction, magnitude calculation, coordinate normalization, or rejection of negative coordinates.

For stable positive counts and ordinary nonwrapping array layout, exactly the specified number of records contributes to the result.

### Final destination writes

After selecting or calculating all four extrema, the function writes the results in this order:

1. Minimum first coordinate.
2. Minimum second coordinate.
3. Maximum first coordinate.
4. Maximum second coordinate.

No coordinate reads follow these final writes in the supplied function.

These are separate writes, not an established atomic publication of a 16-byte result. An external observer could encounter intermediate destination contents if the surrounding execution environment permits such observation.

## Side effects

The established writes are confined to the four destination slots:

- Four initial zero writes occur on every normally executing path.
- Four final result writes occur when the comparison and address gates pass.

There is no explicit write to the input objects or record array. However, destination aliasing can make these writes modify their storage indirectly.

In particular:

- Overlap with comparison fields can change whether the comparison gate succeeds.
- Overlap with the count can turn a nonempty input into an empty one before the first count read.
- Overlap with the array-address field can change the address subsequently loaded.
- Overlap with records can alter coordinates before scanning begins.

The function performs no explicit allocation, deallocation, ownership transfer, callback, I/O, or synchronization. Nothing in the supplied material authorizes the function to release caller-owned storage, and no release operation is shown.

Invalid accesses have no defined recovery path here. There is no error return, exception-handling branch, or error-status write. The exact effects of an invalid or misaligned address depend on the target environment and are documented_only.

## Dependencies

No calls to other functions are present in the supplied body. No additional opaque function dependency is established.

Execution nevertheless depends on external conditions:

- Validity and accessibility of the supplied memory.
- Target integer, pointer, alignment, and byte-order rules.
- Stability of the repeatedly read count.
- Any synchronization required by the surrounding program.

The function itself does not establish those conditions.

## Acceptance examples

The following are conditional acceptance scenarios derived from static inspection. They are not runtime test results. Unless stated otherwise, storage is valid, disjoint, stable, and nonwrapping; comparison fields match; and the address guard permits processing.

### Comparison mismatch

Given unequal comparison fields and a destination initially containing arbitrary values:

- The destination becomes `[0, 0, 0, 0]`.
- The function returns without reading the count or record array.

This remains the expected behavior even if the unused record-array address is invalid.

### Empty sequence

Given a count of zero:

- The destination finishes as `[0, 0, 0, 0]`.
- The array-address field and coordinate storage are not read.
- Four initial zero writes and four final zero writes occur.

### One record

Given count 1 and a record `(7, −3)`:

- The result is `[7, −3, 7, −3]`.
- The first record is read.
- The continuation check rejects the second record, so no second-record coordinate is read.

### Multiple records

Given count 3 and records `(4, −2)`, `(−5, 9)`, and `(4, 3)`:

- The result is `[−5, −2, 4, 9]`.
- Records are visited in array order.
- Within each record, the first coordinate is read before the second.

### Signed-coordinate boundaries

Given two records `(−2,147,483,648, 2,147,483,647)` and `(2,147,483,647, −2,147,483,648)`:

- The result is `[−2,147,483,648, −2,147,483,648, 2,147,483,647, 2,147,483,647]`.
- No coordinate arithmetic is needed, so those coordinate values do not themselves cause arithmetic overflow.

### Negative count

Given count −1, a readable first record `(12, 20)`, and ordinary nonwrapping pointer ordering in which the calculated endpoint lies below the array base:

- The first record is still read.
- No later record qualifies.
- The result is `[12, 20, 12, 20]`.

This is a consequence of the observed control flow, not evidence that negative counts are valid application inputs.

### Maximum positive count

Given a stable count of 32,767 and a valid contiguous array:

- The intended scan covers 32,767 records.
- The byte span is 262,136 bytes.
- The element-count calculation is 65,534 32-bit elements, which fits a signed 32-bit integer.

This does not establish safety if calculating the endpoint overflows the address space or exceeds the valid array object.

## Uncertainties

**Established static behavior:** destination clearing precedes validation; comparison mismatch returns zeros; count zero avoids the array-address read; every nonzero count seeds extrema from the first record; later iteration checks reload the count; signed coordinate extrema are written in the specified order.

**Inferred intent:** the objects likely carry matching identity or type information, and the result likely describes a two-dimensional bounding rectangle. Neither interpretation is necessary to reproduce the established behavior.

**Documented_only unknowns:**

- Semantic meanings of the comparison fields and the special address guard.
- Whether callers are required to reject negative counts.
- Whether aliasing is permitted by the broader interface.
- Whether concurrent mutation is allowed or synchronized.
- Exact pointer-order behavior at wrapped or invalid endpoints.
- Target fault behavior for null, inaccessible, or misaligned memory.
- Whether the concrete implementation guarantees additional machine-level ordering beyond the supplied representation.

A changing count can expand or shorten the scan because continuation checks reload it. That observation is not a guarantee of thread-safe dynamic resizing; no synchronization or memory-lifetime protection is shown.

## Evidence

This contract is based solely on the supplied static function representation.

The visible operations establish the destination layout, field offsets, signed count interpretation, coordinate comparison behavior, traversal structure, and absence of explicit function calls. No surrounding callers, object declarations, allocator contracts, or platform ABI documentation were supplied.

No runtime tests have been performed. The acceptance examples are conditional predictions from static evidence, not observed execution results.
