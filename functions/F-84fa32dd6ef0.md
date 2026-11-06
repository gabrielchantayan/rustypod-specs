<!-- Trusted provenance: F-84fa32dd6ef0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-84fa32dd6ef0

## Purpose

F-84fa32dd6ef0 conditionally creates an object-like value selected by a 32-bit integer, passes that value through an output-producing dependency, and optionally invokes an indirect operation on the resulting object.

The established behavior is a dispatch sequence, not a guarantee of successful construction. The supplied static evidence establishes which dependencies are called, their arguments, the conditions for the indirect call, and the returned value. It does not establish the meaning of the object types, allocation-failure handling, ownership transfer, or the semantics of the indirect operation.

The recognized selector values are **2, 3, 6, 7, 8, and 9**. Selectors 4 and 5 are explicitly nonconstructing cases. Other selector values also take the nonconstructing path.

## Interface

### Inputs

| Input | Representation | Established use |
|---|---|---|
| Selector | 32-bit integer in the represented interface | Chooses a construction path after subtraction of 2. |
| Optional-operation argument | 32-bit integer | Zero disables the indirect operation. Any nonzero value enables it when the output object is nonnull and is passed unchanged to that operation. |

The optional-operation argument is not normalized to a Boolean before forwarding. Negative values and positive values other than 1 remain distinct arguments to the indirect operation.

### Output

The represented return type is a pointer to a 32-bit integer. Operationally, the routine returns the contents of a local pointer-sized output slot.

That slot:

- Is initialized to null before dependency calls.
- Has its address passed to F-fd1a1fc2b4a6 on recognized construction paths.
- Supplies both the possible indirect-call receiver and the eventual return value.
- Is not itself explicitly assigned a constructed value by this routine.

Consequently, the returned pointer is **not necessarily** the allocation result, the initialization result, or the direct return value of F-fd1a1fc2b4a6.

### Representation and preconditions

The supplied representation uses 32-bit integer and address-sized values. For the optional indirect operation, the selected nonnull pointer must support the following accesses:

| Location | Required interpretation |
|---|---|
| Object address, offset 0 | A word containing a dispatch-table address |
| Dispatch-table address, byte offset 28 | A word containing a callable target address |

With four-byte entries, byte offset 28 corresponds to the eighth table entry. The byte offset, rather than a semantic method name, is the established contract.

A nonnull pointer alone is insufficient to establish safety. The object word, dispatch-table entry, and target must all be valid for the represented execution environment.

No validity requirement is established for the selector itself beyond its integer representation. Requirements on the dependencies’ arguments and returned values remain dependency-specific.

## Behavior

### Selection

The routine initializes the output slot to null and selects a path based on the selector minus 2.

The observed dispatch is:

| Selector | Requested size | Initialization dependency | Second initialization argument |
|---:|---:|---|---:|
| 2 | 12 bytes | F-91df7c7be299 | 0 |
| 3 | 12 bytes | F-95094c640cf9 | 0 |
| 6 | 24 bytes | F-a9844e3b050c | 0 |
| 7 | 32 bytes | F-26b2af1883b1 | 2 |
| 8 | 32 bytes | F-26b2af1883b1 | 1 |
| 9 | 32 bytes | F-26b2af1883b1 | 0 |

For each row, F-8157a180d901 receives the requested size as its sole visible argument. Its return value becomes the first argument to the listed initialization dependency.

Selectors 4 and 5 bypass all construction dependencies. Other selectors also bypass them. On such a path, the output slot remains null, the optional indirect operation is skipped, and normal completion returns null.

The subtraction used for selection must not be treated as evidence of portable signed-overflow behavior. Under a 32-bit modular machine interpretation, the recognized values are exactly those listed above; subtracting 2 does not introduce additional recognized values. The source representation alone does not independently establish the language-level handling of signed subtraction at extreme input values.

### Recognized construction paths

For any recognized selector, the visible sequence is:

1. Request the selector-specific size from F-8157a180d901.
2. Pass that dependency’s returned 32-bit value, together with the selector-specific second argument, to the selected initialization dependency.
3. Pass the initialization dependency’s return value to F-fd1a1fc2b4a6, together with the address of the local output slot.
4. Evaluate whether to perform the optional indirect operation.
5. Return the current contents of the output slot.

There is no intervening null check on the return from F-8157a180d901. Whatever value it returns is forwarded to the initialization dependency.

