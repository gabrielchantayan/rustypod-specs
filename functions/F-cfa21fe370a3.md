<!-- Trusted provenance: F-cfa21fe370a3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cfa21fe370a3

## Purpose

F-cfa21fe370a3 performs an ordered state transition on an object and, conditionally, on another object obtained through virtual dispatch. It also traverses an associated collection and writes a zero byte through a pointer associated with each yielded entry.

The established behavior is:

- An object whose captured state byte is already 2 receives no further processing.
- Otherwise, a virtual dependency determines whether recursive processing is attempted.
- Any recursive processing completes before collection processing begins.
- Collection processing, when enabled, clears one byte through each yielded entry’s pointer.
- The captured state byte is then set to 2.
- A final virtual dependency is invoked with the original object and the unchanged second argument.

Interpreting state 2 as “processed,” the recursive relation as a parent relation, or the cleared bytes as flags is **inferred intent**, not an established semantic meaning. This contract uses structural descriptions instead.

## Interface

### Inputs

The function accepts two machine-width arguments:

1. **Object address.** The supplied representation uses 32-bit address-bearing fields. The object must support:
   - An address-bearing field at byte offset 0, used as the base of a virtual dispatch table.
   - An address-bearing field at byte offset 12, pointing to a state record.
2. **Opaque argument.** A 32-bit value forwarded unchanged to every recursive invocation and to the final virtual dependency. This function does not itself dereference or interpret it.

No null checks or object-validity checks are established.

### Object and associated layouts

The following offsets and access widths are supported by the supplied static evidence:

| Structure | Offset | Width | Use |
|---|---:|---:|---|
| Object | 0 | 32 bits | Virtual dispatch table address |
| Object | 12 | 32 bits | State-record address |
| State record | 0 | 8 bits | Early-return test and eventual write of 2 |
| State record | 8 | 32 bits | Collection handle or address |
| Virtual dispatch table | 28 | Address-sized, represented as 32 bits | Recursive-target query |
| Virtual dispatch table | 76 | Address-sized, represented as 32 bits | Final operation |
| Yielded entry | 12 | 32 bits | Address of the byte to clear |

These offsets establish required accesses, not complete structure definitions. Unmentioned bytes and fields have no specified meaning here.

The state-record address is captured once, before the initial state-byte test. Later operations use that captured address even if a dependency changes the object’s field at offset 12.

### Output

There is no established return value or explicit error result. The observable results are memory writes, recursive effects, and dependency invocations.

Normal return requires that all invoked dependencies return normally, all required accesses be valid, and any recursion and iteration terminate.

### Preconditions

For a defined successful execution:

- The object and its state-record field must be readable.
- The captured state record must permit the initial byte read.
- If that byte is not 2, the required virtual dispatch structures and targets must be valid when each dispatch occurs.
- The captured state record must remain usable across intervening dependency calls and recursion.
- If collection processing occurs, each successful iterator step must provide an entry with a readable pointer field at offset 12, and that pointer must designate a writable byte.
- The captured state byte must be writable when the final transition occurs.

The function does not enforce these requirements. No validation failure behavior is established.

## Behavior

### Initial state observation

The function first reads the state-record address from object offset 12, then reads the byte at the beginning of that record.

If the byte equals 2, the function returns immediately. On this path it does not:

- Invoke either virtual dependency.
- Read the collection field at state-record offset 8.
- Create or advance an iterator.
- Perform any explicit memory write.

Any other byte value takes the processing path. There is no additional state-value classification.

### Recursive-target queries

For the processing path, the function invokes the virtual dependency selected by dispatch-table byte offset 28, passing the original object as its argument.

This first result is tested only for zero versus nonzero:

- A zero result suppresses recursion.
- A nonzero result causes a second invocation through dispatch-table byte offset 28.

The first result is **not** reused as the recursive target. The result of the second invocation is passed as the object argument to a recursive invocation of F-cfa21fe370a3. The original opaque argument is forwarded unchanged.

The second result is not checked for zero or validity before recursion. Consequently, a nonzero first result followed by a zero second result does not cancel recursion; it attempts recursive processing with a zero object address. No defined successful outcome is established for that case.

Each virtual invocation obtains its target from the object’s dispatch structures at that point in execution. The evidence does not establish that the two query calls use the same dispatch-table contents or return the same object. Side effects of the first query can therefore matter to the second query.

