<!-- Trusted provenance: F-7fb2c125d7bd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7fb2c125d7bd

## Purpose

F-7fb2c125d7bd traverses a table of fixed-stride records while obtaining items from a byte-addressed input region through F-1e8578a40e00. Records select item types, optionally compare payload bytes or invoke an indirect callback, and determine subsequent records. A record can also redirect processing into an item’s payload while saving an outer continuation.

The function returns no value. Observable results consist of dependency and callback invocations, their effects, and the memory accesses performed during traversal. There is no explicit distinction at the interface between successful completion and rejection.

A parser or recognizer is the inferred purpose, not an established format specification. Concrete interpretation of the table depends on global multipliers whose values are unavailable.

## Interface

The function accepts six arguments:

| Position | Role | Represented width |
|---|---|---|
| 1 | Record-table base address | 32-bit address-bearing integer |
| 2 | Mode used by an incomplete-boundary restart | Signed 32-bit integer |
| 3 | Opaque context forwarded to callbacks | 32 bits |
| 4 | Initial input extent | Unsigned 32-bit integer |
| 5 | Exclusive upper bound for selected record transitions | Signed 32-bit integer |
| 6 | Initial input base address | 32-bit address-bearing integer |

The bound is not an initial table validation. In particular, the initial record index is zero without a preceding bound check.

### Record layout

Records have a 28-byte stride and seven four-byte fields:

| Offset | Observed role |
|---|---|
| 0 | Encoded item-type selector |
| 4 | Indirect callback address |
| 8 | Encoded expected payload length |
| 12 | Expected-byte address |
| 16 | Encoded nested-entry index |
| 20 | Encoded successor index |
| 24 | Flags interpreted using two different multipliers |

The table’s allocation size, alignment, byte order, and mutation permissions are not established.

### Working state

Initially, the current region is the supplied input region, the consumption position and record index are zero, and status is zero. Payload length, payload address, and item type are also initialized to zero. These item-output words are not reset before subsequent dependency calls.

The function retains the initial extent separately from the current extent. It also maintains an encoded continuation depth and storage for 16 four-word continuation entries.

Each saved entry has this layout:

| Entry offset | Stored value |
|---|---|
| 0 | Record index multiplied by its save multiplier |
| 4 | Current extent multiplied by its save multiplier |
| 8 | Current input base, stored directly |
| 12 | Post-dependency consumption position multiplied by its save multiplier |

The input base has no explicit encoding multiplication when saved and no explicit decoding multiplication when restored.

## Behavior

### Status and global values

At entry, the function reads and retains a global value used when assigning rejection status. Other globals supply multipliers for depth, saved fields, record indices, selectors, flags, lengths, and callback descriptor fields.

Zero status permits the relevant matching and transition operations. A nonzero status returns at the outer-loop status decision. Assigning the retained rejection value is not itself a return.

The retained value is not proven nonzero. Consequently, an apparent rejection may leave processing enabled, including in paths that subsequently access an unsuitable record index. Unless stated otherwise, rejection below means assignment of that retained value, not guaranteed termination.

Multipliers and record fields are read at their points of use rather than uniformly snapshotted at entry. Dependency or callback mutation can therefore affect later processing.

### Sentinel and boundary processing

Each outer iteration decodes the saved-context depth.

If the effective depth is zero and the record index is −1, the function returns immediately.

For any other occurrence of index −1, it checks whether either:

- The current extent, rounded upward to a multiple of four, equals the saved initial extent; or
- The consumption position exactly equals the current extent.

Rounding uses unsigned 32-bit addition of three followed by clearing the lowest two bits. Addition can wrap before the bits are cleared. Position greater than extent does not satisfy the second equality.

When either boundary equality holds, status is assigned the retained rejection value. A continuation is then restored for any nonzero effective depth, **including depth one**. The function computes an encoded decremented depth, decodes that new depth to select the saved entry, and restores:

- Extent through its restore multiplier.
- Input base directly.
- Consumption position through its restore multiplier.

The saved record identifier is transformed to locate its record, whose decoded successor becomes the current index. If that successor is signed-less-than the supplied bound, status becomes zero. There is no separate exclusion of −1 or other negative indices at this check.

This restoration depends on the multiplier relationships. The representation does not establish that decrementing and re-decoding depth selects the intended initialized entry.

If neither boundary equality holds, status is assigned the retained rejection value. Only mode exactly 1 changes the index to 1 and restores the status that existed before sentinel processing. No immediate upper-bound check validates that restart index.

If the index remains −1, record processing is skipped. Otherwise, processing continues in the same iteration. A status assigned during sentinel handling does not independently prevent a dependency call: if input remains, that call replaces status with its return value.

### Obtaining and selecting an item

When the index is not −1 and consumption is unsigned-less-than the current extent, the function captures the old consumption position and calls F-1e8578a40e00 with:

1. Current input base.
2. Current extent.
3. Address of the payload-length word.
4. Address of the payload-address word.
5. Address of the consumption-position word.
6. Address of the item-type word.

The return value replaces status. Only zero permits item selection.

