<!-- Trusted provenance: F-4fac58ca213e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4fac58ca213e

## Purpose

F-4fac58ca213e conditionally dispatches an object-associated operation. A signed comparison determines whether dispatch is permitted; the sign of the request then determines which operation is selected.

The positive-request case can produce a single-byte zero write. Its destination depends on an arithmetic input obtained from a dependency, a later observation of an object field, and the result of an indirect operation. The zero-or-negative case selects a different indirect operation and has no direct data write in the supplied representation.

This contract describes observable calls, memory accesses, arithmetic, and conditional effects. It does not assign application meanings such as truncation, clearing, allocation, or destruction to those operations.

**Evidence classification:** Branch conditions, argument relationships, storage offsets, and the conditional byte write are established by the supplied static representation. Application intent, dependency behavior, and exceptional execution remain unknown where identified below. No runtime tests have been performed.

## Interface

### Inputs

The function accepts an object address and a signed integer request.

The represented integer model is 32 bits. Object fields used as integer values, dependency results used in comparisons and arithmetic, and the positive operation’s result are represented at that width. The latter result also serves as an address when nonzero.

The relevant storage layout is:

| Location | Represented width | Use |
|---|---:|---|
| Object at byte offset 0 | 32 bits | Reference used to locate an operation table |
| Object at byte offset 4 | 32 bits | Input to the comparison dependency; later input to positive-path arithmetic |
| Operation table at byte offset 8 | Address-sized callable entry in the represented 32-bit model | Positive-request operation |
| Operation table at byte offset 12 | Address-sized callable entry in the represented 32-bit model | Zero-or-negative-request operation |

These are accessed portions, not a complete object or table definition. No claim is made that the object contains only two fields or that the table contains only these entries.

The supplied representation does not independently establish the complete ABI, byte order, alignment requirements, or integer-to-address conversion rules.

### Outputs

There is no explicit return value or status result.

Caller-visible behavior can include:

- Invocation of the comparison dependency.
- Conditional invocation of an arithmetic dependency and one selected indirect operation.
- Effects produced by those dependencies.
- A conditional, direct write of one zero-valued byte.

An ordinary return does not distinguish skipped dispatch, completed dispatch, or suppression of the direct byte write.

### Valid-execution requirements

No explicit pointer, target, range, or ownership validation is present.

The object field at byte offset 4 must be readable for the initial dependency argument. Later requirements depend on the selected behavior: dispatch requires a usable table reference and callable entry; positive-path arithmetic requires another readable observation of that field; a nonzero positive-operation result requires a writable calculated destination.

These requirements apply when the corresponding access occurs. Dependencies are not known to preserve the object, table, or returned storage. Initial validity alone is therefore insufficient.

A null object is not recognized as a special case. Arithmetic and addressing must also be valid under the applicable execution model; boundary behavior is not resolved by the supplied representation.

## Behavior

### Dispatch eligibility

F-d8cbf9aa79a9 receives the value initially observed at object byte offset 4. Its signed result, rather than the field value itself, controls eligibility.

| Relationship between comparison result and request | Request sign | Local outcome |
|---|---|---|
| Result is less than or equal to request | Any | No further dependency or operation is selected |
| Result is greater than request | Positive | Positive-request behavior |
| Result is greater than request | Zero or negative | Zero-or-negative-request behavior |

Equality suppresses further action. The request’s sign is relevant only after the comparison permits dispatch.

Both operands use signed ordering. A negative comparison result can therefore permit dispatch when the request is still more negative. Conversely, a zero or negative request does not inherently select the corresponding operation.

The comparison dependency executes even when no later action is selected. That outcome does not imply an absence of memory changes, global changes, or other effects from the dependency.

### Positive-request behavior

For an eligible positive request, F-798b2f89ac14 receives the object address and the unchanged request.

The adjustment supplied to the indirect operation is one greater than the signed difference between:

- The arithmetic result returned by F-798b2f89ac14.
- The object field at byte offset 4 as observed after that dependency returns.

The field’s earlier value does not substitute for this later observation. The represented calculation forms the difference before increasing it by one, which matters if an intermediate value exceeds the supported signed range.

F-U1 denotes the indirect operation selected through the table entry at byte offset 8. Its arguments are the object address, the adjustment, and the integer constant 1. No semantic interpretation of that constant is established.

