<!-- Trusted provenance: F-33ade88db1b2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-33ade88db1b2

## Purpose

F-33ade88db1b2 repeatedly selects spans from cursor-addressed input, adds a fixed increment for each attempted span, and optionally updates a caller-provided signed maximum.

Span selection combines an unsigned candidate count from F-40859ed81aa5, sequential decoding through F-2be305f439e7, and adjacent-item classification through F-f684e1462ef9. Classification results exactly equal to 1 or 2 affect boundary selection.

**Inferred intent:** this resembles segmented layout or line breaking. The increment might represent height, and the optional measurement might represent width. Those meanings are not established by the supplied evidence.

The contract distinguishes ordinary cursor progression from the more general behavior possible when dependencies retain pointers to temporary storage and modify that storage during later calls in the same invocation.

## Interface

The function accepts seven arguments and returns a signed 32-bit integer.

| Position | Established role |
|---|---|
| 1 | Opaque 32-bit context passed to F-7ec5c59748e6 and F-40859ed81aa5. |
| 2 | Initial cursor represented by a 32-bit integer. |
| 3 | 32-bit value whose low 16 bits are interpreted as signed 16-bit and sign-extended to form the initial remaining count. |
| 4 | Opaque 32-bit selection parameter passed to F-40859ed81aa5. |
| 5 | Signed 32-bit adjustment added to the result of F-7ec5c59748e6. |
| 6 | No observable use in this function. |
| 7 | Optional pointer to writable signed 32-bit output storage. Null disables output initialization and optional measurement calls. |

Argument 3’s upper 16 bits do not affect the initial count. Low halves 0x7FFF, 0x8000, and 0xFFFF mean 32767, −32768, and −1 respectively.

No pointer-validity, alignment, input-length, or encoding checks are visible. The cursor’s numerical representation does not establish an input layout or decoding stride.

Three distinct temporary 32-bit storage locations have their addresses passed to dependencies:

- **Working-cursor slot:** passed to F-2be305f439e7 during scanning.
- **Boundary-cursor slot:** passed to F-2be305f439e7 when advancing a shortened selection.
- **Saved-position/measurement slot:** used first to save a scan position and later as the optional measurement output.

These locations persist through the invocation. Once exposed, they can be modified through retained pointers by subsequent dependency calls. The source does not enforce exclusive access or invalidate those pointers between calls.

Other values, including the span-start snapshot and predecessor-position value, are copied values whose addresses are not explicitly exposed by this function.

## Behavior

### Initialization and arithmetic

If the output pointer is non-null, the first caller-visible action is a signed 32-bit zero write through it.

The function establishes the working cursor and initial signed count, calls F-7ec5c59748e6 once with the context, and adds argument 5 to its result. This produces the fixed increment used throughout the invocation. The accumulator and pending-boundary flag start at zero.

F-7ec5c59748e6 is called even when the initial count is nonpositive.

No overflow check protects increment formation or accumulation. The representation uses signed 32-bit storage; signed-overflow cases are not given a portable arithmetic guarantee here.

### Entering a span

At each iteration, the remaining count is tested as signed.

If it is less than 1, the function adds one increment if the pending-boundary flag is set, then returns the accumulator.

Otherwise, it:

1. Clears the pending-boundary flag.
2. Adds one increment to the accumulator.
3. Calls F-40859ed81aa5 with the context, the current working-cursor value, the low 16 bits of the remaining count, the selection parameter, and a null final argument.
4. Reads the working-cursor slot after that call to establish an initial predecessor-position value.
5. Returns immediately if the candidate result is zero.

The candidate result is treated as unsigned 32-bit. A zero candidate does not undo the increment already added, and causes no decoding, classification, or optional measurement for that attempted span.

On later spans, the candidate-selection call can modify previously exposed temporary slots through retained pointers. In particular, the cursor passed into the call need not equal the working-cursor value read after it returns.

### Preparing and scanning a nonzero candidate

For a nonzero candidate, the function:

- Writes zero into the boundary-cursor slot.
- Copies the current working cursor into a separate span-start snapshot.
- Calls F-2be305f439e7 with the working-cursor address.
- Uses its returned 32-bit item value as the first item.
- Initializes the pair index to zero and the selected count to the original candidate count.

The span-start snapshot is taken after candidate selection and before initial decoding. Its address is not passed to a dependency. Subsequent valid writes through the three exposed temporary pointers do not change that snapshot.

By contrast, the zero written into the boundary slot is not guaranteed to survive decoding. A dependency can already hold that slot’s address from a previous span.

A pair is examined only while both conditions hold:

- The pair index is less than the unsigned candidate count.
- The pair index is less than the signed remaining count minus one.

For each pair, the function performs these observable operations in order:

1. Copies the current working cursor into the saved-position/measurement slot.
2. Calls F-2be305f439e7 with the working-cursor address to obtain the next item.
3. Calls F-f684e1462ef9 with the previous and next item values.
4. Applies the classification rules below.
5. Unless classification stopped the scan, increments the pair index, reads the saved-position/measurement slot into the predecessor-position value, and adopts the next item as the previous item.

