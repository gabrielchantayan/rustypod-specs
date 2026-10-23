<!-- Trusted provenance: F-2fcf02194bfd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2fcf02194bfd

## Purpose

F-2fcf02194bfd conditionally updates a caller-supplied integer slot using a value returned by a dependency. Its decision depends on whether the slot pointer and requested value are nonzero, whether the slot already equals the requested value, and whether the dependency produces a nonzero result.

The return value is a status indicator, not the resulting slot value:

- **0** indicates either an immediately rejected input or a zero slot observed on a path that does not perform replacement.
- **1** indicates either a nonzero slot accepted without replacement or completion of the replacement path.

The requested value is used only for the initial nonzero check and an equality comparison. It is not passed to either dependency and is not directly written into the slot. Consequently, success does not establish that the final slot equals the requested value.

An inferred intent is to retain an acceptable existing value or replace another value using shared context. Resource acquisition, release, reference counting, and ownership transfer are **not established** by the available evidence.

## Interface

### Inputs

The function accepts:

1. **Slot pointer** — a pointer to a writable integer object.
2. **Requested value** — an integer used as a nonzero gate and comparison target.

The visible representation uses an integer pointer and an integer argument. This contract describes their values and slot accesses under the apparent **32-bit integer model** of the supplied representation. Confirmation of the original ABI, integer size, alignment requirements, and calling convention is `documented_only`.

A null slot pointer is explicitly accepted as an input that returns 0. A zero requested value is also explicitly accepted as an input that returns 0. Neither case requires a slot dereference.

For other inputs, defined execution requires the slot pointer to identify a readable integer object. If replacement is reached, that object must also remain writable through the final store, including across both dependency calls. The function does not visibly validate accessibility, alignment, object lifetime, or write permissions.

### Shared input

The function accesses one shared context location when invoking its dependencies.

The context is loaded separately for each dependency invocation. There is no visible preservation of the first context value for use by the second dependency. A dependency may therefore change the context that a later invocation receives.

The context’s width, representation, validity requirements, and ownership are not established. It must not be assumed to be a pointer merely because the dependencies may plausibly operate on resources.

### Output

The visible result is a four-byte value with only two explicitly returned values:

- **0**
- **1**

No arithmetic transformation of the input or dependency result occurs. Tests distinguish zero from nonzero or compare integer equality. There is no visible signed ordering, addition, subtraction, or overflow-sensitive operation.

The dependency result used for replacement is an integer of the same apparent width as the slot. Every nonzero result, including a negative integer under a signed interpretation, selects replacement and is stored unchanged.

## Behavior

### Immediate rejection

If the slot pointer is null, the function returns 0 without reading the slot, accessing the shared context, or calling either dependency.

If the requested value is zero, the function likewise returns 0 without reading the slot, accessing the shared context, or calling either dependency.

The early zero-request rule applies even when the slot is valid and already contains a nonzero value.

### Initial slot comparison

For a nonnull slot pointer and a nonzero requested value, the function reads the slot and compares that value with the requested value.

If they are equal:

- F-a0437eff9ff9 is not called.
- F-ac669dc5909f is not called.
- The shared context is not accessed by the visible function body.
- The slot is tested again for zero.
- The function returns 0 if that later observation is zero, otherwise 1.

In an ordinary execution with a stable slot and no intervening external modification, equality with the nonzero requested value means this path returns 1. The supplied representation nevertheless contains a later slot test; the contract does not require an implementation to retain the earlier observed value as a snapshot.

The source-level repeated access does not, by itself, establish volatile semantics, atomicity, or a guarantee of two distinct hardware loads.

### Unequal slot: obtaining a candidate

If the initial slot value differs from the requested value, the function loads the shared context and calls F-a0437eff9ff9 with that context as its sole visible argument.

The initial slot value and requested value are not passed to this dependency.

There are two outcomes.

**Zero dependency result**

If F-a0437eff9ff9 returns zero, the function does not call F-ac669dc5909f and does not directly write the slot. It tests the slot after the dependency returns:

- A zero slot produces return value 0.
- A nonzero slot produces return value 1.

This is a post-call observation. The dependency is not known to preserve the slot, the shared context, or other memory. Thus, the returned status need not reflect the slot’s value before that call.

A zero dependency result is not independently reported as failure. The function may return 1 because the subsequently observed slot is nonzero.

**Nonzero dependency result**

If F-a0437eff9ff9 returns a nonzero integer, that integer is retained as the replacement value.

The function then calls F-ac669dc5909f with:

1. The slot value observed when preparing this call.
2. The shared context value observed when preparing this call.

These are fresh argument observations after F-a0437eff9ff9 returns. The first argument is not necessarily the slot value used in the initial equality comparison. The second argument is not necessarily the context previously supplied to F-a0437eff9ff9.

The relative evaluation order of these two argument reads is not established. Both arguments are prepared before F-ac669dc5909f begins.

After F-ac669dc5909f returns, the function writes the retained nonzero replacement value into the slot and returns 1.

No return value from F-ac669dc5909f is used to decide whether to perform that write or what status to return. If that dependency changes the slot and returns normally while leaving the destination valid, the final store overwrites its change.

