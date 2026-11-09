<!-- Trusted provenance: F-b4b5aa4c2f83; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b4b5aa4c2f83

## Purpose

F-b4b5aa4c2f83 rearranges a contiguous sequence of 32-bit words in place according to a caller-supplied comparison callback and a 32-bit context value.

The established implementation combines partitioning for ranges of at least 17 elements with insertion sorting for smaller ranges. It manages pending ranges in fixed-size local storage rather than recursively invoking itself.

Under a consistent comparison relation and the memory and callback preconditions below, the intended result is an ascending sequence containing the same word values as the input. Stability is not guaranteed.

This is a static behavioral contract. No runtime tests have been performed. Sorting guarantees are conditional on callback behavior; the function itself does not enforce a valid ordering relation or protect its storage from callback interference.

## Interface

### Inputs

The function receives four inputs:

| Input | Width and interpretation |
|---|---|
| Sequence address | A 32-bit address identifying the first word of the sequence. The supplied representation carries this address through a 32-bit integer. |
| Element count | An unsigned 32-bit count of elements, not bytes. |
| Comparison callback | An indirect callable address. Its exact calling convention is not established here. |
| Context | An opaque 32-bit value passed to each comparison call. |

Each sequence element occupies exactly four bytes. The function treats element contents as opaque 32-bit words: it copies and exchanges their bit patterns without interpreting their numerical sign or dereferencing them.

The callback receives three arguments, in this order:

1. A 32-bit element value.
2. Another 32-bit element value.
3. The supplied 32-bit context.

These are element **values**, not addresses of the corresponding sequence slots. The context is forwarded as a word, not dereferenced by this function.

The callback result is interpreted as a signed 32-bit integer. Only the distinction between strictly positive and nonpositive is used. Negative results and zero take the same branch at each comparison site.

### Output

There is no defined return value or status result. The observable output is the sequence’s modified contents, together with any effects caused by callback invocations.

No allocation, ownership-transfer result, or returned sequence address is present.

### Preconditions for defined sorting behavior

For a count greater than one:

- The address must identify a suitably aligned, readable and writable contiguous region containing the requested number of four-byte elements.
- Address formation and address differences must retain their intended meaning in the represented 32-bit address model.
- The callback must be callable with the described arguments and must return normally for completion to be guaranteed.
- The callback must implement a consistent ordering relation if a sorted result is required.
- The callback must not mutate sequence storage or invalidate its lifetime if the normal sorting guarantees are required.
- Concurrent access must not interfere with the sequence or the function’s local state.

The function performs no explicit validation of these conditions.

Address arithmetic is materially narrower than a general host-size array abstraction. The final element address is formed using a four-byte scale and a subtraction of four. The count-dependent arithmetic has 32-bit unsigned participation, and range lengths are subsequently derived from signed 32-bit address differences. Consequently, a nonwrapping allocation alone is not sufficient to justify every possible 32-bit count. A conservative supported domain requires nonwrapping address formation and nonnegative endpoint byte differences representable as signed 32-bit integers.

For counts zero and one, no sequence access or callback invocation occurs. Thus the no-op path does not require a dereferenceable sequence address or callable callback.

## Behavior

### 1. Entry and empty or singleton sequences

The first behavioral decision uses the unsigned element count.

If the count is zero or one, the function returns without reading or writing sequence elements and without calling the callback.

For larger counts, it establishes an inclusive range from the first element to the element at index `count − 1`. That range is initially the only pending range.

### 2. Range processing and scheduling

Pending work is represented by paired inclusive lower and upper bounds. The local storage has capacity for 64 such pairs.

The function removes the most recently pending range and processes it. A range whose lower bound is not below its upper bound requires no element operations.

For an active range, the endpoint byte difference is interpreted as a signed 32-bit quantity and shifted right by two to obtain the number of element intervals. Under the valid-address preconditions, this equals the number of elements minus one.

The processing choice is:

- At most 15 intervals, or at most 16 elements: insertion sorting.
- More than 15 intervals, or at least 17 elements: partitioning.

After partitioning, the smaller side is processed immediately and the larger side is deferred. A deferred side is recorded only if it contains at least two elements.

If the two sides have equal lengths, the right side is deferred and the left side is processed immediately.

The pivot’s final position is excluded from both subsequent ranges.

This scheduling reduces pending-range growth for valid range arithmetic. There is no explicit overflow check on the 64-pair storage.

### 3. Insertion sorting of short ranges

The function visits candidate elements in increasing address order, beginning with the second element of the active range.

For each candidate:

