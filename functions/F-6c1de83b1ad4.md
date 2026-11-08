<!-- Trusted provenance: F-6c1de83b1ad4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6c1de83b1ad4

## Purpose

F-6c1de83b1ad4 performs a mode-dependent update to an object, conditionally notifies an associated object, and conditionally propagates an update through related objects.

The established sequence is:

1. Apply the zero-mode or nonzero-mode transition.
2. Perform an additional predicate-controlled notification in zero mode.
3. Notify an associated object if its backlink matches the input object.
4. Qualify propagation using the input object’s flags or a captured related object’s flags.
5. If qualified, update flags, invoke an update dependency, and perform further virtual dispatches.

“Enable,” “disable,” “release,” and “refresh” are possible interpretations, not established semantics. This contract uses structural descriptions instead.

## Interface

### Inputs and result

The function accepts:

- **Input object:** an address of structured, writable storage.
- **Mode:** an unsigned 32-bit integer.

It returns no value. Observable results are memory changes and dependency calls.

Mode zero selects the zero-mode transition. Every value from 1 through `0xFFFFFFFF` selects the nonzero-mode transition. Where the mode controls associated-object notification, it is normalized to 1 for zero mode and 0 for every nonzero mode.

### Represented widths and object layout

The supplied representation uses 32-bit integers and 32-bit pointer-bearing fields. Offsets below are byte offsets from the input object.

| Offset | Access width or extent | Role |
|---|---:|---|
| `0x00` | 32 bits | Virtual-table address |
| `0x34` | 32 bits | Related-object address |
| `0x3C` | 32 bits | Associated-object address |
| `0x48` | 32 bits | Flags |
| `0x80` | Address within object | Embedded address passed to F-6d16652f2d03 |
| `0xA0` | 32-bit read, then narrowing | Gate for related-object dispatch |
| `0xAC` | 32 bits | Optional held-object address |
| `0xB4` | 8 bits | Mode-dependent state |
| `0xB5` | 8 bits | State cleared in both modes |
| `0xB8` | 8 bits | State cleared in zero mode |
| `0xB9` | 8 bits | State cleared in zero mode |
| `0xBA` | 8 bits | State cleared in zero mode |

The gate at `0xA0` reads a **32-bit word**, converts that value to an 8-bit character value under the represented model, and compares the narrowed result with zero. Its decision therefore tests whether the word’s **low-order eight bits** are zero.

This is neither a direct byte read at `0xA0` nor a whole-word zero test. No target byte order is established, so the low-order eight bits must not be identified with the first byte in memory. The full word must be readable whenever this gate is reached.

The state writes at `0xB4` through `0xBA`, by contrast, are individual byte stores. They do not directly overwrite adjacent bytes.

Other accessed layouts are:

| Object | Offset | Access |
|---|---:|---|
| Associated object | `0x78` | 32-bit pointer-bearing backlink |
| Related object | `0x00` | Virtual-table address |
| Related object | `0x48` | 32-bit flags |
| Held object | `0x00` | Virtual-table address |
| Final dispatch result | `0x48` | Read and write of 32-bit flags |

Virtual-table slot offsets are byte offsets within the respective tables.

### Preconditions

The input object must remain valid for every reached access. A null input object is not supported.

Nonzero addresses that are dereferenced must identify readable storage of the required extent. Direct write destinations must be writable. Reached virtual-table entries must identify callable targets.

The result of the input object’s slot `0x5C` dispatch must support a 32-bit flags read and write at offset `0x48`; the function does not check that result for null.

Dependencies may change fields, global bindings, or object lifetimes. No preservation guarantee is supplied. Consequently, validity is required at each access, including accesses following callbacks.

## Behavior

### 1. Zero-mode transition

For mode zero, the function performs five byte stores, in this order:

1. Store 0 at `0xB4`.
2. Store 0 at `0xB5`.
3. Store 0 at `0xB9`.
4. Store 0 at `0xB8`.
5. Store 0 at `0xBA`.

It then checks the held-object address at `0xAC`.