The current record’s selector is multiplied by its decoding multiplier. An ordinary selector matches an equal item type. The special decoded selector −16,777,216 instead matches exactly these types: 12, 19, 20, 28, and 30.

On a mismatch, the function tests bit 3 of the raw flags multiplied by 15. If that bit is clear, it assigns rejection. Otherwise, it reads and decodes the successor and uses that as the current index. A successor of −1 or a successor signed-greater-than-or-equal-to the bound assigns rejection.

The same obtained item is tested against successor records; following a mismatch does not call the dependency again. Negative successor indices other than −1 have no lower-bound rejection.

If rejection is zero, mismatch handling can continue despite a missing transition flag or a rejected successor. In particular, the sentinel value −1 is not protected from a subsequent record access inside this matching loop when the rejection assignment leaves status zero.

### Literal payload comparison

After a type match, bit 0 of flags decoded with the separate flag multiplier selects literal comparison.

The function first compares actual payload length with the decoded expected length. Inequality assigns rejection, but does not suppress the byte-comparison loop.

Comparison proceeds over actual payload positions until the actual length is exhausted or an unequal-byte branch stops the loop. At each visited position:

- The expected-byte address is read from the current record.
- The expected byte is multiplied by −101 and narrowed to a character-sized value.
- The corresponding payload byte is read and compared with that transformed value.

If the first payload read compares unequal, the same payload location is read again and compared with the same already-computed expected character:

- A second unequal comparison stops the loop and assigns rejection.
- A second equal comparison stops the loop without assigning rejection for that position.

Thus, for stable memory, the first unequal byte ends comparison and rejects. For changing memory, the repeated read can instead end comparison without a new rejection assignment.

A length mismatch neither limits comparison to the expected length nor guarantees visits to every actual payload position. Reads beyond the decoded expected length can occur if earlier comparisons permit reaching them. Successful byte comparisons do not clear status assigned by the length check.

### Callback handling

If the first decoded flags read has bit 0 clear, a separate flags read and decoding tests bit 1. If bit 1 is set, the function invokes the address stored at record offset 4.

The arguments are the opaque third argument and the address of a five-word, 20-byte temporary descriptor:

| Descriptor offset | Value |
|---|---|
| 0 | Item type multiplied by the callback-type multiplier |
| 4 | Payload length multiplied by the callback-length multiplier |
| 8 | Payload address, directly |
| 12 | New consumption position minus old position, multiplied by the span multiplier |
| 16 | Current input base plus old consumption position |

The position difference is unsigned 32-bit subtraction; a decreased position therefore wraps. The final field denotes the start position supplied to the dependency for this item, not necessarily the payload address.

The callback return replaces status. Zero permits subsequent transitions; nonzero reaches the outer-loop return without taking the successful-item transition.

For stable flags, bit 0 takes precedence over bit 1: literal comparison suppresses the callback when both are set. If neither is set, there is no additional payload check or callback. Separate reads mean mutable flags must not be modeled as one immutable decoded value.

The descriptor is not read back to update item state after the callback.

### Successors and nested input

After item handling, transitions occur only while status is zero.

If the decoded nested-entry field is −1, the decoded successor becomes the current index. A successor signed-greater-than-or-equal-to the bound assigns rejection. Unlike mismatch handling, this path does not immediately reject −1; it can reach outer sentinel handling on the next iteration.

For a nested-entry field other than −1, the function first assigns rejection. It attempts to save a continuation only when the effective depth, interpreted as unsigned, is less than 16.

The save writes the transformed record index, transformed extent, direct input base, and transformed post-dependency position, in that order. Depth decoding is used again during save-address calculations; the initial capacity check does not establish safety under arbitrary changes to the decoding globals.

After the save, encoded depth is advanced. The nested-entry field is read and decoded again. If this index is signed-greater-than-or-equal-to the bound, rejection remains assigned. The saved entry and depth change are not rolled back.

Otherwise, the new input base becomes the payload address, the new extent becomes the payload length, consumption becomes zero, and status becomes zero.

There is no general lower-bound validation. A negative nested index other than −1 can pass the upper-bound check. Because the field is read twice, its second decoded value can also differ from the value that selected the nesting branch.

At effective depth 16 or greater, no continuation is saved and no nested-input redirection occurs through this path.

### Exhausted input and completion

When consumption is not unsigned-less-than extent, the dependency is not called.

The function first reads and decodes the current record’s successor. If it equals the current index, the index becomes −1.

Otherwise, while status is zero and the index is not −1, it follows successors only when bit 3 of raw flags multiplied by 15 is set. A missing flag or a successor signed-greater-than-or-equal-to the bound assigns rejection. A successor of −1 ends this loop through its loop condition rather than through mismatch-style rejection.

The initial self-successor probe occurs before the loop and is not repeated as a separate probe for every traversed record.

At the outer-loop decision, zero status begins another iteration and nonzero status returns. The empty-context/index-−1 condition also returns independently of status. There is no progress limit or cycle detection.

## Side effects

