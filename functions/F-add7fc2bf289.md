<!-- Trusted provenance: F-add7fc2bf289; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-add7fc2bf289

## Purpose

F-add7fc2bf289 rearranges a contiguous sequence of fixed-size, eight-byte records in place, using a caller-supplied comparison callback and a caller-supplied 32-bit context value.

**Established mechanism:** larger ranges are partitioned around a selected record; smaller ranges are processed by insertion sorting. Pending ranges are maintained in two fixed-capacity local arrays rather than through recursive calls.

**Inferred intent:** ascending ordering according to the callback, while preserving each record’s two 32-bit words as a pair. This outcome requires a consistent ordering callback that does not interfere with the records or the function’s temporary storage. Stability is not guaranteed.

The function has no result value and no explicit error-reporting mechanism.

## Interface

The interface consists of four arguments, in this order:

| Argument | Representation | Meaning |
|---|---|---|
| Record base | 32-bit address representation | Start of the contiguous record sequence. |
| Record count | Unsigned 32-bit integer | Number of eight-byte records. |
| Comparison callback | Callable address | Invoked to compare two record addresses. |
| Context | Opaque 32-bit value | Passed unchanged as the callback’s third argument. |

Each record has this layout:

| Byte offset | Width | Treatment |
|---|---|---|
| 0 | 32 bits | First opaque word |
| 4 | 32 bits | Second opaque word |

Successive records begin eight bytes apart. The function does not interpret either word as a key, pointer, integer, or ownership-bearing value. It copies their bit patterns.

The callback dependency is designated **F-000000000001** in this document. This is a document-local opaque identifier for the runtime-supplied callback, not a resolved external identity.

F-000000000001 receives:

1. An address of a record.
2. An address of another record.
3. The original 32-bit context value.

Its result is interpreted as a signed 32-bit integer. Only whether it is strictly greater than zero matters. The intended interpretation is:

- Positive: the first record should follow the second.
- Zero or negative: the first record does not need to follow the second.

For normal, defined sorting behavior when the count exceeds one:

- The record region must be readable and writable for the full sequence.
- Record addresses must support the observed 32-bit accesses.
- The callback must be callable with the described interface.
- Address calculations, address ordering, and signed address differences must remain meaningful in the execution environment.
- The callback must provide a consistent ordering and must not invalidate storage used by the operation.

No validation of these conditions is present.

## Behavior

### Zero and one record

The count is first tested as an unsigned value.

When the count is zero or one, the function returns without reading or writing the record region and without invoking F-000000000001. A usable record address or callable callback is therefore not exercised on these paths.

There is no status distinguishing an empty sequence from a singleton sequence.

### Initial range and arithmetic

For counts greater than one, the initial active range runs from the supplied base through the address of the last record, inclusive.

The last-record address is formed from the base plus eight times the count, minus eight. These are 32-bit address-scale calculations in the supplied representation; there is no overflow check or wider intermediate established by the evidence.

Range length decisions use a signed address difference followed by a right shift of three bits. For an ordinary, non-wrapping range, this gives the number of eight-byte intervals between its endpoints.

A useful sufficient arithmetic precondition is that the endpoint calculation does not wrap and the greatest byte difference between active endpoints remains representable as a nonnegative signed 32-bit value. Merely having an unsigned 32-bit count does not make every such count safe.

### Selection of sorting path

A range with at most one record requires no work.

For a nontrivial range:

- If the endpoint distance exceeds fifteen record intervals, the function partitions the range.
- Otherwise, it performs insertion sorting over that range.

Under ordinary valid address arithmetic, this means:

- Two through sixteen records: insertion-sort path.
- Seventeen or more records: partitioning path.

These thresholds apply separately to each subsequently processed subrange.

### Insertion-sort path

Records are considered from the second record of the active range through its last record, in increasing address order.

For each considered record:

1. Its first word is read into a temporary record.
2. Its second word is read into the second word of that temporary record.
3. Preceding records are examined backward, beginning immediately before the considered record.
4. F-000000000001 receives the preceding array-record address as its first argument and the temporary-record address as its second argument.
5. A positive result causes that preceding record to be copied forward by one record position.
6. A nonpositive result, or reaching the beginning of the range, ends the backward search.
7. The temporary record is written into the resulting position.

