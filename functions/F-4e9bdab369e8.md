<!-- Trusted provenance: F-4e9bdab369e8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4e9bdab369e8

## Purpose

F-4e9bdab369e8 dispatches on an unsigned byte, with behavior additionally controlled by a signed threshold, a mutable 32-bit word, and a reversal flag. Selected operations include two-pass structured comparison, dependency calls, a literal result, a signed-positive test, and conditional word complementation.

The structured-comparison path resembles a prioritized comparison of collections. This is inferred intent, not an established general comparator contract. Other branches return literal or arithmetic bit patterns rather than conventional comparison results.

Several branches use the numerical tag as an address. Those accesses are part of the supplied static representation; this contract does not replace them with a presumed record layout.

## Interface

| Input | Established interpretation |
|---|---|
| Threshold | Signed 32-bit integer. Extended dispatch occurs only when it exceeds 15. |
| Word pointer | Address of an unsigned 32-bit word. Depending on the path, its contents are read, overwritten, or interpreted as an address. |
| Byte-region pointer | Address of the unsigned 8-bit dispatch tag. Some dependency calls also receive this pointer. |
| Reversal flag | Signed 32-bit value. Any nonzero value enables reversal, but only when the threshold exceeds 15. |

The represented return type is a pointer to an unsigned 32-bit word. Returned values include zero, one, literal tags, arithmetic bit patterns, complemented words, and dependency results. The return type alone establishes neither dereference permission nor ownership.

### Widths and layouts

Words and pointer-bearing fields occupy four bytes in the supplied representation. All offsets below are byte offsets.

For the structured path:

- A **holder** contains a collection handle at offset 0.
- A **record** contains a first word at offset 0, an associated-object address at offset 4, and a tie-break word at offset 8.
- An **associated object** contains a signed 32-bit value at offset 0, a signed 32-bit selector at offset 4, and another field at offset 8.
- In transformed-byte comparison, the associated object’s offset-0 field is used as a length and its offset-8 field as a byte-buffer address.

These are access layouts, not complete object definitions or allocation-size guarantees.

### Preconditions and errors

Every reached read, write, and dependency call requires suitable storage and arguments. Some branches require readable numerical low addresses, including address zero.

There are no explicit null checks, bounds checks, allocation checks, or recoverable memory-error results. Invalid accesses have no established successful-operation behavior; no particular fault, exception, or return value is promised.

The threshold is a dispatch boundary, not an established storage length. Address conversion, alignment, byte order, and address validity depend on the execution environment.

## Behavior

### Entry ordering

When the threshold is at most 15, execution goes directly to fallback. The reversal flag is ignored.

When the threshold exceeds 15:

1. An auxiliary unsigned 32-bit value starts at zero.
2. If reversal is enabled, the supplied word is read and its four bytes are reversed. The reversed word is written back.
3. The tag byte is read after that optional write.
4. Tags 0 through 7 select extended branches. Other values reach fallback.

With reversal enabled, the auxiliary value contains the reversed word with its least-significant byte cleared. For an original word of `0xAABBCCDD`, the stored word becomes `0xDDCCBBAA` and the auxiliary value becomes `0xDDCCBB00`.

Aliasing is observable: if the tag overlaps the word, extended dispatch uses the tag byte present after reversal, not necessarily the entry tag.

### Tags 0 and 4: locating collections

Both tags select the same structured-comparison path.

The function first reads the supplied word. Its value plus eight, using unsigned 32-bit arithmetic modulo \(2^{32}\), identifies the address of a pointer-bearing word. Reading that word obtains the **right holder**.

Next, the numerical tag itself is used as an address:

- Tag 0 reads an unsigned 32-bit word at absolute address 0.
- Tag 4 reads an unsigned 32-bit word at absolute address 4.

That word’s value plus eight, again modulo \(2^{32}\), identifies the address of a pointer-bearing word whose contents obtain the **left holder**.

The left holder is therefore not read directly from address 8 or 12 merely because the tag is 0 or 4.

The holder pointers are retained. Their offset-0 collection handles are read again for subsequent dependency calls; the handles are not entry snapshots.

### Initial count decision

F-10b760faca36 is called with the left holder’s current handle, then with the right holder’s current handle. Both results are treated as signed 32-bit integers.

If these initial results differ:

