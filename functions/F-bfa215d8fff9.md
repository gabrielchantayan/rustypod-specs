<!-- Trusted provenance: F-bfa215d8fff9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bfa215d8fff9

## Purpose

F-bfa215d8fff9 conditionally obtains an output region containing two-component, 32-bit accumulators and applies weighted adjustments for one selected record.

Established behavior includes:

- Returning status 6 when processing is disabled or the descriptor is absent.
- Allocating and publishing the output before checking record availability.
- Retaining that output on successful paths, including an out-of-range record or an initially empty record interval.
- Processing entries whose flags select vector construction, optional point selection, and adjustment decoding.
- Adding dependency-computed contributions to existing output components without explicitly initializing those components.
- Releasing and clearing the output on explicit processing errors after successful allocation.

Weighted two-dimensional adjustment accumulation is inferred intent. The data format, numerical units, allocation initialization, and intended handling of some selection paths remain unestablished.

## Interface

### Arguments and return value

The function accepts four arguments:

| Argument | Representation and use |
|---|---|
| Context address | A 32-bit address used to obtain the stream-like object, enable flag, and descriptor. |
| Record index | Unsigned 32-bit index. |
| Output slot address | Address of a four-byte slot receiving the output address after successful allocation. |
| Output element count | Unsigned 32-bit count supplied to output allocation and used by all-elements accumulation. |

The return value is a 32-bit integer status:

| Status | Established meaning |
|---|---|
| 0 | The executed path leaves no explicit error. Adjustments need not have been applied. |
| 6 | The enable byte is zero or the descriptor address is zero. |
| 8 | An entry selects a stored vector outside the descriptor’s vector count. |
| Other nonzero value | Reported by the allocation dependency or either stream-preparation dependency. |

No explicit status is checked from selection decoding, adjustment decoding, weight computation, scaling, release, or finalization.

### Referenced storage

Addresses and integer words in the supplied representation are four bytes wide.

| Object | Byte offset | Meaning used here |
|---|---:|---|
| Context | 0x68 | Stream-like object address |
| Context | 0x29c | One-byte enable flag |
| Context | 0x2a0 | Descriptor address |
| Stream-like object | 0x00 | Stream base address |
| Stream-like object | 0x1c | Allocation/release handle |
| Stream-like object | 0x20 | Current cursor |
| Descriptor | 0x00 | Unsigned 32-bit axis count |
| Descriptor | 0x18 | Stored-vector count |
| Descriptor | 0x1c | Stored-vector storage address |
| Descriptor | 0x20 | Record count |
| Descriptor | 0x24 | Record-offset table address |

A record uses two adjacent four-byte offset entries. Stored vectors contain four-byte values, with their stride calculated from the descriptor’s axis count.

Each output element occupies eight bytes: a four-byte first component followed by a four-byte second component. Adjustments are read as signed 16-bit values. Indexed point destinations are read as unsigned 16-bit values.

### Preconditions and ownership boundary

Every address reached by the selected path must permit the corresponding access. This includes initial nested reads even when the feature is disabled.

The caller and dependencies must collectively provide enough storage for actual accesses, not merely for the counts initially observed. In particular:

- An in-range record requires two accessible offset entries at each time those entries are read.
- Scratch capacity must cover writes made under subsequently observed axis counts.
- Adjustment arrays must cover the accumulation loop actually selected.
- Indexed destinations must lie within usable output storage.

The function performs no local bounds check on decoded point indices and no explicit output initialization.

Retained output is apparently intended for the caller, but ownership transfer and release obligations are not established by an allocation contract.

## Behavior

### Entry reads and feature gate

The function first obtains the stream-like object address and reads its allocation/release handle. It also obtains the descriptor address before testing the enable flag and descriptor validity.

A zero enable byte or zero descriptor address produces status 6 without an explicit output-slot write or allocation request. This gate does not protect the preceding stream-object and handle reads.

The captured stream-object address, descriptor address, and handle are subsequently used. The context’s pointer fields are not refreshed to obtain replacement objects.

### Output publication and record interval

