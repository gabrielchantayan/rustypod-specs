<!-- Trusted provenance: F-ced6cad2e982; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ced6cad2e982

## Purpose

F-ced6cad2e982 conditionally stages selected portions of a caller-supplied byte range, invokes F-3bc5215cf549 on accumulated staged data, and requests distribution of successful processing results. Other portions are requested for direct input-to-output transfer.

The function also decodes two 16-byte state fields, optionally writes back a chaining field, and overwrites one decoded temporary field before returning.

Block-oriented cryptographic processing is inferred intent, not an established algorithmic contract. The external transfer dependency is unresolved, and output-distribution calls do not expose a transfer length. Those limits prevent an unconditional specification of resulting output bytes.

## Interface

The function receives a descriptor address and returns an unsigned 32-bit value.

### Accessed layouts

All listed addresses and scalar fields are 32 bits. Offsets are in bytes.

| Object | Offset | Width | Use |
|---|---:|---:|---|
| Descriptor | 0 | 32 bits | State-object address |
| Descriptor | 4 | 32 bits | Input address |
| Descriptor | 8 | 32 bits | Output address |
| Descriptor | 16 | 32 bits | Byte count |
| State | 4 | 32 bits | Processing-context address |
| State | 12 | 32 bits | Encoded entry-validation word |
| State | 16 | 32 bits | Encoded flags |
| State | 32 | 32 bits | Encoded initial-bypass parameter |
| State | 36 | 32 bits | Encoded processed-region parameter |
| State | 40 | 32 bits | Encoded bypass-region parameter |
| State | 44 | 16 bytes | First processing field |
| State | 60 | 16 bytes | Chaining field |
| Processing context | 4 | 32 bits | Staging-buffer address |
| Processing context | 8 | 32 bits | Processed-output-buffer address |
| Processing context | 12 | 32 bits | Capacity or batching threshold |

These are accessed layouts, not complete object definitions. Descriptor offset 12 is not accessed.

Two distinct global 32-bit multipliers are used: an **entry multiplier** and a **processing multiplier**. Equality between them is not established. Four further globals supply the invalid-state, rejected-flags, descriptor-unavailable, and missing-buffer results. Their numeric values are unavailable.

### Preconditions and arithmetic

Explicit null checks do not establish address validity. Ordinary completion requires readable objects and valid memory for all resulting accesses and dependency requests. No complete bounds, capacity, alignment, or overlap validation is visible.

Word-sized products, sums, differences, and address calculations retain their low 32 bits. Comparisons are signed or unsigned as identified below. Signed division by 16 truncates toward zero; its remainder has the dividend’s sign or is zero. Byte-field transformations retain the low eight bits.

No ownership transfer, allocation, or release is explicit. The ownership behavior of dependencies is unknown.

## Behavior

### Entry checks and global-read ordering

Entry validation proceeds as follows:

1. A null descriptor returns the invalid-state global.
2. Otherwise, the state address is read from descriptor offset 0. A zero address returns the same global.
3. The entry multiplier and state validation word are read for a 32-bit product. A clear low bit returns the invalid-state global.
4. A separate read of the entry multiplier participates in decoding state flags. If both `0x20000000` and `0x00400000` are set, the rejected-flags global is returned.

The descriptor-unavailable global is then read **before** the repeated descriptor/state check, including on paths that subsequently process data. The descriptor is checked again, and its state-address field is reread when the descriptor is non-null. Failure returns the previously loaded descriptor-unavailable value.

The state address obtained by this later check is retained through processing and final writeback. It is not refreshed after dependency calls.

The byte count is then read and retained:

- Zero returns zero without reading data-buffer addresses or decoding the byte fields.
- Every nonzero count, including one, enters buffer handling.

For a nonzero count, the input address and output address are read and retained. The missing-buffer global is then read **even when both buffers are valid**. The presence condition tests the retained input address and, only when that input address is nonzero, rereads descriptor offset 8 to test output presence. Failure returns the loaded missing-buffer value.