1. It reads and retains that candidate’s 32-bit value.
2. It examines preceding elements, moving toward the range’s lower bound.
3. For each eligible predecessor, it calls the callback with the predecessor’s current value first, the retained candidate value second, and the context third.
4. If the callback result is strictly positive, it copies that predecessor one slot to the right and continues backward.
5. At the lower bound, or at the first nonpositive comparison result, it writes the retained candidate into the resulting destination slot.

The lower-bound check prevents comparing a predecessor outside the active range.

A candidate is written back even if it does not move. Accordingly, an already ordered short range still incurs stores.

A comparison result of zero does not cause the candidate to move past the predecessor. This gives insertion sorting the usual equal-element preservation within that short-range operation, provided the callback does not interfere with storage. It does not make the complete function stable.

The predecessor used for a rightward copy is read again after the callback returns. It is not necessarily the same word value that was passed into that callback if the callback has modified the sequence.

### 4. Pivot placement for long ranges

For a range of at least 17 elements, the function first exchanges:

- The first element.
- The element at index `floor((range length − 1) / 2)` relative to that first element.

This selects the central element for odd lengths and the lower of the two central elements for even lengths. No comparison is used for this selection, and no median-of-three selection is present.

The first slot then serves as the pivot location during scanning. Comparisons read the word currently stored there; the pivot is not held in a separate immutable value throughout partitioning.

### 5. Partition scans and exchanges

The partition maintains a forward scan and a backward scan.

The forward scan begins just after the pivot slot. It continues while it remains strictly before the backward scan and the callback reports:

- First argument: the current pivot-slot value.
- Second argument: the current forward-scan value.
- Result: strictly positive.

Under ordinary comparison semantics, this advances over values smaller than the pivot.

The forward scan does not compare the slot where its address meets or passes the backward scan.

The backward scan continues while the forward position is at or below the backward position and the callback reports:

- First argument: the current backward-scan value.
- Second argument: the current pivot-slot value.
- Result: strictly positive.

Under ordinary comparison semantics, this retreats over values greater than the pivot.

If the scans stop with the forward position strictly before the backward position, their two elements are exchanged. The backward position then retreats by one element, and forward scanning resumes by advancing one element.

If the scans meet or cross, that exchange is not performed. Instead, the pivot slot is exchanged with the final backward-scan slot. That slot becomes the pivot’s final location for this partition.

For a consistent ordering relation and noninterfering callback, this leaves no greater-than-pivot value on the left and no less-than-pivot value on the right. Equal values may occur on either side and may be exchanged.

### 6. Completion

Partitioning and short-range insertion sorting continue until the active range and all deferred ranges have been exhausted.

With normal callback returns and valid memory and arithmetic, the function returns after completing this work.

There is no success flag, callback-error translation, rollback, or explicit postcondition check.

## Side effects

### Sequence access and write order

All direct sequence accesses are four-byte word operations.

Each exchange has the following observable order:

1. Read the first slot into temporary storage.
2. Read the second slot.
3. Write the second slot’s value into the first slot.
4. Write the saved first-slot value into the second slot.

This applies to pivot selection, exchanges between scan positions, and final pivot placement. A final pivot exchange can involve the same slot on both sides; no identity check suppresses it.

During insertion sorting, each successful greater-than comparison is followed by a fresh predecessor read and a write into the slot to its right. The retained candidate is written only after those shifts finish. A callback invoked during this process can therefore observe an intermediate sequence containing duplicated shifted values and not yet containing the retained candidate in its eventual slot.

The ordering of the two element reads used to assemble a callback’s arguments is not established by the supplied representation. The callback argument positions and their relation to subsequent writes are established.

### Callback-visible state

Comparisons are interleaved with in-place mutations. The callback does not receive a snapshot of the original sequence.

During partitioning, it may observe:

- The initial central-element exchange.
- Earlier scan-position exchanges.
- A pivot that has not yet been moved to its final partition position.

During insertion sorting, it may observe earlier shifts and completed candidate insertions.

The context word is forwarded unchanged by this function. No preservation guarantee is made for objects that the context might identify.

### Ownership and external state

The function itself contains no visible allocation or release operation and no direct transfer of sequence ownership. It uses local temporary storage for pending ranges and individual word values.

The callback’s ownership behavior is unknown. It must not be assumed to preserve global state, retain or release memory, or avoid external side effects.

If the callback releases or invalidates sequence storage, subsequent accesses are not protected. If it mutates sequence contents, the visible access mechanics still apply, but permutation and ordering guarantees no longer follow.

No locking, explicit global-state access, file operation, or logging operation is visible in the function body. Callback effects may nevertheless include any of these.

