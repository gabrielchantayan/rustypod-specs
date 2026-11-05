<!-- Trusted provenance: F-f5ca6d8eb9c5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f5ca6d8eb9c5

## Purpose

F-f5ca6d8eb9c5 conditionally dispatches two indirect operations. The first receives the function’s two incoming 32-bit values. The second, reachable only after the first reports zero and a separate gate again reports zero, receives a zero value and a captured 32-bit value from shared storage.

The established behavior is an ordered sequence of gate calls, shared-storage reads, pointer checks, and indirect calls. It is not established that these operations represent initialization, event handling, cleanup, or any other domain-specific activity.

The function has no explicit result value. Its observable effects consist of calls to other functions, reads of shared storage, and whatever effects those callees produce. No direct write to shared storage is visible in the supplied body.

## Interface

### Inputs and result

The function accepts two values, each 32 bits wide:

- **Input 1:** forwarded unchanged as the first argument of the first indirect operation.
- **Input 2:** forwarded unchanged as the second argument of that operation.

The body does not interpret either input as a pointer, signed number, flag set, or structured value. Any such interpretation belongs to the selected indirect target and is unknown here.

There is no explicit return value. All visible normal completion paths return without providing a status to the caller. The integer results used internally determine control flow; they are not forwarded as this function’s result.

### Shared-storage model

The following labels describe storage roles, not source symbols or domain meanings:

| Label | Observed role |
|---|---|
| **Primary anchor** | A shared value used as the base for reading a pointer-sized field at byte offset 4. |
| **Primary object** | The object designated by that field. Its first word supplies the first indirect-call target. |
| **Captured-value source** | Shared storage supplying one 32-bit value after the first indirect operation returns. |
| **Secondary anchor** | A shared value used as the base for reading a 32-bit field at byte offset 4. |
| **Secondary object** | The object designated by that field. Its word at byte offset 4 supplies the second indirect-call target. |

The supplied representation uses 32-bit values and a 32-bit pointer layout. The primary-object field and both indirect-call target slots are represented as pointer-sized words. The secondary-object field is represented as a 32-bit integer that is tested against zero and then used as an address.

The offset of 4 denotes a byte displacement. It is not an object index or an element count. The body performs no visible scaling, bounds calculation, or variable-width arithmetic.

These labels distinguish access roles only. They do not establish that the storage regions or objects are disjoint. Aliasing between roles is not ruled out.

### Preconditions for defined normal execution

Each reached memory access requires an address that is valid for the access performed. In particular:

- The primary anchor must permit reading its field at byte offset 4.
- A nonzero primary-object field must designate storage whose first word can be read as an indirect-call target.
- The captured-value source must be readable whenever execution reaches its read.
- The secondary anchor must permit reading its field at byte offset 4.
- A nonzero secondary-object field must designate storage whose word at byte offset 4 can be read as an indirect-call target.
- Each invoked target must be callable with the arguments supplied.

These are execution requirements, not validations performed by the function. The body does not establish object lifetime, accessible extent, alignment, executable target validity, or ABI compatibility.

## Behavior

### Initial gate

The first operation is a call to **F-bafc2e4b678f**, with no explicit arguments shown.

Its result is treated as a 32-bit integer and compared with zero:

- A nonzero result causes immediate normal return.
- A zero result permits the primary-object check.

No primary-object field read, indirect call, captured-value read, or secondary-object field read is shown before this first gate completes.

The body distinguishes only zero from nonzero. It gives no special meaning to positive versus negative results and does not propagate the gate’s result.

### Primary-object check and first indirect operation

After a zero initial gate result, the function reads the field at byte offset 4 from the primary anchor.

If that field is null, the function returns. In this case, the first indirect operation is not invoked, the captured-value source is not read, and no second gate call occurs.

If the field is nonnull, the function obtains a call target from the first word of the designated primary object and invokes it with:

1. Input 1, unchanged;
2. Input 2, unchanged.

The supplied body expresses the null check and target lookup as separate accesses to the primary-object field. It does not establish one retained object-pointer snapshot spanning both operations. Therefore, a contract requiring both accesses to observe the same pointer needs an additional stability assumption.

