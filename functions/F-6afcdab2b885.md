<!-- Trusted provenance: F-6afcdab2b885; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6afcdab2b885

## Purpose

F-6afcdab2b885 selects one of several result-producing operations from the first byte of an input record and returns a 32-bit value.

The established behavior has four categories:

- Selectors 0, 1, 2, and 6 delegate to an indirectly selected operation, passing a nested receiver and an address within the input record.
- Selectors 3 and 4 delegate to F-ae61d45ec934 through a writable, initially zero 32-bit result cell.
- Selector 5 invokes F-ca665db57587, then an indirect operation, and conditionally reads one of two 32-bit fields.
- Every other selector returns zero.

The meaning of the returned bits is not established. They may represent a scalar, handle, address, or another four-byte value. Interpreting this operation as a particular domain-level query would be inferred intent, not established behavior.

## Interface

### Inputs

The interface has four arguments:

| Argument | Established use |
|---|---|
| First | Address of a record whose first byte selects the behavior. Additional storage is accessed according to that selector. |
| Second | No explicit use in the supplied representation. |
| Third | No explicit use in the supplied representation. |
| Fourth | Initially copied into a local 32-bit cell. Both paths that expose that cell overwrite it with zero first. No returned value is visibly derived from this argument. |

The first argument must be readable for at least one byte. There is no explicit null check before the selector read.

The following offsets are byte offsets from the first argument:

| Offset | Interpretation on relevant paths |
|---|---|
| 0 | One-byte selector. |
| 4 | Address passed to dependencies. For selector 5, this storage also begins an object whose first four bytes supply a dispatch-table address. |
| 12 | Four-byte context address, read on selector 5 only after the indirect operation returns nonzero. |
| 32 | Four-byte address of storage containing another four-byte address, used by selectors 0, 1, 2, and 6. |

These are path-specific interpretations. The evidence does not establish a complete record definition, padding policy, allocator, or total record size.

### Output

The function returns one 32-bit value.

Depending on the selector, it is:

- Zero.
- The contents of a local result cell after a dependency returns.
- A 32-bit field read from a context object.
- The result of an indirect dependency.

No error/result distinction is encoded by this function beyond its explicit zero-return branches. In particular, zero can arise from multiple unrelated causes.

### Width and representation requirements

The supplied representation uses four-byte result words, four-byte pointer-bearing loads, and integer-sized indirect-call results. The contract therefore describes the observed 32-bit layout; it does not establish a portable layout for a wider-pointer environment.

The selector is one byte. All offset calculations are in bytes. The two context-field offsets used by selector 5 are 92 and 100, not element indices.

No byte swapping, numeric conversion, narrowing of the returned word, or arithmetic on its contents is visible. Endianness and calling-convention details are not independently established.

## Behavior

### Initial selection

The function reads the selector at offset 0 and chooses a branch from that value.

There is no visible second selector read. If a dependency changes the selector later, that mutation does not cause this invocation to select a different top-level branch.

The local result cell is initially assigned the fourth argument. This initialization has no established effect on the returned value: the branches using the cell replace its contents with zero before passing its address to a dependency.

### Selectors 0, 1, 2, and 6: nested indirect dispatch

These four selectors share the same visible behavior:

1. Read the four-byte address at record offset 32.
2. Read a four-byte receiver address from the storage identified by that address.
3. Read a four-byte dispatch-table address from the beginning of the receiver.
4. Obtain the callable target from dispatch-table byte offset 28.
5. Invoke that target with the receiver as its first argument and the address at record offset 4 as its second argument.
6. Return the target’s 32-bit result.

This describes the pointer relationships required to obtain the receiver and target. It does not promise that a compiler performs only one physical load of every repeated address expression or establishes an otherwise unspecified operand-evaluation order.

No intervening dependency call is visible between selection of this path and the indirect invocation. The supplied representation contains a recovery warning for this indirect control transfer, so the exact ABI and completeness of the displayed call are less certain than the explicit fixed-target calls.

