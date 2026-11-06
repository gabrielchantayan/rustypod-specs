<!-- Trusted provenance: F-2abff646f58d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2abff646f58d

## Purpose

F-2abff646f58d performs a lookup and, if lookup returns null, attempts to construct and submit a new 12-byte record. It returns the lookup result, the newly allocated record address, or null.

The established control flow includes an optional initialization gate, an opening notification, lookup, optional construction, a closing notification, and conditional failure reporting.

**Inferred intent:** the first input identifies a record in a shared collection. This interpretation is supported by its placement in the lookup descriptor and in the first word of a newly allocated record. The function alone does not establish matching rules, uniqueness, successful insertion, or the business meaning of the record.

The two notification calls must not be treated as a proven lock pair or transaction boundary. Their effects are unknown.

## Interface

### Inputs

The function accepts four 32-bit values. Their established uses are:

| Input | Use |
|---|---|
| First | Copied into the first lookup-descriptor word; written at byte offset 0 of a new record; supplied as the final argument to both notifications. |
| Second | Written into a temporary local word only when the initially checked handle is zero, immediately before the initialization dependency. That word is subsequently overwritten with the first input if execution continues. |
| Third | Stored in the local word immediately following the descriptor’s first word in the displayed layout. |
| Fourth | Stored in the next local word in the displayed layout. |

No input is explicitly validated. No signed interpretation, string interpretation, range restriction, or nullability rule is established. A zero first input does not itself cause rejection.

The second input is not an explicit argument to any dependency. Its temporary storage is established; whether initialization can observe that storage through an implicit convention is unknown.

### Return value

The return is a pointer-shaped value in the displayed 32-bit address model:

- The nonnull result of F-932a35189ea0, if lookup succeeds.
- The allocation address returned by F-43da6b23bd44, if the subsequent creation value is nonzero and F-dbd019788d3d returns normally.
- Null on initialization failure, allocation failure after a lookup miss, or a zero creation value.

A nonnull return does not prove that a record remains valid, that registration succeeded, or that the caller owns it. Dependencies invoked after an address is selected may affect its contents or lifetime.

### New-record layout

The allocation request is exactly 12 bytes. The function treats a nonnull result as storage for three consecutive 32-bit words:

| Byte offset | Width | Value directly written |
|---|---:|---|
| 0 | 32 bits | First input |
| 4 | 32 bits | Result of F-c141a8d8555e |
| 8 | 32 bits | Zero |

These are the function’s writes, not guaranteed final contents after dependencies run. Existing records returned by lookup are neither validated nor directly rewritten here.

### Shared-context access model

A global location supplies a context-base value. Two source-level references to that global must be distinguished:

1. The function first saves the global context base for later use.
2. The initialization condition references the global again and checks the 32-bit field at byte offset 8 through the base obtained by that reference.

Lookup and registration obtain their handles from offset 8 through the **previously saved base**, not through another reference to the global context-base location.

Equality of the saved base and the base used by the initialization condition is not explicitly checked. This describes source-level accesses; it does not establish distinct machine loads, atomicity, or guaranteed behavior under concurrent modification.

### Preconditions

For normal completion:

- The global context-base location must be readable at its source-level references.
- The base used by the initialization condition must support reading a 32-bit field at offset 8.
- The saved base must support each later offset-8 read that execution reaches.
- A nonnull allocation result must support the three 32-bit writes.
- All addresses supplied to dependencies must satisfy those dependencies’ requirements.

These conditions are not explicitly validated. Invalid addresses, faults, non-returning dependencies, and exceptional transfers of control have no recovery contract established by this function.

## Behavior

### 1. Capture context and prepare local inputs

The function first saves the global context base. Before evaluating the initialization gate, it also preserves the first input for notification arguments and stores the third and fourth inputs in their displayed local words.

It then references the global context-base location again to obtain the base used for the initial offset-8 handle check.

This distinction matters even though the two bases ordinarily may be equal. The source does not compare them or replace the saved base with the one used by the condition.

### 2. Apply the initialization gate

If the handle read through the condition’s base is nonzero, initialization is skipped.

If that handle is zero:

1. The second input is written into the temporary local word.
2. F-3ff060c344dc is called without explicit arguments.
3. A zero result causes an immediate null return.

That immediate return precedes both notifications, lookup, allocation, and the final failure reporter.

A nonzero initialization result permits execution to continue. There is no explicit post-initialization check that any handle is nonzero or usable.

In particular, successful initialization does not cause the function to recapture the global context base. Later handle reads still use the base saved before the gate.

### 3. Prepare the descriptor and issue the opening notification

After the gate permits continuation, the descriptor’s first word is written with the first input. This overwrites the temporary second-input value if the initialization branch ran.

The displayed local layout then contains three consecutive 32-bit words:

