<!-- Trusted provenance: F-dd1ee03e56aa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dd1ee03e56aa

## Purpose

F-dd1ee03e56aa searches an indexed collection for the first eligible entry whose 64-bit comparison value equals a value obtained from a second object.

The search proceeds in increasing unsigned index order. Eligibility is checked before either comparison value is requested. On a match, the function finalizes the current temporary and returns the matching index. If no entry matches within the initially reported collection extent, it returns the 32-bit value `0xFFFFFFFF`.

This is a source-derived behavioral contract. The meaning of the collection, eligibility predicate, comparison values, and temporary-finalization operation is not established. No runtime tests have been performed.

## Interface

### Inputs

The function has three explicit arguments:

| Argument | Representation | Established use |
|---|---|---|
| First argument | Opaque 32-bit value | No explicit use in the supplied function body. It is not forwarded to any shown dependency. |
| Collection object | Pointer to an object whose first field supplies a dispatch-table address | Used to obtain the initial collection extent and passed to the per-index temporary initializer. |
| Comparison object | Pointer to an object whose first field supplies a dispatch-table address | Used to obtain a comparison value for each eligible entry. |

The recovered interface represents the two object arguments as pointers to integer storage. Their use establishes an object-dispatch pattern, not a complete object layout or a public integer-array interface.

The dispatch accesses are:

- Collection object: callable entry at byte offset `0x18` in the table identified by the object's first field.
- Comparison object: callable entry at byte offset `0x0C` in the table identified by the object's first field.

These are offsets within dispatch tables, not direct offsets to methods embedded in the objects.

The collection operation receives the collection object as its explicit argument. The comparison operation receives the comparison object as its explicit argument. No additional explicit arguments are shown for either indirect call.

### Output

The return value is an unsigned 32-bit quantity:

- A matching entry's zero-based index on success.
- `0xFFFFFFFF` when the captured collection extent is exhausted without a match.

The function does not return a temporary object or transfer a pointer to its local temporary storage.

### Working storage and widths

The function reserves a 24-byte local region used as the per-entry temporary. The same region is reused across iterations.

The following widths govern the visible behavior:

- Captured collection extent: unsigned 32-bit.
- Current index: unsigned 32-bit.
- Eligibility result: a recovered integer result, tested only for zero versus nonzero.
- Each comparison value: 64-bit.
- Final result: unsigned 32-bit.

The 64-bit comparison is exact equality, not ordering. Interpreting those values as signed rather than unsigned does not change the equality result.

### Preconditions

For defined execution along a particular path:

- The collection object, its dispatch-table reference, and the callable entry at offset `0x18` must be valid when accessed.
- The collection object and requested index must satisfy the initializer's requirements.
- On any eligible-entry path, the comparison object, its dispatch-table reference, and the callable entry at offset `0x0C` must be valid when accessed.
- The dependencies must support the lifecycle and ordering described below.

There is no explicit null check, object-type check, dispatch-target validation, or bounds validation beyond comparison with the captured extent.

The comparison object is not explicitly accessed when the extent is zero or when all examined entries are ineligible. This is a path-specific observation, not a general guarantee that an invalid comparison object is acceptable to all dependencies.

## Behavior

### Initial extent acquisition

Before examining any entry, the function invokes the collection object's operation at dispatch-table offset `0x18`.

Its result becomes the unsigned 32-bit search extent. This operation is called once by the visible function body. The function does not refresh the extent between entries.

The initial index is zero. At the start of every iteration, the function compares the current index with the captured extent using unsigned ordering.

If the index is greater than or equal to the extent, the function returns `0xFFFFFFFF` immediately. No temporary is initialized or finalized for that index.

Consequently, an extent of zero produces the failure result after the initial extent call and without any per-entry dependency calls.

### Per-entry temporary initialization

For each index below the captured extent, the function calls F-88e2e41f5074 with these explicit arguments, in order:

1. The address of the 24-byte local temporary region.
2. The collection object.
3. The current unsigned index.
4. A literal zero.

The initializer's result is not visibly tested. The next operation uses the temporary region directly.