If that address is zero, it skips the held-object dispatch and does not explicitly write `0xAC`.

If the address is nonzero, it dispatches through that object’s virtual-table slot `0x04`. After the dispatch returns normally, it stores zero into the input object’s `0xAC` field.

The field is not cleared before the dispatch. A callback can therefore observe the existing value unless some intervening effect changes it. If the callback installs a replacement address into `0xAC`, the subsequent explicit zero store overwrites that replacement.

No explicit receiver argument appears for this dispatch in the supplied representation. Its implicit calling convention and any ownership operation remain unresolved.

### 2. Nonzero-mode transition

For any nonzero mode, the function first stores:

1. 1 into byte `0xB4`.
2. 0 into byte `0xB5`.

This branch does not directly assign bytes `0xB8`, `0xB9`, or `0xBA`. Dependencies may still affect them.

The remaining nonzero-mode sequence is:

1. Call F-6f9a40ea7bea with `0xB71`.
2. Read a global value and place it in the first word of a local temporary record.
3. Call F-c5a3e2a1a1d4 with the address of the record’s 64-byte region, another global value, 4, and `0x10`.
4. Store `0x1700` into the record’s trailing word.
5. Call F-16920fdc01be with the record’s starting address.
6. Call F-5e828d4b34e9 with that same address.
7. Store 1 through a global destination.
8. Call F-0ddd45153607 with four arguments, each `0x10000`.
9. Store `0xFFFFFFFF` through a second global destination.
10. Store `0xFFFFFFFF` through a third global destination.
11. Call F-b4d2fdaa2e39 with the input object.

The local record has this represented layout:

| Relative offset | Extent | Preparation |
|---|---:|---|
| `0x00` | 4 bytes | Assigned the first global value |
| `0x04` | 64 bytes | Address passed to F-c5a3e2a1a1d4 |
| `0x44` | 4 bytes | Assigned `0x1700` after that call |

The record spans 72 bytes. The numeric arguments 4 and 16 do not establish the helper’s write extent, data format, or initialization coverage. No explicit initialization of the 64-byte region precedes that helper, and the trailing word is explicitly assigned only afterward.

F-5e828d4b34e9 receives the same storage address as F-16920fdc01be, but not necessarily unchanged contents: the first dependency may modify the record.

Global values and destinations are used at their respective points in this sequence. They are not established as immutable snapshots captured before earlier calls. In particular, the two destinations used after F-0ddd45153607 may reflect effects of that call.

The declarations of the three global write destinations are unavailable. The assigned values and ordering are established; their exact store widths are not independently established.

### 3. Additional zero-mode notification

After the selected transition, mode zero calls F-a1a48bd3fbde with the input object.

If its result is nonzero:

1. Dispatch through the input object’s virtual-table slot `0x1C`, explicitly passing the input object.
2. After that dispatch returns normally, call F-20460d1f31d3 without explicit arguments.

A zero predicate result skips both operations.

Nonzero mode skips the predicate call and both dependent operations entirely.

### 4. Associated-object notification

After the preceding work, the function captures two input-object fields in order:

1. Associated-object address from `0x3C`.
2. Related-object address from `0x34`.

If the captured associated-object address is nonzero, it reads that object’s backlink at `0x78` and compares it with the input object address.

A matching backlink causes a call to F-df6da5bd408b with:

- The captured associated-object address.
- The normalized notification value: 1 for mode zero, otherwise 0.

A null associated-object address or mismatching backlink skips the call.

The externally passed notification value is binary across the entire unsigned 32-bit mode range. Modes greater than 1 do not expose a negative or wrapped subtraction result as the notification argument.

The related-object address captured before this notification remains the candidate used for propagation qualification. A notification that changes the input object’s current `0x34` field does not replace that captured candidate.

### 5. Propagation qualification

The function reads the input object’s flags at `0x48` after the associated-object notification opportunity.

Qualification follows these rules:

- If input flag `0x10` is set, propagation qualifies without reading the captured related object’s flags.
- Otherwise, if the captured related-object address is nonzero, its 32-bit flags at `0x48` are examined.
- Propagation qualifies through that related object only when its flag `0x10` is set.
- If neither route qualifies, the function returns.

