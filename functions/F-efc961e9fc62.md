<!-- Trusted provenance: F-efc961e9fc62; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-efc961e9fc62

## Purpose

F-efc961e9fc62 conditionally updates a selected object and performs two indirect dispatches through that object. A difference comparison gates both the direct object writes and the dispatches.

The supplied static representation establishes three principal paths:

- **No selected object:** return without invoking F-82d167821c25.
- **Selected object, unsigned 32-bit difference below five:** invoke F-82d167821c25, then return without the direct object writes or dispatches.
- **Selected object, difference at least five:** store the low byte of the second input, conditionally replace a 32-bit field, and perform two displayed indirect dispatches.

On the accepted path, an external shared dispatch value is read twice before the first dispatch takes effect: once to save a value for the second dispatch, and separately within the first dispatch expression for its argument. These reads must not be treated as one snapshot. Equality of the two supplied argument values requires a stability assumption.

A clock-based throttle followed by a state update and notifications is plausible intent, not an established semantic contract. No time unit, Boolean interpretation, or notification meaning is established.

This document describes static evidence only. No runtime tests have been performed.

## Interface

### Explicit inputs and result

| Input | Recovered width | Established use |
|---|---:|---|
| First input | 32 bits | No visible use in the function body. |
| Second input | 32 bits | Tested for equality with zero at full width; its low eight bits are stored on an accepted path. |

The recovered second input is signed, but the branch distinguishes only zero from nonzero. Negative inputs follow the nonzero path. The byte store truncates the input; it does not normalize it to a Boolean.

The first input has no visible effect on the decisions, memory accesses, or displayed dispatch arguments. Its significance to callers or the original calling convention is unknown.

There is no explicit result value. Observable behavior consists of reads, early returns, direct writes, the direct dependency call, and indirect transfers. No status result or exception-to-result conversion is shown.

### Implicit inputs

The function also consumes:

- A selected-object pointer obtained through global storage.
- The 32-bit result of F-82d167821c25.
- Fields and dispatch-table data reachable through the retained object.
- Three external 32-bit dispatch values.
- Four incoming machine-context words exposed in the first recovered invocation.

The external dispatch values are called the **shared external value**, **first-only external value**, and **second-only external value**. These are role descriptions, not semantic types or ownership claims.

The shared external value has two distinct uses:

1. An explicit read saves its value for the second dispatch.
2. A separate read within the first dispatch expression supplies that dispatch’s argument.

### Relevant layout

The recovered representation uses 32-bit object words and a 32-bit pointer layout.

| Location | Access width | Role |
|---|---:|---|
| Object offset `0x00` | 32 bits | Dispatch-table pointer. |
| Object offset `0x1c` | 8 bits written | Low-byte destination for the second input. |
| Object offset `0x20` | 32 bits | Comparison field, conditionally replaced with the dependency result. |
| Dispatch-table offset `0x58` | 32-bit target pointer | Indirect dispatch target. |

The offset `0x58` is relative to the dispatch table, not the object.

The byte write at `0x1c` does not explicitly modify adjacent bytes. These accesses do not establish a complete object layout, field names, alignment guarantees, or semantic types.

### Preconditions

Completion through the displayed operations requires readable selector storage and valid memory for each access actually reached.

For a non-null selected object:

- The retained object must remain valid across F-82d167821c25 and subsequent accesses.
- The comparison field must be readable after that dependency returns.
- An accepted path requires the byte destination to be writable.
- An accepted path with a nonzero second input also requires the 32-bit comparison field to be writable.
- Each dispatch requires a readable table pointer, readable target entry, and callable target compatible with the actual machine invocation.
- Reaching the second dispatch requires the first transfer to return control with the required retained state usable.

Null selection is checked. Object extent, lifetime, table validity, target validity, and access permissions are not checked in the visible body.

## Behavior

### Selection and direct dependency

The function obtains the selected-object pointer and retains it.

If the pointer is null, the function returns immediately. It does not invoke F-82d167821c25, read the comparison field, perform either direct object write, or perform either dispatch.

Otherwise, it invokes F-82d167821c25 without explicit arguments. The object pointer is acquired before this call and is not visibly reacquired afterward.

Changing the global selection during F-82d167821c25 therefore does not, by itself, redirect subsequent operations to a newly selected object. Subsequent accesses use the retained object. This does not imply that the dependency preserves that object’s lifetime or contents.

### Difference gate

After F-82d167821c25 returns, the function reads the retained object’s 32-bit field at `0x20`.

