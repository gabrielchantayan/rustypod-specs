<!-- Trusted provenance: F-4b64bd6b07aa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4b64bd6b07aa

## Purpose

F-4b64bd6b07aa performs a single indirect dispatch. It obtains a dispatch-base value from caller-supplied storage, obtains a callable address from a fixed displacement relative to that base, and invokes that address. If the invoked function returns normally, F-4b64bd6b07aa returns without providing a result.

The established behavior is the dispatch itself. The supplied representation does not establish the operation performed by the selected function, the meaning of the caller-supplied storage, or an application-level success condition.

An object-style dispatch table is a plausible interpretation of the access pattern, but it is inferred intent rather than an established interface guarantee.

## Interface

**Input:** One pointer to storage whose first accessed value is represented as an integer. That value supplies the base for the subsequent callable-address lookup.

The input is not directly used as the callable address. There are two distinct data accesses:

1. A read through the input pointer to obtain the dispatch-base value.
2. A read of a callable-address slot at displacement `0x128`, or 296 decimal, from that base.

The displacement is applied to the loaded integer value before that value is treated as the address of callable-address storage. It is not a displacement of 296 integer elements or 296 function-pointer elements.

**Output:** No result value is exposed. The wrapper does not inspect, translate, or forward a return value from the selected function.

**Explicit call arguments:** The indirect invocation contains no explicit arguments. In particular, the supplied representation does not explicitly pass the original input pointer to the selected function. A receiver argument or other hidden argument must not be inferred from the dispatch-table interpretation.

**Widths and representation:**

- The first load is represented as an integer load, not a native-width data-pointer load.
- The supplied material does not independently establish the integer width, address width, callable-pointer width, signed-address interpretation, or calling convention.
- The second load must retrieve whatever callable-address representation the actual execution environment requires.
- A port or reimplementation must establish these widths from an authoritative target interface. It must not silently replace the first load with a native pointer load merely because the loaded value is subsequently used to form an address.

**Normal-execution preconditions:** The input must permit the first read; the loaded value and displacement must form a usable address; that address must permit a callable-address read; and the resulting target must support the invocation performed here. These are execution requirements, not conditions checked by this function.

## Behavior

### Dispatch resolution

The function first reads the integer value at the supplied pointer. No preliminary null check, bounds check, type check, or alternate-path selection is shown.

It then forms the slot address using the loaded value and the fixed displacement of 296. There is no input-dependent slot index. No additional field from the caller-supplied storage is accessed explicitly.

Next, it reads the callable address stored at the calculated slot. The address of the slot and the callable address held in that slot have different roles: the function calls the retrieved callable address, not the slot address itself.

The established logical ordering is:

- Read the dispatch-base value.
- Use that value to determine the slot address.
- Read the callable address from that slot.
- Invoke the retrieved callable address.
- Return if the invocation returns normally.

This ordering describes the data dependencies visible in the supplied representation. It does not establish atomic accesses, memory barriers, volatile semantics, or a synchronization protocol.

### Invocation and completion

Exactly one indirect invocation appears on the normal dispatch path. There is no shown retry, fallback target, loop, or second dispatch.

The invocation has no explicit arguments. Nothing in the supplied representation establishes that the selected function receives the input pointer, receives the dispatch base, or is entitled to interpret either value as an object.

After a normal return from the selected function, the wrapper returns immediately. It performs no shown post-call reads, result checks, state restoration, or cleanup.

If the selected function does not return, normal completion of this function is not established. Non-returning behavior could include continued execution, process termination, or a nonlocal transfer, depending on the selected function and execution environment. The wrapper supplies no shown recovery mechanism.

### Arithmetic boundaries

The address displacement is fixed, but its application still depends on the width and semantics of the loaded integer.

The supplied representation does not define a checked-addition policy, a saturation policy, or an explicit wraparound policy. If adding 296 exceeds the usable range of the relevant representation, the available evidence does not establish a valid resulting slot address.

Accordingly, acceptance at arithmetic boundaries requires separate target-width evidence. Neither modulo-address wrapping nor rejection with a defined error may be promised from this material alone.

### Invalid inputs and dispatch state

No deliberate error return is present for any of the following:

- A null or unreadable input pointer.
- An input read that violates the target’s access requirements.
- A dispatch base that does not identify readable slot storage.
- An invalid callable address in an otherwise readable slot.
- A target incompatible with the actual call interface.

