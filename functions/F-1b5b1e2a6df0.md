<!-- Trusted provenance: F-1b5b1e2a6df0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1b5b1e2a6df0

## Purpose

F-1b5b1e2a6df0 coordinates an apparent byte-read operation on a resolved, position-bearing object. It selects buffered or complete-sector dependency calls, maintains a destination address and remaining count, updates three object fields, and reports a final status.

The established behavior is the orchestration: argument construction, dependency calls, selected result checks, arithmetic, and explicit state writes. The supplied body does **not** establish that the dependencies copy bytes or perform storage transfers. Those roles are inferred from their arguments and use.

Recognized transfer failures cause explicit restoration of three saved object fields. They do not establish rollback of destination memory, dependency state, or other object fields. Later dependencies can also alter the restored fields.

This is a static behavioral contract. No runtime tests have been performed.

## Interface

### Arguments and result

| Item | Representation | Established use |
|---|---|---|
| First argument | Opaque 32-bit value | Saved before resolution; the displayed resolver invocation has no explicit arguments, so its binding to this value is unresolved |
| Second argument | 32-bit address-like value | Tested for zero, supplied as a destination-like argument, and advanced by accounted byte quantities |
| Third argument | Unsigned 32-bit value | Requested byte count |
| Return | Unsigned 32-bit value | Zero, the retained effective count, or a value read from a global failure-result location |

No destination capacity is supplied. If the dependencies implement the apparent memory operations, the caller must supply sufficient writable storage and valid address arithmetic. The function itself does not validate capacity.

A recognized transfer failure does not return an accounted successful prefix. It returns the global failure value.

### Object and linked layouts

Successful resolution supplies a mutable object with these accessed fields:

| Byte offset | Width | Role established by use |
|---|---:|---|
| +0 | 32 bits | Pointer to an associated descriptor |
| +8 | 32 bits | Position |
| +12 | 32 bits | Logical allocation/index state |
| +16 | 32 bits | Base-sector state |

The descriptor contains a 32-bit context pointer at +0 and another 32-bit pointer at +4. The latter points to an object whose unsigned 32-bit field at +28 supplies the reported length.

The retained context has these accessed fields:

| Byte offset | Width | Role |
|---|---:|---|
| +104 | 32 bits | Position mask |
| +120 | 16 bits | Opaque handle passed to selected dependencies |
| +460 | Unsigned 16 bits | Allocation-unit sector quantity |
| +462 | Unsigned 16 bits | Shift field; each use selects its low eight bits |

Valid pointers, accessible fields, and appropriate alignment are external preconditions. No explicit validation is visible.

The context pointer is obtained during preparation and retained. Later context-field reads use that pointer; the function does not reacquire it through the descriptor each iteration.

### Arithmetic

A sector is 512 bytes. Counts, positions, sector arithmetic, and address-like bookkeeping operate at 32-bit width. Unsigned additions, subtractions, and left shifts must not be replaced with unbounded arithmetic.

Each shift-field access reads the then-current 16-bit field and uses its low eight bits. Behavior for shift counts outside the supported range of a 32-bit operation remains unresolved; this contract does not impose portable semantics for those cases.

## Behavior

### Resolution and no-transfer exits

F-667717e3fb3e is invoked before testing the destination or requested count.

If resolution returns null, the function selects status 9, reads the global failure-result value, calls F-038b46411c0c with 9, and returns the previously read value. It does not dereference the resolved object. Resolution failure therefore takes precedence over an empty request or zero destination.

With successful resolution, a zero requested count or zero destination produces a zero result and a call to F-038b46411c0c with zero. No position/length comparison or preparation dependency is invoked.

Otherwise, F-4c6d30ac2311 is called. The function then compares the unsigned object position with the reported length reached through the descriptor.

If position is at or beyond length, it calls F-cc60a532ba6c, reports status zero, and returns zero. No transfer-loop dependency is called.

The two preparation dependencies suggest synchronization, but locking, exclusion, and preservation guarantees are not established.