- The dependency is called again for the left holder’s current handle.
- It is then called again for the right holder’s current handle.
- The second left result minus the second right result is returned.

The initial difference is not returned. Repeated dependency results need not agree.

If the initial results match, another left-side call supplies the first traversal’s starting count.

### First traversal: descending structural comparison

The signed traversal count is decremented before its sign is tested. An ordinary nonnegative starting count visits indexes from count minus one down to zero. Zero produces no visits. Negative starting counts other than the signed minimum also produce no visits. Decrementing the signed minimum has unresolved overflow semantics.

For each visited index:

1. F-6bbddce1b930 retrieves the left record using the left holder’s current handle.
2. F-6bbddce1b930 retrieves the right record using the right holder’s current handle.
3. The records’ associated-object addresses and the associated objects’ signed selectors at offset 4 are read.
4. The left selector minus the right selector is calculated.

Both retrieval calls precede the selector-field reads. The second retrieval can therefore affect data subsequently read through the first retrieval’s result.

A nonzero selector difference returns immediately. Otherwise, the captured left selector selects the following comparison.

#### Selector 19

F-4d3d7560af30 receives the associated-object address read from the left record. Its result becomes the associated-object comparison result.

No corresponding right-object argument is supplied.

#### Selector 22 with dependency result 48

For selector 22, F-2eb743f270d1 receives the left record’s first word. Only a result of 48 selects transformed-byte comparison. Any other result uses ordinary associated-object comparison.

After result 48, the function captures the left and right associated-object pointers from their records. These can differ from the objects whose selectors were compared, because the dependency has already run.

The transformed comparison observes memory as follows:

- The right object’s signed length at offset 0 is read and retained.
- The left object’s signed length is read and compared with that retained right length.
- If unequal, the left length is read again. This reread minus the retained right length becomes the comparison result.
- If equal, an ascending signed 32-bit index starts at zero.
- Before each iteration, the left length is read again. An iteration occurs only when the index is less than this current signed length.
- Within an iteration, the left buffer address at offset 8 is read, the indexed byte is acquired, and F-1aef7372e8d8 is called with that byte.
- Only after that call returns are the right buffer address and corresponding right byte read. F-1aef7372e8d8 is then called with the right byte.
- Unequal transformation results produce their signed left-minus-right difference and end this comparison. Equal results permit the index to advance.

If the loop ends without a difference, this comparison yields zero.

The captured object pointers remain fixed during this comparison. Their fields and byte buffers do not: the left length is repeatedly read, both buffer addresses are read for each reached iteration, and bytes are acquired separately. A left transformation may change the right byte read during that same iteration.

The right length is not rechecked as a loop bound. An initially nonpositive common length causes no byte reads. Later left-length changes can shorten or extend traversal, subject to valid accesses.

The index increment cannot overflow signed 32-bit range. Every executed iteration requires the current index to be less than a signed 32-bit length, so that index is at most 2,147,483,646. Its increment can reach 2,147,483,647 but cannot exceed it. No later signed 32-bit length can admit an iteration at that maximum index. Dependency-driven length changes do not defeat this bound.

No particular transformation meaning, including case folding or encoding normalization, is established.

#### Ordinary associated-object comparison

This path applies to selectors other than 19 and to selector 22 when F-2eb743f270d1 does not return 48.

The associated objects’ signed offset-0 fields are subtracted, left minus right. A nonzero result returns immediately.

If those fields compare equal, F-7f14f1ab6851 receives the two associated-object offset-8 fields, with the left value as its first argument and the right value as its second. Its result becomes the associated-object comparison result.

These arguments specify positions, not a general guarantee about evaluation order within the call expression.

### Record-field tie-break

A nonzero associated-object comparison result returns immediately.

Otherwise, the records’ offset-8 fields are read and subtracted, left minus right. This subtraction is unsigned 32-bit arithmetic modulo \(2^{32}\). A nonzero resulting bit pattern is returned; zero continues the first traversal.

These record-field reads occur after any dependencies invoked for that element. They are live reads, not fields captured at record retrieval.

### Second traversal: descending payload comparison

Only after the entire first traversal yields equality does the second traversal begin.

A fresh F-10b760faca36 call with the left holder’s current handle supplies its starting count. The first traversal’s count is not reused.