When recursion is taken, it must return before the current invocation proceeds to its collection field. Recursive calls perform the same initial state check and may themselves recurse.

The current invocation does not set its captured state byte to 2 before recursion. Thus, the state byte does not provide an in-progress cycle guard. A cyclic relation involving objects whose state bytes are not already 2 may recurse indefinitely or exhaust execution resources. A cycle is not necessarily taken: query results and their side effects remain relevant.

### Collection processing

After the optional recursive invocation returns, the function reads the 32-bit field at offset 8 in the originally captured state record.

If this field is zero, it skips iterator initialization, iteration, and iterator finalization.

If this field is nonzero, it reads that field again and invokes F-538044b67754 with:

- The address of a 20-byte invocation-local workspace.
- The value from the second collection-field read.
- The exact 32-bit bit pattern `0xFFFFFFFE`.

The third argument represents −2 under signed 32-bit interpretation and 4,294,967,294 under unsigned interpretation. Its semantic role is unknown. Neither interpretation justifies assigning it an enumeration policy or range meaning.

The separate collection-field reads are observable ordering facts. The nonzero test applies to the first read; the initializer receives the second read. The function does not explicitly snapshot and reuse the tested value. If the field changes between those reads, the initializer can receive a different value, including zero. No synchronization or concurrent-mutation contract is established.

After initialization returns, the function repeatedly invokes F-6e4651b532dc with the workspace address and the address of a 32-bit output slot.

For each nonzero return:

1. The output slot is treated as an entry address.
2. The function reads a 32-bit address from entry offset 12.
3. It writes exactly one zero byte at that address.

The write is through the stored pointer. It does not replace the pointer field with zero and does not clear an entire word or entry.

A zero iterator result ends the loop. On that result, the function performs no entry dereference and no byte clear for the output slot.

After loop termination, the function calls F-944b595e25e9 with the workspace address. There is no explicit status check around initialization or finalization, and there is no separately established error branch for an iterator result. Every nonzero iterator result follows the entry-processing path, regardless of sign.

The entry output slot is not visibly initialized by this function before use by the iterator dependency. A successful iterator result must therefore make the slot usable; preservation of an earlier value is not a valid assumption.

### State transition and final dispatch

After collection processing finishes—or immediately after the collection-field check when it is zero—the function writes the byte value 2 to the beginning of the captured state record.

It then invokes the virtual dependency selected by dispatch-table byte offset 76, passing:

- The original object address.
- The unchanged opaque argument.

This dispatch occurs after the state-byte write. Its dispatch target is obtained from the object at that time, so earlier dependency effects can influence the target selected.

The function does not restore the prior state byte or inspect a result from this final operation. If the final dependency returns normally, this invocation returns.

The explicit write of 2 does not establish that the byte remains 2 on return: the final dependency may modify it. Likewise, the function does not recheck the byte after recursion or iterator callbacks. An intervening write of 2 does not itself suppress the remainder of an already active invocation.

## Side effects

### Direct writes

The function directly performs only these established classes of writes:

- One zero-byte write for each nonzero iterator result.
- One byte write of 2 to the captured state record on the processing path.

The final state write occurs after all successful iterator-step writes and after iterator finalization.

No non-aliasing guarantee is established. Multiple entries may point to the same byte. A cleared byte may overlap the state record or other active data. Such writes occur in the established order; this contract does not silently exclude aliasing or supply safety guarantees for it.

### Indirect effects

Virtual dependencies, iterator dependencies, and recursive invocations may have additional effects. Their global-memory behavior, allocation behavior, callbacks, and mutation guarantees are unknown.

The function uses invocation-local iterator storage and a local entry-output slot. It exposes their addresses to dependencies during collection processing. No permission to retain those addresses beyond the invocation is established.

There is no explicit allocation, release, object destruction, or ownership transfer in the visible behavior. The role of F-944b595e25e9 does not, by itself, prove that it releases collection storage or owns yielded entries.

### Failure and partial completion

No rollback or structured recovery path is established.

If processing stops abnormally:

- Recursive effects may already have occurred.
- Some entry bytes may already have been cleared.
- Iterator finalization is not guaranteed.
- The state write may or may not have occurred, depending on the stopping point.

