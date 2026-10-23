<!-- Trusted provenance: F-ea83ae94b50b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ea83ae94b50b

## Purpose

F-ea83ae94b50b conditionally invokes an indirect callback a configured number of times. After each callback that returns normally, it increments a 32-bit field in the supplied object.

The decision to perform callbacks is controlled by bit 0 of a 32-bit flags field. The requested repetition count is captured before that flag is inspected. For a nonnegative signed count and ordinary execution, an enabled invocation performs exactly that many callbacks.

The callback target and its second argument are obtained from the object separately for each iteration. They are not captured once for the entire invocation. Consequently, changes made by one callback can affect subsequent callbacks. The progress field is also read after each callback returns, so the subsequent increment applies to its then-current value.

These statements describe established static behavior, subject to valid memory, a callable target, normal callback return, and the arithmetic qualifications below. The intended application-level meaning of the callbacks and fields is not established.

## Interface

### Input

The function receives one value used as the base address of an object. The supplied representation uses a 32-bit integer-shaped parameter for that address. This does not establish that the public interface semantically accepts an integer rather than a pointer.

The following offsets are relative to that base address:

| Offset | Width | Role established by behavior |
|---|---:|---|
| `0x18` | 32 bits | Flags word; only bit 0 controls this function |
| `0x1C` | 32 bits | Encoded callback target; its low two bits are cleared before invocation |
| `0x24` | 32 bits | Value supplied as the callback’s second argument |
| `0x34` | 32 bits | Initial repetition count, interpreted as signed |
| `0x3C` | 32 bits | Field incremented after each normally returning callback |

Offsets describe accessed storage, not a complete object layout. The supplied material does not establish names, types, or meanings for intervening fields.

### Output

There is no explicit return value. Observable results consist of:

- Indirect callback invocations.
- Writes to the field at offset `0x3C`.
- Any effects caused by the callbacks.
- Any failure or nontermination caused by invalid accesses, invalid call targets, arithmetic edge behavior, or callback execution.

No status value or structured error result is produced by the visible function.

### Callback interface

The indirect dependency is identified here as **F-unknown-indirect**. This is a documentary identifier for the dynamically selected callee, not evidence of a single fixed implementation.

Each invocation supplies two arguments:

1. The constant value `0x20`, numerically 32.
2. The 32-bit value read from offset `0x24` for that iteration.

The callback target is derived from the 32-bit value at offset `0x1C` by clearing bits 0 and 1. Its remaining 30 bits are retained.

The callback’s return value, if any, is not consumed by the visible function. Its calling convention, argument interpretation, and ownership behavior are unknown.

### Preconditions

For defined, ordinary execution:

- The base address and relevant offset calculations must designate valid storage.
- The initial count and flags fields must be readable, even when the flag ultimately disables callbacks.
- When an iteration occurs, the callback-target and callback-argument fields must be readable.
- The masked target must identify executable code compatible with the actual calling convention.
- After a callback returns, offset `0x3C` must remain readable and writable.
- Any later iteration must still be able to access the callback-related fields.
- Arithmetic must remain within the established domain, or the execution environment’s behavior at the boundary must be separately specified.

The source does not establish access alignment requirements, byte order, concurrency guarantees, or a null-input convention.

## Behavior

### Initial observations and enablement

The function first reads the 32-bit count at offset `0x34` and retains it as local repetition state.

It then reads the flags word at offset `0x18` and tests bit 0.

If bit 0 is clear, the function returns without invoking F-unknown-indirect and without directly writing offset `0x3C`. The earlier count read still occurs. A disabled object therefore cannot rely on the count field being inaccessible.

Other flag bits do not affect this decision.

The enablement decision is made once. The visible function does not recheck the flags between callbacks.

### Repetition rules

When enabled, the function decrements its retained signed count before deciding whether to invoke the callback. An iteration proceeds only if that decremented value is nonnegative.

For values whose decrement is representable in signed 32-bit arithmetic:

- An initial count of zero produces no callbacks.
- An initial positive count of \(N\) produces \(N\) callbacks, provided all callbacks return and execution remains valid.
- An initial negative count greater than the signed minimum produces no callbacks.

The field at offset `0x34` is not reread during the loop and is not directly modified by this function. A callback changing that field does not change the already captured repetition state.

The minimum signed 32-bit count, `−2,147,483,648`, is a separate boundary: its first decrement is outside the signed 32-bit range. A portable callback count for this input is not established.

### Per-iteration callback preparation

For each iteration that proceeds, the function obtains the current encoded target from offset `0x1C` and the current 32-bit argument value from offset `0x24`.

Both values are used for that iteration’s call. The supplied expression does not establish a portable relative ordering between those two reads. It does establish that they supply the callback invocation, rather than being deferred until after it.

Clearing the target’s low two bits means encoded values that differ only in those bits resolve to the same call target. This transformation is not a validity check. A zero or otherwise unusable masked target is not rejected by an explicit guard.

Because these fields are accessed inside the loop, a callback may change which target or second argument is used on the next iteration. This is a consequence of the visible reads, not a guarantee that any particular callback performs such changes.

### Post-callback update

After the callback returns normally, the function:

- Reads the current 32-bit value at offset `0x3C`.
- Adds one using the signed integer operation shown.
- Writes the result back to offset `0x3C`.

The read occurs after the callback. The function does not increment a value saved before entering the callback.

