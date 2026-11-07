<!-- Trusted provenance: F-51745eefdef4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-51745eefdef4

## Purpose

F-51745eefdef4 translates an unsigned integer through a table of input ranges. A matching range supplies an output base; the routine adds the input’s displacement within that range and returns the low 16 bits in an unsigned 32-bit result.

The routine examines up to two records sequentially, then may search a remaining candidate region by binary search. A nonzero translation from the sequential phase returns immediately. A zero translation instead permits binary search, potentially reconsidering the same record or selecting a different matching record.

Zero is therefore ambiguous: it can represent either no match or a successful translation to zero. There is no separate success indicator.

The apparent intent is efficient lookup in an ordered range table. Ordering and interval consistency are not validated. With stable storage, a correctly backed declared record array, and valid address arithmetic, unordered or overlapping records can affect lookup results but cannot by themselves cause an out-of-array search index.

## Interface

### Inputs

The routine accepts:

- **Object address:** A 32-bit address-bearing value identifying an object whose field at byte offset `0x5C` contains the table address.
- **Lookup value:** An unsigned 32-bit integer. It is not truncated before the eligibility comparisons.

The table-address field is read as a 32-bit value. Neither the object address nor the table address is checked for nullness or validity.

### Table layout

Offsets below are relative to the table address.

| Byte offset | Width | Meaning |
|---|---:|---|
| `0x04` | 16 bits, unsigned | Inclusive lower eligibility bound |
| `0x06` | 16 bits, unsigned | Inclusive upper eligibility bound |
| `0x0C` | 16 bits, unsigned | Record count |
| `0x10` | Eight bytes per record | Record array |

Each record has this layout:

| Record-relative offset | Width | Meaning |
|---|---:|---|
| `0x00` | 16 bits, unsigned | Input start |
| `0x02` | 16 bits, unsigned | Output base |
| `0x04` | 16 bits, unsigned | Range length |
| `0x06` | 16 bits | Not read by this routine |

Other header bytes are not read by this routine.

These are native integer accesses, not an independently specified serialized format. Byte order and alignment requirements remain environment-dependent.

### Output

The ordinary return value is an unsigned 32-bit integer in the range zero through 65,535. Its upper 16 bits are zero.

For a selected matching record, the result is:

**(output base + lookup value − input start) modulo 65,536.**

An ordinary lookup failure returns zero. A matching record may also return zero.

### Preconditions and ownership

The object and every accessed table location must be readable for the corresponding load width. Address calculations must identify the intended storage without invalid address wrapping.

For the stable-storage behavior described below, the object’s table-address field, relevant header fields, and record contents remain unchanged throughout the call. A correctly backed declared array provides eight bytes for each of the recorded number of entries.

These storage requirements are distinct from ordering requirements:

- Insufficient backing storage can cause an invalid read even when the selected index is within the declared count.
- Unordered or overlapping records can produce missed matches or path-dependent selection.
- With stable contents and valid address arithmetic, disorder or overlap alone does not make the search index leave the declared array.

No ownership transfer is expressed. The routine neither allocates nor releases storage. Lifetime and synchronization are external responsibilities.

## Behavior

### Header reads and eligibility

The routine first reads the table address from the object, then reads the table’s 16-bit record count.

If the count is zero, it returns zero without reading either eligibility bound or any record.

For a nonzero count:

1. The lower eligibility bound is read. A smaller lookup value returns zero immediately.
2. Otherwise, the upper eligibility bound is read. A larger lookup value returns zero immediately.
3. Only a value satisfying both inclusive bounds proceeds to record lookup.

A lower-bound rejection therefore avoids the upper-bound read. Either rejection avoids all record reads.

The bounds are independent constraints, not values derived from record coverage. If the lower bound exceeds the upper bound, no lookup value passes both checks.

Because the upper bound is a 16-bit unsigned value, every lookup argument greater than 65,535 is rejected before record access. Such arguments are not treated as their low 16 bits.

### Record membership and translation

A record’s exclusive endpoint is:

**(input start + range length) modulo 65,536.**

The record matches only when:

- input start ≤ lookup value; and
- lookup value < exclusive endpoint.

The comparisons are unsigned. The stored fields are widened for the arithmetic and comparisons.

A zero-length record cannot match. If the mathematical sum of start and length reaches or exceeds 65,536, the endpoint wraps below the start, so that record also cannot match. A wrapped endpoint does not define a circular interval.

Although the header’s upper bound is inclusive, record endpoints are exclusive. No record can match lookup value 65,535, because every computed endpoint is at most 65,535.

