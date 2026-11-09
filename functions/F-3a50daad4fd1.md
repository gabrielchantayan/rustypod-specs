<!-- Trusted provenance: F-3a50daad4fd1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3a50daad4fd1

## Purpose

F-3a50daad4fd1 performs an ordered, teardown-like transition on an input object and a separately resolved object. It replaces dispatch-related words, invokes direct and indirect dependencies, conditionally processes an associated record and a collection, and conditionally clears a reference obtained through a shared-state accessor.

Normal completion returns the resolved object address. That address is derived through two dependencies and two address adjustments; equality with the input address is not established.

“Teardown,” “dispatch,” “record,” and “collection” describe apparent roles, not verified type or ownership contracts. The established behavior is the access and call sequence below. Destruction, deallocation, ownership transfer, reference counting, and continued validity of the returned address remain unestablished.

## Interface

### Input and representation

The function accepts one object pointer and returns an address-bearing value represented as a pointer.

The supplied representation uses 32-bit words and 32-bit address-bearing values. All offsets in this contract are byte offsets. Direct field writes are 32 bits wide.

| Input-relative offset | Observed use |
|---|---|
| 0 | Receives the value from the second explicit read of G-01. |
| 28 | Receives the retained first G-01 value plus 256. |
| 40 | Receives the retained first G-01 value plus 268. Its address is passed to an indirect callback. |
| 132 | Its address is passed to F-ac732fdfee54. |

There is no visible null check, extent check, or alignment validation. Ordinary execution requires storage suitable for the direct accesses and for whatever accesses dependencies perform through supplied addresses.

### Resolution and related storage

F-ac732fdfee54 receives the address of input offset 132. Its result is adjusted downward by 88 and passed to F-4753454d91ec. The latter result, adjusted downward by 44, is the **resolved object address**.

| Resolved-object offset | Observed use |
|---|---|
| 0 | Receives three successive dispatch-related replacements. |
| 4 | Controls entry into final processing; is reloaded before an indirect callback. |
| 8 | Tested only within the branch selected by the initial final offset-4 test. |
| 12 | Pointer selecting an optional associated record. |
| 28 | Receives the retained first G-02 value plus 220. |
| 36 | Optional pointer used for an early indirect callback. |

The associated record has these observed fields:

| Record-relative offset | Observed use |
|---|---|
| 4 | A 32-bit value retained for later calls; separately read through a reloaded record pointer on one path. |
| 8 | Optional collection-like pointer. |

Objects returned by F-3fdd14ecd250 have these observed fields:

| Accessor-result offset | Observed use |
|---|---|
| 28 | Supplies the first argument to F-77781c8e0ff9. |
| 36 | Compared with the resolved object; conditionally cleared through a fresh accessor result. |
| 40 | Tested, used for indirect dispatch, supplied to a dependency, or compared with the resolved object. |

These are partial access layouts, not complete object definitions. Distinct role names do not establish non-aliasing.

### Global-memory sources

Five source locations supply **seven explicit global-memory reads** on a path reaching the final replacement:

| Source | Read sequence and provenance |
|---|---|
| G-01 | First read and retained. Read again for the write to input offset 0. The retained first value, not the second value, supplies input offsets 28 and 40 after additions of 256 and 268. |
| G-02 | First read and retained after resolution. Read again for the initial write to resolved offset 0. The retained first value supplies resolved offset 28 after addition of 220. |
| G-03 | Read once after the early resolved-offset-36 callback, or after that callback is skipped. Supplies the intermediate replacement at resolved offset 0. |
| G-04 | Read once after the relevant F-3fdd14ecd250 call returns. Supplies the second argument to F-77781c8e0ff9. |
| G-05 | Read once after associated-record processing completes or is skipped. Supplies the final replacement at resolved offset 0. |

The labels identify source locations, not immutable constants. No stability guarantee establishes equality between the two G-01 reads or between the two G-02 reads. There is no intervening explicit dependency call within either pair, but that alone is not a memory-stability contract.

### Local storage and arithmetic

The function exposes a 20-byte local region to traversal-related dependencies. It also exposes an initially uninitialized two-slot output area to F-ebd94610a438. Each slot is 32 bits; this function directly consumes only the first.

Passing these addresses does not establish initialization, mutation, ownership transfer, or permission to retain them beyond the call.