The saved-position read in step 5 occurs **after both dependency calls**. On later spans, that slot may already have been exposed as a measurement output. A decoder or classifier can therefore replace its saved cursor before the function copies it into the predecessor position.

Consequently, “predecessor position” describes a role, not an unconditional guarantee that the value equals an earlier working cursor. That guarantee requires dependencies not to modify the saved-position slot between its write and subsequent read.

A remaining count of 1 permits the initial decode but no pair classification.

### Classification rules

Only exact results 1 and 2 receive special treatment.

**Result 1: immediate boundary**

The function sets the pending-boundary flag, selects the current pair index as the provisional count, writes the predecessor-position value into the boundary slot, and stops scanning.

This replaces any previously remembered boundary. The saved-position/measurement slot is not copied into the predecessor value on this terminating comparison.

**Result 2: remembered boundary**

If the original candidate count differs from the remaining count, the function selects the current pair index and writes the predecessor-position value into the boundary slot, then continues scanning.

Later eligible results of 2 replace earlier selections. If the original candidate equals the remaining count, result 2 does not select a boundary.

**Other results**

Scanning continues without a classification-directed boundary write.

Remembering a shorter count does not shorten the scan limit: the loop continues to use the original candidate count.

These rules specify the function’s own writes. Between such writes, dependency calls may independently modify an already exposed boundary slot. A classification result without a boundary write does not imply that the slot stayed unchanged.

### Finalizing the boundary

After scanning, the function reads the boundary-cursor slot. If its current value is zero, it replaces that value with the current predecessor-position value.

This is a zero-sentinel fallback, not proof that no classification selected a boundary. A selected zero cursor is indistinguishable from an unset value at this check. Conversely, a retained-pointer write can supply a nonzero boundary even when no classification selected one.

If the selected count is unsigned-less-than the original candidate count, the function increments it and calls F-2be305f439e7 with the boundary-cursor address. The returned item is discarded.

Both pair-index increments and this selected-count increment clear bit 16 of their 32-bit results. This is not truncation to 16 bits. Under the visible positive remaining-count bounds, pair-derived incremented values are below 32768, so clearing that bit has no effect.

The extra decoder call exposes the boundary slot, if it was not already exposed. Its resulting cursor is **not yet an unconditional next-span cursor**: optional measurement occurs before the function reads that slot to establish the next working cursor.

### Optional measurement and output update

When the output pointer is non-null, the function calls F-40859ed81aa5 with:

- The context.
- The saved span-start snapshot.
- The finalized selected count.
- The selection parameter.
- The address of the saved-position/measurement slot.

The dependency’s return value is ignored.

After the call, the function compares the caller’s output with the current contents of the temporary slot using signed 32-bit comparison. If the output is strictly smaller, it writes the slot’s value to the output. Equality causes no write.

Both values are observed after the dependency call. There is no protected pre-call maximum. The dependency may alter the caller’s output through aliases or other access.

The source does not verify that the measurement slot was written. Without a write, its value can be a saved scan position, a value left by an earlier call, another retained-pointer modification, or an indeterminate value if never initialized.

The measurement call can also modify the boundary slot through a pointer retained from an earlier boundary decode, including the extra decode immediately preceding this call.

### Advancing and repeating

After optional measurement and its conditional output write, the function:

- Reads the boundary-cursor slot and copies its **then-current** value into the working-cursor slot.
- Subtracts the selected count from the remaining count.
- Retains the low 16 bits and interprets them as signed 16-bit.
- Starts the next iteration.

Thus the next cursor is determined by the boundary slot’s post-measurement contents, not necessarily by the cursor produced by the extra decoder. If measurement is disabled, the read follows boundary finalization without that intervening call.

Count narrowing is not saturation. Oversized candidates are not rejected, and subtraction can narrow back to a positive count.

The pending flag is cleared whenever another positive-count iteration begins. In particular, an immediate boundary found within the visible scan bounds selects fewer items than the positive remaining count, so it does not by itself exhaust that count.

## Side effects

Established effects include:

- Initial zeroing of a non-null output.
- One fixed-increment dependency call.
- One candidate-selection call per entered positive-count span.
- Decoder and classifier calls in the stated order.
- Writes and dependency-visible access to the three temporary slots.
- An extra decoder call for a shortened selection.
- Optional measurement calls and conditional signed-maximum writes.

There is no explicit allocation, release, ownership transfer, or direct input write in this function. Dependency behavior is not thereby restricted.

The temporary pointers refer to invocation-owned storage. Retaining them does not transfer ownership. Later access during the invocation can affect the function as described above; access after the storage’s lifetime ends has no established safe contract.

Aliasing is unchecked. The initial output zero can change input or other dependency-visible state if storage overlaps. Dependencies need not preserve globals, output storage, or previously exposed temporary slots.

No explicit error status, rollback, cancellation, or iteration limit is present. A zero candidate is an ordinary return path, not an established error indication. Termination depends partly on candidate results and count narrowing.

