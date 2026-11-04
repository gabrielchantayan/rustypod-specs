<!-- Trusted provenance: F-da708cccfb53; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-da708cccfb53

## Purpose

F-da708cccfb53 synchronizes a 32-bit cached value in a caller-supplied object with a value obtained through two external dependencies. When the newly obtained value differs from the cache, it updates the cache and performs three ordered indirect notifications through the object’s dispatch table.

The candidate value is either:

- Zero, when the lookup dependency returns a null result.
- A zero-extended 16-bit value reached through a non-null lookup result.

An unchanged value suppresses the cache write and all three notifications. It does **not** suppress the two preceding dependency calls.

This is a source-facing behavioral contract based on the supplied static representation. It does not establish the application meaning of the value, the notification arguments, or the external operations.

## Interface

### Input

The function accepts one object pointer. No additional explicit arguments are shown.

The supplied representation accesses the object using four-byte integer slots. The relevant layout is:

| Object-relative byte offset | Width | Role |
|---|---:|---|
| `0x00` | 32 bits | Dispatch-table address used for indirect notifications |
| `0x30` | 32 bits | Argument supplied to both external dependencies |
| `0xF0` | 32 bits | Cached value compared against the candidate and conditionally overwritten |

The offsets follow from the four-byte slot layout shown in the supplied representation. The address-bearing words are used as pointers; this describes a 32-bit address model, not a verified native structure declaration or ABI.

The dispatch table must provide a callable entry at byte offset `0x58` whenever a notification is reached. That entry is invoked with three arguments:

1. The original object pointer.
2. A value obtained from a shared global location.
3. A value obtained from one of three notification-specific global locations.

The dispatch-table address and the entry at `0x58` are resolved separately for each notification.

### Lookup-result layout

If F-32dc6b5d9f2d returns a nonzero address, the function follows this layout:

| Location | Width | Interpretation |
|---|---:|---|
| Lookup result plus `0x40` | 32 bits | Address of a secondary record |
| Secondary record plus `0x2E` | 16 bits, unsigned | Value used to form the candidate |

The 16-bit value is zero-extended to 32 bits. Therefore, on the non-null lookup path, the candidate lies in the inclusive range 0–65,535.

Only the outer lookup result is checked for zero. The secondary-record address is not checked before use.

### Output

No return value is specified. Ordinary completion returns control to the caller.

The visible result is conveyed through dependency calls, a possible cache update, and possible indirect notifications.

### Preconditions

For ordinary completion, all memory accesses and calls actually reached must be valid:

- The input object must permit the initial read at `0x30`.
- It must remain valid for subsequent accesses after the external calls.
- A non-null lookup result must support the 32-bit read at `0x40`.
- The resulting secondary-record address must support the unsigned 16-bit read at `0x2E`.
- The object must support the 32-bit cache read at `0xF0`, and a write there if the value changes.
- A changed-value path requires valid dispatch-table reads and callable entries for all three notifications.
- The relevant global locations must be readable when their arguments are obtained.

The source does not establish alignment requirements, byte order, concurrency rules, or the underlying calling convention.

## Behavior

### 1. Perform the initial external operation

The function reads the object’s 32-bit field at `0x30` and calls F-7daef0b57784 with:

- That field value.
- The numeric value 1.
- The numeric value 0.

No result from this dependency is used by the visible function.

This call occurs before the lookup, cache comparison, and notifications. It occurs even when the function will ultimately find that the cache is unchanged.

The dependency’s name and implementation are unavailable. This contract does not assign it a reset, enable, validation, or ownership-transfer meaning.

### 2. Perform the lookup using a fresh object-field read

After F-7daef0b57784 returns normally, the function reads the object’s field at `0x30` again and supplies that value to F-32dc6b5d9f2d.

The first dependency’s argument is not reused as the second dependency’s argument. If the first dependency changes the object field, the second dependency receives the changed value, provided the object remains valid.

The lookup result is retained for the subsequent null decision and, on the non-null path, the nested memory read.

### 3. Obtain the candidate value

A zero lookup result produces the 32-bit candidate zero. No read at lookup-result offset `0x40`, and no secondary-record read, occurs on this path.

A nonzero lookup result causes:

1. A 32-bit read at lookup-result offset `0x40`.
2. Use of that word as a secondary-record address.
3. An unsigned 16-bit read at secondary-record offset `0x2E`.
4. Zero-extension of that 16-bit value into the 32-bit candidate.

