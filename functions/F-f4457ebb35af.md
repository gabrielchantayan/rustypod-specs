<!-- Trusted provenance: F-f4457ebb35af; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f4457ebb35af

## Purpose

F-f4457ebb35af invokes a constructor callback, checks a context field after successful construction, and, only when the resulting status is zero, conditionally edits text referenced by descriptor entries.

Its established phases are:

1. Obtain a temporary value from F-1983453691c7.
2. Publish the descriptor through the context and invoke the callback.
3. Process callback outputs, produce diagnostics on specified failure paths, clear the context field, and make cleanup-like calls.
4. If the status remains zero, inspect entries and remove at most one recognized byte sequence from each qualifying text.

Constructor registration, diagnostic formatting, and marker removal are inferred purposes. The visible operations establish the contract below without establishing those broader interpretations.

## Interface

### Arguments and representation

The function accepts five arguments:

| Argument | Interpretation |
|---|---|
| Context | Address of storage with a four-byte field at byte offset `0x108`. |
| Descriptor | Address of a structure containing four-byte values and addresses. |
| Auxiliary record | Address of storage with relevant four-byte values at offsets `0` and `8`. |
| Constructor callback | Callable address with the six-argument interface below. |
| Diagnostic destination | Address of a writable four-byte slot on reached diagnostic paths. |

The supplied representation uses 32-bit integer values and four-byte pointer-bearing fields. It does not establish a portable interface for wider pointers.

Relevant descriptor layout:

| Byte offset | Meaning in this function |
|---|---|
| `0x00` | Argument to the initial F-1983453691c7 call; read again on one diagnostic path. |
| `0x04` | Entry count, compared as a signed 32-bit integer. |
| `0x08` | Entry-array address. |
| `0x40` | Publication destination for a non-null object produced on callback success. |
| `0x44` | Callback argument captured before the initial dependency call. |
| `0x48` | Callback argument captured before the initial dependency call. |

Entries have a stride of `0x14` bytes:

| Entry-relative offset | Meaning |
|---|---|
| `0x08` | Four-byte text address; zero skips text processing. |
| `0x13` | One-byte flag written as `1` after the editing branch. |

Other fields have no established meaning here.

### Callback interface

The callback receives, in order:

1. The context address.
2. The auxiliary record’s four-byte value at offset `8`.
3. The captured descriptor value from offset `0x44`.
4. The captured descriptor value from offset `0x48`.
5. The address of a four-byte returned-object slot, initially zero.
6. The address of a four-byte error-value slot, initially zero.

It returns a 32-bit status. Zero selects success handling; every nonzero value selects failure handling.

Both output slots are invocation-local storage. Their addresses are valid during the invocation, but no supported lifetime after return is established. The callback can expose those addresses to another participant; subsequent calls therefore cannot be assumed to preserve their contents.

### Return value

The function returns:

- `7` when the initial F-1983453691c7 result is zero;
- `1` when the callback returns zero but the context field is nonzero at the actual post-initialization check;
- otherwise, the callback’s status.

A nonzero callback status is preserved rather than normalized.

### Preconditions

Every reached read, write, and indirect call requires suitable storage or a valid callable address. There is no general null-argument validation.

In particular:

- The descriptor’s captured fields and initial word must be readable before initial acquisition succeeds or fails.
- After initial acquisition succeeds, the context field must be writable.
- A reached object-initialization path requires writable object fields.
- A reached diagnostic path requires a writable diagnostic destination.
- Entry processing requires readable metadata and sufficient readable and writable text storage for the operations below.

These are access requirements, not validations performed by this function. Disjointness of the arguments is not checked.

## Behavior

### Initial capture and acquisition

The returned-object slot is initialized to zero. The descriptor value at offset `0x48` is captured, followed by the value at `0x44`. The error-value slot is initialized to zero before the initial dependency call.

The function calls F-1983453691c7 with:

- the context;
- an opaque constant designated **T0**;
- the descriptor’s current four-byte value at offset `0`.

