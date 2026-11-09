<!-- Trusted provenance: F-a59dff6c15f7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a59dff6c15f7

## Purpose

F-a59dff6c15f7 performs a gated operation that can submit a three-word record or route it through metadata validation, accounting updates, and trailing dependency calls.

Its established behavior is to:

- Invoke an initial indirect method with the integer 3.
- Obtain an applicability object and, if present, query it through another indirect method.
- When applicability succeeds, perform three ordered dependency calls and obtain a state object.
- Evaluate two state gates: a byte read at offset `0x19`, followed conditionally by a 32-bit word read at offset `0x18` narrowed to eight bits.
- When both gates pass, call a dependency with 12 and obtain a nullable record pointer.
- Select submission or accounting/reclamation processing according to the record’s third 32-bit word.

The two state gates must not be described as reads of adjacent bytes. The second gate is the low eight bits of a loaded word; its relationship to a particular byte address depends on byte order, which is not established.

Application-level intent is unknown. “Applicability,” “submission,” “record,” and “accounting context” describe observable roles. Actual allocation, ownership transfer, and memory release are not established.

## Interface

The supplied interface has four argument positions and no explicit return value.

| Position | Representation | Established use |
|---|---|---|
| First | 32-bit address-valued argument | Supplies the address for a 32-bit read at byte offset `0x28` after applicability succeeds. |
| Second | 32-bit value | Not read by this body. |
| Third | 32-bit value | Not read by this body. |
| Fourth | Pointer to 32-bit storage | Copied into a local pointer slot after both state gates pass. Its target is not explicitly accessed. |

The body uses a 32-bit address and word model. Pointer slots, object-leading table addresses, record words, headers, and accounting entries are 32 bits. Two metadata indices are unsigned 16-bit values. The first state gate is an eight-bit read.

Normal exits provide no explicit success indicator, error code, or record pointer. The complete binary calling convention and target byte order are **documented_only**.

### Required storage

Storage requirements are path-dependent. For example, a null applicability object prevents any read through the first argument.

Objects used for indirect calls must provide a readable leading 32-bit method-table address and a callable entry at the selected byte offset:

| Object role | Method-table byte offset |
|---|---:|
| Initial notification receiver | `0x08` |
| Applicability receiver | `0xf4` |
| Submission receiver | `0x1c` |

The state object must support:

- A byte read at offset `0x19`.
- If that byte is nonzero, a 32-bit read at offset `0x18`.
- On submission, a leading-word read and method-table access.

A nonnull record requires three accessible 32-bit words:

| Record byte offset | Use |
|---|---|
| `0x00` | Written with `0x11` on submission. |
| `0x04` | Written with zero on submission. |
| `0x08` | Read to choose the branch. |

The accounting/reclamation branch additionally requires readable storage before the record:

| Relative byte offset | Width | Use |
|---|---:|---|
| `-8` | 32 bits | Header validation and masked size. |
| `-4` | 16 bits, unsigned | First metadata index. |
| `-2` | 16 bits, unsigned | Second metadata index. |

The accounting context must support every computed 32-bit entry access. No context length or index bounds are checked here. Invalid pointers, inaccessible storage, and invalid computed addresses have no defined recovery path in the supplied body.

## Behavior

### Initial notification and applicability

The first dependency call is F-120cb3bb1627. Its result is used immediately as the initial receiver. The routine reads its method-table address and invokes the entry at offset `0x08`, passing the receiver and 3.

There is no null check on that receiver.

Next, F-938c950e5046 supplies the applicability receiver:

- A null result causes a normal return.
- Otherwise, the method at table offset `0xf4` is invoked with that receiver and an opaque global value.
- A zero result causes a normal return.
- Any nonzero result enables the remaining work.

Thus, the initial notification precedes both applicability exits. No later branch undoes it.

### Ordered operations after applicability succeeds

The routine performs the following operations in order:

1. Call F-92bea7c30bd6.
2. Read the 32-bit field at first-argument offset `0x28` and pass it to F-ec34d19ea3a2.
3. Read the opaque global value supplied to F-416575e983b4 and call that dependency.
4. Call F-9f5ab13d92e9 to obtain the state object.

The first-argument field is read after F-92bea7c30bd6 returns, not before. Changes made by that dependency can therefore affect the argument passed to F-ec34d19ea3a2.

Likewise, the global argument to F-416575e983b4 is used at its later point in the sequence. No dependency is assumed to preserve that global.

The state-object result is not checked for null.

### State gates and record acquisition

The first state gate reads the byte at state-object offset `0x19`.

If that byte is zero, the routine returns without reading the second gate.

Otherwise, it reads the 32-bit word at offset `0x18` and narrows the value to its low eight bits. If that narrowed value is zero, the routine returns.

This distinguishes the accesses themselves from any byte-order interpretation. The first gate is a byte access; the second is a word access followed by narrowing.

When both gates pass:

1. The fourth argument is copied into local pointer storage.
2. F-8157a180d901 is called with `0x0c`, or 12 decimal.
3. F-2573f37ac5e3 is called to obtain the record pointer.

