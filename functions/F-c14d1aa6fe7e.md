<!-- Trusted provenance: F-c14d1aa6fe7e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c14d1aa6fe7e

## Purpose

F-c14d1aa6fe7e records two supplied values in a caller-provided object and selects one of three processing paths using an unsigned 32-bit selector.

- Selectors **0 and 1** attempt to obtain a temporary object, process a local two-word record through that object, perform an external operation directed at the caller’s object, clean up the local record, and invoke an indirect method on the temporary object.
- Selector **2** obtains two separate dependency results and stores them in two fields of the caller’s object.
- Selectors **3 through 4,294,967,295** perform only the initial field updates and return failure.

The result is a 32-bit status word: **1** for a completed selected path and **0** for an unsupported selector or a null result from the temporary-object-producing dependency. It does not establish that every dependency succeeded or that the caller’s object satisfies any broader application invariant.

This contract describes statically visible behavior. It does not establish the intended application role of the object, the meanings of its stored fields, or the semantics of the external transfer and indirect method.

## Interface

### Inputs

The function accepts four arguments:

| Position | Meaning in this contract | Representation |
|---|---|---|
| 1 | Base address of the caller-provided object | Address-like input represented as an integer in the supplied listing |
| 2 | Selector | Unsigned 32-bit value |
| 3 | Primary payload | Uninterpreted 32-bit word |
| 4 | Secondary payload | Uninterpreted 32-bit word |

The payloads are not visibly validated or interpreted as numbers, pointers, handles, or ownership-bearing values by this function.

The supplied representation is consistent with a 32-bit address space, but does not independently establish the complete calling convention or platform ABI.

### Return value

The function returns a 32-bit word:

| Condition | Result |
|---|---|
| Selector 0 or 1, and F-bb27b9d7a176 returns non-null, followed by normal completion of all remaining calls | 1 |
| Selector 0 or 1, and F-bb27b9d7a176 returns null | 0 |
| Selector 2, followed by normal completion of both construction paths | 1 |
| Any selector greater than 2 | 0 |

There is no explicit error object, exception translation, or separate error-code output.

### Caller-object layout

Offsets are byte offsets from the first argument. These are the directly visible accesses, not a complete object definition.

| Offset | Width | Visible use |
|---|---:|---|
| +0x04 | 4 bytes | Initially receives the primary payload; reread in the selector-0/1 success path |
| +0x08 | 4 bytes | Initially receives the selector |
| +0x0C | 4 bytes | Receives the first dependency result in the selector-2 path |
| +0x10 | 4 bytes | Receives the second dependency result in the selector-2 path |
| +0x14 | Unknown affected extent | Address supplied as the destination argument to an external operation in the selector-0/1 success path |

No direct access to the caller-object field at offset zero is shown.

The first argument must identify memory suitable for the accesses actually reached. There is no null check, bounds check, alignment check, or object-type check. The object’s minimum required size cannot be established merely from the destination address at +0x14 because the external operation’s access extent is unknown.

## Behavior

### Common prefix

Every invocation begins with these direct writes, in order:

1. Store the primary payload at caller-object offset +0x04.
2. Store the selector at caller-object offset +0x08.

Consequently, returning 0 does **not** mean that the caller’s object is unchanged.

Before entering a selector-specific path, the function also initializes a local record consisting of two adjacent 32-bit words:

- First word: primary payload.
- Second word: secondary payload.

This record is local to the invocation. Dependencies in the selector-0/1 success path receive the address of its first word. The listing does not establish that the record is a particular language-level string, container, or ownership type.

The branch conditions use an unsigned selector. Only 0 and 1 satisfy the first condition; selector 2 uses the second path; all larger values return 0.

### Selectors 0 and 1

The function calls F-8157a180d901 with the numeric argument **72**, then passes its result to F-bb27b9d7a176 together with the numeric constant **2** and the selector.

There is no check on the result of F-8157a180d901 before it is passed onward. A zero result, if returned, is therefore forwarded to F-bb27b9d7a176. The latter dependency’s handling of that value is not established.