Likewise, there is no null or status check on the initialization result before it is forwarded to F-fd1a1fc2b4a6.

F-fd1a1fc2b4a6 receives its arguments in this order:

- First: the address of the null-initialized output slot.
- Second: the selected initialization dependency’s return value.

Its direct return value is not used as the final result. It is also not used as the receiver of the optional indirect operation. When that operation is enabled, the receiver is obtained from the output slot instead.

The routine does not require the initialization dependency to return the same address it received. It also does not establish that F-fd1a1fc2b4a6 writes the supplied value unchanged into the output slot.

### Optional indirect operation

After the construction-and-output sequence, the routine checks the optional-operation argument.

If it is zero:

- No object dispatch-table access is performed by this routine.
- No indirect operation is invoked.
- The output slot is returned.

If it is nonzero:

- The current output-slot value is selected as the prospective receiver.
- A null receiver suppresses the indirect operation.
- A nonnull receiver causes an indirect call through the entry at byte offset 28 in the dispatch table referenced by the receiver’s first word.

The indirect call receives:

1. The selected object pointer.
2. The original, unchanged optional-operation argument.

Its return value, if any, is ignored.

The established access order for a nonnull receiver is: obtain the output pointer, read the object’s first word, read the target at dispatch-table byte offset 28, then invoke that target. The routine contains no explicit write to the object or dispatch table before this call.

The eventual return is still the output-slot value, rather than the indirect operation’s result. The routine performs no explicit assignment to the output slot after the indirect call. However, the output-slot address has already been exposed to F-fd1a1fc2b4a6. Without dependency contracts, it is not valid to assume that indirect or aliased effects cannot alter that slot before the final return.

### Errors and abnormal completion

The routine contains no visible error-reporting branch, retry, fallback construction, or cleanup path.

In particular:

- An allocation-like dependency returning null does not cause an immediate null return.
- An initialization dependency returning null does not prevent the call to F-fd1a1fc2b4a6.
- A null output slot prevents only the optional indirect call.
- Invalid nonnull dispatch data is not checked before dereference and invocation.

Dependency exceptions, faults, nonlocal exits, or nonreturning behavior are not characterized by the supplied evidence. If a dependency does not return normally, subsequent steps and a normal result are not guaranteed.

## Side effects

### Effects directly established

The routine directly initializes one local output slot to null. On recognized selectors, it exposes that slot’s address to F-fd1a1fc2b4a6.

On a nonzero optional-operation argument and a nonnull output pointer, it reads the object and dispatch-table words needed for the indirect call.

It does not visibly write object fields, update global state, release memory, or destroy an object itself. Those statements describe only its explicit operations, not the transitive effects of its dependencies.

### Dependency effects

Every invoked dependency may have additional effects not shown here. The evidence does not establish that any dependency:

- Preserves globals or other shared state.
- Allocates fresh storage.
- Initializes all requested bytes.
- Preserves its input object’s contents.
- Retains or relinquishes ownership.
- Releases an earlier allocation.
- Leaves the output slot unchanged after its initial write.

The size-taking dependency is allocation-like by usage, and the following dependencies are initialization-like by usage. Those are inferred roles, not independently established guarantees.

### Ownership and lifetime

No ownership transfer is proved for any returned or forwarded address.

The output-slot address refers to local storage. Passing it to F-fd1a1fc2b4a6 permits that dependency to write the slot during the call. If the dependency retains the address for later use, its validity and any resulting behavior require an external contract; this routine provides no lifetime extension.

The indirect operation must not be assumed to preserve or destroy the object. Because the routine returns the output pointer after that operation, a lifetime guarantee for the returned pointer requires the indirect target’s contract as well as the construction dependencies’ contracts.

## Dependencies

| Opaque function ID | Visible invocation contract | Not established |
|---|---|---|
| F-8157a180d901 | Receives 12, 24, or 32; its 32-bit return value is forwarded to an initialization dependency. | Allocation mechanism, failure convention, alignment, storage lifetime, ownership. |
| F-91df7c7be299 | Used for selector 2; receives the preceding result and 0. Its result is forwarded to F-fd1a1fc2b4a6. | Object type, initialized layout, return identity, ownership effects. |
| F-95094c640cf9 | Used for selector 3; receives the preceding result and 0. Its result is forwarded to F-fd1a1fc2b4a6. | Object type, initialized layout, return identity, ownership effects. |
| F-a9844e3b050c | Used for selector 6; receives the preceding result and 0. Its result is forwarded to F-fd1a1fc2b4a6. | Object type, initialized layout, return identity, ownership effects. |
| F-26b2af1883b1 | Used for selectors 7, 8, and 9; receives the preceding result and respectively 2, 1, or 0. | Meaning of the second argument, object type, ownership effects. |
| F-fd1a1fc2b4a6 | Receives the output-slot address first and the initialization result second. Its direct return is not the final output or indirect-call receiver. | Whether, when, or how it writes the slot; retention, replacement, ownership, and other effects. |

