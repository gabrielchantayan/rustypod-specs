<!-- Trusted provenance: F-f8b2008aeab2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f8b2008aeab2

## Purpose

F-f8b2008aeab2 allocates a 40-byte-stride storage region and updates a caller-supplied descriptor through one of three paths:

- Initialize a pointer table and two four-word records.
- Append a storage-region address to an existing table with space at its right edge.
- Replace a right-edge-exhausted table, copy its active entries, append the new storage-region address, and rebuild both records.

**Inferred intent:** the descriptor resembles a segmented sequence with two endpoint records and a table of storage-region pointers. This interpretation explains the arrangement, but element construction, logical length, and the meaning of an endpoint are not established here.

**Established behavior:** this function allocates storage, manipulates addresses and descriptor words, and delegates record construction. It does not visibly copy or initialize individual 40-byte elements.

## Interface

### Inputs

The visible interface accepts four inputs:

1. A pointer to a writable descriptor containing at least eleven consecutive 32-bit words.
2. Three additional 32-bit values.

The descriptor pointer is also placed into the first word of a temporary four-word record. The other three arguments initialize that record’s remaining words.

The additional arguments have no visible interpretation beyond their initial placement in this temporary record. They are not otherwise directly used in size calculations or branch selection. A dependency may inspect or overwrite them through the temporary record.

### Descriptor layout

Offsets below are byte offsets from the descriptor base.

| Offset | Width | Established role |
|---|---:|---|
| 0 | 32 bits | First word of the first four-word record; supplied as an address argument during table replacement |
| 4 | 32 bits | Second word of the first record |
| 8 | 32 bits | Third word of the first record |
| 12 | 32 bits | Fourth word of the first record; used as the first active table-slot address during replacement |
| 16 | 32 bits | First word of the second four-word record |
| 20 | 32 bits | Second word of the second record |
| 24 | 32 bits | Third word of the second record |
| 28 | 32 bits | Fourth word of the second record; used as the current rightmost table-slot address |
| 32 | 32 bits | No direct access shown |
| 36 | 32 bits | Pointer-table base address |
| 40 | 32 bits | Pointer-table capacity or count field |

The interpretation of the two records as endpoints is inferred. Their four-word extent and the address calculations involving their fourth words are established.

### Result

The visible result is 64 bits:

- Bits 0–31 contain the temporary record’s final first word.
- Bits 32–63 contain its final second word.

This is a bitwise concatenation, not an established scalar status, pointer, or logical count. The temporary record initially contains the four inputs, but record-construction calls may change it before return.

### Widths and preconditions

The visible representation uses 32-bit words and 32-bit address arithmetic. Pointer-table entries occupy four bytes. Storage-region positioning uses a 40-byte stride.

For ordinary, non-wrapping operation:

- The descriptor and every accessed table slot must be readable or writable as required.
- Allocations must yield usable addresses for the requested ranges.
- Table-slot addresses must be four-byte aligned and mutually consistent.
- Allocation-size arithmetic, address arithmetic, and count transformations must remain within their valid ranges.
- Dependencies must return normally for the described path to complete.

There are no visible checks enforcing these preconditions. There is no visible null check, allocation-failure branch, overflow check, or descriptor-validation failure result.

## Behavior

### Common preparation

Before selecting a descriptor-update path, the function:

1. Initializes a private four-word temporary record from the descriptor pointer and the three additional arguments.
2. Calls F-5681dacec8f5.
3. Multiplies that call’s 32-bit result by 40 and passes the resulting size to F-871e64ddc6de, with a second argument of zero.
4. Retains the returned value as the new storage-region base.
5. Calls F-c6ba54ae5191 with the descriptor pointer.

Thus, the storage-region allocation precedes the branch-selection call on every path. A nonzero result from F-c6ba54ae5191 selects initialization. Zero selects one of the existing-table paths.

Repeated calls to F-5681dacec8f5 are independent observations. Nothing here establishes that they return equal values or that any dependency preserves the descriptor or other shared state.

### Initialization path

For a nonzero branch-selection result, the following actions occur in order:

1. Call F-5681dacec8f5 again.
2. Store its result at descriptor offset 40.
3. Allocate pointer-table storage through F-871e64ddc6de, requesting that result shifted left by two bits, with second argument zero.
4. Store the allocation result at descriptor offset 36.
5. Read the capacity field and place the new storage-region base into the table slot indexed by the unsigned capacity divided by two, rounded down.
6. Call F-5681dacec8f5 a third time.
7. Call F-9e3528e6ab0e with:
   - The address of the temporary four-word record.
   - The new storage-region base plus 40 times the unsigned half of this latest result.
   - The table-slot address computed from the descriptor’s then-current table base and unsigned half-capacity.
8. Copy the temporary record into descriptor offsets 0, 4, 8, and 12, in that order.
9. Copy those four descriptor words into offsets 16, 20, 24, and 28, in that order.
10. Return the temporary record’s first two words as the 64-bit result.