Address adjustments and value additions occur in the supplied 32-bit representation. No overflow, underflow, or address-validity checks are shown. Defined wrapping for invalid arithmetic is not established. Ordinary execution requires usable resulting addresses and valid dispatch entries.

### Completion and errors

Normal completion returns the resolved object address without an explicit status code.

One guarded path calls F-c8eefdf96f4b followed by F-feee9156a257. The latter is annotated as non-returning in the supplied representation. Its mechanism and the annotation’s independent correctness are unknown.

No explicit recovery, rollback, or error translation is shown for invalid memory, dependency failure, callback failure, or nonlocal exit.

## Behavior

### 1. Input transition and initial callback

The initial global reads and writes occur in this order:

1. Read G-01 and retain that first value.
2. Read G-01 again and write the second value to input offset 0.
3. Write the retained first value plus 256 to input offset 28.
4. Write the retained first value plus 268 to input offset 40.

Consequently, input offset 0 need not contain the base value used to derive the other two writes.

The function then calls F-15643c36365d. Its result is treated as an object whose first word identifies a dispatch table. The entry at table offset 12 is invoked with two explicit arguments: that returned object and the address of input offset 40.

Neither the returned object nor its dispatch information is checked for null.

### 2. Resolution and initial resolved-object transition

After the initial callback, the function performs the two resolution calls and address adjustments specified above. Neither dependency result is validated.

It then:

1. Reads G-02 and retains that first value.
2. Reads G-02 again and writes the second value to resolved offset 0.
3. Writes the retained first value plus 220 to resolved offset 28.

The initial resolved-offset-0 value therefore need not be the base used for the offset-28 value.

The function tests resolved offset 36. If nonzero, it uses the pointed-to object’s first word as a dispatch-table address and invokes the entry at table offset 4. No explicit arguments are shown for this callback. The complete calling convention, including any implicit receiver, is not established.

After that callback returns, or immediately if the field was zero, the function reads G-03 and writes it to resolved offset 0. It then calls F-408b56b8b7c8 with the resolved object.

Next, F-3fdd14ecd250 is called. Its returned object’s offset-28 word, the subsequently read G-04 value, and the resolved object are passed to F-77781c8e0ff9, in that argument order. The representation does not establish an observable ordering between the offset-28 field read and the G-04 read.

### 3. Associated-record selection

After F-77781c8e0ff9 returns, the function reads resolved offset 12.

A zero value skips all associated-record processing. Otherwise, the function retains that record address and captures its 32-bit offset-4 value.

Later operations distinguish three sources:

- The retained record address.
- The originally captured offset-4 value.
- A record pointer reloaded from resolved offset 12, whose offset-4 value is read separately on one path.

Mutation of resolved offset 12 does not replace the retained record address or the captured value.

### 4. Predicate-controlled actions

With a retained record, the function calls F-a1a48bd3fbde with the resolved object.

A zero result skips the remaining actions in this subsection. A nonzero result causes F-3fdd14ecd250 to be called and its returned object’s offset-40 field to be tested.

If that field is nonzero:

1. F-3fdd14ecd250 is called again.
2. The new result’s offset-40 pointer identifies the object used for an indirect callback at table offset 72.
3. After the callback, resolved offset 12 is reloaded and the reloaded record’s offset-4 value is read.
4. F-3fdd14ecd250 is called again.
5. F-a5514b9f24f7 receives the latest result’s offset-40 value and the freshly read record value.

The first offset-40 test does not validate the target obtained from the second accessor call. The reloaded record pointer is dereferenced without another null check.

Whether or not that offset-40 branch was taken, the nonzero predicate causes F-65528ded3450 to receive the resolved object and zero, followed by F-a82ec4b43e2f receiving the originally captured record value.

### 5. Optional collection processing

The function tests offset 8 of the retained record. If zero, it skips initialization, iteration, post-loop processing, and the collection callback.

If nonzero, F-538044b67754 receives:

- The address of the 20-byte local region.
- The retained record’s offset-8 value, read again for this argument.
- The 32-bit bit pattern `0xFFFFFFFD`, equivalent to −3 under signed 32-bit interpretation.

The earlier nonzero test is not a validity guarantee for the later field read. F-538044b67754’s effects on the local region are unknown.

The function then repeatedly calls F-ebd94610a438 with the local-region address and the two-slot output-area address:

- A zero return ends iteration without consuming the output slots for that invocation.
- A nonzero return causes the first output slot to be examined.
- A zero first slot causes no element callback.
- A nonzero first slot identifies an object whose table-offset-4 entry is invoked.

Neither slot is initialized by this function before iteration or reset between iterations. A nonzero return does not, by itself, prove that the dependency wrote a new first-slot value. Defined consumption requires an initialized, usable value whenever the caller reads that slot. The second slot is not directly consumed.

After a zero iteration result, F-944b595e25e9 receives the 20-byte local-region address. No particular cleanup or mutation effect is established.

The function then tests the retained record’s offset-8 field again. If nonzero, it invokes that object’s table-offset-4 entry. This callback uses the field available after traversal and F-944b595e25e9, not a saved copy of the pointer supplied to F-538044b67754.

### 6. Shared-reference comparisons and record completion

Still within the retained-record branch, the function calls F-3fdd14ecd250 and compares its result’s offset-40 value with the resolved object address. Equality causes F-a82ec4b43e2f to receive the originally captured record value.

It then calls F-3fdd14ecd250 and compares its result’s offset-36 value with the resolved object. Equality causes another accessor call and a 32-bit zero write to offset 36 of that new result. The write target need not be the object whose field matched.

Finally:

- F-65528ded3450 receives the resolved object and zero.
- F-8349c252c168 receives the retained record address and zero.

On paths reaching these points, F-65528ded3450 is called once within associated processing when the predicate was zero and twice when it was nonzero. F-a82ec4b43e2f can receive the same captured value twice because its predicate-controlled and equality-controlled calls are independent.

### 7. Final replacement and guarded completion

After associated processing completes or is skipped, the function reads G-05 and writes it to resolved offset 0.

It tests resolved offset 4:

- If zero, it returns the resolved object without testing offset 8.
- If nonzero, it calls F-c719314a16bb with the resolved object.

Within that initially selected nonzero branch, offset 4 is reloaded. A nonzero reloaded value causes the pointed-to object’s table-offset-4 entry to be invoked.

The function then tests resolved offset 8, still within the original offset-4 branch. A nonzero value causes F-c8eefdf96f4b followed by F-feee9156a257. Otherwise, execution reaches the normal return.

Clearing offset 4 during F-c719314a16bb suppresses the child callback, but not the offset-8 test.

## Side effects

Established direct writes are:

- Three 32-bit writes within the input object.
- Three successive 32-bit writes to resolved offset 0.
- One 32-bit write to resolved offset 28.
- A conditional 32-bit zero write to offset 36 of a fresh F-3fdd14ecd250 result.

If addresses alias, these logical write destinations may overlap; no disjointness guarantee is supplied.

There are five global source locations and seven explicit reads on a path reaching the final replacement: two from G-01, two from G-02, and one each from G-03, G-04, and G-05. These are not seven reads taken at entry. Dependency effects may influence later reads.

Repeated accessor calls, field reloads, and dependency order are part of the observed behavior. Replacing them with a single stable snapshot requires guarantees absent from the supplied evidence.

Local-storage addresses are exposed to dependencies, but exposure alone proves no writes. Dependency and callback effects, synchronization, memory ownership, and failure behavior are unknown. Cleanup-shaped calls do not establish deallocation.

No explicit allocation or release contract is visible. The returned address is not guaranteed to designate live storage after all calls. Repeated calls with a captured value establish neither safe repeated release nor a double-release defect.

## Dependencies

The following are observed uses, not independently verified callee contracts.

| Opaque function ID | Observed role |
|---|---|
| F-15643c36365d | Returns the object used for the initial table-offset-12 callback. |
| F-ac732fdfee54 | Receives input offset 132; contributes to resolution. |
| F-4753454d91ec | Receives the first adjusted result; supplies the second resolution result. |
| F-408b56b8b7c8 | Receives the resolved object after the G-03 replacement. |
| F-3fdd14ecd250 | Repeated accessor supplying objects read at offsets 28, 36, and 40. |
| F-77781c8e0ff9 | Receives an accessor-result field, G-04, and the resolved object. |
| F-a1a48bd3fbde | Supplies the zero/nonzero predicate. |
| F-a5514b9f24f7 | Receives a freshly accessed target and a freshly read record value. |
| F-65528ded3450 | Receives the resolved object and zero at one or two associated-processing stages. |
| F-a82ec4b43e2f | Receives the captured record value under independent conditions. |
| F-538044b67754 | Receives local storage, the collection-like pointer, and `0xFFFFFFFD`. |
| F-ebd94610a438 | Controls iteration and receives the local-region and output-area addresses. |
| F-944b595e25e9 | Receives local storage after iteration stops. |
| F-8349c252c168 | Receives the retained record address and zero. |
| F-c719314a16bb | Receives the resolved object after the initial final offset-4 test succeeds. |
| F-c8eefdf96f4b | Precedes the annotated non-returning call. |
| F-feee9156a257 | Annotated as non-returning. |

