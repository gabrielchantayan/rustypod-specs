<!-- Trusted provenance: F-b02abe25deeb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b02abe25deeb

## Purpose

F-b02abe25deeb conditionally selects an object through a service interface, computes or preserves a 32-bit quantity, and invokes an external operation with three arguments. Those arguments are a preserved acquisition result, a value obtained from temporary storage through a dependency, and the selected quantity. After another temporary-storage dependency call, it invokes F-5ca71e377766 and F-dece790ddc92, with an intervening local store.

The initial discriminator admits only values whose low 16 bits equal 4. A discriminator mismatch invokes no dependencies and returns the 32-bit value 4.

**Inferred intent:** the external operation may transfer data, its first argument may designate a destination, its second may designate a source, and its third may be a byte count. F-5ca71e377766 and F-dece790ddc92 may perform notifications or updates. None of those interpretations is established by the supplied call sites. This contract therefore specifies argument values, gates, and call order without requiring copying, byte units, or notification semantics.

## Interface

### Explicit inputs

The function receives:

- An opaque 32-bit input, forwarded unchanged to F-5ca71e377766 if execution reaches that call.
- A descriptor pointer used to access 32-bit fields.

The directly accessed descriptor fields are:

| Byte offset | Represented field width | Use |
|---|---:|---|
| 0 | 32 bits | Captured context; checked for zero and passed to later dependencies. |
| 8 | 32 bits | Argument to F-c25a2fcc75b5. |
| 12 | 32 bits, narrowed to 16 bits for comparison | Processing discriminator. |

The discriminator comparison uses a signed 16-bit narrowing, but equality with 4 depends only on the low 16 bits. The high 16 bits do not affect the branch. The representation establishes a field access followed by narrowing; it does not establish the executable’s physical memory-load width.

No descriptor-null check precedes the discriminator access.

### Result

Every established normal return produces the 32-bit value 4. This value does not distinguish bypass from completed processing.

Two dependency call sites are annotated as non-returning. Execution reaching either call is not guaranteed to return normally. The representation also shows a return of 4 after the final annotated call if that call does return.

### Required validity

The descriptor must support the initial discriminator access. On the admitted path, offset 8 must be readable before the preliminary dependencies, and offset 0 must be readable after them.

Every reached indirect call requires a usable interface pointer, dispatch table, and callable entry. No visible check validates the service pointer or its dispatch table. The selected-object pointer is checked for null before its operations are invoked.

Temporary storage must be usable according to the contracts of dependencies receiving its address. The source does not establish the complete storage extent those dependencies require or whether they initialize all subsequently read words.

The validity, capacity, alignment, and ownership requirements of opaque acquisition results and external-operation arguments are **documented_only** unknowns.

## Behavior

### Discriminator and preliminary calls

The function first examines the discriminator at descriptor offset 12.

If its low 16 bits are not 4, execution returns 4 without calling any dependency or reading either of the other descriptor fields.

If its low 16 bits are 4, the following occur in order:

1. The 32-bit field at descriptor offset 8 is read and supplied to F-c25a2fcc75b5.
2. That dependency’s result is preserved for a possible service lookup.
3. F-b7af80e829ec is called to obtain a service-interface pointer.
4. The 32-bit field at descriptor offset 0 is read and preserved as the context.

The context is therefore captured after both preliminary calls. Their effects on the descriptor, globals, or aliased storage are unknown. The contract does not substitute the descriptor’s entry-time context for the value read at this point.

### Context requirement and service gate

If the captured context is zero, F-a817d8d58c10 is invoked without explicit arguments. This call is annotated as non-returning. No error value, diagnostic, exception type, or supported continuation is established for this case.

With a nonzero context, the service operation at dispatch-table byte offset `0x120` is invoked with the service pointer as its explicit argument.

- A nonzero result bypasses lookup and all object-related processing. Normal execution returns 4.
- A zero result enters the lookup path.

There is no selected-object final call on the nonzero service-gate branch.

### Lookup and object qualification

The service operation at dispatch-table byte offset `0xF4` is called with:

- The service pointer.
- The preserved result of F-c25a2fcc75b5.

Its result is treated as a selected-object pointer.

If that pointer is null, object qualification is skipped and execution reaches F-feee9156a257 with null.

If it is non-null, the selected object’s operation at dispatch-table byte offset `0x08` is invoked. The supplied representation shows no explicit arguments to this operation.

- A zero result skips quantity acquisition and subsequent optional processing.
- A nonzero result permits quantity acquisition.

Both branches remain within the lookup path and reach F-feee9156a257 unless an intervening operation fails to return.

Dispatch-table entries are used at their respective call sites. The evidence does not establish that earlier dependencies preserve interface contents or other shared state.

### Initial quantity and acquisition result

For a qualifying selected object:

1. F-1a836ac93559 is called with the captured context and the constant 4.
2. Its result is retained as an unsigned 32-bit initial quantity.
3. F-fab3799b3190 is called.
4. Its 32-bit result is retained as an opaque acquired value.

