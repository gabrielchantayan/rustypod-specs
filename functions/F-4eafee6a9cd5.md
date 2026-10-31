<!-- Trusted provenance: F-4eafee6a9cd5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4eafee6a9cd5

## Purpose

F-4eafee6a9cd5 conditionally changes a globally selected structure so that the supplied record becomes its current record.

The established behavior is a sequence of pointer comparisons and 32-bit memory updates. If the supplied address already equals the current-record address, the function performs no writes. Otherwise, it modifies links associated with the supplied record, replaces the current-record address, clears the supplied record’s first link, and connects it to the previously current record.

**Inferred intent:** move an existing member to the front of a doubly linked structure. This interpretation fits the common nonterminal case, but it does not establish a complete list representation. In particular, the function unconditionally writes through the supplied record’s first link before checking its second link. A conventional null-terminated tail therefore cannot safely be assumed to be an acceptable input.

## Interface

### Input

The function accepts one address identifying a record containing at least two consecutive 32-bit fields:

| Offset | Width | Role in this contract |
|---|---:|---|
| +0 bytes | 32 bits | First link |
| +4 bytes | 32 bits | Second link |

Both fields are used as address-bearing values. The first link also receives the zero value during a successful nontrivial update.

The function also obtains a context address through one fixed external storage location. It accesses the following context fields:

| Offset | Width | Role in this contract |
|---|---:|---|
| +4 bytes | 32 bits | Auxiliary record address |
| +8 bytes | 32 bits | Current-record address |

These descriptions identify only the accessed fields. They do not establish complete object layouts or the meaning of any other bytes.

The supplied representation uses 32-bit integers and 32-bit address-bearing fields. Offsets are byte offsets. The evidence does not establish byte order, alignment rules, or the behavior of invalid or overflowing address calculations.

### Output

There is no explicit return value. Observable results are memory updates and normal completion, provided every executed access is valid.

There is no reported status, validation result, or recoverable error path.

### Preconditions

The fixed external storage location must be readable and must supply an address from which the context’s current-record field can be read.

If the input address equals the current-record address, the input record itself need not be dereferenced by this function. Consequently, equality can produce a no-write return even when that address would not otherwise identify a usable record.

For the non-equal path:

- The supplied record’s two fields must be readable and writable.
- The address obtained from its first link must support a writable 32-bit field at offset +4.
- If the second-link value observed at the branch is nonzero, the address used by that branch must support a writable 32-bit field at offset +0.
- If that second-link value is zero, the context field at offset +4 must be writable.
- The previous current-record address must support a writable 32-bit field at offset +0.
- The context’s current-record field must be writable.

These requirements describe accesses, not validation performed by the function. No membership, topology, nullness, or ownership checks are established.

## Behavior

### Initial observation and equality case

The function first reads the context address from the fixed external storage location. It then reads and retains the 32-bit current-record address at context offset +8.

It compares that retained address with the supplied address.

If they are equal, it returns without writing memory. In this case:

- Neither field of the supplied record is read.
- The auxiliary context field is not accessed.
- No neighboring record is accessed.
- No links are changed or repaired.

Equality is the sole explicit early-return condition.

### Non-equal update

If the addresses differ, the following access stages occur in order.

1. **Publish the supplied address as current.**  
   The function writes the supplied address to context offset +8. This happens before any input-record link is dereferenced.

2. **Update the record reached through the first link.**  
   The function reads the supplied record’s first and second links and writes the second-link value to offset +4 of the record addressed by the first link.

   The evidence establishes that the values needed for this store are obtained before the store occurs. It does not require a particular ordering between the two operand reads.

3. **Inspect the supplied record’s second link again.**  
   The function reads that field after the preceding store and branches on whether it is zero.

   - **Zero:** it reads the supplied record’s first link and writes that value to context offset +4.
   - **Nonzero:** it reads the supplied record’s link fields as needed and writes the first-link value to offset +0 of the record addressed by the second link.

   The branch condition is not necessarily based on the second-link value used in the previous stage. There is a later read, and aliasing can make the values differ.

4. **Clear the supplied record’s first link.**  
   The function writes the 32-bit zero value at input offset +0.

5. **Record the previous current address.**  
   The function writes the initially retained current-record address at input offset +4.

6. **Link the previous current record to the supplied record.**  
   The function writes the supplied address at offset +0 of the initially retained current-record address.

The function then returns without an explicit result.

The context address and original current-record address are retained for this operation. There is no second lookup through the fixed external storage location.

### Ordinary distinct-object interpretation

Under the additional assumptions that the relevant records and context fields do not unexpectedly overlap, memory is not concurrently modified, and all accessed addresses are valid, the non-equal path has a compact interpretation:

- The context’s current-record field becomes the supplied address.
- The original first-link target’s second link becomes the original second-link value.
- If the original second link is nonzero, its target’s first link becomes the original first-link value.
- Otherwise, the context’s auxiliary record address becomes the original first-link value.
- The supplied record’s first link becomes zero.
- The supplied record’s second link becomes the previous current-record address.
- The previous current record’s first link becomes the supplied address.

This interpretation is conditional. It is not a replacement for the ordered access contract when objects overlap.

### Boundary cases

**Zero first link on the non-equal path:** the function still attempts the offset-+4 write through that value. It contains no null-link check before this access.

**Zero previous current address on the non-equal path:** the function still attempts its final write through that address. It does not implement a separate empty-structure insertion case.

**Zero second link:** this selects the auxiliary-context update, but only after the unconditional write through the first link. A zero second link does not make a zero first link safe.