## Dependencies

| Opaque ID | Observed use | Not established |
|---|---|---|
| F-7ec5c59748e6 | Receives context once; its 32-bit result contributes to the increment. | Units, errors, ownership, allocation, global effects. |
| F-40859ed81aa5 | Receives context, cursor, count, selection parameter, and either null or a temporary output address. Candidate result is unsigned; measurement-call return is ignored. | Candidate bounds, measurement initialization, null-mode semantics, pointer retention, other effects. |
| F-2be305f439e7 | Receives a mutable working- or boundary-cursor address and returns a 32-bit item. | Encoding, stride, input bounds, malformed-input behavior, pointer retention, other effects. |
| F-f684e1462ef9 | Receives two item values; its result is compared with 1 and 2. | Classification meanings, other return values, effects through previously retained pointers or shared state. |

A dependency need not receive a temporary pointer in its current call to modify that temporary through access retained or shared from an earlier call.

## Acceptance examples

These are conditional static deductions, not executed tests. Unless stated otherwise, assume valid storage, normal dependency returns, no arithmetic overflow, no disruptive aliasing, and no dependency modification of previously exposed temporary slots except through the output pointer supplied for its current operation. Let the fixed increment be **H**.

### Nonpositive initial count

If argument 3’s low half is 0x0000 or 0xFFFF, a supplied output is initialized to zero, F-7ec5c59748e6 is called, and the function returns zero without calling the other dependencies. Upper bits do not change this result.

### Zero candidate

With initial count 5 and first candidate zero, the function returns **H**, with no decoder, classifier, or measurement call. The output remains zero unless dependency side effects change it.

### One item

With remaining count 1 and candidate 1, there is one initial decode and no classification or extra boundary decode. The zero-boundary fallback uses the initial predecessor position, which equals the span-start cursor under the stated assumptions.

Optional measurement receives count 1 and the span-start cursor. The count becomes zero and the return is **H**. Cursor advancement past the item is not required for termination.

### Immediate and remembered boundaries

With remaining count 2, candidate 2, and result 1 on the first pair, the selected count becomes 1 after finalization and an extra boundary decode occurs. The next remaining count is 1, so the pending flag is cleared on the next positive-count iteration.

With remaining count 6, candidate 4, result 2 at pair indices 0 and 2, and no result 1, the later boundary replaces the earlier one. Scanning continues through the candidate-limited comparisons. The finalized count is 3, an extra decode advances the chosen boundary, and the next remaining count is 3.

### Measurement changes a retained boundary slot

Assume remaining count 3, candidate 2, and result 1 on the first pair. Finalization selects count 1 and passes the boundary-slot address to the extra decoder.

If that decoder leaves cursor **P**, but the following measurement call changes the same slot through a retained pointer to **Q**, the next working cursor is **Q**, not **P**. The next remaining count is 2. This effect requires no out-of-lifetime access.

### Classification changes a previously exposed saved-position slot

On a later span, assume the saved-position/measurement slot’s address was exposed by an earlier measurement. Let remaining count be 3 and candidate be 1.

During the sole pair comparison, the function saves cursor **B** into that slot. Suppose the decoder runs, then the classifier changes the slot to **Q** through retained access and returns a value other than 1 or 2.

The post-call predecessor read obtains **Q**, not **B**. If the boundary slot remains zero, fallback selects **Q**. No extra decode occurs because selected count equals candidate count. If measurement leaves the boundary unchanged, the next working cursor is **Q**.

### Signed maximum and oversized counts

If measurements write −3, 7, and 4 without other output changes, the zero-initialized output remains zero, becomes 7, then remains 7.

With remaining count 1 and candidate 65536, only the initial decode occurs. The selected count remains 65536, and the signed low-16-bit interpretation of \(1-65536\) is 1. Repetition rather than termination is therefore possible.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings and units of arguments, items, classifications, increments, and measurements.
- Input layout, cursor encoding, valid cursor range, and whether zero is valid.
- Candidate bounds and dependency guarantees of progress.
- Whether measurement always initializes its output.
- Whether dependencies retain, share, or modify temporary pointers during the invocation.
- Dependency error conventions, ownership actions, allocation, and global effects.
- Behavior after invalid access, malformed input, or signed overflow.
- A maximum-output guarantee when dependencies modify output storage or fail to supply a defined measurement.

Snapshot-like scan positions and stable finalized boundaries are conditional on the absence of intervening retained-pointer writes. The unexposed span-start copy remains distinct from these live temporary slots.

## Evidence

This contract rests solely on static inspection of the supplied function representation and opaque dependency mapping.

Established evidence includes argument widths, signed count narrowing, call order, explicitly exposed temporary addresses, classification-directed writes, zero-boundary fallback, post-call saved-position reads, signed output comparison, and the post-measurement read that establishes the next working cursor.

Retained-pointer mutations are permitted possibilities, not observed dependency behavior. No dependency implementations, runtime tests, dynamic traces, or ownership checks were available. The acceptance examples state deductions under explicit conditions, not execution results.
