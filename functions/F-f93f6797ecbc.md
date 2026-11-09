<!-- Trusted provenance: F-f93f6797ecbc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f93f6797ecbc

## Purpose

F-f93f6797ecbc performs an ordered, teardown-like operation on an input object and a related object obtained through F-230c59ef0030. It updates leading words, conditionally invokes associated objects, processes an auxiliary record and collection, conditionally clears a context reference, and returns the recovered object’s address.

“Teardown,” “phase,” “collection,” and “context” are descriptive interpretations, not established type or ownership contracts. The established behavior consists of the accesses, comparisons, calls, and ordering below. No dependency is established to free memory or preserve shared state.

Two branches reach F-feee9156a257, which the supplied representation marks as non-returning:

- The input’s 32-bit field at byte offset 0x11C is nonzero.
- The final phase initially observes a nonzero field at recovered offset 0x04 and later observes a nonzero field at recovered offset 0x08.

There is no explicit recoverable-error result.

## Interface

### Input and return value

The function receives one object address, called the **input address**.

On normal completion, it returns the **recovered address**, calculated by passing the input address plus 0xA8 bytes to F-230c59ef0030 and subtracting 0xA8 bytes from that dependency’s result. The addition corresponds to advancing 42 four-byte words.

Nothing in this function establishes that the recovered address equals the input address. Neither equality nor common allocation ownership may be assumed.

### Widths and relevant layout

The supplied representation uses four-byte words and 32-bit address-like values. All offsets below are byte offsets. Zero/nonzero tests do not require an interpretation as signed integers.

| Region | Offset or extent | Established use |
|---|---:|---|
| Input object | 0x00 | Two ordered leading-word assignments on the normal initial path |
| Input object | 0x11C | Four-byte initial guard |
| Recovered object | 0x00 | Three later leading-word assignments |
| Recovered object | 0x04 | Final branch condition and indirect-call object |
| Recovered object | 0x08 | Residual condition, examined only inside the final branch |
| Recovered object | 0x0C | Auxiliary-record address |
| Auxiliary record | 0x04 | Four-byte token |
| Auxiliary record | 0x08 | Collection-like address |
| Context result | 0x1C | Four-byte argument for F-77781c8e0ff9 |
| Context result | 0x24 | Address comparison and conditional four-byte zero write |
| Context result | 0x28 | Address test, comparison, indirect dispatch, and dependency argument |
| Local iterator region | 20 bytes | Address passed to three dependencies |
| Local element slot | One pointer-sized slot | Address passed to F-ebd94610a438; contents read after nonzero returns |

Indirect dispatch obtains a table address from an object’s first word. Observed table offsets are 0x04, 0x48, 0x7C, and 0x80. These are byte offsets into the table, not method indices.

### Global data inputs

Six distinct global data locations provide four-byte values. This contract names them by role:

| Global location label | Use and read timing |
|---|---|
| Entry-phase source | Read for the first input leading-word assignment, before the initial guard |
| Input-cleanup source | Read for the second input assignment, after the guard is found zero |
| Association-phase source | Read for the recovered leading-word assignment after address recovery |
| Record-cleanup source | Read for the next recovered assignment, after F-a6fdc7050c77 |
| Context-call argument source | Read as an argument to F-77781c8e0ff9, after F-408b56b8b7c8 and the context-getter call |
| Final-phase source | Read for the final recovered assignment, after auxiliary-record processing |

These are reads from distinct locations, **not established constants**. Their values may coincide. No evidence establishes immutability or prevents preceding dependencies from changing them. Each use depends on the value at its own read, rather than a snapshot taken on entry.

### Preconditions

Every reached memory access requires suitable accessible storage, and every reached call requires a valid callable target. There are no visible checks for null input, invalid recovered addresses, invalid context results, absent dispatch entries, address overflow, or address underflow.

Non-null tests establish only that a value is not zero. In particular, successful iteration requires the local element slot to hold a valid, readable pointer value whenever a nonzero iterator result causes its contents to be read. The caller performs no visible initialization of that slot.

