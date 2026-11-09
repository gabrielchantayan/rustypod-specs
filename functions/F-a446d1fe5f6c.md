<!-- Trusted provenance: F-a446d1fe5f6c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a446d1fe5f6c

## Purpose

F-a446d1fe5f6c performs an ordered, destruction-like transition on an input object and an associated object obtained through a dependency. It processes embedded components, updates a reference-like record, conditionally processes saved state and a collection, clears selected context associations, and returns the associated-object address.

Three ordered writes install dispatch-like words: one in the input object and two in the associated object. Each value is read from its respective global at the relevant stage of execution. Their values, semantic identities, and pairwise equality are unknown. Separate globals do not establish distinct values.

“Destruction-like,” “dispatch-like,” “reference-like,” and “collection” describe inferred intent. Established behavior is limited to the visible reads, writes, calls, branches, and return calculation. The function does not visibly deallocate the input allocation, and dependency calls do not establish ownership transfer or release.

## Interface

### Argument and result

The function accepts one pointer to writable storage, called the **input object**.

The supplied representation uses 32-bit pointer and scalar accesses. All offsets below are byte offsets. Input-address calculations expressed in four-byte units have been converted to bytes.

An unresolved dependency receives input offset 24 and returns an address called the **anchor**. The **associated object** is the address 24 bytes before that anchor.

On normal completion, the function returns the associated-object address. It does not return an explicit status code, and the result is not established to equal the input pointer.

No null-input, alignment, bounds, or address-arithmetic overflow checks are visible.

### Visible storage

| Storage | Offset | Established use |
|---|---:|---|
| Input object | 0 | Initial 32-bit dispatch-like write |
| Input object | 24 | Address supplied to the anchor-producing dependency |
| Input object | 36 | Address supplied to the draining predicate and operation |
| Input object | 88 | Address supplied to F-a600efb7b035 |
| Input object | 96 | Address supplied to F-1bf9985dc121 |
| Associated object | 0 | Two later 32-bit dispatch-like writes |
| Associated object | 4 | Optional pointer controlling final cleanup |
| Associated object | 8 | Consistency field checked inside final cleanup |
| Associated object | 12 | Optional saved-state pointer |
| Associated object | 20 | Optional reference-like record pointer |
| Reference-like record | 0 | 32-bit word inspected after a decrement produces zero |
| Reference-like record | 4 | 32-bit count-like value decremented |
| Saved state | 4 | 32-bit value captured for later calls |
| Saved state | 8 | Optional collection pointer |
| Context | 28 | 32-bit argument supplied to F-77781c8e0ff9 |
| Context | 36 | Association compared with the associated object and conditionally cleared |
| Context | 40 | Association tested, used for dispatch, and compared with the associated object |

A **context** is an object returned by F-3fdd14ecd250. Repeated calls are not assumed to return the same address.

Four separate global locations supply 32-bit values:

- **Initial-install global:** read before the input-offset-0 write.
- **Intermediate-install global:** read after reference-record processing and before F-408b56b8b7c8.
- **Registration-argument global:** read after the preceding F-3fdd14ecd250 call and before F-77781c8e0ff9.
- **Final-install global:** read after saved-state processing and before final optional cleanup.

These labels identify storage roles, not known constant values. Earlier dependency calls may alter globals before later reads.

### Preconditions for normal execution

Every address actually dereferenced must designate valid, suitably aligned storage. Indirect-call objects must provide readable dispatch tables containing callable entries at the required byte offsets.

Pointers saved across calls must remain usable when later accessed. This includes the anchor-derived addresses, saved-state pointer, and stack-local traversal storage passed to dependencies.

The visible body supplies no synchronization or lifetime protection. Callback mutation, reentrancy, concurrent access, and aliasing can affect whether these preconditions remain satisfied.

## Behavior

### 1. Initial installation and component draining

The function reads the initial-install global and writes that 32-bit value to input offset 0. It then calls, in order:

1. F-389a28346612 with the input object.
2. F-1bf9985dc121 with input offset 96.
3. F-a600efb7b035 with input offset 88.

