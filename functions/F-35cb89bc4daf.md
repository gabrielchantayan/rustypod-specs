<!-- Trusted provenance: F-35cb89bc4daf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-35cb89bc4daf

## Purpose

F-35cb89bc4daf selects one of two fixed navigation-related descriptors according to a dependency’s result, updates a byte in an associated object, builds a dispatch value, invokes an indirect operation on the receiving object, and performs a final cleanup call.

The established branch condition is **an exact result of 1**, not general truthiness. Both branches return the 32-bit value 1 after their calls complete normally.

Navigation and volume-limit terminology is inferred from the descriptive labels associated with the two fixed descriptors. The supplied evidence does not establish their complete string contents, the destination actually displayed, or whether the indirect operation performs navigation synchronously.

## Interface

### Inputs

The function receives four arguments:

| Argument | Observable role |
|---|---|
| Receiver | An address used to access an associated-object pointer and an indirect dispatch table. It is also passed as the receiver of the indirect operation. |
| Second argument | A 32-bit value that is not used by the visible body. |
| Third argument | A 32-bit value copied into a local four-byte slot. The address of that slot is passed to F-9b9c1c7483f4. |
| Fourth argument | A 32-bit value copied into a separate local four-byte slot. The address of that slot is passed first to F-9b9c1c7483f4 and later to F-8a5aadac202f. |

The meanings and types of the third and fourth arguments beyond their visible 32-bit representation are unknown. They must not be assumed to be integers, strings, handles, or ownership-bearing references.

### Layout requirements

The supplied representation uses 32-bit words and addresses. Under that layout:

- The receiver contains an associated-object pointer at byte offset **0xB8**, equivalent to word index 46.
- An associated object has a byte accessed at offset **0xB8** from its own base.
- The receiver’s first word supplies an indirect dispatch-table address.
- The dispatch table contains a callable entry at byte offset **0x11C**, equivalent to entry index 71 when entries are four bytes wide.

The two uses of offset 0xB8 refer to different levels of the object graph: one locates a pointer within the receiver; the other locates a byte within the pointed-to object.

These are layout observations, not a portable object-format guarantee. Behavior under a different address width or word layout is not established.

### Output

On normal completion, the function returns the 32-bit value **1**.

No return value from the indirect operation is propagated. The visible function does not encode separate success, failure, or branch outcomes in its return value.

### Preconditions

For normal execution:

- The receiver must permit every required field read.
- Each associated-object pointer used for a byte write must identify writable storage through offset 0xB8.
- The receiver’s dispatch-table pointer and the callable entry at offset 0x11C must be usable when dispatch occurs.
- The dependencies must accept the supplied argument representations.
- Any dependency-specific requirements for the copied third and fourth arguments must be satisfied.

The visible body performs no null checks, bounds checks, object-type checks, or explicit error validation. The behavior of invalid addresses or invalid dependency inputs is not established by this evidence.

## Behavior

### Initial local state

The incoming third and fourth arguments are copied into distinct local four-byte slots before the first dependency call.

These copies matter because later dependencies receive addresses of the local slots rather than addresses of the caller’s argument storage. A dependency can therefore modify a local slot through the supplied address without that fact alone establishing a write to the caller’s original argument storage. Indirect writes through values contained in those slots remain possible but are not described by the visible body.

### Classification and mandatory zero write

The function reads the associated-object pointer from the receiver and passes that pointer to F-2ab73c80407c.

After that dependency returns, the function reads the receiver’s associated-object pointer again and writes the single byte **0** at offset 0xB8 of the object then referenced.

The zero write occurs on both branches. In particular, a result of 1 does not bypass the zero write.

The post-call pointer read is significant. The evidence does not establish that F-2ab73c80407c preserves the receiver’s field, directly or through shared state. Consequently, the object supplied to that dependency need not be the same object that receives the subsequent zero write.

### Exact-one branch

If F-2ab73c80407c returned exactly 1:

1. The function reads the associated-object pointer again and writes the single byte **1** at offset 0xB8 of the referenced object.
2. It reads the associated-object pointer again and calls F-b0b42966b3f9 with that pointer and the additional argument values **1** and **0**.
3. After that call returns normally, it selects the fixed descriptor associated with the apparent locked-volume-limit navigation case.