F-U1’s result controls the direct write:

- A zero result suppresses the write.
- A nonzero result supplies an address base. The written byte is displaced from that base by one less than the adjustment.

Exactly one byte is directly written, with value zero. No prior read of that destination byte is required by the visible operation.

The adjustment is not range-checked. A positive request does not guarantee a positive adjustment, a destination at or after the returned base, or a destination within any particular object.

### Zero-or-negative-request behavior

For an eligible request of zero or less, F-U2 denotes the operation selected through the table entry at byte offset 12. Its sole visible argument is the object address.

No arithmetic dependency is invoked on this path, and no direct byte write is present. No result from F-U2 is consumed by the visible body.

The supplied representation explicitly reports incomplete recovery of this final indirect transfer. The selected entry and object argument are supported, but whether the underlying mechanism is an ordinary returning call or a tail-style transfer remains **documented_only**. A particular stack or return mechanism is not an acceptance requirement.

### Observation timing and mutation

The following ordering constraints affect externally distinguishable behavior:

| Observation or effect | Required relationship |
|---|---|
| Initial field value supplied to F-d8cbf9aa79a9 | Observed before that dependency executes |
| Field value used in the adjustment | Observed after F-798b2f89ac14 returns |
| Table reference and selected operation | Resolved after the dependencies preceding that dispatch |
| Adjustment passed to F-U1 | Established before F-U1 executes |
| Conditional byte write | Occurs after F-U1 returns a nonzero result |

These constraints allow dependency effects to influence later behavior. In particular, a dependency can change the field used for arithmetic or replace the table reference, provided later accesses remain valid.

There is no visible field reload after F-U1. A field change made by that operation therefore does not revise the adjustment or the resulting write displacement.

F-d8cbf9aa79a9 receives a scalar argument, but that does not establish isolation from the object. It may have another reference to relevant storage or may affect global state. No preservation guarantee is inferred from its argument list.

### Arithmetic and destination boundaries

The comparison and adjustment use the represented signed 32-bit quantities. No explicit overflow, underflow, bounds, or storage-extent check is present.

For valid arithmetic and addressing:

| Adjustment | Destination relative to a nonzero returned base |
|---:|---|
| 1 | The base address |
| 0 | One byte before the base |
| −1 | Two bytes before the base |
| Greater than 1 | After the base |

The represented destination arithmetic combines the returned base with the adjustment before applying the final decrease of one. A mathematically valid final destination does not establish that every represented intermediate calculation is valid.

The evidence does not determine whether out-of-range arithmetic wraps, traps, or has another execution-model consequence. Such cases remain **documented_only**.

Only the returned base is checked against zero. The final destination receives no separate zero, validity, or bounds check. A nonzero error sentinel, if a dependency can return one, is not specially recognized.

## Side effects

### Direct effects

The sole explicit data modification is the conditional one-byte zero write on the positive path. There is no explicit assignment to either exposed object field.

This does not establish that object or table storage remains unchanged. The calculated destination is not shown to be disjoint from either, and dependency effects are unrestricted by the supplied body.

No scan, bulk copy, bulk clearing, retry, or compensating write is visible.

### Ownership and lifetime

No explicit allocation, release, reference-count update, or ownership transfer is present.

The positive operation’s nonzero result is used as an address base. That use does not establish whether it denotes newly allocated storage, an interior address, borrowed memory, or another address-like value. The function does not explicitly release the result or return it to its caller.

No dependency is established to preserve:

- Object lifetime or field values.
- Table lifetime, identity, or contents.
- Global state.
- Accessibility or lifetime of returned storage.

If a dependency invalidates storage required by a later access, there is no visible local recovery.

### Error and abnormal-completion behavior

There is no explicit error-reporting channel, rollback, exception handler, or retry mechanism.

A zero F-U1 result suppresses only the direct write. It does not reverse dependency effects or produce a separately reported failure.

Nonreturning dependencies, exceptions, invalid callable targets, invalid memory, and unusable nonzero results have no visible local handling. Their runtime consequences remain **documented_only**.

## Dependencies

F-U1 and F-U2 are document-local opaque identifiers for indirect dispatch roles, not recovered implementation identities. Different invocations may select different targets.