If the final virtual operation fails after entry, the explicit state write has already occurred. This ordering is not a success-commit guarantee.

## Dependencies

| Dependency | Established use | Unspecified behavior |
|---|---|---|
| F-cfa21fe370a3 | Recursive invocation using the second query result and unchanged opaque argument | Termination and effects depend on the reached objects and dependencies |
| F-538044b67754 | Receives workspace, collection value, and `0xFFFFFFFE` | Initialization semantics, ownership, error signaling, and mutation |
| F-6e4651b532dc | Receives workspace and an entry-output address; nonzero results trigger processing | Ordering, filtering, error distinction, and collection-mutation tolerance |
| F-944b595e25e9 | Invoked once after normal iterator exhaustion | Cleanup obligations, resource release, and other effects |

Two additional dependencies are selected dynamically:

- **Dispatch slot 28:** receives the object; its first result controls whether a second call occurs, and its second result supplies the recursive object.
- **Dispatch slot 76:** receives the object and opaque argument after the state write.

Their function identities are not established. These slot descriptions are opaque dependency selectors, not assertions about domain-level roles.

## Acceptance examples

These examples are conditional consequences of the static evidence, not executed tests.

### Already-marked object

Given a valid object whose captured state byte is 2:

- The invocation returns without querying either virtual slot.
- It does not examine the collection field.
- It performs no explicit writes.

### No recursion and no collection

Given an initial state byte other than 2, a zero result from the first slot-28 query, and a zero collection field:

- Slot 28 is called exactly once.
- No recursive invocation or iterator dependency is invoked.
- The captured state byte is written to 2.
- Slot 76 is then invoked with the object and unchanged opaque argument.

### Distinct query results

Given a nonzero first slot-28 result and a different, valid second result:

- Slot 28 is called twice.
- Recursion targets the second result, not the first.
- Current-object collection processing starts only after that recursive invocation returns.

If the second result is zero, successful processing is not established; there is no second-result guard.

### Empty iteration

Given a nonzero collection field, normal initialization, and a zero result from the first iterator step:

- No entry pointer is dereferenced.
- No entry byte is cleared.
- F-944b595e25e9 is invoked.
- The state write and final dispatch follow.

### Multiple yielded entries

Given two nonzero iterator results followed by zero, with valid entry pointers:

- The first pointed-to byte is cleared before the second pointed-to byte.
- Exactly one byte is written per successful step.
- Finalization precedes the state write.
- The state write precedes slot-76 dispatch.

If both entries point to the same byte, that byte receives two zero writes.

### State-record replacement during a callback

Given that a dependency changes the object’s state-record field while leaving the original captured record valid:

- This invocation continues to use the originally captured record for its collection access and state write.
- The final virtual dispatch still uses the original object and its then-current dispatch structures.

### Final callback mutation

Given that slot 76 changes the state byte after observing the write of 2:

- The function performs no subsequent corrective write.
- The post-return byte value depends on the callback’s effects.

## Uncertainties

The following are **documented_only**:

- Domain meanings of the state byte, collection, recursive relation, and opaque argument.
- Complete object layouts, alignment requirements, and calling-convention details.
- The interpretation of `0xFFFFFFFE` by F-538044b67754.
- Whether iterator termination can also represent an error.
- Whether iterator dependencies allocate, release, retain, or mutate resources.
- Whether virtual dependencies mutate dispatch structures, globals, or state records.
- Whether aliasing or concurrent mutation is supported by the surrounding system.
- Exception, fault, and abnormal-exit semantics.
- Any stronger ownership, lifetime, idempotence, or termination guarantee.

Repeated invocation is a no-op only when the state byte observed at entry is 2. Broader idempotence is not established.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping.

Established evidence includes the offsets and access widths listed above, the two separate recursive-target queries, the captured state-record address, the iterator call sequence, the byte-sized writes, and the state-write-before-final-dispatch ordering.

The final transfer is represented as an indirect call with unresolved control-flow recovery. Its visible target selection and arguments support the specified dispatch; its underlying transfer form and any stronger return guarantee remain uncertain.

Processing terminology and iterator-role descriptions summarize observed use. They do not establish domain semantics or hidden dependency contracts.

No runtime tests, implementation execution, or independent dependency inspection have been performed.
