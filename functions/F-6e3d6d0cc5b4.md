<!-- Trusted provenance: F-6e3d6d0cc5b4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6e3d6d0cc5b4

## Purpose

F-6e3d6d0cc5b4 selects a 28-byte record from an existing free-record chain or from a sequential allocation region. When that region is exhausted, it requests a new region and links its descriptor into the supplied state. It then clears selected fields of the chosen record.

The function has no explicit return value. Its observable results are changes to the supplied state, newly obtained storage, and the selected record.

The interpretation as a record pool is inferred intent. The established behavior is narrower: pointer selection, conditional storage requests, specific state updates, and partial record initialization. The function does not establish a complete initialization policy for all 28 record bytes.

## Interface

### Input and result

The function accepts one pointer to a mutable state object. In the supplied representation, the state contains four consecutive 32-bit words:

| Byte offset | Width | Behavioral role |
|---|---:|---|
| 0 | 32 bits | Address of the current region descriptor, or zero |
| 4 | 32 bits | Address of the first reusable record, or zero |
| 8 | 32 bits | Address of the next sequential record |
| 12 | 32 bits | End address used to detect exhaustion of the sequential region |

There is no explicit returned record address, status, or error value.

The address selected during execution is used internally for record initialization. How a caller subsequently identifies that record is not established by the supplied evidence.

### Region descriptor

A newly requested descriptor occupies 12 bytes:

| Byte offset | Width | Value written |
|---|---:|---|
| 0 | 32 bits | State’s descriptor address, read after both storage requests |
| 4 | 32 bits | Computed record capacity |
| 8 | 32 bits | Address returned by the record-storage request |

For growth sizing, an existing descriptor is accessed only at offset 4. Its capacity field is interpreted as unsigned 32-bit data.

### Record layout relevant to this function

Each record occupies 28 bytes. This function uses or modifies these locations:

| Byte offset | Width | Meaning in this operation |
|---|---:|---|
| 0 | 8 bits | Cleared during initialization |
| 4 | 32 bits | Cleared during initialization |
| 8 | 32 bits | Cleared during initialization |
| 12 | 32 bits | Read as the next free-record address on reuse, then cleared |

Bytes 1–3 and 16–27 are not initialized by this function. Their meaning is unknown.

### Preconditions for ordinary successful operation

The supplied state must be readable and writable at the offsets used by the selected path. Relevant record and descriptor addresses must designate accessible storage with suitable alignment for the 32-bit accesses.

A normal pool interpretation additionally requires:

- A nonzero reusable-record address identifies a writable record with a readable 32-bit link at offset 12.
- A sequential cursor that differs from the stored end address identifies space for a complete 28-byte record.
- A nonzero current descriptor identifies readable capacity data at offset 4.
- Storage supplied by the dependency supports the accesses performed here.
- Address calculations do not wrap into unrelated or inaccessible storage.
- State, descriptors, and records do not overlap in ways that invalidate their distinct roles.

These conditions are not validated. In particular, the function does not prove that a sequential cursor lies below its end address or that 28 bytes remain available.

## Behavior

### Selection priority

The reusable-record address at state offset 4 is read first.

If it is nonzero, the function selects that record. It does not inspect the sequential cursor or end address for allocation purposes and makes no storage requests itself.

If it is zero, the function reads the sequential cursor at state offset 8 and compares it with the end address at offset 12:

- If the addresses differ, the existing cursor is selected.
- If they are equal, a new descriptor and record region are requested.

Exhaustion is an equality test, not an ordering or remaining-size test. A cursor greater than the end address, or otherwise inconsistent with it, still follows the existing-region path when the two addresses differ.

### Reusing a record

For a selected reusable record, the function reads the 32-bit value at record offset 12 and writes that value into state offset 4.

This unlinking occurs before any record initialization. The link is therefore captured before record offset 12 is cleared.

On this path, the function makes no direct writes to state offsets 0, 8, or 12. It does not validate the link or traverse the remainder of the chain.

### Using an existing sequential region

When the reusable-record address is zero and the sequential cursor differs from the end address, the cursor is selected without requesting storage.

Before initializing the record, the function writes the selected address plus 28 to state offset 8. It does not directly change the current descriptor or end address.

This path advances by one full record regardless of the distance between the cursor and the end address.

### Computing a new capacity

When the sequential cursor equals the end address, capacity depends on the descriptor address read from state offset 0.

If that address is zero, the new capacity is 32 records.

