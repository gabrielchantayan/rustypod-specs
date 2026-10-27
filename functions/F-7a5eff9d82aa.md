<!-- Trusted provenance: F-7a5eff9d82aa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7a5eff9d82aa

## Purpose

F-7a5eff9d82aa traverses a source range in three-word records, conditionally copies each record to the current destination position, and returns the destination cursor reached after traversal.

Each word is 32 bits, represented as four bytes. Each record therefore occupies 12 bytes. Traversal ends when the current source cursor equals the supplied source-end cursor; there is no separate record count.

The apparent intent is a forward record-copy operation that returns the end of the destination range. That interpretation does not establish bounds checking, overlap-independent copying, ownership transfer, or a supported null-destination mode.

This contract describes behavior established by the supplied static representation. Address-model assumptions and unresolved behavior are identified separately. No runtime tests have been performed.

## Interface

### Inputs

The function accepts three pointer-valued inputs:

| Input | Meaning |
|---|---|
| Source start | Initial cursor into the sequence of three-word source records. |
| Source end | Equality-based termination marker for the source cursor. |
| Destination start | Initial cursor for destination writes, subject to a per-iteration null check. |

The pointers address 32-bit words. Within a record, the word positions are at byte offsets 0, 4, and 8. Advancing either cursor by one record advances it by three word positions, corresponding to 12 bytes where that arithmetic is valid.

The word width does not establish the pointer width. The evidence does not specify native byte order, concrete address representation, or address-wrap behavior.

### Output

The result is the final destination cursor, with the same represented pointer type as the destination input.

For a traversal of \(N\) records with valid cursor arithmetic, the result is the initial destination advanced by \(3N\) word positions, or \(12N\) bytes. This formula describes the cumulative advances; the function does not visibly calculate a count or multiply a count by the record size.

The result is not a success indicator, an allocation result, or a number of copied records.

An empty source range returns the original destination pointer unchanged.

### Preconditions for ordinary copying

A conventional, nonempty copy requires:

- The source end must be reachable from the source start by repeated three-word advances.
- Every processed source record must permit the three specified 32-bit reads.
- Every destination record taking the copy branch must permit the three specified 32-bit writes.
- All cursor advances must be valid, including the advances after the final processed record.
- The accessed addresses must satisfy the execution environment’s alignment and memory-access requirements.
- Each destination cursor must be nonnull if every source record is expected to be copied.

The function does not check these conditions.

Disjoint source and destination storage avoids overlap interference. Disjointness is not checked, and some overlapping arrangements have predictable results under otherwise valid memory semantics.

## Behavior

### Entry and termination

The first operation controlling traversal is an equality comparison between the current source cursor and the source end.

If they are equal, the function returns the current destination cursor without reading or writing record storage and without advancing either cursor. This includes an empty range with a null destination.

If they differ, an iteration begins.

The termination condition is equality, not numerical ordering. A source end below the source start is not explicitly rejected. A source end lying between successive record positions is not interpreted as a partial final record. There is no visible truncation, range normalization, or fallback termination condition.

### Record layout and access order

For an iteration whose current destination cursor is nonnull, the represented memory-access sequence is:

1. Read the 32-bit source word at record offset 4.
2. Read the 32-bit source word at record offset 8.
3. Read the 32-bit source word at record offset 0.
4. Write the value from source offset 0 to destination offset 0.
5. Write the previously obtained value from source offset 4 to destination offset 4.
6. Write the previously obtained value from source offset 8 to destination offset 8.

All three source values for that iteration are therefore obtained before the first destination write. The values from source offsets 4 and 8 remain available for their later writes even if an earlier destination write overwrites the source locations from which they were obtained.

No arithmetic transformation or word-value conversion is represented. Each destination word receives the corresponding source word value.

This order is established at the level of the supplied static representation. It does not independently establish hardware transaction order, access atomicity, compiler barriers, or suitability for device registers.

### Destination-null branch

The destination null check occurs separately on each iteration.

If the current destination cursor is null, that iteration performs none of the three source-word reads and none of the three destination-word writes. The function does not read and discard the source record.

The null branch does not terminate traversal or produce an error result. Cursor advancement remains part of the represented control flow.

