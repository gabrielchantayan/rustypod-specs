<!-- Trusted provenance: F-aed22785cb0d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aed22785cb0d

## Purpose

F-aed22785cb0d performs an object preparation sequence, searches a shared structure, and conditionally marks and notifies a selected entry.

The established behavior is narrower than a conventional “lookup and register” operation:

- An optional dependency call and an unconditional preparation call occur before searching.
- A byte in the supplied object is cleared regardless of whether an entry is subsequently selected.
- Selection depends on traversal calls, two possible rejection checks, and a possible reverse-order comparison.
- A selected entry whose state byte is already exactly one is accepted without notification.
- Otherwise, the state byte is set to one before two indirect notification calls.

The interpretation of the structure as a registry, and of the indirect calls as notifications, is inferred intent. Their external meanings are not established.

## Interface

### Inputs

The function accepts two 32-bit arguments.

1. **Object address.** The first argument is used as an address. The function directly accesses:
   - A 32-bit field at byte offset `0x2c`.
   - A one-byte field at byte offset `0x54`.

   This argument is also passed unchanged to F-622f6b869f2d.

2. **Opaque value.** The second argument is passed unchanged to F-622f6b869f2d. This function does not otherwise inspect it. Its interpretation as a pointer, identifier, flags, or another value is unknown.

The supplied representation uses 32-bit address-bearing fields and four-byte words. All offsets in this contract are byte offsets. Pointer-like values must be valid in the execution environment; this function provides no general pointer validation.

### Output

On an ordinary return, the result is a 32-bit value:

- **One:** the final F-f063d8ade9f7 check returned zero, and any required indirect calls returned.
- **Zero:** the final F-f063d8ade9f7 check returned nonzero.

A result of one does not imply that notifications occurred. A selected entry already marked with the byte value one follows the successful return path without either indirect call.

### Preconditions

For defined execution, every direct read and write reached on the chosen path must address accessible storage. This includes the object fields, shared structure fields, selected payload, and any indirect dispatch object.

The dependencies must accept the arguments described below and return normally for this contract’s ordinary-return behavior to apply. The function does not check allocation failure, catch exceptions, validate indirect targets, or provide rollback.

## Behavior

### 1. Prepare the supplied object

The function first reads the 32-bit field at object offset `0x2c`.

- If that field is nonzero, it calls F-d0df5eb7e117 with no explicit arguments.
- If that field is zero, it skips that call.

It then calls F-622f6b869f2d with the original object address and opaque second argument.

After that call returns, it writes zero to the single byte at object offset `0x54`. This is a one-byte write, not a four-byte field reset. No further direct access to the object is shown.

The conditional dependency therefore precedes the unconditional dependency, and both precede the byte clear. Their effects must not be assumed to preserve shared state.

### 2. Establish the search inputs

After clearing the object byte, the function captures an address from a shared global slot. This captured address supplies:

- A context address at offset `0x19`, passed to F-bcefa5957e76.
- A four-byte header-like field at offset `0x10`, read at later stages.

The function also initializes a local two-word record:

| Record offset | Width | Initial content |
|---|---:|---|
| `0` | 32 bits | An opaque value read from a fixed storage slot |
| `4` | 32 bits | One |

The first word is later supplied by value to both indirect calls. The two words are adjacent in the supplied stack layout. The evidence does not establish whether either comparison dependency consumes the second word or how it interprets the record.

The initial traversal address is obtained through a fresh read of the shared global address, then its field at offset `0x10`, then that referenced structure’s field at offset `0x04`.

The initial candidate is obtained through another fresh read of the shared global address and its field at offset `0x10`.

These are distinct reads. The supplied function does not explicitly force them to equal the previously captured address or header value. In a stable, single-threaded state they may refer to the same structure; that equality is not a general contract guarantee.

### 3. Traverse the structure

Traversal continues while the current traversal address is nonzero.

For each visited address, F-bcefa5957e76 is called with:

1. The captured structure address plus `0x19`.
2. The current traversal address plus `0x10`.
3. The address of the local two-word record.

Its 32-bit result determines the next read:

- **Zero:** read the four-byte address at current offset `0x08`; use it as both the next traversal address and the new candidate.
- **Nonzero:** read the four-byte address at current offset `0x0c`; use it as the next traversal address, leaving the candidate unchanged.

