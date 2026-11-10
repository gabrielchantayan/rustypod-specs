<!-- Trusted provenance: F-f2daf800ed06; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f2daf800ed06

## Purpose

F-f2daf800ed06 processes a NUL-terminated byte sequence through a context-dependent preparation and execution interface. It can process multiple units from that sequence, report selected execution events through a callback, and optionally return an allocated copy of context-associated text.

Established behavior includes:

- Immediate success-like return of zero for a null input pointer, without updating the optional output slot.
- Repeated preparation from the current input position.
- Execution while the step dependency returns status 100.
- Optional callback delivery of field metadata and row values.
- A limited retry path for status 17.
- Final status translation through a context dependency.
- Conditional allocation and copying of associated text.

An inferred interpretation is that the input contains statements, status 100 denotes a row, status 101 denotes completion, and status 17 requests re-preparation. These interpretations are not necessary to the contract: the numeric comparisons and control-flow consequences below are established independently.

## Interface

The function accepts five arguments:

| Argument | Role |
|---|---|
| Context | A 32-bit context address or handle forwarded to dependencies and used for direct memory accesses. |
| Input | A pointer to NUL-terminated bytes, or null. |
| Callback | An optional callable address. |
| Callback context | An opaque 32-bit value forwarded unchanged as the callback’s first argument. |
| Text output | An optional pointer to a writable 32-bit result slot. |

The return value is a 32-bit integer obtained from F-15a13295fe3c, except that null input returns zero directly.

When called, the callback receives four arguments, in order:

1. The opaque callback context.
2. A signed 32-bit field count.
3. A pointer to an array of 32-bit field-value addresses, or zero.
4. A pointer to an array of 32-bit field-metadata addresses.

The callback’s result is interpreted only as zero versus nonzero. A nonzero result terminates processing through status 4.

The supplied representation establishes a 32-bit address and integer model for this function. It does not establish a portable interface for wider pointers.

### Preconditions

For the defined execution path:

- A non-null input must permit byte reads through a terminating zero and through any subsequent positions returned by the preparation dependency.
- The context must be valid for the invoked dependencies and for the direct accesses described below.
- A non-null text-output argument must designate a writable 32-bit slot.
- A non-null callback must obey the indicated calling interface.
- Returned field counts, pointers, and allocation sizes must be mutually consistent and fit the relevant 32-bit operations.

No explicit validation of these conditions is present. In particular, the function does not check a null context, invalid input addresses, negative field counts, or arithmetic overflow.

## Behavior

### 1. Null input and initial state

If the input pointer is null, the function returns zero immediately. It does not call any dependency, dereference the context, or write the text-output slot.

For non-null input, processing begins with status zero, a zero retry counter, and no active prepared object.

An input whose first byte is zero proceeds directly to final status translation. This differs from null input: dependencies can run, and the text-output slot can be written.

### 2. Preparation and input advancement

At each preparation boundary, the function reads the byte at the current input position.

If it is nonzero, the function clears its prepared-object slot and calls F-afd705c57f19 with:

- The context.
- The current input position.
- The 32-bit all-ones value, represented as −1.
- An address at which to return a prepared object.
- An address at which to return a continuation position.

The returned integer is the preparation status.

If preparation returns zero and no prepared object, the continuation position becomes the next input position immediately. Processing repeats there without executing or releasing an object.

There is no check that this continuation position advances. A dependency that repeatedly returns the same nonzero-byte position with no object can therefore prevent termination.

If preparation returns a nonzero status, execution is skipped. Status 17 enters the retry handling described below; other statuses enter final processing.

### 3. Execution and callback eligibility

For a successfully prepared, non-null object, the function obtains a signed field count from F-4677636484bc. This call has no explicit arguments in the supplied representation.

The function then repeatedly calls F-7fd29baf5f2f with the prepared object.

With no callback, status 100 causes another step. Any other status exits the step loop.

With a callback, only the following cases can reach callback delivery:

- Status 100.
- Status 101 before any callback metadata has been captured, and only if the 32-bit context word at byte offset 12 has bit 0x100 set.

Other statuses exit the step loop without calling the callback. Status 101 after a prior row callback also exits without a completion callback.

The context flag is tested after the step call. The function does not snapshot it before execution.

### 4. Callback storage and metadata

Before the first eligible callback, the function requests temporary storage from F-f053f140662a. The request contains the context and a size of eight times the field count plus one byte.

The storage is divided into two adjacent regions:

- Field metadata begins at the allocation base.
- Field values begin four times the field count bytes after that base.

Each array entry occupies four bytes. The extra byte is not explicitly initialized or used by this function.

For each index from zero up to, but excluding, the field count, F-c859b7b263d8 supplies one metadata address. Each returned address is written immediately to the corresponding metadata entry. If an address is zero, metadata collection stops, the context error byte is set as described below, and final processing begins.