The second copy reads the first record rather than independently copying the temporary record.

The counts used for region allocation, table allocation, and the position within the region come from three separate calls. Equality among them is not established.

### Existing table with right-edge space

After a zero branch-selection result, the function reads the slot address at offset 28. It compares that address with:

**table base + four times table capacity − four**

using the descriptor fields at offsets 36 and 40.

If the addresses differ, it follows the append-without-replacement path. The test is equality only: it does not prove that the current slot is inside the table or before its last slot.

The actions are:

1. Write the old slot address plus four to descriptor offset 28.
2. Write the new storage-region base into that next slot.
3. Call F-9e3528e6ab0e with:
   - The temporary record address.
   - The new storage-region base, without a midpoint offset.
   - The slot address then read from descriptor offset 28.
4. Copy the temporary record into descriptor offsets 16, 20, 24, and 28, in order.
5. Return the temporary record’s first two words.

There is no direct write to the first record, table base, or capacity on this path. Dependencies may nevertheless affect accessible state; preservation is not established.

The temporary advance of offset 28 occurs before the new slot is written and before record construction. The final value at offset 28 comes from the temporary record and is not necessarily the same advanced address.

### Right-edge-exhausted table

If the current slot equals the calculated final slot, the function replaces the table.

For the normal aligned, non-wrapping case, define:

- **D** as the signed byte difference between the current rightmost slot and the slot address at descriptor offset 12, arithmetically shifted right by two.
- **N** as D + 1, subsequently used as an unsigned 32-bit value.
- **B** as the returned replacement-table base.
- **O** as the 32-bit value 2 × N with its lowest two bits cleared.

For valid ordered slot addresses, D counts slot intervals and N counts active entries.

The sequence is:

1. Compute D and N.
2. Allocate 8 × N bytes through F-871e64ddc6de, with second argument zero.
3. Read the descriptor’s current rightmost slot again.
4. Compute O.
5. Copy four-byte entries from the slot address at descriptor offset 12 through the reread rightmost slot, inclusive, into consecutive slots beginning at B + O.
6. Call F-ff1bdc376dcd with the descriptor’s current table base, its current capacity, and zero.
7. Write B to descriptor offset 36.
8. Store the new storage-region base at an address calculated from B, the earlier D, and N.
9. Rebuild the first record through F-9e3528e6ab0e.
10. Rebuild the second record through F-9e3528e6ab0e.
11. Store 2 × N into descriptor offset 40.
12. Return the temporary record’s final first two words.

For ordinary positive counts below the high-bit boundary, the new-entry address in step 8 is:

**B + 4 × (D + floor(N / 2) + 1)**

The exact calculation clears N’s highest bit before taking that particular half-count. It therefore must not be simplified to unrestricted division by two for all 32-bit inputs.

The first record-construction call receives:

- The temporary record address.
- The descriptor word at offset 0, read after the old-table dependency call and replacement-base write.
- The descriptor’s current table base plus O.

Its resulting temporary record is copied to descriptor offsets 0–12.

The second record-construction call receives:

- The same temporary record address.
- The new storage-region base.
- The descriptor’s current table base plus O + 4 × D + 4.

Its resulting temporary record is copied to descriptor offsets 16–28. Only then is the new capacity written.

Under ordinary count arithmetic, the replacement table has capacity 2 × N, the copied entries begin at index floor(N / 2), and the new entry follows the copied range. This leaves some capacity before the copied entries rather than placing them at index zero.

### Copy traversal and arithmetic limits

The replacement copy is a forward sequence of four-byte reads and writes. It terminates when the source address equals the reread rightmost-slot address plus four. It is not bounded by an independently checked entry count.

Consequently:

- Misalignment or an unreachable endpoint can prevent normal termination.
- An intervening descriptor change can make the copy range differ from the range used to calculate allocation size.
- Overlapping source and destination ranges do not receive special overlap handling.
- Overflow can affect allocation sizes, final-slot comparisons, copy endpoints, and destination addresses.

No visible recovery behavior addresses these cases. The contract does not assert a portable outcome for invalid addresses or arithmetic outside the intended range.

## Side effects

### Memory writes

Depending on the path, direct writes affect:

- The newly allocated pointer table.
- A next slot in the existing table.
- All eight words of the descriptor’s two records, or only the second record.
- The table base and capacity fields during initialization or replacement.
- The private temporary record through F-9e3528e6ab0e.

There is no direct access to descriptor offset 32.

The function is not transactional. Allocation, table writes, descriptor writes, and dependency calls occur incrementally. No rollback is visible if a later operation fails or does not return normally.

### Ownership and lifetime

The new storage-region allocation is retained by writing its address into a table. No direct cleanup of that allocation appears on any path.

