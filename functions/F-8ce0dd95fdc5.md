<!-- Trusted provenance: F-8ce0dd95fdc5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8ce0dd95fdc5

## Purpose

F-8ce0dd95fdc5 conditionally extends a destination sequence with 16-bit elements read from a source sequence. Each input points to a 16-bit count field followed immediately by payload elements.

For non-null inputs, the function calculates a target count from the sum of the stored counts, capped by an unsigned 32-bit ceiling. It transfers source payload into the destination when the target exceeds the destination’s existing count, then writes the target’s low 16 bits into the destination count field.

Three behaviors distinguish this operation from a conventional capacity-checked append:

- A ceiling below the existing destination count reduces the stored count without clearing payload.
- The target can exceed 65,535. Transfers use the full target, but the stored count is truncated to 16 bits.
- Overlapping storage is processed through ordered forward transfers, without preserving a source snapshot.

These are deductions from the supplied static body. Interpreting the operation as a counted-sequence append is inferred intent, not an established application-level specification.

## Interface

The function accepts three inputs and returns no value.

| Input | Representation | Role |
|---|---|---|
| Source | Pointer to unsigned 16-bit elements | Identifies the source count field and source payload. |
| Destination | Pointer to unsigned 16-bit elements | Identifies the destination count field and destination payload. |
| Ceiling | Unsigned 32-bit integer | Upper bound on the calculated target count. |

Positions in this contract are measured in 16-bit elements relative to the corresponding input pointer:

- Position zero is the count field.
- Position one is the first payload element.
- Each element occupies two bytes.
- No byte-order or external serialization convention is established.

For count fields that remain stable during the initial reads, define:

- **S**: the source count used in target calculation.
- **D**: the destination count used in target calculation and observed again when locating the initial destination cursor.
- **T**: the smaller of the ceiling and S + D.
- **N**: T − D if T exceeds D; otherwise zero.

Both counts are unsigned 16-bit values, ranging from zero through 65,535. Each is widened to unsigned 32-bit form before addition. Their sum is at most 131,070, so this addition cannot overflow 32 bits.

Consequently:

- T ranges from zero through 131,070.
- N cannot exceed S.
- The final stored destination count is T modulo 65,536.

### Preconditions for successful memory operations

A null source or null destination is explicitly accepted. Either null input prevents all count and payload accesses through both inputs.

For non-null inputs, successful execution requires:

- Readable source and destination count fields.
- A writable destination count field.
- Readable source payload positions one through N at their respective transfer times.
- Writable destination payload positions D + 1 through T when N is positive.
- Valid pointer construction, advancement, and comparison for all positions formed, including cursor positions that are never dereferenced.
- Storage that remains valid throughout the accesses performed.

The function constructs the destination position corresponding to T for its stopping comparison. It also advances both cursors before the first stopping decision. A no-copy case therefore still requires valid address operations beyond simply reading and writing the count fields.

These requirements do not establish who owns the storage or which party supplies its lifetime guarantees. The function does not verify allocation size, capacity, alignment, or lifetime.

Overlap is not explicitly rejected. Its consequences follow from the transfer order described below.

## Behavior

### Null-input handling

If either pointer is null, the function returns without reading either count field, reading payload, or writing storage.

A non-null pointer is not dereferenced when the other pointer is null. Thus a null source also prevents access through a non-null destination, and vice versa.

There is no returned status distinguishing this case from other completion paths.

### Initial reads and target calculation

When both pointers are non-null, the function reads the source and destination count fields and adds their values after unsigned 32-bit widening. If the ceiling is smaller than that sum, the ceiling becomes the target; otherwise the sum becomes the target.

The supplied expression does not establish the relative order of these two initial count reads. Both precede payload transfers.

The function then reads the destination count field again to locate the initial destination cursor. With stable counts, that cursor identifies destination position D: the last existing payload element when D is positive, or the count field when D is zero.

The source cursor initially identifies the source count field.

The formulas involving D assume that the two destination count reads agree. The body performs no intervening write, but it provides no synchronization against external changes. Outcomes under such mutation are not assigned a deterministic result by this contract.

The source count is not reread to determine transfer length. Once T has been calculated, subsequent changes to the source count field do not cause target recalculation.

### Transfer sequence

Before each stopping decision, the function advances both cursors by one 16-bit element.