This body consumes no result from F-8157a180d901. The relationship between that call and the pointer returned by F-2573f37ac5e3 is unknown.

If the record pointer is null, execution returns normally. There are no explicit record writes, submission calls, or accounting updates on that path.

### Submission branch

For a nonnull record, the third 32-bit word, at byte offset `0x08`, is read.

If it is nonzero, the routine:

1. Writes `0x11` to the first record word.
2. Writes zero to the second record word.
3. Stores the record pointer into the local pointer slot.
4. Selects and invokes the state object’s indirect method at table offset `0x1c`, passing the state object and the address of that slot.

The callback receives a pointer to local pointer storage, not the record pointer directly. It can read or replace the slot during the call. The routine does not inspect the slot afterward.

The record writes precede method-table selection. If the record aliases the state object or its table, those writes can affect the subsequent indirect call. A valid submission execution therefore requires that the resulting receiver and method selection remain usable.

The body contains no explicit write to the third record word on this branch. That is not a guarantee that the callback or aliased storage leaves it unchanged.

There is no explicit release after submission. The local slot does not become caller-owned persistent storage merely because its address is passed to the callback.

### Accounting/reclamation branch

A zero third record word selects this branch. Additional checks of the already nonnull record pointer introduce no distinct ordinary outcome.

The visible sequence is:

1. Call F-2fd8826557a4.
2. Obtain the accounting-context pointer through an opaque global pointer location.
3. Form the metadata address eight bytes before the record.
4. Validate the header at that address.
5. Call F-c0fc6d033caa with the captured accounting context.
6. Perform the accounting updates described below.
7. Call F-cc6c9b4a0f68 with the captured context and metadata address.
8. Call F-e586bff64fb7 with the same captured context.
9. Return normally.

The accounting-context pointer is obtained after F-2fd8826557a4 returns. Earlier capture would not preserve this behavior if that dependency changes the global lookup.

Once captured, the context pointer is reused. Later global changes do not cause this body to select a new context, although dependencies may modify the storage reached through the captured pointer.

### Header rejection

Validation rejects a header if the zero check observes zero or the subsequent least-significant-bit check observes an odd value.

The static expression contains separate header accesses for these checks, with the second skipped when the first already detects zero. For stable metadata, this is equivalent to accepting exactly nonzero even values. Concurrent or external mutation is not established as supported.

Rejection calls F-a817d8d58c10, identified by the supplied static metadata as nonreturning. The visible normal path does not proceed to F-c0fc6d033caa or accounting updates after rejection.

The exact effect of that nonreturning call—termination, exception, or another control transfer—is unknown.

### Accounting order and arithmetic

After F-c0fc6d033caa returns, the header is read again and masked with `0x3ffffffc`. This clears bits 0, 1, 30, and 31. The resulting 32-bit quantity is retained for the first four subtractions.

Treating the captured context as an array of 32-bit entries, the updates occur in this order:

| Order | Entry index | Operation |
|---|---|---|
| 1 | `0x89` plus the unsigned 16-bit value at record offset `-4` | Subtract the retained masked quantity. |
| 2 | `0xc3` | Subtract the same quantity. |
| 3 | `0x38` plus the unsigned 16-bit value at record offset `-2` | Subtract the same quantity. |
| 4 | `0x88` | Subtract the same quantity. |

F-09dcb55e8e0a is then called. Its result supplies the index component for the next update:

| Order | Entry index | Operation |
|---|---|---|
| 5 | `0xc4` plus the result of F-09dcb55e8e0a | Subtract one. |
| 6 | `0xe4` | Subtract one. |
| 7 | `2` | Subtract a fresh header read masked with `0x3ffffffc`. |

These are word indices, not byte offsets. Each selected entry occupies four bytes.

The first metadata-derived index ranges from `0x89` through `0x10088`; the second ranges from `0x38` through `0x10037`. Those additions do not overflow a 32-bit index. They can nevertheless exceed the available context storage.

The result of F-09dcb55e8e0a is represented as a 32-bit integer. No nonnegative or upper-bound check is visible. Validity of its resulting entry address is a precondition; neither its domain nor out-of-range address behavior is established.

Counter subtraction is 32-bit modular arithmetic. There are no underflow checks or saturation. Subtracting one from zero produces `0xffffffff`.

The header is not revalidated after F-c0fc6d033caa or F-09dcb55e8e0a. The final masked quantity can differ from the retained quantity used for the first four updates. It is read after updates 5 and 6, so aliases with accounting storage can also affect it.

Updates are sequential, not simultaneous. Metadata indices are accessed at their respective update expressions, rather than both being captured before accounting begins. Overlapping destinations or aliases with metadata can therefore make later operations observe earlier writes. The source does not establish every within-expression read ordering.

## Side effects

Explicit effects include:

- Initial notification and all dependency calls reached before an early exit.
- Two ordered record-word writes on submission.
- A callback receiving the address of a local pointer slot.
- Seven ordered accounting updates on the accepted accounting/reclamation path, potentially targeting overlapping storage.
- A nonreturning failure call for rejected metadata.

