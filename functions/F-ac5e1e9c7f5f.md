<!-- Trusted provenance: F-ac5e1e9c7f5f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ac5e1e9c7f5f

## Purpose

F-ac5e1e9c7f5f associates a supplied item with a state object, selects between two early-exit paths and a continuing path, and conditionally processes a related six-slot collection.

The established behavior consists of field accesses, ordered calls, conditional dispatch, and state writes. Terms such as “item,” “collection,” and “pending” below are descriptive roles, not recovered application types. The paired calls around the state object’s offset `0x44` suggest synchronization, but their actual semantics are unknown.

This contract describes normal execution through the supplied static representation. It does not establish behavior after a dependency fails, transfers control elsewhere, invalidates referenced storage, or changes fields that this function subsequently rereads.

## Interface

### Inputs and result

The function receives two 32-bit address-valued arguments:

- **State:** address of the principal mutable object.
- **Item:** address of the object initially installed into State.

No return value is established. A normal return communicates its result through memory changes and dependency side effects.

No null checks, size checks, ownership checks, or pointer validation are visible. Each path requires all storage it actually accesses to be valid for the access width. Indirect dispatch additionally requires valid callable targets and compatible calling conventions.

### Referenced layouts

Offsets are byte offsets. The observed address model and pointer-bearing fields are 32-bit.

| Object | Offset | Width | Observed role |
|---|---:|---:|---|
| State | `0x02` | 8 bits | Continuing-path processing selector |
| State | `0x05` | 8 bits | Early-exit gate |
| State | `0x1c` | 32 bits | Destination for a later item-field copy |
| State | `0x2c` | 32 bits | Signed threshold operand |
| State | `0x30` | 32 bits | Early-exit gate |
| State | `0x3c` | 32 bits | Current-item pointer |
| State | `0x40` | 32 bits | Continuing-path gate and opaque call argument |
| State | `0x44` | Unestablished extent | Address passed to paired dependencies |
| State | `0x6c` | At least 32 bits on one path | Cleanup argument or first word of a dispatch carrier |
| State | `0x9c` | 8 bits | Collection lookup argument |
| Item | `0x5c` | 32 bits | Flags |
| Item | `0x88` | 32 bits | Signed comparison operand and later copy source |
| Collection | `0x00`–`0x17` | Six 32-bit entries | Slot pointers |
| Collection | `0x1e` | 8 bits | Pending callback flag |
| Collection | `0x20` | 32 bits | Pending callback argument |
| Collection | `0x24` | 8 bits | Configuration-update flag |
| Collection | `0x25` | 8 bits | Change flag |
| Collection | `0x54` | 8 bits | Mode-zero argument |
| Collection | `0x56`–`0x59` | Four separate bytes | Mode-one arguments |
| Collection | `0x90` | 8 bits | Configuration mode |
| Collection | `0x94` | 32 bits | Interface-object pointer |
| Collection | `0xa5` | 8 bits | Collection-processing branch selector |
| Slot object | `0x08` | 8 bits | Type discriminator |
| Slot object | `0x1bc` | 8 bits | Field explicitly cleared on selected paths |

The interface object begins with a 32-bit pointer to a dispatch table. This function uses table entries at byte offsets `0x10`, `0x6c`, `0x70`, `0x74`, and `0x78`.

Two global-backed structures are also accessed:

- A location containing a 32-bit pointer to an early-exit dispatch object.
- A structure whose 32-bit field at offset `0x08` is incremented on the threshold early-exit path.

A separate global callable target is used for fallback dispatch.

### Ownership

No allocation, explicit deallocation, or reference-count operation is established directly. Calls may nevertheless acquire, release, replace, or invalidate objects. This contract does not assign ownership to any dependency.

Continued execution requires that every later reread and dereference remain valid, including after callbacks.

## Behavior

### 1. Initial association and priority early exit

The function first calls F-338d16f9ce72 with the address at State offset `0x44`. After that call returns, it writes Item into State offset `0x3c`.

