<!-- Trusted provenance: F-ef52dd4db7c1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ef52dd4db7c1

## Purpose

F-ef52dd4db7c1 compares a supplied 32-bit key against up to 23 four-byte values located through an external base expression. With a non-null output pointer, it reports the index of the first matching comparison. If all comparisons fail, it writes a distinguished failure value.

The external base expression is evaluated separately for each reached index. The body does not capture a single base before scanning. With a stable base and stable relevant contents, the behavior is a first-match search over 23 contiguous entries. Without those assumptions, the established behavior is an ordered sequence of comparisons at individually evaluated addresses, not a search of one immutable table.

A separate four-byte return value distinguishes successful matching from failure. A null output pointer skips the search and returns the same failure status as an unsuccessful search.

These findings derive from static evidence only. No runtime tests have been performed.

## Interface

### Search key

The first input is a 32-bit integer compared for equality with each reached four-byte entry.

All 32 bits participate. There is no masking, normalization, ordering comparison, or special rejection of negative values. A key with representation `0xFFFFFFFF` can match an entry with the same representation.

### Output pointer

The second input is a pointer to storage for one 32-bit unsigned result. Null is explicitly handled.

For normal completion with a non-null pointer, the destination must support the required four-byte write and satisfy the execution environment’s validity and alignment requirements. The function does not validate these properties.

**No prior output value is explicitly read through the output-pointer expression. This does not establish independence from the destination’s previous contents.** Those bytes can also be accessed through an alias during an entry read or evaluation of the external base expression. Their initial contents may therefore influence address selection, comparisons, the selected index, and the returned status.

For example, if the destination aliases entry 0, its initial four-byte value is also the value examined by the first comparison. Independence from the destination’s prior contents requires that those contents not affect any reached entry read or address selection.

### External base and entry layout

Entry addresses derive from external state rather than an explicit argument. For each reached index, the function evaluates the external base expression and reads a four-byte integer at that base plus four times the index.

| Property | Established value |
|---|---:|
| First eligible index | 0 |
| Last eligible index | 22 |
| Maximum comparisons | 23 |
| Entry width | 4 bytes |
| Entry spacing | 4 bytes |
| First relative offset | 0 bytes |
| Last relative offset | 88 bytes |

With a stable base, the eligible positions occupy a contiguous 92-byte region. Without base stability, that description does not constrain the aggregate addresses accessed: each comparison uses its own base evaluation.

Only addresses actually reached need to support entry reads for a particular invocation. A match at index 0 does not require later entry addresses to be readable. An unsuccessful search reaches all 23 entry reads.

### Results

| Normally completed condition | Returned four-byte pattern | Output write |
|---|---:|---|
| A comparison matches | `0x00000000` | First matching comparison’s index, 0–22 |
| All 23 comparisons fail | `0xFFFFFFCE` | `0xFFFFFFFF` |
| Output pointer is null | `0xFFFFFFCE` | None |

The failure status is 4,294,967,246 as an unsigned 32-bit value, or −50 under signed two’s-complement interpretation. The supplied representation establishes a four-byte result, but not the caller’s signed interpretation or a named error category.

The unsuccessful-search output is 4,294,967,295. It cannot collide with a successful index.

## Behavior

### Null output pointer

The output-pointer check precedes all scan-related base evaluations and entry reads.

When the pointer is null, the function returns `0xFFFFFFCE` without evaluating the external base expression, reading an entry, or writing output storage. Readable entry storage is not a requirement for this branch.

The return value alone does not distinguish a null output pointer from an unsuccessful search.

### Ordered comparisons

With a non-null output pointer, the function examines indices in ascending order, starting at 0 and continuing no farther than 22.

For each reached index:

- The external base expression is evaluated for that comparison.
- A four-byte entry is read at the resulting base plus the index’s four-byte offset.
- The entry is compared for equality with the key.

The first matching comparison ends the scan. No later index is reached.

Under stable-base and stable-content assumptions, this selects the lowest matching index in the table. More generally, it selects the earliest successful comparison among the addresses evaluated during this invocation.

An earlier failed comparison establishes only that the value read for that comparison differed from the key. It does not establish the contents of the corresponding position in a different base used later.

