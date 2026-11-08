<!-- Trusted provenance: F-a3d9c22ead5c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a3d9c22ead5c

## Purpose

F-a3d9c22ead5c conditionally inserts a requested number of 20-byte records at a supplied position in a contiguous-storage descriptor.

The supplied body establishes three paths:

- A zero-count request returns without accessing the descriptor or invoking a dependency.
- A nonzero request that fails an unsigned capacity-like comparison obtains replacement storage, performs three range-processing operations, invokes retirement-related dependencies, and replaces the descriptor’s three visible fields.
- A nonzero request that passes that comparison enlarges the descriptor’s logical end before processing records within the existing storage.

“Insertion,” “capacity,” “construction,” and “retirement” are inferred roles, not independently established dependency semantics. The contract below preserves the actual arguments, arithmetic, calls, and state-access order without assuming that opaque dependencies implement conventional container operations.

## Interface

The function accepts four arguments and has no explicit return value.

| Argument | Visible representation | Role |
|---|---|---|
| Descriptor | Pointer to storage state containing at least three consecutive 32-bit fields | Read and modified by the function; also passed to dependencies |
| Position | 32-bit address-like value | Insertion boundary and source-range boundary |
| Count | Unsigned 32-bit integer | Requested number of records |
| Value operand | Unsigned 32-bit value | Forwarded unchanged to population-related dependencies |

The visible descriptor layout is:

| Byte offset | Width | Inferred meaning |
|---|---:|---|
| 0 | 4 bytes | Storage beginning |
| 4 | 4 bytes | Logical end |
| 8 | 4 bytes | Storage limit |

This layout does not establish the descriptor’s complete size. Dependencies receiving its address may access additional state.

The record stride is 20 bytes. In the directly implemented backward movement, the function transfers a 4-byte field at record offset 4 and passes subobject addresses at record offset 8 to F-eb035a816079. That movement contains no direct transfer of the field at record offset 0. Other effects on record bytes depend on the opaque calls.

### Preconditions

For conventional contiguous-container behavior, the caller needs a readable and writable descriptor, valid backing storage, a position on a 20-byte record boundary between beginning and logical end, and representable interval and allocation arithmetic. The value operand must satisfy the invoked dependencies’ validity, lifetime, and aliasing requirements.

These conditions are not explicitly validated. The body supplies no bounds rejection, alignment rejection, overflow rejection, or invalid-pointer handling.

Count zero bypasses descriptor access within this function. No descriptor-validity requirement is exercised on that path.

## Behavior

### Arithmetic and dependency results

Visible scalar fields, count arithmetic, address-like calculations, and dependency results occupy 32 bits. Unsigned comparisons and logical right shifts are explicit where described below. Overflowing expressions must not be treated as unbounded mathematical calculations or as valid container geometry.

F-870c4fcf7295 receives a byte difference and the constant 20. Its apparent purpose is conversion from bytes to records, but its exact operation, signedness, rounding, and exceptional-input behavior are **documented_only**.

F-193486da17af receives the descriptor and supplies size-like results. Their meaning and consistency across calls are likewise unestablished.

Consequently, a zero byte difference does not, from this body alone, establish that F-870c4fcf7295 returns zero. A difference of count multiplied by 20 does not establish that it returns count. Any such conclusion below is explicitly conditional on exact record-count conversion.

Dependencies may modify descriptor fields or other shared state. Values captured before a call remain the function’s retained operands; fields read afterward are not assumed unchanged.

### Zero-count behavior

Count zero causes immediate return. This path performs no descriptor reads or writes, dependency calls, allocation-like calls, or record accesses.

### Initial storage decision

For nonzero count, the first dependency call is F-193486da17af with the descriptor. After that call, the function reads storage beginning and storage limit and submits their difference, together with 20, to F-870c4fcf7295.

The conversion result is compared unsigned against the first size-like result plus count. A strictly smaller conversion result selects replacement storage. Equality selects existing storage.