| Relative byte offset | Stored input |
|---|---|
| 0 | First |
| 4 | Third |
| 8 | Fourth |

The address of the first word is supplied to lookup. This establishes the displayed storage available at that address, not the number of bytes lookup reads.

F-890414d1fd99 is called with five arguments, in order:

1. 9
2. 2
3. 0
4. 0
5. First input

### 4. Look up an existing record

After the opening notification returns, the function reads the 32-bit handle at offset 8 through the saved context base.

It calls F-932a35189ea0 with:

1. That handle.
2. The address of the descriptor’s first word.

The handle is read after the opening notification. A change that notification makes to the saved context’s handle field can therefore affect the value supplied to lookup. A change only to the global context-base location does not, by itself, replace the saved base used here.

If lookup returns nonnull, that address is selected as the return value. Allocation, creation-value generation, release, and registration are skipped.

### 5. Attempt construction after a lookup miss

If lookup returns null, F-43da6b23bd44 is called with:

1. 12
2. 0
3. 0

If it returns null, the selected result remains null. There are no record-field writes and no calls to F-c141a8d8555e, F-5990a26eaaa3, or F-dbd019788d3d.

If allocation returns a nonnull address, the function performs this sequence:

1. Writes zero at byte offset 8.
2. Writes the first input at byte offset 0.
3. Calls F-c141a8d8555e without explicit arguments.
4. Writes its 32-bit result at byte offset 4.
5. Tests that result for equality to zero.

The first and last words are therefore written before the creation-value call; the middle word is written afterward. The allocation address is not an explicit argument to that call. Whether the dependency can nevertheless observe or affect the allocation is unknown.

The test distinguishes zero from every other 32-bit pattern. There is no positive-only requirement or separate rejection of values with the high bit set.

#### Zero creation value

If F-c141a8d8555e returns zero:

- Zero is written at offset 4 before cleanup is called.
- F-5990a26eaaa3 receives the allocation address.
- Registration is skipped.
- The selected result remains null.

The call is consistent with releasing the allocation, but actual reclamation and any wider cleanup are dependency behavior, not established here.

#### Nonzero creation value

If F-c141a8d8555e returns nonzero, the function reads offset 8 through the saved context base again and calls F-dbd019788d3d with:

1. The newly read handle.
2. The allocation address.

After that dependency returns normally, the allocation address becomes the selected result.

No result from registration is inspected. There is no explicit success check, rollback, or release after this call. The function does not verify that the record remains allocated or unchanged.

### 6. Issue the closing notification and report a null result

Every normally returning path that reaches the opening notification also reaches a closing call to F-890414d1fd99 with:

1. 10
2. 2
3. 0
4. 0
5. First input

This occurs after lookup and any construction, cleanup, or registration.

After the closing notification returns, a null selected result causes a call to F-4fc89ec41a6d with:

1. 15
2. 105
3. 65
4. 0
5. 0

The function then returns the selected result, provided the reporter returns normally.

Initialization failure is different: it returns null before the notification pair and does not invoke this reporter.

## Side effects

### Direct reads and writes

The function directly:

- References the global context-base location to save a base, then references it again for the initialization condition.
- Reads the condition’s offset-8 handle.
- Reads offset 8 through the saved base before lookup and, when reached, before registration.
- Writes local input and descriptor storage.
- Writes the three fields of a nonnull new allocation in the order described above.

It performs no explicit write to the global context-base location or either context’s handle field. Dependencies may modify them; preservation is not established.

### Call ordering

On an existing-record path, the main sequence is:

Opening notification → saved-base handle read → lookup → closing notification → return.

On a successful construction path:

Opening notification → saved-base handle read → lookup → allocation → offset-8 write → offset-0 write → creation-value call → offset-4 write → saved-base handle read → registration → closing notification → return.

For a zero creation value, cleanup replaces registration, and failure reporting follows the closing notification.

The optional initialization call precedes all these sequences. The notification pair is not guaranteed to complete if an intervening dependency faults or fails to return normally.

### Ownership and lifetime

The only explicit cleanup call receives a newly allocated record whose creation value is zero.

On the nonzero path, the record is supplied to registration and its address is subsequently returned. Whether registration borrows, retains, owns, mutates, or releases it is unknown.

Ownership of lookup results is likewise unknown. Neither return path independently authorizes caller-side release.

The lookup descriptor occupies temporary local storage. Its availability for the call is established. Permission to retain its address and dereference it after this function returns is not established.

## Dependencies

