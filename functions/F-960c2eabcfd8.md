<!-- Trusted provenance: F-960c2eabcfd8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-960c2eabcfd8

## Purpose

F-960c2eabcfd8 derives coordinate extrema from a sequence of two-component integer records and writes four results to caller-supplied storage.

For a valid, stable, positive-length sequence, the output order is:

1. Minimum first component.
2. Minimum second component.
3. Maximum first component.
4. Maximum second component.

A zero count produces four zero results. A null descriptor address or null destination causes an immediate return without writing results.

This is a source-facing behavioral contract based on static evidence only. No runtime tests have been performed. The interpretation of the result as a two-dimensional bounding rectangle is inferred intent; the established operation is independent minimum and maximum reduction of two integer components.

## Interface

### Inputs

The function accepts two arguments:

- **Descriptor address:** An integer-valued address identifying a descriptor. Zero is treated as absent. A nonzero value is used to access descriptor fields.
- **Destination address:** An address identifying writable storage for four consecutive integer results. A null address is treated as absent.

The descriptor address is presented as an integer rather than a typed descriptor pointer. Address validity, alignment, representability, and readable extent are caller preconditions; the function does not validate them.

### Descriptor and record layout

Under the recovered 16-bit-short, 32-bit-integer data model, the relevant descriptor fields are:

| Byte offset | Field | Interpretation |
|---|---|---|
| 2 | Count | Signed 16-bit integer |
| 4 | Record-array address | Pointer to consecutive integer component pairs |

The bytes before the count are not inspected. No other descriptor fields are used.

Each record consists of two consecutive signed 32-bit components:

| Byte offset within record | Meaning |
|---|---|
| 0 | First component |
| 4 | Second component |

Records have an eight-byte stride in that model. There is no record-specific stride field, indirection per record, or component conversion.

The source representation alone does not independently establish pointer width, byte order, or every target ABI rule. In particular, the offset of the record-array pointer is established, but its stored width should not be inferred solely from that offset.

### Output layout

The destination receives four consecutive signed 32-bit integers in the recovered data model:

| Byte offset | Result |
|---|---|
| 0 | Minimum first component |
| 4 | Minimum second component |
| 8 | Maximum first component |
| 12 | Maximum second component |

The function has no return value. It does not return a success indicator, record count, destination pointer, or error code.

### Preconditions for ordinary reduction behavior

For the positive-count case:

- Both arguments must be nonzero.
- The descriptor count field and record-array pointer field must be readable.
- The record-array address must identify at least the stated number of complete records.
- The destination must permit four consecutive integer stores.
- Address calculations and pointer comparisons must be valid under the execution model.
- The count and source records must remain stable during the reduction if the caller requires the ordinary fixed-sequence result.

For the zero-count case, the record-array pointer field and record storage are not needed by the function. The descriptor count must still be readable, and the destination must still support all four stores.

## Behavior

### Null-argument gate

The function first checks whether both arguments are nonzero.

If either argument is zero, it returns without reading descriptor fields, reading records, or writing the destination. A valid destination is therefore left unchanged when the descriptor address is zero.

This gate does not check whether a nonzero address is valid. It also does not initialize the destination on an early return.

### Zero-count case

After the null gate, the function reads the signed count at descriptor byte offset 2.

If that value is zero:

- All four pending results become zero.
- The record-array pointer field is not read.
- No record components are read.
- The function writes the four zero results in output order.

Zero is a special case, not a reduction using sentinel extrema. There is no attempt to represent an empty rectangle with integer limits or inverted bounds.

### Nonzero-count initialization

If the initial count is nonzero, the function reads the record-array pointer from descriptor byte offset 4.

It then reads the first record in this order:

1. First component.
2. Second component.

The first component initializes both first-component extrema. The second component initializes both second-component extrema.

Consequently, an initial negative count does not select the zero-count behavior. It still causes the record-array pointer and both components of the first record to be read.

The first record is used directly as the initial result. There is no initial comparison against zero, a numeric limit, or any externally supplied bounds.

### Traversal and reduction

After initialization, the function considers successive records at an eight-byte stride under the recovered layout.

Before processing each candidate record, it compares that record’s address with an end address formed from:

- The originally loaded record-array base address.
- The signed count read again from the descriptor.
- Two integer components per record.

The continuation condition is a strict address comparison: the candidate must precede the computed end address. Equality terminates traversal.

For a stable positive count and valid array, the function therefore processes exactly the stated number of records, including the first record used for initialization.

For each later record, processing proceeds in this order:

1. Read the first component.
2. Compare it against the current first-component minimum and maximum.
3. Read the second component.
4. Compare it against the current second-component minimum and maximum.

All component comparisons are signed comparisons. Equal values leave the corresponding extremum unchanged, with no separately observable tie-breaking result.

The function does not retain a record index or return the location of any extremum. Minima and maxima can come from different records.

### Count rereads

The descriptor count is not captured once for the entire scan. It is read initially to choose the zero or nonzero branch and is read again for subsequent loop-bound evaluations.

The record-array pointer, by contrast, is loaded once on entry to the nonzero branch. Later traversal uses that saved base address rather than reloading the pointer field.

For stable input, this distinction does not change the result. If external activity changes the count during execution, the source does not describe a snapshot reduction. A later bound evaluation can differ from an earlier one.

This observation is not a concurrency guarantee. The source provides no synchronization, atomicity, or memory-ordering mechanism.

### Final writes

No destination write occurs until the zero-count result has been selected or the nonzero traversal has finished.

The source then performs four destination stores in this order:

1. Minimum first component.
2. Minimum second component.
3. Maximum first component.
4. Maximum second component.

The return follows the fourth store.

These are separate stores, not a single atomic result publication. If execution fails during a store, the function supplies no rollback mechanism for earlier stores.

