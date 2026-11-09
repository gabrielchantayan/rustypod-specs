<!-- Trusted provenance: F-192caea9fa2d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-192caea9fa2d

## Purpose

F-192caea9fa2d performs an ordered, teardown-shaped transition involving an input object and a second object located through an embedded region. It writes phase words, invokes association and lifecycle dependencies, conditionally processes a secondary record and collection, and returns the located object’s address.

**Established static evidence:** the supplied representation specifies the accesses, calls, conditions, and ordering described below.

**Inferred intent:** the phase-word assignments and indirect operations are consistent with layered object destruction. They do not establish concrete types, ownership transfer, or storage release.

**Unknown:** dependency implementations, dispatch-table contents, global-source values, and the application meaning of the accessed fields are unavailable.

## Interface

### Input, result, and arithmetic

The explicit input is an object address, designated **A**. Normal completion returns an object address, designated **B**.

The representation uses 32-bit words, 32-bit pointer-bearing fields, and four-byte object indexing. All offsets below are byte offsets. Zero/nonzero decisions examine the represented 32-bit values.

F-230c59ef0030 receives A + 168. Designate its 32-bit result **R**. B is derived by subtracting 168 bytes from R. The function returns B, not necessarily A.

There is no input-null check, object-size check, or validation of R. Defined execution requires valid storage and dispatch structures for every reached access. The supplied representation does not independently establish overflow semantics for the address arithmetic or behavior when the subtraction produces an invalid address.

### Accessed object layouts

| Region | Byte offset | Established use |
|---|---:|---|
| A | 0 | Receives two direct phase-word writes if the initial guard succeeds; otherwise only the first. |
| A | 168 | Address passed to F-230c59ef0030. |
| A | 244 | 32-bit initial guard field. |
| B | 0 | Receives three direct phase-word writes on a normally completing path. |
| B | 4 | Pointer-bearing field controlling final conditional processing. |
| B | 8 | 32-bit field checked only within that final conditional processing. |
| B | 12 | Pointer-bearing secondary-record field. |
| Secondary record | 4 | 32-bit value saved for later calls. |
| Secondary record | 8 | Pointer-bearing collection field. |
| Returned context | 28 | 32-bit first argument to F-77781c8e0ff9. |
| Returned context | 36 | Pointer-bearing field compared with B and conditionally cleared. |
| Returned context | 40 | Pointer-bearing field used for indirect dispatch, calls, and comparisons. |

B + 4, B + 8, and B + 12 correspond respectively to R − 164, R − 160, and R − 156.

These accesses establish neither complete object sizes nor a public class layout.

### Global source words

Five phase assignments obtain their 32-bit values from five distinct global-memory locations. A sixth distinct global location supplies an additional argument to F-77781c8e0ff9. For this contract, these locations are designated:

| Global location | Use and read timing |
|---|---|
| Phase source 1 | Read for the first A + 0 assignment, before the initial guard read. |
| Phase source 2 | Read for the second A + 0 assignment, after the guard succeeds. |
| Phase source 3 | Read for the first B + 0 assignment, after F-230c59ef0030 returns. |
| Phase source 4 | Read for the second B + 0 assignment, after F-a6fdc7050c77 returns. |
| Argument source | Read after the relevant F-3fdd14ecd250 call and before F-77781c8e0ff9. |
| Phase source 5 | Read for the final B + 0 assignment, after secondary-record processing or its bypass. |

Distinct source locations do **not** establish distinct values. No source is established to be immutable. Earlier dependencies may modify a source before its later read.

## Behavior

### 1. Initial transition and guard

The function reads phase source 1 and writes that value to A + 0. It then reads A + 244.

If the guard field is nonzero, it calls F-54385ab29a1f and then F-feee9156a257, both without explicit arguments. The latter is marked nonreturning in the representation. The first phase write has already occurred; no normal result is established for this path.

If the guard field is zero, the function reads phase source 2 and writes its value to A + 0. This second value need not differ from the first. It then calls F-7e3145ad059a with A.