The visible write sequence in this branch is therefore zero followed by one. Those are two distinct byte stores, not merely a final assignment of one.

The function does not inspect a result from F-b0b42966b3f9 before proceeding. Its name or argument pattern does not establish what state it changes.

### All-other-results branch

If F-2ab73c80407c returns any value other than 1:

- The function performs no explicit one-byte write of 1.
- It does not call F-b0b42966b3f9.
- It selects the other fixed descriptor, associated by its label with ordinary volume-limit navigation.

This branch includes a result of zero, a negative signed result, and a nonzero result such as 2. A generic “nonzero means locked” interpretation would not match the supplied evidence.

### Dispatch-value construction

Both branches converge on a call to F-9b9c1c7483f4. Its arguments, in order, are:

1. The address of the local slot initialized from the fourth argument.
2. The selected fixed descriptor.
3. The address of the local slot initialized from the third argument.

The dependency returns a 32-bit value used as the dispatch argument.

The visible caller does not inspect that value for an error indication. It also does not inspect the possibly modified local slots before performing dispatch.

The selected descriptors are distinct fixed references. Their complete contents, encoding, termination, mutability, and lifetime guarantees are not supplied. Treating them as exact, known text would exceed the evidence.

### Indirect dispatch

After F-9b9c1c7483f4 returns, the function obtains the dispatch table from the receiver and invokes the callable entry at offset 0x11C.

The indirect operation receives:

- The original receiver address.
- The 32-bit value returned by F-9b9c1c7483f4.

The table and entry are accessed after the builder call, not captured before it. Therefore, the selected indirect target is not established to be the same target that would have been found before that dependency ran.

The indirect operation’s result is not used by the visible body.

### Cleanup and return

After the indirect operation returns normally, the function calls F-8a5aadac202f with the address of the local slot originally initialized from the fourth argument.

This is the same local slot whose address was passed as the first argument to F-9b9c1c7483f4. If that dependency changed the slot, cleanup receives the slot in its changed state; the original fourth-argument value is not visibly restored first.

No corresponding cleanup call is visible for the local slot initialized from the third argument.

After cleanup returns normally, the function returns 1.

There is no visible alternate return path, retry, rollback, or explicit error handler. If a dependency does not return normally, the evidence does not establish execution of the remaining calls or the final return.

## Side effects

### Direct memory writes

The function’s explicit writes outside its local argument copies are single-byte stores at offset 0xB8 of an associated object:

- Every normal path writes 0.
- The exact-one path additionally writes 1 before calling F-b0b42966b3f9.

No explicit wider store to that field is shown. The direct stores therefore target only that byte, not the surrounding word. Dependencies may independently alter surrounding memory; their effects are unknown.

If the associated-object pointer remains stable and dependencies do not subsequently alter the byte, its value after the explicit update phase is:

- 1 for an exact result of 1.
- 0 for every other result.

That is not an unconditional guarantee about the byte at function return. Later dependencies and the indirect operation may change memory, and their preservation behavior is not supplied.

### Observable call order

On normal completion, the dependency-call order is:

| Classification result | Call order |
|---|---|
| Exactly 1 | F-2ab73c80407c → F-b0b42966b3f9 → F-9b9c1c7483f4 → indirect receiver operation → F-8a5aadac202f |
| Anything else | F-2ab73c80407c → F-9b9c1c7483f4 → indirect receiver operation → F-8a5aadac202f |

The mandatory zero store occurs after the first call. On the exact-one path, the one store occurs before the second call.

This establishes order within the visible invocation. It does not establish atomicity, synchronization, or visibility guarantees to other threads.

### Ownership and lifetime

The visible body does not directly allocate or deallocate memory.

Passing the fourth-argument local slot to a final dependency suggests a cleanup role, but does not prove that this dependency releases memory, decrements a reference count, resets a handle, or performs any particular ownership action.

Likewise, the returned dispatch value is not established to be owned, borrowed, copied, or consumed. Any such contract requires evidence from the dependencies.

Addresses of local slots are supplied to dependencies. The visible function does not establish whether those dependencies retain the addresses or whether retention is permitted. It provides no extended lifetime mechanism for them.

## Dependencies