Per-iteration evaluation is established at the source level. The body does not establish whether concurrent changes are legal, visible, coherent, or synchronized, nor does it establish physical load counts.

### Successful completion

When comparison index \(k\) matches, the function writes \(k\) to the output destination and returns `0x00000000`.

The write follows the matching read and comparison. No later scan-related base evaluation or entry read occurs.

A successful index of zero is valid. The output value zero and return value zero are separate results communicated through different channels.

If the destination aliases entry storage or storage contributing to base selection, the final write may change that storage. It cannot change a comparison already completed, although the destination’s initial contents may have influenced that comparison through aliasing.

### Unsuccessful completion

If all 23 comparisons fail, the function writes `0xFFFFFFFF` to the destination and returns `0xFFFFFFCE`.

The previous destination value is replaced. It is not preserved as an earlier result.

There is no comparison at index 23 and no scan read at relative offset 92. With a stable base, a matching value immediately beyond the eligible region is irrelevant.

This replacement does not mean the previous destination value was necessarily irrelevant to the search: it may already have been read through an alias.

### Read and write ordering

The function performs no preliminary output initialization. Its explicit output write occurs only after either:

- A comparison has matched; or
- All 23 comparisons have failed.

Every normally completing invocation with a non-null output pointer contains exactly one explicit four-byte output write. The return follows that write.

For a match at index \(k\), the source-level scan reaches indices 0 through \(k\), inclusive. For an unsuccessful search, it reaches indices 0 through 22. Each reached index has a separate base-expression evaluation followed by its entry read and comparison.

“No preliminary output write” does not imply “no read of destination storage.” Entry access or base evaluation may read overlapping storage before the final write.

These statements concern the represented evaluation and access sequence. They do not establish atomicity, machine instruction selection, speculative accesses, or cross-thread memory ordering.

### Integer and address arithmetic

The scan index is a 32-bit unsigned value. It starts at zero, reaches at most 22 for a comparison, and reaches 23 when an unsuccessful scan terminates.

Multiplication by four produces entry offsets from 0 through 88. Neither the index progression nor these offsets overflow 32 bits.

The bounded offset does not prove that adding it to the external base yields a valid address. The base’s declared type and address width are not supplied. No guarantee about address wraparound, boundary crossing, or inaccessible memory follows from the small offset alone.

## Side effects

The only explicit memory write is the four-byte result write through a non-null output pointer. There is no direct assignment to the external base state.

The body contains no function calls, explicit allocation, release, logging, or synchronization. It does not visibly retain the output pointer or establish an ownership transfer.

The scan accesses read entry storage, but the interface does not reject output aliasing. Subject to the execution environment permitting the accesses:

- A destination overlapping an entry can supply bytes read during the scan.
- A destination overlapping storage used by the base expression can influence address selection through its initial contents.
- The final output write can modify overlapping entry or base-related storage.

Initial aliasing and final mutation are distinct effects. Initial contents can affect the chosen result. The final write occurs after the last comparison and cannot retroactively affect that choice. It can affect future invocations or other users of the same storage.

There is no visible recovery path for an invalid base-derived address or invalid non-null destination. The failure status is not established as a memory-fault report. Normal-return guarantees are conditional on all required accesses completing.

## Dependencies

No called-function dependencies appear in this body. No additional opaque function IDs are required.

The function depends on:

- External state supplying the base expression evaluated at each reached index.
- Readable four-byte storage at every entry address actually examined.
- Writable four-byte destination storage for a non-null output pointer.
- An execution environment supporting the represented integer operations and memory accesses.

The external state’s initialization, ownership, lifetime, and synchronization are not supplied. The same is true of entry storage and output storage.

No callee-preservation or memory-release contract can be inferred: there are no visible callees and no visible ownership protocol.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Unless stated otherwise, assume a stable base, stable entry contents until the final output write, and valid storage for every reached access.

### First-entry match with independent output storage

Suppose index 0 contains the key, and the destination does not overlap or otherwise affect entry storage or base selection.

The function evaluates the base for index 0, reads the entry at offset 0, writes output 0, and returns `0x00000000`. No later comparison is reached.

Under these independence assumptions, changing the destination’s previous contents does not change the result.

### Duplicate matches