When enabled with a descriptor, the function calls F-6dd003795db7 with the captured handle, size argument 8, two zero arguments surrounding the output element count, and a status-output address.

A nonzero reported allocation status ends processing without an explicit output-slot write or local release of the returned address. A zero status causes the returned address to be published in the output slot. There is no separate nonzero-address check.

After publication:

- A record index at least equal to the descriptor’s current record count succeeds immediately.
- Otherwise, the function reads the record-offset table address and compares the selected adjacent offsets.
- Equal offsets also produce immediate success.

Neither immediate-success path explicitly initializes the output or prepares the stream.

For an initially nonempty interval, F-e40a3ceddab2 is called with the stream-like object. If it succeeds, the function **reads the descriptor’s record-offset table pointer and both selected offsets again** to form the interval difference supplied to F-d036abe1e536.

The earlier comparison does not capture the interval later passed to preparation. F-e40a3ceddab2 may change the descriptor contents, the table pointer, or the entries. Consequently, the second preparation call can receive a different difference, including zero, despite the earlier nonempty comparison. The record-count bound is not checked again. No local monotonicity check is applied, and neither preparation call visibly receives the starting offset as a separate argument.

Failure of the first preparation call prevents the second call. Failure of either releases the output, clears its slot, and returns that status. These failure paths do not call stream finalization.

### Scratch requests and live axis counts

After both preparations succeed, the function captures the current cursor and stream base, then requests three scratch regions in order through F-6dd003795db7. Each request uses size argument 4, two zero arguments, and an axis-count argument.

**Each scratch request independently reads the descriptor’s axis count immediately for that request.** There is no shared count snapshot for the three allocations. An earlier allocation dependency may change the count before a later request, so the three requested counts need not agree.

Allocation stops at the first nonzero reported status. Earlier successful scratch regions are not explicitly released when a later request fails. Stream finalization still occurs, followed by output release and slot clearing.

Only when all three requests report success does header and entry processing begin. Their returned addresses are not separately checked for zero.

### Header and shared selection

Two calls to F-7efb5338f0fa obtain header values:

- The first value’s low 12 bits give the entry count.
- Its bit 15 requests shared selection decoding.
- The second value’s low 16 bits give an initial data displacement.

The data position combines that displacement with the cursor-relative-to-base position captured **before the scratch requests**. Changes made during allocation or the header-reading calls do not replace that saved starting position.

If shared decoding is requested, the function saves the then-current header cursor and base, positions the cursor at the calculated data position relative to the current base, and calls F-1fc61b314574. The returned selection result and count become the shared selection.

The data position is then replaced by the decoder’s resulting cursor relative to the stream base read after the call. The header cursor is restored using its saved relative displacement and a base read at restoration time. Thus restoration preserves a relative position, not necessarily an earlier absolute address.

Without shared decoding, the shared result and shared count remain zero. No decoder error status is checked.

### Entry vectors and weights

Each entry begins with two further F-7efb5338f0fa calls. The first value’s low 16 bits are the entry’s eventual data-position increment. The second provides flags and, when applicable, a stored-vector index.

When bit 15 is clear, the low 12 bits select a stored vector. An index not below the descriptor’s stored-vector count sets status 8 and exits the entry loop before copying, weight computation, or that entry’s data-position increment. Otherwise, an external copy operation transfers a vector into the first scratch region, using descriptor-derived axis-count arithmetic for its address and byte size.

When bit 15 is set, the first scratch region is populated by repeated value reads and two-bit left shifts.

Bit 14 requests two additional such sequences, written first to the second scratch region and then to the third. Without bit 14, neither region is explicitly initialized or refreshed for that entry.

For **each of these inline-vector loops**, the axis count is read again at every loop-condition evaluation. It is not fixed at allocation time or loop entry. After a value-reading call returns, the current iteration writes its shifted result; the next condition reevaluates the live count.

Consequences include:

- Growth during a read can cause additional iterations and writes beyond the count requested for that region.
- Shrinkage during a read does not cancel the current iteration’s write.
- Shrinkage can terminate subsequent iterations.
- The three sequences can have different lengths, even within one entry.