An additional runtime-selected dependency is the target read from dispatch-table byte offset 28. It has no supplied opaque function ID. Its visible arguments are the receiver and the unchanged optional-operation argument; its result is ignored. No semantic method name is established.

## Acceptance examples

These examples are conditional static expectations, not reports of executed tests.

### Selector 2 with the optional operation disabled

Given selector 2 and optional-operation argument 0, normal execution must request 12 bytes through F-8157a180d901, pass the returned value and 0 to F-91df7c7be299, then pass that result and the output-slot address to F-fd1a1fc2b4a6 in the documented argument order.

If F-fd1a1fc2b4a6 leaves the output slot containing a particular pointer, that pointer is returned. The routine does not read its dispatch table.

### Selector 8 with a nonzero forwarded argument

Given selector 8 and optional-operation argument −3, normal execution must request 32 bytes and invoke F-26b2af1883b1 with second argument 1.

If the output-producing dependency leaves a valid nonnull object pointer in the slot, the target at dispatch-table byte offset 28 is invoked with that object pointer and −3. The second argument must not be replaced by 1 merely because it is truthy.

### Explicit nonconstructing selectors

Given selector 4 or 5, no listed dependency is invoked by this routine. The result is null for both zero and nonzero optional-operation arguments.

The same expectation applies to other selectors outside the recognized set under the stated 32-bit dispatch interpretation.

### Direct return differs from output-slot contents

Suppose F-fd1a1fc2b4a6 returns one nonnull pointer but writes a different pointer into the output slot.

The slot’s pointer controls the optional indirect operation and normal return. The dependency’s direct return must not substitute for it.

If the dependency returns a nonnull pointer but leaves the slot null, this routine returns null and performs no indirect call.

### Null allocation-like result

Suppose F-8157a180d901 returns null on a recognized selector.

If execution continues normally, the selected initialization dependency is still invoked with null as its first argument. This routine does not short-circuit the path. The ultimate outcome depends on that dependency and F-fd1a1fc2b4a6.

### Nonnull output with the operation disabled

Suppose the output slot contains a nonnull pointer whose dispatch data is unusable, and the optional-operation argument is zero.

This routine does not inspect that dispatch data. This example establishes only the absence of its optional dispatch accesses, not the safety of earlier dependency activity or later use of the returned pointer.

## Uncertainties

The following remain **documented_only**:

- The domain meaning of each recognized selector.
- Whether the size-taking dependency actually allocates storage.
- Whether initialization results are object pointers or another 32-bit representation.
- The full object layout beyond the two reads required for indirect dispatch.
- The meaning of the indirect operation and its second argument.
- Allocation, initialization, and output-production failure conventions.
- Ownership, reference counting, destruction, and returned-pointer lifetime.
- Global, aliased-memory, reentrant, and asynchronous effects of dependencies.
- Target calling-convention details beyond the two visible arguments.
- Language-level signed-overflow semantics for selector subtraction.
- Whether an exposed output-slot address is retained or subsequently accessed.

These unknowns must not be resolved by assuming conventional constructor, smart-pointer, allocator, or virtual-method behavior.

## Evidence

**Established static evidence:** The supplied body shows the null-initialized output slot, selector-dependent sizes and dependency arguments, unconditional forwarding of dependency results along recognized paths, the output-slot-address call, conditional dispatch-table accesses, and return of the output slot.

**Inferred intent:** The sequence resembles allocation, initialization, output registration or wrapping, and an optional object operation. Those labels explain likely roles but add no behavioral guarantees.

**Unknowns:** Dependency internals, ownership rules, failure semantics, platform details not fixed by the representation, and indirect-target behavior are unavailable.

**Verification status:** No runtime tests have been performed. All acceptance examples are derived from static evidence and their stated conditions.
