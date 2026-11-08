<!-- Trusted provenance: F-eeacd77103f9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eeacd77103f9

## Purpose

F-eeacd77103f9 coordinates a staged interaction that appears to change a passkey and then alter the presentation of a lock dialog. That purpose is inferred from the supplied message identifiers; the established behavior is a state-dependent sequence of dependency calls, object-field updates, message dispatch, and temporary-value cleanup.

The function distinguishes three active states, numbered 0, 1, and 2. Other state-byte values cause no branch-specific action. An error path explicitly assigns state 3.

Every path that returns normally returns the 32-bit value 1. This value does not distinguish successful progression, rejected input, deferred processing, unsupported state, or a reported combination error.

## Interface

### Inputs

The function has four arguments:

1. **Object reference.** A reference to mutable storage with the layout described below. The function also uses this reference as the receiver for an indirect method call and for F-979606fd565e.
2. **Unused 32-bit argument.** The supplied representation does not read or forward this argument.
3. **32-bit contextual value.** Copied into a local slot. Its address is passed to F-9b9c1c7483f4 only on the combination-error path.
4. **32-bit temporary seed.** Copied into a separate local slot. That slot is passed by address to F-9b9c1c7483f4 and, subsequently, to F-8a5aadac202f.

Arguments 3 and 4 have no established application-level type. They may represent scalar values, references, or runtime-managed values. Their roles must not be narrowed without a contract for the dependencies that receive their addresses.

### Referenced layout

Offsets are byte offsets from the object reference:

| Offset | Represented field | Use |
|---|---|---|
| 0x00 | Dispatch-table reference | Used to resolve the indirect message-dispatch method. |
| 0xB4 | 32-bit associated-object value | Supplied to several dependencies; also treated as a reference in state 2. |
| 0xBC | State byte | Selects the active branch; explicitly assigned 1, 2, or 3 on particular paths. |
| 0xBD | Auxiliary byte | Assigned 0 after the state-0 check or 1 after the state-1 check. |

The initial state expression selects a 32-bit field at offset 0xBC and converts it to a character-sized value. Branch selection therefore depends on the low eight bits, not on the entire 32-bit field. The explicit state assignments are byte writes at 0xBC and do not themselves overwrite the adjacent byte at 0xBD.

In state 2, the associated-object value is used to access a 32-bit unsigned field at offset 0x9C from that associated object.

The indirect dispatch entry is selected at byte offset 0x11C within the dispatch table. It receives the original object reference and a 32-bit result from F-9b9c1c7483f4. Its return value, if any, is ignored.

### Output and preconditions

Normal return is always the 32-bit value 1.

The function performs no explicit null, bounds, lifetime, alignment, or dispatch-target validation. Valid storage and callable dependencies are preconditions for any path that accesses them. Not every path dereferences the associated-object value: the explicit access to its field at 0x9C occurs only in state 2.

This is a 32-bit represented interface. Pointer arithmetic, field widths, and address conversions should not be silently reinterpreted as native 64-bit layouts.

## Behavior

### Entry and branch selection

At entry, the function reads a global 32-bit datum and retains its value locally. That entry-time value is used only if execution later reaches the combination-error notification.

The function then obtains the state byte from offset 0xBC and prepares local copies of arguments 3 and 4. Branch selection uses this initially obtained state. There is no subsequent state-field read to select a different branch, even if a dependency changes the object.

For message selection, this contract uses the following descriptive labels:

- **New-passkey message**
- **Confirm-passkey message**
- **Incorrect-combination message**
- **Pop-lock-dialog message**
- **Push-lock-dialog message**

These labels identify the distinct supplied static message references. They do not assert exact string contents, encoding, or command syntax.

### State 0: initial check

The function reads the associated-object value at offset 0xB4 and passes it to F-2d525ad4ccd5.

After that dependency returns, the function writes 0 to the auxiliary byte at offset 0xBD. This write occurs for every normally returning result from that dependency, including a result that causes an immediate return.

The result controls the next action:

- **Result 0:** Write state 1 at offset 0xBC, select the new-passkey message, and perform common message dispatch.
- **Result 3:** Return 1 without explicit message dispatch, cleanup, or notification.
- **Any other nonzero result:** Execute the combination-error path.

The result comparison is against the full represented integer value. Negative results, if possible under the dependency’s contract, belong to the last category rather than to a separate error class.