Expected continuation also requires preceding dependencies and indirect calls to return normally. No lifetime or preservation guarantees follow merely from passing an address to a dependency.

## Behavior

The ordering below describes the visible caller behavior. Intervening calls may change memory or global state; repeated accesses must not be collapsed into an assumed stable value.

### 1. Initial assignment and guard

The function reads the entry-phase source and writes that four-byte value to the input’s leading word. It then reads input offset 0x11C.

If the guard is nonzero, it calls F-54385ab29a1f followed by F-feee9156a257. The guard is not rechecked between these calls. The representation supplies no normal continuation from F-feee9156a257.

If the guard is zero, the function reads the input-cleanup source, writes that value to the input’s leading word, and calls F-7e3145ad059a with the input address.

Both normal-path input assignments precede address recovery.

### 2. Address recovery and associated-object processing

F-230c59ef0030 receives the input address advanced by 0xA8 bytes. Subtracting 0xA8 bytes from its result produces the recovered address.

The function reads the association-phase source and writes its value to the recovered object’s leading word. It then calls F-d277584d672d with the recovered address.

A null result skips associated-object dispatch. A non-null result causes a call through table offset 0x80, explicitly passing that associated-object address. If the returned address equals the recovered address, the function calls through table offset 0x7C on the same associated object, explicitly passing that address and zero.

The second dispatch is resolved after the first call; no preservation of the associated object’s leading word or table is established.

F-a6fdc7050c77 then receives the recovered address, zero, and one, regardless of whether associated-object dispatch occurred.

### 3. Record phase and context-related call

After F-a6fdc7050c77 returns, the function reads the record-cleanup source and writes its value to the recovered leading word.

It calls F-408b56b8b7c8 with the recovered address, then calls F-3fdd14ecd250. For the ensuing F-77781c8e0ff9 call, it reads:

- Offset 0x1C of that context result.
- The context-call argument source.

F-77781c8e0ff9 receives those values, in that argument order, followed by the recovered address. Both reads occur after the context getter; a finer ordering between the two argument reads is not established here.

The global argument is not a literal constant and need not retain a value held before either preceding dependency.

### 4. Auxiliary-record processing

After F-77781c8e0ff9 returns, the function reads recovered offset 0x0C and captures that auxiliary-record address.

A null address skips all processing in this section. Otherwise, the function captures the four-byte token at auxiliary offset 0x04 and calls F-a1a48bd3fbde with the recovered address.

The captured record address and token remain the values used at the later points explicitly described as captured, even if dependencies replace the recovered object’s current record pointer.

#### Conditional token path

A nonzero result from F-a1a48bd3fbde enables the following sequence:

1. Obtain a context from F-3fdd14ecd250 and test its offset-0x28 field.
2. If that field is nonzero:
   - Obtain another context.
   - Use the object at this new context’s offset 0x28 to call through table offset 0x48. No explicit arguments are shown for this indirect call.
   - Re-read recovered offset 0x0C and read a fresh token at offset 0x04 of that current record.
   - Obtain another context.
   - Call F-a5514b9f24f7 with this latest context’s offset-0x28 value and the fresh token.
3. Call F-65528ded3450 with the recovered address and zero.
4. Call F-a82ec4b43e2f with the originally captured token.

The first offset-0x28 test does not validate the fields in later context results. Likewise, the original non-null auxiliary-record check does not validate the newly read record pointer used for the fresh token.

The captured and fresh tokens may differ. A zero context field skips the inner indirect call and F-a5514b9f24f7, but does not skip the following F-65528ded3450 and F-a82ec4b43e2f calls.

#### Collection path

Regardless of the F-a1a48bd3fbde result, the function next tests offset 0x08 of the **captured** auxiliary record.

If nonzero, that field is read again for the collection argument to F-538044b67754. This call receives the address of the 20-byte local region, the collection value, and the four-byte bit pattern 0xFFFFFFFD. That pattern represents −3 under signed 32-bit interpretation.

Passing the region does **not** establish that F-538044b67754 initializes it.