Metadata is collected once for the prepared object. Later row callbacks reuse the stored addresses.

The size calculation and array-offset calculations use 32-bit integer expressions. There is no explicit overflow protection. Safe behavior requires a field count for which the size and offsets are representable and the allocated region is large enough.

### 5. Row values and callback invocation

For status 100, the value-array pointer is set to the second region of the temporary allocation.

For each field index:

1. F-dc482c6e7d00 supplies a value address.
2. The address is written immediately into the value array.
3. If the address is zero, F-574787625ec8 is called for that index.
4. A zero address is accepted only if that dependency returns 5.

Any other result for a zero value address stops collection, sets the context error byte, and enters final processing. The callback is not invoked for that incomplete row.

For the eligible status-101 case, metadata is collected but values are not. The callback receives a zero value-array pointer.

A zero-field callback remains possible. On successful allocation, the function requests one byte, performs no metadata or value-entry writes, and invokes the callback with field count zero. For status 100, the metadata and value pointers coincide; for eligible status 101, the value pointer is zero.

A callback return of zero permits the normal step-loop decision to continue. A nonzero return replaces the processing status with 4 and immediately enters final processing.

### 6. Normal object completion

After a step loop exits normally, F-f0392a59c9a9 is called with the prepared object. Its result becomes the status used for subsequent progression and retry decisions. Thus the finalization result, rather than the last step result alone, determines whether processing continues or retries.

The function then clears its active-object slot.

If finalization returns anything other than 17:

- The retry counter is reset to zero.
- The returned continuation position is selected.
- A classification table obtained through F-2a4c402ba617 is used to advance past consecutive bytes whose table entry has bit 0 set.

The table is indexed by unsigned byte values. The exact character class, including its treatment of the terminating zero byte, is dependency-defined.

Temporary callback storage is then passed to F-94cb53877c15, even when its address is zero.

A finalization result of zero resumes preparation at the selected position. Any other result proceeds to retry handling or final processing.

### 7. Retry handling

Status 17 is retried once within the current retry interval.

On the first such result, the retry counter becomes one and preparation restarts at the beginning of the current unit, not at its continuation position.

On the second such result without an intervening counter reset, processing stops and enters final processing with status 17.

The counter resets only after normal object finalization returns a status other than 17. A successful preparation that produces no object does not reset it.

No temporary-storage or object cleanup is explicitly performed on the preparation-error branch before retrying. Valid behavior therefore depends on what F-afd705c57f19 guarantees about its output arguments when returning an error.

### 8. Early final processing and status translation

Some conditions enter final processing without the normal completion sequence:

- Temporary callback-storage allocation returns zero.
- Metadata contains a zero address.
- A zero value address has a type other than 5.
- The callback returns nonzero.
- A preparation error is not retried.
- The retry limit is reached.
- Input termination is reached.

If an active object remains, F-f0392a59c9a9 is invoked before temporary-storage cleanup. Its result is not used on this path. The explicit argument at this cleanup call site is not recoverable from the supplied representation; this limitation is recorded below.

If temporary callback storage is nonzero, F-94cb53877c15 is then called with it.

F-15a13295fe3c is called with the context and the current processing status. Its result is the function’s eventual return value.

The temporary-allocation and field-extraction failure paths do not substitute a dedicated integer error status themselves. They retain the current step status; field-extraction failure additionally sets the context error byte. Any resulting error conversion depends on F-15a13295fe3c and context state.

### 9. Optional text result

After status translation:

- If the translated result is zero, the function skips the context-status query.
- Otherwise, it calls F-0b90d48d906b with the context and compares that result with the translated result.

If the translated result is zero, or differs from the queried context status, a non-null text-output slot is written with zero. The translated result is returned.

If the translated result is nonzero and matches the queried status:

- A null text-output argument causes an immediate return.
- Otherwise, F-36aa80c7a003 is called with the context, and its direct return value is discarded.
- F-1cb585febd84 is called without explicit arguments to obtain a length-like value.
- F-391e3d30e634 is called with that value plus one.
- The allocation result is written immediately into the text-output slot.
- If allocation returns zero, the translated result is returned unchanged.
- Otherwise, F-36aa80c7a003 is called again, and an external byte-copy dependency copies the computed length plus one bytes to the address then read from the text-output slot.

The source suggests copying terminated diagnostic text, but neither the length dependency’s exact semantics nor termination of the copied region is established here. The extra byte is included in the copy; it is not explicitly written as zero by this function.

## Side effects

Established direct writes are:

- Prepared-object and continuation outputs supplied to the preparation dependency.
- Four-byte entries in temporary metadata and value arrays.
- The single byte at context offset 30, set to 1 after a field-extraction failure.
- The optional four-byte text-output slot.
- The allocated text destination through the external copy operation.