The branch-dependent link read occurs after the dependency returns. The function does not cache both links before the call.

The candidate update on the zero-result branch is important: the candidate becomes the loaded link value, not the address of the node just examined. It may consequently become zero. A conventional tree-search interpretation must not replace this observed rule.

No iteration bound or cycle detection is present. A cyclic structure, or dependency-driven link changes, can prevent termination.

### 4. Filter the candidate

When traversal ends, the function reads the four-byte field at offset `0x10` of the captured structure into a local reference value.

It calls F-f063d8ade9f7 with addresses of:

- The candidate storage.
- That local reference storage.

These are pointer arguments to local four-byte values, not merely copies of their contents. Their post-call values must therefore be used where the function subsequently reads them; preservation by the dependency is not established.

If this check returns nonzero, the function selects a freshly read value from offset `0x10` of the captured structure.

If it returns zero, the function:

1. Calls F-539f08165386 with the current candidate value.
2. Calls F-bcefa5957e76 with the captured context address, the local record address, and the value returned by F-539f08165386.

This second comparison reverses the roles of the local record and the entry-derived comparison argument relative to traversal.

- If the second comparison returns zero, the current candidate is selected.
- If it returns nonzero, a freshly read value from captured-structure offset `0x10` is selected instead.

The fallback value is not guaranteed to equal the earlier reference value. Dependencies have run between the relevant reads, and their state effects are unknown.

### 5. Check the selected value

The selected four-byte value is copied into another local storage location. The function separately rereads captured-structure offset `0x10` into a new local reference location.

It calls F-f063d8ade9f7 again, passing the addresses of these two local values.

- A nonzero result causes the function to return zero.
- A zero result permits entry-state processing.

The selected local value may have been modified through its address during this call. Subsequent payload access uses the value present after the call, not necessarily the originally selected value.

The function does not itself establish whether this dependency implements equality, end-of-structure detection, validity checking, or another predicate. Only its zero-versus-nonzero control effect is established.

### 6. Mark and conditionally notify

After a zero result from the final check, the function reads a four-byte payload address at offset `0x14` of the selected value. It then reads the single byte at payload offset decimal `100`, equivalent to `0x64`.

If that byte is exactly one, the function returns one without acquiring a dispatch object or making either indirect call.

For every other byte value:

1. It reloads the payload address from selected-value offset `0x14`.
2. It writes the byte value one at payload offset `0x64`.
3. It calls F-9d0ecc5f6adf.
4. It uses the returned address as an indirect-dispatch object.
5. It makes the first indirect call.
6. It makes the second indirect call.
7. It returns one if execution reaches the ordinary return.

Each indirect target is obtained from the dispatch object’s first four-byte field, followed by the function-pointer slot at table offset `0x58`. Both calls receive:

- The dispatch object address.
- An opaque value loaded from a fixed storage slot specific to that call.
- The current first 32-bit word of the local record.

The two calls use different fixed storage slots for their second arguments. Whether the loaded values are distinct at runtime is unknown.

The dispatch table and target are resolved again for the second call. The implementation does not establish that the first call preserves the dispatch table, the fixed storage values, or other shared state.

The byte is marked before dispatch-object acquisition and before either indirect call. There is no compensating clear if a later operation fails or does not return.

## Side effects

### Direct writes

The visible direct writes are:

- Zero to object offset `0x54`, after F-622f6b869f2d returns.
- One to the selected payload’s byte at offset `0x64`, only when its previously read value was not one and the final selection check returned zero.
- Initialization and subsequent assignment of local search and reference storage.

The function makes no direct allocation or deallocation call visible as such.

### Dependency effects and ownership

Dependencies and indirect targets may have additional effects not specified by the supplied function. No ownership transfer, retention policy, release obligation, or lifetime extension can be established merely from argument passing.

In particular:

- F-622f6b869f2d is not known to preserve the object or shared globals.
- Comparison and selection dependencies are not known to be pure.
- Passing local addresses does not establish that callees avoid retaining them.
- F-9d0ecc5f6adf is not known to allocate, borrow, or transfer ownership of its result.
- The indirect calls are not known to preserve the dispatch object.

The caller must satisfy any dependency-specific lifetime requirements, which remain undocumented here.

### Failure and partial progress

There is no explicit error result from the preparation or notification phases. Their return values, where present, are not used as success checks.