| Opaque function ID | Established interaction | Not established |
|---|---|---|
| F-2ab73c80407c | Receives the initially read associated-object pointer. Its result is compared with exactly 1. | Meaning of other results, side effects, pointer preservation, failure behavior. |
| F-b0b42966b3f9 | Called only for an exact-one result, after the explicit one-byte store of 1. Receives an associated-object pointer, 1, and 0. | Purpose, effects, ownership behavior, or whether it preserves the receiver and associated object. |
| F-9b9c1c7483f4 | Receives two local-slot addresses around a selected fixed descriptor. Returns the 32-bit dispatch argument. | Construction semantics, slot mutations, allocation, ownership, failure representation, or global-state preservation. |
| Indirect receiver operation | Selected through the receiver’s current table entry at offset 0x11C; receives the receiver and builder result. | Concrete function identity, navigation behavior, ownership transfer, or return semantics. |
| F-8a5aadac202f | Receives the fourth-argument local-slot address after normal dispatch completion. | Actual cleanup effects, memory release, slot mutation, or failure behavior. |

## Acceptance examples

These are conditional acceptance examples derived from static evidence, not executed tests. Dependency effects must be controlled or observed before drawing stronger conclusions about final state.

### Exact-one classification

Given a stable associated-object pointer and a normal classification result of 1:

- The byte at offset 0xB8 receives 0 and then 1, in that order.
- F-b0b42966b3f9 is called with that object, 1, and 0.
- The locked-case descriptor is supplied to F-9b9c1c7483f4.
- The returned dispatch value is passed unchanged to the indirect operation.
- Cleanup follows normal dispatch completion.
- The function returns 1 if all calls complete normally.

An implementation that writes only 1 would omit an established observable write.

### Nonzero classification other than one

Given a classification result of 2:

- The byte receives 0.
- F-b0b42966b3f9 is not called.
- The ordinary-case descriptor is selected.

Selecting the exact-one branch merely because the result is nonzero would violate the contract.

### Associated-object replacement during classification

Suppose the receiver initially refers to object A, and F-2ab73c80407c changes that field to refer to object B before returning.

The classification dependency receives A. The subsequent zero store targets B, assuming B is valid. Reusing the initially read pointer for the store would not preserve the visible access behavior.

### Mutation of the fourth-argument local slot

Suppose F-9b9c1c7483f4 replaces the contents of the local slot initialized from the fourth argument.

The later call to F-8a5aadac202f receives the address of that same modified slot. Passing a separate untouched copy of the incoming fourth argument would not match the observed behavior.

### Dispatch-table change during construction

Suppose F-9b9c1c7483f4 changes the receiver’s dispatch-table pointer or the relevant entry.

The indirect call uses the table and entry read after construction. Preloading the indirect target before construction could produce a different call and would not preserve this ordering.

## Uncertainties

The following remain **documented_only**:

- Complete contents and formats of both fixed descriptors.
- Whether either descriptor causes an actual screen transition.
- Semantic types and ownership rules for the third and fourth arguments.
- Meaning of the associated byte beyond the explicit zero/one updates.
- Mutation and preservation behavior of every dependency.
- Concrete identity and contract of the indirect receiver operation.
- Failure, exception, nonlocal-exit, and invalid-address behavior.
- Thread-safety, reentrancy, and external memory-observation guarantees.
- Whether the apparent cleanup dependency releases resources.
- Whether the fixed return value represents success to callers.

No numerical range calculation, length computation, or variable-width arithmetic is visible. The relevant width distinctions are the 32-bit argument and result slots, 32-bit layout interpretation, and one-byte associated-state stores. Behavior outside valid layout and address preconditions is not established.

## Evidence

**Established static evidence:** the supplied function body supports the exact-one comparison, unconditional zero store, conditional one store and dependency call, two fixed descriptor choices, local-slot address passing, indirect dispatch offset, cleanup order, and constant return value.

**Inferred intent:** descriptive labels suggest ordinary and locked volume-limit navigation cases. The final dependency’s placement suggests cleanup. Neither inference establishes dependency semantics or ownership.

**Unknowns:** dependency bodies, complete descriptor data, caller contracts, runtime object layouts beyond the visible accesses, and actual execution outcomes were not provided.

No runtime tests have been performed.
