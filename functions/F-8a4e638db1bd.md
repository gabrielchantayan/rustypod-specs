<!-- Trusted provenance: F-8a4e638db1bd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8a4e638db1bd

## Purpose

F-8a4e638db1bd advances a context through an indexed collection, invokes two operations on an associated object for each processed index, and conditionally emits a notification. When the current index has reached or exceeded a captured collection bound, it instead obtains a mode-dependent adjustment, adds that adjustment to a context field, and returns a distinct result.

The caller selects between:

- Processing at most one index.
- Repeating until the completion branch is reached, subject to changes made by dependencies.

**Established static evidence:** the function coordinates indexed callbacks, updates two context fields, and returns either 0 or 1 on normal return.

**Inferred intent:** the index appears to represent progress, the second context field appears to accumulate a result, and the optional three-argument call appears to report progress. These interpretations do not establish application-level meanings, units, or success semantics.

## Interface

The function accepts four 32-bit arguments and returns a 32-bit value.

| Argument | Behavioral role |
|---|---|
| Context address | Address of caller-supplied state containing the current index and an accumulator-like field. |
| Mode | A 32-bit value passed unchanged to dependencies. The value 3 suppresses one notification sequence. |
| Iteration control | Zero requests continued iteration. Any nonzero value requests return after one processing iteration, unless the completion branch is taken first. |
| Initial selection address | Initial 32-bit value of a writable local selection slot. An indexed callback receives the address of that slot and may replace its contents. |

The supplied representation establishes a 32-bit address model. Pointer values and the relevant integer fields occupy four bytes. Byte order is not established.

### Context layout

Offsets are relative to the context address.

| Byte offset | Width | Use |
|---|---:|---|
| 284 (`0x11c`) | 32 bits | Current index. Read as unsigned for the completion comparison; used in signed-looking arithmetic elsewhere. |
| 288 (`0x120`) | 32 bits | Accumulator-like field. Its address is passed to an object callback; the completion branch adds a dependency result to its current value. |

No requirements for intervening context bytes are established.

### Collection/provider layout

At entry, a global reference supplies a provider object address. The function captures that address and reads a 32-bit bound from byte offset 4 of that object.

The provider’s first word supplies a dispatch-table address. The dispatch entry at byte offset 60 (`0x3c`) is invoked with:

1. The captured provider address.
2. The current context index.
3. The address of the local selection slot.

The bound is captured once. It is not refreshed between iterations.

### Selected-object layout

After the provider callback returns, the local selection slot is treated as an address. A 32-bit object address is read at byte offset 152 (`0x98`) from that address.

That object’s first word supplies a dispatch-table address. Two dispatch entries are used:

- Byte offset 44 (`0x2c`): receives the object address.
- Byte offset 28 (`0x1c`): receives the same captured object address, the mode, and the address of the context field at offset 288.

The dispatch-table pointer is looked up separately for the second invocation. A change to that pointer during the first invocation can therefore affect the second target.

### Preconditions

The function performs no explicit null, bounds, lifetime, or dispatch-target validation. For defined operation:

- The global reference and captured provider must support the entry reads.
- The context fields must be accessible for every read and write that execution reaches.
- Each reached dispatch-table entry must identify a callable target with a compatible calling convention.
- After each provider callback, the selection slot must identify storage from which the object address at offset 152 can be read.
- That object and its dispatch data must remain usable for both object invocations.
- Dependencies must return normally if the caller expects a normal return.

These are requirements imposed by the accesses, not checks performed by the function.

## Behavior

### Entry capture

The function obtains the provider address from the global reference and captures the provider’s field at offset 4 as an unsigned 32-bit bound. It initializes its local selection slot from the fourth argument.

Subsequent iterations use the captured provider address and captured bound. Changes to the global reference or the provider’s bound field do not, by themselves, replace those captured values.

This does not mean the provider is immutable. Its dispatch-table pointer is read when the provider callback is invoked.

### Completion decision

At the start of each iteration, the function reads the context index and compares it with the captured bound as unsigned 32-bit values.

If the index is greater than or equal to the bound:

1. F-3ecf06e452fb is called with the mode.
2. After that call returns, the context field at offset 288 is read.
3. The returned 32-bit result is added to that field.
4. The sum is written back to offset 288.
5. The function returns 1.

