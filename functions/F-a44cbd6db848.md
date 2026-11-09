<!-- Trusted provenance: F-a44cbd6db848; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a44cbd6db848

## Purpose

F-a44cbd6db848 performs an ordered teardown-like sequence on an input object and a related object discovered through a dependency. It replaces object-leading words, processes embedded regions, sends a notification using a context field, conditionally processes associated state, and performs final associated-object operations.

On normal completion, it returns the related-object address. That address is not established as equal to the input address. One conditional terminal sequence reaches a dependency identified in the supplied representation as non-returning.

**Established static behavior:** memory offsets, 32-bit values, explicit reads and writes, dependency ordering, conditional branches, and the normal return address.

**Inferred intent:** the leading-word replacements and staged operations resemble destruction across object layers. This interpretation does not establish dispatch-table identity, ownership transfer, reference counting, or memory release.

## Interface

### Input and output

The function accepts one input-object pointer. The supplied representation uses 32-bit pointers and 4-byte scalar words.

Before its first input-object write, the function reads a 32-bit value from an opaque global storage location. It then writes that value at input offset 0. There is no input null check or other validation.

F-c65a7e7be442 receives the address at input offset 28 and returns an address called the **anchor**. The **related object** is located 28 bytes before that anchor.

The normal result is the related-object pointer. It is not a status code and does not certify that the returned storage remains usable.

### Object and local-storage layout

All offsets are in bytes. Each field listed below occupies 4 bytes.

| Storage | Offset | Observed role |
|---|---:|---|
| Input object | 0 | Destination of the entry-stage word |
| Input object | 96, 76 | Addresses passed, in that order, to F-eb4bfcfcd9fa |
| Input object | 64, 56 | Addresses passed, in that order, to F-a600efb7b035 |
| Input object | 28 | Address passed to F-c65a7e7be442 |
| Related object | 0 | Destination of intermediate-stage and final-stage words |
| Related object | 4 | Final associated-object pointer |
| Related object | 8 | Terminal-condition field |
| Related object | 12 | Optional associated-state pointer |
| Associated state | 4 | Opaque value saved for later calls |
| Associated state | 8 | Pointer used for iteration and a later indirect operation |
| Context | 28 | First argument of F-77781c8e0ff9 |
| Context | 36 | Compared with the related-object pointer; conditionally cleared |
| Context | 40 | Tested, used for indirect dispatch, supplied to a dependency, and compared |

The function also provides a 20-byte local region to iterator-related dependencies and a local output region containing two 32-bit pointer slots. Only the first output slot is inspected by this function.

The listed offsets describe accesses, not complete object sizes or independently verified object types.

### Global-value inputs

Four separate opaque global storage locations supply 32-bit values:

| Opaque storage label | Read point | Use |
|---|---|---|
| Entry-word global | Before the initial input-object write | Written to input offset 0 |
| Intermediate-word global | After anchor discovery and before F-408b56b8b7c8 | Written to related-object offset 0 |
| Notification-value global | After the first context-provider call, while preparing F-77781c8e0ff9 | Passed as that dependency’s second argument |
| Final-word global | After optional associated-state processing | Written to related-object offset 0 |

These labels distinguish storage locations, not fixed constants. The body does not establish immutability, values, semantic identities, or preservation by intervening dependencies.

For the notification call, both the context’s offset-28 field and the notification global are read after the context-provider call and before the notification dependency is invoked. Their relative evaluation order is not established.

### Operational preconditions

Normal execution requires accessible storage for every actual read and write, valid addresses returned by dependencies, and callable indirect-dispatch entries where used.

The anchor must support the 28-byte backward adjustment to valid related-object storage. Pointer adjustments have no visible range or overflow checks. The contract does not treat arithmetic wraparound as a way to obtain valid storage.

Dependencies may affect memory subsequently accessed by the function. Normal completion therefore requires those later accesses to remain valid, even where the function performs no renewed null check. No ownership or preservation guarantee is supplied by this body.

## Behavior

### Entry operations and related-object discovery

The entry-word global is read, and its current 32-bit value replaces input offset 0.