Indirect dependencies use dispatch-table offsets 4, 12, and 72. Only the initial offset-12 call shows explicit arguments. The other indirect interfaces remain unknown.

## Acceptance examples

These are conditional static expectations, not executed tests. They assume valid storage and normal continuation from preceding calls unless stated otherwise.

1. **Distinct G-01 observations.** If the first G-01 read yields 4096 and the second yields 8192, input offset 0 receives 8192, offset 28 receives 4352, and offset 40 receives 4364. Equality between the reads must not be imposed.

2. **Distinct G-02 observations.** If the first G-02 read yields 12288 and the second yields 16384, the initial resolved-offset-0 write is 16384 and the offset-28 write is 12508.

3. **Later global changes.** If the early offset-36 callback changes G-03, its later read supplies the intermediate replacement. If F-3fdd14ecd250 changes G-04 before returning, the subsequent G-04 read supplies the second argument to F-77781c8e0ff9.

4. **Final source timing.** If associated processing changes G-05, the final replacement uses the value read after that processing, not an entry-time snapshot.

5. **Absent record and final child.** If the selected resolved-offset-12 value and the final offset-4 value are both zero, associated processing and final guarded processing are skipped. Offset 8 is not tested.

6. **Predicate true, first target absent.** A nonzero predicate followed by a zero first offset-40 test skips the offset-72 callback and F-a5514b9f24f7. The early F-65528ded3450 and F-a82ec4b43e2f calls still occur.

7. **Independent captured-value calls.** A nonzero predicate and a later offset-40 equality each cause F-a82ec4b43e2f to receive the original captured record value, yielding two calls.

8. **Traversal outputs.** A nonzero iteration result with an initialized zero first slot causes no callback; an initialized valid nonzero pointer causes a table-offset-4 callback. A zero result stops without reading that invocation’s slot. An uninitialized consumed slot has no established defined behavior.

9. **Reloaded destinations.** If the collection field changes during traversal, post-loop dispatch uses its later value. If an offset-36 comparison matches but the next accessor returns a different object, the zero write targets that different object.

10. **Final child removed.** If F-c719314a16bb clears offset 4, the child callback is skipped, but a nonzero offset 8 still causes the terminal call sequence.

## Uncertainties

The following remain **documented_only**:

- Concrete types, dispatch identities, semantic purpose, and object relationships.
- Global-source meanings, stability, synchronization, and values at individual reads.
- Complete dependency interfaces, implicit callback arguments, and callee effects.
- Initialization or mutation of local storage and output slots.
- Whether successful iteration replaces the first slot or leaves an earlier usable value.
- Traversal order, termination, and the meaning of `0xFFFFFFFD`.
- Ownership, allocation provenance, release behavior, and post-call lifetimes.
- Accessor stability, aliasing, concurrency, reentrancy, and global preservation.
- Behavior for invalid pointers, arithmetic failures, callback failures, and nonlocal exits.
- The mechanism and independent correctness of the non-returning annotation.

## Evidence

Established claims derive from the supplied static representation: 32-bit accesses, byte offsets, explicit global-read occurrences, retained value provenance, address adjustments, branches, dependency order, repeated accessor calls, field reloads, and direct writes.

In particular, G-01 and G-02 are each explicitly read twice. Their first observations supply adjusted secondary-field writes; their second observations supply the corresponding offset-0 writes. Together with G-03, G-04, and G-05, this establishes seven explicit global reads across five locations on a path reaching the final replacement.

Intent terminology is inference, not an additional guarantee. Dependency implementations are unavailable. Neither passing an output address nor receiving a nonzero iteration result proves initialization.

No runtime observations or dependency validation are available. No runtime tests have been performed.