This establishes the call shape and the use of local storage. It does not establish the temporary's internal layout, the interpretation of the final zero argument, or whether the initializer acquires resources.

### Eligibility filtering

After initialization, the function calls F-93c11d1bbd21 with the temporary's address.

A zero result makes the entry ineligible. On that path:

- The comparison object's operation is not called.
- F-05cc4e9ef0f4 is not called.
- The temporary is finalized through F-4e292a3dd745.
- The search advances to the next index.

Any nonzero eligibility result takes the eligible-entry path. The caller does not distinguish positive values, negative values, or different nonzero status codes.

### Comparison-value acquisition and matching

For an eligible entry, comparison values are obtained in a fixed order:

1. Invoke the comparison object's operation at dispatch-table offset `0x0C`, passing the comparison object.
2. Save its 64-bit result.
3. Invoke F-05cc4e9ef0f4 with the temporary's address.
4. Compare its 64-bit result with the saved comparison-object result.

The comparison-object operation is therefore called separately for each eligible entry. Its result is not cached across the search.

This ordering matters if either operation has side effects or if the comparison object's value changes during the search. A matching decision uses the value returned for that particular eligible entry, not necessarily a value observed earlier.

The comparison-object dispatch-table reference is accessed on each eligible-entry path. The visible function does not establish one fixed dispatch target for the entire search.

If the two 64-bit values are equal, the function:

1. Calls F-4e292a3dd745 on the temporary.
2. Returns the current index.

It does not examine later entries after a match.

If the values differ, the function finalizes the temporary and advances to the next index.

### Finalization and advancement

Every initialized temporary receives exactly one explicit call to F-4e292a3dd745 on ordinary returning execution:

- After an ineligible result.
- After an eligible but unequal comparison.
- Before returning a matching index.

The finalizer's result is not visibly used. Its name or position in the lifecycle does not establish whether it destroys an object, releases memory, adjusts references, or performs another operation.

After a nonmatching entry is finalized, the unsigned 32-bit index is incremented by one. The bound check occurs again before another initialization.

No wraparound is needed to terminate the visible loop. Even with the maximum extent, `0xFFFFFFFF`, the largest visited index is `0xFFFFFFFE`. After that entry fails to match, the increment produces `0xFFFFFFFF`, which fails the next bound check.

Thus the failure sentinel cannot also be a successful index under the represented 32-bit extent and index rules.

## Side effects

### Explicit reads and writes

The function's visible local state consists of the captured extent, current index, comparison results, and 24-byte temporary region.

Observable object and dependency interactions occur in this order:

- Read the collection object's dispatch-table reference and invoke its extent operation.
- For each visited index, initialize the temporary and query eligibility.
- For each eligible entry, read the comparison object's dispatch-table reference, invoke its comparison-value operation, and then retrieve the temporary's comparison value.
- Finalize the temporary before advancing or returning success.

The function contains no explicit write to either input object outside dependency calls. That does not establish that the objects remain unchanged: both are passed to operations that may mutate accessible state.

### Ownership and lifetime

The temporary storage belongs to this invocation and is reused for successive entries. Dependencies receive its address.

The visible function neither returns that address nor explicitly allocates or frees heap storage. Resources potentially acquired through initialization are subject to the behavior of the dependencies; their ownership rules are unknown.

F-4e292a3dd745 is called before the local storage is reused and before an ordinary successful return. The supplied body does not establish that this operation releases every acquired resource, or that a dependency cannot retain the temporary's address.

### Mutation and failure behavior

No claim is made that dependencies preserve global state, input objects, collection contents, or externally visible resources.

The collection extent remains a local snapshot even if a dependency changes the collection. The visible function neither detects such changes nor reconciles its remaining indices with a new extent.

There is no explicit exception handler, dependency-error translation, retry, or recovery path. If a dependency fails to return normally, the ordinary finalization and return guarantees above do not establish what happens. In particular, the body provides no visible cleanup mechanism for an interruption before the explicit finalizer call.

`0xFFFFFFFF` specifically denotes exhaustion in the visible control flow. It is not an established encoding for every possible dependency failure.

