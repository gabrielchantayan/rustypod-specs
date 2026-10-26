<!-- Trusted provenance: F-e39ea2a6fc7d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e39ea2a6fc7d

## Purpose

F-e39ea2a6fc7d performs an indirect operation through a table associated with its first argument. It supplies that operation with a writable local copy of its second argument, a separately retrieved function-pointer value, its fourth argument, and a second occurrence of its first argument.

The first indirect operation produces a signed 32-bit result. If that result is anything other than −1, the function performs a second indirect operation using the first argument and the result. On normal completion, it returns the first operation’s result, regardless of the second operation’s return value.

This is the established structural behavior. The operations’ application-level meanings are unknown. In particular, the evidence does not establish that they allocate, register, release, validate, or commit anything. The value −1 selects the branch that omits the second operation; calling it an application-level failure code would be an inference.

## Interface

The function has four inputs and one result:

| Item | Established representation and use |
|---|---|
| First argument | A pointer to storage whose initial 32-bit integer-sized field supplies the base address used for indirect dispatch. Passed to both operations, and passed twice to the first operation. |
| Second argument | A 32-bit value copied into a writable local cell. The first operation receives the address of that cell, not the original argument’s address. |
| Third argument | A 32-bit input with no explicit use in the supplied body. |
| Fourth argument | A 32-bit value forwarded as the fourth argument of the first indirect operation. |
| Result | A signed 32-bit integer returned by the first indirect operation. |

The supplied representation uses a 32-bit integer-sized field as a dispatch-table address. This supports describing the observed layout in 32-bit terms. It does not independently establish the complete target ABI, including function-pointer width, argument-register assignment, stack alignment, or calling convention.

The offsets used to retrieve indirect targets are **byte offsets** from a table base:

- Offset **0x4c**, or 76 bytes, supplies the first operation’s target and a function-pointer-valued argument.
- Offset **0x2c**, or 44 bytes, supplies the conditional second operation’s target.

These are not offsets of inline functions within the first argument’s storage. The function first reads the table-base field through the first argument, then accesses an entry relative to that base.

For ordinary defined execution, the first argument must support the required reads, the resulting table addresses must support the corresponding entry reads, and the invoked entries must designate callable targets compatible with the actual invocation. No checks for these conditions are visible.

## Behavior

### Local input preparation

Before the first indirect invocation, the function initializes a local 32-bit cell with the second argument’s value.

The cell is distinct from caller-owned storage used to supply the argument. Its address is exposed to the first dependency. The wrapper does not explicitly read the cell after that call, and it does not explicitly copy its final contents back to the caller.

A dependency may therefore receive a mutable value without this wrapper providing a direct output channel for the cell’s final contents. Any other consequence of changing the cell depends on behavior outside the supplied body.

### First indirect operation

The first dependency, identified here as **F-unknown-1**, is dynamically selected through the table entry at byte offset 0x4c.

The operation is invoked with five argument positions, in this order:

1. The original first argument.
2. The address of the local 32-bit cell initialized from the second argument.
3. A function-pointer value independently retrieved through the table entry at byte offset 0x4c.
4. The original fourth argument.
5. The original first argument again.

The two occurrences of the first argument carry the same pointer value. They are not copies of the pointed-to object.

The invoked target and the third argument originate from separate evaluations of the same table-slot expression. The evidence does not establish their relative evaluation order. It also does not establish that those evaluations must observe identical values if the object or table can change between reads. With stable storage, both resolve through the same slot and therefore ordinarily agree; stability is a condition, not an unconditional guarantee.

The dependency’s result is captured as a signed 32-bit integer. Call this the **primary result**.

### Conditional second operation

The function compares the primary result with signed −1.

- If the primary result equals −1, it omits the second operation.
- Otherwise, it invokes the second dependency, identified here as **F-unknown-2**, through the table entry at byte offset 0x2c.

The second operation receives two argument positions:

1. The original first argument.
2. The primary result.

There is no positivity check or zero check. Zero, positive values, and negative values other than −1 all select the second operation.

The second target is resolved after the first operation returns. Its lookup reads the table-base field through the first argument at that later point. Consequently, changes made by the first dependency to that field or to relevant table contents may affect which second target is called. The contract must not substitute a table base cached before the first call.

The second operation’s return value, if any, is discarded.

### Return and exceptional paths

After any required second operation returns normally, the function returns the primary result unchanged.

The visible body provides no fallback target, null handling, error conversion, retry, exception handler, or rollback. If a dependency does not return normally, the normal-return description does not apply. The evidence does not determine the platform-specific outcome of an invalid access, invalid indirect target, exception, or nonlocal control transfer.

## Side effects

