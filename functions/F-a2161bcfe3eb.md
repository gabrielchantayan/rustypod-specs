<!-- Trusted provenance: F-a2161bcfe3eb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a2161bcfe3eb

## Purpose

F-a2161bcfe3eb conditionally performs one of two state transitions on a caller-supplied object. A selector of zero requests the first transition; a selector of one requests the second. Other selector values produce no transition.

The recovered behavior combines a state marker, indirect object callbacks, external calls, and—in the second transition—a wait loop. The apparent intent is to enter and leave an operating state. That interpretation is inferred; the supplied evidence does not establish what the state represents.

The function returns a 32-bit zero whenever execution reaches its normal return. This is not evidence that the transition succeeded: there is no explicit success test, error return, rollback, or timeout.

## Interface

### Inputs

The function has two inputs:

- **Object reference:** a pointer to storage accessed through a recovered array-of-integers representation.
- **Selector:** an integer compared exactly with zero and one.

The following layout uses the recovered 32-bit integer and pointer model. These are the relevant accesses, not a complete object definition.

| Object-relative offset | Recovered interpretation | Use |
|---|---|---|
| `0x00` | Address of the primary dispatch table | Resolves the primary-object callback |
| `0x14` | 32-bit integer field | Repeatedly compared with a captured target |
| `0x28` | 32-bit address of a secondary object | Supplies the receiver and dispatch-table lookup for a secondary-object callback |
| `0x54` | Integer-sized slot whose value is narrowed to a character | Supplies the state comparison |
| `0x54`, first addressed byte | One-byte writable state marker | Receives one or zero during a transition |

The same numeric offset, `0x54`, also appears as an offset **within the primary dispatch table**. That dispatch-table entry is separate from the object’s state storage.

The secondary object is interpreted as having a dispatch-table address at its own offset zero. Its callback is selected from dispatch-table offset `0x0c`.

### Output

On normal completion, the function returns a 32-bit value of zero.

It returns no object, pointer, or separately allocated result. No direct allocation or release operation is visible in the supplied body.

### Preconditions

For any path that accesses the object, the caller must supply storage supporting the reads and writes made on that path. For a transition path, the applicable dispatch tables, callback entries, and callback receiver objects must also be valid when used.

No null checks, bounds checks, callback-address checks, or object-lifetime checks are visible.

For selectors other than zero and one, the body does not dereference the supplied object reference. Consequently, object validity is not required by the visible body on that path, although no claim is made about checks outside this function.

The recovered representation uses integer slots as addresses. Its exact ABI, alignment requirements, and endianness are not independently established by the supplied evidence.

## Behavior

### Selection and state comparison

The selector is examined before the state slot.

- If the selector is zero, the first transition is taken only when the state-slot value, narrowed to a character, is not one.
- If the selector is one, the second transition is taken only when that narrowed value is not zero.
- Any other selector skips both transitions.

The state is therefore not treated as a validated two-valued enumeration. A narrowed state value other than zero or one qualifies for either transition, depending on the selector.

The recovered read and write representations differ:

- The condition reads an integer-sized slot and narrows its value to a character.
- A transition writes one byte at the beginning of that slot.

Only the one-byte write is explicit; the function does not explicitly replace the entire integer-sized slot.

Under the conventional 32-bit, eight-bit-byte interpretation, the comparison concerns the integer’s low eight value bits. Whether the byte written at the lowest address is that same low-order byte depends on endianness. The supplied body alone does not establish that relationship. Character signedness does not change equality with zero or one for the relevant bit patterns.

### First transition: selector zero, state comparison not equal to one

When the first transition is selected, the observable sequence is:

1. Call F-d0ab8e462b50 with the integer argument one.
2. Call dependency D-01 with integer arguments one and two, in that order.
3. Call dependency D-02 with no explicit arguments.
4. Write the byte value one at object offset `0x54`.
5. Resolve the secondary-object callback through the object field at `0x28`, the secondary object’s dispatch-table address, and dispatch-table offset `0x0c`.
6. Invoke that callback with the secondary object as receiver and integer argument one.
7. Return 32-bit zero if control returns normally.