It repeatedly calls F-c6ba54ae5191 with input offset 36. A nonzero result ends the loop. A zero result causes one call to F-06db60ce9a71 with that same address, followed by another predicate call.

There is no visible iteration bound. If the predicate continues returning zero and each operation returns, this phase continues indefinitely.

Only after the loop ends does the anchor-producing dependency receive input offset 24.

### 2. Reference-like record processing

The function reads the record pointer at anchor minus 4, equivalent to associated-object offset 20.

A zero pointer skips this phase’s decrement and explicit slot clear.

For a nonzero pointer, the function reads the 32-bit value at record offset 4, subtracts one, writes the result back, and tests the computed result against zero.

- A nonzero result proceeds to clearing associated-object offset 20.
- A zero result enters additional checks before any such clear.

On the zero-result path, the function rereads the record pointer from its slot and dereferences that pointer to inspect its first 32-bit word. No new null check protects this dereference.

If that first word is nonzero, F-1b86b75ed2c5 is called, followed by F-feee9156a257, which the supplied representation marks as nonreturning.

Otherwise, the record-pointer slot is read again. If it is nonzero, F-feee9156a257 is called directly. If this check is zero, execution reaches the explicit slot clear.

Thus, with stable memory and an initially nonzero record pointer, a decrement from one to zero does not complete normally: a nonzero first word selects the first failure path; a zero first word leaves the still-nonzero pointer to select the second.

The decrement has no guard for an initially zero or negative-looking count. The stored quantity is 32 bits, not an unbounded integer. Exact signed-overflow semantics are not established.

### 3. Associated-object installation and registration-like call

After record processing, the function computes the associated-object address, reads the intermediate-install global, and writes that 32-bit value to associated-object offset 0.

It then calls F-408b56b8b7c8 with the associated object, followed by F-3fdd14ecd250.

After that context acquisition, it obtains two arguments for F-77781c8e0ff9:

- The 32-bit field at context offset 28.
- The 32-bit value read from the registration-argument global.

Both reads occur after the context call and before F-77781c8e0ff9. Their relative evaluation order is not established by the supplied representation.

F-77781c8e0ff9 receives the context-field value, the global-derived value, and the associated-object address, in that argument order. No null check precedes the context-field access.

The function next reads associated-object offset 12. A zero pointer skips all saved-state processing below.

### 4. Saved-state processing

For nonzero saved state, the function retains that pointer and captures its offset-4 value before calling F-a1a48bd3fbde with the associated object.

#### Predicate-controlled operations

If F-a1a48bd3fbde returns nonzero:

1. F-3fdd14ecd250 supplies a context whose offset-40 field is tested.
2. If that field is nonzero, another context is acquired.
3. An indirect call is made through the dispatch table of this later context’s offset-40 object, at table byte offset 72.
4. Associated-object offset 12 is reread, and the 32-bit value at offset 4 of that newly loaded state pointer is read.
5. Another context is acquired.
6. F-a5514b9f24f7 receives this context’s offset-40 field and the newly read state value.
7. After the optional indirect-call sequence, F-65528ded3450 receives the associated object and zero.
8. F-a82ec4b43e2f receives the originally captured saved-state offset-4 value.

The nonzero test in step 1 does not validate the object obtained through the next context call. There is no intervening null check on that later context’s offset-40 object.

The value supplied to F-a5514b9f24f7 can differ from the originally captured value. The state pointer and its contents may have changed during earlier calls. The reread state pointer is not null-checked before its offset-4 access.

If F-a1a48bd3fbde returns zero, this entire predicate-controlled sequence is skipped.

#### Collection processing

Regardless of the predicate result, the function next examines offset 8 of the originally saved state.

If nonzero, that field is read for a call to F-538044b67754, which receives:

- A stack-local traversal area of 20 bytes.
- The collection pointer.
- The 32-bit pattern `0xfffffffd`.

The field test precedes its use as an initialization argument. No dependency call intervenes, but the representation contains separate accesses.

F-ebd94610a438 is then called repeatedly with the traversal area and an output area holding two 32-bit pointer slots.