For a nonempty range, advancing a null pointer is not established as valid by the evidence. Consequently, the branch does not establish a portable counting mode or a supported way to traverse arbitrary source ranges without destination storage.

Under a separately defined address model in which advancing null produces a usable nonnull cursor, a later iteration would evaluate that new cursor and could take the copy branch. This is a conditional consequence of the control flow, not a guarantee about the actual execution environment.

### Cursor advancement

After either the copy branch or the null-destination branch:

1. The destination cursor advances by three word positions.
2. The source cursor advances by three word positions.
3. The next source-equality comparison determines whether traversal continues.

The destination advance therefore precedes the source advance. Both advances occur after the final processed record, before the equality comparison that ends traversal.

For an ordinary valid traversal of \(N\) records in which every destination cursor is nonnull:

- There are \(3N\) represented source-word reads.
- There are \(3N\) represented destination-word writes.
- Records are processed in forward order.
- The writes for one record finish before the reads for the following record begin.
- The returned destination cursor follows the last written record.

These counts describe represented accesses, not necessarily separate processor transactions.

### Overlapping storage

The operation does not take a snapshot of the entire source range.

Within one iteration, all three source words are obtained before any destination write. This protects that iteration’s selected values from its own writes, provided the accesses and pointer arithmetic are otherwise valid.

Across iterations, earlier writes can change source words that have not yet been read. A later iteration then uses the changed values. The result can therefore differ from a copy through a temporary buffer covering the entire range.

When source and destination start at the same position, each record’s values are read and written back to their existing positions. Under stable ordinary-memory assumptions, the final values are unchanged. The represented reads and writes still occur.

### Invalid input and interrupted execution

The body contains no explicit error handling for:

- An unreachable source-end cursor.
- Insufficient source or destination storage.
- Misaligned or inaccessible addresses.
- Invalid cursor arithmetic.
- Read or write faults.

No defined error result is established for these conditions. The function does not visibly detect or repair them. Any trap, fault delivery, or other response depends on the execution environment.

An unreachable end does not establish safe infinite traversal: an invalid access or invalid arithmetic operation may occur first.

No rollback is visible. If execution is interrupted after some destination writes have completed, those writes may remain observable. Record-wide atomic completion is not established.

## Side effects

The direct memory effects are the ordered destination writes described above. Aliasing can make these writes also modify storage viewed through the source cursor or through other references.

The body establishes no visible allocation, deallocation, retention, release, or other ownership-management operation. Ownership and storage-management responsibilities for either region are not established by this function; copying the words does not itself establish ownership transfer.

A word may encode an address or another resource-related value, but the function does not visibly interpret it that way. The copy therefore establishes no acquisition, release, reference-count adjustment, or lifetime extension for any resource such a value might designate.

No direct global-state access or called-function dependency appears in the supplied body. This does not imply that arbitrary memory reachable through the inputs is separate from global storage: a destination write can affect such storage if the supplied pointer addresses it.

No synchronization, locking, transaction boundary, or atomic record publication is visible. Concurrent access, volatile storage, memory-mapped devices, and persistent-memory guarantees are not established.

## Dependencies

No called-function dependencies appear in the supplied representation.

The operation depends on the surrounding execution environment for:

- Pointer equality.
- Three-word pointer advancement.
- Null-pointer interpretation.
- 32-bit reads and writes.
- Alignment, address validity, and fault behavior.

The function does not define those environmental rules. There are no unknown callees whose global-state preservation or storage-management behavior must be assumed.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless stated otherwise, assume valid cursor arithmetic, accessible and suitably aligned ordinary memory, stable values except for the described writes, and no concurrent access.

### Empty source range

Given equal source-start and source-end pointers:

- No record words are read or written.
- Neither cursor advances.
- The original destination pointer is returned.
- A null destination is returned unchanged without null-pointer advancement.

No record storage must be accessed to recognize this case.

### One disjoint record

Given one source record containing \((A,B,C)\), a source end exactly one record after the start, and separate writable destination storage:

