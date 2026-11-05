<!-- Trusted provenance: F-2cc38aeae003; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2cc38aeae003

## Purpose

F-2cc38aeae003 coordinates processing of an input object through a subordinate reference stored in an owner object. It conditionally initializes that reference, performs a flag-dependent operation, obtains a primary result and a normalization predicate, and conditionally invokes an owner callback followed by a notification dependency.

The established contract concerns data movement, call ordering, branch conditions, and observable memory accesses. The application-level meanings of the objects, fields, global value, and results are not established.

Every explicit normal return produces the 32-bit value **1**. This value does not distinguish initialization, skipped operations, zero results, callback execution, or notification execution.

## Interface

### Arguments and memory layout

The function accepts an **owner address** and an **input address**. The supplied representation supports a 32-bit target model; it does not establish a wider-pointer ABI.

| Object | Required accessible layout |
|---|---|
| Owner | A first 32-bit word used as a dispatch-table address, and a 32-bit subordinate-reference field at byte offset **0xA4**—164 bytes. |
| Input | A 32-bit data field at byte offset **0x0C**—12 bytes—and a 32-bit flag field at byte offset **0x10**—16 bytes. |

The subordinate-reference field must be readable at entry. It must also be writable if its initial value is zero. Later paths read this field again.

A path reaching the owner callback additionally requires:

- A readable dispatch-table address in the owner’s first word.
- A readable callable entry at byte offset **0x140**—320 bytes—from that dispatch-table address.
- A target compatible with the visible arguments: owner address, selected signed 32-bit result, and integer constant **1**.

The function also reads a value from a **global storage location** on the initialization path. That value supplies the fourth argument to F-1058afc38c04. Its value and semantics are unknown. The supplied material does not include a declaration independently establishing the global’s storage width or type.

No explicit null, bounds, alignment, dispatch-target, or dependency-success validation is present. Each reached access and call must be valid when it occurs. Entry-time validity alone is insufficient if an intervening dependency changes or invalidates relevant state.

### Result and integer interpretation

The explicit normal result is the 32-bit value **1**.

The flag test uses the unsigned 32-bit mask **0x40000000**, selecting bit 30. Other flag bits do not affect this branch.

The primary result is interpreted as a **signed 32-bit integer**. The normalization predicate and callback result are tested only for zero versus nonzero.

Normalization replaces a negative primary result with **−1** and a positive primary result with **1**. It does not negate the primary result or compute its magnitude. The most-negative signed 32-bit value therefore requires no exceptional arithmetic treatment.

No alternate normal error return or explicit error object is shown. Dependency failures, exceptions, faults, and nonlocal exits are outside the established normal-return behavior.

## Behavior

### Conditional initialization and global sampling

The first explicit memory access reads the owner’s subordinate-reference field at offset 0xA4.

If this value is nonzero, the entire initialization sequence is skipped. In particular, the function does **not** perform the initialization-specific global read on that path.

If the value is zero, the following sequence occurs:

1. F-8157a180d901 is called with **0x34**, decimal **52**.
2. After that call returns, the function reads the value held in the global storage location.
3. F-1058afc38c04 is called with four arguments: the captured result of F-8157a180d901, **4**, **1**, and the sampled global value.
4. The 32-bit result of F-1058afc38c04 is stored in the owner’s subordinate-reference field.
5. F-991ae23131ae is called with that captured result and **2**.

The global argument is neither a constant nor an entry-time snapshot. F-8157a180d901 may change the global before the read, and the fourth argument must reflect the value sampled **after** that dependency returns. The function supplies the value read from global storage, not an established address-of-global argument.

The store to the owner occurs before F-991ae23131ae begins. That dependency receives the captured result of F-1058afc38c04, rather than a fresh read of the owner field. Later operations do reread the owner field, so changes made by F-991ae23131ae can affect subsequent calls.

There is no explicit check for a zero result from either creation-stage dependency. If F-1058afc38c04 returns zero normally, the function stores zero and calls F-991ae23131ae with zero and 2. It does not retry initialization or take a separate failure branch.

The constant 52 is established as an argument only. Interpreting it as an allocation size is not part of the established contract.

### First input preparation and flag-dependent operation

After initialization or its omission, F-8bb3020950cd is called with the input address.

Only after that call returns does the function read the flag field at input offset 0x10:

- If bit 30 is **clear**, the current owner subordinate reference is read and passed to F-849b643bed3c.
- If bit 30 is **set**, F-849b643bed3c is skipped.