If the callback writes a new value to offset `0x3C`, the post-callback update applies to that new value. If the callback invalidates the storage, the subsequent access is not protected.

The update is performed once for each callback that returns normally and reaches this operation. It is not performed before the callback. If a callback never returns, the corresponding post-callback increment and all later iterations are not reached.

### Completion

After the final post-callback update, the function performs the next decrement-and-test of its local repetition state. Once that test is negative, it returns.

No final callback, completion notification, or additional field update is visible.

## Side effects

### Direct effects

The visible function directly writes only the 32-bit field at offset `0x3C`.

It does not directly:

- Rewrite the flags.
- Rewrite the encoded callback target.
- Rewrite the callback argument.
- Store the remaining repetition count back to the object.
- Allocate or release storage.
- Validate object lifetime or callback validity.
- Report an error through a return value.

This describes direct operations only. It does not constrain what F-unknown-indirect may do.

### Callback effects and ownership

No ownership transfer is established for the object, the callback argument, or storage reachable through either.

The 32-bit callback argument is passed as a value. Whether the callee interprets it as a pointer, handle, integer, or other representation is unknown. No retain, release, copying, or lifetime promise can be inferred.

A callback may alter the object or other state. The visible function does not save and restore object fields around the call. It also does not establish that callbacks preserve globals, retain the object’s lifetime, or avoid recursive invocation.

### Concurrency and atomicity

No synchronization or atomic update is visible. The post-callback increment is represented as a read, arithmetic operation, and write, not as an established atomic read-modify-write.

Behavior under concurrent mutation is not specified by this evidence. In particular, the function does not establish race-safe counting or a coherent snapshot across the target and argument fields.

## Dependencies

| Opaque ID | Selection | Established interaction |
|---|---|---|
| F-unknown-indirect | Current 32-bit value at offset `0x1C`, with its low two bits cleared | Called once per eligible iteration with `0x20` and the current 32-bit value at offset `0x24`; any return value is ignored |

No fixed implementation of F-unknown-indirect is supplied. Different iterations may resolve to different implementations.

No other explicit function dependency is visible.

## Acceptance examples

These are conditional acceptance cases derived from static evidence. They are not reports of executed tests.

### Disabled invocation still reads the count

Given readable count and flags fields, with flags bit 0 clear:

- The count is read before the flag test.
- No callback occurs.
- This function performs no write to offset `0x3C`.
- The callback fields need not be accessed by this path.

An inaccessible count field is not made safe by disabling callbacks.

### Enabled zero count

Given bit 0 set and an initial count of zero:

- The local decrement produces `−1`.
- No callback occurs.
- Offset `0x3C` is not directly read or written by the loop body.
- The callback-target and callback-argument fields are not accessed by an iteration.

### Enabled positive count with stable state

Given bit 0 set, count 3, a valid stable callback target, stable second argument, and an initial progress value of 7:

If every callback returns without changing the relevant fields and arithmetic remains representable:

- Exactly three callbacks occur.
- Each receives first argument 32 and the same second argument.
- Post-callback progress values become 8, 9, and 10, in that order.
- The stored count remains 3 unless some external effect changes it.

### Callback changes later call selection

Given an initial count of 2:

If the first callback changes the encoded target and the second-argument field, and both calls remain valid:

- The first call uses values obtained for the first iteration.
- The second call uses values obtained for the second iteration.
- The initially selected callback is not guaranteed to receive both invocations.

### Callback changes progress

Given an initial count of 1:

If the callback sets offset `0x3C` to 100 and returns normally, the subsequent update writes 101, provided no intervening mutation or arithmetic failure occurs.

The result is not derived from the field’s value before the callback.

### Callback disables the flag or replaces the stored count

Given an initial enabled count of 2:

If the first callback clears flags bit 0 and stores zero at offset `0x34`, a second iteration still follows when execution otherwise remains valid. Neither field is reread for loop control.

### Ordinary negative count

Given an enabled initial count of `−1`, its first decrement produces `−2`, which fails the nonnegative test. No callback or progress update occurs.

This example does not cover the minimum signed 32-bit count.

## Uncertainties

### Established static evidence

The supplied representation establishes field widths and offsets, initial read ordering, the single flag test, local repetition state, target masking, callback arguments, and the post-callback read and write.

### Inferred intent

The count appears to request a number of repeated operations, and offset `0x3C` appears to track progress or completed calls. These are interpretations, not established domain semantics.

Clearing two target bits may relate to an encoded pointer or execution convention. The reason is not established.

### documented_only

The following remain unresolved:

- Exact public parameter type and calling convention.
- Byte order and alignment requirements.
- Callback identity, return type, and argument semantics.
- Callback ownership and lifetime behavior.
- Recovery, if any, from access faults or invalid calls.
- Whether the object is subject to concurrent access.
- Machine-level behavior when decrementing the minimum signed count.
- Machine-level behavior when incrementing a progress value of `2,147,483,647`.

In particular, wrapping to the signed minimum is not promised for progress overflow. Nor is a large callback count promised for minimum-count decrement overflow.

## Evidence

This contract is based solely on the supplied static function representation. That representation shows one captured count, one flags test, repeated indirect calls through a low-two-bit-masked target, and a 32-bit increment after each normally returning call.

No callback implementation, complete object definition, caller contract, or runtime trace was supplied. No runtime tests have been performed. The acceptance examples describe conditional obligations supported by the static evidence, not observed execution results.