- The reads occur in the order \(B\), \(C\), then \(A\).
- The destination writes occur in the order \(A\), \(B\), then \(C\).
- All three reads precede the first destination write.
- The destination contains \((A,B,C)\).
- The result is the initial destination advanced by 12 bytes.

No conversion of the word values is represented.

### Two disjoint records

Given source records \((A,B,C)\) and \((D,E,F)\), with the source end exactly 24 bytes after the start:

- The destination records become \((A,B,C)\) and \((D,E,F)\).
- All first-record writes precede the second-record reads.
- No third source record is read.
- The result is 24 bytes beyond the initial destination.

The cursor advances after the second record still require valid pointer arithmetic.

### Identical source and destination

Given a valid two-record source range with the destination equal to the source start:

- Both records undergo the specified reads and writes.
- Their final values remain unchanged under the stable-memory assumptions.
- The returned destination cursor equals the source end.

Final-value equality alone does not describe the represented access behavior; the accesses are still part of the static contract.

### Destination one record ahead

Given three contiguous records, initially beginning with \((A,B,C)\) and \((D,E,F)\), copy the first two records to a destination beginning at the second record:

- The first iteration replaces the second record with \((A,B,C)\).
- The second iteration reads that modified record.
- The third record receives \((A,B,C)\), not the original \((D,E,F)\).
- The result points immediately after the third record.

This distinguishes forward record copying from whole-range snapshot copying.

### One-word overlap within a single record

Given one source record \((A,B,C)\), sufficient accessible storage, and a destination starting one word after the source start:

- The destination’s three word positions receive \(A\), \(B\), and \(C\).
- The first write overwrites the original location of \(B\).
- The later write of \(B\) uses the value read before that overwrite.
- The later write of \(C\) likewise uses its previously read value.

This criterion covers one iteration only. It does not establish protection for source records read by later iterations.

### Nonempty source range with null destination

Given unequal source pointers and an initially null destination:

- The first iteration selects the branch with no record reads or writes.
- The represented next operation is destination advancement, followed by source advancement.
- Defined completion and a portable return value are not established.

A numeric result must not be required without a separately specified address-arithmetic model.

### End marker between record positions

Given a source end four bytes after the source start:

- The initial equality comparison does not identify an empty range.
- The function does not copy only one word.
- The first iteration selects its normal three-word path if the destination is nonnull.
- No successful bounded traversal is established merely by supplying that end marker.

This example demonstrates the lack of partial-record handling, not a guarantee about behavior after invalid traversal.

## Uncertainties

### Established static evidence

The supplied representation establishes:

- Three 32-bit words per record.
- A three-word stride for both cursors.
- Equality-based termination checked before each iteration.
- A per-iteration destination-null branch.
- The specified source-read and destination-write order.
- Destination advancement before source advancement.
- Return of the final destination cursor.
- No visible calls, allocation, deallocation, ownership-management operation, validation branch, or error result.

### Inferred intent

Forward copying of fixed-size records is the apparent purpose. Returning the final destination cursor is consistent with use as an end-of-output helper.

These inferences add no bounds validation, ownership rules, overlap independence, or supported nonempty null-destination workflow.

### documented_only

The following remain unresolved:

- Pointer width and concrete address representation.
- Arithmetic outside valid pointer ranges, including advancement from null.
- Native byte order and exact alignment requirements.
- The intended record type and meaning of its three words.
- Ownership, lifetime, and storage-management responsibilities for either region.
- Intended caller constraints on overlap.
- Whether nonempty null-destination calls occur in intended use.
- Environmental responses to invalid accesses or invalid arithmetic.
- Hardware access ordering, atomicity, and special-memory semantics.
- Guarantees under concurrent mutation or interruption.

These unknowns must not be replaced with assumed platform conventions or inferred ownership policies.

## Evidence

This contract is derived solely from the supplied static function representation: three-word records, an equality-controlled traversal, a destination-null branch around record accesses, ordered word reads and writes, unconditional cursor advancement in destination-then-source order, and return of the final destination cursor.

The absence of visible ownership operations establishes only that no such operations appear in this body. It does not establish who owns, manages, or must release either storage region.

No runtime tests have been performed. The acceptance examples are conditional deductions from the static representation, not observed executions.
