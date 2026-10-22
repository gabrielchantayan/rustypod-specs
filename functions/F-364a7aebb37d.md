<!-- Trusted provenance: F-364a7aebb37d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-364a7aebb37d

## Purpose

F-364a7aebb37d searches an indexed sequence of one-byte values associated with a caller-supplied memory base. It returns the index of the first matching value through a caller-supplied output location.

The search uses a signed count stored at a fixed offset from the base. Each candidate byte is separated from the next by an 80-byte stride. The function reports success with status 0 and exhaustion with status 3.

This is an exact numeric comparison, not a comparison against the low byte of the requested value. A requested value greater than 255 cannot match a candidate byte.

The interpretation of the sequence as records, and of the compared byte as an identifier or type field, is inferred intent only. The established behavior is a strided byte search.

## Interface

### Inputs

| Input | Representation | Meaning |
|---|---|---|
| Memory base | Address-bearing input represented by a 32-bit integer | Origin for the count and candidate-byte accesses |
| Requested value | Unsigned 32-bit integer | Value compared with each zero-extended candidate byte |
| Output location | Address of a signed 32-bit integer | Receives the matching zero-based index on success |

The supplied representation uses 32-bit integer values and a four-byte return value. It does not establish a portable interface for arbitrary pointer widths or address spaces.

### Memory layout used

All offsets and strides below are in bytes.

| Location relative to the memory base | Access width | Interpretation |
|---|---:|---|
| 4224, hexadecimal 0x1080 | 4 bytes | Signed 32-bit candidate count |
| 4750, hexadecimal 0x128E, plus 80 times the candidate index | 1 byte | Unsigned candidate value |

The 80-byte stride establishes the spacing between inspected bytes. It does not establish the layout, ownership, or meaning of the intervening bytes.

### Outputs

- **Status 0:** A candidate matched. The function writes its zero-based index to the output location before returning.
- **Status 3:** The search reached the limit established by a count read. The function does not write the output location.
- No other ordinary return status is present in the supplied body.

On success, the output is a four-byte signed integer. On exhaustion, the function leaves that location untouched; status 3 does not imply that the output contains a particular sentinel value.

### Preconditions

For an ordinary, defined execution:

- The count location must support the required four-byte read.
- Every candidate address actually reached must support a one-byte read.
- If a match occurs, the output location must support a four-byte integer write.
- The required address calculations must be valid under the execution environment.
- Alignment requirements, where applicable, must be satisfied.

The function performs no explicit validation of these conditions. In particular, the output location is not checked before searching. A path that returns status 3 does not dereference it.

## Behavior

### Search range and count handling

The search begins with candidate index zero.

Before examining each candidate, the function reads the signed 32-bit count from offset 4224 and compares it with the current index. If the count is less than or equal to that index, the function immediately returns status 3.

Consequences include:

- A zero count causes immediate exhaustion.
- A negative count also causes immediate exhaustion.
- Neither case reads a candidate byte or writes the output.
- For a stable positive count, only indices from zero through count minus one are eligible for examination.

The count read is inside the search loop in the supplied body. The function does not explicitly snapshot the count once at entry. If an execution environment permits the count to change during a call, a later bound check can therefore encounter a different value.

This does not establish thread safety, atomicity, volatile access semantics, or a defined result for unsynchronized concurrent mutation. Those properties require information absent from the supplied evidence.

### Candidate access and comparison

When the bound check permits examination, the function reads exactly one candidate byte at the base-relative location:

**4750 + 80 × current index**

The byte is interpreted as an unsigned value from 0 through 255 and compared numerically with the full unsigned 32-bit requested value.

The function does not truncate the requested value to eight bits. For example, requested value 256 does not match a candidate byte of zero.

If the byte differs from the requested value, the index increases by one and the next iteration begins with another count read.

If the byte matches, the function stops searching immediately. It writes the current index to the output location and returns status 0. There is no further count read between the matching byte read and that write in the supplied body.

For stable memory, this ordering selects the lowest matching index. Later candidates are not examined after a match.

### Access order

For each unsuccessful candidate, the specified access order is:

1. Read the count.
2. If the count permits the current index, read that candidate byte.
3. After a mismatch, advance to the next index.

Successful termination follows a count read, a matching byte read, a four-byte output write, and return of status 0.

Exhausted termination follows a count read that rejects the current index and return of status 3. No candidate is read for that rejected index.

These are the explicit memory accesses in the supplied function body. They are not a claim about physical bus transactions, processor speculation, or compiler behavior under an unspecified memory model.

### Arithmetic limits

The index is signed and begins at zero. It advances only after a bound check has admitted it and the candidate has failed to match.

Within the represented signed 32-bit value range, the loop does not require index wraparound. Even with the largest positive count, index 2,147,483,647 fails the bound check rather than being examined and incremented again.