This decision uses the descriptor state after the first size-like call, not necessarily its entry state.

### Replacement storage

#### Selected capacity and allocation-like request

Two further calls to F-193486da17af supply separate values:

- The first result plus count forms the required-size candidate.
- The second result supplies both growth candidates.

For the second result, the growth candidates are that result plus 32, and that result plus its logical right shift by one plus its logical right shift by three.

Unsigned comparisons select the largest of the required-size candidate and the two growth candidates. The candidates must be considered as their evaluated 32-bit values; mathematical growth guarantees do not survive overflow.

Under stable size queries and nonoverflowing arithmetic, the selected capacity is therefore the maximum of:

- Size plus count.
- Size plus 32.
- Size plus floor(size divided by 2) plus floor(size divided by 8).

The two size calls are observably distinct and need not return the same value.

F-871e64ddc6de receives selected capacity multiplied by 20 and a zero second argument. Its result is retained as the replacement beginning. No explicit null-result or other failure check appears before subsequent use.

#### Range processing

The replacement path performs the following operations in observable order:

1. F-fca3c559c813 receives the descriptor’s then-current beginning, position, replacement beginning, and descriptor.
2. F-870c4fcf7295 receives position minus a newly read descriptor beginning, together with 20.
3. F-28dd59aac057 receives replacement beginning plus that result multiplied by 20, the requested count, the unchanged value operand, and descriptor.
4. F-870c4fcf7295 again receives position minus another newly read descriptor beginning, together with 20.
5. F-fca3c559c813 receives position, the descriptor’s then-current logical end, replacement beginning plus the sum of the second offset result and count multiplied by 20, and descriptor.

The inferred roles are prefix construction, inserted-record construction, and suffix construction. Copying, moving, source consumption, overlap support, and lifetime effects are not established.

The two offset conversions cannot be merged behaviorally: the population call between them may affect shared state. The suffix’s logical-end argument is also obtained after that call and the second conversion.

#### Retirement and descriptor replacement

After suffix processing, another F-193486da17af call supplies a retained size-like result for the eventual logical end.

F-0bd534eb13bf then receives the descriptor and its then-current beginning and logical end. After that call, the function captures the descriptor’s beginning, reads its storage limit, and passes their difference with 20 to F-870c4fcf7295. F-ff1bdc376dcd receives the captured beginning, that conversion result, and zero.

Only afterward does this function install the replacement descriptor fields, in this order:

1. Beginning becomes the retained replacement beginning.
2. Logical end becomes replacement beginning plus the retained size-like result multiplied by 20 plus count multiplied by 20.
3. Storage limit becomes replacement beginning plus selected capacity multiplied by 20.

There is no intervening dependency call between these three writes.

The retirement operands are not necessarily the original storage interval. The cleanup-like call uses fields read after preceding dependencies; the release-like call uses a beginning captured after cleanup. A conversion dependency may also affect shared state, but it does not change the already captured beginning passed to F-ff1bdc376dcd.

### Existing storage

The function captures the logical end and calculates position plus count multiplied by 20. It immediately writes the descriptor’s logical end as captured end plus count multiplied by 20.

This write precedes every record-processing dependency on this path. A dependency may therefore observe an expanded logical interval before the newly exposed records have been processed.

An unsigned comparison between the calculated insertion-span endpoint and the captured end selects one of two branches.

#### Insertion-span endpoint strictly before captured end

The function establishes a source boundary at captured end minus count multiplied by 20. F-fca3c559c813 receives that boundary, captured end, captured end as destination, and descriptor.

Under conventional range semantics, this processes the last count existing records into the newly exposed tail.

Backward movement then proceeds until its descending source boundary equals position. Each iteration first reads the 4-byte quantity 16 bytes before the source boundary and writes it 16 bytes before the destination boundary. It next calls F-eb035a816079 with the addresses 12 bytes before the destination and source boundaries, in that order. Both boundaries then descend by 20 bytes for the next iteration.