The ordering of the first three calls and the state write is significant. The function does not publish its explicit one-byte state update until those calls have returned. It publishes that update before invoking the secondary-object callback.

The secondary-object field is used after the earlier calls. No assumption may be made that its value remains the same as it was at function entry. The visible expression contains receiver and dispatch-resolution accesses, but does not establish a portable order among all accesses within that single call expression.

If the initial state comparison equals one, none of these calls or writes occurs.

### Second transition: selector one, state comparison not equal to zero

When the second transition is selected, the observable sequence is:

1. Resolve and invoke the secondary-object callback, using dispatch-table offset `0x0c`, with the secondary object as receiver and integer argument two.
2. Write the byte value zero at object offset `0x54`.
3. Resolve the primary-object callback through the object’s dispatch-table address and dispatch-table offset `0x54`.
4. Invoke that callback with the original object as receiver.
5. Read a shared 32-bit target value from a fixed external storage location and retain that value for this invocation’s wait loop.
6. Compare the object’s 32-bit field at offset `0x14` with the retained target.
7. While the values differ, call D-03 with integer argument ten, then compare the object field again.
8. Once the values are equal, call D-01 with integer arguments zero and two, in that order.
9. Call D-02 with no explicit arguments.
10. Call F-d0ab8e462b50 with integer argument zero.
11. Return 32-bit zero if control returns normally.

The secondary callback precedes the explicit state clear. The primary callback follows that clear. The function does not restore or reassert the state afterward.

The target is captured **after** the primary callback returns. A target update made by that callback can therefore affect the captured value. A later change to the target’s original storage does not alter the retained comparison value in the recovered behavior.

The object field at `0x14` is evaluated again after each D-03 call. If it equals the captured target on the first comparison, D-03 is never called. Otherwise, D-03 can be called indefinitely.

No timeout, iteration limit, cancellation check, or alternate exit is visible.

If the initial state comparison equals zero, none of the transition’s calls, writes, target reads, or wait-loop field reads occurs.

### Return values and failures

Return values from all dependencies and callbacks are unused.

The visible body has no explicit failure result. Its zero return establishes only that control reached the final return statement. It does not establish a postcondition for shared state, because callees may modify that state.

Invalid memory access, invalid callback dispatch, a dependency that does not return, or a wait condition that never becomes equal can prevent normal completion. The function provides no visible recovery for these cases.

## Side effects

### Direct writes

The only direct object write visible in the body is a one-byte store at offset `0x54`:

- One during the first transition.
- Zero during the second transition.

No direct write to the field at `0x14`, the secondary-object address at `0x28`, or either dispatch table is shown.

The state write is an intermediate action, not an unconditional final-state guarantee. In the first transition, the secondary callback runs after the write. In the second transition, the primary callback and subsequent dependencies run after the write. Their effects are unknown.

### External effects

The function invokes dependencies and callbacks in the orders described above. Their effects may include changes to the supplied object, other objects, shared storage, or external resources. None of those effects is established by the call sites alone.

In particular:

- Earlier dependencies may change the secondary-object field before it is used.
- The secondary callback may change object state before later accesses.
- The primary callback may change the shared target before it is captured.
- D-03 may affect the repeatedly read object field or other state.

No callee is assumed to preserve globals, object fields, dispatch tables, or memory ownership.

### Ownership and lifetime

The function visibly receives an existing object reference and passes existing object references to callbacks. It does not visibly allocate, copy, retain, or release an object.

This establishes no ownership policy for the dependencies. Safe execution requires relevant objects and storage to remain valid across all calls and subsequent accesses. The function contains no visible lifetime protection.

### Synchronization

No lock, explicit atomic operation, or memory barrier is shown. The loop’s repeated field checks do not by themselves establish a cross-thread synchronization contract.

The argument ten passed to D-03 does not establish a duration or unit. Treating that call as a sleep, yield, or scheduler operation would require additional evidence.

## Dependencies

The following D-prefixed identifiers are document-local opaque labels for dependencies without supplied opaque function IDs. They distinguish call sites without asserting implementation identity or semantics.