Consequently, under unstable descriptor memory, the output address used later can differ from the address tested for presence. No subsequent descriptor reload corrects this discrepancy.

These are the established ordering relationships. The supplied representation does not independently establish the evaluation order between operands within an individual multiplication.

### Temporary fields

After successful nonzero-count buffer handling, two temporary 16-byte fields are prepared:

1. The chaining field, from state offsets 60–75.
2. The first processing field, from state offsets 44–59.

Each byte is multiplied by 123 modulo 256.

For each field independently, source and destination addresses determine traversal:

- Source address less than or equal to temporary destination: indices 15 down to 0.
- Source address greater than temporary destination: indices 0 up to 15.

The first field is completely prepared before the second begins. These directions describe individual access order, not a general overlap-safety guarantee.

The processing-context address is then read from the retained state object and retained for the remainder of the function.

### Configuration and retained values

A fresh processing-multiplier read decodes the state flags for the configuration decision.

If `0x80000000` is clear:

- Initial bypass is zero.
- Configured processed-region size is read from context offset 12.
- Configured bypass-region size is zero.

If that bit is set:

- Initial bypass is a wrapped product of state offset 32 and a separate processing-multiplier read.
- A signed 32-bit intermediate is obtained from state offset 36 and another processing-multiplier read.
- Configured processed-region size is the intermediate rounded toward zero to a multiple of 16.
- Configured bypass-region size is the wrapped sum of the product at state offset 40, using another processing-multiplier read, and the intermediate’s signed remainder modulo 16.

Thus the multiplier is not established as a single cached value shared by these products. If it changes between reads, the decoded parameters can reflect different values.

Initial bypass is replaced by the byte count if it is signed-greater than that count. Negative bypass values are not clamped.

The input address, output address, byte count, context address, and configured region sizes are retained across dependency calls. Later context-capacity reads do not update the configured processed-region size.

If the resulting initial bypass is signed-positive and retained input and output addresses differ, F-external-transfer is requested to transfer that many bytes from the beginning of input to the beginning of output.

### Whole-block boundary and iteration

The whole-block boundary is the wrapped sum of:

- Initial bypass.
- The signed difference between byte count and initial bypass, rounded toward zero to a multiple of 16.

For ordinary nonnegative lengths and an in-range initial bypass, this marks the end of the complete 16-byte portion following that bypass.

Iteration begins at the initial-bypass position, with zero staged bytes and a zero processing result. An iteration is entered only while the result is zero and the current position is signed-less-than the retained byte count. A result of one does **not** permit continuation.

For each iteration:

- The available whole-block span is the wrapped difference between the whole-block boundary and current position.
- Selected length initially equals that span.
- The batch-boundary decision initially requests processing.
- If configured processed-region size is signed-less-than the original span, selected length becomes that configured size. In this case, the boundary requests processing **only if the original span is signed-less-than-or-equal-to the wrapped sum of configured processed-region and bypass-region sizes**.

This condition controls batching independently of the context-capacity equality described next.

### Staging and processing

Only a signed-positive selected length enters staging.

The prospective accumulated length is computed as the wrapped sum of existing staged length and selected length. F-external-transfer is requested with:

- Destination: the staging-buffer address freshly read from context offset 4, plus existing staged length.
- Source: retained input address plus current position.
- Explicit count: selected length.

After that request returns, context offset 12 is reread. Processing is triggered if either:

- Accumulated staged length equals this freshly read capacity; or
- The iteration’s batch-boundary decision requests processing.

This capacity read occurs after staging, including when the boundary decision already requests processing. The staging dependency can therefore affect the equality test by changing the context.

When processing is triggered, the staging-buffer address is read **again**. A 16-byte transfer is requested from that address plus accumulated length minus 16 into a separate temporary snapshot. The snapshot can therefore use a different staging-buffer address from the preceding staging request.

There is no guard requiring accumulated length to be at least 16.

F-3bc5215cf549 is then called with:

1. Retained processing-context address.
2. Zero.
3. Temporary field decoded from state offset 44.
4. Temporary chaining field decoded from state offset 60.
5. Accumulated staged length.

