<!-- Trusted provenance: F-d5cd1ae8a1b7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d5cd1ae8a1b7

## Purpose

F-d5cd1ae8a1b7 processes a 32-bit event selector and a 32-bit input value against a caller-supplied state object. Its visible behavior includes:

- Ignoring unsupported event selectors with return value 2.
- Accepting selector 2 without inspecting the state object.
- Filtering selector-1 inputs through one or two dependencies.
- Handling input values 0x3D, 0x3E, and 0x2F specially.
- Creating an associated object when ordinary input arrives without one.
- Forwarding ordinary input to an associated subordinate object.
- Setting a state flag or publishing a diagnostic value on particular branches.

“Ingesting input into a parser-like state” is inferred intent, not an established interface definition. The meanings of the event selectors, return values, flag bit, and diagnostic value are not supplied.

This contract describes the visible local behavior. It does not assign undocumented effects, ownership rules, or stability guarantees to dependencies.

## Interface

### Inputs and result

The supplied representation exposes three 32-bit arguments:

| Argument | Interpretation |
|---|---|
| State address | Address of the mutable state object. Used only on some selector-1 paths. |
| Event selector | Compared for exact equality with 1 and 2. Every other 32-bit value takes the unsupported-selector path. |
| Input value | Passed unchanged to dependencies and compared for exact equality with 0x3D, 0x3E, and 0x2F. |

The result is a 32-bit value. The explicit return values are 2, 3, 4, and 13.

No range check restricts the input value to an octet or a character set. Values outside 0–255 are not truncated by the visible logic.

### State layout used locally

Offsets below are byte offsets from the state address.

| Offset | Width | Visible use |
|---|---:|---|
| 0x28 | 32 bits | Flags word. One branch reads it, sets bit 1 using a bitwise OR with 2, and writes the result back. |
| 0x30 | 32 bits | Associated-object reference. Zero means no associated object for the visible decisions. |
| 0x38 | 32 bits | Diagnostic or status slot, written on branches that return 13. |

A nonzero associated-object reference is used as an address with this additional layout:

- At associated-object offset 4: a 32-bit subordinate reference.
- At subordinate offset 4: a 32-bit value passed to F-d8cbf9aa79a9.

The append-like operation uses the subordinate reference itself, rather than the value at subordinate offset 4.

These are the accessed fields, not a complete object definition. No structure size, alignment policy, packing convention, or byte order is established by the supplied representation.

### Path-dependent preconditions

Selector 2 and unsupported selectors do not locally dereference the state address. A null or otherwise unusable state address is therefore not rejected or accessed on those paths.

Selector-1 paths require readable or writable storage only where the selected path actually accesses it. In particular:

- Reading the associated-object slot requires valid storage at state offset 0x30.
- Traversing the associated object for F-d8cbf9aa79a9 requires valid nested references and readable fields.
- Setting the flag requires readable and writable storage at offset 0x28.
- Publishing a diagnostic requires writable storage at offset 0x38.

There are no explicit checks for malformed nonzero references. The routine provides no defined local recovery behavior for invalid storage.

## Behavior

### 1. Event-selector dispatch

The event selector is examined before any state-field access.

- If it is 2, return 3. No dependency is called.
- If it is neither 1 nor 2, return 2. No dependency is called.
- If it is 1, continue to the input gate.

These outcomes do not depend on the input value.

### 2. Input gate

Call F-2abbdd5d9a43 with the complete 32-bit input value.

If that dependency returns zero, admit the input directly to special-value and ordinary-input handling. The associated-object slot is not read as part of this gate.

If it returns nonzero:

1. Read the associated-object reference at state offset 0x30.
2. If that reference is zero, return 3 without processing the input further.
3. If it is nonzero, traverse the associated object’s field at offset 4 and then the subordinate object’s field at offset 4.
4. Pass the resulting 32-bit value to F-d8cbf9aa79a9.
5. If this dependency returns zero, return 3.
6. If it returns nonzero, admit the input to the remaining handling.

Thus, processing proceeds when either the first dependency returns zero, or the first returns nonzero and an existing associated-object chain yields a nonzero result from the second dependency.

The gate does not snapshot the associated-object reference for later use. Later decisions read the state slot again. Effects of F-d8cbf9aa79a9 can therefore influence the subsequent branch.