The embedded-region operations then occur in this order:

1. F-eb4bfcfcd9fa receives input plus 96 bytes.
2. F-eb4bfcfcd9fa receives input plus 76 bytes.
3. F-a600efb7b035 receives input plus 64 bytes.
4. F-a600efb7b035 receives input plus 56 bytes.
5. F-c65a7e7be442 receives input plus 28 bytes.

The returned anchor determines the related object by subtraction of 28 bytes. Only after this discovery does the function read the intermediate-word global and write the value obtained to related-object offset 0. It then invokes F-408b56b8b7c8 with the related-object pointer.

The input and related object may alias. Distinctness is not established; if they share offset-0 storage, the intermediate write replaces the earlier entry-stage value there.

Next, F-3fdd14ecd250 supplies a context address. F-77781c8e0ff9 receives:

- The 32-bit field at offset 28 of that context.
- The 32-bit value read from the notification-value global.
- The related-object pointer.

The context-provider call precedes both argument reads. No order is imposed between those two reads. In particular, the notification value is not an entry-time snapshot: changes made before its read, including changes during the context-provider call, can affect the argument.

After the notification dependency returns, the function reads related-object offset 12.

### Optional associated-state processing

If related-object offset 12 is zero, all associated-state processing is skipped.

Otherwise, the function retains that state pointer and reads its offset-4 value into a saved 32-bit snapshot. Subsequent uses of this saved value do not reload it from memory.

F-a1a48bd3fbde then receives the related-object pointer. Its zero versus nonzero result selects the following optional group.

#### Nonzero predicate-result group

A zero result skips this group entirely.

For a nonzero result, the function obtains a context through F-3fdd14ecd250 and tests its offset-40 field.

If that field is nonzero:

- F-3fdd14ecd250 is called again.
- The newly returned context’s offset-40 field identifies an object whose leading dispatch pointer is read.
- The dispatch-table entry at byte offset 72 is invoked.
- Related-object offset 12 is reloaded, and the offset-4 value of that newly referenced state is read.
- F-3fdd14ecd250 is called again.
- F-a5514b9f24f7 receives the latest context’s offset-40 value and the freshly read state value.

The first nonzero test does not validate the object used by the later indirect call: a separate context lookup intervenes, and no second null check is performed. Likewise, the reloaded state pointer is dereferenced without a null check. It need not equal the retained state pointer.

Whether or not the tested context field was nonzero, the nonzero predicate-result group finishes by calling:

1. F-65528ded3450 with the related object and a 32-bit zero.
2. F-a82ec4b43e2f with the originally saved state value.

The value passed to F-a5514b9f24f7 does not replace that saved value.

#### Iteration and collection-object operation

After the predicate-controlled group, the function tests offset 8 of the retained associated state.

If that field is zero, iterator initialization, iteration, finalization, and the collection-object operation are all skipped.

If it is nonzero, F-538044b67754 receives the 20-byte local region, the state’s offset-8 value, and the 32-bit bit pattern `0xFFFFFFFD`. The field access used for this argument is separate from the preceding nonzero test; no intervening dependency call appears between them.

The function then repeatedly invokes F-ebd94610a438 with the local iterator region and two-slot output region:

- A zero return ends the loop without processing an output item.
- A nonzero return causes inspection of the first output slot.
- A zero first slot causes no indirect invocation.
- A nonzero first slot identifies an object whose dispatch-table entry at byte offset 4 is invoked.

The second output slot is not inspected. There is no visible iteration bound, cycle detection, or requirement that every successful iteration produce a nonnull item.

After the first zero iterator result, F-944b595e25e9 receives the iterator region.

The retained state’s offset-8 field is then tested again. If nonzero, the currently referenced object is used for a dispatch-table call at byte offset 4. This is not necessarily the pointer supplied to iterator initialization: iteration and finalization calls may have changed the field.

The function itself does not establish iterator order, item ownership, or whether these indirect operations release anything.

#### Context comparisons and state cleanup

