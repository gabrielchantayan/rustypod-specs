<!-- Trusted provenance: F-d526deebba68; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d526deebba68

## Purpose

F-d526deebba68 selects a record for an object, captures that record’s one-byte result, performs an initial action, and conditionally invokes either a virtual notification or a sequence of entry operations.

Selection uses the supplied lookup argument first. If that lookup fails, the function obtains a fallback range and tries successive position values from that range.

The function also changes one object byte and interacts with a shared bookkeeping facility. Its state restoration, further object processing, and final bookkeeping operation follow distinct conditions. Successful selection does not guarantee that the object byte is restored.

“Record,” “notification,” and “bookkeeping” identify visible roles. Their application meanings and ownership implications are not established.

## Interface

### Inputs and result

The interface has four arguments in the supplied 32-bit model:

| Input | Representation | Visible use |
|---|---|---|
| Object | Pointer | Receiver for lookups, actions, entry operations, and virtual dispatch; supplies directly accessed state. |
| Lookup argument | 32-bit value | Passed unchanged to the primary lookup and fallback-range provider. |
| Entry argument | 32-bit value | Passed unchanged to each entry operation. |
| Initial-action argument | 32-bit value | Passed unchanged to the initial selected-record action. |

The last three inputs are not locally validated or interpreted as ownership-bearing values. Their application types are unknown.

The return value is one byte:

- Zero if no record is selected.
- Otherwise, the byte captured from selected-record offset `0x10`, before the initial action.

The result is not normalized to a Boolean and does not report processing success. A selected record may itself supply zero, making that outcome indistinguishable from selection failure through the return value alone.

### Object accesses

| Offset | Width and use |
|---|---|
| `0x00` | A 32-bit virtual-table pointer is read on the notification branch. |
| `0x40` | **Initially read as a 32-bit word**, then converted to a character value retained for later decisions and restoration. Subsequent direct writes here are **one byte**. |
| `0x41` | A one-byte condition is read on the notification decision path, only if the saved character value is zero. |

The initial state access therefore covers offsets `0x40` through `0x43`. It is not an initial one-byte load. The later write of `1` and the restoring writes affect only the byte at `0x40`; they do not restore or otherwise directly write the remaining three bytes of the initially read word.

The notification target is the function pointer at virtual-table byte offset `0xb8`. The call receives the original object and `0x10`.

### Record and descriptor accesses

A selected record supplies:

| Offset | Access |
|---|---|
| `0x00` | Read as a 32-bit value for the initial action. |
| `0x04` | Its address is passed to entry-iteration setup. This is not a load of a pointer from that offset. |
| `0x10` | Read as the one-byte return value. |

The fallback descriptor contains a 32-bit starting position at offset `0x00` and a 32-bit ending position at offset `0x04`.

The local entry descriptor is used as two consecutive 32-bit slots: current position followed by ending position. A separate 32-bit local slot supplies entry stop state. These slots have observable address relationships because dependencies receive their addresses.

### Preconditions

Every reached access and call must remain valid when reached. In particular:

- The object must initially permit a 32-bit read through offset `0x43`.
- Its byte at `0x40` must permit the subsequent writes.
- A requested fallback descriptor must permit both endpoint reads; no null check is shown.
- A selected record must permit the result-byte and first-word reads.
- The notification path requires the later state-byte read, virtual-table access, and indirect call to be valid.
- Entry setup must establish storage usable by the subsequent descriptor accesses.

No object null check is shown. Lookup results are checked for null before becoming selected records. Initial validity does not establish that dependencies preserve validity for later use.

## Behavior

### Entry state and initial bookkeeping

The function first reads the **32-bit object word at `0x40`** and converts it to a character value. That converted value, rather than a separately loaded byte, is saved.

Next, it initializes a local object-pointer slot with the original object pointer and calls F-78a6ba689c62 with:

1. The address of local bookkeeping-output storage.
2. The value obtained by reading the shared global bookkeeping operand for this call.
3. The address of the object-pointer slot.

