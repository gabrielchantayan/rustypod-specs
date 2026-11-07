<!-- Trusted provenance: F-621afaaa3bec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-621afaaa3bec

## Purpose

F-621afaaa3bec attempts to register one indexed record in a table’s active-record pointer array. On first registration, it marks the record and searches the existing active entries for the first record whose signed interval endpoints satisfy an inclusive intersection test. It stores that record’s address as a link, or stores a null link if no match is found.

The function has no explicit return value. Registration failures are reported through two opaque dependencies. A capacity failure is not transactional: the selected record has already been marked and its link initialized or assigned before the failure is reported.

“Registration,” “active record,” and “interval” are descriptive interpretations of the observed operations, not independently established domain semantics.

## Interface

### Inputs and result

The function takes:

- A pointer to a table descriptor composed of 32-bit words.
- A 32-bit unsigned record index.

It returns no explicit value. Callers must observe table state, record state, or diagnostic effects to distinguish outcomes.

The supplied representation uses 32-bit addresses and 32-bit integer fields. Address calculations, including index multiplication and address addition, must be interpreted at that width.

### Table descriptor layout

| Byte offset | Width | Observed use |
|---|---:|---|
| +0 | 32 bits | Total record count; also the limit checked before appending an active pointer |
| +4 | 32 bits | Active-record count |
| +8 | 32 bits | Base address of the contiguous record storage |
| +12 | 32 bits | Not directly accessed |
| +16 | 32 bits | Base address of the active-record pointer array |

Each active-array entry is one 32-bit record address.

The field at +0 serves both as the valid-index bound and the append limit. No separate allocation capacity is checked.

### Record layout

Records have a stride of 28 bytes.

| Byte offset | Width | Observed use |
|---|---:|---|
| +0 | 32 bits | Start value, interpreted as signed during comparisons |
| +4 | 32 bits | Span or endpoint increment, used in 32-bit addition |
| +8 | 32 bits | Not directly accessed |
| +12 | 32 bits | Not directly accessed |
| +16 | 32 bits | Flags; bit mask `0x00000004` identifies an already-marked record |
| +20 | 32 bits | Link to the first matching active record, or zero |
| +24 | 32 bits | Not directly accessed |

No interpretation of the other flag bits is established.

### Memory and ownership preconditions

For an ordinary successful call, the descriptor must be readable, its active count writable, the selected record accessible, and the active pointer array sufficiently allocated for both scanning and appending. Every scanned entry must identify a record with the fields needed by the comparison.

The function performs no visible pointer validation, allocation-size validation, or verification that existing active entries belong to the contiguous record storage.

No allocation or release operation is directly present. The function writes borrowed record addresses into existing storage. This does not establish ownership transfer, reference counting, or the lifetime required by later consumers.

## Behavior

### 1. Select the record address and validate the index

The selected record address is formed from the record-storage base plus the unsigned index multiplied by 28. Both multiplication and address addition are 32-bit operations.

The record-storage base is read and the selected address is formed before the index is checked against the total record count. Therefore, even an invalid-index call requires access to the descriptor’s record-storage-base field.

The index is compared with the total count as an unsigned value:

- If the index is less than the total count, processing continues.
- Otherwise, the function takes the invalid-index diagnostic path.

No selected-record field is directly read or written on the invalid-index path. Forming the address is not itself a record dereference.

### 2. Return immediately for an already-marked record

For a valid index, the function reads the selected record’s flags.

If bit `0x00000004` is set, the function returns immediately. It does not scan active records, reset the link, append a pointer, or report a capacity error.

The flag alone controls this shortcut. The function does not verify that a marked record actually appears in the active array. Consequently, a marked-but-unregistered record is not repaired by another call.

### 3. Mark the record and initialize its link

If the bit is clear, the function writes the flags with bit `0x00000004` set, preserving the other flag bits.

It then:

1. Reads the active-array base.
2. Reads the active-record count to establish the scan length.
3. Writes zero to the selected record’s link field.

The record is thus marked before the link reset and before any capacity check.

### 4. Search active records in array order

