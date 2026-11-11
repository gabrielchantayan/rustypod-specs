<!-- Trusted provenance: F-5b548e2edc9f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5b548e2edc9f

## Purpose

F-5b548e2edc9f converts an extracted input region into one or more caller-selected destination fields. A descriptor selects the conversion category, destination-field offset, and, for scalar categories, stored width.

The function supports boolean-like values, delegated numeric conversions, allocated byte strings, and a four-word conversion. An extraction tag controls whether it processes one destination or a caller-specified number of destinations beginning at index one.

This is a source-facing behavioral contract based solely on the supplied static evidence. It does not establish that inputs conform to a particular serialization format. Names such as “boolean,” “string,” and “numeric” describe observable operations or inferred intent, not independently verified format semantics.

## Interface

### Arguments

The interface has four arguments, interpreted under the displayed 32-bit address and integer model:

1. **Input-state address.** Points to a writable structure containing at least five 32-bit words:
   - Byte offset 0: a word temporarily replaced during one extraction path.
   - Byte offset 8: another word temporarily replaced on that path.
   - Byte offset 16: a word passed to the allocation dependency.

   Other state fields and their meanings are unknown.

2. **Descriptor address.** Points to readable storage containing:
   - Byte offset 5: an 8-bit conversion selector.
   - Byte offset 12: a 32-bit destination-field offset.
   - Byte offset 16: an 8-bit scalar-storage selector.

   The function may read the conversion selector several times during one iteration. It does not cache the complete descriptor.

3. **Destination-table address.** Points to an array of 32-bit destination-base addresses. The function selects an entry and adds the descriptor’s field offset to obtain the address to write.

4. **Repetition control.** A signed 32-bit value. It affects iteration count only when the effective extraction tag equals 3. Otherwise, exactly one iteration is attempted.

### Return value

The function returns a 32-bit integer:

- **0:** the selected number of iterations completed.
- **3:** an explicit rejection path was taken.
- **An allocation-dependency status:** returned unchanged when that dependency reports a nonzero status.

Zero is not proof that all text was recognized, all input was consumed, or every selected destination was written. Some branches complete successfully without writing.

### Required storage and address validity

The caller must provide valid readable state, descriptor, table entries, and input regions for all accesses actually taken. Destination fields must be writable for the branch-specific size:

- One, two, or four bytes for ordinary scalar storage.
- Four bytes for an allocated-string address.
- Sixteen bytes for the four-word conversion.

There are no visible null, alignment, destination-capacity, table-length, or arithmetic-overflow checks. Address arithmetic follows the displayed 32-bit model; the exact language-level validity of overflow and out-of-object pointer arithmetic is not established.

## Behavior

### Extraction and iteration selection

The first operation calls F-85e566545ec7 with the input-state address and scratch output storage. The function subsequently treats that output as:

- A working-region start address.
- A boundary address.
- A tag byte.

It captures the boundary for later scalar and string processing. If the initial tag is zero, it returns 3 immediately, before inspecting the descriptor’s conversion selector or computing a destination.

Otherwise, it initializes a separate working cursor from the extracted start address and initially selects destination-table index zero.

When the descriptor selector is 7, the function performs an additional extraction before deciding whether to enter repetition mode:

1. It saves input-state words at byte offsets 0 and 8.
2. It replaces those words with the first extracted start address plus one and the first extracted boundary address minus one.
3. It calls F-85e566545ec7 again, using separate scratch output.
4. On normal return, it restores the two saved state words.
5. It uses the low byte of the second extraction’s third output word as the effective tag.

This second extraction does not replace the already captured scalar-processing cursor or boundary. Restoration covers only the two explicitly saved state words. Other dependency side effects are not undone.

If the effective tag equals 3:

- A repetition control of zero causes return value 3.
- The working cursor advances by one byte.
- The captured boundary moves backward by one byte.
- Processing starts at destination-table index one.
- The repetition count becomes the fourth argument.

For every other effective tag, processing starts at table index zero and uses a count of one. In particular, an effective tag of zero obtained by the selector-7 second extraction is not subjected to the initial zero-tag rejection again.

A negative repetition control is not rejected. It is not a supported finite-count precondition: the displayed loop decrements it until zero, with potential signed overflow and out-of-range accesses before completion. No portable eventual result is established.

### Per-iteration ordering

For each iteration, the function:

1. Reads the selected destination-table entry and the descriptor field offset, and computes the destination-field address.
2. Calls F-1fe3b1afff2c with the address of the working cursor and the captured boundary.
3. Reads the descriptor conversion selector and performs the selected operation.
4. Decrements the remaining count and increments the destination-table index.
5. Returns zero when the remaining count becomes zero; otherwise, repeats.

Thus even a selector that immediately rejects is reached only after destination-address computation and the cursor-helper call.

