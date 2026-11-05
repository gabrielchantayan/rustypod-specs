<!-- Trusted provenance: F-2e2c5aa7c45d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2e2c5aa7c45d

## Purpose

F-2e2c5aa7c45d conditionally dispatches through an object referenced by its first input. Its result has three distinct origins:

- Zero when the referenced object is initially absent.
- The unmodified 32-bit result of a fallback dispatch when three preceding dispatches all return zero.
- A normalized zero-or-one result derived from F-6d70e45e29c9 when any of those three preceding dispatches returns nonzero.

The function therefore does **not** have an established uniformly Boolean return contract. Only the branch involving F-6d70e45e29c9 is visibly normalized.

The apparent intent is to choose between a direct fallback operation and an operation involving temporary state. The meanings of the three selection operations, the fallback operation, and the temporary state are unknown. This contract does not assign them domain-specific meanings.

## Interface

### Inputs

The visible interface has four inputs, each represented by a 32-bit value:

| Input | Established use |
|---|---|
| First input | Used as an address from which an object-reference field is read at byte offset `0x1c`. |
| Second input | No explicit read or forwarding is visible. |
| Third input | Copied into the first word of temporary storage when the initially referenced object is non-null. |
| Fourth input | Copied into the second word of that temporary storage on the same path. |

The third and fourth inputs are opaque 32-bit values. Their signedness, encoding, and possible interpretation as addresses are not established.

The lack of a visible use for the second input does not establish its role in a broader calling convention. It establishes only that the displayed body contains no explicit operation using it.

### Result

The function returns a 32-bit integer value.

Its interpretation depends on the selected path:

- Initial null object reference: zero.
- Three zero selection results: the fallback dispatch’s complete result, without visible normalization.
- Any nonzero selection result: one if F-6d70e45e29c9 returns unsigned zero; otherwise zero, assuming the subsequent finalization dependency returns normally.

No separate error output is visible.

### Required memory layout

For a non-faulting ordinary execution, the first input must identify memory supporting a readable 32-bit field at byte offset `0x1c`.

A non-null value read from that field is treated as an object address. The first 32-bit word at that address is treated as a dispatch-table address. The following dispatch-table offsets are used:

| Byte offset | Role in this function |
|---|---|
| `0xf0` | First selection dispatch |
| `0x110` | Second selection dispatch |
| `0x114` | Third selection dispatch |
| `0xec` | Fallback dispatch |
| `0x7c` | Temporary-state processing dispatch |

These offsets are byte offsets, not entry indices. The visible representation uses 32-bit object references and dispatch entries.

The temporary storage is an eight-byte region containing two adjacent 32-bit words. Its address is passed to three named dependencies and, on one path, to the processing dispatch.

### Preconditions and calling-convention limits

There is no visible check that the first input itself is null, valid, aligned, or accessible. An initially null object-reference field is handled; an invalid address for that field is not.

Every dispatch actually reached requires a readable object header, a readable corresponding dispatch entry, and a valid callable target. Because object references are re-read, these requirements apply to the values observed at each reached use, not merely to the object observed by the initial null check.

The complete calling convention of the indirect dispatches is not established. The processing dispatch visibly receives an object reference and the temporary-storage address. The other four indirect dispatches have no explicit arguments shown. Hidden receiver arguments, register-carried inputs, and other ABI details remain documented_only.

## Behavior

### Initial object check

The function begins with a prospective result of zero and reads the object-reference field at offset `0x1c` from the first input.

If that field contains zero:

- No selection dispatch is performed.
- No fallback or processing dispatch is performed.
- The temporary-storage dependencies are not called.
- The function returns zero.

The third and fourth inputs are not visibly copied into temporary storage on this path.

If the field is nonzero, the third and fourth inputs are copied, in that order, into the two words of temporary storage before the first selection dispatch.

### Ordered selection

The function evaluates up to three indirect dispatches in this order:

1. The entry at byte offset `0xf0`.
2. The entry at byte offset `0x110`, but only if the first result was zero.
3. The entry at byte offset `0x114`, but only if both preceding results were zero.

Each result is tested for exact equality to zero. Any nonzero 32-bit result, including a negative signed value, takes the temporary-state path.

Evaluation short-circuits:

- A nonzero first result prevents the second and third selection dispatches.
- A nonzero second result prevents the third selection dispatch.
- Only three zero results reach the fallback path.

The particular nonzero value does not otherwise affect the visible computation. It is not the final return value on the temporary-state path.

For each selection dispatch that is reached, the function obtains its target through the object-reference field and the object’s dispatch-table pointer. It does not visibly retain one object or one dispatch table for all selection operations.

Consequently, a preceding dispatch can affect which object or table supplies the next dispatch, if it changes the relevant reachable state.

### Fallback path

If all three selection results are zero, the function resolves and calls the entry at byte offset `0xec`.

The fallback result becomes the function’s return value without a visible zero test, range restriction, or Boolean conversion.

On this path:

- The temporary words have already received the third and fourth input values.
- F-5b24edfb22c0 is not called.
- The processing entry at offset `0x7c` is not called.
- F-6d70e45e29c9 is not called.
- F-289ec4311c64 is not called.

The presence of temporary copies does not establish that their contents require cleanup on this path. No such cleanup is visible.

### Temporary-state path

If any reached selection dispatch returns nonzero, the function performs these operations in order:

1. Calls F-5b24edfb22c0 with the temporary-storage address.
2. Resolves and calls the processing entry at byte offset `0x7c`, visibly supplying an object reference and the same temporary-storage address.
3. Calls F-6d70e45e29c9 with that address.
4. Converts that dependency’s unsigned 32-bit result into the final zero-or-one value.
5. Calls F-289ec4311c64 with the same address.
6. Returns the previously calculated result, provided the final call returns normally.

The return values of F-5b24edfb22c0, the processing dispatch, and F-289ec4311c64 are not visibly used.

There is no visible reconstruction of the temporary words between these calls. Each dependency therefore receives the storage as left by preceding operations, subject to whatever behavior those dependencies actually have.

### Integer-width behavior

The value returned by F-6d70e45e29c9 is treated as an unsigned 32-bit quantity.

The visible computation subtracts that quantity from one and overrides the result with zero when the quantity exceeds one. Its final result is therefore:

| Unsigned dependency result | Function result |
|---|---|
| `0` | `1` |
| `1` | `0` |
| `2` through `4,294,967,295` | `0` |

The subtraction has 32-bit unsigned arithmetic semantics. For values greater than one, any wrapped intermediate value is replaced by zero before return. No wrapped intermediate value is passed to F-289ec4311c64 as an explicit argument.

This normalization is an exact zero test, not a signed positivity test. In particular, `0xffffffff` produces zero on this path.

## Side effects

### Direct memory activity

The visible body directly:

- Reads the field at offset `0x1c` from the first input.
- Re-reads that field for reached indirect dispatches.
- Reads object-header and dispatch-entry words needed to resolve those dispatches.
- Writes the third input into the first temporary word.
- Writes the fourth input into the second temporary word.

No direct write to the caller’s object, its dispatch table, or a global is visible.

This does not imply that the overall operation is read-only. Every reached callee may have side effects not established here.

### Re-reading and mutation

The initial non-null check is not a snapshot guarantee. Subsequent dispatch expressions re-read the object-reference field.

If a callee replaces that field with another valid object reference, later dispatches can use the replacement. If a callee clears it, a later dispatch can attempt to dereference zero; no second null guard is visible.

The processing dispatch also visibly uses the field to obtain its target and to supply an object argument. The source does not establish a total order between evaluating that target and evaluating its explicit arguments. No atomic snapshot, synchronization, or guarantee that independently evaluated reads observe the same object is established.

### Temporary storage and ownership

The two-word temporary region has local storage duration. Its address is exposed to callees on the temporary-state path.

The presence and placement of F-5b24edfb22c0 and F-289ec4311c64 are consistent with preparation and finalization, respectively, but those roles are inferred intent rather than established ownership semantics.

The visible body does not establish that either dependency:

- Allocates or releases memory.
- Takes ownership of either original input value.
- Preserves globals or object state.
- Retains or does not retain the temporary-storage address.
- Leaves either temporary word unchanged.

The function itself does not explicitly allocate heap memory or explicitly release the object referenced by the first input.

### Errors and abnormal completion

No explicit validation error, exception handler, retry, or recovery path is visible.