For each nonzero traversal result, the first output pointer is tested. A nonzero pointer causes an indirect call through its dispatch table at byte offset 4. A null pointer skips that call without ending traversal. The second output pointer is not inspected by this function.

A zero traversal result ends the loop. F-944b595e25e9 is then called with the traversal area.

After that call, saved-state offset 8 is reread and tested. If nonzero, the function invokes dispatch-table byte offset 4 through the current collection pointer.

There is no visible iteration bound, deduplication, or explicit clearing of saved-state offset 8. Repeated output pointers can produce repeated indirect calls. The meaning of `0xfffffffd`—including whether it represents a signed mode of −3—is unknown.

#### Context association processing

After collection processing, including when the collection field was zero:

1. A fresh context is acquired. If its offset-40 field equals the associated-object address, F-a82ec4b43e2f receives the originally captured saved-state value.
2. Another context is acquired. If its offset-36 field equals the associated-object address, a further context is acquired and zero is written to that context’s offset-36 field.
3. F-65528ded3450 receives the associated object and zero.
4. F-8349c252c168 receives the originally saved state pointer and zero.

The context whose offset-36 field is cleared need not be the context whose field passed the comparison.

F-a82ec4b43e2f can receive the same captured value twice: once on the predicate-controlled path and once on the later equality path. F-65528ded3450 likewise occurs twice when the predicate-controlled path is taken. There is no visible deduplication.

### 5. Final installation and optional cleanup

After saved-state processing, or after its absence is established, the function reads the final-install global and writes that 32-bit value to associated-object offset 0.

It then tests associated-object offset 4.

If zero, this phase performs no offset-8 check and returns the associated-object address.

If nonzero:

1. F-c719314a16bb receives the associated object.
2. Associated-object offset 4 is reread.
3. If still nonzero, an indirect call is made through the current referenced object’s dispatch table at byte offset 4.
4. Associated-object offset 8 is read and tested.
5. If nonzero, F-c8eefdf96f4b is called, followed by nonreturning F-feee9156a257.

If the offset-8 check is zero, the function returns the associated-object address.

## Side effects

Directly established effects are:

- Four global-value reads at the stages specified above.
- Three ordered 32-bit writes of global-derived words to object offset 0.
- A conditional 32-bit decrement in a reference-like record.
- Conditional clearing of associated-object offset 20.
- Conditional clearing of a context’s offset-36 field.
- Calls and indirect calls with potentially externally visible effects.
- Conditional nonreturning failure paths.

The three installed values are not established as constants or as unequal. Calls before later global reads may change the values subsequently installed or passed as arguments.

The body does not directly clear associated-object offsets 4, 8, or 12, or saved-state offset 8. Dependencies may do so.

Ownership and release semantics are **documented_only**. Dispatch-table offset 4 and F-8349c252c168 may be release-like, but neither their position nor their arguments prove deallocation, reference-count ownership, or pointer validity after the call.

No rollback or recovery path is visible. If a dependency does not return, later writes, finalization, and the normal return are not reached.

## Dependencies

The anchor-producing dependency is labeled **F-unknown-01** here solely as a document-local opaque identifier; its mapped identity is unavailable.

| Opaque ID | Established role |
|---|---|
| F-389a28346612 | Initial input-object operation |
| F-1bf9985dc121 | Operation on input offset 96 |
| F-a600efb7b035 | Operation on input offset 88 |
| F-c6ba54ae5191 | Drain-loop predicate |
| F-06db60ce9a71 | Drain operation after each zero predicate result |
| F-unknown-01 | Produces the anchor from input offset 24 |
| F-1b86b75ed2c5 | Precedes one record-related failure call |
| F-feee9156a257 | Marked nonreturning at failure sites |
| F-408b56b8b7c8 | Associated-object operation after intermediate installation |
| F-3fdd14ecd250 | Repeated context acquisition |
| F-77781c8e0ff9 | Receives context field, global-derived argument, and associated object |
| F-a1a48bd3fbde | Controls the first saved-state subpath |
| F-a5514b9f24f7 | Receives a context association and freshly read state value |
| F-65528ded3450 | Receives associated object and zero |
| F-a82ec4b43e2f | Receives originally captured state value |
| F-538044b67754 | Initializes traversal storage |
| F-ebd94610a438 | Advances traversal and supplies output |
| F-944b595e25e9 | Finalizes traversal after loop completion |
| F-8349c252c168 | Receives saved-state pointer and zero |
| F-c719314a16bb | Begins final optional cleanup |
| F-c8eefdf96f4b | Precedes final consistency failure |