### 3. Diagnostic-value sampling

On entry to the admitted-input path, read a 32-bit external value and retain it as the prospective diagnostic.

This read occurs before testing the input against 0x3D, 0x3E, and 0x2F. It consequently occurs even when the admitted-input branch later returns without writing a diagnostic.

The external value’s meaning, valid range, and mutability are unknown. It is not established to be an error number, address, or fixed constant.

### 4. Input 0x3D

Read the associated-object slot again.

- If nonzero, return 2. Do not locally write the diagnostic slot.
- If zero, write the previously sampled external value to state offset 0x38 and return 13.

No creation or append dependency is called on this branch.

### 5. Input 0x3E

Read the associated-object slot again.

- If zero, return 4. Do not locally write the diagnostic slot.
- If nonzero, write the sampled external value to state offset 0x38 and return 13.

No creation or append dependency is called on this branch.

### 6. Input 0x2F

Read the associated-object slot again.

- If zero, read the 32-bit flags word at offset 0x28, set bit 1, write the resulting word back, and return 3.
- If nonzero, write the sampled external value to state offset 0x38 and return 13.

The flag operation preserves every other bit of the value read. Repeating this local operation does not toggle or clear bit 1.

No creation or append dependency is called on this branch.

### 7. Every other input value

Read the associated-object slot.

If it is nonzero, proceed directly to forwarding the input.

If it is zero:

1. Call F-8157a180d901 with the 32-bit value 12.
2. Call F-e55c7839f093 with no explicit arguments.
3. Store its 32-bit result into state offset 0x30.
4. Test that returned result for zero.

The result is stored even when it is zero. Any change a dependency made to the associated-object slot before that store is overwritten by this local assignment.

If the result is zero, write the value 12 to state offset 0x38 and return 13. This diagnostic is independent of the earlier sampled external value. Do not call the forwarding dependency.

If an associated object was already present, or the creation-like dependency returned nonzero:

1. Read the current associated-object reference from state offset 0x30.
2. Read its subordinate reference at offset 4.
3. Call F-96b68e21fa81 with that subordinate reference and the original 32-bit input value.
4. Return 3.

The forwarding dependency’s result is not used. No local write follows that call.

## Side effects

### Direct memory effects

The routine can directly perform these writes:

- Store a creation-like dependency’s result into state offset 0x30.
- Replace the flags word at offset 0x28 with its previous value OR 2.
- Replace the diagnostic word at offset 0x38 with either the sampled external value or 12.

No direct clearing of the diagnostic slot occurs on a successful or ignored-input path. A return of 2, 3, or 4 therefore does not imply that the slot contains a newly initialized value.

Likewise, return 3 does not uniquely identify an append: it also covers selector 2, gate rejection, and the flag-setting branch.

### Ordering and dependency interference

Observable local ordering matters:

- Input classification precedes the optional nested-object predicate.
- Both gating operations precede the external diagnostic-value read.
- The external read precedes special-input handling.
- F-8157a180d901 precedes F-e55c7839f093.
- The returned associated-object reference is stored before its zero-result branch is taken.
- The failure diagnostic is written after that store.
- Forwarding occurs only after the relevant associated-object and subordinate-reference reads.

Dependencies must not be presumed to preserve state fields, external data, or object storage. In particular, a dependency invoked during gating can change which later special-input branch outcome applies.

The flags read-modify-write is not established to be atomic. No synchronization, concurrent-access safety, or reentrancy guarantee is visible.

### Ownership

No explicit release operation is visible.

Storing a nonzero result from F-e55c7839f093 establishes that the state object retains that reference locally. It does not establish whether the result is newly allocated, borrowed, shared, or ownership-transferred.

Passing a subordinate reference to F-96b68e21fa81 does not establish whether that dependency retains, mutates, or releases it. The same limitation applies to references used to obtain the argument for F-d8cbf9aa79a9.

## Dependencies