Its unsigned 32-bit result becomes the current result. Only zero permits distribution and further iteration.

### Backward output distribution

Following a zero processing result, distribution begins with:

- Remaining staged length equal to the accumulated batch length.
- Current distribution length equal to the current selected length.
- Destination position equal to the current iteration position.

A distribution request occurs **only while current distribution length is unsigned-less-than-or-equal-to remaining staged length**.

For every such request, in order:

1. Current distribution length is subtracted from remaining staged length with 32-bit wrapping.
2. The processed-output-buffer address is freshly read from context offset 8.
3. F-external-transfer is called with destination at retained output plus destination position, and source at that freshly read buffer address plus the reduced remaining staged length.
4. Current distribution length becomes the retained configured processed-region size.
5. Destination position decreases by the wrapped sum of retained processed-region and bypass-region sizes.

Only destination and source are explicit in these calls. Their transfer lengths and resulting write extents are **documented_only**.

The unsigned guard is material: a bit pattern interpreted as negative elsewhere can be a large unsigned distribution length and fail this guard. Conversely, wrapping or non-progressing values do not establish termination merely because the intended operation is backward distribution.

After the distribution loop terminates, staged length is reset to zero. A 16-byte transfer from the pre-processing snapshot into the temporary chaining field is requested. The snapshot originates from staging, not from processed output.

### Bypass progress

If the current processing result remains zero, the following bypass length is initially the wrapped sum of:

- Configured bypass-region size.
- Configured processed-region size minus current selected length.

The remaining count is measured after the selected portion. If this remaining count is signed-smaller than the proposed bypass length, it replaces that length.

For a signed-positive bypass length, a transfer from retained input to retained output at the corresponding position is requested only when their base addresses differ.

The position then advances by selected length plus bypass length, including when either length is not positive. No independent forward-progress check exists.

For ordinary positive, compatible configurations, a final incomplete block can pass through this bypass path. That is not a guarantee for arbitrary parameter bit patterns. There is no separate post-loop flush of pending staged data.

### Final state decision, overwrite, and return

After loop termination, including termination caused by a nonzero processing result, the flags at offset 16 of the retained state object and the processing multiplier are read again.

If their product has `0x40000000` set and `0x80000000` clear, the temporary chaining field is encoded into state offsets 60–75. Each byte is multiplied by 179 modulo 256.

Traversal is descending when the temporary source address is less than or equal to the state destination address, and ascending otherwise. Since 123 and 179 are multiplicative inverses modulo 256, an unchanged decoded field re-encodes to its original bytes. This observation does not assume that dependencies preserve the temporary field.

The temporary field decoded from state offset 44 is then overwritten at indices 15 down to 0. Each overwrite byte is the low eight bits of:

- A successive address byte XOR `0xBA`,
- Multiplied by that same address byte plus `0x11`.

The address bytes begin with the low byte of a local counter location and advance by one per overwrite. The overwrite is not necessarily zero.

The function returns zero if no processing call supplied a nonzero result; otherwise it returns the terminating processing result. Transfer return values do not visibly replace that result.

## Side effects

- Initial and inter-region bypass requests may precede or follow processing calls.
- Selected data is requested into the context’s staging buffer.
- Successful processing is followed by backward output-distribution requests.
- Equal input and output addresses suppress bypass requests, but not staging or distribution.
- Partial overlap is not checked.
- Earlier effects are not rolled back after processing failure.
- Optional chaining writeback can occur after processing failure.
- Dependency mutations can affect fresh buffer-address reads, the post-staging capacity test, and final flags decoding.
- Such mutations do not cause retained descriptor values or configured region sizes to be reloaded.
- Only one decoded temporary field is explicitly overwritten. Complete erasure of chaining data, snapshots, staging, or processed output is not established.
- Invalid accesses have no visible recovery or exception translation.

## Dependencies

### F-3bc5215cf549

Opaque processing dependency with the five arguments specified above.