A flag change made by F-8bb3020950cd therefore affects this branch. The function does not preserve or consult an entry-time flag snapshot.

The reference supplied to F-849b643bed3c is also current at this stage. It need not equal the reference initially observed or the value stored during initialization.

### Token capture and second input preparation

Next, F-c0a513a969e0 is called with the input address. Its 32-bit result is retained as an opaque **token**.

F-8bb3020950cd is then called with the input address a second time. This second call occurs regardless of the flag branch and regardless of the token value, provided preceding calls return normally.

The token remains the value captured before the second preparation call. Later input-data reads occur after that call. The token and input data therefore need not represent the same moment in the input’s state.

### Primary result and normalization predicate

After the second preparation call, F-aa672b15efb7 is called with:

1. The owner’s current subordinate reference.
2. The current 32-bit input field at offset 0x0C.
3. The captured token.

Its result is retained as the signed 32-bit **primary result**.

The subordinate-reference and input-data reads both occur after the second preparation call and before F-aa672b15efb7 is invoked. Their relative order is not established by the supplied representation.

After F-aa672b15efb7 returns, F-be0ad8f71bac is called with:

1. A fresh read of the owner’s subordinate reference.
2. The same captured token.

Its result becomes the **normalization predicate**.

The subordinate reference is not preserved across these two calls. If F-aa672b15efb7 changes the owner field, F-be0ad8f71bac receives the later value. No assumption that either dependency preserves globals or other reachable memory is justified.

### Result selection

Both result-producing dependencies run before any zero-primary-result return.

| Normalization predicate | Primary result | Continuation |
|---|---:|---|
| Zero | Zero | Return 1 without owner callback or notification. |
| Zero | Negative | Continue with the unchanged primary result. |
| Zero | Positive | Continue with the unchanged primary result. |
| Nonzero | Zero | Return 1 without owner callback or notification. |
| Nonzero | Negative | Continue with selected result −1. |
| Nonzero | Positive | Continue with selected result 1. |

Thus zero always suppresses callback dispatch. A nonzero normalization predicate does not independently request a callback; it changes the selected argument only when the primary result is nonzero.

### Owner callback and subsequent notification

For a nonzero primary result, the function reads the owner’s current dispatch-table address, then obtains the callback target from its slot at offset 0x140.

The callback receives:

1. The owner address.
2. The selected signed result.
3. The constant **1**.

Dispatch is resolved after the processing dependencies. An earlier dispatch-table address or callback target is not retained by this function.

If the callback returns zero, no notification call occurs.

If the callback returns nonzero, the function calls an additional notification dependency with the selected result as its sole visible argument. This dependency has no assigned opaque function ID in the supplied mapping; its identity remains unresolved rather than being invented here.

The notification receives the selected result, not the callback’s return value. If normalization selected −1 or 1, that normalized value is used by both calls.

Following a normal callback return and, where applicable, a normal notification return, the function returns **1**.

## Side effects

The only explicit direct data write shown is the conditional 32-bit store to the owner’s subordinate-reference field.

The explicit global read is also behaviorally significant:

- It occurs only when the initial subordinate reference is zero.
- It occurs after F-8157a180d901 returns.
- It supplies the fourth argument of F-1058afc38c04.
- Its value is not cached at entry or reused by another visible operation.

All other effects arise through dependency calls or the owner callback. Their implementations are unavailable, so their effects on owner memory, input memory, subordinate objects, globals, and other reachable state remain unconstrained by this evidence.

Important state-ordering properties include:

- Initialization stores the reference before calling F-991ae23131ae.
- Subsequent operations reread the owner field instead of retaining one subordinate reference for the entire invocation.
- Flags are sampled after the first preparation call.
- Input data is sampled after the second preparation call.
- The token is captured before the second preparation call and retained across later calls.
- Callback dispatch uses the owner’s then-current dispatch state.
- A zero primary result does not undo initialization or earlier dependency effects.

No direct destruction, release, rollback, or replacement cleanup is shown. This does not establish that dependencies perform no such actions.

Ownership and lifetime rules are **documented_only**. The source does not establish whether the creation-stage calls allocate storage, whether the owner acquires ownership, whether arguments are consumed, or whether dependencies release memory.

No synchronization or reentrancy protection is visible. Concurrent access and reentrant mutation are not characterized.

## Dependencies

