<!-- Trusted provenance: F-c18a7a4a6ea8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c18a7a4a6ea8

## Purpose

F-c18a7a4a6ea8 combines mask bytes associated with two active table entries, retains the entry selected by the smaller signed index, and removes the entry selected by the larger signed index.

The established operations are bytewise OR, optional preparation through a dependency, explicit resetting of two removed-descriptor words, optional transfer of trailing descriptors, restoration of the reset descriptor beyond the transfer’s destination range, and decrement of the active count.

Mask union, table compaction, and preservation of removed-entry storage for reuse are inferred intent. The transfer dependency’s implementation and the storage-ownership model are not supplied.

Rejected index pairs cause a diagnostic call. If that dependency returns, this function returns zero. A nonzero result from the preparation dependency is returned immediately. Zero therefore does not distinguish successful removal from rejection.

This contract is based on static evidence only. No runtime tests have been performed.

## Interface

The function accepts four arguments and returns a 32-bit integer.

| Argument | Established interpretation |
|---|---|
| Table reference | Address of a structure containing a signed 32-bit active count at byte offset 0 and a 32-bit entry-array address at byte offset 8. |
| First index | A 32-bit value interpreted as signed for ordering and validation; not dereferenced as a pointer. |
| Second index | Another signed 32-bit index, interchangeable with the first for selecting the lower and higher indices. |
| Opaque context | A 32-bit value forwarded to dependencies without interpretation. |

The table word at byte offset 4 is not explicitly accessed.

Each entry occupies 16 bytes:

| Entry byte offset | Established use |
|---|---|
| 0 | Unsigned 32-bit logical bit length; explicitly reset for the removed entry. |
| 4 | Opaque 32-bit word, saved and restored during the transfer path. |
| 8 | 32-bit address of mask bytes. |
| 12 | Opaque 32-bit word; explicitly reset for the removed entry. |

“Logical bit length” and “mask” describe how the fields participate in this function. Their broader application meaning is not established.

### Preconditions

Defined accepted-path execution requires:

- Readable table fields and a writable count field.
- Valid storage for all selected descriptors and any transfer range.
- Readable source mask bytes and readable/writable destination mask bytes for the accesses actually performed.
- Valid address calculations and representable transfer sizes.
- Continued validity of cached descriptor addresses after dependency calls.
- Dependency behavior compatible with the function’s subsequent accesses.

These are operational requirements, not checks performed here. There is no explicit null-reference, allocation-capacity, mask-address, aliasing, or arithmetic-overflow validation.

### Result

The function returns:

- The exact nonzero 32-bit result of F-6a2a5ddf37ed, when that call returns nonzero.
- Zero after completing its normal removal path.
- Zero after a rejected request’s diagnostic call returns.

No independent error result reports rejection, transfer failure, or invalid storage.

## Behavior

### Index ordering and rejection

The two indices are compared as signed 32-bit values. The lower index selects the retained entry; the higher index selects the removed entry. Reversing two distinct arguments preserves those selections.

Validation has observable short-circuit behavior:

1. Equal indices are rejected without reading the table count.
2. For distinct indices, a negative lower index is rejected without reading the table count.
3. For a nonnegative lower index, the table count is read.
4. Acceptance requires the higher index to be strictly below that signed count. Equality with the count is explicitly excluded.

Thus, under ordinary table invariants, acceptance requires two distinct nonnegative indices below the active count. Index zero is permitted.

A rejected request invokes F-4457a9a44a68 with a fixed diagnostic message, the normalized lower and higher indices, and two copies of the opaque context. Its return value is ignored. If it returns, this function returns zero without performing its own descriptor or mask writes.

The supplied representation does not expose the diagnostic message’s text.

### Descriptor selection and ordered reads

After validation, the following reads occur in this order:

1. Read the entry-array address from the table word at byte offset 8.
2. Calculate the removed-entry address using that value, the higher index, and a 16-byte stride.
3. Read the removed entry’s unsigned length.
4. **Read the table’s entry-array address at byte offset 8 again.**
5. Calculate the retained-entry address using this fresh value, the lower index, and a 16-byte stride.
6. Read the retained entry’s unsigned length.

The retained address is therefore not calculated from a cached copy of the first array-address read. Both descriptors normally belong to the same array if that field remains stable, but the two calculations use separately observed values.

Once calculated, the two descriptor addresses are retained. The function does not recompute them from the table after later dependency calls.

The removed entry’s initially observed length controls whether mask processing occurs and determines its OR byte count. If that length is zero, preparation, clearing, mask-address reads for combination, and bytewise OR are skipped. Removal still proceeds.

### Conditional preparation and bit clearing

When the removed entry’s initial unsigned length exceeds the retained entry’s initial unsigned length, F-6a2a5ddf37ed is called with:

- The retained descriptor address.
- The removed entry’s initial length.
- The opaque context.