| Opaque ID | Locally established interaction | Not established |
|---|---|---|
| F-2abbdd5d9a43 | Called once for selector 1 with the input value. Zero admits processing directly; nonzero triggers the associated-object gate. | Classification meaning, side effects, accepted input range. |
| F-d8cbf9aa79a9 | Called only after the first result is nonzero and the associated-object slot is nonzero. Receives the nested 32-bit field. A nonzero result admits processing. | Predicate meaning, mutation behavior, ownership effects. |
| F-8157a180d901 | Called with 12 when admitted ordinary input finds no associated object. Its result, if any, is unused. | Whether 12 is a size, category, or other control value; any allocation behavior. |
| F-e55c7839f093 | Called without explicit arguments after F-8157a180d901. Its result is stored as the associated-object reference; zero selects the failure branch. | Construction mechanism, allocation size, ownership, broader failure behavior. |
| F-96b68e21fa81 | Called with the current subordinate reference and original input. Its result is unused, followed by return 3. | Append semantics, validation, failure reporting, retention or release behavior. |

Exceptions, nonlocal transfers, termination, and memory-access failures inside dependencies are outside the established normal-return contract.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless otherwise stated, assume valid required storage, normally returning dependencies, and no dependency-induced mutation of relevant state.

1. **Unsupported selector.** Selector 0 returns 2 without reading state memory or invoking dependencies, regardless of input.

2. **Selector 2.** Selector 2 returns 3 without classifying the input, sampling the external diagnostic value, or accessing the state object.

3. **Gate rejection with no associated object.** For selector 1, if F-2abbdd5d9a43 returns nonzero and the associated-object slot is zero, return 3. Even input 0x3E does not reach its special branch and therefore does not return 4.

4. **Gate rejection with an associated object.** If the first dependency returns nonzero, the associated-object chain is valid, and F-d8cbf9aa79a9 returns zero, return 3 without sampling or writing a diagnostic.

5. **0x3D without an associated object.** If admitted, sample the external diagnostic value, write it to offset 0x38, and return 13. With a nonzero associated-object slot instead, return 2 without that write.

6. **0x3E without an associated object.** If admitted, return 4 after sampling the external value, but leave the diagnostic slot locally unchanged. With a nonzero associated-object slot instead, publish the sampled value and return 13.

7. **0x2F flag operation.** If admitted with no associated object and a flags word of 0x80000004, write 0x80000006 and return 3. Do not create an object or forward the input.

8. **Ordinary-input creation failure.** For admitted input 0x41 with no associated object, call F-8157a180d901 with 12, then F-e55c7839f093. If the latter returns zero, store zero at offset 0x30, write 12 at offset 0x38, and return 13. Do not forward 0x41.

9. **Ordinary-input creation success.** Under the preceding conditions, if F-e55c7839f093 returns a valid nonzero object reference, store it, obtain its subordinate reference, forward that reference and 0x41 to F-96b68e21fa81, and return 3.

10. **Full-width input preservation.** An admitted ordinary input of 0x100 is forwarded as 0x100, not truncated to zero.

11. **Mutation during gating.** If F-d8cbf9aa79a9 returns nonzero but clears the associated-object slot, admitted input 0x3E takes the subsequently observed zero-slot branch and returns 4. The earlier nonzero reference does not force return 13.

## Uncertainties

The following remain **documented_only**:

- Semantic names for event selectors and return values.
- The meaning of the external diagnostic value.
- The meaning of flag bit 1.
- Whether the special numeric inputs represent characters.
- Allocation, construction, retention, and release responsibilities.
- Dependency side effects and guarantees about global or state preservation.
- Object lifetime, alignment, byte order, and concurrency requirements.
- Behavior after invalid references, interrupted calls, or nonlocal dependency exits.
- Whether either creation-related dependency can fail in ways other than a normally returned zero result.

The supplied representation uses 32-bit integers, fields, results, and address-like values. That supports a 32-bit representation contract here, but does not independently establish a portable native ABI for another platform.

## Evidence

Established static evidence consists of the supplied function representation and the supplied opaque dependency mapping. It supports the branch conditions, short-circuit evaluation, field offsets, explicit call arguments, direct writes, diagnostic sampling order, and return values described above.

Parser-like intent, construction-like naming, and forwarding-like naming are interpretive descriptions only. They do not strengthen the unknown dependency contracts.

No runtime tests have been performed. Acceptance examples are conditional consequences of the supplied static representation, not observations of an executing implementation.