### Preparation and effective count

When the initial comparison finds position below length, preparation proceeds in this order:

1. Call F-cc60a532ba6c.
2. Obtain and retain the context pointer through the object’s descriptor.
3. Read the context handle and call F-6443787f9616 with it.
4. Call F-e657f684a3c8 with the resolved object.
5. Save position, then base-sector state, then logical allocation/index state.
6. Read the reported length again.
7. Determine the effective count.

The snapshots are therefore post-preparation values, not necessarily entry values. The descriptor and length used for the second length read need not match those observed before the intervening calls.

Initially, the effective count is the requested count. If the reread length is less than or equal to the unsigned 32-bit sum of saved position and requested count, the effective count becomes length minus saved position, also at 32-bit width.

Consequences include:

- Under stable metadata and nonwrapping arithmetic, the count is limited to the reported end.
- Equality takes the limiting branch without changing the numerical count.
- Overflow of position plus request can prevent an expected limit.
- A reread length below saved position can produce a large unsigned count rather than zero.

The effective count is retained for successful return. A separate remaining count controls loop progress. An effective count of zero still reaches the completion dependency described below.

### Iteration planning

For each iteration with a nonzero remaining count, the function derives a sector offset by intersecting the current position with the current context mask, retaining the low 16 bits, and shifting right by nine. This produces an offset from 0 through 127.

It also saves the current base-sector field before calling F-5a8a1a11a291. That saved value is used by the buffered path.

F-5a8a1a11a291 receives:

- The resolved object.
- The retained context pointer.
- The current logical allocation/index value.
- An address for a 32-bit output value.
- A planning quantity.

The planning quantity uses the current remaining count and reads the allocation-unit quantity and shift field. Remaining count plus 511 is first evaluated at 32-bit width and shifted right by nine. The allocation-unit quantity is added, one is subtracted, and the resulting 32-bit quantity is shifted right by the low eight bits of the shift-field read associated with this planning calculation.

A zero planning result is a recognized failure. A nonzero result is treated as a span quantity. The output slot may later supply the next logical allocation/index value.

The supplied body does not establish that the dependency initializes its output whenever consumed, preserves object or context fields, or returns a span consistent with the context geometry.

### Buffered path

After planning succeeds, the function reads position again to select a path. The buffered path is selected if that position is not 512-byte aligned or fewer than 512 bytes remain.

It then reads position again to obtain the low nine bits as the within-sector offset. Thus, an entry position or pre-planning position does not establish this offset.

F-04b7df4e1a09 receives the retained context and a sector-like argument formed from the **pre-planning saved base-sector value** plus the **pre-planning derived sector offset**. A null result is a recognized failure.

For a nonnull result, the accounted byte quantity is the smaller of remaining count and 512 minus the saved within-sector offset. F-2eceb10e9abc is called with:

- The current destination address.
- The returned address plus 24 plus the saved within-sector offset.
- The accounted byte quantity.

Its result is not checked. F-2b1ba6d8de9d is then called with the returned address and zero. These calls establish neither copying semantics nor ownership or release semantics.

Only after both calls does the function read object position, advance the destination, write position plus the accounted quantity, and reduce remaining count. The position increment therefore uses a value read after both dependencies return, not the position used for path selection.

The advanced position is tested against a fresh read of the full 32-bit context mask. This differs from the low-16-bit masking used to derive the sector offset.

If the masked advanced position is nonzero, iteration resumes without an explicit allocation/index transition.

If it is zero:

- For a planning span other than one, the current base-sector field is incremented by the then-current unsigned 16-bit allocation-unit quantity, then the current logical allocation/index field is incremented by one.
- For a planning span of one, the logical allocation/index field is replaced by the planning output. F-08426f0ea83d receives the context and that selected value, and its result is stored in the base-sector field.

The mapping result is not checked for failure.

### Direct path

The direct path requires the post-planning alignment read to show a 512-byte-aligned position and at least 512 remaining bytes.