A callback in the associated-object notification can affect qualification by modifying the input flags or the captured related object’s flags. Changing only the input object’s current related-object field does not change the captured candidate.

The early return does not undo any preceding stores or calls.

### 6. Qualified propagation

On a qualifying path, the function:

1. Reads the input object’s 32-bit flags, sets bit `0x20`, and writes the result back.
2. Calls F-6d16652f2d03 with the input object and its embedded address at `0x80`.
3. Examines the current related-object field at `0x34`.
4. If that field is nonzero, reads the 32-bit word at input offset `0xA0`, narrows its value to an 8-bit character value, and compares that result with zero.
5. If the narrowed result is zero, dispatches through the current related object’s virtual-table slot `0x128`.
6. Dispatches through the input object’s virtual-table slot `0x5C`, explicitly passing the input object.
7. Treats that dispatch result as an object address, reads its 32-bit flags at `0x48`, sets bit `0x40`, and writes the result back.

If the current related-object field is zero, the gate word at `0xA0` is not read for this condition, and the slot `0x128` dispatch is skipped.

If the field is nonzero, the gate passes exactly when the low-order eight bits of the word are zero under the represented 8-bit-character model. Nonzero higher-order bits do not block the dispatch.

The related object used here is selected after F-6d16652f2d03 returns. It need not be the captured object used for qualification.

The slot `0x128` dispatch has no explicit receiver argument in the supplied representation. Regardless of whether it is skipped, normal completion proceeds to slot `0x5C`.

Both flag updates preserve other bits from the value read for that particular update. They do not guarantee preservation against subsequent dependency effects.

## Side effects

### Direct changes

Depending on the path, the function directly:

- Clears five state bytes in zero mode.
- Assigns two state bytes in nonzero mode.
- Clears the held-object address after its dispatch returns.
- Prepares portions of a local 72-byte record.
- Performs three global-destination writes in nonzero mode.
- Sets input flag `0x20` on qualifying paths.
- Sets flag `0x40` on the object returned by slot `0x5C`.

The `0xA0` gate is a read, not a write. Its represented access width remains 32 bits even though only eight low-order value bits determine the decision.

### Ownership and lifetime

Clearing the held-object address after a dispatch does not establish destruction, deallocation, or a reference-count decrement.

The temporary record is local storage passed to dependencies. Whether those dependencies copy its contents, retain its address, or manage referenced resources is unknown. The function itself provides no established ownership-transfer guarantee or longer-lived allocation for that storage.

No allocation or deallocation is directly established by this function.

Aliasing is not excluded. For example, the final returned object could be the input object. In that case, the final flag update applies to the input flags as they exist at that later point.

### Errors and interruption

There is no explicit error result, rollback mechanism, or recovery branch. Propagation failure to qualify is an ordinary early return.

No setup dependency result is checked for success. If a dependency does not return normally, later operations are not established as occurring. Invalid storage or dispatch targets have no defined recovery behavior in this contract.

No synchronization or atomic-update guarantee is established.

## Dependencies

| Opaque dependency or dispatch | Established use |
|---|---|
| F-6f9a40ea7bea | Nonzero-mode call with `0xB71` before record preparation |
| F-c5a3e2a1a1d4 | Receives the 64-byte region address, a global value, 4, and 16 |
| F-16920fdc01be | First recipient of the temporary record address |
| F-5e828d4b34e9 | Second recipient of the same record address |
| F-0ddd45153607 | Receives four arguments equal to 65,536 |
| F-b4d2fdaa2e39 | Receives the input object after nonzero-mode global writes |
| F-a1a48bd3fbde | Zero-mode predicate controlling two later calls |
| F-20460d1f31d3 | Called after the predicate-controlled slot `0x1C` dispatch |
| F-df6da5bd408b | Associated-object notification with normalized mode value |
| F-6d16652f2d03 | Qualified update receiving the input object and its `0x80` embedded address |
| Held-object slot `0x04` | Dispatch before clearing `0xAC` |
| Input-object slot `0x1C` | Predicate-controlled zero-mode dispatch |
| Related-object slot `0x128` | Dispatch gated by a nonnull current related object and narrowed word value |
| Input-object slot `0x5C` | Returns the destination for the final flag update |

