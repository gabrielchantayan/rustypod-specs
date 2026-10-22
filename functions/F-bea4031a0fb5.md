<!-- Trusted provenance: F-bea4031a0fb5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bea4031a0fb5

## Purpose

F-bea4031a0fb5 conditionally makes two receiver-selected indirect calls, then returns a character read through a pointer captured before the condition.

The condition and return use **separate reads of the shared character-pointer slot**:

- An early read supplies the pointer retained for the final character read.
- A subsequent read supplies the pointer dereferenced to decide whether the calls occur.

The representation does not explicitly reuse the captured pointer for the condition. Conversely, the final return does not reload the shared pointer slot.

The result is a character obtained from memory, without Boolean normalization. Whether callers interpret that character as success, status, data, or something else is unknown.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

### Receiver

The function accepts one receiver pointer.

On the call-taking path, the receiver participates in the following dispatch layout:

| Location | Visible role |
|---|---|
| First 4-byte field of the receiver | Supplies the dispatch-table base. |
| Byte offset 88, hexadecimal 0x58, from that base | Supplies the indirect-call target. |

Each indirect invocation receives three arguments:

1. The original receiver pointer.
2. A 4-byte common-argument value.
3. A payload obtained from a shared slot.

The representation supports a 32-bit receiver-field and dispatch-address model. The complete object layout, callable-entry width, and machine-level calling convention are not independently established.

The function performs no explicit receiver validation. When the condition is false, it does not dereference the receiver or resolve a dispatch target.

### Shared inputs

Four shared-storage roles are visible:

| Role | Accesses and purpose |
|---|---|
| Character-pointer slot | Read once to capture the final-read pointer, then read separately to obtain the condition-read pointer. |
| Common-argument slot | Read before the pointer capture to retain a 4-byte value; read again for the first call when the condition is true. |
| First-payload slot | Read to supply the first call’s third argument. |
| Second-payload slot | Read after the first call returns to supply the second call’s third argument. |

For this contract:

- The **captured pointer** is the value retained for the return.
- The **condition pointer** is the value obtained by the later shared-slot read for the branch test.

These terms distinguish access roles. They do not assert that the pointer values differ, or that a mechanism permitting them to differ is available.

The semantic types of the common argument and payloads are unknown. Their use does not establish that they are strings, handles, counts, or owned allocations. The payload widths are not established by the supplied representation.

### Result

The declared result is a character value read through the captured pointer after the conditional call sequence.

On an 8-bit-character target, that memory read supplies one byte. Character signedness and extension into any wider machine return location are unresolved. The function does not visibly normalize the character to zero or one.

Neither indirect call’s result supplies the function’s return value.

### Preconditions for normal completion

The required accesses must be valid when they occur:

- The common-argument slot must be readable for its initial 4-byte capture, including on the no-call path.
- The character-pointer slot must be readable for both its capture read and its separate condition read.
- The condition pointer must designate readable character storage when the condition is evaluated.
- The captured pointer must designate readable character storage when the final read occurs.
- If the condition character is nonzero, receiver-based dispatch and the argument-slot reads required for each invocation must be valid.
- Each invoked dependency must return normally for execution to continue to the next operation.

The captured pointer is not dereferenced merely by being captured. Its required character access is the final read. The condition pointer has a separate required character access during the branch decision.

No checks enforce these preconditions.

## Behavior

### Initial captures and condition

The represented order begins with a 4-byte read of the common-argument slot. That value is retained for the second indirect call.

Next, the function reads the character-pointer slot and retains the resulting captured pointer for the final return.

The condition then performs another read of the character-pointer slot. It dereferences the resulting condition pointer and compares the character with zero.

Consequently, the following operations are distinct:

- Capturing an address for the eventual return.
- Obtaining an address for the condition.
- Reading the condition character.
- Reading the returned character later.

If the condition character is zero, neither indirect call occurs. Execution proceeds to the final read through the captured pointer.

If the condition character is nonzero, execution proceeds to the two indirect calls. No additional character test appears between them.

Where both pointer-slot reads yield the same address, the condition and return access that address at different points in execution. The representation nevertheless contains two shared-slot reads, not one shared pointer capture used for both purposes.

No atomicity, synchronization, or supported concurrent-mutation model is established by this access structure.

