<!-- Trusted provenance: F-d9bb88a0a8d3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d9bb88a0a8d3

## Purpose

F-d9bb88a0a8d3 decrements a context-associated counter and, when the computed result is exactly zero, processes work values and traverses a notification list. After that conditional phase, it independently checks auxiliary state, consults a gate dependency, and may decrement a second counter and invoke a target obtained from a global function-pointer slot.

These are established control-flow relationships, not application-level interpretations. The evidence does not establish that either counter represents ownership, nesting, outstanding work, or a reference count.

The ordering specified below concerns operations directly performed by this invocation’s visible body. Dependencies and callbacks may perform additional operations, including notifications or reentrant calls; their behavior is not established.

## Interface

### Input and result

The function accepts one 32-bit value used as the address of a context object. No explicit result value is supplied.

There is no visible validation of the context address before dereferencing it. Successful execution requires appropriate readable or writable storage for each access reached, and valid callable targets for calls taken.

The representation uses 32-bit addresses and four-byte scalar accesses. Offsets below are byte offsets. They identify accessed locations, not complete structure definitions or allocation sizes.

### Context fields

| Offset | Access width | Established role |
|---|---:|---|
| `0x24` | 4 bytes | Address of the current state object. |
| `0x90` | 4 bytes | First argument to a notification callback directly invoked by this body. |

The state reference is read at multiple points. It is not captured once and reused for the entire invocation.

### State fields

| Offset | Access width or form | Established role |
|---|---|---|
| `0x00` | 4 bytes | Notification-list head address. |
| `0x04` | Address of this location | Argument to F-eea31461590b. |
| `0x54` | 4 bytes | Entry counter, decremented before any conditional work processing. |
| `0x58` | 4 bytes | Optional auxiliary-object address. |

For F-eea31461590b, the argument is the state address plus `0x04`. It is not a pointer value loaded from that location. The size and layout of the region beginning there are unknown.

### Notification-node fields

| Offset | Access width | Established role |
|---|---:|---|
| `0x0c` | 4 bytes | Callback address, checked against zero before invocation. |
| `0x14` | 4 bytes | Next-node address; zero ends traversal. |

A callback directly invoked here receives two explicit arguments: the current four-byte context value at `0x90`, followed by zero. Its result is not used.

### Auxiliary-object fields

| Offset | Access width | Established role |
|---|---:|---|
| `0x00` | 4 bytes | Optional address of another object. |
| `0x08` | 4 bytes | Counter decremented after gate acceptance. |

If the address at `0x00` is nonzero, the function reads the first four-byte integer of the referenced object and tests it against zero.

That integer is a dispatch condition. It is not the source of the final callable target.

### Global callable storage

The final callable target is obtained from a global function-pointer slot. This slot is separate from the context, state, auxiliary object, and referenced object.

The slot is read on the final dispatch path, after the auxiliary decrement and referenced-object checks. There is no visible capture of its value at function entry and no visible null-target check.

## Behavior

### 1. Entry decrement

The function reads the context’s state reference and reads the signed 32-bit counter at state offset `0x54`. It computes that value minus one and writes the result to the counter location obtained through another read of the context’s state reference.

There is no intervening dependency call between the counter read and write. Nevertheless, the representation does not establish an atomic read-modify-write operation or make concurrent replacement of the state reference safe.

The computed value controls the next branch:

- A computed zero enters work processing and notification traversal.
- Any other computed value skips both and proceeds to the auxiliary check.

There is no pre-decrement zero check, lower-bound check, or saturation behavior.

For ordinary representable signed values, an incoming one becomes zero and enters the conditional phase; two becomes one and skips it; zero becomes negative one and also skips it.

The entry expression is represented using signed 32-bit arithmetic. The precise contract at the minimum signed 32-bit value is unresolved. Portable wraparound at that boundary must not be assumed.

### 2. Work processing when the computed counter is zero

The function calls F-eea31461590b with the address at offset `0x04` in the currently referenced state object.

Its returned 32-bit value determines whether processing continues:

- Zero ends this work-processing loop.
- Every nonzero value is passed first to F-796b82d1c424 and then to F-21122d06e906.

Both processing dependencies receive the original context address and the same returned work value. The second call occurs only after the first returns. The next fetch occurs only after both processing calls return.

Every fetch obtains its argument through a fresh read of the context’s state reference. A dependency that replaces that reference can therefore redirect a later fetch to another state object.

The work value is forwarded unchanged. Its integer representation does not establish whether it is an address, handle, identifier, or another kind of token.

There is no visible iteration bound or separate error classification. A nonzero return is treated as work regardless of its sign. Normal loop termination requires a fetch to return zero.

### 3. Notification traversal

After the terminating work fetch, the function reads the notification-list head from offset `0x00` of the then-current state object.