Still within the nonnull-state branch, the function obtains another context and compares its offset-40 field with the related-object pointer. Equality causes F-a82ec4b43e2f to receive the originally saved state value.

Next, a fresh context is obtained and its offset-36 field is compared with the related-object pointer. If equal, another context lookup occurs, and zero is written at offset 36 of that latest context.

Thus, the comparison and clearing can concern different contexts. The write is not conditional on the latest context’s own offset-36 value matching anything.

The branch ends with these calls, in order:

1. F-65528ded3450 receives the related object and zero.
2. F-8349c252c168 receives the retained associated-state pointer and zero.

Consequently:

- F-65528ded3450 is called once whenever associated state exists, with an additional call when the predicate result is nonzero.
- F-a82ec4b43e2f can be called twice with the same saved value under two independent conditions.
- Cleanup uses the retained state pointer even if related-object offset 12 has changed.
- No explicit write clears related-object offset 12.

### Final related-object stage

After the associated-state branch completes or is skipped, the final-word global is read. Its current value replaces related-object offset 0. Earlier reads of other globals do not supply or constrain this value.

The function then tests related-object offset 4.

If zero, the related-object pointer is returned immediately. Related-object offset 8 is not tested on this path.

If offset 4 is nonzero:

- F-c719314a16bb receives the related-object pointer.
- Offset 4 is reloaded and tested.
- If still nonzero, the referenced object’s dispatch-table entry at byte offset 4 is invoked.
- Related-object offset 8 is then tested.

A zero offset-8 value leads to normal return of the related-object pointer. A nonzero value leads to F-c8eefdf96f4b, followed by F-feee9156a257, which the supplied representation identifies as non-returning.

Clearing offset 4 during F-c719314a16bb suppresses the indirect call but does not suppress the subsequent offset-8 test. Changes made by the indirect call can also affect the later offset-8 test.

## Side effects

Direct external writes established by the body are:

- The entry-global value written to input offset 0.
- The intermediate-global value written to related-object offset 0.
- The final-global value written to related-object offset 0.
- A conditional zero written to offset 36 of a freshly obtained context.

There are four explicit global-value reads and no explicit writes to those four global locations. Dependencies and indirect invocations are not established to preserve them.

Dependencies receive addresses of local iterator and output storage and may populate those regions. Neither local region is returned.

Other external changes depend on callees. An offset-4 dispatch invocation does not prove destruction, deallocation, or reference-count decrement. Passing the state pointer to F-8349c252c168 does not establish transfer or release of ownership.

No allocation, explicit memory release, rollback, exception handler, or recoverable-error result is visible in the body. If an operation does not return normally, later operations are not established as occurring. Already completed writes are not visibly undone.

## Dependencies

| Opaque function ID | Observable interaction |
|---|---|
| F-eb4bfcfcd9fa | Processes input-region addresses at offsets 96, then 76 |
| F-a600efb7b035 | Processes input-region addresses at offsets 64, then 56 |
| F-c65a7e7be442 | Receives input plus 28; supplies the anchor |
| F-408b56b8b7c8 | Receives the related object after the intermediate-word write |
| F-3fdd14ecd250 | Supplies a context separately at every lookup |
| F-77781c8e0ff9 | Receives context offset 28, a read of the notification global, and the related object |
| F-a1a48bd3fbde | Supplies the zero/nonzero predicate result |
| F-a5514b9f24f7 | Receives a fresh context field and a freshly reloaded state value |
| F-65528ded3450 | Receives the related object and zero once or twice within state processing |
| F-a82ec4b43e2f | Receives the saved state value under either of two conditions |
| F-538044b67754 | Receives local iterator storage, a collection pointer, and `0xFFFFFFFD` |
| F-ebd94610a438 | Advances iteration and supplies output slots |
| F-944b595e25e9 | Receives iterator storage after normal loop termination |
| F-8349c252c168 | Receives the retained state pointer and zero |
| F-c719314a16bb | Runs before the final associated-pointer reload |
| F-c8eefdf96f4b | Runs immediately before the designated non-returning dependency |
| F-feee9156a257 | Designated terminal, non-returning dependency |