A zero return can occur after the object byte has already been cleared and after dependencies have executed. A failure after the payload mark can leave that mark set without completion of both indirect calls.

No synchronization or atomic read-modify-write is shown. Concurrent mutation and reentrant execution may affect selection and notification multiplicity.

## Dependencies

| Opaque ID or target | Established role and arguments | Result use |
|---|---|---|
| F-d0df5eb7e117 | Optional initial call; no explicit arguments | No result consumed |
| F-622f6b869f2d | Receives both original arguments | No result consumed |
| F-bcefa5957e76 | Receives context address and two comparison-related arguments | Zero versus nonzero controls traversal or candidate selection |
| F-f063d8ade9f7 | Receives addresses of two four-byte local values | Zero versus nonzero controls candidate filtering and final acceptance |
| F-539f08165386 | Receives the candidate value after the first check | Result becomes the third argument of the second comparison |
| F-9d0ecc5f6adf | Called after writing the payload mark; no explicit arguments | Supplies the dispatch object address |
| Indirect slot `0x58` | Called twice through the acquired object’s dispatch table | No result consumed |

No opaque ID is available for the indirect targets. Their concrete identities are undocumented.

## Acceptance examples

These examples are conditional contracts, not runtime observations. They assume all reached accesses are valid and all called dependencies return normally unless stated otherwise.

1. **Optional preparation is skipped.**  
   Given a zero 32-bit object field at `0x2c`, F-d0df5eb7e117 is not called. F-622f6b869f2d still receives both inputs, and the object byte at `0x54` is then cleared.

2. **Optional preparation precedes initialization.**  
   Given any nonzero 32-bit value at object offset `0x2c`, F-d0df5eb7e117 is called before F-622f6b869f2d. The byte clear occurs only after both calls return.

3. **Zero comparison replaces the candidate with a link.**  
   Given a nonzero traversal address and a zero traversal-comparison result, the next address and candidate both become the value loaded from offset `0x08` after that call. If that link is zero, traversal ends with a zero candidate; this function does not substitute the visited node.

4. **Nonzero comparison preserves the candidate.**  
   Given a nonzero traversal-comparison result, traversal follows the link at `0x0c`. The candidate remains unchanged by this branch.

5. **Final rejection returns zero.**  
   Given a nonzero result from the final F-f063d8ade9f7 call, the function returns zero without accessing the selected payload or acquiring a dispatch object. Earlier preparation effects remain.

6. **Exactly one suppresses notification.**  
   Given final acceptance and a selected payload byte equal to one, the function returns one without calling F-9d0ecc5f6adf or either indirect target.

7. **Other byte values trigger marking and both calls.**  
   Given final acceptance and a payload byte of zero, two, or `0xff`, the function writes one before acquiring the dispatch object. It then makes the two indirect calls in order and returns one if both return.

8. **Later failure does not roll back the mark.**  
   Given that marking has occurred and the first indirect call does not return, the second call and ordinary return are not reached. No rollback of the marked byte is shown.

## Uncertainties

The following are **documented_only**, not resolved by assumption:

- The meanings of the object fields and fixed storage values.
- Whether the shared structure is a tree, registry, or another container.
- Whether its header-like value represents a sentinel or end position.
- The comparison relation and interpretation of the two-word record.
- Whether dependency calls modify their pointer arguments, globals, structure links, or object lifetimes.
- The ownership and lifetime rules for the selected payload and dispatch object.
- The concrete indirect targets and meanings of their two call-specific values.
- Behavior under invalid pointers, arithmetic outside valid address ranges, concurrency, or dependency exceptions.

Address calculations use fixed byte offsets; no checked overflow or range validation is shown. The supplied representation establishes access widths but not a portable language-level contract for invalid address arithmetic.

## Evidence

**Established static evidence:** the supplied function establishes access widths and offsets, explicit call order, zero/nonzero branching, candidate-update rules, local-address arguments, the exact-byte mark check, mark-before-notification ordering, indirect slot offset, and ordinary return values.

**Inferred intent:** preparation followed by lookup, an already-active marker, and paired notifications. These descriptions aid orientation but do not strengthen the contract.

**Unknown:** dependency implementations, external data definitions, indirect target identities, ownership guarantees, synchronization, and runtime validity of any particular input.

No runtime tests have been performed.