Direct effects include record, global, expected-byte, and payload-byte reads; writes to local state and temporary descriptors; calls to F-1e8578a40e00; and indirect callback calls.

No explicit allocation, deallocation, table write, or input-buffer write appears. This does not establish that the dependency or callbacks preserve those objects or any globals.

Input and payload addresses are forwarded without copying their contents. The callback descriptor has local lifetime; passing its address does not establish ownership transfer or permission to retain a usable descriptor after the call.

The function does not validate table capacity, expected-byte allocation lengths, payload address validity, callback addresses, or address arithmetic. Invalid accesses and invalid indirect calls are possible instead of reported errors. Valid execution requires every actually accessed address and callable target to be usable at the time of access.

## Dependencies

### F-1e8578a40e00

Established interface: input base and extent, followed by four addresses for payload length, payload address, consumption position, and item type. Its return determines whether item matching proceeds.

The following are unknown:

- Item encoding and validation rules.
- Which outputs are written on each return.
- Whether consumption advances, decreases, remains unchanged, or exceeds extent.
- Whether payload storage lies inside the input region.
- Allocation, release, ownership, and global-mutation effects.
- Nonzero status meanings beyond this caller’s control flow.

Outputs not written by a call retain their previous local contents.

### Record-selected indirect target

No fixed opaque function ID is supplied for dynamically selected callback targets. Their established caller-facing interface is the opaque context plus the temporary descriptor address, with a status result.

Their ownership rules, retained references, mutation effects, and broader error meanings remain unknown. Neither dependency is assumed to preserve globals or release memory.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

1. **Dependency refusal.** If F-1e8578a40e00 returns nonzero, that item receives no selector comparison, literal comparison, callback, or successful-item transition. The outer-loop status decision returns.

2. **Wildcard boundary.** A decoded selector of −16,777,216 matches item type 28 but not 29.

3. **Literal precedence.** With stable decoded flags containing bits 0 and 1, only literal comparison occurs; the record callback is not invoked.

4. **Length mismatch and early exit.** With actual length five and decoded expected length three, the function assigns rejection and still enters comparison. If earlier comparisons permit it, comparison can reach positions three and four, beyond the decoded expected length. A first-position mismatch can instead stop the loop immediately; five attempted positions are not guaranteed.

5. **Repeated payload read.** If a payload byte first differs from the transformed expected byte but equals it on the second read, comparison stops without assigning rejection for that byte. Any rejection already assigned for length mismatch remains.

6. **Callback positions.** If old consumption is 8, new consumption is 20, and current input base is B, the descriptor’s final field is B plus 8 under represented address arithmetic. Its span field is 12 multiplied by the span multiplier.

7. **Directly saved input base.** If a continuation save occurs with current base B, the entry’s base word receives B directly. If that entry is later selected for restoration and remains unchanged, the current base receives B directly, independently of extent and position restore multipliers.

8. **Single continuation restoration.** At index −1 with a boundary equality true and effective depth one, restoration is attempted. If depth transformations select entry zero, that entry supplies the outer region and saved-record successor. Depth one is not excluded.

9. **Capacity rejection.** At effective depth 16, nesting performs no continuation write or input redirection. Status receives the retained rejection value; termination requires that value to be nonzero.

10. **Mode-specific restart.** At index −1 with neither boundary equality true, mode 1 selects index 1 and restores the prior status. Mode 2 does not. This restart does not independently validate index 1 against the bound.

11. **Empty initial input.** With initial extent zero, the first iteration does not call the dependency. Record zero’s successor and exhausted-input rules determine subsequent traversal.

## Uncertainties

The following remain **documented_only**:

- Global multiplier values and whether the retained rejection value is nonzero.
- Whether save and restore transformations are inverses.
- Whether decoded depth always selects initialized, in-range continuation storage.
- External constraints on negative indices, table capacity, and record mutation.
- Character signedness and exact narrowing of transformed expected bytes.
- Signed multiplication, subtraction, and address-overflow behavior where not established by unsigned operands.
- Byte order, alignment, and target calling conventions.
- Dependency and callback ownership, mutation, and error contracts.
- Termination for cyclic transitions or non-progressing dependency outputs.

Unsigned extent rounding and consumption subtraction use the represented 32-bit width. Signed arithmetic must not be treated as unbounded or uniformly wrapping without further target evidence.

No guarantee is made that a payload points inside the current input, that actual length fits expected-byte storage, or that a checked record index remains valid for a later read.

## Evidence

Established behavior comes solely from the supplied static function representation: field offsets, 28-byte stride, arithmetic widths, branch conditions, call arguments, descriptor construction, continuation accesses, and return paths.

The continuation base is directly assigned on both save and restore; only the other three saved fields have explicit multiplier transformations. The boundary-restoration condition admits every nonzero effective depth, including one. The literal loop remains reachable after length mismatch but can stop at the first unequal-byte position.

The parser interpretation and semantic names assigned to modes and item types are inferred intent, not independently established format rules.

No runtime tests have been performed. Dependency implementations, callback implementations, concrete global values, and independent format documentation were not supplied.