The function examines the number of entries captured at scan initialization, stopping at the first match. It does not sort, deduplicate, or validate entries.

For each examined entry, the direct access order is:

1. Read the selected record’s start value.
2. Read the candidate address from the current active-array slot.
3. Read the selected record’s span and form its endpoint.
4. Read the candidate’s start value.
5. If the candidate start is no greater than the selected endpoint, read the candidate span and form the candidate endpoint.

The selected start and span are read again for each examined candidate; they are not captured once for the entire scan.

An endpoint is the low 32 bits of start plus span, interpreted as a signed 32-bit value for comparison. The comparisons implement signed relations rather than simply testing the sign of a potentially overflowing subtraction.

A candidate matches exactly when both conditions hold:

- Candidate start is less than or equal to the selected endpoint.
- Candidate endpoint is greater than or equal to the selected start.

If the first condition fails, the candidate span need not be read.

On a match, the function writes the candidate address into the selected record’s link and stops scanning. If no candidate matches, the link remains zero.

The comparisons are inclusive. Equal endpoints can therefore match. There is no visible requirement that spans be positive, endpoints be ordered, or additions avoid signed-range wraparound. For irregular or wrapping inputs, the two stated signed comparisons remain the established behavior; ordinary geometric interval intuition is not sufficient.

Existing entries may include duplicate pointers or even the selected record’s own address. There is no exclusion check.

### 5. Re-read counts and attempt to append

After the scan, the function reads the active-record count again and compares it with the total record count using an unsigned comparison.

If the active count is less than the total count, the function:

1. Writes the active count increased by one.
2. Reads the active-array base again.
3. Writes the selected record’s address at the slot indexed by the pre-increment count.
4. Returns.

The count write precedes the pointer-slot write. The function does not provide an atomic publication mechanism.

The append uses the newly read count and newly read array base, rather than necessarily using the values captured for scanning. With ordinary non-overlapping storage and no external mutation, the values coincide. The distinct accesses remain relevant when storage aliases or is externally modified.

A scan match does not suppress appending. The match determines only the link.

If the active count is not less than the total count, the function reports a capacity error without directly incrementing the count or writing an append slot.

### 6. Report errors

Two conditions lead to diagnostics:

- The index is outside the unsigned record-count bound.
- A previously unmarked record has been processed, but the active count is not below the append limit.

For either condition, the function selects a corresponding diagnostic message and calls F-72d6ac25a5a5 with:

- A shared diagnostic-context value.
- The selected message.
- An address into the current call’s stack area.

It then calls F-58e7ae060acf with the shared diagnostic-context value.

The shared context is loaded separately for the two calls. It is not established that the first dependency preserves that context or any other global state.

If both dependencies return normally, this function returns. Whether either dependency can terminate execution, transfer control elsewhere, or modify relevant memory is unknown.

## Side effects

### Successful registration

For an initially unmarked record with append capacity:

- The registration flag is set.
- The link is set to zero or to the first matching candidate address.
- The active count is incremented.
- The selected record address is written into the next active-array slot.

### Already-marked record

There are no direct writes and no dependency calls after detecting the set flag.

### Invalid index

There are no direct record, count, or active-array writes before diagnostics. Effects performed by the diagnostic dependencies are unknown.

### Capacity failure

Before reporting failure, the function has already:

- Set the selected record’s registration flag.
- Cleared its link.
- Possibly assigned the first matching candidate address to that link.

There is no rollback.

With stable, non-overlapping storage, a subsequent call for that same record returns immediately because the flag is already set, even though the failed call did not append it.

### Aliasing and concurrency

No synchronization is visible. Concurrent use is not established as supported.

The ordinary state descriptions above assume descriptor fields, pointer-array entries, and writable record fields do not unexpectedly overlap. The function does not enforce that assumption. With aliasing, an earlier write may affect a later read—for example, a record-field write could alter a count or pointer field if they occupy the same storage.

No callee preservation, ownership, or cleanup guarantee should be inferred from the absence of further direct operations.

## Dependencies