If F-bb27b9d7a176 returns null:

- The function returns 0.
- None of the subsequent processing, external-operation, local-record cleanup, or indirect-method calls is made.
- The common-prefix field writes remain performed.
- There is no explicit release of the result previously obtained from F-8157a180d901.

If F-bb27b9d7a176 returns non-null, the following sequence occurs:

1. **Read the primary field again.**  
   The function reads the 32-bit word currently at caller-object offset +0x04. This read occurs after both preceding dependency calls. It is not merely reuse of the original primary-payload argument.

2. **Process the local record.**  
   F-679e81700ed5 receives:
   - The address of the local record’s first word.
   - The non-null object returned by F-bb27b9d7a176.
   - The value just read from caller-object offset +0x04.

   Any return value from this call is not used by this function.

3. **Invoke the external operation.**  
   An external dependency receives:
   - Caller-object base plus 0x14.
   - The address of the local record’s first word.
   - The numeric constant **5**.

   The third argument’s unit and meaning are unknown. In particular, the static evidence does not establish whether 5 denotes bytes, elements, a tag, a mode, or another quantity. This call must not be contracted as a five-byte copy without further evidence.

4. **Clean up or process the local record.**  
   F-282b4cfec88b receives the address of the local record’s first word. Its precise effect is unknown; “cleanup” is an inferred role rather than proof of a particular release operation.

5. **Perform indirect dispatch.**  
   After F-282b4cfec88b returns, the function reads the first address-sized word of the temporary object, obtains a callable target from byte offset **+0x1C** relative to that value, and invokes it with the temporary object as its explicit argument.

6. **Return 1.**

The indirect target is resolved after the earlier calls, not visibly cached before them. Those calls therefore cannot be assumed to leave either the temporary object or its dispatch structure unchanged.

No null check is shown for the dispatch-table address or callable target. Successful completion requires them to be usable when reached.

### Selector 2

The function performs two dependency chains, in order:

1. Call F-8157a180d901 with **72**.
2. Pass that result to F-df2422ba76c2 with additional numeric arguments **2** and **0**.
3. Store the result from F-df2422ba76c2 at caller-object offset +0x0C.
4. Call F-8157a180d901 again with **72**.
5. Pass that second result to F-fb045f7ae136 with additional numeric argument **2**.
6. Store the result from F-fb045f7ae136 at caller-object offset +0x10.
7. Return 1.

Neither allocation-like result nor either final result is checked for null or another failure indication. If these calls return normally with zero results, zero is stored and the function still returns 1.

The write at +0x0C occurs before the second call to F-8157a180d901. A later failure or non-returning dependency therefore does not undo that write.

The local record is initialized but is not passed to a dependency on this path. Neither the external operation nor F-282b4cfec88b nor the indirect method is invoked by this path.

### Other selectors

For any unsigned selector greater than 2, the function returns 0 after the common-prefix work.

It makes no dependency calls and performs no direct writes at +0x0C, +0x10, or the destination beginning at +0x14.

## Side effects

### Direct and dependency-mediated effects

The established direct caller-object writes are:

- +0x04, followed by +0x08, on every reached invocation.
- +0x0C, followed later by +0x10, on the selector-2 path.

The function does not directly restore overwritten values, clear other fields, or roll back partial work.

The external operation may affect memory beginning at +0x14, but its actual writes and extent are undocumented. Other dependencies may also modify memory or global state. The absence of a direct write in this function is not a guarantee that a dependency leaves the corresponding memory unchanged.

No synchronization or atomicity guarantee is visible. Ordered accesses within this function do not establish safe concurrent access.

### Ownership and lifetime

The function visibly obtains results from allocation-like and construction-like dependencies, but their ownership contracts are unknown.

Specifically:

- The numeric argument 72 suggests a storage-size request, but that interpretation is inferred.
- It is unknown whether the construction-like dependencies consume, retain, replace, or release their first arguments.
- The null-result path contains no explicit release call.
- The indirect method may be finalization or release, but its identity and ownership effect are not established.
- Storing results at +0x0C and +0x10 does not by itself prove ownership transfer to the caller’s object.
- Existing values at +0x0C and +0x10 are overwritten without a visible read or explicit release by this function.