A forward copy transfers the first word before the second word. The final placement likewise writes the temporary’s first word before its second.

The callback’s second record argument on this path is **not** necessarily an address within the input sequence. It points to temporary storage belonging to this invocation. Its useful lifetime is limited to the ongoing operation; the function does not grant ownership or a persistent reference.

Even if a considered record does not move, its two words are still copied into temporary storage and subsequently written back.

For a comparison callback that is consistent and non-mutating, this local insertion procedure does not move a preceding record past the temporary record when their comparison is zero. That local property does not establish stability for the complete function.

### Partitioning path

The pivot-selection position is approximately the middle of the active range. More precisely, its record index relative to the lower endpoint is the floor of half the endpoint distance measured in record intervals.

The complete record at that position is exchanged with the record at the lower endpoint. The lower endpoint then serves as the pivot’s array location during scanning.

Two scan positions are used:

- A forward position starting one record after the pivot.
- A backward position starting at the upper endpoint.

The forward scan advances past records for which comparing the pivot against that record produces a positive result. A forward comparison is performed only while the forward position is strictly below the current backward position.

The backward scan compares the backward record against the pivot while the forward position is at or below the backward position. A positive result moves the backward position back by one record.

Thus, the callback argument order differs between scans:

| Scan | First record argument | Second record argument |
|---|---|---|
| Forward | Pivot at the lower endpoint | Forward record |
| Backward | Backward record | Pivot at the lower endpoint |

When the forward position remains below the backward position after those scans, the two records are exchanged. The backward position is then decreased by one record, and scanning resumes by advancing the forward position.

When the positions no longer permit that exchange, the pivot record is exchanged with the record at the final backward position. This establishes the pivot’s split position.

The split record itself is excluded from subsequent subranges.

### Subrange scheduling

The two resulting sides are compared by their number of record intervals from the split:

- The larger side is deferred when it contains at least two records.
- The smaller side is processed immediately.
- If the two side sizes are equal, the right side is deferred and the left side is processed immediately.

Deferred ranges are kept in a last-in, first-out work list. Each entry consists of a lower endpoint and an upper endpoint.

There are 64 local slots for lower endpoints and 64 corresponding slots for upper endpoints. There is no explicit capacity check. Processing the smaller side immediately is consistent with keeping pending depth small under valid range arithmetic; the implementation does not independently detect corrupted or nonsensical range calculations.

### Record exchange ordering

Exchanges are performed as individual 32-bit accesses, not as an indivisible eight-byte operation.

For each exchange, the observed order is:

1. Save the first record’s first word.
2. Save the first record’s second word.
3. Read the second record’s second word.
4. Read the second record’s first word and write it to the first record’s first-word location.
5. Write the previously read second word to the first record’s second-word location.
6. Write the saved first word to the second record’s first-word location.
7. Write the saved second word to the second record’s second-word location.

This order applies to pivot selection, scan-position exchanges, and final pivot placement. A final pivot exchange may address the same record on both sides; the reads and writes are not explicitly skipped in that case.

## Side effects

The function directly modifies only the supplied record region and its own working storage. No direct allocation, deallocation, input/output operation, or global-state access is established by the supplied body.

The record region can be temporarily unsorted and can contain intermediate duplicate word values while records are being moved. Eight-byte records are not updated atomically. Concurrent observation or mutation is outside the established safe-use conditions.

Callback calls are interleaved with reads and writes:

- Insertion sorting invokes the callback before each possible backward shift.
- Partition scanning invokes it after pivot selection and between exchanges.
- No callback is invoked within the individual word operations of an exchange.

The callback’s effects are not constrained by evidence about this function. In particular, it must not be assumed to preserve globals, avoid allocation, or refrain from releasing memory merely because the sorting function itself has no such direct operations.

For a callback that only observes records and context, successful completion preserves the multiset of complete eight-byte records. If the callback changes array contents, the temporary record, or storage validity, neither record preservation nor sorted output is established.

