<!-- Trusted provenance: F-dc63abd0393d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dc63abd0393d

## Purpose

F-dc63abd0393d checks two input words and, on one successful path, constructs and submits a four-word record. A non-null submission result may cause an indirect callback before being passed to a final dependency.

The function has three observable result classes:

- **0:** an unsuccessful check, unsuccessful initialization, null allocation, or nonzero status after a null submission result.
- **1:** completion of the submission path without the rejecting status condition.
- **Any other nonzero 32-bit value:** possible through the equal-input shortcut, which returns the second check’s result unchanged.

These results do not imply transactional behavior. Returning 0 does not undo earlier dependency effects. Returning 1 does not establish that submission returned an object.

Domain interpretations such as registration, expiration handling, or resource release are inferred intent only; none is established by the supplied representation.

## Interface

### Input

The function accepts a pointer to storage containing at least two accessible 32-bit words.

| Byte offset | Width | Use |
|---|---:|---|
| 0 | 32 bits | First input word |
| 4 | 32 bits | Second input word |

The function reads the first word before making any dependency call. It does not explicitly validate the pointer, storage accessibility, or alignment.

It makes no direct writes to the input storage. However, the first two checking calls receive the original input pointer. Later reads therefore cannot be assumed to reproduce entry values. Other dependencies may also affect accessible state; their preservation guarantees are unknown.

The supplied representation uses 32-bit integer words and pointer-like values. Transformation results and record payloads are opaque 32-bit values except where a mask or signed comparison explicitly establishes their treatment.

### Output

The return value is a 32-bit integer. Most exits return exactly 0 or 1. The equal-input shortcut returns the complete nonzero result of the second checking call, including a negative value if that dependency returns one.

No structured error, reason code, or output parameter is provided.

### Shared state and accessed layouts

Two shared values are directly accessed:

- A 32-bit configuration word.
- A shared pointer to a control block.

The control-block fields used by the function are:

| Byte offset | Width | Use |
|---|---:|---|
| 0 | 32 bits | Object address or handle used for submission and status lookup |
| 8 | 32 bits | Callback-related handle |

A null submission result leads to a 32-bit status read at byte offset 92 from the object address subsequently obtained through the cached control block.

A non-null submission result is used with this layout:

| Byte offset | Width | Use |
|---|---:|---|
| 0 | 32 bits | Signed comparison operand; later a callback argument |
| 8 | 32 bits | First callback argument |
| 12 | 32 bits | Third callback argument |

The function does not directly use the returned object’s word at offset 4.

All accessed storage must remain usable for the accesses actually reached. The function supplies no recovery path for invalid addresses, inaccessible fields, or invalid indirect-call targets.

## Behavior

### 1. Transform the first input and perform the first check

The function reads the first input word and passes it to **F-85ea43423066**. It saves the dependency’s 32-bit result for possible later use.

It then calls **F-b462ed996869** with:

1. The saved transformation result.
2. The literal value 1.
3. The original input pointer.

A zero result causes an immediate return of 0. No later stage is reached.

The saved transformation result is retained even if subsequent dependencies change the input.

### 2. Reread the first input and perform the second check

After the first check succeeds, the function reads the first input word again and passes that value to **F-5b103afaa2d2**.

It calls **F-b462ed996869** again with:

1. This new transformation result.
2. The literal value 1.
3. The original input pointer.

A zero result causes a return of 0.

The two transformations need not receive the same first-input value. In particular, the first checking call may modify the storage before the second transformation reads it.

### 3. Take the equal-input shortcut when applicable

After both checks succeed, the function compares the first and second input words as read at this stage.

If they are equal, the function returns the second check’s nonzero result without normalization.

This path performs no configuration read, third check, control-block availability handling, allocation, or submission. Equality concerns the input words, not their transformed values.

The supplied comparison does not establish an order between its two operand reads.

### 4. Transform the second input, read configuration twice, and perform the third check

If the input words compare unequal, the function reads the second input word and passes it to **F-85ea43423066**.

After that transformation returns, there are **two distinct reads of the shared configuration word**, in this order:

1. A configuration value is read and saved for eventual record construction.
2. The configuration word is read again to supply the second argument of the third checking call.

The function then calls **F-b462ed996869** with:

1. The transformation result obtained from the second input.
2. The value from the **second configuration read**.
3. The saved result from the initial transformation of the first input.

The third argument is not the input pointer passed to the first two checks.

There is no explicit dependency call between the two configuration reads. Nevertheless, the representation does not establish that they yield the same value. Without an external stability guarantee, the third check’s configuration argument may differ from the configuration saved for the record.

A zero third-check result causes a return of 0. A nonzero result permits continuation. The check result does not replace either saved payload or the saved record-construction configuration.

