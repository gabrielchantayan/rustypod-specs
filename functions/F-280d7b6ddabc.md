<!-- Trusted provenance: F-280d7b6ddabc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-280d7b6ddabc

## Purpose

F-280d7b6ddabc conditionally coordinates a notification and a possible state update on an object acquired from a dependency. It accepts or rejects the request according to an unsigned 32-bit selector, normalizes four accepted selector values, obtains the object's current value, issues one of two notifications, and invokes an indirect setter only when the previously obtained value differs from the normalized request.

The established behavior is this coordination sequence. Interpreting the selector as an application mode, or treating the notification as an event announcing a mode transition, is inferred intent rather than an established interface meaning.

A normal return of zero means the accepted path reached its end. It does not establish that a notification succeeded or that the object's resulting state equals the requested value. A return of four identifies the selector-rejection path.

## Interface

The function has four inputs and a 32-bit result:

| Input | Established representation | Use |
|---|---|---|
| Context | Opaque 32-bit value | Passed unchanged as the first argument to the selected notification dependency. |
| Selector | Unsigned 32-bit integer | Determines acceptance, normalization, and the value supplied to the optional setter. |
| Notification discriminator | 32-bit integer | Selects the notification dependency solely by whether it equals zero. |
| Payload | Opaque 32-bit value | Passed unchanged to the nonzero-discriminator notification dependency; otherwise unused. |

The discriminator is not restricted to a Boolean encoding. Every nonzero bit pattern selects the same branch, including values interpreted as negative signed integers.

No pointer validity requirement can be established for the context or payload merely from their use here: this function forwards their 32-bit values without directly dereferencing them. Their meanings and any validity requirements belong to the selected dependency.

The object returned by F-b7af80e829ec must support the indirect accesses described below for execution to complete normally. No explicit null check, object validation, or dispatch-target validation is present.

The normal result values are:

- **4:** the selector is in the rejected interval, inclusive from 4 through 252.
- **0:** the selector is accepted, and all calls taken on that path return control normally.

There is no separate result for “already matched,” “setter invoked,” or a dependency-reported failure.

## Behavior

### 1. Gate the request using the original selector

Acceptance uses the full unsigned 32-bit selector, not an eight-bit truncation.

The accepted set is:

- 0 through 3, inclusive; and
- 253 through 4,294,967,295, inclusive.

The rejected set is 4 through 252, inclusive.

For a rejected selector, the function returns four without acquiring an object, invoking either notification dependency, or performing either indirect method call.

Consequently, values above 255 are accepted. The hexadecimal constants used for normalization do not establish an eight-bit interface.

### 2. Acquire an object and normalize the request

For an accepted selector, the first dependency call is F-b7af80e829ec with the single numeric argument 4. Its returned object reference is retained for the subsequent indirect calls.

After that dependency returns, the selector is normalized as follows:

| Original selector | Normalized selector |
|---|---|
| 2 or 3 | 1 |
| 253 or 254 | 255 |
| Any other accepted value | Unchanged |

Normalization does not affect whether the request was accepted; acceptance has already been decided using the original value.

Thus 0, 1, 255, 256, and 4,294,967,295 all retain their original values. Neither saturation nor general byte masking is established.

### 3. Read a current value through indirect dispatch

The function reads the acquired object's dispatch-table reference and invokes the entry located at byte offset 0x60 from that table's base. The object reference is the sole explicit argument.

The result is captured as an unsigned 32-bit value. All 32 bits participate in the later comparison.

The indirect target's identity and implementation are not supplied. Describing this call as a “getter” refers only to its role in this function: its result is used as the current value. It is not evidence that the call is read-only.

This call occurs before either notification dependency.

### 4. Issue exactly one notification

The discriminator selects one of two direct dependencies:

- **Discriminator equals zero:** call F-0c4d2e445922 with the context and numeric argument 14.
- **Discriminator is nonzero:** call F-bffb4f8e6161 with, in order, the context, numeric argument 14, payload, captured current value, and numeric argument 0.

The nonzero branch receives the captured current value, not the normalized selector.

Neither dependency's return value affects the subsequent comparison or the function's normal result. There is no explicit branch on a notification outcome.

The notification is issued even if the captured current value already equals the normalized selector. Conversely, any optional setter call occurs only after the notification returns normally.

### 5. Compare the saved value and conditionally dispatch a setter

The function compares the captured unsigned 32-bit current value with the normalized selector.

- If they are equal, it does not invoke the setter.
- If they differ, it obtains the dispatch-table reference from the retained object again and invokes the entry at byte offset 0x5c, passing the object reference and normalized selector.

The second dispatch lookup occurs after the notification. The function does not preserve the earlier dispatch-table reference for this call.

The comparison nevertheless uses the value captured before the notification. There is no second getter call. If the notification changes the object's current state, that change does not itself alter the comparison criterion.

The setter's result, if any, is not used. Following normal completion of the notification and any required setter call, the function returns zero.

## Side effects

### Observable ordering

On an accepted request that completes normally, the ordering is:

1. Call F-b7af80e829ec with 4.
2. Read the returned object's dispatch-table reference and the target at offset 0x60.
3. Invoke that target with the object reference and capture its 32-bit result.
4. Invoke the notification dependency selected by the discriminator.
5. Compare the saved result with the normalized selector.
6. Only if different, read the object's dispatch-table reference again, read the target at offset 0x5c, and invoke that target with the normalized selector.
7. Return zero.