| Opaque ID | Established call-site role | Not established |
|---|---|---|
| F-3ff060c344dc | Called without explicit arguments when the condition’s handle is zero; zero return aborts immediately. | Initialization effects, implicit inputs, context replacement, handle guarantees. |
| F-890414d1fd99 | Receives the opening and closing argument sets. | Locking, tracing, mutation, lifetime effects, failure behavior. |
| F-932a35189ea0 | Receives a saved-base handle and descriptor address; nonnull return is selected. | Matching rules, read extent, uniqueness, ownership, mutation. |
| F-43da6b23bd44 | Receives 12, 0, 0; nonnull return is used for three word writes. | Allocation policy, alignment guarantees, ownership details. |
| F-c141a8d8555e | Supplies a 32-bit value stored in the middle field and tested against zero. | Value meaning, resource acquisition, implicit access, global effects. |
| F-5990a26eaaa3 | Receives the allocation when the creation value is zero. | Actual release and broader cleanup behavior. |
| F-dbd019788d3d | Receives a freshly read saved-base handle and the new record. | Acceptance, failure signaling, ownership transfer, mutation, release. |
| F-4fc89ec41a6d | Receives fixed arguments after a null main-path result. | Reporting destination, termination, other state changes. |

These are call-site roles, not verified dependency contracts.

## Acceptance examples

These are conditional consequences of static control flow, not executed tests. They assume valid required memory and normal dependency returns unless stated otherwise.

### Existing record, including a zero first input

Given a nonzero condition handle and lookup result A:

- Initialization is skipped.
- Lookup follows the opening notification and receives a handle read through the saved base.
- No allocation or construction dependency runs.
- The closing notification occurs.
- A is returned without failure reporting.

A zero first input does not change this branch selection.

### Initialization failure

Given a zero condition handle and a zero initialization result:

- The second input is temporarily stored before initialization.
- The function returns null.
- Neither notification, lookup, allocation, nor final failure reporting occurs.

### Successful initialization with a zero later handle

Given a zero condition handle, nonzero initialization result, and a zero handle at the saved base when lookup is reached:

- Execution continues.
- Lookup receives zero.
- There is no additional local rejection of that handle.

Whether lookup accepts it is **documented_only**.

### Saved base differs from condition base

If the source-level saved base is A and the subsequent global reference used by the condition yields B:

- Initialization gating uses the offset-8 field through B.
- Later lookup and registration handle reads use A.
- No equality check reconciles them.

This case illustrates the source-level distinction. It does not assert a machine-level reload pattern or a guaranteed concurrent execution scenario.

### Allocation failure

Given a lookup miss and null allocation result:

- No record fields are written.
- Creation-value generation, cleanup, and registration are skipped.
- Closing notification precedes failure reporting.
- The return is null.

### Zero creation value

Given allocation address A and creation value zero:

- Offset 8 is written before offset 0.
- Offset 4 receives zero after the creation-value call.
- Cleanup receives A.
- Registration is skipped.
- Closing notification and failure reporting follow.
- The return is null.

### Nonzero creation value

Given allocation address A and any nonzero 32-bit creation value V:

- The function writes the first input at offset 0, V at offset 4, and zero at offset 8.
- Registration receives A and a newly read handle through the saved base.
- Its result is not checked.
- After registration and closing notification return, A is returned.

The field values are not guaranteed to survive dependency mutations.

### Context changes during calls

If the opening notification changes the saved context’s handle field, lookup uses its subsequently read value. If a later dependency changes that field before registration, registration uses the later value.

If initialization or another dependency instead changes only the global context-base location, subsequent handle reads still use the previously saved base.

## Uncertainties

The following remain **documented_only**:

- Business meanings of the inputs, record fields, notification codes, and reporting arguments.
- Whether lookup reads only the first descriptor word or also adjacent words.
- Whether temporary storage of the second input supplies an implicit initialization input.
- Whether initialization produces a usable context or handle.
- Lookup matching, uniqueness, and concurrency behavior.
- Synchronization or transactional effects of notifications.
- Registration success and how failure might be communicated.
- Ownership, validity, and permitted release of returned records.
- Dependency behavior on invalid handles, faults, memory exhaustion, or nonlocal exits.
- Machine-level realization of the two source-level global references.

The visible operations use 32-bit words, fixed offsets 0, 4, and 8, and a fixed 12-byte request. No input-derived size calculation or multiplication is present. Pointer validity, address-offset validity, alignment, and object bounds are preconditions rather than checked properties.

## Evidence

Established claims derive from the supplied static body: source-level reference order, branch conditions, fixed arguments, field widths and offsets, direct writes, and dependency call order.

The saved-base capture and the condition’s separate global reference are distinct in that evidence. Lookup and registration explicitly use the saved base. No equality check or later base recapture is shown. This supports the access distinction without proving separate machine loads or concurrency semantics.

Lookup-or-create intent and the allocation, cleanup, and registration descriptions are interpretations of call-site use. They do not establish dependency internals or ownership contracts.

No runtime tests have been performed. The acceptance examples are conditional static expectations, not observed executions.