The function then repeatedly calls F-ebd94610a438 with the local region’s address and the local element-slot address:

- A zero return ends iteration without reading the element slot on that iteration.
- Every nonzero return causes the current element-slot contents to be read.
- A null slot value causes no element dispatch.
- A non-null slot value supplies an object whose table-offset-0x04 target is called. No explicit arguments are shown.

There is no visible caller initialization before the first iteration and no caller reset between iterations. The source does not establish that F-ebd94610a438 writes the slot on any call, or supplies a fresh element on every nonzero return. Any initialization and update guarantees must come from dependency contracts not supplied here.

After a zero iterator return, F-944b595e25e9 receives the local region’s address. This call is not established to finalize or destroy anything.

The function then rechecks captured auxiliary offset 0x08. If non-null, it resolves a table-offset-0x04 call through the object currently stored there. This is not necessarily the collection value passed earlier. No explicit arguments are shown.

No element count, duplicate suppression, or iteration bound is visible.

#### Remaining record processing

After collection processing, or immediately if the initial collection test was zero:

1. Obtain a context and compare its offset-0x28 field with the recovered address.
2. If equal, call F-a82ec4b43e2f with the captured token.
3. Obtain another context and compare its offset-0x24 field with the recovered address.
4. If equal, obtain another context and write four-byte zero to offset 0x24 of that latest context.
5. Call F-65528ded3450 with the recovered address and zero.
6. Call F-8349c252c168 with the captured auxiliary-record address and zero.

The context receiving the zero write need not be the context whose field matched. The destination field is not checked again.

With a non-null captured record, F-65528ded3450 is called once or twice. F-a82ec4b43e2f can also be called twice with the same captured token under its two independent conditions.

### 5. Final phase and residual condition

After auxiliary processing, including F-8349c252c168 when reached, the function reads the final-phase source and writes its value to the recovered leading word.

It then tests recovered offset 0x04.

If zero, it returns the recovered address without examining offset 0x08 in this phase.

If nonzero:

1. Call F-c719314a16bb with the recovered address.
2. Re-read recovered offset 0x04.
3. If currently non-null, resolve and call that object’s table-offset-0x04 target, with no explicit arguments shown.
4. Read recovered offset 0x08.
5. If nonzero, call F-c8eefdf96f4b followed by F-feee9156a257, with no normal continuation represented.
6. Otherwise, return the recovered address.

Clearing offset 0x04 during F-c719314a16bb suppresses the indirect call but does not suppress the offset-0x08 check.

## Side effects

Established direct externally addressed writes are the five leading-word assignments on a complete normal path and the conditional zero write to context offset 0x24. The initial exceptional path performs only the first of those leading-word assignments.

If input and recovered addresses coincide, their leading-word assignments affect one location in the stated sequence. Otherwise, they affect separate locations.

The function also exposes addresses of local storage to dependencies. This establishes access opportunities, not actual dependency writes. Neither initialization of the 20-byte region nor writes to the element slot are established.

All direct and indirect calls may have additional effects not characterized here. No ownership transfer, reference-count change, deallocation, or lifetime extension is proven for the input, recovered object, record, token, collection, elements, associated object, or contexts. Returning an address does not establish that dereferencing it remains safe.

No rollback is visible. Earlier writes and completed calls precede either non-returning endpoint.

## Dependencies

| Opaque ID | Established role |
|---|---|
| F-54385ab29a1f | Called after a nonzero initial guard |
| F-feee9156a257 | Endpoint marked non-returning on both exceptional paths |
| F-7e3145ad059a | Receives input after its second leading-word assignment |
| F-230c59ef0030 | Supplies the address used to calculate the recovered address |
| F-d277584d672d | Supplies an optional associated object |
| F-a6fdc7050c77 | Receives recovered address, zero, and one |
| F-408b56b8b7c8 | Receives recovered address after the record-phase assignment |
| F-3fdd14ecd250 | Independently supplies each context result |
| F-77781c8e0ff9 | Receives context field, global-read argument, and recovered address |
| F-a1a48bd3fbde | Controls the conditional token path |
| F-a5514b9f24f7 | Receives a later context field and freshly read token |
| F-65528ded3450 | Receives recovered address and zero |
| F-a82ec4b43e2f | Receives the captured token under independent conditions |
| F-538044b67754 | Receives local-region address, collection value, and 0xFFFFFFFD |
| F-ebd94610a438 | Receives region and element-slot addresses; return controls iteration |
| F-944b595e25e9 | Receives local-region address after iteration terminates |
| F-8349c252c168 | Receives captured record address and zero |
| F-c719314a16bb | Receives recovered address inside the final conditional branch |
| F-c8eefdf96f4b | Precedes the final exceptional endpoint |