Changes to configuration after the first read do not change the value already saved for record construction.

### 5. Obtain the later payload and check shared-object availability

After the third check succeeds, the function reads the second input word again and passes it to **F-5b103afaa2d2**. Its 32-bit result is retained as a record payload.

Next, the function reads the shared control-block pointer and caches it for later submission, status, and callback-related accesses.

The availability expression separately reads the shared control-block pointer and examines the first word of the block addressed by that read:

- A nonzero first word skips initialization.
- A zero first word causes a call to **F-ea867029bc2c**.
- A zero initialization result causes a return of 0.
- A nonzero initialization result permits continuation.

Thus, caching the control-block pointer and selecting the block used by the availability test are distinct pointer reads. No intervening dependency call is shown, but equality of those pointer values requires a stability guarantee not supplied here.

After successful initialization, the function does not explicitly recheck the first word. Initialization’s nonzero result alone permits allocation.

Later accesses use the previously cached control-block pointer. Replacing the shared pointer does not redirect those accesses.

### 6. Allocate and populate the record

The function calls **F-43da6b23bd44** with arguments 16, 0, and 0.

A null result causes a return of 0 without submission. A non-null result is treated as writable storage for 16 bytes.

Four 32-bit stores occur in increasing offset order:

| Byte offset | Stored value |
|---|---|
| 0 | Configuration from the **first** configuration read, with bit 15 cleared |
| 4 | Configuration from the **first** configuration read, retaining only bit 15 |
| 8 | Result of the later F-5b103afaa2d2 call on the second input |
| 12 | Result of the initial F-85ea43423066 call on the first input |

The masks are respectively `0xFFFF7FFF` and `0x00008000`.

The field at offset 4 is either zero or `0x00008000`, not a Boolean normalized to 1. The field at offset 0 retains every other configuration bit, including bit 31. These operations partition the saved 32-bit configuration without addition, carry, or signed arithmetic.

The allocation request is fixed at 16 bytes. No input-dependent size calculation is visible. The stores use the native word representation; no byte-order conversion or portable serialization format is established.

### 7. Submit the populated record

After all four stores, the function reads the first word of the cached control block and calls **F-dbd019788d3d** with:

1. That freshly read object value.
2. The allocated record pointer.

The object value used for submission is not retained from the availability check. Initialization or allocation may have changed the cached block’s contents.

#### Null submission result

If submission returns null, the function rereads the first word of the cached control block and accesses the 32-bit status field at byte offset 92 from the resulting object address.

- A nonzero status returns 0.
- A zero status returns 1.

This lookup uses the object reference obtained after submission, which need not equal the object value passed into submission.

No explicit record cleanup, callback handling, or final-object call occurs on this branch. A null submission result alone is not classified as failure.

#### Non-null submission result

If submission returns non-null, the function reads the callback-related handle at offset 8 of the cached control block.

If that handle is zero, comparison and callback resolution are skipped.

If it is nonzero, the function calls **F-10b760faca36**. After that dependency returns, it reads the returned object’s first word and compares the two values as signed 32-bit integers.

The callback path is taken only if the object’s first word is **strictly less than** the dependency’s result. Equality does not qualify. This is not an unsigned comparison or a modular-difference test.

When the comparison succeeds:

1. The callback-related handle is reread from the cached control block.
2. That current value is passed to **F-6bbddce1b930**.
3. The resulting address is used to read a callable pointer at byte offset 8.
4. That callable is invoked with the returned object’s words at offsets 8, 0, and 12, in that argument order.

The handle passed for resolution can differ from the handle that enabled the path. It is not separately revalidated as nonzero.

The callback argument fields are read after resolution returns. Their evaluation order relative to one another is not established. In particular, the callback’s second argument need not equal the earlier comparison operand if a dependency changes the object.

No callback return value is used.

Every normally completed non-null-result path then calls **F-5990a26eaaa3** with the submission result and returns 1. No result from this final dependency influences the function’s return value.

## Side effects

The directly established writes are the four stores into the allocated record. The function makes no direct assignments to the input words, configuration word, or control-block fields.

Dependencies may have additional effects. No dependency is established to preserve globals, input storage, returned objects, or other reachable memory.

Ownership remains unresolved:

- The allocated record is passed to submission and is not explicitly released by this function.
- Submission may retain, consume, release, or otherwise use the record; its contract is unavailable.
- The final dependency receives every non-null submission result, but its call shape does not establish destruction or release.
- If the callback invalidates that object, the final call is still present in the control flow. Safety would require an external lifetime guarantee.

There is no explicit rollback after an unsuccessful exit. There is also no visible locking, atomic-access protocol, exception handling, pointer validation, or indirect-target validation.