Such conditions can prevent normal execution. The precise outcome—such as a fault, termination, or another target-specific effect—is not specified by the supplied representation. They must not be described as successful no-ops.

## Side effects

The wrapper explicitly performs two data reads and one indirect invocation. It contains no explicit data writes.

That does **not** make the complete operation read-only. The selected function may modify global state, external resources, the input storage, the dispatch storage, or other reachable state. Its effects are not available here.

No ownership transfer is explicitly expressed by the wrapper. It does not visibly allocate, retain, release, or free storage. Conversely, the absence of those operations in the wrapper does not establish that the selected function preserves or releases any particular allocation.

The wrapper neither saves nor restores application state around the call. Callers must not rely on it to undo changes made by the selected function.

Once a callable address has been retrieved, the supplied behavior contains no second lookup before invocation. A change to the slot after that retrieval does not introduce a shown revalidation step. This observation does not define safe concurrent mutation: atomicity, race behavior, and memory visibility remain unspecified.

## Dependencies

**F-unresolved-0128** is an opaque designation used in this document for the dynamically selected dependency. It is not a resolved identity from the provided dependency map.

Its callable address is obtained from the slot at displacement 296 relative to the value read through the input pointer.

The established dependency interaction is limited to:

- One invocation with no explicit arguments.
- No wrapper-level use of a returned value.
- Normal wrapper return following normal dependency return.

The dependency’s signature, state effects, ownership rules, failure behavior, and termination behavior are unknown. Different valid dispatch bases may select different dependencies; the evidence does not identify any concrete target.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not results of executed tests.

1. **Valid dispatch and normal return.**  
   Given readable input storage, a representable slot address, a readable callable-address slot, and a compatible callable target that returns normally, the function invokes that target once and then returns without an exposed result.

2. **Different bases select different slots.**  
   Given two valid input values identifying different dispatch bases, each invocation resolves its target from displacement 296 relative to its own loaded base. The function does not substitute a fixed global target.

3. **Unrelated adjacent contents.**  
   If all required accesses remain valid, changing data adjacent to the input’s first accessed integer does not change any explicit wrapper read. Effects caused by the selected function remain outside this criterion.

4. **Dependency modifies state.**  
   If the selected function changes externally observable state and returns normally, the wrapper returns without a shown restoration step. Acceptance must not require the wrapper to reverse those changes.

5. **Dependency does not return.**  
   If the selected function never returns normally, the wrapper is not required to reach its normal return.

6. **Readable slot containing an invalid target.**  
   Readability of the slot alone is insufficient for successful execution. No null-target check, fallback dispatch, or defined error result is established.

7. **Boundary address calculation.**  
   If the loaded value plus 296 is not valid under the actual target’s integer and address semantics, normal dispatch is outside the established valid-input contract. A wrapping implementation cannot be accepted solely on the basis of this evidence.

8. **No inferred receiver.**  
   A replacement that explicitly forwards the original input pointer to the selected function adds behavior not established here. Any such forwarding requires separate interface evidence.

## Uncertainties

The following remain **documented_only**:

- Exact integer, data-address, and callable-address widths.
- Byte order and required alignment for both reads.
- Integer-to-address interpretation and overflow behavior.
- The precise ABI and any machine-level implicit call state.
- The identities and contracts of possible dispatch targets.
- Whether the input represents an object, handle, table reference, or another structure.
- Storage lifetime, aliasing constraints, and concurrent mutation rules.
- Dependency effects on globals, resources, and memory ownership.
- Target-specific outcomes of invalid reads or invalid calls.

The object-dispatch interpretation is inferred intent only. It does not establish inheritance, a class layout, an implicit receiver, or a named operation.

## Evidence

**Established static evidence:** The supplied representation contains one integer read through the input pointer, a fixed displacement of `0x128`, one callable-address read, one indirect invocation without explicit arguments, and a subsequent resultless return. It contains no explicit validation, data writes, allocation, release, retry, or recovery.

**Inferred intent:** The fixed-offset callable lookup resembles dispatch through a table associated with caller-supplied state.

**Unknown:** Target ABI details, concrete dependency behavior, and application semantics are not supplied.

No runtime tests have been performed.
