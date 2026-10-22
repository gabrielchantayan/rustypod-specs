<!-- Trusted provenance: F-f25f0cb9bed0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f25f0cb9bed0

## Purpose

F-f25f0cb9bed0 conditionally writes two 32-bit values into one of two pairs of positions in caller-supplied storage.

A selector of 5 targets the first pair of 32-bit positions. A selector of 6 targets the second pair. Both recognized selectors produce two ordered writes and return zero. Every other selector returns the 32-bit value 23 without accessing the destination storage.

The established behavior is a small selector-controlled storage update. Any higher-level interpretation—such as configuring fields, selecting channels, or updating a particular record type—is unknown.

## Interface

The function has four inputs and one 32-bit result.

| Input | Width or representation | Behavioral role |
|---|---|---|
| Destination | Pointer to storage addressed in 32-bit units | Supplies the base address for the selected writes. |
| Selector | 32-bit integer in the supplied representation | Recognized only when equal to 5 or 6. |
| First value | 32-bit opaque value | Written to the earlier position of the selected pair. |
| Second value | 32-bit opaque value | Written to the later position of the selected pair. |

The values are treated as complete 32-bit quantities. There is no established signed interpretation, numeric conversion, masking, byte swapping, or relationship between them.

The result is also 32 bits:

| Result | Condition |
|---|---|
| 0 | Selector is 5 or 6, and execution reaches the normal return after both writes. |
| 23, hexadecimal 0x17 | Selector is neither 5 nor 6. |

The destination layout relevant to this function is:

| Position | Byte offset from destination | Selected by | Stored value |
|---|---:|---:|---|
| First 32-bit position | 0 | 5 | First value |
| Second 32-bit position | 4 | 5 | Second value |
| Third 32-bit position | 8 | 6 | First value |
| Fourth 32-bit position | 12 | 6 | Second value |

The byte offsets follow from the supplied four-byte element representation. The source does not establish the machine’s byte order.

For defined successful execution, both selected 32-bit destinations must be writable and satisfy the execution environment’s addressing, alignment, and object-access requirements. The function does not check these conditions.

A selector of 5 does not require the function to access offsets 8 through 15. A selector of 6 does not require it to read or write offsets 0 through 7. Whether an address expression is valid for a particular allocated object remains an execution-environment constraint; the offset table does not independently establish allocation validity.

## Behavior

### Selector dispatch

The selector is checked before any destination access.

If the selector equals 5, the function selects the pair beginning at byte offset 0.

Otherwise, it checks whether the selector equals 6. If it does not, the function returns 23 immediately. There is no destination read or write on this rejection path.

If the selector equals 6, the function selects the pair beginning at byte offset 8.

There is no range-based acceptance. Values below 5, between the recognized values only in some wider representation, or above 6 receive no special treatment. At the established 32-bit interface, only equality to the two recognized integer values matters.

### Successful write sequence

For selector 5, the observable source-level sequence is:

1. Store the complete first value at byte offset 0.
2. Store the complete second value at byte offset 4.
3. Return zero.

For selector 6, the sequence is:

1. Store the complete first value at byte offset 8.
2. Store the complete second value at byte offset 12.
3. Return zero.

The second write follows the first. There is no read of either previous destination value, no comparison against the new value, and no conditional omission of a write when the stored bits already match.

Each selected position is replaced, rather than incremented, merged, or updated by a bitwise operation. No arithmetic is performed on either supplied value.

The offsets are fixed; the selector is not multiplied into an arbitrary index. There is consequently no source-level value-dependent index calculation or value arithmetic whose overflow affects selection. Validity of the fixed-offset address calculations still depends on the destination and execution environment.

### Completion and failure boundaries

Zero is the normal completion result after the selected pair has been written. It is not a validation result issued before the writes.

The explicit rejection result, 23, concerns only an unsupported selector. The function contains no separate result for an inaccessible destination, insufficient storage, misalignment, or a failed memory access.

If an access faults or otherwise violates the environment’s rules, the supplied source does not establish a normal return value or recovery behavior. In particular, there is no rollback mechanism: the first store precedes the second, so this contract does not promise that a failure affecting the second store leaves the first position unchanged.