The fixed offsets establish required accesses, not bounds checks. Arithmetic and validity rules for forming object addresses are not independently supplied.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| **F-85ea43423066** | Receives a 32-bit input word; its initial result is saved, and its conditional later result supplies the third check’s first argument. |
| **F-b462ed996869** | Called up to three times; zero rejects continuation. Its second result may be returned unchanged. The third call receives the second configuration read, not the saved record configuration. |
| **F-5b103afaa2d2** | Receives freshly read input words; its later result becomes the record field at offset 8. |
| **F-ea867029bc2c** | Called when the availability-test object value is zero; zero rejects continuation. |
| **F-43da6b23bd44** | Receives 16, 0, 0; its non-null result is used for four 32-bit stores. |
| **F-dbd019788d3d** | Receives a current object value from the cached control block and the populated record pointer. |
| **F-10b760faca36** | Supplies a signed 32-bit comparison value when callback handling is enabled. |
| **F-6bbddce1b930** | Receives the reread callback-related handle and returns an address containing a callable pointer at offset 8. |
| **F-5990a26eaaa3** | Receives each non-null submission result after optional callback handling. |

The indirect callback has no supplied opaque function ID. Only its three visible arguments and call position are established.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. They assume accessible storage, valid call targets, and dependencies that return normally.

1. **First or second check rejects.**  
   A zero first check prevents the second transformation. A zero second check prevents the equality comparison. Both paths return 0.

2. **Equal inputs preserve a noncanonical result.**  
   If the second check returns 7 and the subsequently compared input words are equal, the function returns 7. No configuration read or allocation occurs.

3. **Input mutation affects rereads, not saved results.**  
   If the first check changes the first input word, the next transformation receives the changed value. Any eventual record still contains the initial transformation result at offset 12.

4. **The two configuration reads can serve different values.**  
   Suppose the first configuration read yields `0x80008005` and the second yields `0x00000002`. The third check receives `0x00000002`. If processing continues, the record begins with `0x80000005` and `0x00008000`. The check argument does not replace the saved record value.

5. **Third-check mutation does not revise the record snapshot.**  
   If the third check changes configuration, the record fields at offsets 0 and 4 still derive from the first configuration read.

6. **Availability and cached-pointer identity are distinct.**  
   If the shared pointer changes between caching and the availability read, availability is tested through the later pointer. Subsequent submission and callback-related accesses still use the cached pointer.

7. **Initialization rejection prevents allocation.**  
   If the availability-test first word is zero and initialization returns zero, the function returns 0 without requesting the record.

8. **Allocation failure prevents submission.**  
   A null allocation result returns 0. None of the record stores or submission occurs.

9. **Null submission is conditionally successful.**  
   A null submission result followed by a zero status at the subsequently resolved offset 92 returns 1. A nonzero status returns 0. Neither path calls the final-object dependency.

10. **Callback threshold is strict and signed.**  
    With callback handling enabled, object value 20 and comparison result 20 skip the callback. Object value −1 and comparison result 0 take it.

11. **A zero handle skips comparison but not final processing.**  
    With a non-null submission result and an initially zero callback-related handle, neither comparison nor callback resolution occurs. The final dependency still receives the object, and the function returns 1.

12. **Handle replacement affects resolution.**  
    If F-10b760faca36 changes the handle and the comparison succeeds, F-6bbddce1b930 receives the replacement value, including zero if that is the replacement.

## Uncertainties

The following remain **documented_only**:

- Meanings of input words, transformation results, configuration bits, handles, and status values.
- Whether the two configuration reads are guaranteed to agree.
- Whether the two shared control-block pointer reads are guaranteed to agree.
- Dependency side effects, memory ownership, and global-preservation contracts.
- Whether successful initialization establishes a usable object in the cached control block.
- Whether the signed comparison represents time, priority, sequence, or another quantity.
- The callback’s identity, calling convention beyond visible arguments, and lifetime guarantees.
- Alignment, address-arithmetic rules, synchronization, and valid object lifetimes.
- Behavior when dependencies invalidate storage subsequently accessed by this function.
- Behavior on faults, nonreturning dependencies, or invalid indirect-call targets.

These unknowns must not be replaced with assumed allocator, container, timer, or destructor semantics.

## Evidence

The established behavior comes solely from the supplied static representation: direct reads and writes, call arguments, branch conditions, masks, signed comparison, and return expressions.

In particular, that representation distinguishes the first configuration read saved for record construction from the later read used as the third check’s second argument. It also distinguishes caching the control-block pointer from the subsequent pointer read used for availability testing.

The layouts, fixed allocation request, record-store order, later field rereads, and optional callback sequence are statically supported. Domain intent, dependency implementations, ownership contracts, and stability guarantees are not supplied.

No implementation access was used, and no runtime tests have been performed.