| Dependency | Visible contract at this call site |
|---|---|
| **F-8157a180d901** | Initialization-only call with 52. Its result is captured before the initialization global is read. It may change that global. |
| **F-1058afc38c04** | Initialization-only call receiving the first dependency’s result, 4, 1, and the subsequently sampled global value. Its result is stored in the owner. |
| **F-991ae23131ae** | Called after that store with the captured stored value and 2. |
| **F-8bb3020950cd** | Called twice with the input address, at distinct points in processing. |
| **F-849b643bed3c** | Receives the current subordinate reference only if bit 30 is clear after the first preparation call. |
| **F-c0a513a969e0** | Receives the input address and supplies the captured token. |
| **F-aa672b15efb7** | Receives current subordinate reference, current input data, and token; supplies the signed primary result. |
| **F-be0ad8f71bac** | Receives a freshly read subordinate reference and the captured token; supplies the normalization predicate. |
| Owner callback, opaque ID unavailable | Indirect target selected through dispatch-table offset 0x140; receives owner, selected result, and 1. |
| Notification dependency, opaque ID unavailable | Called only after a nonzero callback result; receives the selected result. |

Dependency argument meanings, failure behavior, ownership, and global-preservation guarantees are not supplied. The primary result is not established as an error code, count, comparison, or status.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, **not executed tests**. Each assumes valid reached accesses, compatible call targets, and normal return from every reached dependency.

1. **Existing subordinate reference.**  
   With an initially nonzero reference, the initialization calls, initialization store, and initialization-specific global read are skipped. The first preparation call still occurs.

2. **Global changed by the first initialization dependency.**  
   Suppose the reference is initially zero, the global initially contains value A, and F-8157a180d901 changes it to B before returning. If B is the value present when the global is read, F-1058afc38c04 receives B as its fourth argument—not A. Its other arguments are the captured first-stage result, 4, and 1.

3. **Store visible to initialization follow-up.**  
   If F-1058afc38c04 returns reference R, the owner field is written with R before F-991ae23131ae is called with R and 2. If that dependency changes the field to S, later field reads observe the then-current value rather than automatically reusing R.

4. **Preparation changes the flag.**  
   If bit 30 is set at entry but the first F-8bb3020950cd clears it before the flag read, F-849b643bed3c is called. If preparation instead leaves bit 30 set, that call is skipped.

5. **Zero primary result.**  
   With primary result zero, both result-producing dependencies have already run. Regardless of the normalization predicate, neither callback nor notification occurs, and the normal return is 1.

6. **Unnormalized positive result.**  
   With primary result 37 and predicate zero, the callback receives owner, 37, and 1. A zero callback result suppresses notification.

7. **Normalized negative result.**  
   With primary result −37 and any nonzero predicate, the callback receives −1. If it returns nonzero, notification also receives −1, not −37 or the callback result.

8. **Signed lower boundary.**  
   With primary result −2,147,483,648, predicate zero preserves that exact value. A nonzero predicate selects −1 without negating the original value.

9. **Reference changes between result calls.**  
   If F-aa672b15efb7 changes the owner field from R to S, F-be0ad8f71bac receives S. Both calls receive the same captured token.

10. **Zero initialization result.**  
    If F-1058afc38c04 returns zero, zero is stored and passed to F-991ae23131ae with 2. No explicit creation-failure return occurs. Validity of subsequent operations depends on the reached dependencies’ contracts.

## Uncertainties

The following remain **documented_only**:

- Application-level types and meanings of the owner, input, subordinate reference, token, and results.
- The initialization global’s value, semantics, declared storage width, and lifetime.
- Whether 52 is an allocation size.
- Dependency failure behavior, memory effects, global effects, ownership, and lifetime guarantees.
- The exact identity and broader behavior of the notification dependency.
- Callback ABI details beyond the visible arguments.
- The relative order of the two memory reads supplying the first two arguments to F-aa672b15efb7.
- Whether owner and input storage may alias.
- Thread safety, reentrancy, and behavior under concurrent mutation.

The global’s **origin and timing are established**, despite its unknown meaning: it is read from global storage after F-8157a180d901 returns and before F-1058afc38c04 is invoked, only on initialization.

A plausible intent is lazy subordinate-object initialization followed by result-dependent owner notification. That is an inference, not an additional guarantee.

## Evidence

This contract rests solely on the supplied static function body and dependency-to-opaque-ID mapping.

Established static evidence includes field offsets, the 32-bit target representation, constants, initialization-path global sampling, store placement, dependency-call ordering, repeated owner-field reads, flag masking, signed normalization, indirect dispatch, notification gating, and the constant normal return.

No dependency implementations, global declaration, runtime traces, ownership contracts, or independent ABI documentation were supplied. **No runtime tests have been performed.**