The unsigned interpretation matters. A secondary value with bit 15 set remains a positive 32-bit value; it is not sign-extended.

A non-null lookup whose secondary value is zero produces the same candidate as a null lookup. These paths differ in their memory accesses, but not in the candidate subsequently compared with the cache.

### 4. Compare against the current cache

After obtaining the candidate, the function reads the object’s 32-bit cache at `0xF0`.

The comparison is a full-width, 32-bit equality comparison. Although the object is represented through signed integer slots, comparison against the unsigned candidate does not reduce the comparison to 16 bits. Equality depends on the complete 32-bit value.

If the cache equals the candidate:

- The function returns normally.
- It does not write the cache.
- It does not perform any of the three indirect notifications.
- It does not perform the notification-related global or dispatch-table reads.

Both external dependencies have already been called on this path.

If the cache differs from the candidate, execution continues to the update and notification sequence.

### 5. Update the cache before notifying

The function writes the 32-bit candidate to object offset `0xF0`.

This write precedes all three notifications. At entry to the first notification, the cache therefore contains the candidate unless another actor has changed it in the meantime.

The function does not re-read the cache to decide whether later notifications should occur. A notification that changes the cache does not, by itself, cancel the remaining notifications.

### 6. Obtain notification arguments and invoke three callbacks

There are four relevant global locations:

- One shared-argument global.
- Three distinct trailing-argument globals, one for each notification.

These descriptions identify their usage, not their application meaning. The supplied representation shows their values being passed as four-byte arguments. It does not establish whether those values are numbers, pointers, handles, or tokens.

Before the first notification, the function reads the shared-argument global and retains that value for the second and third notifications.

The notifications then occur in this order:

| Notification | First argument | Second argument | Third argument |
|---|---|---|---|
| First | Original object pointer | A read of the shared-argument global for this call | First trailing-argument global |
| Second | Original object pointer | Retained shared value | Second trailing-argument global |
| Third | Original object pointer | Retained shared value | Third trailing-argument global |

The first notification’s second argument is represented as a separate global access from the earlier retained-value read. It should not be specified as necessarily equal to the retained value under an execution model allowing intervening global changes.

Before each notification, the function resolves the dispatch-table address from object offset `0x00` and the callable entry at dispatch-table offset `0x58`. It does not retain one callable target for all three notifications.

Consequently, the first notification may change the target used by the second, and the second may change the target used by the third. This remains conditional on the object and replacement dispatch data being valid.

The second and third trailing-argument globals are read for their respective calls, after preceding notifications have returned. Changes to those globals made by earlier callbacks can therefore affect later arguments.

Within a single indirect-call expression, the supplied representation does not establish an exact order among the dispatch-resolution reads and argument-global reads. The contract establishes their position between callbacks, not an unsupported instruction-level ordering.

Return values from the notifications are not used.

### 7. Return after the third notification

If all dependencies and notifications return normally, the function returns after the third notification.

No subsequent explicit cache write, rollback, or additional notification is shown.

## Side effects

### Direct effects

The visible function directly performs:

- Two ordered external calls.
- Object-field reads.
- Conditional nested-record reads.
- One conditional 32-bit cache write.
- Conditional notification-global reads.
- Three ordered indirect calls on the changed-value path.

It does not explicitly allocate, copy, acquire, retain, release, or free memory.

### Effects through dependencies

The external dependencies and notification targets may have additional effects. Their implementations are not supplied, so this contract does not assume that they:

- Preserve object fields or globals.
- Leave dispatch targets unchanged.
- Preserve the lifetime of referenced records.
- Avoid allocation or release.
- Return normally.
- Avoid reentry into this function.

Those unknowns matter because several values are deliberately read after earlier calls rather than captured before them.

### Partial completion

The cache is updated before notification begins. If a notification does not return normally, the function provides no visible rollback of that update and no guarantee that later notifications occur.

Likewise, if a callback changes the cache after the function’s write, the function does not restore the candidate afterward. Normal return does not alone guarantee that the cache still equals the candidate.

No explicit error status, retry, exception handler, or recovery path is shown.

## Dependencies