The function does not reconcile these live counts with allocation capacity.

F-29e0f2d2fa66 then receives the descriptor, the low 16 flag bits, and all three scratch addresses. Its integer result is the entry weight. How it interprets stale or unpopulated scratch contents is unknown.

### Adjustment decoding and accumulation

A zero weight skips selection decoding, adjustment decoding, accumulation, and their cleanup block. The entry’s data-position increment still applies.

For a nonzero weight, the function saves the header cursor as a base-relative displacement and starts with the shared selection result and count.

If bit 13 is set, it moves the cursor to the current data position and calls F-1fc61b314574 for a private selection, replacing the selected result and count for this entry.

If bit 13 is clear, there is no corresponding move to the data position. Adjustment decoding begins at the cursor left by header/vector processing and weight computation.

F-f6968fd5e8be is then called twice, in first-component and second-component order. Each request uses the selected count when nonzero, otherwise the output element count. The second call is made even if the first returns zero.

Accumulation occurs only when the selected result and both adjustment addresses are nonzero:

- A selected result of −1 causes iteration over the output element count.
- Any other nonzero selected result causes iteration over the selected count.

For every processed adjustment, the function reads a signed 16-bit value, passes its sign-extended value and the weight to F-74efb2674226, and adds the returned value to the current four-byte output component. The first component is updated before the second component’s scaling call and update.

In the indexed branch, destination indices are obtained from the **retained private-selection address**, not from the selected shared-or-private result. That retained address initially equals zero. A non-sentinel private result is cleared during cleanup; a private result of −1 remains retained.

Accordingly, a shared non-sentinel selection without bit 13 does not establish safe indexed behavior. Its index source can be zero or a retained sentinel rather than the shared array.

The indexed destination is derived after the first scaling call and fetched again after the second scaling call. No guarantee establishes that the index storage remains unchanged between component updates. Repeated indices are not deduplicated.

A selected result of −1 does not override the decoder-request count: a nonzero selected count still controls both decoder requests, while accumulation processes the output element count. Sufficient adjustment storage therefore remains a precondition.

### Cleanup, restoration, and completion

After every nonzero-weight entry, whether or not accumulation occurred:

1. The retained private-selection result is passed to F-bf7242c98d06 unless it equals −1. This includes zero. A released result is reset to zero.
2. The first adjustment address is passed to that dependency.
3. The second adjustment address is passed to it.
4. The header cursor is restored from its saved displacement and the then-current stream base.

The shared selection result is never explicitly released.

Each normally completed entry advances the data position by its saved low-16-bit increment. An invalid stored-vector index exits before this advancement.

After the fully allocated processing block, the three scratch addresses are released in allocation order. F-9787e15c337d then finalizes the prepared stream; its return value is not used as a status.

A zero status retains the output. A nonzero status releases it, writes zero to the output slot, and returns the status. Thus status 8 discards output even after earlier entries modified it.

## Side effects

Observable effects include output-address publication, ordered accumulator read-add-write operations, direct stream-cursor writes, dependency calls, and explicit releases.

The function has no output-zeroing operation. Output contents on immediate success, and initial accumulator values on processing paths, are dependency-defined.

Arithmetic includes four-byte table indexing, four-byte vector strides, eight-byte output strides, two-byte adjustment/index strides, offset subtraction, cursor-relative additions, left shifts by two, and accumulator additions. No overflow checks are visible. Exact consequences of signed overflow, shifted negative values, and invalid computed addresses are **documented_only**.

Unknown dependencies may modify descriptor fields, stream fields, backing storage, or other state. In particular, safety cannot be justified by assuming stable interval data across preparation or stable axis counts across allocations and reads.

Explicit release omissions do not prove that memory leaks: ownership and possible external reclamation are unknown. Conversely, no external reclamation may be assumed.

## Dependencies

