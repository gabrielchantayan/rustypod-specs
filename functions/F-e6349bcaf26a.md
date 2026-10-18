<!-- Trusted provenance: F-e6349bcaf26a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e6349bcaf26a

## Purpose

F-e6349bcaf26a is a dispatch wrapper. It selects a callable entry through an object-like value obtained from shared storage and invokes that entry with five arguments. The final argument carries the wrapper’s incoming 32-bit value. The other arguments come from shared state, including a value paired with a version adjusted by negative three.

The wrapper has no explicit result value, validation, allocation, cleanup, or error translation. If the dispatched dependency returns normally, the wrapper returns normally without exposing the dependency’s result.

An object-method or interface-dispatch interpretation is plausible, but not established as an application-level contract. The supplied material does not identify the operation, the meaning of the incoming value, or the semantic roles of the shared arguments.

## Interface

### Input

The wrapper accepts one value represented as an uninterpreted four-byte quantity.

- Its established width is 32 bits.
- No signedness, enumeration, flag layout, pointer interpretation, or valid-value range is established.
- The wrapper performs no explicit arithmetic, masking, validation, or dereference on this input.
- It supplies the input as the fifth argument of the indirect invocation.

All 32-bit input patterns are treated alike by the wrapper’s visible logic. That does not establish that the dependency accepts every pattern.

### Output

The wrapper has no explicit return value.

Any result produced by the indirect dependency is unused by the visible wrapper. This does not establish that the dependency itself has a void return interface.

Normal completion is conditional on successful evaluation of the dispatch expression and arguments, followed by normal return from the dependency. Behavior when the dependency does not return normally is not specified by the supplied material.

### Shared inputs

The wrapper relies on three shared expressions, described here by role rather than by source name:

1. **Receiver source:** supplies an object-like address used both to locate the callable entry and as the first invocation argument.
2. **Auxiliary value:** supplies the second invocation argument.
3. **Adjustment source:** supplies the unadjusted fourth argument and the expression used to produce the third argument by adding negative three.

These descriptions identify uses, not application semantics. In particular, the adjustment source is not established to be a count, endpoint, address, index, or length.

### Preconditions

For a defined, normally returning invocation:

- The receiver-source access must be valid.
- The receiver used for dispatch must permit the initial load from its starting address.
- The resulting table-like base must permit access to the callable slot at byte offset 128.
- That slot must contain a callable target compatible with the actual invocation.
- Shared-value accesses and the negative-three adjustment must be valid under their actual declarations and runtime representation.
- Any additional preconditions imposed by the selected dependency must hold.

The wrapper contains no explicit checks establishing these conditions.

## Behavior

### Dispatch selection

The callable target is obtained through successive memory accesses:

1. Obtain an object-like address from the receiver source.
2. Read the integer-typed value at the beginning of the referenced object-like storage.
3. Use that value as the base for locating a callable slot at byte offset 128.
4. Read the callable target from that slot.

The offset is 128 bytes in the displayed address arithmetic. It must not be described as “entry 128.” Converting it into an entry index would require an established callable-slot width, which the supplied material does not provide.

The initial receiver load is represented using an integer type. The material does not independently document the target ABI or the reliability of the recovered types. A four-byte dispatch-header interpretation is consistent with a 32-bit integer model, but a portable layout specification cannot be derived from this expression alone.

The visible operation is indirect. No concrete dependency identity is established.

### Invocation arguments

The selected callable receives five arguments in this logical order:

| Position | Value supplied |
|---|---|
| First | An object-like address obtained from the receiver source, presented as an integer-pointer value |
| Second | The auxiliary shared value |
| Third | The adjustment-source expression adjusted by negative three |
| Fourth | The unadjusted adjustment-source expression |
| Fifth | The wrapper’s incoming 32-bit value |

Argument positions are established. Argument evaluation order is not.

The first argument is not shown as a saved copy of the receiver already used for dispatch. The receiver source appears separately in target selection and in construction of the first argument. Consequently, the contract must not require a single read or an atomic receiver snapshot.

Likewise, the third and fourth arguments reference the adjustment source separately. The wrapper does not visibly capture one shared value and derive both arguments from that captured value.

If the relevant shared state remains stable throughout evaluation, these separate references yield the corresponding stable receiver and adjustment values. If it changes during evaluation, the supplied material does not establish that the observations agree. Whether such changes constitute valid concurrent execution depends on declarations and synchronization rules that are unavailable.

### Negative-three adjustment

The third argument is formed by adding negative three to the adjustment-source expression. The fourth argument uses that expression without the adjustment.

The supplied material omits the adjustment source’s declaration. Therefore, the following are not established:

- Its width or signedness.
- Whether the operation is integer arithmetic or pointer arithmetic.
- Whether three denotes scalar units or pointed-to elements.
- The conversion rules used when passing the result to the dependency.
- The behavior at arithmetic boundaries.

It would be incorrect to specify unconditional 32-bit wraparound, saturation, clamping, or a three-byte address decrement.

Under an integer interpretation, the relationship is subtraction by three subject to the actual arithmetic type and conversions. Under a pointer interpretation, the displacement depends on the pointed-to type and the validity rules for pointer arithmetic.

### Completion

There is one visible indirect call expression and no visible loop or conditional branch.

Provided evaluation succeeds and the selected dependency returns normally:

- The wrapper invokes the dependency once.
- It does not inspect the dependency’s result.
- It performs no visible post-call operation other than returning.

There is no explicit retry, fallback target, status mapping, or recovery path.

## Side effects

### Directly visible effects

The wrapper visibly performs shared-state reads, receiver and dispatch-slot reads, and an indirect invocation. It contains no explicit assignment to shared state and no explicit write through the receiver.

The material does not establish an exact machine-level load count. Repeated source expressions, compiler optimization, volatile qualification, and aliasing information are insufficiently documented to prescribe such a count.

### Ordering

The target lookup has data dependencies: the callable slot cannot be located using the displayed scheme until the receiver and table-like base have been obtained.

The invocation receives the five arguments in their specified positions. Their positions do not establish left-to-right evaluation.

No single total order is established among independent shared reads used for target selection and argument construction. A conforming description must not invent an atomic snapshot or synchronization guarantee.

There is no explicit post-call read, write, release, or cleanup action.

### Dependency effects

The indirect dependency may have effects not visible in the wrapper. The supplied material does not establish that it:

- Preserves shared state.
- Preserves the receiver or dispatch table.
- Leaves referenced storage allocated.
- Refrains from retaining argument values.
- Avoids I/O, callbacks, or further dispatch.
- Returns normally.

The absence of direct writes in the wrapper is not evidence that the overall operation is read-only.

### Ownership

The wrapper shows no allocation, copying of an owned buffer, reference-count adjustment, or release operation.

No ownership-transfer rule is established for the receiver, the auxiliary value, the adjusted value, the unadjusted value, or the incoming 32-bit value. Passing a value is not sufficient evidence of either ownership transfer or borrowed lifetime.

## Dependencies

There is one unresolved indirect dependency. Its opaque function ID is unavailable in the supplied dependency map; none is invented here.

Its selection mechanism is established: the callable entry is read from a slot at byte offset 128 relative to the table-like base obtained through the receiver.

Its visible invocation shape is established: five arguments, with the incoming 32-bit value in the fifth position.

The following remain **documented_only**:

- Concrete target identity.
- Calling convention and complete parameter types.
- Return type.
- Application-level operation.
- Ownership and retention rules.
- Global-state effects.
- Error signaling and nonlocal control flow.
- Validity requirements beyond those implied by the visible accesses.

No direct helper call is shown.

## Acceptance examples

These examples are conditional contract checks, not reports of execution.

### Stable integer-valued adjustment source

Assume valid receiver and dispatch storage, stable shared inputs, an integer adjustment source equal to ten, and arithmetic in which subtracting three is defined.

Expected invocation:

- First argument: the stable receiver value.
- Second argument: the auxiliary value.
- Third argument: seven.
- Fourth argument: ten.
- Fifth argument: the incoming 32-bit value.

If the dependency returns normally, the wrapper returns without exposing its result.

This example does not establish that the actual adjustment source is integer-valued.

### Incoming value containing a high-order bit

Assume valid dispatch state and an incoming 32-bit pattern with its highest bit set.

The wrapper does not visibly reject the value, reinterpret it through arithmetic, or clear that bit. It supplies the incoming value as the fifth argument. Whether the dependency treats it as signed, unsigned, flags, or another representation is unspecified.

### Dependency mutates shared state

Assume the dependency modifies the receiver source or other shared storage, then returns normally.

The wrapper contains no visible restoration step. Acceptance must not require the pre-call shared state to be restored.

### Dependency produces a result

Assume the dependency returns a status or other result through its own interface.

The wrapper does not branch on, transform, or explicitly return that result.

### Receiver source changes during evaluation

Assume the execution environment permits the receiver source to change between its separate evaluations.

The contract does not require the receiver used for target lookup to equal the first invocation argument. A specification requiring that equality would need additional stability or synchronization evidence.

### Adjustment near a boundary

Assume the adjustment source is near a numeric or address boundary.

No expected adjusted value can be specified without its actual type and validity rules. A test that demands wraparound, clamping, or rejection would add behavior not established here.

### Invalid dispatch storage

Assume the receiver or callable slot is invalid.

The wrapper provides no visible validation or recoverable error result. No particular fault, diagnostic, exception, or termination behavior is established.

## Uncertainties

The principal **documented_only** uncertainties are:

- Shared-storage declarations, including volatile or atomic qualification.
- ABI, address width, callable-pointer width, and calling convention.
- Reliability of the recovered integer and pointer types.
- Adjustment-source type, arithmetic width, and overflow behavior.
- Relative ordering and exact count of independent reads.
- Concrete dependency identity and full contract.
- Ownership, lifetimes, synchronization, and reentrancy.
- Failure behavior for invalid memory or incompatible callable entries.

**Inferred intent:** the receiver-based lookup resembles interface or virtual-method dispatch, and the adjusted/unadjusted argument pair may express related positions or bounds. Neither interpretation establishes application semantics.

## Evidence

**Established static evidence:** the supplied function contains one indirect invocation using a callable slot at byte offset 128 from a receiver-derived base. It supplies five arguments, including an adjusted/unadjusted shared-expression pair and the incoming four-byte value. It then returns without an explicit result.

**Not established:** runtime target identity, application purpose, concrete memory contents, dependency effects, ownership rules, or behavior under invalid preconditions.

No runtime tests have been performed. The acceptance examples are conditional consequences of the visible structure, not observed outcomes.