It then compares the advanced destination cursor with the destination position corresponding to T:

- If the cursor is beyond the target position, transfer processing ends.
- Otherwise, the function reads the current source element and writes that value to the current destination element.

For stable initial counts and valid address operations:

- Exactly N payload transfers occur.
- Transfer k, numbered from one, reads source position k.
- That transfer writes destination position D + k.
- Transfers occur in increasing k order.
- Each transfer reads its source value before performing its destination write.
- Each transfer completes before the next transfer.
- If T is equal to or below D, no payload element is read or written.

The terminating iteration advances both cursors but performs no payload access. At that point:

- The source cursor is at source position N + 1.
- The destination cursor is at destination position one greater than the larger of D and T.

These terminal positions are relevant to pointer-validity requirements even though the function does not dereference them on the terminating iteration.

The target bounds the transfer endpoint, not the validity of the underlying storage. No capacity check accompanies any payload read or write.

### Final count write

After transfer processing ends, the function writes T converted to unsigned 16-bit form into destination position zero.

For valid execution with both pointers non-null, this write occurs even when:

- No payload was transferred.
- The ceiling reduced the destination count.
- The new stored count equals the previous count.
- T exceeds the representable range of the count field.

The conversion retains T modulo 65,536. For example, T = 65,536 produces a stored count of zero.

The function does not clear payload excluded by a reduced count, append a terminator, record the full-width target elsewhere, or reject an unrepresentable count.

### Overlap and aliasing

The function does not take a payload snapshot or select a different direction for overlapping buffers.

An earlier destination write can overwrite a source element that a later transfer will read. That later transfer uses the overwritten value. Propagation therefore depends on the relative positions of the source and destination regions.

A transfer that reads and writes the same physical element is a self-copy. Self-copying alone does not propagate that value into subsequent source elements.

A destination payload write may also overlap the source count field. Such a write changes the shared storage, but does not change the previously calculated T.

If the input pointers are identical, the initial count reads observe the same field under stable-memory conditions. Subject to the ceiling, the function copies the existing payload into positions after that payload. The final count write is visible through both pointers.

## Side effects

For stable counts and valid execution, direct writes are limited to:

1. Destination payload positions D + 1 through T, when T exceeds D.
2. Destination position zero, after all payload transfers.

Destination payload positions one through D are not direct payload-write targets. Count reduction leaves the corresponding old payload contents in storage.

Overlap can make destination writes visible through the source pointer. The source designation does not imply that the underlying storage remains unchanged.

The supplied body shows no allocation, deallocation, ownership-management operation, pointer retention, global-state access, or external function call.

**No ownership or lifetime management is shown; actual storage ownership and caller-provided lifetime guarantees are not established.** Either pointer could identify borrowed, shared, or otherwise owned storage. Absence of memory-management operations does not establish caller ownership.

There is no explicit error result, diagnostic, validation failure branch, or recovery path for invalid non-null storage. Null pointers receive the specified no-access treatment; insufficient capacity and invalid pointer operations are not checked.

No atomic-access or synchronization mechanism is shown. Concurrent mutation and memory-mapped device semantics remain outside the established successful-execution behavior.

## Dependencies

No function calls appear in the supplied body. There are no dependency function IDs to enumerate.

The operation relies on the execution environment’s semantics for:

- Unsigned 16-bit loads and stores.
- Unsigned 32-bit addition and comparison.
- Conversion from unsigned 32-bit to unsigned 16-bit values.
- Pointer arithmetic in 16-bit-element units.
- Ordering comparisons between destination positions.

The source establishes the arithmetic widths used here, but does not establish deployment-specific alignment rules, the complete pointer-validity model, or an external byte representation.

No assumptions about unknown callees preserving globals or managing memory are needed because no callees appear. This absence does not resolve the ownership or lifetime of either input region.

## Acceptance examples

These examples are conditional deductions from static evidence, not runtime test results. Unless overlap is specified, the regions are disjoint. Each example assumes stable initial counts and valid storage, pointer construction, comparisons, and terminal cursor positions.

### Ordinary append

Source count is 3, with payload 11, 22, 33. Destination count is 2, with payload 7, 8. The ceiling is 10.

T is 5 and N is 3. Transfers write 11, 22, and 33 to destination positions three, four, and five. Positions one and two remain unchanged. The final destination count is 5.

### Ceiling limits source consumption

