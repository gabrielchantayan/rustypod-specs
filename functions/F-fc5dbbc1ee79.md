<!-- Trusted provenance: F-fc5dbbc1ee79; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fc5dbbc1ee79

## Purpose

F-fc5dbbc1ee79 conditionally substitutes a stored integer for its supplied integer. An externally referenced control value determines whether substitution occurs.

When the observed control value equals zero, the function returns the supplied integer unchanged. Otherwise, it assigns zero through the external reference and returns an integer from an input-derived location. The zero assignment precedes the retrieval.

**Established static evidence:** conditional substitution, the zero assignment, the displacement of 1068, and assignment-before-retrieval ordering.

**Inferred intent:** the control state may enable a single substitution before subsequent calls revert to unchanged returns. This interpretation requires stable control-object identity, no intervening re-enablement, and valid accesses. It is not an exactly-once guarantee or a synchronization protocol.

The function does not establish that an arbitrary input identifies valid storage.

## Interface

### Input and output

The function accepts one signed integer by value and returns a signed integer of the same declared type. The retrieved value also has that type.

The supplied evidence does not establish the integer width, storage size, alignment, representation, address width, or calling convention. A 32-bit argument, 32-bit return value, and four-byte retrieval are therefore not unconditional requirements.

The input has mode-dependent significance:

- With substitution disabled, it is only the value to return.
- With substitution enabled, it is the integer basis for the retrieval address.

There is no separate status channel. Zero and negative results are not designated failures. A retrieved value equal to the input is also an ordinary result; return-value equality cannot identify the selected mode.

### External control state

The function accesses control storage through an external reference. Neither the reference declaration nor the target declaration is supplied.

The control operation distinguishes equality with zero from inequality with zero. All values comparing unequal to zero select substitution; their other properties do not distinguish behavior.

The target’s width remains unknown. In particular, the zero comparison and assignment do not establish a one-byte control object or a one-byte write.

The reference is evaluated for selection and separately for assignment. No captured target identity connects those evaluations. With a stable reference, both accesses concern the same object. If the environment permits its target to change between the accesses, the assignment concerns the later target. The evidence does not establish that concurrent changes are legal or synchronized.

### Retrieval layout

The retrieval address is obtained by adding 1068 to the input as a signed-integer calculation, then interpreting the sum as an address for the result type.

The displacement is not scaled by the result type’s storage size. Describing it as a byte offset additionally requires a platform where consecutive integer address values denote consecutive bytes.

Under byte-addressing and four-byte-result assumptions, retrieval covers offsets 1068 through 1071 relative to the address represented by the input. Those assumptions are not established by the supplied body. No record boundary, field identity, enclosing allocation size, or ownership relationship follows from the displacement alone.

### Preconditions for defined completion

Both modes require a valid selection access through the external reference.

Substitution additionally requires:

- A valid reference evaluation and writable target for the zero assignment.
- A representable signed sum of the input and 1068.
- A suitable integer-to-address conversion.
- Result storage with sufficient extent, accessibility, alignment, lifetime, and compatible object-access semantics.

These are requirements on the caller and environment, not validations performed by this function.

Unchanged return does not require the input to represent an address or satisfy the retrieval arithmetic.

## Behavior

### Mode-dependent outcome

| Control value observed for selection | Result on defined completion | Explicit mutation |
|---|---|---|
| Equal to zero | Input unchanged | None |
| Unequal to zero | Integer from the displaced location | Zero assigned through the later reference evaluation |

The selection observation determines the mode for the call. There is no second control-value test.

Consequently, a control-state change after selection does not cancel an already selected substitution. Conversely, a call that selected unchanged return does not begin retrieval because the control state later becomes nonzero.

The unchanged-return path performs neither the displacement calculation nor the input-derived storage read. Even an input that would be unusable for retrieval can be returned normally on that path.

### Clearing and reference identity

Substitution includes a zero assignment regardless of the assignment target’s existing value. If the later target is already zero, the assignment is still part of the selected behavior.

With stable reference identity, the observed nonzero control object is cleared. Without that stability, the originally observed object need not change: a different target can receive zero.

The contract therefore guarantees an assignment through the reference used at assignment time, not unconditional clearing of the object used for selection.

### Arithmetic boundaries

Retrieval uses signed addition before address conversion. Its upper arithmetic boundary is the maximum representable input value minus 1068. Inputs above that boundary do not have a specified wrapped-address result.

Because the displacement is positive, this addition introduces no lower-bound overflow for a representable input. Negative inputs nevertheless require an independently valid address interpretation and valid result storage.

Representable arithmetic alone is insufficient for successful retrieval. The resulting address can still be inaccessible, misaligned, outside an object’s lifetime, or unsuitable for access as the result type.

None of these retrieval-specific restrictions applies when the function returns the input unchanged.

### Access ordering and overlap

The relevant ordering is:

1. Control observation used for selection.
2. Zero assignment, if substitution was selected.
3. Result-storage read.

This describes semantic ordering, not exact machine-instruction counts or cross-thread visibility guarantees.

The retrieved value comes from the storage state after the zero assignment, subject to the applicable memory rules. Overlap can therefore affect the result. The relevant overlap is with the assignment target, which is not necessarily the initially observed control object.

Overlap does not automatically imply a zero result. Its effect depends on the assignment width, result width, storage representations, byte order, and legality of the accesses. A permitted narrow assignment may change only part of the retrieved representation.

The evidence neither guarantees non-overlap among reference storage, control storage, and result storage nor establishes that overlap occurs.

### Return and repeated calls