### 2. Locate B and process an association

The function calls F-230c59ef0030 with A + 168, derives B from the returned R, reads phase source 3, and writes that value to B + 0.

It calls F-d277584d672d with B. If the returned object pointer is null, the association-specific indirect operations are skipped.

Otherwise:

1. The returned object’s offset-zero dispatch-table pointer selects the operation at byte offset 128.
2. That operation is invoked with the returned object as an explicit argument.
3. If its result equals B, the same object is used for an indirect operation at dispatch offset 124, with explicit arguments consisting of the object and zero.

The second indirect operation resolves dispatch through the object again; the first operation is not assumed to preserve its dispatch-table pointer. A result other than B skips the offset-124 operation.

The function then calls F-a6fdc7050c77 with B, zero, and one. After that call returns, it reads phase source 4, writes the value to B + 0, and calls F-408b56b8b7c8 with B.

### 3. Context-dependent call

The function obtains a context from F-3fdd14ecd250. It calls F-77781c8e0ff9 with:

1. The 32-bit field at that context’s offset 28.
2. The 32-bit value read from the global argument source.
3. B.

Both argument-producing reads occur after the context lookup and before F-77781c8e0ff9. Their relative evaluation order is not independently established by the supplied representation.

The second argument is not an established constant. In particular, F-3fdd14ecd250 or earlier dependencies may modify its global source before this read.

Only after F-77781c8e0ff9 returns does the function read B + 12. Consequently, preceding dependency effects can determine which secondary record, if any, is processed.

### 4. Secondary-record processing

Designate the pointer read from B + 12 as **S**. If S is null, all processing in this subsection is skipped.

If S is nonnull, the function reads S + 4 and saves the 32-bit value as **H**. It then calls F-a1a48bd3fbde with B.

#### Optional work controlled by F-a1a48bd3fbde

If that dependency returns zero, this optional work is skipped.

For a nonzero result, the function obtains a context and tests its offset-40 field. If that field is nonnull:

1. It obtains another context.
2. It follows that context’s offset-40 object pointer and invokes the operation at dispatch offset 72. No explicit arguments are shown for this indirect call.
3. It rereads B + 12 and reads offset 4 through the newly obtained secondary-record pointer.
4. It obtains another context.
5. It calls F-a5514b9f24f7 with that context’s offset-40 field and the freshly read secondary-record value.

These context lookups are separate. The first successful nonnull check does not guarantee that a later context is identical or that its offset-40 field is still nonnull. Likewise, the fresh B + 12 read need not return S. Valid execution requires the later dereferences to remain valid.

After the optional offset-40 work, the function calls F-65528ded3450 with B and zero, then F-a82ec4b43e2f with saved H. These two calls occur for every nonzero F-a1a48bd3fbde result, even if the initial offset-40 check was null.

#### Collection traversal

The function tests the collection field at saved S + 8. A null value skips traversal and its associated finalization and collection dispatch.

For a nonnull field, the function reads the collection field again and calls F-538044b67754 with:

- A local traversal area occupying 20 bytes.
- The collection value.
- The 32-bit bit pattern `0xFFFFFFFD`, equal to −3 under signed 32-bit interpretation.

The internal traversal-area layout and the argument’s semantic meaning are unknown.

It repeatedly calls F-ebd94610a438 with the traversal area and the address of a local pointer output slot:

- A nonzero result causes the output slot to be examined.
- A null yielded pointer receives no indirect operation.
- A nonnull yielded pointer receives an indirect operation at dispatch offset 4.
- A zero result terminates traversal; the output slot is not examined for that result.

After termination, F-944b595e25e9 receives the traversal area. The function then checks S + 8 again and, if nonnull, resolves an offset-4 indirect operation through the current collection field.

The collection used after traversal can differ from the one supplied to initialization. No explicit write clears S + 8. The offset-4 indirect calls show no explicit receiver argument; their complete calling conventions are unresolved.

#### Context comparisons and final secondary-record call