| Opaque ID | Role established by the supplied representation | Unknowns |
|---|---|---|
| F-72d6ac25a5a5 | First call on either error path; receives a shared context, selected diagnostic message, and stack-area address | Exact signature, formatting rules, stack-data interpretation, memory effects, global preservation, retention of passed addresses, and control-flow behavior |
| F-58e7ae060acf | Second call on either error path; receives a fresh load of the shared context | Whether it finalizes a diagnostic, terminates execution, releases resources, or returns normally |

Neither dependency is directly called on successful registration or the already-marked shortcut.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not results of executed tests. Unless stated otherwise, storage is valid and non-overlapping, values remain stable, and diagnostic dependencies return normally.

### Invalid index

Given a total count of 3 and an index of 3:

- The function takes the invalid-index diagnostic path.
- It does not dereference the calculated selected record.
- It calls F-72d6ac25a5a5, then F-58e7ae060acf.

An index whose 32-bit representation is `0xFFFFFFFF` is a large unsigned index, not a negative index accepted by signed comparison.

### Already marked despite full capacity

Given a valid selected record whose flags already contain `0x00000004`, and an active count equal to the total count:

- The function returns without reporting capacity failure.
- The existing link and active-array contents are unchanged by direct function operations.

### Empty active array

Given total count 2, active count 0, and an unmarked valid record:

- No candidate record is read.
- The link becomes zero.
- The flag is set.
- The active count becomes 1 before slot 0 receives the selected record address.

### Inclusive endpoint contact

Given a selected start of 10 and span of 5, and a candidate start of 15 and span of 4:

- The selected endpoint is 15.
- Both signed comparison conditions hold.
- The candidate qualifies despite contact occurring only at the boundary.

Given a candidate start of 16 instead, the first condition fails and that candidate does not qualify.

### First matching entry wins

Given two qualifying candidates in different array slots:

- The earlier candidate’s address becomes the link.
- The later candidate is not examined after the match.
- The selected record is still appended if capacity permits.

### Zero-span record

Given a selected start of 10 and span of 0, and a candidate with start 10 and span 0:

- Both endpoints are 10.
- The candidate qualifies.

This establishes point-contact behavior without implying how all negative-span inputs should be interpreted geometrically.

### Wrapped endpoint

Given selected start 2,147,483,647 and span 1:

- The endpoint bit pattern is `0x80000000`, interpreted as −2,147,483,648.
- A candidate starting at 0 fails the first comparison, regardless of whether its mathematical interval would intersect an unbounded-integer interpretation of the selected range.

### Capacity failure leaves a marked record

Given total count 2, active count 2, and a valid unmarked selected record:

- The function marks the record and computes its link before reporting capacity failure.
- It does not directly append the record or increment the count.
- A later call with otherwise unchanged state returns through the already-marked shortcut.

## Uncertainties

**documented_only — Domain meaning.** The interval interpretation is supported by the endpoint arithmetic and comparisons. Exact application-level meanings of the fields and link are not established.

**documented_only — Valid input invariants.** Nonnegative spans, nonwrapping endpoints, active-count consistency, unique entries, and non-overlapping allocations may be intended invariants, but none is enforced here.

**documented_only — Diagnostic contract.** Message formatting, stack-argument structure, context lifetime, dependency memory effects, and nonreturning behavior are unavailable.

**documented_only — Invalid-memory behavior.** No explicit recovery or error result is present for inaccessible memory, invalid candidate pointers, or insufficient allocated storage.

**documented_only — Ownership and concurrency.** Address retention, lifetime management, supported sharing, and synchronization requirements are not established beyond the direct reads and writes described above.

## Evidence

The established behavior comes solely from the supplied static function representation and dependency mapping: field offsets, widths, address arithmetic, branches, access order, flag mask, scan order, endpoint comparisons, append ordering, and error-call sequence.

The signed-comparison interpretation accounts for the represented subtraction-overflow checks. It does not assume that endpoint additions are unbounded or nonwrapping.

Domain terminology is inferred intent. Dependency internals and environmental guarantees remain unknown. No runtime tests have been performed.
