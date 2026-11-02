<!-- Trusted provenance: F-e1befaf5b078; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e1befaf5b078

## Purpose

F-e1befaf5b078 conditionally polls a shared flag, invokes dependencies while a condition remains unsatisfied, and returns a 32-bit status based on a flag observation.

The established behavior has three parts:

- An initial flag check determines whether any dependency is invoked.
- On the active path, a context value is acquired once and reused in repeated condition evaluations.
- Each unsuccessful condition evaluation triggers a service call and, depending on its result, an action call.

The function does not directly clear the flag. Its normal return values are 0 and 21, hexadecimal 0x15. Their application-level meanings are not established.

**The initial flag check and subsequent checks need not address the same object.** The function first reads a global base value and retains it for later polling and the final check. It then reads that global location again to obtain the base for the initial flag check. No dependency runs between these two global reads, but the supplied evidence does not guarantee that their values are equal.

A waiting or timeout role is plausible because a value acquired once is repeatedly passed to a condition dependency alongside the caller’s argument. That interpretation is inferred intent, not an established semantic contract.

## Interface

### Input

The function accepts one 32-bit argument.

Whenever F-730b695a8642 is invoked, this argument is supplied unchanged as its second argument. This function performs no visible validation, arithmetic, dereference, or interpretation of the argument.

Its semantic type is unknown. The evidence does not establish a duration, deadline, pointer, identifier, or signed numerical quantity. All 32-bit patterns receive the same visible treatment at this layer.

### Output

Normal completion produces a 32-bit result:

| Result | Established condition |
|---|---|
| 0 | The initial flag check finds the bit clear, or the active path’s final flag check finds it clear |
| 21, hexadecimal 0x15 | The initial flag check finds the bit set, and the active path’s final flag check also finds it set |

On the active path, the returned status is determined by a final read through the retained base. It is not a direct translation of the condition dependency’s result.

### Shared-state layout and base selection

A global location holds a 32-bit base value used to address a shared object. The function reads this global location twice before any dependency call:

1. **Retained-base read:** The first value is retained for every loop flag check and the active path’s final flag check.
2. **Initial-gate-base read:** The global location is read again. This second value addresses the initial flag check.

The global location is the same; the values obtained from it need not be.

For each base actually used for a flag access, the relevant field has this layout:

| Property | Established behavior |
|---|---|
| Field location | Byte offset 4 from the selected base |
| Access width | 32 bits |
| Relevant mask | 0x00000020 |
| Tested bit | Bit 5, counting from bit 0 |
| Other field bits | Do not affect the decision |
| Direct field writes | None |

If the two global reads yield different bases, the initial gate examines the object selected by the second read, while later checks examine the object selected by the first read. A set bit in the initial-gate object therefore does not establish that the retained-base object has that bit set.

The complete object layout, byte order, and required alignment are not established.

### Preconditions and lifetime

For defined normal operation:

- The global base location must support both reads.
- The base obtained by the second read must identify a readable 32-bit field at byte offset 4 for the initial flag check.
- If the initial check finds the bit set, the first, retained base must identify a readable 32-bit field at byte offset 4 for all subsequent checks.
- Every accessed object must remain valid whenever accessed, including after dependency calls.
- Address formation, alignment, and access permissions must satisfy the execution environment.
- Each invoked dependency must accept the supplied arguments.

If the initial gate is clear, no flag-field access through the retained base occurs. On that path, reading and retaining the first base does not itself establish a requirement to dereference it.

No local validation or recovery for invalid storage is visible. The supplied evidence does not define behavior for address overflow, inaccessible storage, or misalignment; callers must not rely on a particular fault or wraparound outcome.

## Behavior

### Initial gate

The function first captures the base for later checks, then obtains the initial-gate base from a second read of the global location. It tests bit 5 of the 32-bit field at offset 4 through that second base.

If the bit is clear:

- Return 0.
- Invoke none of the four dependencies.
- Perform no loop or final flag check through the retained base.

The initial clear path therefore depends only on the initial-gate field, not on the value of the corresponding field in any different object selected by the retained base.

If the bit is set, execution enters the active path.

### Context acquisition

The active path invokes F-c84c3e66ec8d with no explicit arguments and retains its 32-bit result.