If the result is zero, the function returns `7`. It does not invoke the callback or directly write the context field, diagnostic destination, descriptor object field, or entries. It makes no cleanup-like call on this path. Effects of F-1983453691c7 itself remain unknown.

Otherwise, the result is retained as the initial temporary value.

The two captured descriptor arguments are not refreshed after this call. The descriptor’s other fields are not thereby frozen.

### Context publication and callback invocation

The function writes the descriptor address to context offset `0x108`, without saving or inspecting the old value.

It then invokes the callback. The auxiliary value at offset `8` is obtained for this invocation after initial acquisition and context publication. Thus the callback arguments combine earlier-captured descriptor values with a later-read auxiliary value.

### Zero callback status

If the callback returns zero and the returned-object slot is nonzero, object processing occurs before the context check:

1. The auxiliary record’s then-current first word is written to the returned object’s first word.
2. The returned object’s second word is written as `1`.
3. The returned-object address is published at descriptor offset `0x40`.

The auxiliary first word is read after the callback, so callback changes can affect the value copied.

If the returned-object slot is zero, this function does not directly write descriptor offset `0x40`. Its previous contents are not automatically cleared.

These operations have no disjointness protection. If output storage or object storage overlaps other relevant storage, earlier writes can affect later accesses. A fixed object identity throughout these writes requires that the returned-object slot itself remain unchanged.

After optional object processing, the function reads context offset `0x108`:

- Zero leaves the status zero.
- Any nonzero value triggers a call to F-1983453691c7 with the context, opaque constant **T1**, and a fresh read of descriptor offset `0`. The status becomes `1`, and the dependency result is written to the diagnostic destination.

The test is not equality with the published descriptor address. It is a zero/nonzero test at this particular point.

A nonzero context field immediately when the callback returns does **not** alone establish failure: intervening object initialization or publication can change that field through aliasing. Conversely, those writes can make the field nonzero.

There is no rollback of object writes or publication when this check produces status `1`. A nonzero error-value slot is ignored on the zero-callback-status path.

### Nonzero callback status

The function does not initialize or publish the returned object when the callback returns nonzero, even if the returned-object slot is nonzero.

It reads the error-value slot to select one of two branches:

- **Slot is zero at the branch check:** Call F-1983453691c7 with the context, opaque constant **T2**, and the initial temporary value. Write the result to the diagnostic destination.
- **Slot is nonzero at the branch check:** Call F-1983453691c7 with the context and **T0**; no third argument is visible at this call site. Write the result to the diagnostic destination. Then read the error-value slot again and pass its **then-current value** to F-94cb53877c15.

The second branch does not preserve the initially checked error value for cleanup. The diagnostic-producing call may change the slot if its address has been exposed, and the diagnostic-destination write may itself overlap the slot. Consequently, F-94cb53877c15 can receive a different value, including zero.

There is no second nonzero test before that call. Nor is the callback-produced error value visibly passed as an argument to the diagnostic-producing call.

Neither branch changes the nonzero callback status.

### Context clearing and temporary handling

After every path that invoked the callback, the function:

1. Writes zero to context offset `0x108`.
2. Calls F-94cb53877c15 with the initial nonzero temporary value.

On the nonzero-error branch, the separate call using the reread error slot occurs before these operations.

The old context-field value is not restored. The final direct context write precedes a dependency call, so the field’s value after that call is not established without a dependency contract.

Entry processing occurs only if the status is zero after callback handling.

### Entry traversal

Traversal starts at index zero and continues while the signed 32-bit index is less than the descriptor’s signed 32-bit count.

A nonpositive count at the initial comparison performs no iterations. The count is read through the descriptor for loop comparisons. Entry addresses use the descriptor’s entry-array field; that field is not captured as an immutable array base.

For each reached entry:

1. Read its text address at offset `0x08`.
2. If the address is zero, skip the entry without writing its flag.
3. Otherwise, call F-1cb585febd84 with that address and retain its signed integer result as the length bound.
4. Perform the prefix comparison described below.

