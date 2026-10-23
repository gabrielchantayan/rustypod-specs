<!-- Trusted provenance: F-d5c8ead6793a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d5c8ead6793a

## Purpose

F-d5c8ead6793a performs a comparison-driven search over four-byte records and returns a raw 16-bit payload. Each record contains an unsigned 16-bit key followed by a 16-bit payload.

For stable records whose keys are in nondecreasing order, with nonwrapping search arithmetic and valid record addresses, the function selects the **last record whose key is less than or equal to the unsigned 32-bit query**, provided such a record exists. Duplicate qualifying keys are resolved toward the highest record index.

This predecessor-lookup interpretation is inferred intent. The established behavior is narrower: the function searches by unsigned key comparisons, then reads a payload using **the final upper search bound minus one**. It does not validate that this identifies a qualifying record.

An empty table or a query below every key causes a read two bytes before the table base, modulo the represented 32-bit address arithmetic. Wrapped midpoint arithmetic can also terminate the search with unequal bounds. In that case, the final payload address still depends on the upper bound, not the lower bound.

## Interface

### Inputs

The function receives three inputs in this order:

| Input | Represented width and interpretation | Role |
|---|---|---|
| Query | Unsigned 32-bit integer | Compared against each examined key |
| Table base | 32-bit address-bearing value, represented as signed | Base for all record-address calculations |
| Record count | Unsigned 32-bit integer | Initial exclusive upper search bound |

The table base is not checked for sign, alignment, accessibility, or range. Its signed representation does not establish that positive addresses are required.

The query retains its full 32-bit unsigned value during comparisons. A loaded key has an unsigned value between 0 and 65,535; it is not compared against a truncated 16-bit query. Consequently, every possible key satisfies the comparison when the query exceeds 65,535.

### Record layout

Logical record indices start at zero. Each record occupies four bytes:

| Offset within record | Width | Meaning |
|---|---:|---|
| 0 | 16 bits | Unsigned comparison key |
| 2 | 16 bits | Raw returned payload |

There is no observed record header, terminator, embedded count, or pointer indirection. The supplied count determines the initial search interval.

The function uses native two-byte loads rather than explicit bytewise decoding. The supplied evidence does not identify byte order or establish whether unaligned loads are supported.

### Output

On successful completion, the function returns the raw 16-bit value from its final payload read.

There is no separate success flag, selected index, not-found status, or error code. The payload’s signedness and application-level meaning are not established. A particular payload bit pattern cannot be identified as an error solely from this function.

### Preconditions for conventional lookup semantics

The predecessor interpretation requires:

- Keys in nondecreasing unsigned order.
- Stable enough contents to reason about one table throughout the search.
- A qualifying record, if the result is expected to belong to the supplied table.
- Readable memory at every examined key address and at the final payload address.
- Search arithmetic that does not wrap.
- Address calculations that identify the intended records without wrapping or aliasing.

These are caller-side conditions. The function does not enforce them.

## Behavior

### Search decisions

The initial lower bound is zero. The initial exclusive upper bound is the supplied record count. A key is examined only while the lower bound is strictly less than the upper bound.

The examined index is the unsigned 32-bit sum of the bounds, reduced modulo 2³², then shifted right by one bit. This is a wrapped-sum midpoint, not an overflow-avoiding midpoint.

Each examined key controls which part of the current search state is retained:

- A key less than or equal to the query advances the lower bound to one beyond the examined index and retains the upper bound.
- A key greater than the query replaces the upper bound with the examined index and retains the lower bound.

Equality therefore moves the search past the examined record. With ordered keys and nonwrapping arithmetic, this selects the last qualifying record, including the last record in a run of equal keys.

### Termination and payload selection

The search ends when the lower bound is no longer less than the upper bound.

Under nonwrapping search arithmetic, the ordinary interval invariant is preserved and the bounds coincide at termination. That equality is **not unconditional**. A wrapped midpoint can fall below the lower bound; if its key is nonqualifying, the upper bound can become smaller than the lower bound and terminate the search immediately.