The same decrement-before-sign-test rule applies. At each visited index:

1. F-6bbddce1b930 retrieves the left record.
2. F-6bbddce1b930 retrieves the right record.
3. F-9878699bd825 receives their first words, left as its first argument and right as its second.

The first nonzero dependency result is returned unchanged. If no indexes are visited, or all results are zero, the function returns zero.

No payload comparison occurs before the structural traversal completes.

### Tags 1 and 2: temporary-storage dispatch

For tag 1, a pointer read from a fixed global slot replaces the word pointer. A represented comparison between the established tag value 1 and address value 7 is false, so this branch continues to the same dependency call as tag 2.

Tag 2 retains the caller’s word pointer.

F-50507157c742 receives:

1. An address of temporary stack storage.
2. The current word pointer.
3. Zero.

Its result is returned unchanged. The temporary region’s extent, initial contents, and dependency-specific meaning are not established.

Any earlier reversal under tag 1 modified the original caller-supplied word, not the subsequently substituted pointer.

### Tags 3 and 5: four-argument dispatch

Tag 3 performs these reads in order:

1. A pointer-bearing word at absolute address 7 replaces the word pointer.
2. A pointer-bearing word at absolute address 11 replaces the byte-region pointer.
3. An unsigned word at absolute address 3 replaces the auxiliary value.

It then joins tag 5.

F-5502e89527d6 receives a value read from a fixed global slot, the current word pointer, the current byte-region pointer, and the current auxiliary word. Its result is returned unchanged.

For direct tag 5, the pointers remain the caller’s pointers. The auxiliary value is zero without reversal, or the partial reversal value described above with reversal. For tag 3, the last three arguments instead derive from the numerical-address reads.

### Tag 6: literal result

The function returns pointer-shaped bit pattern 6 without calling a dependency. Any enabled reversal has already occurred.

### Tag 7: signed-positive test

The function reads the 32-bit word at absolute address 11 and interprets it as signed. It returns one if that value is greater than zero, otherwise zero.

This test does not read through the caller’s word pointer. Earlier reversal may nevertheless have modified the caller’s word.

### Fallback

Fallback is reached directly when the threshold is at most 15, or after an unhandled extended tag.

It reads the tag byte afresh and compares its zero-extended value with four address-valued constants. Three have numerical values 5, 6, and 7. The fourth constant’s numerical value is unresolved.

If no constant matches, the tag itself is returned without a fallback-path word access.

If a constant matches, the supplied unsigned word is read, complemented over all 32 bits, written back, and returned as a pointer-shaped bit pattern.

Extended tags 5, 6, and 7 return before fallback. Their behavior therefore differs across the threshold boundary.

## Side effects

Direct writes are limited to optional reversal and fallback complementation of the caller-supplied word. An unhandled extended tag can reach fallback after reversal and, if it matches the unresolved constant, cause both writes.

Completed writes are not rolled back before early returns or later invalid accesses.

Dependencies may affect memory subsequently read, including globals, holders, records, objects, lengths, buffers, and bytes. Their mutation behavior is not supplied. Global-slot reads occur when their branches reach them, not at entry.

No explicit allocation, ownership transfer, or cleanup is established in this function. This does not imply that dependencies preserve ownership, avoid allocation or release, or decline to retain arguments. Returned pointer-shaped values carry no general release contract.

## Dependencies

| Opaque ID | Established call-site role |
|---|---|
| F-10b760faca36 | Supplies signed comparison values and traversal counts from current holder handles. |
| F-6bbddce1b930 | Retrieves an indexed record; left call precedes right call. |
| F-4d3d7560af30 | Supplies a result from the left associated object for selector 19. |
| F-2eb743f270d1 | Selects transformed-byte comparison when selector 22 produces result 48. |
| F-1aef7372e8d8 | Transforms an individual byte; the left call precedes right-byte acquisition. |
| F-7f14f1ab6851 | Supplies a result from two associated-object offset-8 fields. |
| F-9878699bd825 | Supplies the second-pass result from record first words. |
| F-50507157c742 | Receives temporary storage, a current word pointer, and zero. |
| F-5502e89527d6 | Receives a global value, two current pointers, and an auxiliary word. |

These are call-site contracts only. Dependency implementations, purity, ownership rules, global preservation, and error conventions are unknown. Separately sequenced operations have the ordering stated above; operands within a single expression do not generally establish an evaluation order.