It then reads Item’s 32-bit flags at `0x5c`.

If mask `0x02` is set, this branch takes precedence over all remaining decisions:

1. Call F-9713e3f8622e with State.
2. Call F-1333331b2da1 with State’s `0x44` address.
3. Call F-5bcae2eff9a7 with State’s `0x6c` address.
4. Read a 32-bit dispatch-object pointer from the global-backed dispatch location.
5. Perform the shared early-exit dispatch described below.
6. Return.

The global dispatch pointer is read after the three calls. Its value must not be assumed to be the value that existed at entry.

### 2. Threshold early exit

If the priority branch was not taken, the following conditions are evaluated in order, with later conditions reached only when earlier ones hold:

1. State’s 32-bit field at `0x30` is zero.
2. Item’s signed 32-bit field at `0x88` is strictly less than State’s signed 32-bit field at `0x2c` minus `33`.
3. State’s byte at `0x05` is zero.
4. A read of Item’s flags at `0x5c` has mask `0x08` clear.

If all hold:

1. Increment the global-backed 32-bit counter at offset `0x08`.
2. Call F-9713e3f8622e with State.
3. Call F-1333331b2da1 with State’s `0x44` address.
4. Read the 32-bit word at State offset `0x6c` as the dispatch-object pointer.
5. Perform shared early-exit dispatch.
6. Return.

Unlike the priority branch, this path does not visibly call F-5bcae2eff9a7.

The comparison is signed and strict: equality does not select this path. Arithmetic overflow behavior for subtracting `33`, and for incrementing the counter, is not established by the supplied representation. Neither checked failure nor saturation is visible.

### 3. Shared early-exit dispatch

Both early-exit paths supply a dispatch-object pointer to F-9813035c7ce7, with the explicit arguments:

- First: the 32-bit value `1`.
- Second: the selected dispatch-object pointer.

If the returned 32-bit value is anything other than signed `−1`, the function returns.

If the result equals `−1`, the function reads a 32-bit value at dispatch-object offset `0x04` and passes it to the global indirect callable target. It then returns if that call returns.

The dispatch-object pointer remains the selected value across F-9813035c7ce7. Its offset-`0x04` contents are read afterward and may have changed during the call.

### 4. Continuing-path gate and item marking

When neither early-exit branch applies:

1. Call F-1333331b2da1 with State’s `0x44` address.
2. Read State’s 32-bit field at `0x40`.
3. Return immediately if that field is zero.

For a nonzero gate, call F-338d16f9ce72 again with State’s `0x44` address.

The byte at State offset `0x02` determines whether later collection processing is enabled:

- Value `0`: enabled.
- Value `1` through `255`: disabled.

The displayed arithmetic uses a 32-bit intermediate, but its resulting decision is exactly this zero-versus-nonzero distinction.

Next, regardless of that decision:

1. Read the current-item pointer from State offset `0x3c`.
2. Set mask `0x04` in that pointed-to object’s 32-bit flags at `0x5c`, preserving the other observed bits.
3. If collection processing is enabled, invoke an unresolved direct dependency with the current 32-bit value at State offset `0x40` and the original Item argument.
4. Call F-1333331b2da1 with State’s `0x44` address.
5. Reread the current-item pointer from State offset `0x3c`.
6. Copy its 32-bit field at `0x88` into State offset `0x1c`.

If collection processing is disabled, return after the copy.

The original Item argument and State’s current-item field are not interchangeable. A dependency may change the latter. Marking and copying follow the current-item pointer at their respective access times; the unresolved direct call receives the original Item argument.

### 5. Collection lookup

When collection processing is enabled, read State’s byte at `0x9c` and pass it to F-bb50a072e902. Treat its returned 32-bit value as the Collection address.

There is no visible null-result check.

Read Collection’s byte at `0xa5` to choose between the following branches.

### 6. Nonzero collection selector

When Collection offset `0xa5` is nonzero, perform two complete ascending scans of slots `0` through `5`.

**First scan:**