Assuming normal return, this call occurs exactly once per invocation that enters the active path. The returned value is reused unchanged as the first argument to every subsequent F-730b695a8642 call.

There is no additional flag check between the initial gate and context acquisition. In particular:

- A later change to the initial-gate field does not cancel context acquisition.
- A clear flag in the retained-base object does not prevent context acquisition.
- The first retained-base flag check occurs only after context acquisition returns.

No guarantee is established that F-c84c3e66ec8d preserves globals, object contents, or object lifetime.

### Repeated condition evaluation

Every loop attempt starts with a 32-bit flag-field read through the retained base.

Evaluation is short-circuited:

- If bit 5 is clear, repetition ends without a condition dependency call for that attempt.
- If bit 5 is set, F-730b695a8642 is invoked with the retained context first and the original caller argument second.
- A nonzero condition result ends repetition without executing the body.
- A zero condition result permits the body to execute.

The condition result is a 32-bit integer tested only for zero versus nonzero. No ordering, magnitude, or distinction between different nonzero results affects control flow.

The flag check precedes the condition call. There is no intervening flag recheck after that call returns zero. Consequently, if F-730b695a8642 clears the relevant flag and returns zero, the body still executes.

### Loop body

Every executed body invokes F-60f3f46b16bc with no explicit arguments.

Its 32-bit result controls one optional call:

- A zero result causes the action call to be skipped.
- Any nonzero result causes F-6750d7407908 to be invoked with numerical arguments 0 and 1, in that order.

No return value from F-6750d7407908 is consumed by this function.

After the body’s calls return normally, execution returns to a new flag check through the retained base. A nonzero service result does not itself terminate repetition; it controls only the action call.

There is no visible counter update, explicit delay, iteration limit, or modification of the caller argument or retained context between iterations. The internal behavior of the dependencies is outside the supplied evidence.

### Final status observation

When repetition ends, the function makes a separate 32-bit field read through the retained base:

- Bit 5 clear: return 0.
- Bit 5 set: return 21.

This read is distinct from any flag check that ended repetition. The function does not reuse the earlier observed field value to choose the return status.

Therefore:

- A nonzero condition result can lead to either return value, depending on the final flag.
- A clear loop flag can end repetition, yet the final check can produce 21 if the flag becomes set between those observations.
- Changes to the initial-gate object do not determine the final result when it differs from the retained-base object.

### Termination and abnormal behavior

No intrinsic iteration bound is visible.

If the retained-base flag remains set, every condition result is zero, and all invoked dependencies return normally, repetition continues without a normal return.

No local handling is shown for dependency nontermination, exceptions, faults, or other abnormal transfers of control. Such events are not visibly converted into status 0 or 21.

## Side effects

### Direct effects

The function directly performs:

- Two reads of the global base location, in the order described above.
- One initial 32-bit flag-field read through the second base.
- On the active path, repeated flag-field reads and a final flag-field read through the first, retained base.
- Conditional dependency calls.
- A 32-bit return.

No direct write to the global base, either object’s flag field, or caller-supplied storage is visible. No direct allocation or release is visible.

The lack of direct writes does not make the function side-effect-free. Dependencies can have effects not established by this evidence.

### Observable ordering

Both global-base reads precede the initial field read. The initial field read precedes any dependency call.

On the active path, context acquisition precedes all loop checks. Within each successful loop-condition evaluation, the flag read precedes the condition call. A zero condition result precedes the service call, and a nonzero service result precedes the action call. The next loop flag read occurs after the body calls return. The final field read occurs after repetition ends.

No further global-base read is visible after the initial gate. A dependency’s replacement of the global base therefore does not itself redirect later accesses: those continue through the previously retained base.

This ordering describes the visible accesses and calls. It does not establish atomicity, volatility, memory barriers, or inter-context visibility guarantees.

### Ownership

The function does not visibly acquire ownership of either selected object or transfer ownership of its input.

The acquired context is retained and forwarded, with no explicit cleanup. Its representation and resource semantics are unknown. Neither the lack of cleanup nor the later calls establish whether a dependency owns, consumes, preserves, or releases any associated resource.

Required object lifetime must not be inferred from undocumented dependency behavior.

## Dependencies