## Dependencies

### F-callback

F-callback is an opaque label for the runtime-supplied comparison dependency. No fixed target identity is established.

Its observed interface consists of two 32-bit element values and the 32-bit context, returning a result interpreted as signed 32-bit.

For conventional ascending sorting, its result should be positive exactly when its first value belongs after its second value. The magnitude of the result is irrelevant.

The function does not require a distinct negative branch: zero and negative results are both nonpositive. Nevertheless, a coherent relation is needed for the sorting postcondition. Inconsistent, state-dependent, or storage-mutating comparison behavior is not corrected or rejected.

No other direct function dependency is visible.

## Acceptance examples

These examples are conditional static expectations, not executed tests.

### Zero elements

Given a count of zero, an arbitrary sequence address, and an unusable callback address:

- No sequence read or write occurs.
- No callback invocation occurs.
- The function returns without reporting a result.

### One element

Given a count of one:

- The sole element is neither read nor rewritten by this function.
- The callback is not invoked.
- The element’s existing bits remain unchanged, absent unrelated interference.

### Two elements requiring exchange

Given the sequence `[9, 3]` and a callback implementing ordinary ascending integer order:

- The short-range path retains `3`.
- The callback receives `9`, `3`, and the context.
- A positive result causes `9` to be copied into the second slot.
- The retained `3` is written into the first slot.
- The final sequence is `[3, 9]`.

This example assumes the callback does not mutate the sequence.

### Two already ordered elements

Given `[3, 9]` under the same comparison relation:

- One comparison receives `3` and `9`.
- Its nonpositive result prevents a shift.
- The second slot is still written with the retained `9`.
- The final values remain `[3, 9]`.

Thus unchanged final contents do not imply an absence of writes.

### Threshold boundary

For a 16-element sequence, the initial range uses insertion sorting without an initial pivot-selection exchange.

For a 17-element sequence, the initial range uses partitioning and first exchanges index 0 with index 8, before its first comparison.

For an 18-element sequence, that initial exchange also uses index 8.

These expectations assume valid ordinary address arithmetic.

### Equal elements

If every comparison returns zero:

- Insertion sorting performs no rightward shifts, but still writes each retained candidate.
- Long-range partitioning still performs its pivot-selection exchange and may exchange scan-position elements.
- Completion does not imply preservation of the original relative order of logically distinct equal elements.

### Callback mutation

Suppose an insertion comparison returns positive after changing the predecessor slot:

- The subsequent shift reads that slot again.
- The shifted value is the post-callback value, not necessarily the value supplied as the first comparison argument.

This is an access-order example, not a supported sorting guarantee for mutating callbacks.

### Invalid large count

A count whose four-byte scaling wraps, or whose endpoint difference does not have the intended signed 32-bit interpretation, is outside the defined sorting preconditions.

No particular exception, error return, bounds rejection, or partial-result shape is promised.

## Uncertainties

The following are **documented_only**:

- The exact calling convention and callback-address representation.
- Whether element words encode integers, addresses, handles, or another application type.
- The application meaning and ownership implications of the context.
- Whether a particular callback preserves globals, sequence storage, or memory lifetimes.
- Behavior after callback nonlocal exit, fault, or abnormal termination.
- Behavior under misalignment, invalid addresses, address wraparound, or concurrent mutation.
- The precise fault behavior for inaccessible memory.
- The relative order of evaluating the two element reads used as callback arguments.
- Platform byte order. The function copies whole words and does not explicitly inspect their bytes.

Ascending sorting and permutation preservation are inferred semantic consequences under the stated ordering and noninterference assumptions. They are not unconditional guarantees for arbitrary callbacks.

The fixed pending-range capacity is established. Its adequacy follows from smaller-side-first scheduling within the valid arithmetic domain; there is no dynamic capacity check.

## Evidence

The supplied static function representation establishes:

- The unsigned count gate that excludes counts zero and one.
- Four-byte element accesses and addressing.
- Inclusive range endpoints.
- Two fixed local arrays of 64 endpoints.
- The 16-element insertion-sort cutoff.
- Central-element pivot selection by exchange.
- The callback argument orientation at each comparison site.
- Strictly positive signed comparison tests.
- In-place scan exchanges and insertion shifts.
- Re-reading predecessors after successful insertion comparisons.
- Smaller-side-first scheduling, including its equal-size tie rule.
- The absence of an explicit result value, validation path, allocation, or release call.

The interpretation as a hybrid in-place ascending sort is inferred from those operations and depends on the stated callback contract.

No runtime tests, observed executions, or empirical performance measurements have been performed.