Zero permits distribution and continuation. Every nonzero return stops main-loop progress and becomes the eventual return value, provided subsequent operations return normally.

Its algorithm, memory accesses, context changes, global changes, temporary-field changes, aliasing requirements, and ownership behavior are unknown.

### F-external-transfer

Document-local opaque identifier for the unresolved transfer dependency.

Staging, snapshot, snapshot-to-chaining, and bypass requests expose destination, source, and count. Output-distribution requests expose only destination and source.

Ordinary copying is inferred intent. Actual transfer semantics, missing argument interpretation, overlap handling, return behavior, global preservation, and ownership behavior are **documented_only**. No assumption that this dependency preserves context fields or globals is justified.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Null descriptor.** The invalid-state global is returned without dereferencing the descriptor or calling either dependency.

2. **Rejected entry flags.** With a valid entry-validation bit and both rejection masks set in the separately entry-decoded flags, the rejected-flags result is returned before reading the descriptor-unavailable global or byte count.

3. **Zero bytes.** After successful entry checks, the descriptor-unavailable global is read and the repeated state check succeeds. A zero count returns zero without reading buffer addresses, reading the missing-buffer global, or decoding either byte field.

4. **Valid nonzero buffers.** The missing-buffer global is still read. The cached output address is retained for later use; the separate output-presence reread does not replace it.

5. **One-byte bypass.** Assume stable state, non-patterned configuration, positive capacity, and ordinary transfer behavior. Count one produces a zero whole-block boundary. No processing call occurs; one byte is requested for bypass transfer if the buffer addresses differ. Final flags handling and temporary overwrite still occur.

6. **Boundary-triggered 32-byte batch.** Assume patterned configuration with initial bypass zero, processed-region size 16, bypass-region size 16, count 64, stable capacity 64, and ordinary transfers. At position zero, the original whole-block span is 64, so `64 ≤ 32` is false: 16 bytes are staged without processing. After bypassing 16 bytes, the next span is 32, so `32 ≤ 32` is true. Another 16 bytes are staged and processing is called with length 32 despite the capacity being 64.

7. **Distribution order for that batch.** If processing returns zero, distribution first reduces remaining staged length from 32 to 16 and requests output at offset 32 from processed output at offset 16. It then reduces the remainder to zero and requests output at offset zero from processed output at offset zero. The next unsigned guard fails because 16 exceeds zero. Request lengths remain unresolved.

8. **Dependency-sensitive rereads.** If staging changes context offset 12, the new capacity governs the equality trigger. If it changes context offset 4, a triggered snapshot uses the newly read staging address. If a distribution request changes context offset 8, the next distribution request uses the new processed-output address.

9. **Processing failure.** A nonzero processing return suppresses distribution and subsequent bypass progress for that iteration. Final flags decoding, conditional chaining writeback, and temporary overwrite remain reachable before that value is returned.

10. **Zero-capacity non-progress.** With non-patterned configuration, zero capacity, and a positive signed count, selected and bypass lengths remain zero. No processing call or position advance occurs. No configuration-error return is provided.

## Uncertainties

The following remain **documented_only**:

- Numeric global values and complete object definitions.
- The transfer dependency’s actual contract, particularly distribution lengths.
- Buffer capacities, supported overlaps, and valid alignment.
- Dependency preservation of globals, state, context, or temporaries.
- Ownership and lifetime requirements beyond the visible accesses.
- Termination and memory safety for negative sizes, wrapped addresses, malformed configurations, and insufficient staging storage.
- Snapshot validity when accumulated length is below 16.
- Platform-level concurrency, faults, and pointer-ordering guarantees.
- Whether the final overwrite provides any security-grade erasure.

Cryptographic interpretation and conventional byte-copy effects remain inferred intent rather than established behavior.

## Evidence

Established claims derive from static inspection of the supplied function representation: field offsets, operand widths, explicit global reads, retained values, dependency arguments, signed and unsigned comparisons, branch conditions, and access ordering.

No dependency implementation or independently validated calling convention was supplied. No runtime tests, traces, or behavioral experiments have been performed.