The function now reads the shift field **again**, after F-5a8a1a11a291. Using the low eight bits of this new read, it shifts the planning span left and subtracts the pre-planning sector offset. This 32-bit candidate sector count is capped by the number of complete sectors remaining.

The shift used here need not equal the shift used to form the planning argument: F-5a8a1a11a291 can change the field.

If the capped count is zero, the function returns to iteration planning without an explicit transfer call or bookkeeping advance. No progress guard is visible.

Otherwise, F-a63889a8554d receives:

1. The then-current 16-bit context handle.
2. The then-current object base-sector field plus the pre-planning sector offset.
3. The current destination address.
4. The capped sector count.
5. Zero.

Unlike the buffered path, this sector-like argument uses a base-sector field read after planning, rather than the saved pre-planning base.

A zero result is a recognized failure. Any nonzero result is accepted for bookkeeping as success for the full submitted sector count; it is not interpreted as a short-transfer count. Actual transfer semantics are not established.

On an accepted result, destination advances and remaining count decreases by sector count times 512. The function then **reads object position after F-a63889a8554d has returned** and writes that position plus the same byte quantity.

After that position write, the function reads the shift field **a third time**, now after the accepted direct dependency result. The low eight bits of this read govern the allocation-unit advance: transferred sector count plus the pre-planning sector offset, shifted right.

This third read need not equal either earlier shift read. In particular, F-a63889a8554d can change the field before it is used.

If the calculated advance equals the planning span, the selected logical allocation/index value is the planning output. Otherwise, it is the then-current logical allocation/index field plus the advance. The selected value is written to the object and passed with the context to F-08426f0ea83d. Its result is stored as base-sector state without a failure check.

### Completion and recognized failures

When remaining count reaches zero, the function reads the context handle, calls F-591e5dd605b9, calls F-038b46411c0c with zero, and returns the retained effective count. Success denotes completion of this bookkeeping protocol, not independently proven delivery of bytes.

Recognized failures are limited to these checked results:

- Zero from F-5a8a1a11a291.
- Null from F-04b7df4e1a09.
- Zero from F-a63889a8554d.

The failure path explicitly writes saved position, saved base-sector state, and saved logical allocation/index state, in that order. It selects status 28, reads the current context handle, calls F-591e5dd605b9, reads the global failure-result value, calls F-038b46411c0c with 28, and returns the selected value.

The global is read after F-591e5dd605b9 on this path, but before status reporting. On resolution failure it is also read before status reporting, without the completion call.

## Side effects

Established direct effects are:

- Writes to the three tracked object fields.
- Advancement of destination and count bookkeeping.
- Calls exposing destination-like, source-like, object, context, handle, and output-slot arguments.
- A final status-reporting invocation on every visible returning path, assuming dependencies return normally.

Destination writes are **dependency-mediated and unproven**. If the dependencies implement the inferred operations, destination contents may change even when a later recognized failure occurs. No undo operation is visible.

Restoration is limited to three explicit writes. It does not restore context fields, descriptor links, length, dependency state, or globals. Subsequent completion and status dependencies may alter restored fields, so final-state equality additionally requires their preservation.

No allocation, deallocation, ownership transfer, or buffer release is directly established. Ignored dependency results do not establish success or failure propagation.

## Dependencies

| Opaque ID | Observed invocation role |
|---|---|
| F-667717e3fb3e | Supplies the resolved object or null |
| F-4c6d30ac2311 | Precedes the initial position/length comparison |
| F-cc60a532ba6c | Follows that comparison on either branch |
| F-6443787f9616 | Receives a context handle before preparation |
| F-e657f684a3c8 | Receives the object before snapshots |
| F-5a8a1a11a291 | Returns a checked span-like value and receives a 32-bit output address |
| F-04b7df4e1a09 | Returns a checked buffer-like address |
| F-2eceb10e9abc | Receives destination-like and source-like addresses and a byte quantity; copying is inferred |
| F-2b1ba6d8de9d | Receives the buffer-like address and zero; release semantics are unknown |
| F-a63889a8554d | Receives handle, sector-like address, destination, sector count, and zero; transfer semantics are inferred |
| F-08426f0ea83d | Supplies the value stored as base-sector state |
| F-591e5dd605b9 | Receives a handle at prepared completion or failure |
| F-038b46411c0c | Receives final status 0, 9, or 28 |