Source count is 3, destination count is 2, and the ceiling is 4.

T is 4 and N is 2. Source positions one and two are read and written to destination positions three and four.

Source position three is not read as payload, although the terminating iteration advances the source cursor to that position. The final destination count is 4.

### Ceiling reduces the count without clearing payload

Source count is 3, destination count is 5, and the ceiling is 2.

T is 2. No payload is read or written. The destination count becomes 2, while old payload positions three through five retain their contents.

The destination cursor still advances to position six before the first stopping decision. Validity requirements include that cursor position and the target position two.

### Zero ceiling

Both pointers are non-null and valid. The ceiling is zero.

T is zero regardless of the count values. The function reads both counts, rereads the destination count, transfers no payload, and writes zero to the destination count.

This differs from a null-input return, which performs no count accesses or writes.

### Empty source

Source count is zero and destination count is 4.

With a ceiling of at least 4, no payload is transferred and the destination count is written as 4. With a ceiling of 2, no payload is transferred and the count is written as 2.

An empty source does not prevent count reduction or the final count write.

### Target exceeds count-field capacity

Source count is 1, destination count is 65,535, and the ceiling is at least 65,536.

T is 65,536. One source payload element is written to destination position 65,536. The final stored count is zero.

This requires writable storage at destination position 65,536, a valid terminal destination cursor at position 65,537, and valid source advancement to position two.

### Identical input pointers

Both pointers identify the same sequence, whose count is 2 and whose payload is 7, 8. The ceiling is at least 4.

T is 4. The function reads positions one and two and writes 7 and 8 to positions three and four. The final shared count is 4.

The result depends on sufficient storage and valid terminal cursor construction; identical pointers do not establish capacity.

### Overlap with self-copies

The source begins at destination position one. Initially:

- Destination position zero contains 1.
- Destination position one contains 3 and serves as the source count.
- Destination positions two, three, and four contain 10, 20, and 30.
- The ceiling is 4.

D is 1, S is 3, and T is 4. Every transfer reads and writes the same physical element.

Positions two through four remain 10, 20, and 30. The final destination count is 4. This layout does not propagate 10 into later elements.

### Overlap with forward propagation

The source again begins at destination position one. Initially:

- Destination position zero contains 2.
- Destination position one contains 3 and serves as the source count.
- Destination positions two, three, and four contain 10, 20, and 30.
- The ceiling is 5.

D is 2, S is 3, and T is 5.

The first transfer reads destination position two through the source view and writes 10 to position three. The second reads that newly written position three and writes 10 to position four. The third reads the newly written position four and writes 10 to position five.

Positions three through five end as 10, 10, and 10. Position two remains 10. The final destination count is 5.

This requires writable storage through destination position five and a valid terminal destination cursor at position six. The terminal source cursor reaches source position four, corresponding to destination position five.

### Null input

If either input pointer is null, no count or payload accesses occur through either pointer, and no storage is written. No status value is returned.

## Uncertainties

The following remain **documented_only**:

- The application-level meaning and encoding of payload elements.
- Whether the ceiling represents capacity, a logical-length limit, or another constraint.
- Actual allocation sizes and validity guarantees supplied by surrounding code.
- Actual storage ownership, including whether either region is borrowed or shared.
- Which party guarantees storage lifetime and how that guarantee is enforced.
- Whether callers prohibit overlap or targets above 65,535.
- Deployment-specific alignment and pointer-validity requirements.
- Outcomes under external mutation between count reads or during transfers.
- Whether final count truncation is intended or violates a broader application contract.

The counted-sequence append interpretation is inferred intent. No hidden capacity field, terminator convention, overflow-rejection policy, ownership transfer, lifetime extension, or application-level error protocol is established.

## Evidence

Established static evidence consists of the supplied body’s null guard, widened count addition, unsigned ceiling comparison, destination-count reread, forward cursor advancement, destination-position comparison, ordered element transfer, and final 16-bit count write.

The transfer count, maximum target, terminal cursor positions, count truncation, and conditional overlap outcomes are deductions from those operations under the stated preconditions.

The body contains no external calls or explicit memory-management operations. This establishes only their absence from this function, not ownership of either region or the surrounding lifetime arrangements.

No runtime tests have been performed. No caller implementation, allocation metadata, external application specification, or separate dependency implementation was supplied.