The cursor helper’s exact behavior is unknown. Whitespace skipping is plausible intent, not an established guarantee.

### Selector 0 and unsupported selectors

Selector 0 returns 3. Any selector outside 0 through 7 also returns 3.

These paths do not perform the branch’s destination write, but earlier extraction, cursor-helper, or previous-iteration effects may already have occurred.

### Selector 1: boolean-like recognition

The tentative result is initially zero.

If the cursor plus three is strictly below the boundary, the function compares four consecutive bytes against the lowercase sequence “true.” A match selects result one and advances the cursor by **five** bytes.

If that match does not occur, and the cursor plus four is strictly below the boundary, it compares five consecutive bytes against “false.” A match leaves result zero and advances the cursor by **six** bytes.

If neither sequence matches, the result remains zero and this branch does not explicitly advance the cursor.

No delimiter value is checked after either recognized spelling. The extra byte skipped after a match is not validated. The cursor can consequently move beyond the boundary even though all compared bytes were within the guarded range.

The result is then stored using the scalar-storage rules below. Unrecognized text is not rejected by this branch.

### Selectors 2, 3, and 4: delegated scalar conversions

- Selector 2 calls F-9239ebb6c943 with the address of the working cursor and the boundary.
- Selector 3 calls F-2f16e714b3bb with those arguments and an additional mode value of 0.
- Selector 4 calls F-2f16e714b3bb with those arguments and a mode value of 3.

The returned 32-bit value is stored using the scalar-storage rules. This function does not separately inspect a parse status, validate full consumption, or translate conversion errors.

Signedness, accepted syntax, cursor advancement, overflow handling, and any error encoding belong to those dependencies and remain unknown here.

### Scalar-storage rules

Selectors 1 through 4 inspect the descriptor byte at offset 16 after computing their result:

- Value 1: write the low eight bits.
- Value 2: write the low sixteen bits.
- Any other value: write all thirty-two bits.

These are truncating stores, not checked range conversions. No wider destination clearing occurs.

### Selectors 5 and 6: allocated byte strings

These branches act only when the working cursor is strictly below the boundary. Otherwise, they perform no allocation or destination write and proceed to normal iteration completion.

Let the byte distance from the working cursor to the boundary be the distance used by this branch:

- Selector 5 derives a copy length equal to that distance minus two.
- Selector 6 derives a copy length equal to that distance minus one.

The cursor then advances by one byte. The function requests an allocation of the copy length plus one from F-3ce007b692cc, passing input-state word 4 and the address of a scratch status value.

On a nonzero reported status, the function returns that status immediately. It does not execute the copy or destination store for this iteration.

On zero status, it:

1. Copies the derived number of bytes from the advanced cursor into the returned allocation.
2. Writes a zero byte immediately after those bytes.
3. Writes the allocation’s 32-bit address into the destination field.

The scalar-storage selector is ignored.

Neither leading nor trailing delimiter bytes are validated. The explicit guard establishes only a positive distance. For selector 5, a distance of one produces a negative mathematical copy length and a zero requested allocation size; subsequent size and address interpretation is unsafe or dependency-specific. Safe normal use therefore requires a representable distance of at least two for selector 5, or at least one for selector 6, together with valid allocation and copy behavior.

After the explicit one-byte cursor advance, this branch does not advance the cursor over the copied payload.

### Selector 7: four-word conversion

This branch calls F-7f65f5b64486 with:

- The address of the original extraction-start scratch slot.
- The original extraction-boundary scratch value.
- A constant value of 4.
- The address of a four-word scratch area.
- A final zero argument.

It then invokes F-7431d9b27a62 four times, once for each successive scratch word. Each result is stored immediately into the corresponding four-byte destination slot before the next conversion call.

The four destination writes occupy offsets 0, 4, 8, and 12 from the computed destination field. The scalar-storage selector is ignored.

This path uses extraction scratch state rather than the separate working cursor adjusted by the per-iteration helper. The first dependency receives the extraction-start slot by address and may update it, affecting a later iteration.

The last three words of the four-word scratch area overlap storage previously used by the second extraction. Their values after F-7f65f5b64486 depend on what that dependency writes. This function checks no status from that call before reading all four words.

## Side effects

- Extraction and conversion dependencies may mutate state reachable through their arguments or other state they access. Their broader effects are unknown.
- The selector-7 preliminary extraction temporarily changes two input-state words and restores those two words on normal return.
- Successful scalar iterations write one, two, or four bytes.
- Successful string iterations allocate storage, copy bytes, append a zero byte, and publish the allocation address.
- Selector 7 writes four successive 32-bit results, interleaved with four dependency calls.
- Earlier destination writes remain visible if a later iteration returns an error. There is no rollback.
- The function does not explicitly free allocations, release an overwritten destination pointer, or clean up previously completed iterations.
- Allocation lifetime, allocator ownership, and the caller’s release obligation are not established by this function alone.
- No explicit final write-back of the working cursor to the input state occurs. Dependency-mediated changes remain possible.