A zero head skips traversal. For each nonzero current node, the body:

- Reads the callback field at `0x0c` and checks whether it is zero.
- If nonzero, obtains the callback target from that field and invokes it with the current context value at `0x90` and a second argument of zero.
- After the callback returns, or immediately if it was skipped, reads the next-node address at `0x14`.

The representation shows a callback-field read for the null check and another use of that field to obtain the invocation target. It does not establish atomicity between those accesses or a safe concurrent-update protocol.

The next link is read after the callback returns. A callback can therefore change the link subsequently followed. If it invalidates the current node, continued execution still requires the later link access to be valid; no protection is visible here.

The context field at `0x90` is obtained for each direct callback invocation, not snapshotted for the traversal. Changes made by one callback can affect the argument to a later callback.

The list head is selected once when traversal begins. Replacing the context’s state reference during traversal does not itself restart traversal from a new head.

For this invocation’s visible body, all work-fetch and work-processing calls in the conditional phase precede its direct notification callbacks. This does not prohibit those dependencies from invoking notifications themselves or reentering F-d9bb88a0a8d3.

There is no visible cycle detection or traversal limit.

### 4. Auxiliary presence check and gate

The auxiliary phase is reached whether or not the entry decrement produced zero, provided earlier execution reaches this point.

The function reads the auxiliary address at offset `0x58` through the current state reference.

- If that address is zero, the function returns.
- Otherwise, it calls F-f4c5ae40135d with no explicit arguments shown.
- A zero gate result causes a return.
- Any nonzero gate result permits the remaining auxiliary operations.

After gate acceptance, the function reloads the auxiliary address through a fresh read of the context’s state reference. It does not reuse the address that passed the earlier presence check.

Neither the state reference nor the auxiliary address is established to be preserved by F-f4c5ae40135d. There is no second null check before accessing the reloaded auxiliary object. Gate acceptance therefore does not by itself prove that subsequent accesses are valid.

### 5. Auxiliary decrement and dispatch checks

Using the post-gate auxiliary address, the function reads the four-byte counter at `0x08`, subtracts one, and stores the four-byte result.

This counter is represented as a four-byte word rather than an explicitly signed integer. The stored result is the 32-bit decrement: a zero word becomes `0xffffffff`. There is no visible saturation or underflow rejection.

The newly decremented value is not tested and does not determine whether dispatch occurs.

After the decrement, the function reads the address at auxiliary offset `0x00`:

- If the address is zero, it returns without reading a referenced-object word.
- If the address is nonzero, it reads that object’s first four-byte integer.
- If that integer is zero, it returns.
- If both the address and the integer are nonzero, it proceeds to final dispatch.

Thus, both rejection paths occur after the auxiliary counter write. A nonzero first word satisfies a condition only; it is not interpreted here as the final callable address.

### 6. Final global-slot dispatch

After the preceding checks succeed, the function obtains its target from the global function-pointer slot and performs the represented indirect invocation.

No explicit arguments are shown, and no result is used. The complete target contract and calling convention are unresolved.

The target is not fixed at entry. Earlier dependencies, notification callbacks, and any reentrant execution are not established to preserve the global slot. A replacement made before this final read can change the target obtained by this invocation.

There is no visible validation that the slot contains a nonzero or executable address. Valid dispatch requires an appropriate target at the time it is obtained and used.

## Side effects

The visible body directly writes:

1. The entry counter at state offset `0x54`.
2. The auxiliary counter at auxiliary offset `0x08`, but only after a nonzero initial auxiliary reference and a nonzero gate result.

It also reads the global function-pointer slot on the successful final-dispatch path. No direct write to that slot is shown.

Dependencies and indirect targets may have additional effects on memory, globals, allocation lifetimes, or control flow. Their preservation behavior is not established.

The visible body contains no explicit allocation, release, or ownership transfer. Passing a work value to the processing dependencies does not establish that either dependency consumes or releases it.

There is no visible rollback mechanism. This is not a guarantee that earlier writes remain observable after a later failure or nonreturn: unknown callees may overwrite them, reverse their effects, invalidate their storage, or prevent later observation.

No error value is returned. Zero values have specific branch roles—work termination, list termination, callback omission, auxiliary absence, gate rejection, or dispatch rejection. Invalid nonzero addresses are not visibly converted into reported errors.

No synchronization is shown. Concurrent mutation and reentrant execution require external lifetime and ordering guarantees not established by this body.

## Dependencies

