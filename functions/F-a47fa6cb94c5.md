<!-- Trusted provenance: F-a47fa6cb94c5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a47fa6cb94c5

## Purpose

F-a47fa6cb94c5 conditionally replaces occurrences of one unsigned 32-bit value with another in data associated with a caller-supplied object.

Two distinct mechanisms are visible:

- When a dependency-produced comparison indicates equal representations, the function traverses byte-addressed storage through cursor-based helpers and counts matching replacement operations.
- Otherwise, it requests temporary storage, obtains a sequence of 16-bit units, replaces matching units, and passes the sequence to an object-update dependency. This path returns zero even when substitutions occur.

The apparent intent is character replacement with separate handling for equal-width and different-width representations. That interpretation is inferred. The established behavior is defined by memory accesses, argument values, dependency results, and call ordering; no encoding or character model is established.

## Interface

### Inputs

| Input | Established representation | Role |
|---|---|---|
| Object address | Address carried in a 32-bit integer-shaped input | Identifies an object containing a pointer-sized field at byte offset 4. |
| Search value | Unsigned 32-bit integer | Compared directly with decoded values or zero-extended 16-bit units. |
| Replacement value | Unsigned 32-bit integer | Passed unchanged to the cursor-based write dependency; narrowed to 16 bits in the temporary-buffer path. |
| Additional pointer | Byte-addressed pointer | Initially assigned to local cursor storage, but overwritten before a meaningful cursor read in the visible paths. No direct dereference of this input is shown. |

The only established object layout is a 32-bit pointer-shaped field beginning at byte offset 4. No structure size, type tag, length field, ownership flag, or other member is established.

### Output

The return value is a signed 32-bit integer:

- Zero when the object’s pointer field is null.
- Zero when the search and replacement values are equal.
- On the cursor-based path, the number of times the decoded value matches the search value and the two replacement-related dependencies return normally.
- Zero throughout the temporary-buffer path, including allocation failure and successful processing.

The result is therefore **not a uniform count of changed values** and is not a general success indicator.

### Preconditions and validity requirements

The object address must permit reading its field at byte offset 4. There is no visible check for a null object address.

For an active operation, subsequent reads and writes must be valid for the selected path and for all dependencies. In particular:

- The cursor-based path requires accessible storage for byte terminator checks and whatever accesses its dependencies perform.
- The temporary-buffer path requires a valid, terminated sequence of 16-bit units after the conversion-like dependency returns.
- The requested allocation must be adequate for the conversion-like dependency’s actual output.

The function does not visibly establish these conditions itself. Violating them has no defined error result in the supplied evidence.

## Behavior

### Entry gate

The function initializes its result to zero and reads the object’s pointer field.

If that field is null, it returns zero without calling any dependency. Because the pointer-field test occurs first, even equal search and replacement values do not remove the requirement that the object field be readable.

If the pointer field is nonnull but the two unsigned 32-bit values are equal, the function also returns zero without calling any dependency. It does not inspect the pointed-to contents in this case.

The remainder applies only when the initially observed pointer is nonnull and the two values differ.

### Representation comparison and pointer reload

The function calls F-ff169115f82a with the search value and retains a 32-bit result.

It then calls the same dependency with the replacement value and the retained result as an additional argument. The supplied representation exposes a 64-bit result from this second call. The function compares its upper and lower 32-bit halves for equality.

This establishes an equality test on two 32-bit portions of a dependency result. It does not independently establish what those portions mean or that they are simply the respective encoded lengths of the inputs.

After these calls, the function reloads the object’s pointer field. Thus, the initial nonnull observation does not fix the pointer used later. Any changes made by either dependency can affect subsequent behavior.

Equal halves select the cursor-based path; unequal halves select the temporary-buffer path.

### Cursor-based path

Traversal starts from the reloaded pointer.

At each iteration, the function reads one byte at the current cursor:

- A zero byte ends traversal.
- A nonzero byte permits a call to F-d3922c7fe0ec, passing the address of the local cursor storage.

The unsigned 32-bit result of F-d3922c7fe0ec is compared directly with the full search value. The dependency can also change the cursor through the supplied address.

For a nonmatching result, traversal continues from the cursor left by that dependency.

For a matching result, the function performs these operations in order:

1. Calls F-13bcc08bc7a3 with the address of the same cursor storage.
2. Calls F-a0bacab96f08 with that address and the full unsigned 32-bit replacement value.
3. Increments the signed 32-bit result count.
4. Continues traversal from the cursor left after these calls.