A nonzero return terminates this function immediately with the same result. No subsequent function-owned clearing, OR, descriptor reset, transfer, restoration, or count decrement occurs. This is not a rollback guarantee: dependency effects may already have occurred.

Following a zero return, the function visits bit positions from the retained entry’s **initial** length, inclusive, to the removed entry’s **initial** length, exclusive.

For each position:

- The retained descriptor’s current length is reread.
- Clearing occurs only when the position is below that current unsigned length.
- For a position that passes this check, the current mask address is read from the retained descriptor.
- The addressed byte is read, the selected bit is cleared, and the byte is written back.

Within each byte, positions are mapped most-significant-bit first: position zero uses the high bit, and position seven uses the low bit.

The byte offset is formed by interpreting the 32-bit position as signed and shifting it right by three. For ordinary nonnegative signed positions this agrees with division by eight. High-bit-set positions must not be described as unrestricted unsigned division; their signed-shift and address behavior remains qualified in **Uncertainties**.

A zero preparation result is not checked against the requested length. A dependency that leaves a shorter current length can cause some clearing operations to be skipped. It does not reduce the later OR byte count.

No explicit retained-length assignment occurs in this function. Any length adjustment on this path belongs to F-6a2a5ddf37ed.

### Bytewise combination

For a nonzero removed-entry initial length, the function next reads:

1. The removed descriptor’s current mask address.
2. The retained descriptor’s current mask address.

These two addresses are then advanced through the combination loop; they are not reread from the descriptors for every byte.

The OR byte count is calculated by adding seven to the removed entry’s initial length in unsigned 32-bit arithmetic, then shifting the wrapped result right by three.

For lengths through 4,294,967,288, this gives the mathematical byte count rounded upward. For lengths from 4,294,967,289 through 4,294,967,295, the addition wraps and the resulting byte count is zero.

Each iteration reads a source byte and a destination byte and writes their bitwise OR to the destination. Iterations proceed in increasing address order. The supplied representation does not establish the relative evaluation order of the two reads within one iteration.

The whole last byte participates. Bits beyond the source’s logical bit length are not masked away, so source padding bits can set corresponding destination bits.

No aliasing protection is performed. Clearing can alter the source if the masks overlap. During combination, an earlier destination write can affect a later source read when the byte ranges overlap in a suitable arrangement.

### Descriptor reset, transfer, and restoration

After mask processing, including the zero-length case, the function writes:

1. Zero to the removed descriptor’s word at byte offset 0.
2. Zero to its word at byte offset 12.

It does not explicitly clear the words at offsets 4 or 8 and does not explicitly release mask storage.

The table count is then reread. The trailing-entry count is calculated as this newly observed signed count minus the higher index minus one.

If the result is not positive, the function skips descriptor saving, transfer, and restoration.

If it is positive:

1. All four words of the reset removed descriptor are read and saved in increasing word-offset order.
2. F-transfer-unmapped is called with the removed descriptor as destination, the immediately following descriptor as source, and a byte size of sixteen times the trailing-entry count.
3. After that call returns, the four saved words are written in increasing word-offset order at the removed-entry address plus sixteen times the previously computed trailing-entry count.

The restoration address is **immediately beyond the destination transfer range**, not its final slot. More precisely:

- The destination range begins at the removed descriptor and spans the computed number of trailing descriptors.
- Restoration begins at the first byte after that range.
- That address is also the start of the last descriptor in the source transfer range.
- With a stable table layout and count, it is the old final active slot.

For example, moving two trailing entries writes the transfer destination into two slots beginning at the removed slot. Restoration writes the saved descriptor into the next slot, not over either transferred destination slot.

Under overlap-safe transfer semantics, trailing active entries move one position toward the start, and the reset removed descriptor is parked at the old final active slot. That compaction outcome is conditional on the dependency’s behavior.

The saved descriptor and restoration location are based on observations made before the transfer call. They are not recomputed from a count or array address changed by that dependency. No transfer result is checked.

Finally, the function rereads the table count, subtracts one, writes the result to the count field, and returns zero. This final update uses the post-transfer count observation, not necessarily the validation count or the count used to determine the transfer size.

## Side effects

Function-owned writes can include:

- Clearing selected retained-mask bits after successful preparation.
- OR-writing retained-mask bytes.
- Resetting the removed descriptor’s offset-0 and offset-12 words.
- Restoring four saved words immediately beyond the transfer destination range.
- Decrementing the currently observed table count.

Dependencies can additionally perform preparation, diagnostics, transfer, and unknown unrelated effects.

There is no explicit allocation or deallocation operation in this function. Retaining a mask address in an inactive descriptor is consistent with storage reuse, but it establishes neither ownership transfer nor eventual release.

The operation is not transactional. Preparation failure may leave dependency changes behind. Later operations have no rollback path. Dependencies must not be presumed to preserve the table, descriptors, masks, globals, or other state.

Relocation of the entry array during preparation is unsafe unless the previously calculated descriptor addresses remain usable. The function continues using those addresses rather than following a changed table field.