The source-level ordering does not establish concurrent atomicity, processor memory barriers, or visibility guarantees to other threads or devices.

## Side effects

On a recognized selector and normal completion, the function writes exactly two 32-bit positions in destination-relative storage.

It does not explicitly read destination contents. The unselected pair is not accessed by the function. Statements that those bytes remain unchanged assume ordinary writable memory and exclude external writers or special memory-mapped behavior.

The function contains no explicit allocation, deallocation, ownership transfer, pointer retention, global-state access, logging, or external call. The destination remains caller-supplied storage; the supplied function establishes no release obligation and performs no release operation.

If the caller deliberately points the destination into global storage or another shared object, the selected writes modify that storage. The absence of a named global access is not a promise that the memory reached through the destination is private.

An unsupported selector produces no explicit side effects beyond returning 23. In particular, that path does not dereference the destination. This is a control-flow property, not a general endorsement of passing pointer representations prohibited by the calling environment.

## Dependencies

No dependency function calls are present in the supplied body.

The established behavior relies on the execution environment for:

- Representation and passing of the pointer and 32-bit arguments.
- Writable destination storage for the selected positions.
- Valid pointer arithmetic and memory access.
- The consequences of invalid, unaligned, protected, or special-purpose addresses.
- Delivery of the 32-bit return value to the caller.

No unknown callee is invoked, so there is no callee preservation or memory-ownership assumption to make. Calling convention details, byte order, and memory-access atomicity remain unspecified.

## Acceptance examples

These examples are conditional requirements derived from static evidence, not records of executed tests. Unless otherwise stated, assume ordinary valid writable storage, no concurrent modification, and normal completion.

### First-pair selection

Given four 32-bit positions initially containing 10, 20, 30, and 40, a selector of 5 with supplied values 100 and 200 must:

- Write 100 to the first position before writing 200 to the second.
- Return zero after both writes.
- Leave the third and fourth positions unaccessed by this function.

The resulting four-position contents are 100, 200, 30, and 40.

### Second-pair selection

With the same initial contents, a selector of 6 and supplied values 100 and 200 must write the third position before the fourth and return zero.

The resulting contents are 10, 20, 100, and 200. The first two positions are not read or written by this function.

### Unsupported selector

For selector 4, 7, 0, or −1, the function must return 23 without reading or writing destination storage. The supplied values do not affect this outcome.

The same rule applies to every other representable selector except 5 and 6.

### Full-width value preservation

For selector 5, a first value with bit pattern 0xFFFFFFFF and a second value with bit pattern 0x80000000 must be stored as those complete 32-bit patterns in the first and second positions, respectively.

No signed-range rejection or truncation below 32 bits is established.

### Repeated selection

Two normally completing calls with selector 6 overwrite the same two selected positions. The later call replaces both values written by the earlier call; it does not append data or advance an internal position.

### Invalid destination distinction

With an unsupported selector, the function does not attempt a destination access.

With selector 5 or 6, an invalid destination does not have an established “return 23” outcome. Such a call falls outside the successful-access preconditions, and its consequences are environment-dependent.

## Uncertainties

### Established static evidence

The supplied body establishes the two recognized selectors, the four fixed destination positions, the order of the two stores on each successful path, the absence of destination reads, and the two normal return values.

### Inferred intent

The layout is consistent with selecting one of two adjacent pairs of 32-bit fields. That interpretation is descriptive only. No field names, record meaning, or application-level validity rules are established.

### documented_only unknowns

The following remain unknown and are not additional requirements:

- The destination’s higher-level object type and total allocated size.
- Whether supplied values represent numbers, addresses, flags, or other data.
- The application-level meaning of result 23.
- Exact calling convention, byte order, and alignment rules.
- Whether any caller uses shared, volatile, or memory-mapped storage.
- Hardware store atomicity and cross-thread visibility.
- Fault handling and externally observable effects of an invalid access.

## Evidence

This contract is based solely on static inspection of the supplied body for F-f25f0cb9bed0. That body contains selector comparisons against 5 and 6, two ordered 32-bit stores on each accepted path, an immediate return of 23 on rejection, and a return of zero after accepted-path stores.

No runtime tests have been performed. Acceptance examples specify conditional behavior; they do not claim observed execution.
