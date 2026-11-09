<!-- Trusted provenance: F-cc6c9b4a0f68; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cc6c9b4a0f68

## Purpose

F-cc6c9b4a0f68 updates a supplied block’s metadata, optionally combines it with adjacent blocks, inserts the resulting block into a linked structure, adjusts a state counter, and returns the resulting block address.

**Established static behavior:** eligibility checks, conditional adjacent-block removal, header arithmetic, boundary-tag writes, list insertion, counter adjustments, and a rejection dependency.

**Inferred intent:** the operation resembles free-block coalescing followed by size-ordered reinsertion into an allocator’s available-block list. That interpretation does not establish ownership transfer, arena validity, double-release protection, or the meaning of the counter.

The contract below describes the supplied static representation. It does not assume that surrounding allocator invariants hold. No runtime tests have been performed.

## Interface

### Arguments and result

The represented interface has four arguments:

| Argument | Represented width and use |
|---|---|
| State address | A 32-bit address of storage containing a counter and an embedded list anchor. |
| Supplied block address | A 32-bit address of a header and associated metadata. |
| Third argument | An uninterpreted 32-bit value; no visible use. |
| Fourth argument | An unsigned 32-bit value whose incoming value does not affect acceptance or a successful result. The forward-boundary header replaces it before its relevant use. |

There are no explicit null-address checks.

On success, the result is a block address:

- The supplied address when no backward combination occurs.
- The preceding block’s address when backward combination occurs.
- Forward combination alone does not change the returned address.

On rejection, the function invokes **F-a817d8d58c10**, represented as non-returning. No ordinary failure result is visible.

### Widths and arithmetic

Addresses, headers, tags, and links are represented as 32-bit words. A word occupies four bytes.

A header’s **extracted size** is its value masked with **0x3ffffffc**:

- Bits 0 and 1 are excluded.
- Bits 30 and 31 are excluded.
- Bit 2 remains included.
- Extracted sizes are multiples of four, ranging from zero through 0x3ffffffc.

The visible flag conditions are:

| Bit mask | Behavior |
|---|---|
| 0x1 | An adjacent block must have this bit set to be combined. Finalization sets it on the resulting block. |
| 0x2 | Must be clear in the initial forward-boundary header. When set in the supplied header after optional forward combination, it requires backward validation and combination. Finalization sets it at the resulting following boundary. |
| 0x4 | Must be clear in the supplied block’s initial header. It is also part of the extracted size, not an excluded flag bit. |

Header enlargement adds an extracted adjacent size to a **complete header word**. It does not replace only the size field or explicitly preserve the header’s high bits.

No explicit overflow, underflow, or address-wrap check is visible. In particular, avoiding 32-bit overflow does **not** guarantee that combined sizes remain representable under 0x3ffffffc: a carry into bit 30 is discarded by subsequent size extraction.

The counter is accessed through a represented signed 32-bit word. Its exact signed-overflow semantics are not established. Counter claims therefore refer to the observed additions and subtractions, with ordinary mathematical interpretations conditional on representability.

### Relevant storage layout

At the state address:

| Byte offset | Meaning in the visible operation |
|---|---|
| 0x00 | Counter. |
| 0xd0 | Anchor comparison word; this location’s address is also the anchor link target. |
| 0xd4 | Anchor forward link. |
| 0xd8 | Anchor backward-link position, reachable through generic link manipulation. |

At a block address:

| Byte offset | Meaning |
|---|---|
| −0x04 | Previous-size tag, consulted only when backward combination is required. |
| 0x00 | Header. |
| 0x04 | Forward link. |
| 0x08 | Backward link. |
| Extracted size − 0x04 | Trailing size-tag location used during finalization. |
| Extracted size | Following boundary-header location. |

Links are interpreted directly as addresses. A zero link suppresses the corresponding neighbor update.

Accessible storage is a prerequisite for every address actually read or written. The function does not independently establish that blocks are distinct, large enough for their metadata, or within an arena.

## Behavior

### Initial checks and forward-boundary read

The function first checks eight-byte alignment of the address eight bytes after the supplied address. Under the represented address arithmetic, this imposes the same low-bit alignment requirement as eight-byte alignment of the supplied address.

If alignment fails, rejection occurs without first reading the supplied header.

If alignment passes, the function reads the supplied header. Its 0x4 bit must be clear. If that bit is set, rejection occurs without a forward-boundary read.

Otherwise, the function extracts the supplied size, adds it to the supplied address, and reads the header at that boundary. The boundary header’s 0x2 bit must be clear. If it is set, rejection occurs before any direct counter, header, tag, or link write by this function.

No visible check requires a positive size or a boundary distinct from the supplied header.

### Optional forward combination

The inspected forward-boundary header controls forward combination:

- If its 0x1 bit is clear, this combination is skipped.
- If its 0x1 bit is set, the forward block is combined and removed from its linked position.

The ordered effects of forward combination are:

1. Subtract the extracted size from the already inspected forward header from the state counter.
2. Reread the forward header and extract its size again.
3. Add that reread size to the supplied block’s complete header word.
4. Unlink the forward block.