### First indirect call

The first invocation resolves a target through the receiver’s first field and the dispatch entry at byte offset 88.

Its arguments are:

1. The original receiver pointer.
2. A new 4-byte read from the common-argument slot.
3. A value read from the first-payload slot.

The first call’s common argument is therefore not specified to use the value retained before the condition. The initial capture and this argument read are distinct accesses.

Dispatch resolution and argument preparation precede entry into the dependency. The representation does not establish a total order among all reads within that call expression.

The dependency’s result is unused.

### Second indirect call

After the first invocation returns normally, the function resolves receiver-based dispatch again at byte offset 88.

The second invocation receives:

1. The original receiver pointer.
2. The 4-byte common-argument value retained before the pointer capture and condition.
3. A value read from the second-payload slot after the first invocation returns.

The dispatch target is not retained across the calls. A first invocation that changes the receiver’s dispatch field or the selected table entry can affect the second target, provided the resulting access remains valid.

The second payload is likewise not captured before the first invocation. Changes to that slot during the first invocation can affect the second call’s payload.

In contrast, changing the common-argument slot during the first invocation does not replace the retained value used for the second invocation.

There is no intervening condition that suppresses the second invocation because a character was changed to zero. Continued execution depends on normal return and valid subsequent accesses, not on a repeated character test.

The second dependency result is also unused.

### Final character read

After the no-call path, or after both invocations return normally, the function dereferences the captured pointer and returns that character.

This is a character read at the end of the represented execution, not reuse of the condition character.

The shared character-pointer slot is not read again to select the return address. Therefore:

- A change to the character at the captured address can affect the result.
- Replacing the shared pointer-slot value does not redirect the final read.
- The character at the condition address determines whether calls occur, but does not necessarily determine the returned character.
- Invalidating the captured address can invalidate the final read even if the shared slot now contains a valid replacement pointer.

These distinctions do not establish whether different addresses can actually be observed between the initial pointer capture and condition in any particular runtime environment.

### Widths and arithmetic

The retained common argument is a 4-byte value. No arithmetic transformation, narrowing, widening, or Boolean conversion of that value is visible.

Dispatch selection uses a byte displacement of 88 from the base obtained through the receiver. The displacement is not described as an element index and must not be multiplied by a table-entry size.

No counter, length calculation, payload-size computation, or other explicit numeric operation appears. Address-overflow handling and invalid-address behavior are not specified.

## Side effects

The function has no explicit stores to shared slots, receiver fields, dispatch storage, or character storage. Its direct actions consist of reads and, conditionally, two indirect invocations.

The dependencies may produce side effects. The supplied material does not constrain those effects or establish that either invocation preserves:

- The receiver’s dispatch field or dispatch entries.
- The common-argument or payload slots.
- The shared character-pointer slot.
- The contents or lifetime of either relevant character-storage location.
- Other process state.

The retained pointer and retained common value are not automatically refreshed by subsequent changes to their source slots. This retention does not guarantee that pointed-to storage remains alive or unchanged.

No explicit allocation, deallocation, retain, release, or ownership-transfer operation appears. Passing a value to an unresolved dependency does not establish ownership transfer or a promise that the dependency will not release referenced storage.

No explicit error status, recovery path, retry, rollback, or exception handler is visible. If an invocation does not return normally, subsequent operations are not established to occur. Invalid access behavior is outside the established contract; the absence of checks does not define a particular fault or return value.

The represented read order is not a cross-thread visibility or memory-ordering guarantee.

## Dependencies

### F-opaque-0001 — unresolved receiver-selected dependency

F-opaque-0001 identifies the unresolved indirect dependency role, not a known implementation.

Its visible invocation interface has three arguments: the receiver, a 4-byte common argument, and a payload value.

When the condition character is nonzero and execution continues normally, the function invokes this dependency role twice in sequence. Both invocations select byte offset 88, but target selection is repeated, so the actual implementations reached may differ.

The following remain unknown:

- Exact callable signature and calling convention.
- Payload widths and semantic types.
- Dependency return type and error signaling.
- Ownership and lifetime rules.
- Effects on shared storage and receiver state.

The only established treatment of dependency results is that this function does not use them.

## Acceptance examples

