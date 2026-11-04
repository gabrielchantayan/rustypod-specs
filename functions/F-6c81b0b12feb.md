<!-- Trusted provenance: F-6c81b0b12feb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6c81b0b12feb

## Purpose

F-6c81b0b12feb conditionally invokes F-5990a26eaaa3, then performs indirect dispatch through a globally selected table arrangement.

For a nonzero object address, the established sequence is:

1. Read an object field and invoke F-5990a26eaaa3 if that field is nonzero.
2. Evaluate the global table-base value and retain that first value.
3. Evaluate the global table-base value again and use this second value to read an optional target at offset +0x30.
4. Invoke that optional target if nonzero.
5. Read and invoke a required target at offset +0x18 from the retained first base.
6. Read an optional target at offset +0x30 from the retained first base and invoke it if nonzero.

The two global evaluations are distinct. Their results are not established to be equal. Consequently, the initial optional target can come from a different table than the required target and final optional target.

The apparent intent is an operation surrounded by optional notifications. This is an inference only. The representation does not establish notification semantics, paired callbacks, cleanup, locking, or resource release.

## Interface

### Inputs

Four explicit inputs are exposed:

| Input | Representation and observed use |
|---|---|
| First | Integer-valued object address. Zero selects an immediate return. Otherwise, it locates an object field and is supplied to the first optional target and required target. |
| Second | Opaque four-byte value, with no direct use shown. |
| Third | Opaque four-byte value, with no direct use shown. |
| Fourth | Opaque four-byte value supplied to the first optional target, if invoked. |

The first optional invocation additionally includes a value from unresolved ambient machine state. Its presence does not establish a fifth explicit input to F-6c81b0b12feb.

The three opaque explicit inputs are represented as four-byte quantities, nominally 32 bits. The first input, object field, and retained table-base value have an integer representation, nominally 32 bits in the apparent model. A verified target ABI is not supplied; the exact integer width, pointer width, byte order, and binary calling conventions remain documented_only. In particular, pointer-slot storage width cannot be established solely from the offsets.

### Referenced layout

Offsets below are byte offsets, not element indices.

| Region | Offset | Directly observed use |
|---|---:|---|
| Object addressed by the first input | +8 | Read an integer field and test it for zero |
| Table addressed by the **second** global evaluation | +0x30, or 48 | Read the initial optional target |
| Table addressed by the **retained first** global evaluation | +0x18, or 24 | Read the required target |
| Table addressed by the **retained first** global evaluation | +0x30, or 48 | Read the final optional target |

The same global storage supplies both table-base evaluations. These are separate evaluations of that storage, not two explicit table arguments.

If both evaluations yield the same base, the two optional accesses address the same slot at different times. Otherwise, they address slots in different tables. No complete table size, object size, padding arrangement, or other field layout is established.

### Output

No result value is specified. Dependency return values are not used.

The observable result consists of the reads, dependency invocations, and effects of those dependencies. There is no established success indicator, failure indicator, or output buffer.

### Preconditions for normal progression

A zero first input does not require object or table storage to be accessible.

For a nonzero first input, normal progression requires:

- A readable integer field at object offset +8.
- Readable global table-base storage at both evaluations.
- A readable initial optional slot through the second evaluated base.
- Readable required and final optional slots through the retained first base when reached.
- A callable required target and callable nonzero optional targets.
- Normal return from each dependency whose successor steps are to occur.

These requirements are not validated by the function. A nonzero address need not be valid, aligned, live, or callable. Neither table-base value is checked for zero.

## Behavior

### Zero first input

If the first input is zero, the function returns immediately.

No object-field read, global table-base evaluation, table-slot read, or dependency invocation is shown on this path. The remaining explicit inputs do not affect the decision.

### Object-field condition

For a nonzero first input, the function first reads the integer field at object offset +8.

A zero field skips F-5990a26eaaa3. Any nonzero field invokes F-5990a26eaaa3 before either global table-base evaluation.

This is a zero-versus-nonzero condition, not a positive-versus-negative condition. No arithmetic involving the field value is shown.

No explicit arguments are shown for F-5990a26eaaa3, and its result is unused. This does not establish that it lacks ambient inputs or global effects.

The field is not read again. Subsequent changes to it do not alter this invocation’s completed branch decision.

### Two global table-base evaluations