Consequently, the size subtracted from the counter and the size added to the supplied header originate from different reads. They are equal only if the relevant storage remains unchanged.

Unlinking first examines the removed block’s forward link. If nonzero, the block at that address receives the removed block’s backward-link value in its backward-link field. The function then examines the removed block’s backward link. If nonzero, the block at that address receives the removed block’s forward-link value in its forward-link field.

These links are not established as one immutable snapshot. Reads in the second part can observe effects of earlier writes if storage overlaps.

### Required backward validation

After optional forward combination, the function rereads the supplied header and tests 0x2.

If clear, the supplied block is selected for finalization.

If set, backward combination is required for success. The function reads the word immediately before the supplied address and requires all of the following:

- The previous-size tag is nonzero.
- Its low two bits are zero.
- The address obtained by subtracting that tag from the supplied address contains a header whose extracted size equals the tag.
- That preceding header has 0x1 set.

For a nonzero previous-size tag, an additional supplied-header read occurs before the low-bit validation determines whether the preceding header is inspected. A tag with invalid low bits does not cause a preceding-header read through the rejected computed address.

Failure of any requirement invokes F-a817d8d58c10. This happens **after** any completed forward combination; those changes are not rolled back.

Successful backward combination:

1. Subtracts the preceding size established during validation from the state counter.
2. Rereads the preceding and supplied headers, adding the supplied header’s extracted size to the preceding block’s complete header word.
3. Unlinks the preceding block with the same forward-neighbor-before-backward-neighbor write order.
4. Selects the preceding address for finalization and eventual return.

The supplied size used here includes any earlier forward enlargement, subject to the extraction mask. The static expression does not independently establish an operand-read ordering between the two header reads within the enlargement.

### Final header, boundary, and tag

For the selected final block, the function:

1. Reads its header and writes that value with 0x1 set.
2. Uses the size extracted from that earlier header read to locate the following boundary.
3. Reads the boundary header and writes it with 0x2 set.
4. Reads the final header again to derive the trailing-tag address and the extracted size written there.

The trailing-tag calculation is not guaranteed to reuse the header value that located the boundary. The representation contains header accesses for both the tag address and tag value; it does not establish one shared snapshot for them.

For stable, nonoverlapping storage, the tag receives the final extracted size at four bytes before the following boundary. For overlapping or malformed layouts, earlier writes can change later reads or address calculations.

No minimum-size validation prevents a zero or small size from placing tags on headers, links, or preceding storage.

### List traversal and insertion

Traversal begins at the anchor address, **not** at the anchor’s forward link. The anchor’s comparison word participates in size comparison.

At each candidate, the final block’s current extracted size is compared with the candidate’s extracted size:

- If the final size is less than or equal to the candidate size, insertion is selected at that candidate.
- Otherwise, traversal follows the candidate’s forward link.
- If that forward link is zero, the final block is appended after the current candidate.

The final size and candidate fields are read during traversal rather than fixed as a complete list snapshot. Forward links also have repeated accesses.

Under a stable ascending-list invariant, equality at a non-anchor candidate inserts the new block before the first encountered equal-sized block. The function itself does not establish that the list is sorted.

There are three insertion forms.

**When the selected candidate is the anchor**, the function performs these writes in order:

1. Final block’s forward link receives the anchor’s current forward link.
2. Final block’s backward link receives the anchor address.
3. If the anchor forward link is nonzero, that block’s backward link receives the final block address.
4. Anchor forward link receives the final block address.

Thus, although the anchor participates in comparison, selecting it means insertion immediately **after** the anchor.

**When the selected candidate is not the anchor**, the function:

1. Sets the final block’s forward link to the candidate.
2. Copies the candidate’s backward link into the final block’s backward link.
3. If that candidate backward link is nonzero, updates the predecessor’s forward link to the final block.
4. Sets the candidate’s backward link to the final block.

**When appending after the last candidate**, the function:

1. Sets the final block’s forward link to zero.
2. Sets the last candidate’s forward link to the final block.
3. Sets the final block’s backward link to that candidate.

A represented branch also handles an anchor address of zero by choosing the anchor insertion form. The validity of state storage producing such an address is not established.

No visible cycle detection, reciprocal-link validation, repeated-membership check, or supplied-block membership check exists.

### Final accounting and return

After insertion, the function rereads the final block’s header, extracts its size, adds that size to the counter, and returns the final block address.

The accounting is defined by the actual extracted values:

- Subtract the inspected forward size if forward combination occurs.
- Subtract the validated preceding size if backward combination occurs.
- Add the final size extracted after insertion.

A net increase equal to the supplied block’s original extracted size follows only under additional conditions: stable nonaliasing storage, successful structural processing, representable counter arithmetic, and **every combined size remaining representable under 0x3ffffffc, with no carry into its excluded high bits**. Avoiding full-width overflow alone is insufficient.

## Side effects

Direct writes may affect:

- The state counter.
- The supplied or preceding block’s header.
- The final following-boundary header.
- The trailing size tag.
- Link fields in removed blocks’ neighbors.
- The final block’s links, insertion neighbors, and anchor.

The operation is not transactional. A rejected backward validation can leave forward counter subtraction, supplied-header enlargement, and forward unlinking in place.

No explicit allocation, external memory release, locking, atomic operation, or rollback is visible. Metadata combination does not establish that memory is returned to an external provider.

Ownership before and after the operation is **documented_only**. Neither successful insertion nor rejection establishes who may subsequently access, reuse, or release the memory.

## Dependencies

### F-a817d8d58c10

This dependency is invoked for:

- Failed initial alignment.
- Initial supplied-header 0x4 set.
- Initial forward-boundary-header 0x2 set.
- Failed mandatory backward validation.

No explicit arguments are supplied in the represented invocation. The dependency is annotated as non-returning.

Its diagnostics, termination mechanism, exception behavior, global changes, memory access, ownership effects, and memory-release behavior are **documented_only**. It must not be assumed to preserve globals or already modified metadata.

No other explicit dependency invocation is visible.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless stated otherwise, they assume accessible stable storage, nonconflicting metadata, consistent links, terminating traversal, representable counter arithmetic, and a valid insertion position.

### Standalone insertion

Given an eight-byte-aligned supplied address, a supplied header with extracted size 0x20 and bits 0x4 and 0x2 clear, and a forward header with bits 0x2 and 0x1 clear:

- Neither adjacent block is removed.
- The supplied header gains 0x1.
- The boundary at supplied address plus 0x20 gains 0x2.
- The trailing tag at supplied address plus 0x1c receives 0x20.
- The supplied block is inserted.
- The counter increases by 0x20.
- The supplied address is returned.

### Forward combination without backward combination

Given:

- An eight-byte-aligned supplied address.
- A supplied header with extracted size 0x20 and bits 0x4 and 0x2 clear.
- A forward header with extracted size 0x30, 0x1 set, and 0x2 clear.
- No interfering mutation or aliasing.

Then the initial eligibility checks succeed, forward combination occurs, and the supplied header’s 0x2 remains clear. Therefore:

- The counter is reduced by 0x30.
- The supplied header is enlarged and the forward block is unlinked.
- No backward validation or combination occurs.
- The final extracted size is 0x50.
- Final accounting adds 0x50, for a net increase of 0x20.
- The supplied address is returned.

An original extracted size of 0x20 alone does not establish these outcomes; the supplied header’s 0x2 condition is essential.

### Backward combination

Given an aligned supplied block with extracted size 0x20, 0x4 clear, and 0x2 set; an ineligible but otherwise accepted forward boundary; a previous-size tag of 0x40; and a preceding header with extracted size 0x40 and 0x1 set:

- Backward validation succeeds.
- The counter is reduced by 0x40.
- The preceding header is enlarged to extracted size 0x60.
- The preceding block is unlinked and reinserted.
- Final accounting adds 0x60.
- The preceding address is returned.

### Combined size crossing the extraction mask

Given successful forward eligibility, no backward combination, an original extracted size of 0x3ffffff0, and a forward extracted size of 0x30:

- The combined size contribution reaches 0x40000020 without requiring 32-bit overflow.
- Subsequent extraction yields 0x20.
- Forward accounting subtracts 0x30.
- Final accounting adds 0x20.

With an initial counter of 0x100, those updates produce 0xd0 and then 0xf0, provided the stated storage and arithmetic conditions hold. The net change is −0x10, not an increase of 0x3ffffff0.

### Equality and rejection

For a non-anchor candidate equal in extracted size to the final block, with no earlier selected position, insertion occurs before that candidate.

For an otherwise initially eligible supplied block whose forward boundary has 0x2 set, rejection occurs before direct mutation.

For a completed forward combination followed by required backward validation with a zero previous-size tag, rejection occurs after forward mutation, without visible rollback.

## Uncertainties

The following remain **documented_only**:

- Authoritative meanings of the flags, counter, and anchor comparison word.
- Caller-imposed arena, alignment, minimum-size, and list invariants beyond the visible checks.
- Ownership and lifetime obligations.
- Whether malformed, overlapping, wrapped, or cyclic structures are reachable.
- Exact target arithmetic and fault behavior beyond the represented operations.
- Complete ABI and hidden calling-context requirements.
- External synchronization and concurrent-mutation rules.
- Actual behavior of the rejection dependency, including its non-returning annotation.
- Evaluation ordering not independently established within compound expressions.

Allocator terminology describes inferred roles, not additional validation or safety guarantees.

## Evidence

Established claims derive from independent inspection of the supplied static representation: its conditional reads, 32-bit masks and arithmetic, explicit writes, unlink and insertion order, counter updates, return paths, and dependency invocation.

The accounting qualification follows from the extraction mask itself: bit 30 can become set through size addition without full-width overflow, and is then discarded. The forward-only acceptance conditions follow from the separate supplied-header 0x2 test after forward combination.

No implementation execution, runtime tests, caller inspection, dependency inspection, or independent provenance verification has been performed.