| Identifier | Inputs supplied | Result consumed |
|---|---|---|
| F-d8cbf9aa79a9 | Initial value of the object field at byte offset 4 | Signed result for the eligibility comparison |
| F-798b2f89ac14 | Object address and unchanged eligible positive request | Signed input to adjustment arithmetic |
| F-U1 | Object address, established adjustment, integer 1 | Zero/nonzero decision and nonzero address base |
| F-U2 | Object address | No result consumed |

No dependency implementation or independent dependency contract is supplied. Argument and result shapes do not establish string handling, allocation, release, destruction, or preservation of globals.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Except where an uncertainty is being illustrated, assume normal dependency returns, usable storage and targets, and valid intermediate arithmetic.

### Equality suppresses dispatch

With request 7 and comparison result 7, neither F-798b2f89ac14 nor either indirect operation is selected. F-d8cbf9aa79a9 has nevertheless executed, and its effects are not undone.

### Zero is not an unconditional operation request

With request 0, comparison results 0 and −1 suppress dispatch. A comparison result of 1 selects F-U2.

Always invoking F-U2 for a zero request would violate the comparison requirement.

### Negative comparisons remain signed

With request −5, comparison result −3 selects F-U2. Results −5 and −6 suppress dispatch.

Requiring a nonnegative comparison result would incorrectly exclude the first case.

### Ordinary positive adjustment

Suppose the request is 3, the comparison result is 9, the arithmetic dependency returns 4099, and the later field observation is 4096.

F-U1 receives adjustment 4 and the constant 1, together with the object address. If it returns address 8192, the direct write places zero in the byte at address 8195.

The function itself performs no additional neighboring-byte write. Dependency effects on neighboring storage remain unspecified.

### Mutation changes the arithmetic input

Under the same request and comparison conditions, suppose F-798b2f89ac14 changes the relevant field to 4098 while returning 4099.

The required adjustment is 2, not 4. If it also replaces the table reference with another usable table, selection of F-U1 uses the replacement.

### Mutation after adjustment does not revise it

Suppose F-U1 receives adjustment 4, changes the object field, and returns address 8192.

The direct write still targets address 8195. Recomputing the adjustment from the changed field would introduce an observation absent from the supplied behavior.

### Zero result prevents only the direct write

If F-U1 returns zero, no direct byte write follows. Earlier changes remain permitted. No rollback, retry, or explicit failure status is required.

### Nonpositive adjustments are not rejected

An arithmetic result of 4095 and later field value of 4096 produce adjustment 0. A nonzero returned base leads to a write one byte before that base, provided addressing is valid.

An arithmetic result of 4094 with the same field value produces adjustment −1 and a destination two bytes before the base. Neither adjustment is rejected locally.

### Arithmetic boundaries are unresolved

If the subtraction, increase by one, or destination arithmetic exceeds the applicable representation, no particular wrapped value, successful write, or fault is required by this contract. The runtime rule remains **documented_only**.

## Uncertainties

The following remain **documented_only**:

- Application meanings and units of the request, exposed field, comparison result, and adjustment.
- Complete object layout, ABI, byte order, alignment, and address-conversion rules.
- Whether the table corresponds to language-level virtual dispatch or another mechanism.
- Actual indirect target identities and dependency contracts.
- Dependency effects on global state, ownership, fields, and lifetimes.
- The nature and extent of storage associated with a nonzero F-U1 result.
- Whether zero signifies failure, absence, or another condition.
- Whether nonzero error sentinels are possible.
- Exact transfer and return mechanics for F-U2.
- Runtime consequences of invalid storage, invalid targets, arithmetic overflow, and abnormal dependency completion.

A boundary-marker interpretation of the zero write is plausible **inferred intent**, not established semantics. No buffer, string, resizing, or destruction contract follows from it.

## Evidence

The supplied static representation supports the signed eligibility comparison, sign-dependent dispatch, field and table offsets, dependency arguments, post-dependency field observation, adjustment relationship, positive-operation result check, and conditional single-byte zero write.

The supplied opaque mapping identifies the two direct dependencies. It does not provide their implementations or behavioral guarantees.

The incomplete recovery notice associated with the final indirect transfer limits conclusions about F-U2’s transfer mechanics. It does not supply missing ownership, preservation, or error semantics.

The prior draft and reviewer feedback are not independent behavioral evidence. Established observations, inferred intent, and unresolved behavior are distinguished above.

No runtime tests have been performed.