The fourth argument’s target is never explicitly read or written. Its value is copied into local storage, then that slot is replaced on either nonnull-record branch before any visible use of its address.

No synchronization covering the routine is established. F-c0fc6d033caa and F-e586bff64fb7 must not be assumed to form a lock/unlock pair merely because they surround accounting work.

Ownership remains unresolved. Record acquisition does not prove allocation, submission does not prove ownership transfer, and the trailing calls do not prove deallocation. Dependencies may modify globals, records, metadata, or accounting storage; preservation cannot be assumed.

## Dependencies

| Opaque function ID | Established use |
|---|---|
| F-120cb3bb1627 | Supplies the initial notification receiver. |
| F-938c950e5046 | Supplies the nullable applicability receiver. |
| F-92bea7c30bd6 | First direct call after applicability succeeds. |
| F-ec34d19ea3a2 | Receives the first argument’s 32-bit field at offset `0x28`. |
| F-416575e983b4 | Receives an opaque global value. |
| F-9f5ab13d92e9 | Supplies the state object and submission receiver. |
| F-8157a180d901 | Receives 12 before record acquisition. |
| F-2573f37ac5e3 | Supplies the nullable record pointer. |
| F-2fd8826557a4 | Runs before accounting-context lookup. |
| F-a817d8d58c10 | Nonreturning destination for rejected metadata. |
| F-c0fc6d033caa | Receives the captured context before accounting. |
| F-09dcb55e8e0a | Supplies an unchecked accounting-index component. |
| F-cc6c9b4a0f68 | Receives the context and metadata address after accounting. |
| F-e586bff64fb7 | Receives the captured context after the preceding call returns. |

The three indirect methods are identified only by their receiver roles and table offsets. Independent opaque IDs and complete contracts for those targets are unavailable.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless an example states otherwise, required storage is valid, dependencies return normally, metadata remains stable, and relevant locations do not alias.

1. **No applicability receiver.**  
   F-938c950e5046 returns null. Initial notification has already occurred. No applicability method, first-argument field read, state access, or record acquisition follows.

2. **Applicability denied.**  
   The applicability method returns zero. F-92bea7c30bd6 is not called, and the first argument need not provide readable storage at offset `0x28` on this path.

3. **First state gate fails.**  
   The byte at state offset `0x19` is zero. The three permitted-path calls have occurred, but the word at offset `0x18` is not read for the second gate.

4. **Second state gate fails.**  
   The first gate passes, but the word read at offset `0x18` has low eight bits equal to zero. Neither F-8157a180d901 nor F-2573f37ac5e3 is called. This example does not identify the narrowed value with a particular addressed byte.

5. **Null record.**  
   Both gates pass. F-8157a180d901 receives 12, then F-2573f37ac5e3 returns null. No submission or accounting work follows.

6. **Submission.**  
   The third record word is nonzero. The first two words are written with `0x11` and zero, in that order. The callback receives a slot containing the record pointer. No explicit post-callback release occurs.

7. **Rejected header.**  
   The third record word is zero and the stable header is zero or odd. F-2fd8826557a4 and context lookup precede F-a817d8d58c10. No normal-path accounting updates occur.

8. **Accepted zero masked size.**  
   A stable header of `0x00000002` passes validation and masks to zero. Size entries are still accessed and written; both count entries are decremented.

9. **Header mutation across a dependency.**  
   The header is `0x24` after F-c0fc6d033caa. F-09dcb55e8e0a changes it to `0x44`. With disjoint accounting storage, the first four updates subtract `0x24`, while entry 2 subtracts `0x44`.

10. **Overlapping counters.**  
    F-09dcb55e8e0a returns `0x20`, making update 5 target entry `0xe4`, also targeted by update 6. If that entry starts at zero and the dependency leaves it unchanged, the two ordered decrements leave `0xfffffffe`.

## Uncertainties

The following remain **documented_only**:

- Domain meaning of the operation, gates, opaque globals, and tag `0x11`.
- Full indirect-method identities and contracts.
- Allocation, release, retention, and ownership-transfer behavior.
- Dependency preservation of globals or reachable storage.
- Synchronization and supported concurrent mutation.
- Valid context dimensions and the dynamic index’s permitted range.
- Byte order, complete layouts, alignment requirements, and full calling convention.
- Behavior for invalid storage or computed addresses.
- The exact control-transfer mechanism of F-a817d8d58c10.

Reclamation intent is inferred from metadata checks, accounting reductions, and trailing calls. Actual memory reclamation is not established.

## Evidence

Established claims derive from the supplied static body: accessed widths and offsets, branch conditions, constants, explicit writes, dependency order, indirect-call arguments, and repeated reads.

Dependency identities provide no evidence of callee ownership or preservation behavior. The nonreturning characterization of F-a817d8d58c10 comes from supplied static metadata.

No runtime tests have been performed. Acceptance examples are conditional consequences of the static operations, not execution evidence.