On replacement, F-ff1bdc376dcd is called with the old table’s base and capacity before the descriptor is updated to the replacement base. This suggests old-table disposal, but its ownership and release behavior are unknown.

No corresponding call is made for the storage regions referenced by copied entries. Their addresses are copied as four-byte values.

### Visibility during calls

Dependencies can run while descriptor state is only partly updated. Notably:

- Initialization publishes the table base and capacity before constructing the records.
- In-place append advances the second record’s slot field before constructing its replacement record.
- Replacement publishes the new table base before rebuilding the records, but leaves the old capacity until both record updates finish.

No synchronization or exclusion mechanism is visible.

## Dependencies

| Opaque ID | Observed interaction | Not established |
|---|---|---|
| F-5681dacec8f5 | Returns a 32-bit value used in allocation sizing or midpoint positioning | Meaning, stability, range, side effects |
| F-871e64ddc6de | Receives a calculated byte-size value and zero; result is used as a storage address | Allocation semantics, initialization, alignment guarantees, failure convention, ownership |
| F-c6ba54ae5191 | Receives the descriptor; zero versus nonzero selects the path | Predicate meaning, validation guarantees, mutation behavior |
| F-9e3528e6ab0e | Receives a writable temporary-record address and two address-valued arguments | Exact record layout semantics, which words it writes, other side effects |
| F-ff1bdc376dcd | Receives the current table base, current capacity, and zero during replacement | Whether it releases memory, interprets capacity as an entry count, or alters accessible state |

No dependency is assumed to preserve globals, preserve descriptor fields, or acquire or relinquish ownership beyond the visible interactions.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. They assume valid aligned addresses, successful non-overlapping allocations, normal dependency returns, and no unmentioned dependency mutations.

### Initialization with three different sizing results

Suppose the three sizing calls return 8, 6, and 4, respectively, and initialization is selected.

Expected behavior:

- Request 320 bytes for the storage region.
- Publish capacity 6 and request 24 bytes for its pointer table.
- Store the storage-region base in table slot 3.
- Supply the storage-region base plus 80 bytes to record construction.
- Copy the constructed temporary record to the first descriptor record, then copy that record to the second.
- Return the temporary record’s first two words.

This example intentionally does not assume equal sizing results.

### Append without replacement

Suppose the table has capacity 8 and the second record’s slot address is table index 5.

Expected behavior:

- The comparison with index 7 is unequal.
- Advance the stored slot address to index 6.
- Write the new storage-region base at index 6.
- Construct the second record using the unoffset storage-region base and the slot address.
- Leave the first record directly unchanged.

### Replacement with three active entries

Suppose capacity is 8 and active slots extend from index 5 through index 7.

Then D is 2 and N is 3.

Expected behavior:

- Request 24 bytes for a six-entry replacement table.
- Copy the three active entries into replacement indices 1, 2, and 3, in order.
- Call F-ff1bdc376dcd with the old table base, capacity 8, and zero.
- Publish the replacement base.
- Store the new storage-region base at replacement index 4.
- Rebuild the first record using replacement slot 1.
- Rebuild the second record using replacement slot 4.
- Publish capacity 6 last.

The new capacity is based on the active range, not twice the old capacity.

### Return-word ordering

If the final temporary record’s first two words are respectively 0x11223344 and 0x55667788, the returned 64-bit bit pattern is 0x5566778811223344.

### Invalid allocation result

If F-871e64ddc6de returns an unusable address, no visible branch converts that outcome into a defined error result. Subsequent behavior depends on the address use and dependency behavior. Graceful failure is not an acceptance requirement established by this evidence.

## Uncertainties

- **documented_only — Semantic type:** a segmented-sequence interpretation is inferred; no element type or abstract container API is established.
- **documented_only — Dependency contracts:** allocation, disposal, predicate, sizing, and record-construction semantics are unavailable.
- **documented_only — Error behavior:** no explicit status protocol, exception contract, or allocation-failure handling is shown.
- **documented_only — Signedness boundaries:** the visible operations distinguish unsigned midpoint calculations from a signed shifted address difference. Source-language overflow rules are not established.
- **documented_only — Aliasing and concurrency:** no guarantees establish disjoint allocation ranges, stable descriptor fields across calls, or safe concurrent access.
- **documented_only — Lifetime:** the eventual owner and disposal rules for storage regions and the final pointer table are unknown.
- **documented_only — Result meaning:** the result’s bit composition is established; its intended public interpretation is not.

## Evidence

The supplied static representation establishes the branch conditions, dependency-call order, descriptor offsets, four-byte table entries, 40-byte positioning stride, copy direction, write order, and 64-bit result composition described above.

Semantic names such as “record,” “table,” and “storage region” summarize observed uses. Endpoint and segmented-sequence meanings remain inferred intent.

No implementation execution, runtime tests, allocation experiments, or dependency inspection has been performed. The acceptance examples are consequences of the visible operations under their stated assumptions, not observed runtime results.