Indirect targets are unidentified. Explicitly absent arguments do not establish the complete calling convention.

## Acceptance examples

These are conditional static cases, not executed tests. They assume valid reached accesses and normal returns where continuation is described.

1. **Initial rejection.** A nonzero input guard permits the entry global read and assignment, then F-54385ab29a1f and F-feee9156a257. No later phase is reached.

2. **Changed phase source.** If F-a6fdc7050c77 changes the record-cleanup source, and nothing changes it again before its read, the subsequent leading-word assignment uses the changed value.

3. **Changed call argument.** If the context getter changes the context-call argument source, its post-getter value is passed to F-77781c8e0ff9, not an entry-time value.

4. **Associated mismatch.** A non-null associated object whose slot-0x80 result differs from the recovered address receives no slot-0x7C call. F-a6fdc7050c77 still runs.

5. **Absent record.** A null record pointer skips all token and collection processing and F-8349c252c168. The final global read and leading-word assignment still occur.

6. **Iterator slot reuse.** If one iterator call leaves a valid non-null element pointer in the slot and the next nonzero-returning call leaves that slot unchanged, both iterations dispatch through the stored pointer, provided it remains valid. No caller reset or deduplication prevents this.

7. **Immediate iterator termination.** If the first iterator call returns zero, the caller does not read the element slot. It still calls F-944b595e25e9 and performs the subsequent collection-field check.

8. **Changed collection.** If the captured record’s collection field changes during iteration, the post-iteration dispatch uses its later value. A null later value skips that dispatch.

9. **Repeated token processing.** A nonzero F-a1a48bd3fbde result and a later matching context offset 0x28 cause two F-a82ec4b43e2f calls with the captured token.

10. **Different context destination.** A matching offset-0x24 comparison followed by a different context result causes zero to be written into the latter context, without checking its prior field value.

11. **Final branch distinction.** Initially zero offset 0x04 bypasses the offset-0x08 check. Initially nonzero offset 0x04 followed by its clearing still permits nonzero offset 0x08 to reach the exceptional endpoint.

## Uncertainties

The following remain **documented_only**:

- Numeric values, semantic identities, mutability rules, and alias relationships of the six distinct global locations. Their reads and timing are established; constancy is not.
- Identity and allocation relationship of input and recovered objects.
- Address-arithmetic overflow and underflow semantics, including any stronger caller constraints. No wider intermediate or explicit arithmetic check is shown.
- Dependency initialization requirements for the local region and element slot, actual writes to them, and whether nonzero iterator returns imply newly supplied contents.
- Stability of contexts, fields, dispatch tables, globals, and object lifetimes across calls.
- Ownership, release behavior, hidden arguments, exception behavior, concurrency, and reentrancy.
- Actual runtime non-returning behavior of F-feee9156a257 beyond its annotation.

Invalid pointers, unreadable slot contents, invalid targets, and violated dependency preconditions have no established fallback behavior.

## Evidence

The supplied static representation supports the widths, offsets, six distinct global reads, address adjustments, conditional paths, repeated accesses, direct writes, explicit arguments, call ordering, and normal return described above.

Teardown and iterator terminology expresses inferred intent only. Dependency bodies and runtime behavior are unavailable. In particular, passing local-storage addresses does not prove initialization or output writes, and global loads do not prove immutable constants.

No runtime tests have been performed.