Aliasing is not prohibited by visible checks. If destination writes or dependencies modify the descriptor, destination table, input bytes, or input state, later reads may observe those modifications. Stability of those objects is therefore a necessary precondition for straightforward repeated-conversion behavior, not an enforced invariant.

## Dependencies

| Opaque ID | Established use | Not established |
|---|---|---|
| F-85e566545ec7 | Produces extraction scratch output; receives mutable input state. Called once, or twice for initial selector 7. | Tag meanings, validation, complete output layout, other mutations. |
| F-1fe3b1afff2c | Receives a mutable cursor slot and boundary before every iteration’s selector dispatch. | Whether it skips whitespace, validates bounds, or changes other state. |
| F-9239ebb6c943 | Produces a 32-bit scalar result from a cursor slot and boundary. | Syntax, signedness, failures, overflow, consumption. |
| F-2f16e714b3bb | Produces a 32-bit scalar result with mode 0 or 3. | Meaning of modes and conversion guarantees. |
| F-3ce007b692cc | Receives a state word, requested size, and status-output address; its result is used as an allocation address. | Allocation mechanism, ownership, failure conventions beyond the checked status. |
| F-7f65f5b64486 | Receives extraction scratch, boundary, constant 4, four-word output storage, and zero. | Output completeness, parsing rules, return semantics. |
| F-7431d9b27a62 | Converts each of four scratch words into a stored 32-bit result. | Conversion meaning and side effects. |

An external byte-copy dependency is also present, but no opaque function ID was supplied for it. Its result is ignored. Correct string completion requires it to copy the requested bytes successfully.

No dependency is assumed to preserve globals, preserve unrelated reachable memory, or own or release memory.

## Acceptance examples

These examples are conditional static expectations, not executed tests. They assume valid addresses, stable metadata, and dependency behavior sufficient for the stated path.

1. **Initial rejection:** If the first extraction returns tag zero, the function returns 3 without reading the descriptor selector or writing a destination.

2. **Single-item mode:** If the effective tag is not 3, a repetition control of zero does not suppress processing. One iteration targets table entry zero.

3. **Repeated mode:** If the effective tag is 3 and repetition control is two, the function adjusts the working region inward by one byte at each end and attempts table entries one and two. Entry zero is not selected.

4. **Zero repeated count:** If the effective tag is 3 and repetition control is zero, return value is 3 before any iteration.

5. **Boolean truncation:** If selector 1 sees “true” within its comparison bounds and storage selector is 1, it writes byte value one and advances the cursor by five bytes.

6. **Unrecognized boolean:** If neither spelling matches, selector 1 stores zero rather than returning a parse error.

7. **Scalar narrowing:** If a delegated scalar conversion returns 65,537 and storage selector is 2, the stored sixteen-bit value is one.

8. **Allocated substring:** If selector 5 sees the four-byte region consisting of a quote, “ab,” and a quote, it requests three bytes, copies “ab” from the advanced cursor, appends a zero byte, and stores the allocation address.

9. **Empty selector-6 result:** If selector 6 sees a region of distance one, it requests one byte and produces a zero-terminated empty allocation, assuming allocation and zero-length copy succeed.

10. **No-write success:** If selector 5 or 6 sees a cursor at or beyond the boundary, that iteration performs no destination write and can still contribute to a final return value of zero.

11. **Partial completion:** If a first iteration writes successfully and a later string allocation reports a nonzero status, the earlier write remains and that status is returned.

12. **Four-word ordering:** For selector 7, the first converted word is stored before the second conversion dependency is called; the four results are not committed atomically.

## Uncertainties

The following are **documented_only**:

- Exact extraction-output representation beyond the fields visibly consumed.
- Serialization format, delimiter meanings, and intended interpretation of tag 3.
- Dependency parsing, error, allocation, ownership, and global-state contracts.
- Behavior after invalid pointers, insufficient storage, arithmetic overflow, abnormal dependency exit, or incomplete scratch initialization.
- Exact byte-copy ABI and handling of invalid or negative-derived lengths.
- Destination alignment and target endianness requirements.
- Intended semantics of negative repetition counts.

Likely intent is typed field population from a bounded textual region, with tag-controlled aggregate handling. That interpretation does not strengthen the established checks or erase the unusual cursor advances, permissive boolean fallback, or no-write success paths.

## Evidence

Established behavior above derives from the supplied function body: explicit argument accesses, branch conditions, dependency calls, arithmetic, stores, and returns. Dependency bodies were not supplied.

No implementation access, runtime execution, smoke run, or tests were performed. Acceptance examples describe conditional consequences of static evidence only. No snapshot hash or independently verified provenance is asserted.