The function performs no explicit null, bounds, dispatch-table, or target validation. A valid call on this path requires readable storage at every dereferenced location and a callable target compatible with the invocation.

### Selectors 3 and 4: writable result-cell delegation

These selectors are equivalent in the supplied representation:

1. Set the local 32-bit result cell to zero.
2. Invoke F-ae61d45ec934 with the address of that cell.
3. After normal return, read the cell and return its contents.

The dependency’s direct return value is not used.

If the dependency returns normally without changing the cell, this function returns zero. If it writes a four-byte value to the cell, that value becomes the result.

The supplied representation imposes no restriction on dependency behavior beyond showing the cell’s address being passed. The dependency’s other effects, including changes to shared state, are unknown.

### Selector 5: classification, readiness check, and conditional field read

This path first invokes F-ca665db57587 with the address at record offset 4.

Its result is classified solely as zero or nonzero. There is no distinction between different nonzero values.

After that dependency returns, the function uses the storage beginning at record offset 4 as an object:

1. Read its dispatch-table address from the first four bytes.
2. Obtain the callable target at dispatch-table byte offset 8.
3. Invoke that target with the object’s address as its only visible argument.

This indirect invocation occurs for both zero and nonzero results from F-ca665db57587. A zero result from the first dependency does not skip the indirect invocation.

If the indirect invocation returns zero, the function immediately returns zero. It does not perform the explicit context-pointer or context-field read described below.

If the indirect invocation returns nonzero, the function reads the context address at record offset 12 and then reads one four-byte field:

| Result from F-ca665db57587 | Result from indirect invocation | Result |
|---|---|---|
| Zero | Zero | Zero |
| Nonzero | Zero | Zero |
| Zero | Nonzero | Word at context byte offset 92 |
| Nonzero | Nonzero | Word at context byte offset 100 |

The context address is read after the indirect invocation returns. Consequently, the explicit read uses the context address present at that later time, not a value visibly cached before either dependency call.

Likewise, the dispatch table used for the indirect invocation is obtained after F-ca665db57587 returns. The evidence does not support assuming that the first dependency preserves the object, its dispatch table, or shared state.

The zero/nonzero classification from F-ca665db57587 remains the branch-selection value for the final field choice. There is no visible second invocation of that dependency.

### Other selector values

Every selector other than 0 through 6 returns the 32-bit value zero.

On this branch, no dependency call or additional input-record field read is visible after the selector read. In particular, the branch does not require the nested pointers used by other selectors to be valid.

## Side effects

The function’s own explicit data writes are to its local result cell. No direct write to the input record, a context object, or global storage appears in the supplied representation.

That observation is not a side-effect-free guarantee:

- F-ca665db57587 receives an address within the input record.
- The selector-5 indirect dependency receives that same address as its receiver.
- The shared dispatch for selectors 0, 1, 2, and 6 receives a nested receiver plus the address at record offset 4.
- F-ae61d45ec934 receives writable access to a local four-byte cell.
- Any dependency may have additional effects not described by the supplied evidence.

There is no explicit allocation, deallocation, reference-count adjustment, ownership transfer, or resource acquisition in this function. Dependency behavior on those matters remains unknown.

The result cell is local to the invocation. The evidence establishes its use during the call to F-ae61d45ec934; it does not establish a supported lifetime for retaining its address after F-6afcdab2b885 returns.

No locking, atomic operations, retry loop, or concurrency protocol is visible. Concurrent modification of the accessed storage has no defined synchronization behavior in this contract.

Invalid addresses or incompatible indirect targets are not converted into a documented error return. There is no explicit exception handler or recovery path.

## Dependencies