After the call returns, the function reads a 32-bit word from the bookkeeping output and saves it separately for possible final use. It then writes the one-byte value `1` at object offset `0x40`.

Thus the initial word read precedes bookkeeping, and bookkeeping precedes the temporary byte write. Only the converted character is retained for restoration; the full initial object word is not restored.

The object-pointer slot is exposed by address. Its contents need not remain equal to the original object pointer after dependencies run. Later bookkeeping queries receive that same slot’s address, whereas the ordinary object operations shown continue to receive the original object argument.

### Record selection

F-b57670b7d06e is first called with the original object and lookup argument.

A non-null result is selected immediately. In that case, F-f67487ff8436 is not called.

If the primary result is null, F-f67487ff8436 receives the same object and lookup argument. Its result is treated as the fallback descriptor. The function loads its starting position once.

Each fallback loop-condition evaluation reads the descriptor’s ending position again and compares it for equality with the current position. If they differ:

- An already non-null lookup result ends the search without another lookup.
- Otherwise, F-b57670b7d06e receives the object and the current position value itself.

The position advances by `4` after a fallback lookup, including a successful one. The next endpoint comparison therefore occurs before selection leaves the fallback loop. If that comparison is unequal, the already-selected-record check ends the search; if equal, normal loop termination ends it. Neither case issues another lookup.

The function does not dereference a fallback position to obtain its lookup argument.

Consequences:

- Equal initial endpoints produce no fallback lookup.
- Failed candidates are followed by positions four units higher.
- Endpoint termination uses equality, not ordering.
- A successful fallback lookup still precedes an increment and another endpoint read.
- No local validation establishes alignment, increasing endpoints, or endpoint reachability.

Arithmetic beyond representable positions and malformed ranges are not given a portable wrapping or termination guarantee.

### Selection failure

If no record was found, the function:

1. Establishes a zero return byte.
2. Writes the saved character value back to the object byte at `0x40`.
3. Proceeds to the final bookkeeping query.

There is no initial action, processing-gate query, notification, entry setup, entry operation, or post-entry operation on this path.

This restoring byte write is unconditional with respect to bookkeeping results and occurs before the final query.

### Result capture and initial action

For a selected record, the function captures its byte at `0x10` and then calls F-ec490cb76064 with:

- The original object.
- The record’s first 32-bit word.
- The initial-action argument.

Later modification of the record’s result byte does not change the captured result. The caller does not use the initial action’s return value.

The selected-record address remains relevant on the entry-processing path after this action. Nothing shown establishes that the action preserves that record.

### Processing gate and restoration

After the initial action returns, the function reads the shared global bookkeeping operand again and calls F-9def54813658 with that value and the object-pointer-slot address.

If the query returns zero, the function goes directly to final bookkeeping. It performs no restoring write at this point, does not inspect byte `0x41`, and invokes neither notification nor entry processing.

If the query returns nonzero, the function first writes the saved character value to byte `0x40`. It then chooses the processing branch.

The notification branch requires both:

- The saved character value is zero.
- The byte currently read at object offset `0x41` is nonzero.

This uses the saved character, not a fresh read of the word or byte at `0x40`. If the saved character is nonzero, the condition short-circuits without reading byte `0x41`.

When notification is selected, the virtual-table entry at offset `0xb8` is called with the original object and `0x10`. Entry setup, entry callbacks, and the post-entry operation are skipped.

### Entry processing

On the non-notification branch, the function first writes zero to the full 32-bit stop-state slot. It then calls F-c5588aad472f with the local descriptor address and the address at selected-record offset `0x04`.

Iteration continues while the stop-state word, converted to a character, is zero and the descriptor’s current and ending positions differ. The stop condition is evaluated first; a nonzero controlling character prevents evaluation of the endpoint comparison on that check.

For each reached entry, F-cab3918621e3 receives:

1. The original object.
2. The current position value, not a value loaded from that position.
3. The entry argument.
4. The stop-state slot’s address.