| Opaque ID | Observed role |
|---|---|
| F-6dd003795db7 | Returns allocation-like addresses and reports status for output and scratch requests. |
| F-bf7242c98d06 | Receives the captured handle and addresses for release-like operations. |
| F-e40a3ceddab2 | First preparation operation; success is followed by fresh interval reads. |
| F-d036abe1e536 | Second preparation operation, receiving the newly calculated interval difference. |
| F-9787e15c337d | Finalization after successful preparation, including scratch-allocation failure paths. |
| F-7efb5338f0fa | Supplies header and inline-vector values; may affect subsequent live field reads. |
| F-1fc61b314574 | Returns a selection result and writes a count. |
| F-f6968fd5e8be | Returns an adjustment address for a requested count. |
| F-29e0f2d2fa66 | Supplies the entry weight from descriptor, flags, and scratch regions. |
| F-74efb2674226 | Supplies the contribution added to one output component. |

The external vector-copy operation has no supplied opaque identifier or independent contract. Overlap handling and other effects are unknown.

## Acceptance examples

These are conditional static cases, not executed tests.

1. **Disabled processing:** With valid memory for initial nested reads and a zero enable byte, return 6 without an explicit output-slot write.

2. **Unavailable record:** After successful output allocation, an out-of-range record or initially equal adjacent offsets retains the published output and returns 0 without preparation. Output initialization is not implied.

3. **Interval changes during preparation:** Suppose the initial comparison observes offsets 20 and 28. If F-e40a3ceddab2 succeeds after replacing the table pointer with one whose selected entries are 100 and 112, the second preparation call receives difference 12, not 8. If those replacement entries are equal, it receives zero rather than taking the earlier empty-interval shortcut.

4. **Changing allocation counts:** If the axis count is 2 for the first scratch request, becomes 4 before the second, and becomes 1 before the third, the requests use counts 2, 4, and 1 respectively. No common count is enforced.

5. **Changing inline count:** Suppose a scratch region was requested for two values. During an inline sequence, the first value-reading call changes the count to three. The first value is written, and subsequent condition checks can admit values at indices one and two. Accept this as observable live-count behavior, not as evidence of adequate capacity.

6. **Count shrinks during a read:** If an inline iteration has already passed its condition and its reader reduces the count to zero, that iteration still writes its returned value before the next condition terminates the loop.

7. **Third scratch request fails:** Finalize the prepared stream, release the output, clear its slot, and return the allocation status. Do not assert explicit release of the first two scratch regions.

8. **Invalid stored vector:** An index equal to the stored-vector count produces status 8 before copying or weight computation. Release all three successfully allocated scratch regions, finalize, then release and clear the output.

9. **Zero weight:** Make no adjustment-decoder calls or accumulation for that entry; still advance its data position.

10. **All-elements accumulation:** Given selected result −1 and sufficient adjustment/output storage, process exactly the output element count, updating each first component before scaling and updating its second component.

11. **Private repeated indices:** Given a valid private selection containing the same destination twice, apply both contributions in selection order. No deduplication occurs.

12. **Shared sparse selection:** A non-sentinel shared selection without private decoding does not establish valid index access; accumulation uses the retained private address instead.

## Uncertainties

The following remain **documented_only**:

- Dependency signatures, initialization, ownership, release behavior, and state-preservation guarantees.
- Decoder failure conventions and validity of returned storage and counts.
- The formal meaning of −1 beyond its observed branch behavior.
- Intended cursor positioning without bit 13 and intended shared sparse-selection handling.
- Whether bit-14-clear scratch contents are valid inputs to weight computation.
- Storage safety under changing axis counts or altered interval tables.
- Reclamation of shared selections and scratch regions without explicit release.
- Zero-count allocation/decoding semantics and malformed-input behavior.
- Exact overflow semantics and concurrency guarantees.

## Evidence

This contract derives from the supplied static function representation and opaque dependency mapping, not from the prior draft’s authority.

Established evidence comprises access widths and offsets, branch conditions, dependency-call ordering, fresh interval reads after first preparation, independently evaluated allocation counts, live inline-loop count checks, explicit cursor assignments, accumulator operations, and visible cleanup paths.

“Allocation,” “stream,” “selection,” “weight,” and “finalization” describe inferred roles without supplying missing dependency guarantees.

No runtime tests have been performed.