The accumulator read occurs after F-3ecf06e452fb returns. A change to that field made during the dependency call is therefore part of the subsequent addition.

This branch does not itself invoke the provider callback, invoke either selected-object callback, issue the optional notification sequence, or increment the index.

An index with its high bit set is still compared as unsigned. It is not rejected as a negative value.

### Processing an index

If the index is below the captured bound, the function invokes the provider’s dispatch entry at offset 60.

The callback receives a newly read current index, not a separately retained index value from the comparison. With no intervening external mutation, the two reads agree; the function does not explicitly synchronize them.

The provider callback can replace the local selection slot. After it returns, the function:

1. Reads the object address at offset 152 from the address currently held in the selection slot.
2. Invokes that object’s dispatch entry at offset 44.
3. Looks up that same object’s dispatch entry at offset 28 and invokes it with the mode and the context accumulator address.

The object address used for both object calls is the one captured immediately after the provider callback. The function does not reread the selection slot or its offset-152 field between those calls.

The first object callback’s return value is unused. The second callback’s return value is also unused. The second callback can directly modify the context accumulator through the supplied address.

The local selection slot is initialized only once, before iteration starts. If execution repeats, the next provider callback receives the same slot containing the value left by prior activity. The function does not reset it to the original fourth argument.

### Conditional notification

If the mode is not 3, the function performs this sequence after both object callbacks:

1. It reads the current context index and forms the arithmetic value “index plus one, multiplied by 500.”
2. It calls F-d7be9cfb4000 with that value and the captured bound.
3. It calls F-93fd250aa9bf.
4. It calls F-b63ebee7ff72 with the result from F-93fd250aa9bf, the constant 6, and the saved result from F-d7be9cfb4000.

The arithmetic uses the index present after the object callbacks, not a saved copy of the index originally sent to the provider.

F-d7be9cfb4000 completes before F-93fd250aa9bf is called. Both results are available before F-b63ebee7ff72 is called. No further semantic meaning for their values is established.

When the mode is exactly 3, all three named calls in this notification sequence are skipped. The provider and object callbacks still execute.

### Index update and repetition

After the optional notification sequence, the function reads the context index again, adds one, and writes the result back.

This is an increment of the value present at that point. It is not necessarily an increment of the value observed at iteration entry. Earlier dependencies can affect the value that is incremented.

The iteration-control argument is then applied:

- If nonzero, the function returns 0.
- If zero, execution returns to the completion comparison.

Consequently, processing the last in-range index in single-iteration mode returns 0, even if the increment makes the index equal to the bound. The completion adjustment and return value 1 require entering the completion branch, either on a subsequent call or a subsequent iteration.

### Arithmetic limits

The comparison is explicitly unsigned and 32-bit. Other index and accumulator operations are presented as 32-bit signed integer arithmetic.

For operands whose sums and products fit the signed 32-bit range, the arithmetic results are established directly. The supplied representation does not establish a portable overflow rule for:

- Adding the completion adjustment to the accumulator.
- Adding one to the current index.
- Multiplying the notification numerator by 500.

Modulo wraparound, trapping, or saturation must not be silently added to the contract. Overflow behavior is `documented_only` pending stronger instruction-level or execution evidence.

## Side effects

Direct writes established for this function are:

- A completion-branch write to the context accumulator.
- A processing-branch write to the context index.
- Initialization of the local selection slot.

Dependencies receive writable addresses to the selection slot and, for the second object callback, the context accumulator. They may also affect other reachable state. Their effects are not limited by the visible argument lists.

The supplied body contains no direct allocation, release, ownership transfer, synchronization operation, or error reporting. This does not establish that its dependencies perform none.

No ownership transfer is established for the provider, selection address, selected object, context, or values returned by dependencies. In particular, it is unknown whether a dependency retains the address of the local selection slot or the accumulator. Retaining the local slot for use after this function returns would exceed that slot’s lifetime.

There is no explicit rollback. Writes and dependency effects occur in the stated sequence and are not reversed by this function.

## Dependencies