If the acquired value is zero, execution skips:

- All selected-object operations at offset `0x14`.
- All temporary conversions and subsequent temporary-storage calls.
- The external three-argument operation.
- F-5ca71e377766 and F-dece790ddc92.
- F-58ba61416cc9.

The selected-object final call still follows.

A nonzero acquired value enables quantity selection and the external-call sequence. There is no separate zero-quantity gate.

### Temporary-storage layout and quantity selection

Two distinct stack-storage regions are exposed to selected-object operations at dispatch-table byte offset `0x14`.

The first region begins with a represented four-byte area followed immediately by an associated 32-bit word. The second begins with a 32-bit word followed immediately by another associated 32-bit word. These observations do not establish aggregate types, complete valid extents, or callee write behavior.

For the initial comparison:

1. The selected-object operation at `0x14` receives the first region’s address.
2. The associated 32-bit word following its initial four-byte area is read and passed to F-8810f53aaa0b.
3. The conversion result is retained as an unsigned 32-bit comparison quantity.
4. F-289ec4311c64 receives the first region’s address.
5. The retained comparison quantity is compared with the initial quantity using unsigned ordering.

If the comparison quantity is greater than the initial quantity, the initial quantity is retained.

If the comparison quantity is less than or equal to the initial quantity, replacement occurs:

1. The selected-object operation at `0x14` receives the second region’s address.
2. The associated following 32-bit word is read and passed to F-8810f53aaa0b.
3. Only the low 16 bits of that conversion result are retained as the selected quantity.
4. F-289ec4311c64 receives the second region’s address.

The selected quantity is consequently either the full unsigned 32-bit initial quantity or a replacement in the range 0 through 65,535.

The comparison and replacement conversions consume values from separate object-operation invocations. Replacement is not assignment of the comparison quantity. No invariant establishes that replacement is smaller than the initial quantity.

The arithmetic consists of unsigned comparison and a low-16-bit mask. There is no visible addition, multiplication, unit conversion, or capacity calculation.

### External operation and following calls

With a nonzero acquired value, execution continues regardless of whether the selected quantity is zero:

1. The selected-object operation at `0x14` receives the first temporary region’s address again.
2. F-44362605ab79 receives that address and supplies another opaque value.
3. The external operation is invoked with, in order:
   - The preserved acquired value from F-fab3799b3190.
   - The result just obtained from F-44362605ab79.
   - The selected unsigned 32-bit quantity.
4. F-289ec4311c64 receives the first temporary region’s address.
5. F-5ca71e377766 receives the opaque first input, acquired value, and selected quantity.
6. The selected quantity is stored into the initial 32-bit word of the second temporary region.
7. F-dece790ddc92 receives the descriptor pointer, captured context, 0, 4, `0x21`, and acquired value.
8. F-58ba61416cc9 receives the acquired value.

The external operation’s return value is not used to select a branch. The supplied representation does not establish that this operation copies anything, interprets the third argument in bytes, permits overlap, or reports failure in any particular way.

Likewise, the two later calls are established calls with the listed arguments, not established notifications. Their application-level effects remain unknown.

The local store between those calls has no explicit later read in this function. It is not a descriptor-field write and is not an additional explicit argument to F-dece790ddc92. Because temporary-storage addresses have been exposed, possible alias-visible effects of this store remain unresolved.

### Lookup-path final call

Every entered lookup path reaches F-feee9156a257 with the lookup result, unless an intervening dependency does not return. This includes:

- A null lookup result.
- A non-null object that fails qualification.
- A qualified object with a zero acquired value.
- Completion of the full external-call sequence.

F-feee9156a257 is annotated as non-returning. Its ownership semantics and the accuracy of that annotation are unknown. If it returns, the displayed continuation returns 4.

## Side effects

The directly established local effects and interactions are:

- Ordered descriptor reads.
- Service-interface and selected-object dispatches.
- Exposure of stack-storage addresses to dependencies.
- Reads of associated temporary words after object-operation calls.
- An external invocation with three preserved or computed values.
- Ordered calls to F-5ca71e377766 and F-dece790ddc92.
- A 32-bit store into previously exposed temporary storage.
- Calls to F-58ba61416cc9 and F-feee9156a257 on their respective branches.

There is no explicit store to the descriptor. Descriptor immutability is nevertheless not guaranteed: F-dece790ddc92 receives its pointer, and other dependencies may have aliases.

No ownership transfer, allocation, release, destruction, or copying follows solely from these call sites. In particular:

- F-fab3799b3190 is not established as allocating storage.
- F-58ba61416cc9 is not established as freeing the acquired value.
- F-feee9156a257 is not established as releasing the selected object.
- F-289ec4311c64 is not established as destroying a temporary.
- F-44362605ab79 is not established as returning a buffer pointer.

Effects performed inside dependencies, including changes to globals, shared objects, descriptor storage, or memory reachable through opaque arguments, remain unknown.

## Dependencies