These examples are conditional requirements derived from static evidence, not executed tests. Unless stated otherwise, all required accesses are valid and invoked dependencies return normally.

### Zero condition with stable pointer storage

Suppose both pointer-slot reads yield address X, the character at X is zero for the condition, and that character remains zero through the final read.

Expected behavior:

- The initial common-argument capture still occurs.
- The character-pointer slot is read for capture and separately for the condition.
- Neither indirect call occurs.
- The receiver is not dereferenced by this function.
- The final character read through the captured pointer returns zero.

A zero-path implementation that omits the initial common-argument access or conflates the two represented pointer-slot reads would not preserve the described access structure.

### Stable nonzero case

Suppose both pointer-slot reads yield X, the character at X is nonzero and unchanged, and dispatch and argument slots remain stable.

Expected behavior:

- Two indirect invocations occur in order.
- Both receive the original receiver.
- The first receives the first payload; the second receives the second payload.
- Both receive the same common-argument bit pattern because its slot is stable.
- The returned character equals the character at X.

This establishes neither an application-level success meaning nor ownership semantics.

### Distinct pointer observations, if supported externally

Suppose an independently established execution environment permits the capture read to yield X and the later condition-pointer read to yield Y.

Then the required behavior is conditional on those observations:

- The character at Y controls whether calls occur.
- The final character is read at X.
- A zero character at Y does not imply a zero return if X contains a different character.
- A nonzero character at Y can trigger both invocations even if X contains zero.

This example specifies the consequences of distinct observations. It does not establish concurrent mutation, asynchronous updates, or any mechanism that produces them.

### First invocation clears the captured character

Suppose the condition character is nonzero and the first dependency writes zero to the captured address.

The second invocation still occurs if the first returns normally and subsequent accesses remain valid. If the second invocation leaves that character unchanged, the function returns zero.

Thus, a zero return does not establish that no calls occurred.

### First invocation changes shared arguments

Suppose the initial common-argument capture is A, the first invocation receives A, and that invocation changes the common-argument slot to B and the second-payload slot to P.

The second invocation receives A as its common argument and P as its payload. The change to B does not alter the retained value.

### First invocation replaces the shared pointer

Suppose the captured pointer is X. The first dependency replaces the shared character-pointer slot with Y while keeping X readable.

The final read remains at X. The new slot value does not redirect the return to Y. The condition has already been evaluated, so this replacement does not cause a new branch decision.

### First invocation changes dispatch

Suppose the first dependency changes receiver-based dispatch so that byte offset 88 resolves to another valid target.

The second invocation uses the newly resolved target. Unconditionally retaining the first target would not match the represented behavior.

### First invocation does not return normally

If the first dependency fails to return normally, the second invocation and final character return are not established to occur. No local recovery mechanism is visible.

## Uncertainties

The following are **documented_only**:

- Application-level meaning of the result, including any success or status interpretation.
- Semantic meaning of the condition character, common argument, and payloads.
- Character width beyond the declared character type, signedness, and return extension.
- Payload widths and complete receiver and dispatch layouts.
- Exact calling convention and dependency implementation identities.
- Ownership rules and dependency effects on storage lifetime.
- Behavior for invalid pointers, invalid targets, or address overflow.
- Evaluation order among reads within each indirect-call expression.
- Concurrency, atomicity, volatile-access properties, and supported mechanisms for distinct pointer-slot observations.

An inferred intent is that a nonzero character enables a pair of related receiver operations. That interpretation is not required for conformance and does not identify what the operations do.

## Evidence

Established behavior comes from the supplied static representation:

- A 4-byte common value is captured first.
- A character pointer is then captured for the final read.
- The condition separately reads the shared character-pointer slot and dereferences that resulting pointer.
- A nonzero condition character selects two ordered indirect invocations.
- Each invocation independently resolves receiver-based dispatch at byte offset 88.
- The first invocation rereads the common-argument slot; the second uses the retained value.
- Separate shared payload slots supply the invocations.
- The final return dereferences the captured pointer rather than reloading the shared slot or reusing the condition character.
- No explicit stores, ownership operations, or error handling appear.

No runtime tests, dependency implementations, or caller context were supplied. Application meaning and unresolved runtime properties are therefore kept separate from the static observations.