### State 1: new-passkey check

The function reads the associated-object value at offset 0xB4 and passes it to F-96b51f1d77df.

After that dependency returns, the function writes 1 to the auxiliary byte at offset 0xBD.

The result controls the next action:

- **Result 0:** Write state 2 at offset 0xBC, select the confirm-passkey message, and perform common message dispatch.
- **Any nonzero result:** Return 1 without explicit message dispatch, cleanup, state advancement, or notification.

Result 3 is explicitly recognized in the supplied representation, but its externally visible path is the same as that of every other nonzero result in this state.

Unlike state 0, a nonzero result other than 3 does not lead to the combination-error path.

### State 2: confirmation status and dialog selection

The function first reads the associated-object value at offset 0xB4, dereferences its unsigned 32-bit field at offset 0x9C, and saves that field value locally.

It then calls F-c4a767154623 with no explicit arguments.

The returned integer selects the next action:

- **Result 2:** Continue with the success sequence described below.
- **Result 4:** Execute the combination-error path.
- **Any other result:** Return 1 without further explicit action.

The read at associated-object offset 0x9C occurs before the status call, including when that call later returns 4 or an otherwise inactive status.

For result 2, the function reads the associated-object value from offset 0xB4 again and passes that current value to F-6b4015338d43. This second read is significant: the status dependency may have changed the object’s associated-object field. The function does not necessarily pass the same value used for the earlier field read.

After F-6b4015338d43 returns, the saved field value determines message selection:

- If its low eight bits are zero, select the pop-lock-dialog message.
- Otherwise, select the push-lock-dialog message.

Upper bits do not affect this decision. For example, a saved value of 0x00000100 selects the pop message, while 0x00000101 selects the push message.

The decision uses the saved value obtained before the status call. It is not recomputed after either dependency.

The selected message is then handled by common message dispatch. This path does not explicitly update either the state byte or the auxiliary byte.

### Other initial states

If the initial state byte is neither 0, 1, nor 2, the function returns 1.

Apart from entry-time reads and local argument copies, it performs no explicit dependency calls or object writes on this path. State 3 is included in this inactive category.

### Common message dispatch

For the new-passkey, confirm-passkey, pop-lock-dialog, and push-lock-dialog messages, the function performs these externally relevant operations in order:

1. Call F-9b9c1c7483f4 with the address of the temporary slot initialized from argument 4 and the selected static message reference.
2. Resolve the object’s current dispatch table and its method entry at offset 0x11C.
3. Invoke that method with the original object reference and the result returned by F-9b9c1c7483f4.
4. Call F-8a5aadac202f with the address of the same temporary slot.
5. Return 1.

The dispatch target is obtained after message construction. A dependency-induced change to the object’s dispatch table can therefore affect the target invoked.

State advancement on the state-0 and state-1 success paths occurs before message construction and dispatch. Cleanup occurs after dispatch, not before it.

### Combination-error path

This path is reached only through:

- State 0 when F-2d525ad4ccd5 returns a value other than 0 or 3.
- State 2 when F-c4a767154623 returns 4.

The sequence is:

1. Call F-9b9c1c7483f4 with the address of the temporary slot initialized from argument 4, the incorrect-combination message reference, and the address of the contextual slot initialized from argument 3.
2. Resolve and invoke the indirect dispatch method at offset 0x11C, passing the original object reference and the constructor result.
3. Call F-8a5aadac202f on the temporary slot.
4. Write state 3 at offset 0xBC.
5. Call F-979606fd565e with the original object reference and the global datum saved at entry.
6. Return 1.

The incorrect-combination dispatch and cleanup occur before the explicit state-3 assignment. The notification dependency runs after that assignment.

The notification uses the saved global value even if an intervening dependency changes the global.

## Side effects

The function’s own explicit persistent writes are limited to:

- Auxiliary byte 0 in state 0, after F-2d525ad4ccd5 returns.
- Auxiliary byte 1 in state 1, after F-96b51f1d77df returns.
- State byte 1 on state-0 success.
- State byte 2 on state-1 success.
- State byte 3 after combination-error dispatch and cleanup.

These are not guarantees about final object contents. Dependencies and the indirect dispatch method may perform additional writes, including writes to the same fields. For example, F-979606fd565e runs after the explicit state-3 assignment and might change that state again.