For each slot, read its pointer. If it is nonzero and its object’s byte at `0x08` equals `5`:

1. Call F-03baa9c670b2 with no explicit arguments.
2. If its result is nonzero:
   - Reread the slot pointer and pass it to F-8c761d2d09dc.
   - Reread the slot pointer and clear that object’s byte at `0x1bc`.
3. Reread the slot pointer and pass it to F-8826342b149e.

Thus F-8826342b149e is reached for a qualifying type-`5` slot regardless of the predicate result.

**Second scan:**

For each slot whose pointer is nonzero when checked:

1. Call F-8826342b149e. No explicit argument is recoverable for this call occurrence.
2. Reread the slot pointer and clear that object’s byte at `0x1bc`.

Return after the second scan.

The first scan finishes before the second begins. This is not a per-slot combination of both scans.

### 7. Zero collection selector

When Collection offset `0xa5` is zero:

1. Call F-bdd02c544bdc with Collection.
2. Scan slots `0` through `5` in ascending order.

For each slot with a nonzero pointer when checked, call F-03baa9c670b2. If its result is nonzero:

1. Reread the slot pointer and pass it to F-8c761d2d09dc.
2. Write `1` to Collection offset `0x25`.
3. Reread the slot pointer and clear its object’s byte at `0x1bc`.

After this scan, process the pending callback and configuration state.

#### Pending callback

If Collection’s byte at `0x1e` is nonzero:

1. Dispatch through interface-table offset `0x10`, explicitly supplying the interface-object pointer from Collection offset `0x94` and the 32-bit value at Collection offset `0x20`.
2. Clear Collection offset `0x1e`.
3. Set Collection offset `0x25` to `1`.

These writes occur after the callback returns.

#### Configuration dispatch

Read Collection offset `0x24`.

If it is zero, read offset `0x25`. When both are zero, skip the table-offset-`0x78` call and proceed to the final slot scan.

If offset `0x24` is nonzero, inspect the byte at `0x90`:

- **Mode `0`:** call table offset `0x6c` with the interface-object pointer and the byte from Collection offset `0x54`.
- **Mode `1`:** issue these calls in order:
  1. Table offset `0x74`: interface pointer and byte at `0x56`.
  2. Table offset `0x70`: interface pointer, `0`, and byte at `0x57`.
  3. Table offset `0x70`: interface pointer, `1`, and byte at `0x58`.
  4. Table offset `0x70`: interface pointer, `2`, and byte at `0x59`.
  5. Table offset `0x70`: interface pointer and `3`; no third explicit argument is established for this occurrence.
- **Other modes:** no mode-specific call is visible.

After any of these mode cases, clear Collection offset `0x24` and set offset `0x25` to `1`.

Then invoke interface-table offset `0x78`. No explicit arguments are established for that call.

The interface pointer and relevant arguments are loaded at their respective call sites. Earlier callbacks may therefore affect later dispatch targets or arguments.

#### Final slot scan

Scan slots `0` through `5` in ascending order. For every slot whose pointer is nonzero when checked, invoke F-65edd8002f52. No explicit arguments are established.

Return after this scan.

## Side effects

Directly established writes are:

- Installing Item in State offset `0x3c`.
- Incrementing the global counter on the threshold early-exit path.
- Setting current-item flag mask `0x04` on the continuing path.
- Copying the current item’s 32-bit field at `0x88` to State offset `0x1c`.
- Clearing selected slot objects’ bytes at `0x1bc`.
- Setting Collection’s byte at `0x25`.
- Clearing Collection’s bytes at `0x1e` and `0x24`.

No visible operation clears Collection offset `0x25`.

All calls may have additional effects not established here. In particular, slot entries are repeatedly read across calls. An entry may differ between its initial nonzero check, the argument passed to a dependency, and the object later written. No post-call null recheck is visible before several such writes.

No rollback, local exception handling, or ordinary error-result propagation is established. A dependency that does not return prevents subsequent visible operations.