| Opaque function ID or dispatch mechanism | Established interaction |
|---|---|
| F-7daef0b57784 | Called once before lookup, with the current field at object offset `0x30`, followed by 1 and 0. No returned value is used. |
| F-32dc6b5d9f2d | Called once after the first dependency returns, using a fresh read of object offset `0x30`. Its result is interpreted as a nullable address. |
| Object dispatch entry at `0x58` | Called three times, in order, only after a changed candidate has been written to the cache. The target is re-resolved for each call. |

The indirect targets have no supplied opaque function IDs. They are identified here only by the dispatch mechanism that selects them.

No dependency-specific ownership contract can be derived from the visible caller.

## Acceptance examples

These examples are conditional expectations from static evidence, not reported runtime results. Unless stated otherwise, dependencies return normally, required memory is valid, and there are no additional concurrent mutations.

1. **Null lookup with a zero cache.**  
   Both external dependencies run. The candidate is zero. The cache is read but not written, and no notifications occur.

2. **Null lookup with a cache of 9.**  
   Both external dependencies run. No nested-record reads occur. The function writes zero to the cache, then invokes the three notifications in order.

3. **Non-null lookup with an unsigned 16-bit value of `0xFFFF`.**  
   The candidate is 65,535. A cache containing `0x0000FFFF` is unchanged and produces no notifications. A cache containing `0xFFFFFFFF` differs, is overwritten with `0x0000FFFF`, and produces all three notifications.

4. **Non-null lookup with a zero secondary value.**  
   The nested reads still occur. Subsequent comparison and notification decisions use candidate zero, just as on the null-result path.

5. **The first dependency changes the field at `0x30`.**  
   The lookup dependency receives the new field value. It is not required to receive the value passed to the first dependency.

6. **The lookup dependency changes the cache.**  
   The comparison uses the cache read after lookup and candidate extraction. It does not use a cache snapshot from function entry.

7. **The first notification changes the shared-argument global.**  
   The second and third notifications still receive the retained shared value captured before the first notification. Their shared argument is not refreshed from that global.

8. **The first notification changes the second trailing-argument global.**  
   The second notification obtains that global’s value after the first callback returns and can observe the change.

9. **The first notification replaces the dispatch-table address.**  
   The second notification resolves its target through the replacement table. The third resolves its target again after the second callback returns.

10. **The first notification overwrites the cache.**  
    The remaining notifications still occur if execution remains valid and returns normally. The function does not reapply its candidate after the callbacks.

11. **A non-null lookup contains an invalid secondary-record address.**  
    No explicit validation or graceful error is established. The function attempts the 16-bit read through that address; the resulting failure behavior depends on the execution environment.

## Uncertainties

### Inferred intent

The cache comparison and conditional notification sequence suggest change detection followed by notification. This is an interpretation, not proof of an application-level event or property.

The constants 1 and 0 supplied to the first dependency do not establish what operation that dependency performs.

### documented_only

The following remain unresolved:

- Exact ABI, alignment rules, byte order, and address-arithmetic fault behavior.
- Semantic types of the object fields, lookup records, and global arguments.
- Meaning of the three trailing arguments.
- Dependency effects on globals, object state, memory ownership, and lifetime.
- Whether concurrent or reentrant execution is permitted or synchronized.
- Whether the globals are volatile, atomic, constant in practice, or externally mutable.
- Exact ordering of reads within individual indirect-call expressions.
- Behavior when a dependency throws, traps, terminates, transfers control elsewhere, or fails to return.
- Whether the final indirect transfer has machine-level tail-call behavior.

The supplied representation flags incomplete recovery of the final indirect transfer and presents it as a call through the same dispatch slot. The third invocation is supported by that representation; stronger machine-level control-flow claims are not established.

## Evidence

Established static evidence consists solely of the supplied function representation:

- Four-byte object-slot accesses establish the documented offsets.
- Two external calls precede candidate extraction and comparison.
- The lookup’s zero result selects candidate zero.
- The nonzero path reads a 32-bit address followed by an unsigned 16-bit value.
- Full-width equality controls the early return.
- The cache write precedes three indirect notifications.
- One shared value is retained for the second and third notifications.
- Each notification independently resolves the object’s dispatch entry.
- No explicit allocation, release, rollback, or error-reporting operation appears.

No runtime tests, execution traces, dependency implementations, or independent ABI validation were available. Acceptance examples describe consequences of the supplied static representation, not observed execution.