No direct allocation or deallocation is established. The constructor-like and cleanup-like dependencies receive the address of the same local slot, but this does not prove ownership transfer, reference-counting behavior, allocation, or release semantics.

The returned message value is forwarded to the indirect method. No separate cleanup of that returned value is visible. Whether the returned value aliases the local slot’s contents or represents separately owned storage is unknown.

The two local-slot addresses are usable during synchronous calls. No evidence establishes whether a dependency retains either address or whether such retention would be valid.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-2d525ad4ccd5 | State-0 check using the associated-object value; result controls progression, early return, or combination-error reporting. |
| F-96b51f1d77df | State-1 check using the associated-object value; only result 0 advances the interaction. |
| F-c4a767154623 | No-argument status query in state 2; results 2 and 4 select active paths. |
| F-6b4015338d43 | Called with the freshly read associated-object value after state-2 status 2 and before dialog-message construction. No return value is used. |
| F-9b9c1c7483f4 | Produces the value forwarded to message dispatch and receives a local slot by address. The error invocation additionally receives the contextual slot’s address. |
| F-8a5aadac202f | Called on the temporary slot after each completed explicit dispatch. Its exact cleanup semantics are unknown. |
| F-979606fd565e | Called after the explicit state-3 assignment with the object reference and entry-time global value. No return value is used. |

The indirect dispatch method has no supplied opaque function ID. Its target is selected dynamically through the object’s dispatch table.

No dependency is assumed to preserve globals, preserve object fields, return normally, or acquire or release memory ownership.

## Acceptance examples

These examples are static acceptance conditions, not executed tests. Dependency outcomes and mutations must be controlled or observed before asserting a final state.

1. **State 0, check result 0.**  
   The auxiliary byte is written to 0, then the state byte to 1. The new-passkey message is constructed, dispatched, and followed by temporary-slot cleanup. The function returns 1 if those calls return normally.

2. **State 0, check result 3.**  
   The auxiliary byte is still written to 0. The function returns 1 without its own state write, message construction, dispatch, cleanup, or notification.

3. **State 0, check result −1.**  
   If the dependency can return that represented integer, it takes the combination-error path. Cleanup precedes the explicit state-3 write; notification follows it.

4. **State 1, check result 4.**  
   The auxiliary byte is written to 1 and the function returns 1. It does not report the incorrect-combination message merely because the result is nonzero.

5. **State 1, check result 0.**  
   The state is explicitly advanced to 2 before confirm-passkey message construction. A dispatch observer therefore sees that write unless a dependency has subsequently changed the field.

6. **State 2, saved field 0x00000100, status 2.**  
   F-6b4015338d43 is called, then the pop-lock-dialog message is selected because only the low byte participates.

7. **State 2, associated-object replacement during status query.**  
   If F-c4a767154623 replaces the associated-object field and returns 2, F-6b4015338d43 receives the replacement value. Message selection still uses the field captured from the earlier associated object.

8. **State 2, status 4.**  
   The associated-object field at 0x9C has already been read. F-6b4015338d43 is not called. The combination-error sequence executes.

9. **Initial state byte 3 or 255.**  
   Neither value selects an active branch. The function returns 1 without explicit object writes or dependency calls.

10. **Global changes during error-message dispatch.**  
    F-979606fd565e receives the entry-time global value, not the changed value.

## Uncertainties

The following are **documented_only**:

- Exact message bytes, encoding, localization behavior, and whether dispatch interprets them as text or commands.
- The meaning and provenance of the global datum.
- Application-level meanings of dependency result codes.
- Ownership and lifetime rules for constructed values and the two address-passed local slots.
- Dependency mutations, callbacks, reentrancy, and nonlocal failure behavior.
- Calling-convention details beyond the represented arguments and 32-bit values.
- Actual machine access width for the initial state read; the supplied representation uses a wider field expression followed by character conversion.
- Thread safety and synchronization. None is shown.
- Behavior after invalid references, invalid dispatch targets, or dependency failures. No recovery path is established.

The passkey-workflow interpretation is inferred intent. The branch conditions, explicit writes, call arguments, and sequencing are established static evidence.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

The representation establishes three active state branches, byte-sized explicit state and auxiliary writes, a low-byte dialog-selection condition, two message-construction call shapes, an indirect dispatch followed by cleanup, and an unconditional normal return value of 1.

No implementation of a dependency, dispatch target, message body, or global initializer was supplied. No runtime tests have been performed.