Indirect dependencies are selected through dispatch-table byte offsets 72 and 4. Their identities, complete signatures, implicit receiver conventions, and effects are unknown. No dependency is assumed to preserve globals or own or release memory.

## Acceptance examples

These are conditional static scenarios, not executed tests. Unless a scenario states otherwise, dependencies return normally and relevant storage remains valid and stable.

1. **Immediate drain completion, absent optional state.**  
   A nonzero first drain predicate result and zero associated-object offsets 20, 12, and 4 produce no drain-operation calls, record decrement, traversal, or final optional cleanup. All three installation stages occur. The result is anchor minus 24.

2. **Two unsuccessful drain checks.**  
   Predicate results zero, zero, then nonzero produce exactly two F-06db60ce9a71 calls, interleaved with three predicate calls. Anchor acquisition follows.

3. **Globals change during earlier calls.**  
   If F-389a28346612 changes the intermediate-install global, that changed value is used when its later read occurs, provided no further change intervenes. The function does not snapshot all installation values on entry.

4. **Equal installed words.**  
   If all three installation-global reads yield the same bit pattern, the three writes still occur at their specified stages. No inequality requirement is imposed.

5. **Registration argument changes during context acquisition.**  
   If the relevant F-3fdd14ecd250 call changes the registration-argument global, the subsequent read observes the resulting value under stable-memory assumptions. No order is required between that read and the context-offset-28 argument read.

6. **Record count two becomes one.**  
   A nonzero record with count two receives count one; associated-object offset 20 is cleared; processing continues.

7. **Record count one becomes zero.**  
   With a stable nonzero record pointer, F-feee9156a257 is reached. A nonzero first record word causes F-1b86b75ed2c5 first. Intermediate installation is not reached.

8. **Saved-state predicate returns zero.**  
   The first predicate-controlled sequence is skipped. Collection processing, association comparisons, the unconditional F-65528ded3450 call, and F-8349c252c168 remain reachable.

9. **Null traversal element.**  
   A nonzero traversal result with a null first output pointer causes no element indirect call for that iteration. Traversal continues.

10. **Repeated captured-value operation.**  
    A nonzero saved-state predicate and a later context-offset-40 match produce two F-a82ec4b43e2f calls with the originally captured value.

11. **Final cleanup clears its pointer.**  
    If F-c719314a16bb clears associated-object offset 4, the later indirect call is skipped, but offset 8 is still checked.

12. **Nonzero offset 8 with initially zero offset 4.**  
    Final cleanup does not inspect offset 8 or take its failure path.

## Uncertainties

The following remain **documented_only**:

- Values and semantic identities of all four globals, including equality among installation values.
- Relative evaluation order of the two memory-derived arguments to F-77781c8e0ff9.
- Whether the associated object aliases the input object.
- The anchor-producing dependency’s identity and guarantees.
- Ownership, allocation provenance, dispatch semantics, and deallocation behavior.
- Context identity across repeated acquisitions.
- Callback mutation, reentrancy, concurrency requirements, and pointer invalidation.
- Traversal mode meaning and the second output pointer’s purpose.
- Signed-overflow and address-wrap behavior at arithmetic boundaries.
- Behavior if a dependency marked nonreturning unexpectedly returns.
- Exceptions, nonlocal exits, and external error-reporting mechanisms.

## Evidence

This contract is based on the supplied static function representation and opaque dependency mapping. Established claims concern visible access widths, offsets, global reads, writes, call order, conditions, loops, and the returned address.

Lifecycle, registration, reference-counting, and release terminology is inferred intent, not a verified dependency contract. The global accesses establish read locations and ordering constraints, not immutable values.

No runtime tests have been performed.