Regardless of whether the bounds coincide, the final payload read uses:

- The **final upper bound**, reduced by one using unsigned 32-bit subtraction.
- Four-byte record spacing.
- A two-byte payload offset within the selected record position.

There is no final key check, qualifying-record check, or bounds check. The lower bound does not determine the final payload address when the two bounds differ.

### Observable read order

At the level represented by the supplied body, execution performs zero or more key reads followed by one payload read, if it reaches that read successfully.

Each key read is a 16-bit unsigned load. Its address depends on the bounds resulting from earlier comparisons. Payloads are not read during the search.

The final payload read is also 16 bits. Its loaded bits become the return value. No further table read is shown after it.

This describes logical access order, not processor speculation, bus transactions, atomicity, or a concurrency memory model.

### Ordinary boundaries

For a stable ordered table with nonwrapping search and address arithmetic:

- **Query below the first key:** the upper bound reaches zero, and the payload read is two bytes before the table base.
- **Query equal to a key:** the last record with that key supplies the payload.
- **Query between keys:** the last record with a smaller key supplies the payload.
- **Query at least as large as all keys:** the last record supplies the payload.
- **Query greater than 65,535:** every examined key qualifies, so a nonempty table supplies its last payload.

For a zero count, the search performs no key reads. The final upper bound is zero, so the function immediately attempts the pre-base payload read.

For a one-record table, a query at least as large as its key returns that record’s payload. A smaller query produces the pre-base read.

### Widths and wrapping

Search bounds, midpoint addition, and index updates use unsigned 32-bit arithmetic. The midpoint sum discards any carry beyond bit 31 before the right shift.

A record count no greater than 2,147,483,648 is sufficient to avoid midpoint-sum overflow while the ordinary interval invariant holds. This arithmetic condition does not guarantee a valid memory region or distinct record addresses.

Record offsets also use represented 32-bit arithmetic. Multiplication by four can wrap, and adding the table base can wrap. Logical indices separated by 1,073,741,824 therefore have the same four-byte displacement modulo 2³².

When the final upper bound is zero, subtracting one yields 4,294,967,295. Multiplication by four followed by addition of the payload offset produces displacement 4,294,967,294, equivalent to minus two modulo 2³². “Two bytes before the base” refers to this modular address relationship, not necessarily to valid preceding storage.

Wrapped midpoint arithmetic invalidates the usual shrinking-interval argument. It may move a bound in the wrong direction or terminate with reversed bounds. General termination and lookup correctness for such inputs are not established here.

### Unordered or changing records

The function does not check ordering.

With unordered keys, execution still follows the observed comparison results, but the payload need not belong to the greatest qualifying key, the last qualifying record, or any qualifying record.

No snapshot, lock, or other synchronization is shown. If keys change during execution, the search follows the values actually read. Its result need not correspond to a single stable arrangement of records.

## Side effects

The supplied body contains memory reads and no explicit memory writes.

It shows no allocation, deallocation, ownership transfer, reference-count update, or release operation. The table is not consumed or freed. Internal search-state changes do not update caller-provided storage.

Reads may themselves have external effects if the calculated addresses refer to device registers or other special memory. The evidence does not restrict the table base to ordinary RAM.

An inaccessible or unsupported address may prevent successful completion. There is no visible fault recovery or conversion of memory-access failure into an error return. The specific exception, signal, or platform response is not established.

There are no explicit calls through which additional ownership behavior or global-state preservation must be assumed.

## Dependencies

No explicit function calls appear in the supplied body. No additional opaque function IDs are required for callee dependencies.

Execution relies on the surrounding platform for:

- The represented 32-bit arithmetic and address formation.
- Unsigned 16-bit key loads and raw 16-bit payload loads.
- Interpretation of two-byte memory values.
- Alignment and memory-access rules.
- Delivery of the returned 16-bit value.

The function performs no observed byte swapping, bytewise reconstruction, memory allocation, or error reporting. Platform behavior outside these established operations remains unspecified.