No dependency is assumed to preserve globals, preserve object fields, or acquire or release ownership.

## Acceptance examples

These are conditional static expectations, not executed tests. Except where an example specifies otherwise, dependencies return normally and leave relevant state unchanged.

### Zero mode without propagation

Given mode zero, no held object, a zero result from F-a1a48bd3fbde, no associated object, clear input flag `0x10`, and no related object:

- Bytes `0xB4`, `0xB5`, `0xB9`, `0xB8`, and `0xBA` are cleared in that order.
- F-a1a48bd3fbde is called.
- Neither zero-mode notification call controlled by that predicate occurs.
- The function returns without setting flag `0x20` or reaching the final dispatch.

### Held-object replacement

Given mode zero and a nonzero held-object field, suppose slot `0x04` installs a replacement address at input offset `0xAC`.

After normal callback return, the explicit store clears `0xAC`. This function does not retain the replacement through that field.

### Maximum nonzero mode

Given mode `0xFFFFFFFF` and a matching associated-object backlink:

- The nonzero transition runs.
- Byte `0xB4` receives 1, not the original mode.
- F-df6da5bd408b receives notification value 0.
- F-a1a48bd3fbde is not called.

### Captured versus current related object

Suppose the captured related object has flag `0x10`, while the input object does not. During the associated notification, the input object’s `0x34` field changes to another object.

Qualification still uses the captured object. Any later slot `0x128` dispatch uses the current related object selected after F-6d16652f2d03, subject to the gate.

### Word read with low-order-eight-bit gate

Suppose propagation qualifies and the current related-object address is nonzero.

- If the 32-bit word read at `0xA0` has numeric value `0x12340000`, its narrowed value is zero, so slot `0x128` is dispatched.
- If that word has numeric value `0x12340001`, its narrowed value is nonzero, so that dispatch is skipped.

These expectations do not depend on assigning either value to a particular first byte in memory. Both cases require a readable 32-bit word.

If the current related-object address is zero instead, this condition does not read the word at `0xA0`.

### Flag preservation and aliasing

If the input flags are `0x110` when the explicit `0x20` update occurs, that store produces `0x130`.

If the final destination flags are `0x05` when the explicit `0x40` update occurs, that store produces `0x45`.

If the final destination aliases the input object, the final update uses the flags present at that later time, including any intervening dependency changes.

## Uncertainties

The following remain **documented_only**:

- Application meanings of the modes, state bytes, flags, global destinations, and constants.
- Target byte order. The low-order eight bits of the gate word are identified by value, not by memory position.
- Exact types and store widths of the global destinations.
- Implicit receiver conventions for dispatches lacking explicit object arguments.
- Dependency ownership, destruction, allocation, and deallocation behavior.
- Initialization coverage and interpretation of the 64-byte temporary region.
- Whether either record recipient retains its address or changes its contents.
- Whether dependencies alter global bindings, replace object relationships, or invalidate storage.
- Aliasing relationships among input, held, associated, related, and returned objects.
- Thread safety, synchronization, and reentrancy guarantees.
- Runtime object invariants and behavior following invalid accesses or abnormal dependency termination.

The 32-bit gate read and subsequent narrowing are established behavior, not an unresolved choice between byte and word access.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes explicit access widths, field offsets, sequential stores and calls, branch conditions, constants, virtual-table slot offsets, captured-versus-current pointer use, and the two 32-bit flag updates.

For the gate at `0xA0`, the evidence distinguishes the 32-bit read from conversion to the 8-bit value used in the zero comparison. It does not establish endianness.

Application-level purpose beyond the structural behavior is inferred intent. Dependency internals, omitted global declarations, implicit calling conventions, and runtime invariants remain unknown.

No runtime tests have been performed.