Address arithmetic is a separate limitation. The candidate displacement increases by 80 bytes per index, and large indices can exceed signed arithmetic or address-space limits. The function contains no explicit overflow check or allocation-size check.

A positive count alone therefore does not prove that all implied candidate accesses are valid. Wraparound addressing is not an established supported behavior; exact overflow handling is documented_only.

## Side effects

The only explicit write is the four-byte store of the matching index to the supplied output location.

There is no explicit:

- Write to the count.
- Write to a candidate byte.
- Allocation or deallocation.
- Ownership transfer.
- Resource acquisition or release.
- Callback, logging operation, or external function call.

The absence of an explicit write through the base does not guarantee that the associated storage remains unchanged. The output location is not checked for overlap with the count, candidate bytes, or other storage.

If such overlap is valid in the execution environment, a successful output write can alter that overlapping storage. The write occurs only after the matching byte has been read, and the function returns without examining another candidate. An exhaustion path performs no corresponding write.

The function assumes access to caller-provided memory. It neither establishes nor extends that memory’s lifetime.

## Dependencies

No external function calls appear in F-364a7aebb37d. There are no callee dependencies to identify by opaque function ID.

Execution depends on the caller supplying suitable storage and on the execution environment supporting the required reads, write, comparisons, and address calculations.

No claim about another function preserving global state, retaining memory, or releasing memory is needed or supported.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. No runtime tests have been performed.

Unless stated otherwise, each example assumes stable memory, valid required addresses, valid arithmetic, and a writable output location when a match occurs. Candidate lists describe only the inspected bytes at 80-byte intervals, not contiguous byte arrays.

| Conditions | Required result |
|---|---|
| Count is 0; output initially contains 41 | Return 3; preserve output as 41; read no candidate byte |
| Count is −1; output initially contains 41 | Return 3; preserve output as 41; read no candidate byte |
| Count is 1; candidate 0 is 7; requested value is 7 | Write 0 to output; return 0 |
| Count is 1; candidate 0 is 7; requested value is 8 | Return 3; do not write output |
| Count is 4; candidates are 9, 2, 9, 4; requested value is 9 | Write 0; return 0; do not examine candidates 1 through 3 |
| Count is 4; candidates are 9, 2, 9, 4; requested value is 4 | Write 3; return 0 |
| Count is 3; candidates are 1, 2, 3; requested value is 4 | Return 3; do not write output |
| Count is 1; candidate 0 is 255; requested value is 255 | Write 0; return 0 |
| Count is 1; candidate 0 is 0; requested value is 256 | Return 3; do not write output; do not treat 256 as zero |
| Count is 1; candidate 0 is 255; requested value is 4,294,967,295 | Return 3; do not write output |

Additional ordering and boundary criteria:

- With stable count 2, a matching byte at index 2 is outside the search range. It must not be read.
- If an admitted execution model permits count changes, and the count becomes 1 after a mismatch at index 0 but before the next bound read, that next read causes status 3 without reading candidate 1.
- If candidate 0 matches and the output validly overlaps the count storage, the function writes zero to that output location and returns 0. The resulting count change does not trigger another bound check.
- With a nonpositive count, an unusable output location alone does not cause an explicit output access. The count read must still be valid.

Invalid-memory and overflowing-address cases have no specified status result in this contract.

## Uncertainties

### Inferred intent

The count and 80-byte spacing suggest a sequence of records with one searched byte per record. Neither the record interpretation nor the semantic meaning of that byte is established.

### documented_only

The supplied evidence does not establish:

- The allocation size or full structure layout associated with the base.
- Whether the count accurately describes accessible candidates.
- The byte’s domain meaning.
- Platform byte order, integer-access alignment rules, or memory protections.
- A portable pointer-width contract.
- Exact behavior when address arithmetic exceeds its valid range.
- Hardware fault or exception behavior for invalid accesses.
- Synchronization guarantees for concurrently changing memory.
- Whether overlapping output and input storage is supported by the broader application.
- Application-level names or meanings for statuses 0 and 3 beyond their demonstrated outcomes.

These unknowns do not justify inventing validation, recovery behavior, ownership rules, or additional return statuses.

## Evidence

**Established static evidence:** The supplied body initializes an index to zero, performs a signed count comparison before each candidate access, reads candidate bytes at an 80-byte stride, compares each unsigned byte with an unsigned 32-bit request, and increments after mismatches. It writes the index and returns 0 on a match, or returns 3 when a count read excludes the current index. It contains no external call or explicit cleanup operation.

**Derived consequences:** First-match selection for stable memory, unchanged output on exhaustion, immediate exhaustion for nonpositive counts, and inability of requests above 255 to match follow from those operations.

**Verification boundary:** This document is based solely on the supplied static representation. No runtime tests, execution traces, allocation metadata, or independent implementation evidence were provided.