A nonzero selection result is a branch condition, not an established error code. Likewise, the meaning of the unsigned result from F-6d70e45e29c9 is unknown beyond its effect on normalization.

If a dependency faults, transfers control elsewhere, or does not return, the remaining operations are not guaranteed to occur. In particular, there is no visible guarantee that F-289ec4311c64 executes after abnormal completion of an earlier call.

## Dependencies

| Dependency | Established interaction |
|---|---|
| F-5b24edfb22c0 | Called once with the temporary-storage address after a nonzero selection result, before the processing dispatch. Its return value is ignored. |
| F-6d70e45e29c9 | Called once with the temporary-storage address after the processing dispatch. Its unsigned 32-bit result determines the normalized return value. |
| F-289ec4311c64 | Called once with the temporary-storage address after normalization. Its return value is ignored. |
| Unidentified indirect dependencies | Selected through dispatch-table offsets `0xf0`, `0x110`, `0x114`, `0xec`, and `0x7c`. Their opaque function IDs are unavailable. |

No semantic contract for these dependencies is supplied. Their names here identify call relationships, not guarantees about purity, ownership, or state preservation.

## Acceptance examples

These examples are conditional static acceptance criteria, not reports of executed tests. Unless stated otherwise, assume valid memory, valid call targets, and normally returning dependencies.

1. **Initially absent object.**  
   If the first input’s field at offset `0x1c` contains zero, the result is zero and no dependency is called.

2. **All selection results are zero.**  
   If the three selection dispatches each return zero and the fallback returns `-7`, the selection dispatches occur in their stated order, the fallback follows, and the final result is `-7`. None of the temporary-state dependencies is called.

3. **First selection succeeds in the nonzero sense.**  
   If the first selection returns `5`, the second and third selection dispatches and the fallback are skipped. If F-6d70e45e29c9 later returns zero, the final result is one after F-289ec4311c64 returns.

4. **Second selection is nonzero.**  
   If the first selection returns zero and the second returns `-1`, the third and fallback dispatches are skipped. If F-6d70e45e29c9 returns one, the final result is zero.

5. **Third selection is nonzero.**  
   If the first two selection results are zero and the third is nonzero, the temporary-state path runs. An unsigned F-6d70e45e29c9 result of two produces zero, not an exposed subtraction result.

6. **Maximum unsigned normalization input.**  
   If F-6d70e45e29c9 returns `4,294,967,295`, the final result is zero. The value is not interpreted as a negative number for the range check.

7. **Object replacement between selections.**  
   If the first selection returns zero after replacing the referenced object with another valid object, the second selection is resolved through the subsequently observed object, not necessarily the original one.

8. **State mutation before processing.**  
   If F-5b24edfb22c0 modifies the temporary words, the processing dispatch receives their address without the original third and fourth inputs being visibly recopied over those modifications.

9. **Abnormal finalization.**  
   If F-289ec4311c64 does not return, the fact that the normalized result was already calculated does not establish a normal return from F-2e2c5aa7c45d.

## Uncertainties

The following remain documented_only:

- The domain-level purpose of the object and each dispatch.
- The complete ABI and receiver convention of the indirect calls.
- The semantic role of the unused second input.
- Whether the temporary words form an established aggregate type.
- Whether any dependency allocates, retains, transfers, or releases resources.
- Whether callees mutate globals, the object-reference field, dispatch tables, or temporary storage.
- Any intended invariant keeping the object reference non-null after the initial check.
- Behavior under concurrent mutation, including alignment, atomicity, and synchronization guarantees.
- The meaning and valid range of fallback results.
- The meanings of zero and nonzero dependency results beyond the visible branch decisions.
- Unwinding behavior and cleanup guarantees outside the displayed body.

None of these unknowns is resolved by assigning conventional meanings to dispatch offsets or to call ordering.

## Evidence

This contract is based solely on static inspection of the supplied function representation and the supplied opaque dependency mapping.

Established evidence includes the field and dispatch offsets, the two adjacent 32-bit temporary words, the ordered short-circuit selection, the raw fallback return, the temporary-state call sequence, and unsigned normalization.

Inferred intent is limited to describing the temporary-state boundary calls as potentially preparatory and finalizing. Their resource semantics are not established.

No runtime tests have been performed. The acceptance examples describe behavior implied by the visible control flow under their stated conditions; they are not observed execution results.