There is no explicit null check on the first indirect-call target itself. A nonnull primary object whose first word is null does not receive a graceful early-return path in the visible body.

The result of the first indirect operation is treated as a 32-bit integer. Only its equality or inequality to zero matters.

### Captured-value read

After the first indirect operation returns normally, the function reads one 32-bit value from the captured-value source and retains it for possible use by the second indirect operation.

In the supplied representation, this read precedes the branch on the first indirect operation’s result. Consequently, the represented sequence includes the read even when that result is nonzero and the retained value will not subsequently be used.

If the first indirect operation returned nonzero, the function then returns. It does not call the gate again and does not inspect the secondary-object field.

The captured value is read after the first indirect operation, not before it. Changes made to that source by the first indirect operation can therefore affect the retained value.

### Second gate

If the first indirect operation returned zero, the function calls **F-bafc2e4b678f** again.

This is a separate call. Its result is not assumed to equal the first gate result.

- A nonzero result causes return.
- A zero result permits the secondary-object check.

The captured value has already been obtained before this second gate call. If the gate changes the captured-value source, that change does not replace the retained argument value. The gate may nevertheless affect other shared state used by later accesses; no preservation guarantee is supplied.

### Secondary-object and target checks

After a zero second gate result, the function reads the 32-bit field at byte offset 4 from the secondary anchor.

If the field equals zero, the function returns without invoking the second indirect operation.

Otherwise, the field is used as an object address. The function reads a target from byte offset 4 within that object. It checks this target against null:

- A null target causes return.
- A nonnull target is invoked.

As with the primary-object field, the supplied body expresses the secondary-object zero check and subsequent target lookup through separate field accesses. A stable pointer value across those accesses is not guaranteed by this function.

The second indirect operation receives:

1. A zero value;
2. The retained 32-bit value previously read from the captured-value source.

The first argument is visibly zero. Its intended type—such as an integer zero or a null pointer—is not established.

No result from this operation is inspected. After it completes normally, the enclosing function completes without an explicit result. The supplied representation does not establish whether this final transfer uses an ordinary call-and-return sequence or a tail transfer.

### Dispatch bounds

For one invocation following the visible body:

- F-bafc2e4b678f is called once or twice.
- The first indirect operation is called at most once.
- The second indirect operation is called at most once.
- The second indirect operation cannot be reached unless the first indirect operation returned zero and both gate calls returned zero.

There is no visible loop, retry, fallback target, or recovery operation.

## Side effects

The body directly reads shared storage but shows no direct shared-storage writes, allocation, deallocation, or ownership transfer.

That does not make the function side-effect-free. Every invoked callee may have effects not specified by the supplied material. In particular, neither F-bafc2e4b678f nor either indirect target is known to preserve:

- The anchors or their fields;
- Either object or either target slot;
- The captured-value source;
- Memory reachable through the incoming arguments;
- Object lifetimes or external resources.

The function does not visibly retain, release, destroy, or copy an object. Passing the inputs onward does not by itself establish transfer of ownership. Likewise, passing the captured value does not establish that it is a resource handle or an owned pointer.

The ordering of visible operations matters. The first gate can affect the primary lookup. The first indirect operation can affect the captured value and later shared-state accesses. The second gate can affect the secondary lookup, but it occurs too late to change the already-retained second argument.

No locking, atomic access, or synchronization protocol is visible. The repeated pointer-field accesses do not support a claim of race safety or snapshot consistency.

The function reports no explicit errors. Early returns indicate that a condition prevented further dispatch, but the caller receives no reason code. Invalid memory or invalid call targets are not converted into a documented error result by the visible body.

## Dependencies

### F-bafc2e4b678f

This is the only statically identified external dependency.

Established usage:

- Called with no explicit arguments shown.
- Returns a value treated as a 32-bit integer.
- Zero allows the next stage.
- Nonzero stops further processing.
- Called a second time only after the first indirect operation returns zero.

Unknown behavior includes its purpose, state dependencies, side effects, failure modes, and whether it returns normally. Its two results must not be assumed identical.

### Dynamically selected targets