Otherwise, the existing descriptor’s unsigned 32-bit capacity, denoted here by \(c\), determines two candidates:

\[
a = (c + 32) \bmod 2^{32}
\]

\[
b = \left(c + \left\lfloor c/2 \right\rfloor
              + \left\lfloor c/8 \right\rfloor\right)
    \bmod 2^{32}
\]

The new capacity is the unsigned maximum of these wrapped values. If they are equal, the numerical result is that common value.

The shifts producing the half-capacity and eighth-capacity terms are unsigned right shifts. Their discarded fractions are not combined before rounding.

For capacities that do not encounter arithmetic wraparound, the rule provides at least 32 additional records and may instead choose approximately 1.625 times the previous capacity. That growth description is inferred intent, not a guarantee across all 32-bit inputs.

The record-storage request size is:

\[
s = (28 \times \text{new capacity}) \bmod 2^{32}
\]

There is no overflow rejection, saturation, minimum-size correction, or explicit check that the requested storage can accommodate the nominal capacity.

### Obtaining and publishing a new region

The function calls F-871e64ddc6de twice, in this order:

1. Request 12 units of storage with second argument zero.
2. Request \(s\) units of storage with second argument zero.

The returned values are used as the descriptor address and record-region address respectively. The apparent unit is bytes, based on subsequent accesses and record spacing; the dependency’s independent contract is unavailable.

After both calls return, writes occur in this order:

1. The record-region address is written to descriptor offset 8.
2. State offset 0 is read again, and that value is written to descriptor offset 0.
3. The computed capacity is written to descriptor offset 4.
4. The new descriptor address is written to state offset 0.
5. Descriptor offset 8 is read to obtain the selected record address.
6. The selected address plus \(s\) is written to state offset 12.
7. The selected address plus 28 is written to state offset 8.

The descriptor-chain link is therefore based on the state value read after the dependency calls. It is not guaranteed to equal the descriptor address used earlier to calculate capacity.

The function does not clear state offset 4 during region creation. It observed that field as zero when choosing this path, but does not recheck it after the dependency calls.

### Initializing the selected record

All selection paths converge on the same initialization sequence:

1. Write a 32-bit zero at record offset 4.
2. Write a 32-bit zero at record offset 8.
3. Write a 32-bit zero at record offset 12.
4. Write an 8-bit zero at record offset 0.

The byte at offset 0 is cleared last. No established evidence assigns it a publication, validity, or synchronization role.

The function does not clear the entire record. In particular, clearing offset 0 is a one-byte write, not a 32-bit write covering bytes 0–3.

## Side effects

### Direct memory changes

For valid, nonoverlapping objects:

| Path | State fields directly written | Other direct writes |
|---|---|---|
| Reusable record | Offset 4 | Selected record’s initialization fields |
| Existing sequential region | Offset 8 | Selected record’s initialization fields |
| New region | Offsets 0, 12, then 8 | Three descriptor fields and selected record’s initialization fields |

These are direct effects of this function. They do not constrain additional effects performed by F-871e64ddc6de.

If objects alias, an apparent record write can also modify state or descriptor storage. No alias detection or protective copying is present. The ordinary field roles above assume distinct objects; the ordered accesses remain relevant when that assumption fails.

### Ownership and lifetime

The function obtains storage and retains its addresses through the descriptor chain and state fields. It does not directly release any descriptor or record region.

The descriptor link is consistent with retaining earlier regions, but neither ownership transfer nor eventual reclamation is established. The dependency might have its own ownership rules or unrelated memory effects; none may be assumed from this body alone.

### Errors and partial changes

There is no explicit error return, failure branch, rollback, or diagnostic.

Neither storage result is checked before use. If the descriptor request returns zero and returns normally, the second request is still attempted before the first descriptor write. A zero or unusable record-region result is likewise used without validation.

Failures can occur after state has changed:

- A reused record is unlinked before initialization.
- A sequential cursor is advanced before initialization.
- A new descriptor and its end/cursor values are published before record initialization.

Consequently, a fault during initialization need not leave the original state intact.

The evidence does not establish whether invalid accesses produce a process fault, runtime trap, or other platform-specific outcome. They are not a supported recoverable-error interface.

No locking, atomic operation, or other synchronization is evident. Concurrent or reentrant use of shared state has no established safety guarantee.

## Dependencies

### F-871e64ddc6de

This is the only external dependency visible in the function body.