## Dependencies

### F-6a2a5ddf37ed

Receives the retained descriptor address, a requested unsigned length, and the opaque context. Called only when the removed entry’s nonzero initial length exceeds the retained entry’s initial length.

Zero permits processing to continue. Any nonzero result is returned unchanged.

Growth or capacity preparation is inferred intent. Allocation strategy, initialization, length updates, failure atomicity, ownership, and global effects are documented_only unknowns.

### F-4457a9a44a68

Receives a fixed diagnostic message, the ordered indices, and two copies of the opaque context for rejected requests.

Its result is ignored. Diagnostic reporting is inferred intent. Output destination, formatting, termination behavior, and other effects are documented_only unknowns.

### F-transfer-unmapped

This is a document-local opaque label for the external transfer dependency; no resolved opaque function ID was supplied.

It receives destination, source, and byte-count arguments describing adjacent descriptor ranges. The ranges overlap when more than one trailing entry is transferred.

Overlap-safe movement is required for the apparent compaction behavior. Actual copying semantics, failure behavior, ownership effects, and unrelated state changes are documented_only unknowns.

## Acceptance examples

These are conditional static deductions, not runtime observations. Unless stated otherwise, they assume valid storage, representable arithmetic, stable array addresses and counts, no interfering aliasing or dependency mutations, and overlap-safe transfer semantics.

1. **Reversed indices.** With count five, indices three and one retain entry one and remove entry three. Reversing the arguments preserves this selection.

2. **Equal indices.** Indices two and two invoke F-4457a9a44a68 without a table-count read. If it returns, the result is zero.

3. **Negative lower index.** Indices negative one and two are rejected without a table-count read. This function does not itself dereference the table on that path; the diagnostic dependency’s effects remain unknown.

4. **Upper bound and zero.** With count four, indices one and four are rejected after reading the count. Indices zero and one are accepted.

5. **Separate array-address observations.** If the first offset-8 read supplies array address A and the second supplies B, the removed descriptor is selected relative to A and the retained descriptor relative to B. This is an observation-level case, not a guarantee of supported concurrent mutation.

6. **Zero-length source.** A removed initial length of zero skips preparation, clearing, and OR. Descriptor reset, any required transfer/restoration, and count decrement still occur.

7. **Combination without preparation.** Retained length sixteen and removed length nine produce two bytewise OR writes. All eight bits of the second source byte participate, including seven beyond its logical ninth bit.

8. **Preparation and clearing.** Retained initial length three and removed initial length ten request length ten. If preparation returns zero and establishes current length ten, positions three through nine are cleared before two bytes are OR-written. Positions zero through two are not cleared.

9. **Preparation failure.** A nonzero preparation result is returned unchanged. No later function-owned merge or removal writes occur; dependency changes are not undone.

10. **Two trailing entries.** With count five and removed index two, the transfer size is 32 bytes. Its destination covers slots two and three; its source covers slots three and four. The saved reset descriptor is restored into slot four, immediately beyond the destination range. Under the stated transfer assumptions, the count becomes four.

11. **Last-entry removal.** With count four and removed index three, no transfer or save/restore occurs. The two reset words become zero and the count becomes three.

12. **Wrapped byte count.** A removed initial length of 4,294,967,295 produces a wrapped addition result of six and an OR byte count of zero. This does not establish safe completion: preparation and a potentially enormous clearing loop may precede that calculation.

## Uncertainties

The following remain documented_only:

- Full table and descriptor types and the meanings of opaque words.
- Allocation capacities, alignment, ownership, and lifetime rules.
- Preparation behavior beyond its observed arguments and status handling.
- Diagnostic effects and whether the diagnostic call returns.
- Transfer semantics, including overlap handling.
- Supported aliasing among table, descriptors, and masks.
- Concurrency, reentrancy, and synchronization guarantees.
- Valid application-level length limits.
- Target-specific consequences of overflowing signed index, count, transfer-size, or address calculations.
- Signed right-shift and resulting address behavior for high-bit-set clearing positions.

Unsigned wrap in the OR byte-count calculation is separately established; it must not be generalized into a guarantee for signed arithmetic elsewhere.

## Evidence

Established static evidence includes signed index normalization, short-circuit validation, **two ordered entry-array-address reads**, the 16-byte descriptor stride, field widths and offsets, unsigned length comparisons, dependency conditions, per-position length rereads, MSB-first clearing, wrapped OR byte-count arithmetic, forward bytewise combination, ordered descriptor resets, transfer arguments, restoration **immediately beyond the destination transfer range**, final count rereading, and return-value handling.

Inferred intent includes mask union, capacity growth, diagnostic reporting, overlap-safe compaction, and storage reuse.

No dependency implementations, independent interface documentation, runtime traces, or runtime tests were supplied. Unknown dependency behavior is not treated as a preservation, ownership, or success guarantee.