The wrapper’s explicit data write is initialization of its local 32-bit cell. It does not explicitly write into the first argument’s object or its dispatch table.

Observable effects can nevertheless arise through:

- Reads of the first argument’s initial table-base field.
- Reads of entries at offsets 0x4c and, conditionally, 0x2c.
- Writes by F-unknown-1 through the supplied local-cell address.
- Any other side effects of either dependency.

Only a partial ordering of these effects is established:

1. The local cell is initialized before F-unknown-1 receives its address.
2. The reads needed to supply F-unknown-1’s target and argument values precede that invocation.
3. F-unknown-1 returns its primary result before the branch is selected.
4. If selected, the second dispatch lookup and invocation occur after the first invocation.
5. A normal wrapper return follows completion of the selected calls.

The order among the separate lookup evaluations used for the first invocation is not established. Argument-position order must not be mistaken for memory-read evaluation order.

No explicit allocation, deallocation, reference-count adjustment, or ownership-transfer operation appears in the wrapper. This does not constrain what its dependencies may do.

The local cell has storage associated with this invocation. The wrapper does not provide an enduring allocation or an explicit ownership transfer for that cell. Retaining its address for later use would require a lifetime arrangement not established here. Similarly, passing the first argument to a dependency does not prove that the dependency preserves it, releases it, or assumes ownership.

## Dependencies

The following opaque labels identify unresolved dependency roles in this contract, not verified identities of particular implementations:

| Opaque ID | Resolution and invocation | Established result handling |
|---|---|---|
| F-unknown-1 | Invoked through offset 0x4c with the five arguments described above. | Its signed 32-bit result controls the conditional call and becomes the wrapper’s normal return value. |
| F-unknown-2 | Invoked through offset 0x2c after F-unknown-1, only when the primary result is not −1. Receives the first argument and primary result. | Its return value is ignored. |

The function-pointer value passed in the third position to F-unknown-1 is data supplied to that dependency. The wrapper does not separately invoke that value merely because it was passed.

Neither dependency’s implementation is provided. Their memory accesses, global-state changes, callbacks, ownership behavior, and termination behavior remain unknown. In particular, the contract does not assume that either dependency preserves globals or the dispatch structure.

## Acceptance examples

These are conditional contract examples, not results of runtime tests. Each assumes valid accesses, compatible targets, and normal returns unless stated otherwise.

1. **Primary result is −1.**  
   F-unknown-1 returns signed −1. No second dispatch is performed by the wrapper. The wrapper returns −1.

2. **Primary result is zero.**  
   F-unknown-1 returns 0. The wrapper resolves the offset-0x2c target and invokes it with the first argument and 0. After that call returns, the wrapper returns 0.

3. **Primary result is another negative value.**  
   F-unknown-1 returns −2. The second operation is still required. Suppressing it for all negative results would violate the observed condition.

4. **The local input copy is changed.**  
   F-unknown-1 changes the local cell from an initial 32-bit pattern to another pattern, then returns 7. The wrapper passes 7 to the second operation and ultimately returns 7. It performs no explicit copy-back of the changed cell.

5. **Dispatch state changes during the first operation.**  
   F-unknown-1 replaces the table-base field with another valid table and returns 3. The second target is obtained through the later table-base read, rather than through a required snapshot of the original table.

6. **The third input differs.**  
   Two invocations differ only in their third input. The wrapper contains no explicit use of that input. This does not establish whole-program equivalence if an unknown dependency observes additional ABI or ambient state.

7. **The second operation returns a different value.**  
   F-unknown-1 returns 12 and F-unknown-2 returns some other value. The wrapper’s normal result remains 12.

## Uncertainties

The following items are **documented_only**:

- The semantic type and required size of the first argument’s object beyond the accessed field.
- Whether the table is formally a virtual-method table or another dispatch structure.
- Concrete identities and complete signatures of both dependencies.
- Exact calling convention, pointer representation, alignment requirements, and function-pointer width.
- Relative evaluation order of the separate first-dispatch lookups.
- Synchronization requirements and behavior under concurrent mutation.
- Application-level meanings of the primary result and the −1 sentinel.
- Dependency ownership rules, global effects, and failure behavior.

An adapter-like purpose is plausible: the function supplies a mutable input copy to one operation and conditionally follows it with another. That interpretation is inferred intent, not an additional behavioral requirement.

## Evidence

The supplied static body establishes the local 32-bit copy, indirect-dispatch offsets, argument positions, signed −1 comparison, conditional second invocation, discarded second return value, and return of the primary result.

No dependency implementations, surrounding type definitions, ABI documentation, or runtime observations were supplied. No runtime tests have been performed. The acceptance examples are deductions under their stated conditions, not empirical verification.