The count records matching helper sequences, not verified byte changes. The caller does not compare storage before and after replacement, inspect helper status values, or confirm that the replacement is represented as intended.

Cursor advancement is delegated. The visible function contains no independent step to guarantee progress. If the dependencies leave the cursor at the same nonzero byte, traversal can repeat indefinitely. If they move it outside accessible storage or make it null, a later byte read may be invalid. There is no visible iteration limit or recovery.

The byte terminator check happens before decoding. Consequently, the first zero byte ends this path without invoking the decoder at that position. Whether the decoder can return zero for a nonzero leading byte is unknown.

The result counter uses signed 32-bit arithmetic. No overflow check is shown. Behavior after exceeding the signed range is not established; wrapping must not be assumed.

### Temporary-buffer path

The function calls F-d8cbf9aa79a9 with no explicit arguments shown and interprets its result as a signed 32-bit integer.

It forms an allocation request by adding one and multiplying by two, then passes that request to F-fab3799b3190. This is consistent with reserving one additional 16-bit unit, but the meaning of the dependency’s result is unknown.

No check is visible for a negative result, arithmetic overflow, an implausible size, or consistency with later output. Both the addition and multiplication are signed 32-bit operations in the supplied representation. A mathematical size outside that range does not have an established wrapped interpretation.

If F-fab3799b3190 returns null, processing ends and the function returns zero. The conversion-like, object-update, and final temporary-pointer dependencies are not called.

If allocation returns a nonnull pointer, the function reloads the object’s pointer field and calls F-1e32f7e1be72 with:

- The allocated pointer as the first argument.
- The newly loaded object field as the second argument.

This reload occurs after both the sizing-like call and the allocation call. The conversion input therefore need not equal either earlier observation of the object field.

After F-1e32f7e1be72 returns, the function traverses the allocated storage in 16-bit units, advancing by two bytes per unit:

- A zero unit terminates the scan.
- Each nonzero unit is compared, after zero extension, with the full unsigned 32-bit search value.
- A matching unit is overwritten with the low 16 bits of the replacement value.

Consequences of these widths include:

- A search value above 65,535 cannot match any scanned unit.
- A search value of zero cannot match because zero ends traversal before the match comparison.
- A replacement value above 65,535 is truncated on a match.
- A nonzero replacement value whose low 16 bits are zero writes a terminator into the sequence.

Writing zero does not immediately end the scan: after that write, the cursor advances to the following unit. Later original units can still be processed before an existing zero unit is encountered. What the later object-update dependency does with the newly inserted zero is unknown.

After scanning, the function calls F-a6afbdf5375f with the object address and the allocated pointer. It then calls F-58ba61416cc9 with that same allocated pointer.

These calls also occur when the sequence is empty or no unit matches. There is no visible optimization that skips the object-update call for unchanged content.

The return count is never incremented on this path and remains zero.

## Side effects

### Established local reads and writes

The function directly:

- Reads the object field at byte offset 4 before the early-exit decision.
- Reloads that field after the representation-related calls.
- On the cursor-based path, reads a byte at each current cursor before invoking decoding.
- On the temporary-buffer path, reloads the object field after the allocation call succeeds.
- Reads the temporary sequence as 16-bit units and writes matching units as 16-bit values.
- Updates local cursor storage and a local result counter.

No direct assignment to the object’s pointer field is shown. Changes to the object or its original storage may nevertheless occur through dependencies.

### Dependency ordering and shared state

Dependency calls are ordered as described above. There is no basis for assuming that they preserve global state, object fields, allocated memory, or the original storage.

In particular, pointer reloads are observable: a dependency that changes the object field may change the storage used by a later step. The initial nonnull check is not repeated after those changes.

No rollback or transaction boundary is visible. The cursor-based path can perform earlier replacement operations before a later failure. The temporary-buffer path can modify its buffer before calling the object-update dependency, whose failure behavior is unknown.

### Ownership and lifetime

The temporary buffer is obtained from F-fab3799b3190, passed to F-1e32f7e1be72, edited locally, passed to F-a6afbdf5375f, and finally passed to F-58ba61416cc9.

The final call resembles a release operation, but actual deallocation is not established. Likewise, the object-update dependency is not known to copy, retain, consume, or release the supplied storage.

This function performs no further visible access through the temporary pointer after the final call. Safe interaction among those dependencies’ ownership rules is a prerequisite, not a proven property.