This ordering is significant when any dependency has side effects. In particular, the notification precedes the setter and can affect the object or its dispatch information before the second lookup.

The supplied body contains no explicit write to an object field, caller-provided memory, or a named global. Its externally visible mutations may occur through its dependencies and indirect calls; those mutations cannot be enumerated from the available evidence.

### Ownership and lifetime

There is no explicit allocation, reference-count adjustment, destruction, or release operation in this function beyond whatever its dependencies may perform.

F-b7af80e829ec returns an object reference, but the evidence does not establish whether that reference is borrowed, newly owned, shared, or otherwise managed. The absence of an explicit release here does not establish a leak or an ownership transfer.

Normal completion requires the retained object reference to remain usable through every subsequent access that is actually taken. In particular, a notification that invalidates the object can make the later setter dispatch unsafe. The function provides no visible lifetime guard around the notification.

No dependency may be assumed to preserve globals, preserve the object's state, or retain or release memory in any particular way.

### Failure handling

There is no explicit handling for:

- An invalid object returned by F-b7af80e829ec.
- An invalid dispatch table or indirect target.
- A failure reported by either notification dependency.
- A failure reported by an indirect method.
- A dependency that does not return normally.

The evidence establishes neither a defined error result nor recovery behavior for these conditions. Zero and four describe normal control-flow outcomes, not universal outcomes in the presence of invalid memory or abnormal dependency behavior.

## Dependencies

| Dependency | Established interaction |
|---|---|
| F-b7af80e829ec | Called once with 4 on every accepted request, before the indirect current-value call. Supplies the object reference. |
| F-0c4d2e445922 | Called once on accepted requests whose discriminator is zero, with context and 14. |
| F-bffb4f8e6161 | Called once on accepted requests whose discriminator is nonzero, with context, 14, payload, captured current value, and 0. |
| Unresolved indirect target at offset 0x60 | Called once per accepted request with the acquired object. Produces the 32-bit value used for notification and comparison. |
| Unresolved indirect target at offset 0x5c | Called only when the saved current value differs from the normalized selector. Receives the acquired object and normalized selector. |

No opaque function IDs are supplied for the indirect targets. Their table offsets identify dispatch positions, not globally stable function identities.

The visible layout requires an initial object field usable as a dispatch-table reference and callable entries at the two stated byte offsets. The evidence does not establish the remainder of the object layout, a complete class interface, or a portable pointer-width requirement.

## Acceptance examples

These are conditional contract examples, not executed tests. Examples involving dependencies assume valid objects and targets, with every invoked dependency returning normally.

1. **Lower rejected boundary.** With selector 4, any discriminator, and arbitrary context and payload, the result is four. No dependency or indirect method is called.

2. **Upper rejected boundary.** Selector 252 also returns four without external calls.

3. **Low-value normalization with no setter.** With selector 3 and a current-value result of 1, the normalized selector is 1. The selected notification occurs, the setter is omitted, and the result is zero.

4. **Low-value normalization requiring a setter.** With selector 2, discriminator zero, and a current-value result of 2, F-0c4d2e445922 receives context and 14. The setter is then invoked with 1, and the function returns zero.

5. **High-value normalization.** With selector 254 and a current-value result of 7, the normalized request is 255. For a nonzero discriminator, F-bffb4f8e6161 receives the captured value 7 in its fourth argument. The later setter receives 255.

6. **Accepted value above one byte.** With selector 256 and a current-value result of 0, the request is accepted and remains 256. After notification, the setter receives 256—not zero.

7. **Full-width upper boundary.** With selector 4,294,967,295 and the same current-value result, notification still occurs, no setter is invoked, and the result is zero.

8. **Notification changes state to the requested value.** Suppose selector 1 produces a saved current value of 0, and the notification changes the object's state to 1 while leaving it valid. The setter is still invoked with 1 because the saved value differs.

9. **Notification changes state away from the requested value.** Suppose selector 1 produces a saved current value of 1, and the notification changes the state to 0. The setter is omitted because the saved value matched. Zero does not guarantee a final state of 1.

10. **Notification changes dispatch information.** If the saved value differs and the notification validly replaces the object's dispatch-table reference, the setter target is obtained from the reference present after that notification.

## Uncertainties

The following are **documented_only** because the supplied static evidence does not resolve them:

- The domain meaning of selectors, including 0, 1, and 255.
- The meanings of numeric arguments 4 and 14 at their respective dependency boundaries.
- The object acquisition dependency's ownership and failure conventions.
- The implementations and side effects of both indirect targets.
- The context and payload's semantic types.
- Dependency effects on global state, object lifetime, dispatch information, or memory ownership.
- Thread safety, synchronization, reentrancy, and behavior under concurrent mutation.
- The exact machine calling convention and portable pointer representation.
- Defined behavior following invalid memory access or abnormal dependency termination.

A getter/notification/setter interpretation is useful inferred intent, but it must not imply purity, successful state commitment, or ownership guarantees absent from the evidence.

## Evidence

This contract is based on static inspection of the supplied function body and dependency mapping. Established facts include the unsigned acceptance range, four normalization cases, direct-call arguments, two dispatch offsets, notification-before-setter ordering, saved-value comparison, and normal result values.

No dependency implementations were supplied. Claims about their internal behavior, success conditions, or memory-management responsibilities remain unresolved.

No runtime tests have been performed. The acceptance examples specify expected conditional behavior; they are not observations of execution.