## Dependencies

| Dependency | Visible role | Unknown contract details |
|---|---|---|
| Collection indirect operation at table offset `0x18` | Supplies the once-captured 32-bit search extent | Meaning of the extent, mutations, synchronization, error behavior, and resource effects |
| F-88e2e41f5074 | Initializes or prepares the 24-byte temporary for a collection index, with a final argument of zero | Internal layout, initialization guarantees, resource acquisition, and interpretation of zero |
| F-93c11d1bbd21 | Supplies the zero/nonzero eligibility result | Predicate meaning, mutations, and failure behavior |
| Comparison indirect operation at table offset `0x0C` | Supplies the first 64-bit value for each eligible-entry comparison | Value meaning, stability, mutations, and error behavior |
| F-05cc4e9ef0f4 | Supplies the temporary's 64-bit comparison value | Value meaning, mutations, and lifetime effects |
| F-4e292a3dd745 | Finalizes each initialized temporary on ordinary returning paths | Release behavior, ownership effects, mutations, and failure behavior |

The two indirect targets have no supplied opaque function IDs. Their table offsets identify the established invocation points without asserting their identities.

## Acceptance examples

These are conditional acceptance criteria derived from the supplied body, not executed tests. They assume valid inputs and ordinarily returning dependencies unless stated otherwise.

### Empty extent

Given an extent result of zero:

- The result is `0xFFFFFFFF`.
- No temporary initialization, eligibility query, comparison-value retrieval, or finalization occurs.

### Ineligible entry followed by a match

Given an extent of three, index zero ineligible, and index one eligible with equal comparison values:

- Index zero is initialized, checked, and finalized without either comparison-value call.
- Index one is initialized and checked.
- The comparison object's value is obtained before the temporary's value.
- Index one is finalized before the function returns `1`.
- Index two is not visited.

### First matching entry wins

Given multiple eligible entries that would compare equal:

- The function returns the lowest matching index under the actual ordered calls.
- Every earlier visited entry is finalized.
- The matching entry is finalized.
- No later entry is initialized.

### Unequal eligible entries

Given an extent of two and two eligible entries with unequal comparison values:

- Both entries receive initialization, eligibility checking, both comparison-value calls, and finalization.
- The result is `0xFFFFFFFF`.

### Changing comparison-object value

Given two eligible entries, a comparison object that returns different values on successive calls, and no earlier match:

- Each entry is compared against the value returned on its own comparison-object call.
- The first returned value is not reused for the second entry.

### Full-width equality

Given comparison values with identical low 32 bits but different high 32 bits:

- That entry does not match.
- Finalization occurs, and the search continues or exhausts.

### Maximum extent boundary

Given extent `0xFFFFFFFF`:

- Index `0xFFFFFFFE` is within the search range.
- Index `0xFFFFFFFF` is never initialized.
- If all visited entries fail to match, exhaustion occurs without index wraparound.

## Uncertainties

The following are **documented_only**:

- The domain meaning of eligibility and the 64-bit values.
- Whether the extent represents a count, capacity, or another bound. Its established use is an exclusive unsigned bound.
- Complete object layouts, dispatch-table layouts, and ABI details beyond the represented accesses and widths.
- Whether initializer and finalizer operations implement construction and destruction in a language-specific sense.
- Dependency allocation, retention, release, and global-state behavior.
- Dependency error signaling and behavior under invalid inputs.
- Thread safety, concurrent mutation guarantees, and reentrancy.

The inferred intent is an indexed search using a filtered temporary view of each entry. That interpretation is consistent with the observed lifecycle but is not needed for the operational contract.

## Evidence

Established static evidence consists of the supplied function body and dependency-identity mapping. It supports the unsigned bound check, ascending index order, 24-byte temporary, dependency arguments, eligibility gating, ordered 64-bit equality comparison, explicit finalization paths, and return values described above.

No dependency implementations or independent interface declarations were supplied. Their internal behavior and ownership contracts remain unknown.

No runtime tests, execution traces, or empirical acceptance results have been produced.