The retrieved integer is returned without an explicit adjustment, mask, normalization, or status conversion.

A completed substitution can cause a later call to return its input unchanged when that later call observes the same control object still equal to zero. Reference redirection or subsequent writes can prevent this transition.

There is no atomic test-and-clear guarantee. Multiple callers are not guaranteed that only one selects substitution.

### Errors and abnormal completion

No explicit failure result, retry, fallback, or restoration operation is present.

An invalid selection access prevents defined mode selection. An invalid assignment, overflowing signed addition, unsuitable converted address, or invalid result access prevents defined completion of substitution.

Assignment-before-retrieval ordering does not establish that the zero assignment remains observable after every failure. Undefined execution supplies no such guarantee. If the environment instead defines a fault or other abnormal termination, that environment determines handling and surviving effects.

In particular, failed retrieval does not promise return of the original input or restoration of the previous control state.

## Side effects

The only explicit mutation is the zero assignment on the substitution path.

The unchanged-return path observes the external reference and its target but performs no explicit write. The substitution path additionally evaluates the reference for assignment, writes zero through it, and reads the input-derived result storage.

These descriptions do not establish exact hardware access counts. Missing declarations also leave ordinary, volatile, atomic, and device-memory access properties unresolved. Any special consequences of accessing such storage remain environment-dependent.

There is no explicit allocation, release, ownership transfer, or function invocation. Retrieving an integer does not transfer ownership of the containing storage. Clearing control storage does not establish ownership of that storage.

The storage needed for each performed access must remain usable for that access. Responsibility for maintaining its lifetime and coordinating competing accesses is not identified.

## Dependencies

There are no explicit called-function dependencies, so no additional opaque function IDs are required.

The contract depends on environmental facts not supplied with the body:

- External-reference and control-object declarations.
- Reference identity and permitted changes to it.
- Signed-integer range, representation, size, and alignment.
- Integer-to-address conversion and address-unit mapping.
- Storage lifetime, extent, accessibility, and object-access compatibility.
- Legal overlap and its representation effects.
- Access qualifiers, synchronization, and abnormal-completion rules.

No behavior is attributed to an unidentified initializer, owner, or other consumer of the external state. No unknown callee is assumed to preserve globals or manage memory.

## Acceptance examples

These examples are conditional consequences of static evidence, not observed executions. Unless stated otherwise, assume valid ordinary-memory accesses, stable reference identity, permitted object access, and no interference.

### Unchanged ordinary input

With control equal to zero and input 73, the result is 73. There is no zero assignment and no input-derived result read.

### Unchanged input beyond the retrieval boundary

With control equal to zero and input equal to the maximum representable signed value, the result is that value. The overflowing displacement calculation that substitution would require is not performed.

Likewise, returning input zero does not require accessible result storage at an input-derived address.

### Independent retrieval storage

With control unequal to zero, suppose the displaced location validly contains 205 and does not overlap the assignment target. The target receives zero before retrieval, and the result is 205.

A stored zero or negative value is returned on the same basis; neither is treated as an error.

### Different selection and assignment targets

Suppose the environment permits the reference to identify one object during selection and another during assignment.

If the first object compares unequal to zero, substitution remains selected. The second receives zero even if it was already zero. The first object need not be changed.

This example does not establish that an unsynchronized concurrent redirection is permitted.

### Conditional control width

If additional declaration evidence establishes an unsigned eight-bit control object, values 1 and 255 both select substitution. The zero assignment then follows that object’s declared width and semantics.

Without that evidence, neither representability of 255 nor a one-byte write is a requirement.

### Conditional partial overlap

Assume eight-bit bytes, a four-byte little-endian result, and a permitted one-byte zero assignment overlapping the result’s lowest-addressed byte.

If the result initially has representation 0x12345678, the later retrieval observes 0x12345600. This depends on every stated layout and access assumption; it does not establish those assumptions.

### Later unchanged return

If a substitution clears the same control object used for selection, and a later call observes that object still zero, the later call returns its own input unchanged.

This is a conditional state transition, not an exactly-once protocol.

### Invalid substitution conditions

With substitution selected, input above the maximum signed value minus 1068 has no specified wrapping behavior.

With unusable result storage, no fallback result or restoration is guaranteed. Visibility of the preceding assignment after abnormal completion depends on the applicable execution rules.

## Uncertainties

The following remain **documented_only**:

- Exact integer and address widths, representations, alignment, and calling convention.
- External-reference type, target type, and access widths.
- Reference stability and lawful mechanisms for changing its target.
- Integer-to-address mapping and the physical unit represented by the displacement.
- Byte order and object-access restrictions.
- Actual overlap among relevant storage regions.
- Ordinary, volatile, atomic, or device-memory properties.
- Synchronization, fault handling, and surviving effects after abnormal completion.
- Ownership, lifetime management, and application-level meaning.

The possible single-substitution interpretation remains inferred intent. It does not resolve these uncertainties.

## Evidence

The supplied static body establishes a signed-integer input and result, a control comparison with zero through an external reference, and unchanged return when that comparison succeeds.

For substitution, it establishes zero assignment through a separate reference evaluation before retrieval at an address derived from the input plus 1068. Addition precedes address conversion, and the displacement is not scaled by the result type’s size.

No explicit calls, loops, validation, retry, error-status mechanism, restoration, allocation, or release appear.

Missing declarations prevent fixing control widths and access properties. Missing platform evidence prevents fixing physical layout and address-conversion behavior.

No runtime tests have been performed. The acceptance examples describe conditional requirements derived from static evidence, not measured outcomes.