After the call returns, the current-position slot advances by `0x10`, including when that call has requested stopping. The following condition decides whether another entry is processed.

The stop decision uses the character-width portion of the 32-bit state, not a whole-word nonzero test. In the supplied eight-bit character model, `0x00000100` has a zero controlling low byte, whereas `0x00000001` does not.

Descriptor storage is address-exposed. This contract does not assume that dependencies leave the current position, endpoint, or other reachable storage unchanged. In particular, advancing the current-position slot after a callback is not a guarantee that an earlier position value survives that callback unchanged.

Once iteration ends, the function calls, in order:

1. F-6ad14e20a3dc with the original object.
2. F-37e94bb0f0ac with the descriptor address.

Both calls occur even for an initially empty entry range. Neither occurs on the notification branch or when the processing gate is zero.

### Final bookkeeping and reused storage

Every visible normal-return path reaches a final F-9def54813658 call. The function **reads the shared global bookkeeping operand again** for that call and supplies the object-pointer-slot address.

This is independent of any earlier query. The earlier result and earlier operand value are not reused.

If the final query returns nonzero, the function performs these writes in order:

1. Writes the saved 32-bit bookkeeping word into the slot used earlier for stop state.
2. Writes the same saved word into the slot used earlier for the entry descriptor’s ending position.

It then calls F-29578640674e with:

- The original bookkeeping-output address.
- A **newly read** value of the shared global bookkeeping operand.
- The address of the reused iteration-end slot.

These are two distinct writes, not merely initialization of unspecified cleanup storage. They occur even when the execution path never initialized stop state or an entry descriptor. If entry processing did occur, they overwrite those slots’ prior contents.

The same iteration-end slot is reachable through the earlier descriptor address. The stop-state address may also have been supplied to entry callbacks. The evidence does not establish that dependencies discard those addresses, so the overwrites cannot be dismissed as necessarily unobservable.

If the final query returns zero, neither overwrite nor F-29578640674e occurs.

The function then returns the established result byte. No unconditional final restoration of object byte `0x40` is present.

## Side effects

Directly established effects include:

- Reading four object bytes at entry, followed by one-byte state writes.
- A temporary write of `1` after initial bookkeeping.
- Restoration on selection failure or a nonzero processing-gate result.
- Writes to bookkeeping, object-pointer, descriptor, and stop-state local storage.
- Two conditional final writes reusing the stop-state and iteration-end slots.
- Conditional calls to external dependencies and a virtual target.

The shared global bookkeeping operand is evaluated separately at these positions:

| Position | Condition |
|---|---|
| Initial F-78a6ba689c62 call | Reached on every normal invocation. |
| Post-action F-9def54813658 call | Only after successful selection and the initial action. |
| Final F-9def54813658 call | Reached on every visible normal-return path. |
| F-29578640674e call | Only after a nonzero final query, following both local-slot overwrites. |

Intervening dependencies may change the global. In particular, final cleanup need not receive the operand value used by the immediately preceding query.

No direct heap allocation or release is shown. This does not constrain allocation, mutation, reference retention, or resource release inside dependencies. No exception handling, nonlocal-exit cleanup, or failure rollback is established.

## Dependencies

| Opaque ID | Visible role |
|---|---|
| F-78a6ba689c62 | Receives bookkeeping-output storage, the current global operand value, and the object-pointer-slot address. |
| F-b57670b7d06e | Returns a record or null for the supplied object and lookup value. |
| F-f67487ff8436 | Supplies the fallback descriptor after primary lookup failure. |
| F-ec490cb76064 | Performs the initial action using the selected record’s first word and the fourth input. |
| F-9def54813658 | Returns the zero/nonzero gate for processing or final bookkeeping. Each invocation receives a separately read global operand. |
| F-c5588aad472f | Receives the entry descriptor address and selected-record address plus `0x04`. |
| F-cab3918621e3 | Processes a position and receives writable stop-state storage. |
| F-6ad14e20a3dc | Runs after entry iteration, including an empty iteration. |
| F-37e94bb0f0ac | Receives the entry descriptor after the post-entry operation. |
| F-29578640674e | Performs conditional final bookkeeping using the original output storage, a newly read global operand, and the reused iteration-end slot. |