The text address is retained for that entry. Later changes to the metadata’s text-address field do not redirect its already-selected text processing. The final flag destination, however, is calculated using the descriptor’s entry-array field at that later point.

### Match selection

Two opaque constants participate:

- **H**, used in a comparison of six bytes at the beginning of the text;
- **P**, used in comparisons of seven bytes at successive text positions.

Their bytes are unavailable. F-abcb7c332db0 receives the opaque constant, candidate text address, and width. Its zero result controls the match branches; ordinary byte-comparison semantics are conditional, not independently established.

The initial comparison occurs even when the reported length is zero or negative.

The prefix qualifies when the six-byte comparison returns zero and text byte `6` is zero or ASCII space (`0x20`). The boundary read is skipped if the comparison returns nonzero. If the first read of byte `6` is nonzero, a further read is used for the space check.

A qualifying prefix selects removal position zero.

Otherwise, scanning begins at position zero and continues while the signed position is less than the reported length. At each position:

- Compare seven bytes against P.
- Only if the comparison returns zero, inspect the byte seven bytes after that position.
- If that byte is zero or ASCII space, select a removal position one byte beyond the comparison start and stop scanning.

Without a qualifying scanned match, the position reaches the length for ordinary nonnegative lengths, preventing editing.

A qualifying scanned comparison does not independently guarantee editing: the incremented removal position must still be less than the reported length.

### Editing and flagging

If the selected position is less than the reported length, the function reads the byte six bytes beyond that position and fixes a removal distance:

- Six when that byte is zero.
- Seven otherwise.

This decision is not recalculated during copying. The nonzero case does not perform another space check.

Bytes are copied individually in increasing destination-offset order, from the destination plus that fixed distance. Copying continues while the source offset is less than or equal to the reported length.

Under conventional string-length semantics, this includes the terminating zero when the loop reaches it. With inconsistent dependency results or changed text, termination is not guaranteed by that interpretation.

After copying, the function reads the byte at the selected position. If it is zero and the selected position is positive, the preceding byte is overwritten with zero. There is no check that the preceding byte is a space.

Finally, the selected entry’s flag byte is written as `1`. This write occurs even if the copy loop performed no iterations. The flag is not cleared on skipped or nonmatching paths.

At most one editing branch is executed per entry visit. The resulting text is not rescanned.

## Side effects

Direct effects include context publication and clearing, optional two-word object initialization, optional object publication, diagnostic writes, in-place byte copying and truncation, and entry-flag writes.

The function also passes values to F-94cb53877c15:

- The initial temporary value after every completed callback-processing path.
- A freshly reread error-slot value on the branch selected by an earlier nonzero error-slot check.

The latter argument need not remain nonzero or equal the callback-produced value.

No direct allocation is visible. No ownership transfer, deallocation, reference-count operation, or destructor contract is established by these calls. The returned object is not visibly passed to a cleanup dependency.

Aliasing is observable, not ruled out. Object writes can affect the later context check; a diagnostic write can affect the subsequent error-slot read; text writes can affect metadata; and the final flag write can alter overlapping text. Dependency calls may also mutate reachable state. There is no transaction or rollback.

## Dependencies

| Opaque ID | Established interaction | Unestablished behavior |
|---|---|---|
| F-1983453691c7 | Initial acquisition and later diagnostic-producing calls; four-byte results; differing visible argument counts. | Constant interpretation, formatting, ownership, failure meanings beyond tested zero, and mutations of accessible storage. |
| F-94cb53877c15 | Receives the initial temporary value and, on one branch, the error slot’s value reread after diagnostic production and publication. | Whether the argument is freed, released, recorded, or otherwise consumed; behavior for zero; other mutations. |
| F-abcb7c332db0 | Receives a constant, text address, and width `6` or `7`; zero selects possible matches. | Actual comparison semantics, read bounds, and mutations. |
| F-1cb585febd84 | Receives a nonzero text address; its signed result bounds scanning and copying. | Actual length semantics, access requirements, and mutations. |