| Dependency | Visible interaction | Established result use |
|---|---|---|
| F-ca665db57587 | Called once on selector 5 with the address at record offset 4. | Zero chooses context offset 92; nonzero chooses offset 100, provided the subsequent indirect result is nonzero. |
| F-ae61d45ec934 | Called once on selectors 3 and 4 with the address of an initially zero four-byte cell. | Post-call cell contents are returned; direct return value is ignored. |
| Unresolved indirect dependency, dispatch-table offset 8 | Called once on selector 5, after F-ca665db57587, with the object at record offset 4 as receiver. | Zero prevents the context-field read and yields zero; nonzero permits it. |
| Unresolved indirect dependency, dispatch-table offset 28 | Called on selectors 0, 1, 2, and 6 with a nested receiver and the address at record offset 4. | Its 32-bit result is returned. |

Opaque function IDs are unavailable for the indirect targets. Their identities, implementations, global-state effects, and ownership behavior are documented_only unknowns.

## Acceptance examples

These are conditional acceptance examples derived from the static representation, not executed tests. Each assumes valid storage, compatible callable targets, and normal dependency return unless stated otherwise.

1. **Unsupported selector:** With selector 255, the result is zero. No dependency is invoked by this function, and no nested pointer needs to be dereferenced.

2. **Result-cell dependency leaves zero unchanged:** With selector 3, if F-ae61d45ec934 leaves the supplied cell unchanged, the result is zero regardless of the fourth argument.

3. **Result-cell dependency writes a value:** With selector 4, if F-ae61d45ec934 writes the four-byte value 37 into the cell, the result is 37. Its direct return value does not determine this result.

4. **Selector 5, zero classification and successful indirect result:** If F-ca665db57587 returns zero and the subsequent indirect invocation returns nonzero, the result is the four-byte value at context offset 92. Offset 100 is not the selected field.

5. **Selector 5, nonzero classification:** If F-ca665db57587 returns any nonzero value and the indirect invocation returns nonzero, the result comes from context offset 100. The magnitude and sign of either nonzero result do not affect the choice.

6. **Selector 5, indirect zero result:** If the indirect invocation returns zero, the function returns zero for either classification result. This function does not then read the context address or either context field. Dependencies may have accessed such storage themselves.

7. **Context replacement during a dependency:** If the selector-5 indirect dependency changes the context address at record offset 12 and returns nonzero, the later explicit field read follows the changed address. The field offset still follows the earlier classification result.

8. **Dispatch-table replacement during classification:** If F-ca665db57587 changes the dispatch-table address at record offset 4, the subsequent indirect invocation resolves its target from the post-call object state.

9. **Shared dispatch result:** For selector 6, if the nested receiver’s target at dispatch-table offset 28 returns the word 0x80000000, this function returns that same 32-bit word without a visible conversion or special case.

10. **Zero field value:** A selected context field containing zero produces zero even when both dependencies returned nonzero where required. The result alone cannot distinguish this case from an indirect zero result.

## Uncertainties

The following remain documented_only:

- The semantic names and purposes of the record, receivers, context objects, and returned words.
- The meanings of the selector values beyond their branch behavior.
- Complete object layouts, alignment requirements, and allocation sizes.
- The exact ABI and fidelity of the warned-about indirect control transfer for selectors 0, 1, 2, and 6.
- The identities and contracts of both indirect targets.
- Dependency ownership rules, memory retention, resource management, and global-state effects.
- Behavior when dependencies do not return normally.
- Runtime behavior for invalid pointers, inaccessible fields, or incompatible callable targets.
- Endianness and concurrent-access guarantees.

The object/table terminology describes the observed indirection pattern. It does not independently establish a source-language class hierarchy.

## Evidence

The supplied static representation establishes selector routing, explicit byte offsets, four-byte data accesses, fixed-target calls, indirect-call structure, and the order of dependency calls relative to the conditional field reads.

The interpretation as a 32-bit layout follows the represented word and pointer-bearing accesses. Higher-level intent is not established.

A recovery warning specifically limits confidence in the shared indirect-dispatch path. No independent implementation, dependency body, ABI documentation, or runtime observation was supplied.

No runtime tests have been performed.