The function does not acquire ownership of the array, record contents, callback, or context. Copying pointer-shaped words does not imply retaining or releasing their referents.

## Dependencies

### F-000000000001

**Role:** runtime comparison callback.

**Established call behavior:**

- Receives two record addresses and the same 32-bit context value on each invocation.
- Returns a signed 32-bit comparison result.
- Is tested only for a strictly positive result.
- Can receive either two array addresses or an array address followed by a temporary-record address.
- May observe records after prior in-place rearrangements.
- Is not invoked for counts zero and one.

**Required for the intended sorting guarantee:** consistent comparison behavior over the records being sorted, with no interference that changes the relevant records, temporary record, or storage validity.

**Not established:** global-state preservation, ownership behavior, failure behavior, reentrancy, synchronization, or an exception/unwinding contract.

No other external function dependency is visible in the supplied body.

## Acceptance examples

These examples are conditional behavioral expectations derived from the supplied body, not runtime test results.

### Empty sequence

Given count zero, the function returns without dereferencing the record base and without invoking F-000000000001.

Acceptance does not require a meaningful ordering relation because no comparison occurs.

### Singleton sequence

Given count one, an existing record remains untouched by this function, and the callback is not invoked.

### Two records already ordered

Given two records A and B, and a non-mutating callback reporting that A does not follow B:

- The callback receives A’s array address first and a temporary copy of B second.
- A is not shifted.
- B’s temporary copy is written back into B’s position.

The final sequence remains A, B.

### Two records in reverse order

Given two records A and B, and a non-mutating callback returning positive for A compared with B:

- B is held as a two-word temporary record.
- A is copied forward into the second position.
- B is written into the first position.

The final sequence is B, A, with each record’s two words preserved together.

### Path boundary

Given valid, non-wrapping storage:

- Sixteen records enter insertion sorting for the initial range.
- Seventeen records first enter partitioning.
- For seventeen records, pivot selection exchanges the initial first record with the record at zero-based index eight before any comparison callback is made.

### Equal comparison results

Given a callback that always returns zero:

- Small insertion-sorted ranges do not shift preceding records.
- Partitioned ranges can still exchange records, including records that compare equal.

Acceptance must therefore not require preservation of original order among equal records.

### Context forwarding

Given any 32-bit context bit pattern and at least one callback invocation, the callback receives that same bit pattern as its third argument. No interpretation or dereference of that value by this function is established.

### Invalid ordering or arithmetic

A callback that changes its answers inconsistently, or an input whose address calculations wrap, is outside the sorting guarantee. There is no required error return, recovery action, or diagnostic for either case.

## Uncertainties

The following are **documented_only**:

- Exact external calling convention and address-space rules beyond the visible 32-bit representation.
- Platform byte order and precise alignment requirements.
- Callback identity and its full behavior.
- Callback failure, exception, nonlocal-exit, and reentrancy semantics.
- Behavior for invalid pointers, inaccessible memory, arithmetic overflow, or unsupported address relationships.
- Outcomes when the callback mutates records, temporary storage, globals relevant to comparison, or allocation lifetimes.
- Any externally required stability or performance guarantee.

The interpretation as an ascending sort is inferred from comparison direction and movement. The individual accesses, comparison conditions, thresholds, and range scheduling are established by static evidence.

## Evidence

The supplied function body establishes:

- Four input arguments and no returned result.
- An unsigned count guard excluding counts zero and one.
- Eight-byte record spacing and two 32-bit words per record.
- Inclusive lower and upper range endpoints.
- Two fixed-capacity, 64-entry local endpoint arrays.
- Insertion sorting for endpoint distances of at most fifteen record intervals.
- Midpoint-based pivot selection and bidirectional partition scanning for larger ranges.
- Strictly positive callback-result tests.
- The callback argument orientations and context forwarding described above.
- Word-by-word record copying and exchange ordering.
- Deferred larger-side processing, with the smaller side processed immediately.
- No visible validation, explicit error reporting, direct memory management, or direct global-state operation.

No runtime tests have been performed. No dynamic behavior, timing result, or platform-specific fault behavior is claimed.