## Dependencies

| Opaque ID | Established interaction | Not established |
|---|---|---|
| F-ff169115f82a | Called first with the search value; then with the replacement value and the first result. Two 32-bit portions of the second result select the path. | Representation metric, purity, full calling convention, or meaning of the result portions. |
| F-d3922c7fe0ec | Receives writable cursor storage and returns an unsigned 32-bit value. | Encoding, progress, validation, and error behavior. |
| F-13bcc08bc7a3 | Called on matching decoded values with writable cursor storage, before the write-like dependency. | Whether it retreats, removes data, or otherwise changes memory. |
| F-a0bacab96f08 | Receives writable cursor storage and the full replacement value. | Encoding, capacity requirements, write extent, and cursor position afterward. |
| F-d8cbf9aa79a9 | Called without explicit arguments; its signed 32-bit result determines the allocation expression. | What is measured and which state supplies the measurement. |
| F-fab3799b3190 | Receives the computed allocation request; a null result is checked. | Allocation policy, initialization, ownership rules, and size interpretation outside valid arithmetic. |
| F-1e32f7e1be72 | Receives the allocated pointer and a freshly read object-field value. | Output bounds, terminator guarantee, conversion rules, and error signaling. |
| F-a6afbdf5375f | Receives the object address and edited temporary sequence. | Copying, retention, consumption, failure behavior, and object changes. |
| F-58ba61416cc9 | Receives the allocated pointer after the object-update call. | Whether it releases memory and what ownership preconditions it requires. |

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests.

1. **Null stored pointer.** Given a readable object whose field at byte offset 4 is null, the result is zero and no dependency is called.

2. **Equal values.** Given a readable object with a nonnull field and equal search and replacement values, the result is zero and no dependency is called, regardless of the pointed-to contents.

3. **Empty cursor-based input.** If the representation comparison selects the cursor-based path and the reloaded pointer addresses a zero byte, the result is zero. No decoding or replacement helper is called.

4. **Two decoded matches.** If that path decodes three values, the first and third match, all helpers return normally, cursor changes permit termination, and the count remains representable, the result is two. Each match calls F-13bcc08bc7a3 before F-a0bacab96f08.

5. **Allocation failure.** If the temporary-buffer path is selected and allocation returns null, the result is zero. Neither conversion, object update, nor the final temporary-pointer call occurs.

6. **Successful temporary substitutions.** Suppose conversion produces the units 65, 66, 65, zero; search is 65 and replacement is 67. The locally edited sequence is 67, 66, 67, zero when passed to F-a6afbdf5375f. The result is still zero.

7. **Search outside the unit range.** If search is 65,601, no 16-bit unit can match it, including a unit equal to 65. After successful allocation and conversion, the object-update and final temporary-pointer calls still occur.

8. **Replacement truncation.** If search is 65 and replacement is 65,602, a matching temporary-buffer unit is written as 66.

9. **Inserted terminator with continued scanning.** If converted units are 65, 66, 65, zero, search is 65, and replacement is 65,536, the edited units are zero, 66, zero, zero. Local traversal still reaches the third unit. The object-update dependency’s interpretation of that sequence remains unspecified.

10. **Changed object field.** If a representation-related dependency changes the object field before the post-call reload, cursor-based traversal uses the reloaded value, not necessarily the pointer observed at entry.

## Uncertainties

The following remain **documented_only**:

- The object’s full type and layout, and the semantic role of its pointer field.
- The encoding and the exact meaning of the representation comparison.
- Whether the exposed 64-bit result corresponds to a conventional return value or a multi-part calling-convention result.
- All dependency guarantees concerning errors, globals, ownership, termination, bounds, and cursor progress.
- The state measured by F-d8cbf9aa79a9, which receives no explicit object or pointer argument.
- Whether the additional pointer input has any purpose outside this visible body.
- Behavior after invalid memory access, unterminated data, signed overflow, inadequate allocation, or incompatible ownership.
- Whether an object-update call succeeds; no status is inspected.

Equal-representation in-place editing and alternate-representation reconstruction are inferred intent, not additional guarantees.

## Evidence

This contract is based solely on the supplied static function body and the supplied opaque dependency mapping.

Established claims cover visible input widths, object-field offsets, branch conditions, dependency arguments and ordering, cursor and buffer access widths, return-count updates, and unchecked arithmetic.

No dependency implementation, external documentation, runtime execution, or test result was available. No runtime tests have been performed.