| Dependency | Established interaction | Unresolved behavior |
|---|---|---|
| F-eea31461590b | Receives current state address plus `0x04`; returns a 32-bit work value, with zero ending the loop. | Container semantics, ownership, mutations, failure behavior, and preservation of globals. |
| F-796b82d1c424 | Receives context and nonzero work value before F-21122d06e906. | Processing meaning, ownership effects, notifications, reentrancy, and other side effects. |
| F-21122d06e906 | Receives the same context and work value after F-796b82d1c424 returns. | Whether the work value remains usable afterward, allocation effects, and global mutations. |
| F-f4c5ae40135d | Called after the initial auxiliary-presence check; a nonzero result enables subsequent operations. | Gate meaning and preservation of state, auxiliary references, globals, or storage lifetimes. |
| Node-selected callback; opaque ID unavailable | Conditionally invoked with the current context field at `0x90` and zero. | Identity, complete calling convention, reentrancy, ownership effects, and mutations. |
| Global-slot-selected target; opaque ID unavailable | Invoked after the auxiliary decrement and both referenced-object checks succeed. | Identity, complete calling convention, result, side effects, and validity of its dynamically read slot value. |

No dependency is assumed to preserve the global target slot or any context-reachable storage merely because its explicit arguments omit those locations.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Assume valid accessed storage, representable entry arithmetic, returning calls where continuation is described, and no mutation of the named example fields unless stated otherwise.

Call-order claims concern calls directly issued by this invocation’s visible body. They do not exclude additional notifications or reentrant calls inside dependencies.

### Entry result remains nonzero

With an entry counter of two and a zero auxiliary reference:

- The entry counter becomes one.
- This invocation directly issues no work-fetch, work-processing, or notification calls.
- It does not call F-f4c5ae40135d.

A valid nonzero auxiliary reference instead permits the gate call, despite the entry result remaining nonzero.

### Empty work source

With an entry counter of one, an initial fetch result of zero, and a valid notification chain:

- The entry counter becomes zero.
- This invocation calls F-eea31461590b once and neither work-processing dependency.
- It then traverses the chain, directly invoking non-null callbacks in the encountered link order.
- A node with a null callback still has its next link read.

### Two work values

With an entry counter of one and fetch results comprising two nonzero values followed by zero:

- Each value goes first to F-796b82d1c424, then to F-21122d06e906.
- The next fetch follows both calls for the preceding value.
- This invocation begins its own notification traversal only after the terminating fetch.

Dependencies may independently invoke notifications during these calls.

### Gate rejection

With a nonzero initial auxiliary reference and a zero gate result:

- F-f4c5ae40135d is called.
- The visible body performs no auxiliary decrement or final dispatch.

This does not constrain mutations performed inside the gate.

### Missing referenced object

With gate acceptance, an auxiliary counter of seven, and a zero referenced-object address:

- The auxiliary counter is written as six.
- No referenced-object word is read.
- The global target slot is not read for final dispatch.

The visible body performs no rollback of that decrement.

### Dispatch is independent of counter zero

With gate acceptance, an auxiliary counter of seven, a valid referenced-object address, and a nonzero first word:

- The counter becomes six.
- The function obtains the final target from the global slot and invokes it.

Changing only the referenced object’s first word to zero suppresses that dispatch, not the preceding decrement.

### Gate replaces the state reference

If the gate returns nonzero after replacing the context’s state reference, subsequent auxiliary access uses the replacement state’s auxiliary address. A zero replacement auxiliary address is not checked again before access.

### Earlier call replaces the global target

If an earlier dependency or callback replaces the global function-pointer slot, and execution later satisfies the final checks without another replacement, final dispatch obtains the replacement target. The referenced object’s nonzero first word remains only a condition; it does not supply that target.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of both counters and the referenced-object word.
- Entry arithmetic at the minimum signed 32-bit value.
- Complete object layouts, alignment requirements, and allocation sizes.
- Work-value representation and ownership.
- Dependency effects on globals, including the final target slot.
- Storage preservation, reentrancy safety, and concurrent-use validity.
- Work-loop and list-traversal termination.
- Identities and complete calling conventions of indirect targets.
- Exact failure behavior for invalid addresses or nonreturning dependencies.
- Atomicity and memory-ordering guarantees.
- The final indirect transfer’s complete low-level call behavior beyond the represented invocation.

The zero-transition phase may serve deferred-work or nesting-related purposes. That is inferred intent, not an acceptance requirement.

## Evidence

The supplied static representation establishes field offsets and widths, explicit counter writes, zero/nonzero branches, direct dependency ordering, notification-link timing, repeated state and auxiliary lookups, and the final global-slot dependency.

In particular, it establishes that:

- The auxiliary decrement precedes the referenced-object checks.
- The referenced object’s first word is tested, not used as the final target.
- The final target comes from separate global function-pointer storage after those checks.
- Notification-order guarantees apply to this invocation’s visible direct calls, not unknown callee behavior.
- The body contains no rollback; continued observability of earlier effects is not guaranteed.

No runtime tests have been performed. The acceptance examples describe conditional static behavior, not observed execution results.