| Opaque ID or dispatch slot | Established use | Unknown contract |
|---|---|---|
| F-3ecf06e452fb | Receives the mode; its 32-bit result is added to the accumulator on completion. | Meaning, result range, side effects, failure behavior, and ownership. |
| F-d7be9cfb4000 | Receives the computed numerator and captured bound; its result is saved for the notification call. | Whether it divides, scales, clamps, or interprets either argument in another way. |
| F-93fd250aa9bf | Called without explicit arguments after F-d7be9cfb4000. | Meaning and ownership of its result; side effects. |
| F-b63ebee7ff72 | Receives the previous call’s result, constant 6, and the saved result from F-d7be9cfb4000. | Notification semantics, result interpretation, and effects. |
| Provider dispatch offset 60 | Receives provider, index, and writable selection-slot address. | Selection rules, error behavior, and mutations beyond that slot. |
| Object dispatch offset 44 | Receives the selected object. | Preparation semantics and state changes. |
| Object dispatch offset 28 | Receives selected object, mode, and accumulator address. | Accumulator transformation and other effects. |

The indirect targets have no resolved opaque IDs in the supplied dependency map. Dispatch offsets identify their distinct roles without inventing identities.

## Acceptance examples

These examples are conditional static consequences, not executed tests. Unless stated otherwise, dependencies return normally, addresses are valid, dependencies do not modify the index, and arithmetic stays within range.

1. **Already complete.**  
   Captured bound 5, current index 5, accumulator 20. If F-3ecf06e452fb returns 7 and leaves the accumulator unchanged, the function writes 27 and returns 1. Iteration control does not alter this outcome. No processing callbacks execute.

2. **Zero bound.**  
   A captured bound of zero takes the completion branch for every unsigned index value. There is no processing iteration and no direct call to F-d7be9cfb4000.

3. **One ordinary iteration.**  
   Bound 5, index 2, nonzero iteration control, mode other than 3. The provider receives index 2. Both object callbacks execute in order. F-d7be9cfb4000 receives 1500 and 5. The remaining notification calls execute, the index becomes 3, and the function returns 0.

4. **Notification suppression.**  
   Under the preceding conditions but with mode 3, both object callbacks still execute. None of the three notification dependencies is called. The index becomes 3 and the return value is 0.

5. **Last index in single-iteration mode.**  
   Bound 5 and index 4 produce one processing iteration, an index update to 5, and return 0. No completion adjustment occurs during that call.

6. **Repeated processing through completion.**  
   Bound 5, index 3, and zero iteration control process indices 3 and 4. The next comparison reaches completion. F-3ecf06e452fb then supplies the accumulator adjustment, and the function returns 1.

7. **Callback changes the index.**  
   Bound 10 and initial index 2. Suppose an object callback changes the index to 7, with no later dependency changing it. The notification numerator is 4000, the final index becomes 8, and a single-iteration call returns 0.

8. **Completion helper changes the accumulator.**  
   Initial accumulator 20. If F-3ecf06e452fb changes it to 100 and returns 7, the completion write is 107, not 27.

9. **Unsigned comparison.**  
   With bound 5 and index bits `0xffffffff`, the function takes the completion branch. It does not treat that index as an in-range negative value.

10. **Changed bound field.**  
    If a callback changes the provider’s bound field from 5 to 100, the current invocation continues comparing against the captured value 5. This says nothing about other mutations made by that callback.

## Uncertainties

The following remain `documented_only`:

- Application-level names, units, and meanings of the fields, mode values, notification constant, and return values.
- The global reference’s initialization and lifetime guarantees.
- The concrete identities and full contracts of indirect callbacks.
- Dependency mutation of globals, context, provider data, or selected objects.
- Ownership and retention rules for all supplied addresses and returned values.
- Overflow behavior outside the signed 32-bit arithmetic range.
- Concurrency, reentrancy, atomicity, and external mutation between reads.
- Fault, exception, non-return, or other failure behavior of dependencies.
- Termination when iteration control is zero.

In particular, termination is not guaranteed merely because the function increments the index: dependencies can change that index, and overflow behavior is unresolved.

## Evidence

This contract is based exclusively on the supplied static function representation and opaque dependency mapping. The representation establishes field offsets, dispatch offsets, argument flow, branch conditions, call order, repeated reads, direct writes, and normal return constants.

No dependency implementation, runtime trace, memory-layout declaration, or arithmetic instruction sequence was provided. No runtime tests have been performed. Acceptance examples state consequences under explicit conditions; they are not observations of execution.