For a matching record, the output base is read and the translation is reduced modulo 65,536. Output overflow wraps; it does not clamp or generate an explicit error.

### Sequential prefix

The routine sequentially considers one record when the count is one, or two records when the count is at least two.

For each candidate:

- The input start is read first.
- The length is read only if the input start is no greater than the lookup value.
- The output base is read only if both membership comparisons succeed.

A nonmatch advances the sequential position by one.

A match stops the sequential phase without advancing that position:

- A nonzero translation returns immediately.
- A zero translation proceeds to the binary-search decision.

Thus, a zero-producing match in record zero prevents record one from being examined sequentially. It does not guarantee that record zero will be the record ultimately selected.

If all sequential candidates fail to match, the retained position is one for a one-record table and two for a larger table.

### Binary-search entry

If the retained position is at least the record count, the routine returns zero.

Otherwise, binary search starts with inclusive bounds:

- Lower index: the retained sequential position.
- Upper index: record count minus one.

Consequently:

- After two sequential nonmatches, binary search begins at index two.
- After a zero-producing match at index zero, binary search includes index zero.
- After a zero-producing match at index one, binary search includes index one.

A zero-producing match in a one-record table is therefore reconsidered by binary search.

### Binary-search selection

At each search iteration, an unsigned upper index below the lower index causes a zero return.

Otherwise, the candidate index is obtained by adding the bounds in unsigned 32-bit arithmetic, retaining the low 17 bits, and shifting right once.

For bounds inside the declared array, this is the ordinary floor midpoint. The largest possible count is 65,535, so the greatest valid index is 65,534; the sum of two valid indices is at most 131,068 and fits within 17 bits.

For the selected candidate:

1. Its input start is read.
2. If the start exceeds the lookup value, the upper index becomes candidate index minus one.
3. Otherwise, its length is read and the exclusive endpoint is calculated.
4. If the lookup value is below that endpoint, its output base is read and the translation returns.
5. Otherwise, the lower index becomes candidate index plus one.

A binary-search match returns immediately even when its translation is zero.

The search does not exhaustively inspect the candidate region. Without appropriate ordering and interval relationships, it can miss a matching record. With overlaps, the selected result depends on the sequential prefix and subsequent search path rather than a general first-match rule.

### Width-sensitive boundaries and search safety

The header fields and record fields used by the routine are unsigned 16-bit quantities. The lookup argument, search bounds, candidate index, and relevant arithmetic intermediates are unsigned 32-bit quantities.

The endpoint calculation clears bit 16 of the sum. Since two unsigned 16-bit operands sum to at most 131,070, that operation is exactly reduction modulo 65,536.

The sequential-position increment also clears bit 16, but the position never exceeds two during that phase, so the mask does not affect its value.

Unsigned candidate-minus-one arithmetic would underflow at candidate index zero. Under the stable-storage preconditions, however, that subtraction cannot occur at zero:

- If the binary-search lower bound is positive, every examined midpoint is positive.
- A lower bound of zero is possible only after record zero matched and produced zero in the sequential phase.
- If binary search later examines record zero while its contents are unchanged, its input start is still no greater than the lookup value. The branch that subtracts one is therefore not taken. Its unchanged length also establishes the same match again.

Accordingly, stable unordered records, overlaps, zero lengths, and wrapped endpoints do not cause search-index escape. Until termination, examined candidates remain within zero through count minus one. Each unsuccessful iteration strictly shrinks the candidate interval, so the search also terminates under these assumptions.

If record zero changes between its sequential match and its binary-search examination, this argument no longer applies. A newly larger start can enable subtraction at index zero, producing 4,294,967,295 as the unsigned upper bound. Subsequent midpoint masking is not an array-bounds defense. This is conditional on a storage-stability violation, not a consequence of ordering or overlap alone.

## Side effects

The supplied body performs memory reads and local arithmetic. It contains no explicit memory writes, allocation, deallocation, callbacks, or external calls.

Its source-level access ordering includes:

- The object’s table-address field precedes the count read.
- The count read precedes either eligibility-bound read.
- The lower-bound read precedes the upper-bound read when both occur.
- Eligibility checks precede record reads.
- Within a candidate examination, the start precedes any length read, and successful membership precedes any output-base read.
- A zero-producing sequential match can cause further reads of that record or other records.

Repeated examinations are separate accesses; the routine does not establish a snapshot.

No result is cached in the object or table. No storage is explicitly retained, transferred, or released.

These observations describe the supplied body’s access relationships. They do not specify exact machine scheduling, physical bus transactions, atomicity, or memory-mapped-device semantics.