Thus, each iteration directly transfers record offset 4 before invoking delegated processing at record offset 8. Higher-addressed source records are processed before lower-addressed ones.

Finally, F-798138a35e09 receives position, the previously calculated insertion-span endpoint, and the value operand.

The loop terminates by exact equality, not by a bounds or ordering check. Misaligned or otherwise malformed positions are not protected by a visible validation step.

#### Insertion-span endpoint at or after captured end

This branch includes exact address equality.

F-870c4fcf7295 receives captured end minus position and the constant 20. Let its returned 32-bit quantity be the tail-conversion result. The population count supplied next is the requested count minus that result, as a 32-bit unsigned quantity.

The subsequent call order is:

- F-28dd59aac057 receives captured end, that population count, the value operand, and descriptor.
- F-fca3c559c813 receives position, captured end, the previously calculated insertion-span endpoint, and descriptor.
- F-798138a35e09 receives position, captured end, and the value operand.

The first call is not suppressed when its calculated count is zero.

Address equality between the insertion-span endpoint and captured end establishes branch selection only. It establishes a zero population count **only if** the conversion result equals the requested count. Exact conversion of a nonoverflowing, record-aligned byte difference would satisfy that condition; the supplied body does not establish that conversion behavior.

Likewise, position equal to captured end supplies a zero byte difference to the conversion dependency, but the population call receives the full requested count **only if** that dependency returns zero.

Neither existing-storage branch performs a later direct descriptor-end write. If a dependency changes the descriptor after the initial expansion, this function does not reassert the expanded end.

## Side effects

Established direct effects include descriptor-field reads and writes, the backward loop’s 4-byte record-field transfers, and dependency invocations with storage, range, subobject, and descriptor operands.

Replacement storage involves an allocation-like call and cleanup- and release-like calls. Existing storage contains no direct call to those particular allocation or retirement dependencies, although other dependencies may manage resources internally.

Ownership is **documented_only**. The ordering suggests replacing descriptor-owned storage after retirement of the previous storage, but the body does not establish whether:

- Range operations copy, move, or consume their sources.
- Cleanup destroys records.
- Release deallocates memory.
- Record fields own external resources.
- The value operand can alias records being modified or retired.

No visible status return, rollback, allocation-failure branch, or overflow rejection is provided. Dependency failures, exceptions, nonlocal exits, and memory faults have unknown contracts. An interrupted existing-storage operation may leave the logical end already enlarged.

No dependency is assumed to preserve globals or descriptor fields merely because those objects are absent from its explicit argument list.

## Dependencies

| Opaque ID | Visible use | Unestablished contract |
|---|---|---|
| F-193486da17af | Descriptor to size-like result | Meaning, consistency, state mutation |
| F-870c4fcf7295 | Byte difference and 20 to count-like result | Exact conversion, rounding, signedness, zero-input behavior |
| F-871e64ddc6de | Byte quantity and zero to address-like result | Allocation, alignment, failure, ownership |
| F-fca3c559c813 | Two source boundaries, destination, descriptor | Range interpretation, construction, overlap, source lifetime |
| F-28dd59aac057 | Destination, count, value operand, descriptor | Population semantics, zero-count behavior, aliasing |
| F-798138a35e09 | Two boundaries and value operand | Replacement semantics, range inclusivity, aliasing |
| F-eb035a816079 | Destination and source subobject addresses | Assignment or transfer semantics, affected extent |
| F-0bd534eb13bf | Descriptor and two boundaries | Cleanup, destruction, state mutation |
| F-ff1bdc376dcd | Captured storage address, count-like result, zero | Release semantics, units, ownership |

The inter-call ordering described above is established. Evaluation order among independent argument expressions within an individual call is not established.

## Acceptance examples