After F-5990a26eaaa3 returns, or immediately after that call is skipped, the function evaluates the global table-base value and retains it.

It then evaluates the same global again while locating the initial optional slot at +0x30. This second value is used for that initial slot access; the retained first value is not substituted for it.

The established order is therefore:

- First global evaluation and retention.
- Second global evaluation.
- Initial optional-slot read through the second value.

There is no intervening explicit dependency invocation between these global evaluations. Nevertheless, the representation supplies no stability or synchronization guarantee permitting them to be collapsed into one value.

If F-5990a26eaaa3 modifies the global and returns, both evaluations occur after that modification. Its effects do not establish that the evaluations must agree with each other.

After the second evaluation, no further evaluation of this global is shown. The required and final optional slot accesses use the retained first base even if a subsequent dependency replaces the global value.

These are statements about the supplied representation’s evaluation order, not a verified count of physical machine loads under an independently established memory model.

### Initial optional invocation

The function reads the target at offset +0x30 through the second evaluated global base.

If the selected pointer is zero, the initial optional invocation is skipped. Otherwise, the represented argument sequence is:

1. Original object address.
2. Zero.
3. The selected optional target’s own pointer value.
4. Original fourth explicit input.
5. An unresolved ambient value.

The third argument is the target selected by this slot read. It is not obtained from another slot read.

This describes the displayed invocation, not a verified high-level function prototype. The ambient value’s initialization, validity, width, meaning, and calling-convention role are unresolved.

The target’s result is ignored. If it returns normally, execution proceeds to required dispatch.

### Required invocation

The function reads the required target at offset +0x18 through the retained first base and invokes it with the original object address.

This slot read occurs after the initial optional invocation returns, if that invocation occurred. Therefore, modifications that invocation makes to the required slot in the retained table can affect the selected required target.

If the initial optional invocation came from a different table, that does not change the required slot’s base. The relevant question is which storage the dependency modified, not which table supplied its target.

The required pointer has no zero guard. “Required” means dispatch is attempted whenever execution reaches this stage, not that validity is guaranteed.

Only the object-address argument is explicitly shown. No additional argument contract is established from possible ambient machine state. The return value is unused.

### Final optional invocation

After the required target returns, the function reads the target at offset +0x30 through the retained first base.

If this target is zero, the function returns. Otherwise, the represented invocation supplies zero followed by one. No object address is explicitly supplied.

This selection is independent of the initial optional selection:

- With equal base values, it is a later read of the same optional slot.
- With unequal base values, it is a read of a different optional slot.
- In either case, dependencies may have changed the selected slot before this read.

A final optional invocation can occur when the initial one was skipped. An initial optional invocation does not guarantee a final one or an identical target.

After the final invocation returns normally, the function returns. The exact low-level transfer mechanism remains unresolved; no tail-transfer guarantee is established.

### Complete ordering and arithmetic boundaries

The direct evaluation and invocation order is:

1. Read the object field at +8.
2. Conditionally invoke F-5990a26eaaa3.
3. Evaluate and retain the global table base.
4. Evaluate the global table base again.
5. Read the initial optional slot at +0x30 through the second value.
6. Conditionally invoke that target.
7. Read the required slot at +0x18 through the retained first value.
8. Invoke the required target.
9. Read the final optional slot at +0x30 through the retained first value.
10. Conditionally invoke that target.

Each step following a call depends on that call returning normally.

No bounds, alignment, address-overflow, or lifetime checks are shown. The additions use fixed offsets of 8, 24, and 48 bytes; there is no size multiplication or length-controlled traversal. Exact overflow behavior cannot be specified without a verified integer and address model. Invalid address formation or access has no established recovery behavior.

## Side effects

No direct write to the object, global table-base storage, or table entries is shown. No direct allocation, release, reference-count modification, or ownership transfer is shown.

These observations do not make the overall operation read-only. F-5990a26eaaa3 and every selected indirect target may modify global state, either table, the object, or unrelated storage.

Ownership and lifetime are not established:

- Passing the object address does not establish transfer, retention, or preservation.
- Retaining a table-base value does not acquire ownership of its storage.
- Normal return from a dependency does not establish that it preserved global values or memory.
- Neither the first nor second global evaluation establishes a lifetime guarantee for the referenced table.

If an initial optional target invalidates the retained table, subsequent reads still address that retained base. No refresh or recovery is shown.