It is called only when the reusable-record address is zero and the sequential cursor equals the end address. Its two calls request a descriptor-sized allocation followed by a capacity-derived record region, each with a zero second argument.

Established use:

- Each returned value is treated as an address.
- The first is used for three 32-bit descriptor writes.
- The second becomes the initial record address and the basis of the end address.
- Both calls precede descriptor initialization.

Unknown dependency properties include:

- Whether requests are guaranteed to return normally.
- Whether returned storage is zeroed, aligned, distinct, or sufficiently large.
- Whether failure is represented by zero or another mechanism.
- Whether it reads or modifies the supplied state or other globals.
- Whether it allocates, recycles, retains, or releases other memory.
- What the second argument means.

The caller does not establish these properties through checks.

## Acceptance examples

These are conditional behavioral examples derived from the supplied static evidence. They are not runtime test results.

### Reusable record has priority

Assume state offset 4 identifies a valid record whose offset-12 link identifies another record. The sequential region may be exhausted.

Expected behavior:

- No call to F-871e64ddc6de occurs.
- State offset 4 becomes the saved link.
- The selected record’s offsets 4, 8, and 12 become 32-bit zero.
- Its offset 0 becomes an 8-bit zero.
- Its other bytes remain unchanged by this function.
- State offsets 0, 8, and 12 receive no direct writes.

### Existing region advances exactly one record

Assume state offset 4 is zero, the cursor and end differ, and a complete writable record exists at the cursor.

Expected behavior:

- No storage request occurs.
- The cursor advances by 28.
- The selected record receives the partial initialization described above.
- The end and descriptor address receive no direct writes.

Even if only fewer than 28 bytes remain according to the numeric addresses, the function performs the same operations; it has no remaining-space check.

### First region

Assume the reusable-record address is zero, cursor equals end, descriptor address is zero, and both dependency calls supply suitable storage without modifying the state.

Expected behavior:

- Request sizes are 12 and 896.
- Descriptor capacity becomes 32.
- Descriptor offset 0 becomes zero.
- State offset 0 identifies the new descriptor.
- The end becomes the region base plus 896.
- The cursor becomes the region base plus 28.
- The first record is partially initialized.

### Growth chooses between two rules

Assume a valid current descriptor and noninterfering successful storage requests.

- Previous capacity 32 produces candidates 64 and 52, so the new capacity is 64 and the region request is 1,792.
- Previous capacity 64 produces candidates 96 and 104, so the new capacity is 104 and the region request is 2,912.

These examples demonstrate both branches of the capacity comparison.

### Capacity arithmetic wraps

For previous capacity 4,294,967,280, the additive candidate wraps to 16. The shifted-sum candidate wraps to 2,684,354,534, which wins the unsigned comparison.

The resulting record-storage request wraps to 2,147,482,920. The function does not reject the discrepancy between nominal capacity and wrapped byte count. This example specifies arithmetic only, not a safe allocation scenario.

### A dependency changes the current descriptor

Assume growth sizing observes one descriptor, but a dependency call changes state offset 0 to another address before descriptor initialization.

Expected behavior:

- Capacity remains based on the earlier observation.
- The new descriptor’s link receives the later state value.
- The function then replaces state offset 0 with the new descriptor address.

No guarantee that the dependency preserves state is required to explain this ordering.

## Uncertainties

The following are documented_only:

- The semantic type of a record and the meaning of its uncleared bytes.
- How callers recover the selected record despite the lack of an explicit return value.
- Whether clearing byte 0 has significance beyond writing zero.
- The dependency’s storage, failure, ownership, alignment, and global-state contract.
- Whether external invariants prevent overflow, invalid cursors, overlapping objects, or cyclic links.
- Whether the surrounding system provides synchronization or excludes reentrancy.
- The platform-specific consequences of invalid addresses or wrapped address calculations.

Record-pool terminology and the apparent growth policy are inferred intent. They do not add validation, lifetime, or safety guarantees absent from the supplied body.

## Evidence

The supplied static body establishes the four-word state layout, 12-byte descriptor accesses, 28-byte record stride, unsigned 32-bit capacity arithmetic, dependency call order, branch conditions, and ordered field writes.

The source representation supports a 32-bit address model for this contract. It does not independently establish a host ABI, byte order, allocator specification, or runtime fault model.

No implementation was executed. No runtime tests have been performed. All acceptance examples are conditional consequences of static evidence, with dependency behavior and valid-storage assumptions stated explicitly.