Two other dependencies are selected from shared memory rather than identified by a supplied opaque function ID.

The **primary dynamic target** comes from byte offset 0 in the selected primary object. It receives the two original 32-bit inputs. Its 32-bit result controls whether the second gate is reached.

The **secondary dynamic target** comes from byte offset 4 in the selected secondary object. It receives zero and the captured 32-bit value. Its result is not consumed.

No additional opaque IDs are available for these targets. Their identities, signatures beyond the observed argument use, ownership rules, and broader contracts are documented_only unknowns.

## Acceptance examples

These examples are conditional contracts derived from the supplied static representation. They are not runtime observations. Except where an example addresses instability, assume valid accessible storage, stable fields during each lookup, and normal callee returns.

1. **Initial gate stops processing.**  
   If the first F-bafc2e4b678f call returns any nonzero 32-bit value, the function returns without reading the primary-object field or invoking either dynamic target.

2. **Primary object absent.**  
   If the first gate returns zero and the checked primary-object field is null, the function returns. There is no captured-value read and no second gate call.

3. **First target stops further dispatch.**  
   If the first gate returns zero, the primary object and its target are valid, and that target returns nonzero, the target receives both inputs unchanged. The represented sequence then reads the captured-value source and returns without the second gate call.

4. **First target changes the captured value.**  
   If the first target changes the captured-value source from one 32-bit pattern to another and returns zero, the subsequent capture uses the later pattern, subject to any other intervening effects.

5. **Second gate stops processing.**  
   If the first target returns zero but the second gate returns nonzero, the function returns without the secondary-object field lookup. The captured-value read has already occurred.

6. **Second gate changes the source after capture.**  
   If the captured value is `0x12345678` and the second gate changes its source to `0x87654321` while returning zero, a subsequently reached secondary target receives `0x12345678`, not the replacement value.

7. **Secondary object absent.**  
   If both gates and the first target return zero, but the checked secondary-object field is zero, the second target is not invoked.

8. **Secondary target absent.**  
   If the secondary object is present but its target slot is null, the function returns without invoking that target.

9. **Complete dispatch.**  
   If both gate results and the primary target’s result are zero, and all required pointers and targets are valid, the observable call order is: initial gate, primary target, second gate, secondary target. The secondary target receives zero and the value captured between the primary target and second gate.

10. **Primary target absent despite a present object.**  
    A nonnull primary object with a null first target does not satisfy a graceful-return case. The body attempts indirect dispatch without a target-null check; the resulting platform behavior is not specified here.

11. **Pointer field changes between accesses.**  
    If a checked object-pointer field changes before the separate target lookup, the supplied representation does not guarantee dispatch through the previously checked object. A stronger outcome requires a stability or synchronization precondition.

## Uncertainties

The following remain documented_only:

- The domain purpose of the function, gate, storage, and callbacks.
- Exact calling conventions and complete callback signatures.
- Whether incoming values or the captured value represent integers, pointers, flags, or handles.
- Whether the two anchors or referenced objects alias.
- Storage mutability, volatility, atomicity, synchronization, and lifetime guarantees.
- Whether callees preserve globals, return normally, release memory, or initiate nonlocal control transfer.
- The platform’s response to invalid addresses or invalid indirect-call targets.
- Whether the final indirect transfer is an ordinary call or a tail transfer.
- Whether the represented read order and repeated field accesses exactly match machine-level observations.

A two-stage notification or fallback-dispatch design is a possible interpretation, but it is inferred intent, not an established requirement. No such interpretation is needed to specify the visible behavior.

## Evidence

The established static evidence is the supplied function representation and the supplied opaque dependency mapping. It supports the two zero/nonzero gate decisions, the two indirect dispatch layouts, unchanged forwarding of the incoming values, capture of a 32-bit shared value, the explicit null checks, and the listed early-return paths.

The representation is not sufficient to establish domain semantics, concurrency guarantees, ownership contracts, or machine-level access fidelity beyond the qualifications above. Source comments and indirect-control-flow annotations are not independent proof of those properties.

No implementation access or runtime tests have been performed. All acceptance examples are conditional consequences of the supplied static evidence, not measured execution results.