There is no explicit error handling, rollback, retry, compensating invocation, or cleanup path. A dependency that fails to return normally prevents any guarantee about subsequent steps.

No synchronization or atomicity contract is established. The distinct global evaluations permit documenting different resulting bases conditionally; they do not establish that any particular concurrent mutation is supported or well-defined.

## Dependencies

### F-5990a26eaaa3

Invoked only when the initially read object field is nonzero. It precedes both global table-base evaluations.

No explicit arguments or used result are shown. Its purpose, effects, failure behavior, memory ownership, and preservation of state remain unknown.

### Unresolved indirect dependencies

No opaque function identities are supplied for the slot-selected targets. Their dependency roles are established without inventing stable identities:

| Role | Selection location | Invocation condition | Represented arguments |
|---|---|---|---|
| Initial optional target | +0x30 through the second global value | Selected target is nonzero | Object address; zero; selected target pointer; fourth input; unresolved ambient value |
| Required target | +0x18 through the retained first global value | Execution reaches this stage | Object address |
| Final optional target | +0x30 through the retained first global value | Selected target is nonzero | Zero; one |

Sharing an offset does not establish shared storage. Sharing storage, when the bases agree, does not establish a stable target. Neither case establishes a common verified prototype for the two optional invocations.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless otherwise stated, all reached storage and targets are valid and dependencies return normally.

1. **Zero object address.**  
   The function returns without evaluating the global or invoking dependencies.

2. **Zero field, equal bases, absent optional targets.**  
   F-5990a26eaaa3 is skipped. Both global evaluations occur. If the optional slot is zero at both accesses, only the required target is invoked, with the object address.

3. **Nonzero field.**  
   F-5990a26eaaa3 runs before either global evaluation. Any nonzero field value selects this branch.

4. **Direct dependency replaces the global.**  
   If F-5990a26eaaa3 installs a valid replacement table and the global remains stable across both evaluations, that table supplies all three slot accesses. Stability is an explicit condition, not a guarantee.

5. **The two evaluations produce different bases.**  
   If the retained first base designates table A and the second designates table B, the initial optional target comes from B at +0x30. The required and final optional targets come from A at +0x18 and +0x30, respectively.

6. **Different tables have different optional availability.**  
   Under the preceding conditions, a nonzero optional target in B and a zero optional target in A produce an initial optional invocation but no final one, provided those slots retain the relevant values until accessed. The reverse availability produces only the final optional invocation.

7. **Initial optional target changes the retained table’s required slot.**  
   If that target installs another valid target at retained-base offset +0x18, the subsequent required-slot read selects the replacement, regardless of which table supplied the initial optional target.

8. **Required target installs a final optional target.**  
   Even if the initial optional selection was zero, installing a nonzero target at retained-base offset +0x30 causes the final invocation with zero and one.

9. **A dependency changes only the global after initial selection.**  
   If the retained table remains valid, subsequent slot reads still use it. No third global evaluation is shown.

10. **Required slot is zero or a dependency does not return.**  
    A zero required target is not gracefully skipped. Abnormal termination by any dependency establishes neither subsequent dispatch nor compensating action.

## Uncertainties

The following remain documented_only:

- Verified ABI, exact integer and pointer widths, byte order, alignment, and call prototypes.
- Object semantics and meaning of its +8 field.
- Purpose and effects of F-5990a26eaaa3.
- Global storage properties and conditions under which its two evaluations can differ.
- Table initialization, bounds, lifetime, synchronization, and ownership.
- Indirect target identities and their state-preservation behavior.
- Meaning of the optional argument sequences and ambient argument.
- Final transfer mechanism.
- Behavior under invalid addresses, arithmetic overflow, reentrancy, faults, and abnormal termination.

The inferred bracketing intent must not be promoted into guaranteed pairing, cleanup, destruction, locking, or memory release.

## Evidence

This contract uses only the supplied static representation and opaque dependency mapping. No runtime tests, implementation inspection beyond that representation, or ABI validation have been performed.

Established static evidence includes the zero-input return, single object-field read, conditional F-5990a26eaaa3 invocation, **two ordered global table-base evaluations**, initial optional selection through the second value, required and final optional selection through the retained first value, displayed argument sequences, and absence of explicit result handling or recovery.

Equality of the two base values, semantic purpose, dependency effects, ownership guarantees, and exact machine-level execution properties remain unestablished.