Indirect dependencies are selected through dispatch-table byte offsets 72 and 4. Their identities, implicit receivers, complete calling conventions, and ownership effects are unresolved.

## Acceptance examples

These are conditional static expectations, not executed tests. Conditions apply when the relevant reads occur. Any preservation assumptions below are example premises, not callee guarantees.

### Global changes during earlier operations

Suppose the entry global contains value A at entry. An embedded-region dependency changes the intermediate global to B before anchor discovery finishes.

The input offset-0 write uses A. The later intermediate write uses B, assuming no further change before its read. The intermediate value is not captured before the embedded-region calls.

### Notification-provider mutation

Suppose F-3fdd14ecd250 changes the notification global to C and returns a valid context.

If the global remains C until the argument read, F-77781c8e0ff9 receives C as its second argument. The relative read order of the context field and global remains unspecified.

### Final-global mutation

Suppose associated-state processing changes the final global from D to E before returning.

If no further change precedes the final read, related-object offset 0 receives E. An entry-time value D is not substituted.

### No associated state or final associated object

If offsets 12 and 4 are zero when tested:

- Embedded-region processing and notification still occur.
- Associated-state processing is skipped.
- The final global is still read and written.
- The related-object pointer is returned.
- A nonzero offset-8 value alone does not reach the terminal calls.

### Saved and reloaded state values differ

Suppose the saved state value is A, the predicate returns nonzero, and the context-object indirect call changes related-object offset 12 to another valid state containing B.

F-a5514b9f24f7 receives B. The predicate-group call to F-a82ec4b43e2f receives A. If the later context offset-40 comparison matches, that dependency receives A again.

### Null iteration item and replaced collection

Suppose iteration returns success with a null first slot, then success with a valid object, then zero. Suppose finalization changes the retained state’s collection field to another valid object.

The first item causes no indirect call. The second causes one offset-4 invocation. After iterator finalization, the collection-object invocation targets the replacement object, not necessarily the original collection.

### Fresh context used for clearing

Suppose a context’s offset-36 comparison succeeds, but the next lookup returns a different valid context.

The zero write targets the latter context’s offset 36. Clearing the compared context instead would not match this behavior.

### Final pointer cleared

Suppose related-object offset 4 is initially nonzero, and F-c719314a16bb clears it.

The final associated-object dispatch is skipped. Offset 8 is nevertheless tested; if nonzero, the terminal dependency sequence is selected.

## Uncertainties

The following remain **documented_only**:

- Values and semantic identities of the four separate global storage locations.
- Whether dependencies preserve those globals, contexts, objects, state fields, or dispatch pointers.
- Whether input and related-object storage coincide or overlap.
- Whether the saved 32-bit state value represents a pointer, handle, identifier, or another quantity.
- Allocation ownership and lifetime effects of every dependency and indirect operation.
- Iterator ordering, filtering, mutation tolerance, and output guarantees.
- The meaning of `0xFFFFFFFD`; its signed 32-bit interpretation is −3, but signed interpretation is not established.
- Complete indirect-call conventions and implicit receiver handling.
- Behavior under invalid memory, asynchronous mutation, reentrancy, dependency failure, or arithmetic producing unusable addresses.
- The mechanism behind the final dependency’s non-returning designation. Its implementation is unavailable, so termination, exception transfer, or another mechanism is not established.

The body provides no synchronization contract. Separate reads must not be converted into claims of stable values, but this does not establish defined behavior under concurrent modification.

## Evidence

The supplied function body supports the described 32-bit representation, byte offsets, four distinct global reads, direct writes, dependency sequence, retained values, reloads, branch conditions, and normal return address.

The notification arguments establish no relative evaluation order between the global read and context-field read. The body also supplies no basis for treating any global value as immutable or preserved across calls.

Destructor-like and collection-like interpretations are inferred intent only. Comments, prior descriptions, and apparent naming do not establish ownership or dependency semantics. The non-returning designation is reported with its implementation uncertainty retained.

No dependency implementations or runtime observations were available. No runtime tests have been performed; acceptance examples express conditional consequences of static evidence only.