The virtual target has no supplied opaque function ID. Its dispatch offset and arguments are established; its implementation is unknown. No dependency is assumed to preserve globals, retain or discard local-storage addresses, or own or release memory.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Initial access width.**  
   Storage readable only through object offset `0x40` does not satisfy the entry precondition. The initial read requires readable storage through `0x43`, even if later processing would need only a byte write at `0x40`.

2. **Primary success and captured result.**  
   If the primary lookup selects a valid record containing result byte `0x7e`, no fallback descriptor is requested. If dependencies return normally, the result remains `0x7e` even if the initial action changes that record byte.

3. **Empty fallback.**  
   If the primary lookup fails and the fallback endpoints are equal, no fallback lookup occurs. The saved character is written to object byte `0x40`, the result is zero, and final bookkeeping is queried.

4. **Second fallback candidate succeeds.**  
   With stable endpoints `0x1000` and `0x100c`, a failure at `0x1000` followed by success at `0x1004` produces no lookup at `0x1008`. The position nevertheless advances to `0x1008`, and the endpoint is read again before selection exits.

5. **Processing gate is zero.**  
   Successful selection followed by a zero post-action query skips restoration at that stage, notification, and entry processing. Final bookkeeping still occurs independently. No claim is made that the object byte remains `1`, because intervening dependencies may have changed it.

6. **Notification condition.**  
   If the saved character is zero, the processing gate is nonzero, and the later byte at `0x41` is nonzero, restoration precedes virtual dispatch. No entry descriptor is established.

7. **Empty entry range.**  
   If entry setup supplies equal positions and the stop-state check allows endpoint evaluation, no entry callback occurs. F-6ad14e20a3dc then F-37e94bb0f0ac still run.

8. **Stop-state width.**  
   Assuming otherwise stable descriptor state and a remaining entry, a callback writing `0x00000100` does not stop through the low-byte condition. Writing `0x00000001` prevents the next entry. The current-position update occurs after either callback.

9. **Global operand changes.**  
   If the final query receives operand value A, changes the global to B, and returns nonzero, cleanup receives B, assuming no further intervening change. It does not automatically receive A.

10. **Cleanup storage reuse.**  
    If entry processing occurred and the final query is nonzero, both the callback-exposed stop-state slot and descriptor-end slot are overwritten with the saved bookkeeping word. Cleanup receives the latter slot’s address. Observation through retained aliases is conditional on dependency behavior and address validity, not ruled out by this contract.

## Uncertainties

The following remain **documented_only**:

- Application meanings of all state, records, actions, and bookkeeping values.
- The shared global operand’s declaration, mutation rules, and facility semantics.
- The target-specific character conversion and byte-order rules; the initial word-to-character conversion must not be replaced with an assumed byte load.
- Full record and descriptor layouts beyond the accesses described.
- Whether dependencies retain caller-local addresses and how any retained aliases are used.
- Lifetime preservation of objects, records, dispatch tables, or descriptors.
- Ownership and resource-release responsibilities.
- Overflow behavior and termination for malformed fallback or entry ranges.
- Behavior after faults, exceptions, nonlocal exits, or dependencies that do not return normally.

The apparent use of bookkeeping as an object-processing guard is inferred intent, not an established lifetime guarantee.

## Evidence

The supplied static body establishes access widths, offsets, call arguments, branch conditions, normal-return ordering, equality-based iteration, position increments, result capture, repeated global-operand evaluation, and final reuse of two local slots.

It does not establish dependency internals, ownership, global stability, alias non-observability, or portable arithmetic and character-conversion behavior.

No implementation access, runtime execution, or tests were performed.