The constructor callback is an additional dependency. Its permitted mutations and ownership obligations are not supplied.

## Acceptance examples

These are conditional branch specifications, not runtime observations. Unless an example expressly discusses aliasing, it assumes unrelated storage does not overlap and dependencies do not make additional relevant mutations.

1. **Initial acquisition fails.**  
   If the first F-1983453691c7 result is zero, return `7`. Do not invoke the callback or directly write the diagnostic destination.

2. **Successful callback with no object.**  
   If the callback returns zero, leaves its object slot zero, and the context field is zero at the subsequent check, retain status zero. Do not directly write descriptor offset `0x40`. Process entries after context clearing and the temporary-handling call.

3. **Post-initialization context failure.**  
   If the callback returns zero and the context field is nonzero when checked **after** optional object initialization and publication, produce the T1 diagnostic and return `1`. Do not process entries. Any preceding object writes remain in effect.

4. **Failure with a nonzero error-slot check.**  
   If the callback returns `23` and the error slot is nonzero at the branch check, return `23`. Produce and store the diagnostic before rereading the slot for F-94cb53877c15. If the diagnostic destination aliases that slot and the diagnostic result is zero, this cleanup-like call receives zero. Then clear the context field and pass the initial temporary value to F-94cb53877c15.

5. **Success with an error value.**  
   A nonzero error slot does not independently cause failure or a call using that value when the callback status is zero.

For the following text examples, additionally assume conventional length and comparison semantics, sufficient storage, no relevant mutations by either text dependency, no intervening concurrent mutation, and no overlap between text and descriptor or entry metadata.

6. **Prefix removal.**  
   If the text consists of the six H bytes, a space, and a suffix, the prefix selects position zero and distance seven. The suffix and terminating zero move to the beginning; the entry flag becomes `1`.

7. **Terminal scanned match.**  
   If there is no qualifying prefix, the first qualifying P comparison starts at position three, and the reported length is ten with the terminator at position ten, removal starts at four. Distance six moves the terminator to position four; the subsequent truncation writes zero at position three. Bytes before position three remain unchanged.

8. **Multiple candidates.**  
   A qualifying initial prefix takes precedence over scanned candidates. Otherwise, the first qualifying scanned candidate stops scanning. At most one editing branch runs for that entry; later candidates are not processed again during that visit.

9. **No eligible entries.**  
   With a stable nonpositive signed count, no entries are read. A null text address skips its entry without writing the flag.

10. **Nonpositive reported length.**  
    With a non-null text address and reported length zero or negative, the initial six-byte comparison still occurs. No text-editing branch is reached under non-overflowing arithmetic.

## Uncertainties

The following remain **documented_only**:

- Exact bytes and meanings of T0, T1, T2, H, and P.
- Dependency bodies, global effects, memory ownership, and release obligations.
- Whether the callback is intended to clear the context field directly or indirectly.
- Validity rules for malformed text, overlapping storage, exposed output-slot addresses, and concurrent mutation.
- Runtime behavior after invalid access or arithmetic overflow.

Fixed-width comparisons and look-ahead reads are not bounded by the reported length. A scanned comparison near the end can request seven bytes from that position, followed by another boundary read on a zero comparison result. Readable capacity must come from caller or dependency guarantees not supplied here.

Arithmetic includes signed index increments, multiplication by twenty, address additions, and offsets increased by six or seven. The representation establishes 32-bit values but not portable overflow behavior. The traversal and text descriptions assume those calculations remain representable.

## Evidence

Established evidence is limited to the supplied static function representation: field widths and offsets, visible call arguments, access ordering, branch conditions, byte operations, and return paths.

The error-slot reread follows both diagnostic production and the diagnostic-destination write. The context check follows object initialization and publication. Exact text outcomes require assumptions beyond dependency return values because dependency mutations and metadata overlap are not excluded.

Purpose labels and conventional text-operation meanings are interpretations, not additional guarantees. No dependency bodies, constant contents, independent interface declarations, or runtime observations were supplied. No runtime tests have been performed.