## Side effects

The only explicit writes are the four result stores. The function does not explicitly modify the descriptor or record array.

No allocation, deallocation, ownership transfer, callback, external function call, or explicit global access appears in the supplied body. The descriptor, records, and destination remain caller-managed. Nothing in this function releases memory or establishes a lifetime extending beyond the call.

Destination aliasing matters:

- The scan finishes before the first destination store.
- If destination storage validly overlaps source records, output stores can overwrite those records after their values have been consumed.
- If destination storage validly overlaps descriptor fields, output stores can overwrite those fields after traversal has ended.

Such overlap does not cause an earlier destination store to feed back into the scan, because there are no destination stores during traversal. This does not establish that every possible overlap is valid under the target’s alignment and aliasing rules.

There is no explicit error-reporting path for invalid memory. A nonzero invalid descriptor, invalid record pointer, insufficient record extent, or insufficient destination extent can result in an invalid access. The function does not convert those conditions into a documented return status.

The stated read and write order describes the supplied source’s abstract operations. It is not a promise of instruction ordering or cross-thread visibility for ordinary, non-atomic storage.

## Dependencies

F-960c2eabcfd8 contains no explicit calls to other functions. No additional opaque dependency IDs are required.

Its behavior depends on caller-provided storage and the target execution model:

- Signed interpretation and width of the count and components.
- Representation of the stored record-array pointer.
- Validity of address arithmetic and pointer comparisons.
- Alignment and readable or writable memory extent.
- Stability of input memory during execution.

There is no visible dependency responsible for validating the descriptor, normalizing a negative count, allocating output storage, or recovering from invalid accesses.

## Acceptance examples

These examples are conditional behavioral expectations derived from static evidence, not executed tests. Unless otherwise stated, they assume the recovered scalar widths, valid aligned storage, stable inputs, and defined address operations.

### Absent descriptor

Given a zero descriptor address and a destination initially containing `(9, 8, 7, 6)`, the function returns with the destination unchanged.

It does not attempt to read the count.

### Absent destination

Given a nonzero descriptor address and a null destination, the function returns without reading the descriptor.

The descriptor need not be dereferenceable for this specific early-return path.

### Empty sequence

Given a readable descriptor with count zero and writable destination storage, the result is:

`(0, 0, 0, 0)`

The record-array pointer field need not contain a usable array address because that field is not read on this path.

### Single record

Given count one and the record `(7, -3)`, the result is:

`(7, -3, 7, -3)`

The next candidate address equals the end address, so no second record is read.

### Extrema supplied by different records

Given these three records in order:

- `(4, -2)`
- `(-8, 9)`
- `(6, -5)`

The result is:

`(-8, -5, 6, 9)`

The output is not required to reproduce any input record.

### Signed comparisons

Given records `(-10, -20)` and `(-3, -40)`, the result is:

`(-10, -40, -3, -20)`

Negative components are compared as signed values, not as unsigned bit patterns.

### Component limits

Given records `(-2147483648, 2147483647)` and `(2147483647, -2147483648)`, the result is:

`(-2147483648, -2147483648, 2147483647, 2147483647)`

The reduction compares and copies component values. It does not subtract extrema or calculate a width, height, or area that could overflow.

### Destination overlapping records

Given records `(5, 8)` and `(-2, 10)`, with the destination validly occupying the same four-integer region, the scan first consumes both records. The final region then contains:

`(-2, 8, 5, 10)`

This expectation is conditional on the overlap being valid under the execution model.

### Negative count

Given an initially negative count, acceptance must not require zero output or a no-read return. The first record’s two components are read before the subsequent traversal condition is evaluated.

Under an address model in which the computed negative-count end lies before the first candidate and the comparison is defined, traversal stops after that first record, and its components supply both minima and maxima. That outcome is conditional, not an unconditional contract for malformed descriptors.

## Uncertainties

### Established static behavior

The supplied body establishes the null gate, zero-count special case, first-record initialization, signed component comparisons, count rereads, single record-pointer load, delayed output stores, and output order.

### Inferred intent

The layout and reduction strongly suggest a bounding-extents operation over two-dimensional points. No coordinate system, units, geometry type, or rectangle convention is established.

The function does not compute dimensions, normalize coordinates, or add inclusive-bound adjustments.

### documented_only: negative counts

Negative counts are not rejected. Their initial read behavior is established, but a portable complete outcome is not: forming or comparing an end pointer before the array may violate execution-model requirements.

No unconditional negative-count result should be invented.

### documented_only: integer and address model

The count is represented as signed short and components as signed integer in the supplied source. The numerical widths used above reflect the recovered 16-bit/32-bit model rather than a portable guarantee of those language type names.

At the largest positive signed 16-bit count, doubling the count yields 65,534 integer elements, or 262,136 bytes in the recovered layout. Those quantities fit signed 32-bit arithmetic, but that does not establish that adding the extent to a particular address is valid.

### documented_only: concurrent mutation and faults

There is no established behavior for unsynchronized concurrent mutation, invalid pointers, inaccessible storage, or failures partway through output publication. No memory snapshot, retry, rollback, or fault recovery is provided.

## Evidence

This contract is derived solely from the supplied body for F-960c2eabcfd8.

Static evidence supports:

- Immediate return when either argument is zero.
- A signed count read at descriptor offset 2.
- A record-array pointer read at descriptor offset 4 only for a nonzero initial count.
- Two-component initialization from the first record.
- Independent signed minimum and maximum reduction.
- Count-dependent strict traversal bounds evaluated repeatedly.
- Four destination stores after computation, in the documented order.
- No explicit external calls or memory-management operations.

No runtime tests, executable observations, ABI validation, or independent caller evidence have been performed.