## Acceptance examples

These are conditional static consequences, not observed executions. All reached accesses and dependency calls must satisfy their preconditions.

1. **Threshold boundary:** Threshold 15, tag 6, and word `0x12345678` cause fallback to write and return `0xEDCBA987`. The reversal flag has no effect.

2. **Extended literal:** Threshold 16, nonoverlapping tag 6, enabled reversal, and word `0x12345678` cause a write of `0x78563412`, followed by return of bit pattern 6.

3. **Auxiliary value:** Direct tag 5 with enabled reversal of `0xAABBCCDD` stores `0xDDCCBBAA` and supplies auxiliary argument `0xDDCCBB00`.

4. **Unhandled tag:** Threshold 15 and tag 9 return 9 without word access, provided the unresolved fallback constant is not 9. At threshold 16 with reversal enabled, this result requires the post-reversal tag to be 9; an entry tag of 9 alone is insufficient.

5. **Collection indirection:** For tag 4, a word value of `0x00001000` at absolute address 4 causes the left-holder pointer to be read at `0x00001008`, not address 12. The preceding right-holder access must also be valid.

6. **Repeated counts:** If the initial left and right counts differ, but the repeated counts are both 4, the function returns zero without retrieving records.

7. **Descending precedence:** With stable count 3, a nonzero structural result at index 2 prevents structural visits to indexes 1 and 0 and prevents the entire payload pass.

8. **Live length:** If transformed comparison initially observes both lengths as 2, then the first left transformation changes the left length to 1, equal transformation results allow the next condition to stop traversal. Index 1 is not read.

9. **Live right buffer:** If a left transformation replaces the captured right object’s buffer address, the right byte for that iteration is read through the replacement address.

10. **Bounded increment:** If a transformed iteration reaches index 2,147,483,646 while the current left length is 2,147,483,647 and both transformations agree, incrementing reaches 2,147,483,647 without overflow. The next condition cannot admit another iteration for any signed 32-bit length.

11. **Unsigned tie-break:** After earlier comparisons yield zero, left record field zero and right record field one produce returned bit pattern `0xFFFFFFFF`, without signed-subtraction overflow ambiguity.

12. **Signed-positive test:** Under extended tag 7, `0xFFFFFFFF` at absolute address 11 yields zero; `0x00000001` yields one.

13. **Empty collections:** Matching initial counts followed by starting counts of zero for both traversals cause no record retrievals and return zero.

## Uncertainties

- **documented_only — representation fidelity:** Numerical-address dereferences and pointer-shaped arithmetic results may reflect imperfect representation. No alternative interpretation is substituted.
- **documented_only — fallback constant:** The fourth fallback constant’s numerical identity is unresolved.
- **documented_only — signed arithmetic:** Signed differences of counts, selectors, lengths, ordinary object values, and transformation results can exceed signed 32-bit range. Traversal-count decrement at −2,147,483,648 is also unresolved. Source-level overflow behavior versus machine-level wrapping is not established. The transformed-loop index increment cannot overflow under its signed bound check; unsigned tie-break subtraction, reversal, and complementation are likewise excluded from this uncertainty.
- **documented_only — dependencies:** Mutation, stability, ownership, retention, global preservation, and failure behavior are unknown.
- **documented_only — temporary storage:** Its valid extent, initial contents, and permitted retention are unknown.
- **documented_only — environment:** Address formation and conversion, alignment, byte order, low-address accessibility, and fault handling are not supplied.
- **Inferred intent:** The structured branch resembles prioritized comparison. Symmetry, transitivity, normalized result signs, and successful termination under unrestricted dependency behavior are not established.

## Evidence

The established behavior derives from the supplied static function representation and opaque dependency mapping. This supports the dispatch boundary, write-before-tag-read ordering, collection indirection, repeated count calls, separate traversals, live transformed-byte reads, and unsigned record-field subtraction.

The transformed-loop increment bound follows from its signed condition: an executed iteration has an index strictly below a signed 32-bit length, hence at most 2,147,483,646 before increment.

No dependency implementations, caller contracts, memory image, or runtime traces were supplied. No runtime tests have been performed. The acceptance examples are conditional specifications derived from static evidence, not execution results.