The recovered comparison interprets the result-minus-field difference as an unsigned 32-bit quantity. In the fixed-width behavioral model:

**difference = (dependency result − comparison field) modulo 2³²**

A difference from zero through four causes an immediate return. A difference of five or more accepts the operation.

The comparison is not a signed elapsed-value test. A small backward change can produce a large unsigned difference and pass. Wraparound can produce a small difference and reject.

This describes the recovered 32-bit comparison; it does not establish the original language’s overflow semantics or an independently inspected instruction sequence.

The second input does not affect gate acceptance. A dependency-side change to the comparison field affects the gate because the field is read after the call.

### Accepted-path writes

On acceptance, the function:

1. Writes the second input’s least significant eight bits to object offset `0x1c`.
2. Tests the full 32-bit second input against zero.
3. If nonzero, writes the dependency result to object offset `0x20`.

The byte write precedes the conditional 32-bit write. Both precede the shared-value capture and the dispatches.

A nonzero input can have a zero low byte. For example, input 256 stores a zero byte but still replaces the comparison field. Input zero stores a zero byte without directly replacing that field.

### Saved shared value and first dispatch

After the object writes, the function reads the shared external value and saves that 32-bit value for the second dispatch.

The first dispatch expression then separately reads the shared external value for its own argument. The first dispatch is therefore not specified to receive the saved value.

The recovered first invocation supplies:

- The retained object pointer.
- The shared external value obtained by the separate read within this invocation.
- The first-only external value.
- A target-pointer value obtained through the retained object’s dispatch-table slot at `0x58`.
- Four additional incoming machine-context words.

The call target is also obtained through the retained object’s dispatch-table slot at `0x58`.

The representation contains both a target lookup and a target-pointer argument lookup. It does not establish their precise evaluation order, their order relative to the other first-dispatch argument reads, or the exact number of machine-level slot loads. The two target-related values must not be assumed identical under intervening mutation.

The explicit saved-value read precedes the first dispatch expression. Within that expression, the ordering of argument and target evaluation remains unspecified.

The four additional words are not shown as initialized by this function. They cannot be replaced with assumed zeros or treated as proven irrelevant. Whether they are intentional parameters or recovery artifacts is unresolved.

No use of a first-dispatch result is visible.

### Second dispatch

If the first dispatch returns control, the function performs the second displayed indirect dispatch.

Its target is resolved again through the retained object’s dispatch-table pointer and slot at `0x58`. The first target is not visibly saved for reuse. A first-dispatch change to the table pointer or slot can therefore change the second target.

The recovered second invocation supplies:

- The retained object pointer.
- The shared value saved before the first dispatch expression.
- The second-only external value.

It does not reread the shared external value for this argument. A change to that external storage during the first dispatch does not, by itself, change the saved argument supplied to the second dispatch.

The second-only value appears in the second invocation expression. Its semantic meaning and the nature of its underlying external storage are unknown.

The supplied representation marks the final indirect transfer as incompletely recovered. Its presence is established in that representation; its exact calling convention, call-versus-tail-transfer mechanics, and return behavior remain documented_only.

## Side effects

### Direct writes and ordering

The visible body directly writes only:

- One byte at object offset `0x1c` on an accepted path.
- One 32-bit value at object offset `0x20` on an accepted path with a nonzero second input.

The supported sequence is:

1. Obtain and retain the selected object.
2. Return if it is null.
3. Invoke F-82d167821c25.
4. Read the retained object’s comparison field.
5. Apply the unsigned difference gate; return if below five.
6. Write the input’s low byte.
7. Conditionally write the dependency result.
8. Read and save the shared external value for the second dispatch.
9. Evaluate the first dispatch expression, including a separate shared-value read, the first-only value, and target-related accesses; their internal evaluation order is unspecified.
10. Perform the first dispatch.
11. If control returns, resolve and perform the second dispatch using the saved shared value and the second-only value.

No rollback or restoration is visible. Earlier object writes remain performed if a later access or transfer fails. The first dispatch is reached after both applicable direct writes.

### Ownership, interference, and failure

No allocation, deallocation, reference-count operation, lock operation, or explicit ownership transfer is visible in this body.

That observation does not constrain the callees. F-82d167821c25 and the unidentified indirect targets may change globals, mutate the object, affect lifetime, fail, or not return. Their contracts are unavailable.

The object is not visibly released or reacquired between dispatches. Safe continuation cannot assume that the first target preserves it.