There is no explicit synchronization or error-reporting channel. Invalid storage can cause an access failure rather than an ordinary zero return. Behavior under concurrent modification is not guaranteed.

## Dependencies

No external function dependency is present in the supplied body.

The routine depends on externally supplied object and table storage, native integer loads, and unsigned arithmetic. There are no callees whose preservation of global state or memory-ownership behavior must be assumed.

The identities and requirements of table producers and result consumers are unknown.

## Acceptance examples

These are conditional expectations from static inspection, not executed tests. Except where explicitly stated otherwise, storage is valid and stable, address arithmetic is valid, and the declared record array is fully backed.

### Empty table

With count zero, the result is zero. Neither eligibility bound nor any record is read.

### Eligibility rejection

With a nonempty table, lower bound 100, and upper bound 200:

- Lookup 99 returns zero without reading the upper bound.
- Lookup 201 returns zero after reading both bounds.
- Lookup 65,536 is rejected, not truncated to zero.

No record is read in these cases.

### Ordinary translation

With one record having start 100, output base 500, and length three, and header bounds admitting the requested values:

- Lookup 100 returns 500.
- Lookup 102 returns 502.
- Lookup 103 returns zero because the endpoint is exclusive.

### Output wrapping and reconsideration

With one record having start 100, output base 65,535, and length three:

- Lookup 100 returns 65,535.
- Lookup 101 translates to zero during the sequential phase, is reconsidered by binary search, and returns zero.
- Lookup 102 returns one.

The middle case is a successful zero translation, indistinguishable at the interface from failure.

### Wrapped input endpoint

A record with start 65,530 and length ten has endpoint four. It matches neither lookup 65,532 nor lookup two. Neither side of the numerical boundary is treated as part of a circular range.

### Search beyond the prefix

With increasing, disjoint intervals 10–11, 20–21, and 30–31, represented by length-two records with output bases 100, 200, and 300, lookup 31 returns 301 from the third record after the first two fail.

### Nonzero sequential precedence

If record zero matches and translates to seven, the routine returns seven without examining another record, even if later records also match.

### Zero changes selection precedence

With three records all having start ten and length one, and output bases zero, 100, and 200, lookup ten first produces zero from record zero.

Binary search then covers indices zero through two. Its first midpoint is index one, whose matching translation returns 100.

The overlap changes selection but does not cause an out-of-array index.

### Disorder can hide a match without escaping the array

With four length-one records having starts 10, 20, 100, and 30, respectively, let the fourth record’s output base be 300. Admit lookup 30 through the header bounds.

The first two records fail sequentially. Binary search begins over indices two through three, first examines index two, and lowers the upper bound to one because that record starts at 100. It then returns zero without examining the matching fourth record.

This is a missed match caused by disorder. Every examined index remains within the declared array.

### Conditional mutation hazard

Suppose a one-record table initially has start ten, length one, and output base zero, with lookup ten admitted by the header. The sequential phase matches and produces zero.

If an intervening change makes that record’s start eleven before binary search reads it, subtraction at candidate zero can underflow the upper bound. The next midpoint calculation can select index 65,535, outside this one-record array.

This example violates stable storage. It does not establish an escape path for unchanged malformed records.

## Uncertainties

### Inferred intent

The short sequential prefix appears intended as a fast path. Binary search suggests an intended ordering of ranges. Neither interpretation establishes producer guarantees or a validated format requirement.

### documented_only

The supplied material does not establish:

- Domain-specific meanings for inputs, results, or zero.
- Required byte order, alignment, or broader execution-environment memory rules.
- Producer guarantees concerning ordering, overlap, lengths, endpoints, or header coverage.
- The purposes of unused header bytes and unread record bytes.
- Caller-provided lifetime or synchronization guarantees.
- A defined failure mode for invalid addresses, insufficient backing storage, or address-calculation violations.
- Consistent results, termination, or search-index confinement when relevant contents change during execution.

These unknowns do not negate the established confinement and termination argument for stable, correctly backed storage with valid address arithmetic.

## Evidence

Established behavior follows from static inspection of F-51745eefdef4: access widths and offsets, conditional reads, unsigned comparisons, arithmetic masks, retained sequential positions, search-bound updates, return expressions, and the absence of explicit writes or external calls.

The stable-storage safety argument follows from the search-entry conditions: a zero lower bound requires a prior zero-producing match at record zero, whose unchanged start prevents the candidate-zero subtraction branch.

The proposed lookup purpose and expected ordering are inferred intent. Environmental and producer guarantees remain documented-only unknowns.

No runtime tests have been performed.