| Opaque function ID | Established role at this call site |
|---|---|
| F-c25a2fcc75b5 | Receives descriptor offset-8 value; supplies a preserved lookup argument. |
| F-b7af80e829ec | Supplies the service-interface pointer. |
| F-a817d8d58c10 | Invoked for zero captured context; annotated non-returning. |
| F-1a836ac93559 | Receives context and 4; supplies the initial unsigned 32-bit quantity. |
| F-fab3799b3190 | Supplies an acquired value whose zero/nonzero status gates further processing. |
| F-8810f53aaa0b | Receives associated temporary words; supplies comparison or replacement quantities. |
| F-289ec4311c64 | Receives temporary-region addresses after their relevant uses. |
| F-44362605ab79 | Receives the final temporary-region address; supplies the external operation’s second argument. |
| F-5ca71e377766 | Receives the first input, acquired value, and selected quantity. |
| F-dece790ddc92 | Receives descriptor, captured context, 0, 4, `0x21`, and acquired value. |
| F-58ba61416cc9 | Receives the acquired value after F-dece790ddc92. |
| F-feee9156a257 | Receives the selected-object pointer at the end of the lookup path; annotated non-returning. |

No opaque IDs are supplied for the external operation or the four indirect operations. They are distinguished only by call structure: service dispatch offsets `0x120` and `0xF4`, and selected-object dispatch offsets `0x08` and `0x14`. Their full contracts are **documented_only**.

## Acceptance examples

These are conditional static expectations, not executed tests. Except where a non-returning annotation is relevant, each example assumes reached dependencies return normally and supply usable values and storage.

1. **Discriminator mismatch.** A discriminator field of `0x12340005` causes no dependency calls and returns 4.

2. **High discriminator bits ignored.** A field of `0xABCD0004` admits processing. The high 16 bits do not block entry.

3. **Context captured after preliminary calls.** If an earlier dependency changes the context field from nonzero to zero before the context read, F-a817d8d58c10 is invoked. The entry-time context does not determine this branch.

4. **Service gate blocks lookup.** With nonzero captured context and a nonzero service-gate result, there is no lookup or selected-object final call. Normal execution returns 4.

5. **Null lookup result.** A zero service-gate result followed by null lookup invokes F-feee9156a257 with null, without object qualification.

6. **Qualification fails.** A non-null object whose qualification operation returns zero causes neither quantity acquisition nor F-fab3799b3190. The selected-object final call follows.

7. **Acquired value is zero.** Initial quantity acquisition still precedes F-fab3799b3190. A zero acquisition result prevents temporary production, the external operation, both subsequent argument-bearing calls, and F-58ba61416cc9.

8. **Initial quantity retained.** Initial quantity 100 and comparison quantity 101 select 100. The object operation at `0x14` occurs twice: once for comparison and once before the external operation.

9. **Equality selects replacement.** Initial and comparison quantities both equal to 100 enter replacement. A replacement conversion result of `0x00010002` selects 2. Three object operations at `0x14` occur.

10. **Replacement may increase the quantity.** Initial quantity 10, comparison quantity 5, and replacement conversion result 500 select 500. No later visible bound check reduces it.

11. **Unsigned comparison.** Initial quantity 1 and comparison quantity `0xFFFFFFFF` retain 1. The comparison quantity is not treated as negative.

12. **Zero selected quantity does not bypass calls.** A nonzero acquired value with selected quantity zero still causes final temporary production, the external invocation with third argument zero, and the subsequent dependency sequence.

13. **No established external effect.** If the external operation returns normally without changing memory, the displayed caller still proceeds to F-289ec4311c64 and then F-5ca71e377766. This caller imposes no verified-copy or byte-transfer postcondition.

## Uncertainties

- **Inferred intent:** resource selection, data transfer, destination/source roles, byte-count interpretation, and notification or update roles.
- **documented_only:** Full interface types, calling conventions, implicit receivers, temporary aggregate layouts, complete storage extents, and alignment.
- **documented_only:** Whether temporary-producing operations initialize the associated words subsequently read.
- **documented_only:** Pointer-versus-handle interpretation of opaque values.
- **documented_only:** External-operation semantics, quantity units, overlap rules, required capacity, and failure reporting.
- **documented_only:** Ownership, allocation, release, and lifetime behavior of every dependency.
- **documented_only:** Dependency effects on globals, descriptors, dispatch tables, exposed temporary storage, or other shared state.
- **documented_only:** Accuracy and consequences of the two non-returning annotations.
- **documented_only:** Alias-visible consequences of the late temporary-word store.
- **documented_only:** Concurrency, reentrancy, exceptions, and unwinding. No synchronization or recovery contract is established.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping.

Established evidence comprises field offsets, narrowing widths, explicit arguments, dispatch-table offsets, branch conditions, unsigned comparison, the low-16-bit replacement mask, temporary-address exposure, the local store, call order, and the normal return constant.

Call placement does not establish copying, byte units, notification semantics, allocation, ownership, or release. Non-returning annotations are reported separately from the displayed continuation.

No dependency-body inspection, implementation access, runtime execution, or runtime tests were performed.