These roles are not independent dependency contracts. No preservation of mutable shared state is assumed.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Resolution failure precedes an empty request.** With requested count zero and null resolution, status reporting receives 9 and the result is the global failure value read before that call.

2. **Successful resolution with zero destination.** The result is zero and status reporting receives zero. No preparation or position/length comparison occurs.

3. **Already at end.** If the comparison after F-4c6d30ac2311 observes position equal to length, F-cc60a532ba6c is called, no loop operation occurs, and the result and status argument are zero.

4. **Ordinary limiting.** Assume both relevant length reads report 1,000, preparation leaves the saved position at 900, and request is 300. The effective count is 100. If subsequent dependencies permit completion and preserve position except for the explicit increments, position reaches 1,000 and the return is 100. Actual destination contents require separate dependency guarantees.

5. **Buffered boundary.** Suppose ten bytes remain and planning succeeds. Require the post-planning alignment and offset reads both to observe position 510. If buffer acquisition succeeds, F-2eceb10e9abc receives byte quantity two and a source-like address equal to the buffer address plus 24 plus 510. This follows regardless of whether copying actually occurs. If the intervening buffer-related dependencies preserve position, the explicit position write advances it to 512.

6. **Two-sector direct submission.** Suppose 1,024 bytes remain. Require the post-planning alignment read to be aligned, the pre-planning sector offset to be zero, planning to return span one, and the post-planning shift read to yield one. The candidate is two sectors, so F-a63889a8554d receives count two and final argument zero. A nonzero result causes accounting for 1,024 bytes. The position increment uses the position observed after that dependency returns.

7. **Distinct shift reads.** With sufficient remaining count, pre-planning sector offset zero, and planning result one, a post-planning shift of one permits a two-sector submission. If the direct dependency succeeds but changes the shift to two, allocation advance is zero, not one. The current logical allocation/index value is therefore selected through the unequal-span branch.

8. **Failure after earlier accepted accounting.** If an earlier operation is accepted and later planning returns zero, the three snapshot values are explicitly restored and status reporting receives 28. The return is the global failure value, not the earlier accounted prefix. Final preservation of the restored fields requires the later dependencies not to change them.

9. **Unsigned limit overflow.** Saved position 512, reread length 4,096, and request 4,294,967,040 give a wrapped position-plus-request value of 256. The limiting comparison is false; the effective count remains the large request rather than 3,584. This example establishes only count selection, not safe completion.

## Uncertainties

The following remain **documented_only**:

- Resolver argument binding and the first argument’s detailed meaning.
- The global failure value and whether dependencies change it.
- Actual copying, storage access, failure atomicity, synchronization, and ownership.
- Valid geometry relationships among mask, allocation-unit quantity, shifts, span, and state.
- Initialization and validity of the planning output.
- Behavior for invalid pointers, address wraparound, insufficient destination capacity, unsupported shifts, or inconsistent metadata.
- Errors from unchecked dependencies and exceptions or nonreturning behavior.
- Termination when direct iterations repeatedly produce zero sectors.
- Concurrent changes and preservation guarantees across dependencies.

## Evidence

The supplied function body establishes the field accesses, arithmetic, branch conditions, explicit writes, calls, and checked results described here. The supplied opaque dependency map provides only dependency identifiers.

Storage, copying, synchronization, and release interpretations are inferred intent, not established dependency behavior. Prior prose and reviewer feedback are not independent evidence. No runtime tests, executable smoke runs, or dependency-implementation inspection have been performed.