| Opaque function ID | Invocation condition | Supplied inputs | Result use |
|---|---|---|---|
| F-c84c3e66ec8d | Once after the initial gate finds bit 5 set | No explicit arguments | Retain its 32-bit result as context |
| F-730b695a8642 | A loop flag read through the retained base finds bit 5 set | Retained context, then original 32-bit argument | Zero enters the body; nonzero ends repetition |
| F-60f3f46b16bc | Once per executed body | No explicit arguments | Zero skips the action; nonzero invokes it |
| F-6750d7407908 | The service result is nonzero | Numerical values 0, then 1 | No result consumed |

These are caller-side contracts, not complete dependency specifications.

The formal widths and meanings of F-6750d7407908’s parameters are not independently established. The supplied evidence establishes the constant values and their order.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests. Unless stated otherwise, all required storage remains valid and all dependencies return normally.

### Initial gate clear

If the second base selects a field containing 0x00000000:

- Return 0.
- Invoke no dependencies.
- Do not access the field through the retained base.

A value of 0xFFFFFFDF gives the same decision because bit 5 is clear.

### Different bases: initial gate clear, retained flag set

Suppose the first global read selects one object and the second selects another. The retained-base object has bit 5 set, but the initial-gate object has it clear.

Expected behavior:

- Return 0 without context acquisition.
- Do not inspect the retained-base field.

The retained object’s flag does not override the initial gate.

### Different bases: initial gate set, retained flag clear

Suppose the second base selects an object whose flag is set. The retained-base field is clear at both its first loop check and its final check.

Expected behavior:

- Invoke F-c84c3e66ec8d once.
- Invoke no condition, service, or action dependency.
- Return 0.

The set initial-gate flag triggers context acquisition but does not guarantee a condition call.

### Immediate condition completion

Given a set initial flag, a set first loop flag, a nonzero first condition result, and a set final flag:

- Acquire context once.
- Invoke the condition once with that context and the original argument.
- Invoke neither body dependency.
- Return 21.

### Condition clears the flag but returns zero

Given a set loop flag and a condition call that clears the retained-base flag but returns zero:

- Still invoke the service dependency.
- If its result is nonzero, invoke the action with 0 and 1.
- Only after the body returns, check the flag again.

If the next loop check and final check both find the flag clear, return 0.

### Global replacement during a dependency

Suppose a dependency changes the global base to select another object.

Expected behavior:

- Subsequent loop and final checks still address the originally retained object.
- The replacement object’s flag does not redirect these checks.

This scenario requires the retained object to remain valid; global replacement alone supplies no lifetime guarantee.

### Final observation determines status

If a condition call returns nonzero and the final retained-base flag is clear, return 0.

Conversely, if a clear loop check ends repetition but the final retained-base flag is set, return 21.

### Continued repetition

If every loop flag check finds bit 5 set and every condition result is zero:

- Execute the service call on every iteration.
- Execute the action only when the corresponding service result is nonzero.
- Continue without a visible iteration bound while these conditions persist.

## Uncertainties

The following remain **documented_only**:

- The application meanings of bit 5 and status 21.
- The semantic types of the input and acquired context.
- Whether the dependencies implement timing, readiness, expiration, or another operation.
- Whether the two global reads can observe different values in the actual execution environment.
- Dependency effects on globals, either object, ownership, and lifetime.
- Concurrency, interrupt, or hardware-update mechanisms.
- Atomicity, volatility, synchronization, and memory-ordering guarantees.
- Complete object layout, byte order, alignment, and address-overflow behavior.
- Fault and abnormal-control-flow behavior.
- Any termination guarantee.

The visible address calculation uses byte offset 4. No arithmetic on the input or acquired context is visible. Signed-range handling, elapsed-time calculations, and wraparound inside dependencies cannot be specified from this evidence.

## Evidence

The supplied static body establishes the 32-bit input and output, the two ordered reads of the global base, and the distinct uses of their results: first for retained-base polling and final checks, second for the initial gate.

It also establishes the offset-4 32-bit field accesses, mask 0x20, conditional call ordering, zero-versus-nonzero result tests, unchanged context reuse, and normal return alternatives 0 and 21.

No dependency implementations were supplied. Their identities are represented only by the provided opaque function IDs.

Polling is established by the repeated observations and calls. A more specific waiting, timeout, or device-operation interpretation remains inferred intent.

No runtime tests have been performed.