The context error byte is not explicitly reset.

For a non-null text-output argument, an existing slot value is overwritten without first being released. Null input is the exception: it leaves the slot untouched.

Temporary array storage is passed to F-94cb53877c15 on the observed cleanup paths. Its allocator’s ownership semantics and that dependency’s actual release behavior remain external contracts.

The successful text allocation is not explicitly released by this function. Caller ownership and the appropriate release operation are not established.

The function does not explicitly release metadata or value addresses returned by field dependencies. Their validity during or after callback execution must come from those dependencies; no retention guarantee is established.

Callbacks and dependencies may alter context or global state. The contract does not assume that they preserve globals, retain object validity, or leave aliased output slots unchanged.

## Dependencies

| Opaque ID | Observed role |
|---|---|
| F-afd705c57f19 | Prepares input and supplies object and continuation outputs. |
| F-4677636484bc | Supplies the field count, with no explicit argument shown. |
| F-7fd29baf5f2f | Steps a prepared object. |
| F-f053f140662a | Supplies context-associated temporary storage. |
| F-c859b7b263d8 | Supplies indexed metadata addresses. |
| F-dc482c6e7d00 | Supplies indexed value addresses. |
| F-574787625ec8 | Supplies the discriminator checked against 5 for zero values. |
| F-f0392a59c9a9 | Completes or cleans up an active object. |
| F-94cb53877c15 | Receives temporary storage on cleanup. |
| F-2a4c402ba617 | Supplies access to a byte-classification table. |
| F-15a13295fe3c | Translates the processing status using the context. |
| F-0b90d48d906b | Supplies the context status used to gate text copying. |
| F-36aa80c7a003 | Obtains or selects context-associated text. |
| F-1cb585febd84 | Supplies the length-like value used for allocation and copying. |
| F-391e3d30e634 | Supplies the optional text-copy allocation. |

An additional external byte-copy operation is visible, but no opaque function ID was supplied for it.

## Acceptance examples

These examples are conditional static acceptance criteria, not runtime observations.

1. **Null input:** Given a null input and a text-output slot containing an existing address, the result is zero, the slot is unchanged, and no dependency is called.

2. **Empty non-null input:** Given an input beginning with zero, F-15a13295fe3c receives processing status zero. If it returns zero, a supplied text-output slot becomes zero.

3. **No callback:** Given step results 100, 100, and 101, the function makes three step calls, performs no callback-array allocation, and then invokes normal object finalization.

4. **Two row callbacks:** Given two status-100 events and a positive valid field count, metadata is captured for the first callback only. Values are fetched for each row. A following status 101 does not trigger a third callback.

5. **Completion-only callback:** Given an initial step result of 101, a callback, and context bit 0x100 set, the callback receives metadata and a zero value-array pointer. With that bit clear, no callback occurs.

6. **Accepted zero value:** Given a zero value address and discriminator 5, the zero is retained in the value array and callback delivery remains eligible. Any other discriminator sets the context error byte and prevents that row’s callback.

7. **Callback cancellation:** Given a callback returning nonzero, final status translation receives status 4 after early cleanup, rather than the preceding step status.

8. **One retry:** Given two consecutive preparation results of 17 from the same input position, there are two preparation attempts before final status translation with status 17.

9. **Finalization controls progression:** Given a non-100 step result followed by finalization result zero, processing continues at the classified continuation position. The step result is not independently returned.

10. **Text allocation failure:** Given a nonzero translated result matching the context status and an allocation result of zero, the output slot becomes zero and the translated return value remains unchanged.

## Uncertainties

The following are documented_only:

- The semantic meanings of statuses 4, 17, 100, and 101, and discriminator 5.
- The full context layout and the interpretation of its flag and error fields.
- The implicit state used by dependencies shown without explicit arguments.
- The argument supplied to early object cleanup.
- Preparation-output validity on failure and resulting cleanup obligations.
- Allocation, ownership, release, aliasing, and reentrancy guarantees.
- Whether field addresses remain valid across steps, callbacks, or finalization.
- The classification table’s exact membership and terminator behavior.
- The text source’s encoding, length semantics, termination, and stability between its two retrieval calls.
- Behavior for overflowing arithmetic, invalid counts, invalid pointers, or dependencies that fail to advance input.

## Evidence

This contract is based solely on static control flow, argument use, numeric comparisons, memory widths, and observable memory operations in the supplied representation. Descriptive roles such as “row,” “metadata,” “finalization,” and “diagnostic text” are inferred interpretations; the numeric and ordering rules define the established behavior.

No runtime tests have been performed. No external dependency implementation or ownership contract was available.