After collection processing or its bypass:

1. A context lookup supplies an offset-40 field for comparison with B.
2. Equality causes F-a82ec4b43e2f to receive saved H.
3. Another context lookup supplies an offset-36 field for comparison with B.
4. Equality causes a further context lookup, followed by a zero write to offset 36 of that newly returned context.
5. F-65528ded3450 receives B and zero.
6. F-8349c252c168 receives saved S and zero.

The context whose field is cleared need not be the context whose field matched B.

F-a82ec4b43e2f can receive H twice: once in the earlier nonzero-result branch and once after the independent offset-40 comparison. Changes to S + 4 do not replace saved H.

F-65528ded3450 is called twice when F-a1a48bd3fbde returns nonzero, and once when it returns zero. These counts apply when S is nonnull and execution continues normally through this phase.

There is no explicit clearing write to B + 12.

### 5. Final transition and guarded completion

After secondary-record processing, or immediately after its null-pointer bypass, the function reads phase source 5 and writes its value to B + 0. Dependencies executed earlier may have changed this global source.

It then tests B + 4.

If the field is zero, the function skips all remaining guarded work and returns B. In particular, it does not check B + 8.

If B + 4 is nonzero:

1. F-c719314a16bb receives B.
2. B + 4 is checked again.
3. If nonnull, an offset-4 indirect operation is resolved through the current field.
4. B + 8 is read.
5. A nonzero B + 8 causes F-c8eefdf96f4b and then F-feee9156a257 to be called without explicit arguments.

F-feee9156a257 is again marked nonreturning. Clearing B + 4 during F-c719314a16bb skips the indirect operation but does not bypass the B + 8 check. Normal completion returns B.

## Side effects

### Direct memory effects

The function directly performs:

- Reads from all five phase-source locations on normal completion.
- A read from the separate argument source before F-77781c8e0ff9.
- Two writes to A + 0 on the successful initial-guard path.
- Three writes to B + 0 on a normally completing path.
- A conditional zero write to a returned context’s offset 36.
- Local traversal and output-slot accesses associated with the traversal dependencies.

On initial guard failure, only phase source 1 and the first A write are reached among the phase assignments.

If A and B alias, their writes affect the same storage in the specified order. Equal phase-source values do not remove any specified assignment.

### Ownership and dependency effects

No explicit allocation is visible. No ownership or deallocation guarantee follows from a dependency’s position, its zero argument, or an indirect dispatch offset.

Dependencies may modify object fields, global source words, contexts, dispatch pointers, and storage lifetimes. None is assumed to preserve globals or retain or release memory.

Valid continued execution requires all subsequently accessed storage to remain usable. Saved S is reused after multiple calls; B is written after F-8349c252c168 receives S. Returning B does not itself establish that B remains usable by the caller.

### Errors and interruption

There is no explicit recoverable error result. Two guarded paths reach the dependency marked nonreturning.

Behavior for invalid pointers, invalid dispatch entries, dependency exceptions, nonlocal exits, or an unexpected return from F-feee9156a257 is not established. No rollback or exception-cleanup guarantee is visible.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-54385ab29a1f | Precedes the initial nonreturning call. |
| F-feee9156a257 | Marked nonreturning; reached on either explicit failure path. |
| F-7e3145ad059a | Receives A after its second phase assignment. |
| F-230c59ef0030 | Receives A + 168 and supplies R. |
| F-d277584d672d | Receives B; may supply the association object. |
| F-a6fdc7050c77 | Receives B, zero, one. |
| F-408b56b8b7c8 | Receives B after its second phase assignment. |
| F-3fdd14ecd250 | Repeatedly supplies contexts; identity and stability are unknown. |
| F-77781c8e0ff9 | Receives context offset 28, a global-source value, and B. |
| F-a1a48bd3fbde | Receives B; controls optional secondary-record work. |
| F-a5514b9f24f7 | Receives a context field and freshly retrieved secondary-record value. |
| F-65528ded3450 | Receives B and zero once or twice when S is nonnull. |
| F-a82ec4b43e2f | Receives saved H under two independent conditions. |
| F-538044b67754 | Initializes traversal with a collection and `0xFFFFFFFD`. |
| F-ebd94610a438 | Advances traversal and supplies an output pointer. |
| F-944b595e25e9 | Finalizes traversal after termination. |
| F-8349c252c168 | Receives saved S and zero. |
| F-c719314a16bb | Receives B when the initial final-phase B + 4 check succeeds. |
| F-c8eefdf96f4b | Precedes the final nonreturning call. |