Suppose indices 4 and 17 contain the key, while indices 0 through 3 do not.

Comparisons 0 through 4 are reached, each with its own base evaluation. The destination receives 4, and the return value is zero. Index 17 is not examined.

Returning 17 would violate first-match behavior under these assumptions.

### Last eligible entry

Suppose indices 0 through 21 differ from the key and index 22 matches.

All 23 comparisons are reached. The last entry read uses relative offset 88. The destination receives 22, and the function returns zero.

### No matching entry

Suppose all 23 eligible entries differ from the key.

The destination receives `0xFFFFFFFF`, replacing its prior contents, and the function returns `0xFFFFFFCE`. No comparison occurs at index 23.

A matching value at relative offset 92 does not change this result.

### Null destination

Suppose the output pointer is null, even though the external entry storage is unreadable.

The function returns `0xFFFFFFCE` without evaluating the external base expression, reading an entry, or writing output storage. This expectation concerns the represented operations, not unrelated failures outside them.

### All-one-bits key

Suppose the key is `0xFFFFFFFF`, index 6 contains the same representation, and no earlier comparison matches.

The destination receives 6, and the function returns zero. The key’s representation is not itself a failure condition. That representation serves as a sentinel only in the unsuccessful-search output channel.

### Destination aliases entry 0

Assume the destination denotes exactly the four bytes of entry 0, and the environment permits both the entry read and result write. Let the key be 7, and let entries 1 through 22 contain no 7.

If entry 0 initially contains 7, its initial contents cause the first comparison to match. The function writes 0 into those same bytes and returns zero.

If entry 0 instead initially contains 9, all comparisons fail. The function writes `0xFFFFFFFF` into those bytes and returns `0xFFFFFFCE`.

Thus, changing only the destination’s initial contents can change both the selected outcome and returned status. This occurs without any explicit read through the output-pointer expression.

### Different bases across comparisons

Conditionally assume an execution model in which successive base evaluations can yield different values without invalidating execution or the required accesses.

If comparison 0 fails using its evaluated base and comparison 1 succeeds using a different evaluated base, the destination receives 1 and the function returns zero.

The matching read is at the second evaluated base plus four bytes. The result does not assert that index 1 under the first base matches. This is an illustration of address selection, not a concurrency guarantee.

## Uncertainties

The following remain **documented_only**:

- **Domain meaning:** The meaning of the keys and entries is not established.
- **Initialization and ownership:** No construction, allocation, ownership-transfer, or storage-lifetime protocol is shown.
- **Base stability:** Per-index evaluation is established; equality of successive base values is not.
- **Actual alias relationships:** The body does not establish whether a particular caller’s destination overlaps entries or storage used in base evaluation. Independence from prior destination contents cannot be promised without excluding such influence.
- **Concurrency:** Legality, visibility, coherence, and ordering of concurrent mutations are unspecified.
- **Physical execution:** Source-level evaluation does not determine machine-load counts, atomicity, or speculative behavior.
- **Address representation:** Base type, pointer width, address-boundary behavior, and invalid-address outcomes are not established.
- **Byte order and alignment:** Access width is four bytes; environmental byte order and misalignment handling are unspecified.
- **Status meaning:** No named error meaning or caller interpretation of `0xFFFFFFCE` is established.
- **Caller behavior:** Destination initialization, alias restrictions, and reactions to results are not shown.

A fixed-table key-to-index lookup is a reasonable inference about intent under stable-state assumptions. It is not an unconditional snapshot guarantee.

## Evidence

The supplied static body establishes the initial null check, ascending unsigned index, 23-comparison bound, four-byte entry width and spacing, equality comparison, first-match exit, output writes, and returned bit patterns.

The external base expression occurs within each reached iteration’s entry access rather than being captured before scanning. This supports per-iteration source-level evaluation, not claims about concurrent visibility or physical load counts.

The output-pointer expression is used for the null check and final result write, not to fetch an earlier output value. Entry reads and base evaluation are not shown to be disjoint from destination storage. Consequently, lack of an explicit output-value read does not establish independence from its initial contents.

No calls, explicit allocation, explicit release, or direct external-base assignment appear. Output aliasing remains distinct from those absent operations.

No runtime tests, caller inspection, or external initialization analysis have been performed.