No synchronization is shown. The evidence establishes neither atomic visibility of the object updates nor a coherent snapshot across the two shared-value reads or repeated table accesses. It also does not establish valid concurrent-access rules for the original implementation.

There is no visible recovery from invalid memory, invalid targets, exceptions, or non-returning callees.

## Dependencies

### F-82d167821c25

Established use:

- Called once when the initially selected object is non-null.
- Called before reading the comparison field.
- Supplies the 32-bit value used by the gate and conditional field replacement.

Unknowns include units, progression, monotonicity, global effects, object mutation, lifetime effects, failure behavior, and whether control returns. A clock interpretation is inferred intent only.

### Unidentified indirect dependencies

Both indirect targets are obtained through the retained object’s dispatch-table slot at `0x58`. No opaque function ID is available for either concrete target; no identity is invented here.

The first and second targets need not be the same. Their parameter types, exact calling conventions, ownership contracts, state effects, failure behavior, and return behavior are unknown.

The first recovered invocation exposes more operands than the second. This does not establish a stable source-level prototype for either target.

## Acceptance examples

These are conditional predictions from the static representation, not executed tests. Unless stated otherwise, assume valid accesses, returning callees, and no intervening mutation.

1. **Null selection.** A null selected-object pointer causes immediate return for any second input. Neither the direct dependency nor either indirect target is invoked.

2. **Rejected boundary.** A comparison field of 100 and dependency result of 104 produce difference four. No direct object writes, shared dispatch-value reads, or dispatches are reached.

3. **Accepted boundary.** A comparison field of 100, result of 105, and second input 1 cause byte value 1 to be written, followed by 105 at `0x20`. The saved shared-value read and first dispatch expression follow. The second dispatch is reached if the first returns.

4. **Zero input.** With an accepted difference and second input zero, the byte write stores zero, but this function does not replace the comparison field. Dispatches still proceed. Callee effects prevent asserting the field’s final value without further assumptions.

5. **Truncation versus full-width test.** With an accepted difference and input 256, the byte write stores zero and the comparison field is replaced. With input −1 in the recovered 32-bit representation, the byte write stores `0xff` and the comparison field is also replaced.

6. **Wraparound.** A field of `0xfffffffe` and result 2 produce difference four and reject. Result 3 produces difference five and accepts.

7. **Backward movement.** A field of 100 and result 99 produce unsigned difference `0xffffffff`; the gate accepts rather than treating the difference as negative.

8. **Dependency changes state.** If F-82d167821c25 changes the retained object’s comparison field before returning, the gate uses that changed field. If it only changes global selection, subsequent accesses still use the originally retained object.

9. **Distinct shared-value reads.** Suppose the explicit saved-value read obtains 17, and a change between the two reads causes the separate first-dispatch read to obtain 23. The first dispatch receives 23 in the shared-value position; the second receives saved value 17. Both receive the same value only if the relevant reads produce equal values, such as under a stability assumption.

10. **Mutation during the first dispatch.** Suppose both shared-value reads obtain 17, then the first target changes the external storage to 29. The second dispatch still receives saved value 17. If the first target also installs another valid table or slot, the second target is resolved from that updated object state.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the fields and external values.
- Complete object layout, ownership, and lifetime rules.
- Original interface types beyond the recovered widths and operations.
- Exact instruction-level arithmetic and memory-access realization.
- Internal evaluation order of the first dispatch’s arguments and target lookups.
- Exact machine-level count of target-slot reads.
- Whether external values reside in mutable storage, relocation data, or another representation.
- Whether the additional first-dispatch words are intentional arguments.
- The final indirect transfer’s precise control-flow mechanics.
- Callee effects, concurrency rules, and failure handling.

The two shared-value reads are explicit in the supplied representation; their equality is not. Conversely, their distinct representation does not prove that a particular compiled execution performs two physical memory transactions.

## Evidence

The evidence is one supplied recovered function representation and the opaque identity mapping for F-82d167821c25.

That representation supports the null-selection return, retained object pointer, direct dependency call, post-call comparison-field read, unsigned threshold gate, field offsets and widths, accepted-path write order, full-width nonzero test, explicit shared-value save, separate first-dispatch shared-value read, saved-value use by the second dispatch, and two displayed indirect transfers.

It also explicitly identifies incomplete recovery of the final indirect transfer. The contract retains that limitation rather than assigning an unsupported calling convention.

No runtime tests, instruction-level inspection, callee inspection, caller inspection, or object-definition inspection have been performed. The acceptance examples are conditional consequences of the static model, not observed executions.