**Zero input address:** if the saved current-record address is also zero, the equality case returns without dereferencing the input. Otherwise, the function publishes the zero input as current before attempting to read its fields.

**Inconsistent links:** the function does not verify that neighbors point back to the supplied record, or that the supplied record belongs to the structure selected by the context.

### Aliasing and observation

The source does not establish that the context, supplied record, or linked records occupy disjoint storage. An earlier store can therefore affect a later read or be overwritten by a later store.

The ordered stages above govern such cases. Simplifying the behavior to a simultaneous reassignment of initially captured link values would not preserve all evidenced behavior.

In particular, only the context address and previous current-record address are retained across the operation. The input links are read again at later stages.

## Side effects

The equality path performs no writes.

The non-equal path performs six 32-bit store operations in the following order:

1. Context current-record field.
2. Second-link field reached through the input’s first link.
3. Either the context auxiliary field or a first-link field reached through the input’s second link.
4. Input first-link field.
5. Input second-link field.
6. Previous current record’s first-link field.

These are six store operations, not necessarily six distinct storage locations. Aliasing can cause multiple stores to affect the same word.

The update is not presented as atomic or transactional. The current-record field is changed before the links are rearranged. No lock, rollback, memory barrier, or synchronization operation is evidenced. Suitability for concurrent observation is unknown.

If an access is invalid, no recovery behavior is specified. On a platform that faults at such an access, earlier stores may already have occurred; this function supplies no explicit rollback path.

There are no explicit allocation, deallocation, ownership-transfer, I/O, logging, or callback operations. Rewriting a reference does not establish that ownership has changed. Storage lifetime and release obligations remain undocumented.

## Dependencies

No called functions are present in the supplied body. There are therefore no function dependencies to identify with additional opaque IDs.

The function depends directly on:

- A fixed external storage location containing the context address.
- Readable context state at offset +8.
- Writable context and record fields on the executed path.
- An execution environment in which the represented 32-bit address-bearing values can be used for the indicated accesses.

How the fixed storage location is initialized, who maintains the context, and what larger data-structure invariants apply are not established.

## Acceptance examples

These examples are conditional static expectations, not runtime test results. Unless stated otherwise, every referenced field is valid, objects are distinct, and no concurrent changes occur.

### Already current

The context’s current-record address equals the supplied address.

Expected:

- The function returns after comparing the addresses.
- No fields are written.
- Neither input link is read.
- Existing link inconsistencies are left unchanged.

### Nonzero second link

Initially:

- The current record is A.
- The supplied record is B.
- B’s first link addresses C.
- B’s second link addresses D.
- A, B, C, and D are distinct.

Expected:

| Location | Final value |
|---|---|
| Context current-record field | B |
| C second link | D |
| D first link | C |
| B first link | Zero |
| B second link | A |
| A first link | B |

The context auxiliary field is not written on this path.

### Zero second link with a valid first-link target

Initially:

- The current record is A.
- The supplied record is B.
- B’s first link addresses writable record C.
- B’s second link is zero.

Expected:

- The current-record field becomes B.
- C’s second link becomes zero.
- The auxiliary context field becomes C.
- B’s first link becomes zero.
- B’s second link becomes A.
- A’s first link becomes B.

The validity of C is required even though B’s second link is zero. This example does not establish whether C is an ordinary record or a sentinel.

### Null-terminated-tail assumption

Suppose B is not current and its first link is zero.

Expected from the static evidence:

- The current-record field is written with B first.
- The function then attempts a store at offset +4 from the zero first-link value.
- No null-tail handling precedes that store.

On a platform where that address is inaccessible, successful completion is not an acceptable expectation. The exact fault behavior is platform-dependent.

### Empty-current assumption

Suppose the saved current-record address is zero and the supplied address is nonzero.

Expected:

- The equality shortcut is not taken.
- Earlier link updates are attempted.
- The final store targets offset +0 of the saved zero address.

This is not an established empty-structure insertion operation.

## Uncertainties

The following remain **documented_only**:

- **Complete structure identity:** a doubly linked organization is inferred; its full layout and invariants are unknown.
- **Auxiliary field meaning:** its update may represent boundary maintenance, but a specific head, tail, or sentinel interpretation is not established.
- **Sentinel rules:** no sentinel allocation or validity rule is supplied.
- **Ownership:** no evidence assigns ownership or release responsibility to this function.
- **Aliasing restrictions:** intended callers may impose restrictions not visible here.
- **Concurrency:** no thread-safety or interrupt-safety contract is established.
- **Memory-model details:** byte order, alignment, invalid-access handling, address-overflow behavior, and device-memory semantics are unknown.
- **Caller guarantees:** membership, nonempty state, and reciprocal-link consistency may be required externally, but are not demonstrated.
- **Runtime validity:** no runtime tests have been performed.

## Evidence

Established static evidence consists of one supplied function body with:

- One explicit input address and no explicit return value.
- A context address loaded through fixed external storage.
- Context accesses at byte offsets +4 and +8.
- Two 32-bit input fields at byte offsets +0 and +4.
- One address-equality early return.
- An unconditional first-link-target write on the non-equal path.
- A later zero test of the input’s second link.
- Six ordered store operations on either completing non-equal branch.
- No function calls, validation routine, synchronization operation, or recovery path.

The move-to-front interpretation is inferred from those accesses. The memory-access behavior, equality shortcut, repeated link reads, and write ordering are established by the supplied static evidence. No runtime observations or trusted provenance identifier are asserted here.