The paired State-offset-`0x44` calls do not establish thread safety. Accesses also occur between pairs and after them, and the dependencies’ synchronization semantics are unknown.

## Dependencies

| Opaque ID or dispatch | Established use |
|---|---|
| F-338d16f9ce72 | Receives State’s `0x44` address before initial association and again on the gated continuing path |
| F-1333331b2da1 | Receives the same address at branch-specific completion points |
| F-9713e3f8622e | Receives State on either early-exit path |
| F-5bcae2eff9a7 | Receives State’s `0x6c` address only on the priority early-exit path |
| F-9813035c7ce7 | Receives `1` and a dispatch-object pointer; result is tested against `−1` |
| F-bb50a072e902 | Receives State’s byte at `0x9c`; returns the Collection address |
| F-03baa9c670b2 | No explicit arguments; nonzero result gates selected slot work |
| F-8c761d2d09dc | Receives a reread slot pointer |
| F-8826342b149e | Explicit slot pointer on the first nonzero-selector scan; unresolved arguments on the second |
| F-bdd02c544bdc | Receives Collection before zero-selector processing |
| F-65edd8002f52 | Called for occupied slots in the final zero-selector scan; unresolved arguments |
| Unresolved direct dependency | Receives State offset `0x40` and original Item; no supplied opaque ID identifies it |
| Global indirect dependency | Receives dispatch-object offset `0x04` after the `−1` result |
| Interface indirect dependencies | Selected by table offsets described above |

## Acceptance examples

These are conditional static acceptance scenarios, not executed tests. Unless stated otherwise, dependencies return normally and leave relevant pointers and fields unchanged.

1. **Priority overrides threshold.** With Item flag `0x02` set, take the priority early exit even if every threshold condition also holds. Do not increment the threshold counter.

2. **Strict boundary.** With State’s threshold operand equal to `100`, the comparison threshold is `67`. Item value `66` qualifies if the other gates hold; `67` does not.

3. **Fallback sentinel.** A shared-dispatch result of `0`, `1`, or `−2` returns without fallback. Exactly `−1` invokes the global indirect dependency.

4. **Zero continuing gate.** If neither early exit applies and State offset `0x40` is zero when checked, return without setting item flag `0x04` or copying into State offset `0x1c`.

5. **Disabled collection processing.** With a nonzero continuing gate and State byte `0x02` equal to `2`, still mark the current item and perform the 32-bit copy. Do not invoke the unresolved direct dependency or look up Collection.

6. **Repeated slot processing.** In the nonzero-selector branch, a stable type-`5` slot with a false predicate reaches F-8826342b149e once in each scan. A stable occupied non-type-`5` slot reaches it only in the second scan.

7. **No pending collection work.** In the zero-selector branch, if no scan sets `0x25`, and offsets `0x1e`, `0x24`, and `0x25` remain zero, skip table offset `0x78` but still perform the final slot scan.

8. **Unsupported configuration mode.** With `0x24` nonzero and mode `2`, make no mode-specific dispatch, then clear `0x24`, set `0x25`, and invoke table offset `0x78`.

## Uncertainties

The following are **documented_only**:

- Actual application meaning, ownership, and synchronization semantics.
- The identity and contract of the unresolved direct dependency.
- Missing explicit arguments at several direct and indirect call sites, especially the final mode-one call.
- Signed overflow behavior and machine-level counter wrap semantics.
- Whether callbacks preserve globals, object lifetimes, slot entries, or interface pointers.
- Error behavior of dependencies and indirect targets.
- Whether the global fallback normally returns.
- Runtime validity of the represented layouts and call signatures.

Inferred intent is limited to association, early diversion, collection processing, and pending-state notification. Those interpretations do not add guarantees.

## Evidence

The supplied static representation establishes the field widths, offsets, branch precedence, masks, signed comparison, six-entry scan bounds, explicit writes, call ordering, and recoverable arguments described above.

It does not establish missing prototypes, dependency implementations, runtime object validity, or application-level semantics. No runtime tests have been performed.