## Acceptance examples

These are analytical acceptance cases derived from static evidence. They have not been executed. Unless stated otherwise, assume stable memory, native two-byte values matching the stated numbers, and valid nonwrapping addresses.

### Duplicate keys select the last duplicate

Assume four records:

| Index | Key | Payload |
|---:|---:|---:|
| 0 | 10 | 101 |
| 1 | 20 | 201 |
| 2 | 20 | 202 |
| 3 | 40 | 401 |

For query 20, key reads occur at indices 2 and 3, observing 20 and 40. Both bounds finish at 3. The final payload read is at index 2, returning 202.

Returning 201 would contradict the established equality behavior.

### A query between keys does not require an exact match

Using the same table, query 30 reads keys at indices 2 and 3 and returns 202. The absence of an exact key match does not produce a checked failure.

### Full-width query comparison

Using the same table, query 65,536 reads keys at indices 2 and 3. Both qualify, the final upper bound is 4, and the returned payload is 401.

This distinguishes the established comparison from truncating the query to 16 bits. Such truncation would produce zero, cause key reads at indices 2, 1, and 0, and lead to the pre-base payload read instead.

### Query below the first key

Using the same table, query 9 reads keys at indices 2, 1, and 0. The final upper bound is zero.

If the two bytes immediately before the table base are readable and represent raw value 777, the function returns 777. If that address is inaccessible, successful completion is not guaranteed. No checked not-found result is established.

### Empty table

With count zero, no key is read. The function still uses the table base to attempt the payload read at displacement minus two modulo 2³².

An empty count therefore does not make the table base irrelevant or permit assuming that no memory access occurs.

### Unordered keys follow the comparison path

Assume three records with keys 30, 10, and 20. For query 15, the function reads index 1, whose key qualifies, then index 2, whose key does not.

The final upper bound is 2, so the payload at index 1 is returned. This specifies the observed path without assigning ordered-search guarantees to unordered input.

### Wrapped midpoint terminates with unequal bounds

Assume count 0xE0000000 and that all accesses described below succeed under 32-bit modular address formation.

The first examined index is 0x70000000. If its key qualifies, the lower bound becomes 0x70000001 while the upper bound remains 0xE0000000.

Their mathematical sum is 0x150000001. Reduction to 32 bits yields 0x50000001, and shifting right produces the next examined index, 0x28000000.

If that second key is nonqualifying, the upper bound becomes 0x28000000. The lower bound remains 0x70000001, so the search ends with the upper bound strictly below the lower bound.

The final payload read uses record index **0x27FFFFFF**, derived from the final upper bound minus one. Its displacement from the table base is **0x9FFFFFFE** modulo 2³². It does not use the lower bound minus one.

## Uncertainties

The following remain **documented_only**:

- Whether callers guarantee ordered keys or a qualifying record.
- Whether readable pre-table storage is intentionally supplied as a sentinel.
- The application-level interpretation of payload bits.
- Count and address limits imposed by callers or the surrounding system.
- Byte order, alignment requirements, and the actual memory map.
- Concurrency guarantees and the stability of table contents.
- Fault behavior for invalid or unsupported reads.
- General termination behavior when midpoint arithmetic wraps.
- Any intended meaning for aliased record addresses or wrapped-search results.
- The precise machine-level return convention.

Predecessor lookup is inferred intent. Deliberate sentinel support and meaningful use of wrapped addresses are not established.

## Evidence

Established static evidence consists of three inputs; unsigned 32-bit search arithmetic; four-byte record spacing; unsigned 16-bit key loads; comparison against the full-width unsigned query; and a final raw 16-bit load based on the final upper bound minus one.

The loop condition permits termination whenever the lower bound is at least the upper bound. It does not require equality. The wrapped-midpoint example demonstrates analytically how unequal-bound termination follows from the represented operations.

The supplied body contains no explicit calls, writes, validation, ownership operations, or error reporting.

No runtime tests have been performed. All acceptance examples are conditional analytical expectations, not observed execution results.