| Identifier | Visible call contract | What remains unknown |
|---|---|---|
| F-d0ab8e462b50 | Called with one before the first transition’s other external calls; called with zero after the second transition’s final external calls | Meaning of the argument, side effects, failure behavior, ownership |
| D-01 | Called with `(one, two)` during the first transition and `(zero, two)` after the second transition’s wait completes | Argument meanings, shared-state effects, resource behavior |
| D-02 | Called without explicit arguments immediately after D-01 in either transition | Purpose, effects, completion guarantees |
| D-03 | Called with ten once per unsuccessful wait-loop comparison | Units, scheduling behavior, state changes, return guarantees |
| D-04 | Secondary-object indirect callback at dispatch-table offset `0x0c`; receives its object and either one or two | Concrete implementation, argument meanings, mutation and lifetime effects |
| D-05 | Primary-object indirect callback at dispatch-table offset `0x54`; receives the original object | Concrete implementation, effects on the wait field or target, ownership |

D-04 is resolved when its call is reached; the function does not visibly cache its entry at function entry. D-05 is resolved after D-04 and after the second transition’s byte clear.

The shared target storage is an additional data dependency. Its initialization, producer, semantic meaning, and update protocol are not supplied.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

1. **Unsupported selector**  
   Given a selector other than zero or one, the function performs no visible object access, dependency call, callback, or state write, and returns 32-bit zero.

2. **First transition already marked**  
   Given selector zero and a state-slot value whose character conversion equals one, the function makes no dependency calls and performs no state write.

3. **First transition required**  
   Given selector zero and a state comparison unequal to one, normal execution orders F-d0ab8e462b50 with one, D-01 with one and two, D-02, the byte write of one, and D-04 with one. A dependency result does not select an alternate branch.

4. **Second transition already clear**  
   Given selector one and a state comparison equal to zero, the function does not call D-04 or D-05 and does not read the wait target.

5. **Second transition needs no waiting**  
   Given selector one, a nonzero state comparison, and equality between the object field and the target captured after D-05, D-03 is not called. The trailing order is D-01 with zero and two, D-02, then F-d0ab8e462b50 with zero.

6. **One unsuccessful wait comparison**  
   Given the same entry conditions, suppose the first field comparison differs, D-03 returns, and the next comparison equals the captured target. D-03 is called exactly once with ten before the trailing calls occur.

7. **Target changes during waiting**  
   Suppose the captured target is a particular value and its original storage later changes. Completion still depends on equality with the captured value, not the replacement value.

8. **Noncanonical state**  
   A state comparison equal to neither zero nor one is accepted into the first transition for selector zero and into the second transition for selector one. There is no visible invalid-state rejection.

9. **Wait never completes**  
   If every field comparison differs and D-03 always returns, execution remains in the loop. The trailing calls and final return are not reached.

10. **Callee changes state after the explicit write**  
    If a later callback changes the state storage, the function does not visibly repair it. Acceptance must not require an unconditional final marker of one or zero without additional callee guarantees.

## Uncertainties

The following are **documented_only**:

- Exact target ABI, endianness, alignment rules, and character representation.
- Whether the recovered integer-sized state read corresponds to an actual full-width memory read in the executable.
- The semantic meanings of the selector, marker, callback arguments, and target field.
- Dependency implementations, return conventions, failure mechanisms, and side effects.
- Ownership or release behavior inside dependencies.
- The mechanism that makes the wait field equal the captured target.
- Threading, interrupt, reentrancy, and synchronization requirements.
- Whether the target and object field are intended to be updated asynchronously.
- The ordering of subexpression accesses within an individual indirect-call expression.
- Any behavior of wrappers or callers outside the supplied body.

“Enter state,” “leave state,” and “wait for completion” are useful inferred descriptions, not established domain semantics.

## Evidence

The established static evidence is the supplied recovered body for F-a2161bcfe3eb and the supplied mapping to F-d0ab8e462b50.

That evidence supports the selector branches, state comparisons, one-byte writes, indirect dispatch offsets, call ordering, post-callback target capture, repeated wait-field comparison, and constant 32-bit normal return described above.

It does not establish dependency internals, a complete object type, a concurrency protocol, or runtime correctness. No runtime tests have been performed.