Indirect dependencies are selected at dispatch offsets 4, 72, 124, and 128. Their identities and complete signatures remain documented_only.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless stated otherwise, they assume valid storage, returning ordinary dependencies, and no additional mutations.

1. **Initial failure:** A nonzero A + 244 causes the phase-source-1 read and A write before F-54385ab29a1f and F-feee9156a257. Phase source 2 is not read on this path.

2. **Equal phase values:** If the first two phase sources contain equal words and the guard succeeds, both A assignments still occur. Distinct sources do not require distinct assigned values.

3. **Changed phase source:** If F-a6fdc7050c77 changes phase source 4, the following B assignment uses its post-call value, not a value captured earlier.

4. **Changed argument source:** If the relevant F-3fdd14ecd250 call changes the argument source, F-77781c8e0ff9 receives the subsequently read value.

5. **Different returned object:** If R − 168 is valid and differs from A, normal completion returns that B.

6. **Association mismatch:** An offset-128 result unequal to B skips offset 124 but does not skip F-a6fdc7050c77.

7. **No secondary record:** A null B + 12 skips secondary-record processing. Phase source 5 is still read and assigned before final guarding.

8. **Saved versus fresh value:** If S + 4 changes after H is saved, calls to F-a82ec4b43e2f retain H. The optional F-a5514b9f24f7 call instead uses its later, fresh secondary-record read.

9. **Traversal and replacement:** A null yielded entry receives no dispatch. A later valid entry receives offset 4 before finalization. If S + 8 is replaced during traversal, the post-finalization dispatch uses the then-current collection field.

10. **Two independent H calls:** A nonzero F-a1a48bd3fbde result and a later offset-40 equality with B cause two calls to F-a82ec4b43e2f with H.

11. **Changing contexts:** An offset-36 match on one context followed by a different context result causes the zero write to target the latter.

12. **Final global mutation:** If secondary-record processing changes phase source 5, the final B assignment uses that changed value.

13. **Final guard behavior:** Initially zero B + 4 bypasses B + 8 even when B + 8 is nonzero. Initially nonzero B + 4 that is subsequently cleared skips its dispatch but still leads to the B + 8 check.

## Uncertainties

The following remain **documented_only**:

- Concrete global-source values, mutability constraints, and phase-word meanings.
- Whether any phase-source values happen to be equal.
- The semantic meaning of the global argument-source value.
- Ownership, allocation, release, and lifetime effects of every dependency.
- Context identity and field stability across lookups.
- Complete indirect calling conventions and dispatch target identities.
- Traversal order, mutation tolerance, termination guarantees, and initialization-argument meaning.
- Concurrency, synchronization, and reentrancy requirements.
- Whether A and B ordinarily identify the same object.
- Exact invalid-address and arithmetic-overflow behavior.
- Enforcement of the nonreturning annotations and external error classifications.

Local saves of an incoming machine value do not visibly affect the specified object writes or returned address. They do not establish an additional public parameter.

## Evidence

This contract derives from the supplied static function representation and opaque dependency mapping. It records five phase assignments sourced from five distinct global locations, the separate global argument read, their placement relative to dependency calls, accessed widths and offsets, branch conditions, repeated lookups, and the normal return expression.

Lifecycle terminology is inferred intent only. Dependency internals and unresolved execution properties are not promoted to guarantees.

No runtime tests have been performed. The acceptance examples describe observations required under their stated conditions; they are not execution reports.