The local record’s storage ceases to be valid after the invocation returns. Whether any dependency retains its address is unknown; the function supplies no persistent replacement storage for that address.

### Errors and arithmetic

No explicit handling is shown for invalid addresses, dependency failures other than the single null check, or failures during indirect dispatch.

Offsets are byte displacements. The function does not visibly compute a variable-sized allocation, multiply an element count, or increment the selector. Address validity and overflow behavior are preconditions or platform concerns, not checked behavior.

## Dependencies

| Dependency | Invocation and visible role |
|---|---|
| F-8157a180d901 | Called with 72; once for selectors 0/1, twice for selector 2 |
| F-bb27b9d7a176 | Receives the preceding result, 2, and selector 0 or 1; its result is explicitly null-checked |
| F-679e81700ed5 | Receives the local-record address, temporary object, and reread primary field |
| F-282b4cfec88b | Receives the local-record address after the external operation |
| F-df2422ba76c2 | Receives the first selector-2 allocation-like result, 2, and 0; result stored at +0x0C |
| F-fb045f7ae136 | Receives the second selector-2 allocation-like result and 2; result stored at +0x10 |

Two additional dependency identities are **documented_only**:

- The external operation receiving destination address, local-record address, and 5 has no supplied opaque function ID.
- The indirect method selected through offset +0x1C has no statically supplied opaque function ID.

No purity, global-state preservation, memory ownership, or error behavior is assumed for any dependency.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

1. **Unsupported selector**  
   Given selector 3 and valid writable fields, the primary payload is written at +0x04, then 3 at +0x08. No dependency is called, and the result is 0. The same branch applies to 4,294,967,295; it is not treated as a negative selector.

2. **Null temporary object**  
   Given selector 0 and a null result from F-bb27b9d7a176, the result is 0. The initial field writes have occurred. No external operation, local-record cleanup call, or indirect method follows.

3. **Successful selector-1 path**  
   Given selector 1, a non-null usable temporary object, and normally returning dependencies, the visible dependency order is allocation-like call, object-producing call, local-record processing, external operation, local-record cleanup, then indirect dispatch. The result is 1.

4. **Dependency changes the stored primary field**  
   If an earlier dependency changes caller-object offset +0x04 before F-679e81700ed5 is reached, that dependency receives the value read afterward, not necessarily the original primary payload.

5. **Zero results on selector 2**  
   If both construction-like dependencies return zero normally, the function stores zero at +0x0C and +0x10 and returns 1. It does not convert either result into failure.

6. **Existing selector-2 fields**  
   If +0x0C and +0x10 already contain values, the function overwrites them in sequence. No explicit release of the old values is part of this function’s visible behavior.

## Uncertainties

The following remain **documented_only**:

- Full ABI, pointer representation, object alignment, and complete object size.
- Semantic meanings of the selector, payloads, and stored dependency results.
- Whether the local record is an owning object or merely raw storage.
- Meaning of the external operation’s argument 5 and its memory-access extent.
- Allocation failure behavior, constructor ownership rules, and dependency side effects.
- Whether the indirect method destroys, releases, resets, or otherwise processes its receiver.
- Exception, fault, nonlocal-exit, reentrancy, and concurrency behavior.

None of these unknowns is resolved by assigning conventional meanings to allocation-like calls, field offsets, or dispatch structure.

## Evidence

**Established static evidence:** The supplied function body establishes the common writes, unsigned branch partition, dependency arguments, null check, selector-2 stores, ordering of calls and field accesses, indirect dispatch offset, and literal return values.

**Inferred intent:** Storage acquisition, object construction, local-record cleanup, and finalization are plausible descriptions of the observed call patterns. They are not dependency guarantees.

**Verification status:** No runtime tests have been performed. Acceptance examples express conditional consequences of the supplied static evidence, not observed execution results.