These are conditional acceptance examples, not executed tests. Conventional insertion outcomes require valid storage and position, nonoverflowing arithmetic, stable size-like results, exact byte-to-record conversion, and dependencies implementing the inferred record and lifecycle roles. Where only call-level behavior is claimed, the stated dependency results determine that behavior without implying those roles.

### Zero request

With count zero, acceptance requires return without descriptor access or dependency calls. Position and value operand do not affect that path.

### Capacity exactly fits

If the first size-like result is 5, the initial capacity-conversion result is 8, and count is 3, existing storage is selected. Equality must not select replacement storage.

The direct logical-end write adds 60 bytes before record-processing calls.

### Minimum growth dominates

With stable size-like results of 10, an initial capacity-conversion result of 10, and count 1, replacement storage is selected.

The required-size candidate is 11. The growth candidates are 42 and 16. Selected capacity is 42, so F-871e64ddc6de receives 840 and zero. If the retained final size-like result is also 10, the installed logical end is 220 bytes beyond replacement beginning.

### Requested size dominates

With stable size-like results of 10, initial capacity-conversion result 10, and count 50, selected capacity is 60. The allocation-like request is 1,200 bytes.

### Backward movement

Given eight records, position at index 2, count 2, and sufficient capacity under the conventional assumptions:

- Logical end advances from index 8 to index 10 before record processing.
- The first range operation processes indices 6 and 7 toward indices 8 and 9.
- Direct backward movement processes source indices 5, 4, 3, and 2 toward destination indices 7, 6, 5, and 4.
- The final population-related range has boundaries at indices 2 and 4.

The descending movement order and field-transfer-before-subobject-call order are observable requirements, not merely one way to obtain the final arrangement.

### Insertion span equals old tail length

Given eight records, position at index 6, count 2, sufficient capacity, and nonoverflowing address calculations, the insertion-span endpoint equals the captured end. The at-or-after branch is selected.

The conversion dependency receives a byte difference of 40 and the constant 20. **If it returns the exact record count, 2**, F-28dd59aac057 receives count zero. That call still precedes the tail-processing and final replacement calls.

Without that explicit conversion assumption, its count is the 32-bit unsigned difference between 2 and the actual conversion result. For example, a result of 1 produces a population count of 1 despite the address equality.

### Insertion at the end

Given position equal to captured end, positive count, sufficient capacity, and nonoverflowing address calculations, the conversion dependency receives zero and 20.

**If exact conversion returns zero for that input**, F-28dd59aac057 receives the full requested count. Otherwise it receives requested count minus the actual conversion result.

Independently of that result, F-fca3c559c813 receives identical source boundaries, and F-798138a35e09 receives identical range boundaries. Those calls remain present even when their ranges are empty under the conventional interpretation.

## Uncertainties

The following remain **documented_only**:

- Exact ABI details and the complete descriptor and record layouts.
- Dependency meanings, state preservation, and error behavior.
- Exact conversion, including the result for zero bytes or an exact multiple of 20.
- Record construction, replacement, destruction, and ownership rules.
- Overlap and value-operand aliasing support.
- Allocation failure and recovery after partial progress.
- Behavior for malformed descriptors, invalid positions, and arithmetic overflow.

Growth candidates, allocation byte products, count subtraction, and address calculations are bounded by the visible 32-bit representation. No general meaningful insertion behavior is established when those calculations overflow or when a dependency result disagrees with the inferred record geometry.

## Evidence

This contract uses static inspection of the supplied body and opaque dependency mapping.

Established evidence includes the zero-count guard, visible field widths, 20-byte stride, unsigned branch comparisons, separate size and conversion calls, growth expressions, call ordering, descriptor-access timing, direct transfer offsets, and final descriptor-write order.

Container terminology, half-open range interpretation, and lifecycle roles are inferred intent. Dependency internals, exact conversion, ownership, global-state preservation, and failure guarantees are not established.

No runtime tests have been performed.