There is no comparison between the replacement value and the requested value, nor between the replacement value and the value passed as the first argument to F-ac669dc5909f.

## Side effects

### Direct memory effects

The visible function body directly writes only one location: the caller-supplied slot.

That write occurs only when all of the following hold:

- The slot pointer is nonnull.
- The requested value is nonzero.
- The initial slot observation differs from the requested value.
- F-a0437eff9ff9 returns a nonzero integer.
- F-ac669dc5909f returns and execution reaches the store.

The stored value is the retained result from F-a0437eff9ff9, not the requested value.

The visible body does not directly assign to the shared context. Either dependency may have additional effects that are not available in the supplied evidence.

### Ordering and state changes

On the replacement path, the established order is:

- Initial slot comparison.
- Context read for F-a0437eff9ff9.
- Invocation and return of F-a0437eff9ff9.
- Preparation of the current slot and context arguments for F-ac669dc5909f.
- Invocation and return of F-ac669dc5909f.
- Store of the retained replacement value.
- Return of 1.

No rollback or compensating action is visible. If a dependency does not return normally, the normal-path final store and return are not established.

### Ownership and synchronization

The function does not visibly allocate or free memory itself. Whether either dependency acquires, releases, retains, transfers, or invalidates anything is unknown.

In particular, the call to F-ac669dc5909f must not be specified as releasing the previous slot value without additional evidence. Its first argument can be zero if the slot is zero when its arguments are prepared.

No locking, atomic operation, memory barrier, or concurrency protocol is visible. The function must not be treated as an atomic compare-and-replace operation. Concurrent access requirements remain `documented_only`.

## Dependencies

### F-a0437eff9ff9

- Called only after a nonzero requested value differs from the initial slot observation.
- Receives the shared context value read for that call.
- Returns an integer whose zero/nonzero classification selects the subsequent path.
- A nonzero result is retained for the final slot write.
- A zero result leads to a post-call slot test.
- Preservation of the slot, shared context, memory validity, and ownership is unknown.

### F-ac669dc5909f

- Called only after F-a0437eff9ff9 returns nonzero.
- Receives a current slot value and a separately read shared context value.
- Runs before the direct slot write.
- Has no return value used by the visible caller.
- Its effects, failure behavior, and ownership contract are unknown.

Neither dependency receives the requested value as a visible argument.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless stated otherwise, assume valid storage, normal dependency returns, and no unmentioned state changes.

1. **Null pointer:** With a null slot pointer and requested value 7, return 0. Do not call either dependency or access the shared context.

2. **Zero request:** With a valid slot containing 12 and requested value 0, return 0. Leave the slot untouched by the visible body and call neither dependency.

3. **Existing match:** With a slot containing 7 and requested value 7, return 1. Call neither dependency and perform no direct slot write.

4. **No candidate, empty slot:** With an initial slot of 0 and requested value 7, call F-a0437eff9ff9. If it returns 0 and leaves the slot at 0, return 0 without calling F-ac669dc5909f.

5. **No candidate, existing unequal value:** With an initial slot of 12 and requested value 7, if F-a0437eff9ff9 returns 0 and leaves the slot at 12, return 1. Success does not imply equality with 7.

6. **Replacement differs from request:** With an initial slot of 12 and requested value 7, if F-a0437eff9ff9 returns 25, call F-ac669dc5909f with the then-current slot and context. After it returns, store 25 and return 1.

7. **Post-call state determines fallback status:** With an initial slot of 12 and requested value 7, if F-a0437eff9ff9 changes the slot to 0 and returns 0, return 0. The initial nonzero slot does not determine the outcome.

8. **Changed arguments on replacement:** If F-a0437eff9ff9 changes the slot to 0, changes the shared context, and returns 25, F-ac669dc5909f receives 0 and the updated context. After normal return, the function stores 25.

9. **Overwrite after the second dependency:** If F-ac669dc5909f changes the slot to 99 and returns normally, while the retained candidate is 25, the final slot becomes 25.

10. **Negative candidate:** Under the apparent 32-bit integer model, a candidate of −1 is nonzero. It selects replacement and is stored unchanged.

## Uncertainties

The following remain `documented_only`:

- Original ABI, exact integer representation, pointer width, and alignment requirements.
- Shared context type, width, lifetime, and permitted values.
- Dependency semantics, global preservation, ownership effects, and error reporting.
- Whether dependencies can invalidate the slot or prevent normal return.
- Whether the slot can alias shared context storage and what such aliasing permits.
- Volatile, atomic, signal-handler, interrupt, or concurrent-access guarantees.
- Behavior for invalid nonnull pointers.

The apparent replacement pattern suggests resource management, but that interpretation is inferred intent, not an established contract.

## Evidence

Established static evidence consists of the supplied function representation: explicit early returns, short-circuit equality logic, two dependency call sites, separate shared-context expressions, a post-call slot test on the zero-result path, and a single direct slot assignment after the second dependency returns.

The opaque dependency mapping establishes the identities used in this document. It does not establish either dependency’s internal behavior.

No implementation outside the supplied representation was inspected. No runtime tests have been performed.
