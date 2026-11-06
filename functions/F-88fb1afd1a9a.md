<!-- Trusted provenance: F-88fb1afd1a9a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-88fb1afd1a9a

## Purpose

F-88fb1afd1a9a looks up an object using an address derived from its first argument. A null lookup result bypasses all subsequent processing. A non-null result triggers an ordered sequence of indirect object calls, operations on local argument storage, a direct overwrite of dependency-returned storage, two manager-field decrements, submission of a two-word result, and final dependency calls.

The function returns a 64-bit value assembled from two local 32-bit words initially populated by its first and second arguments. Dependencies receive addresses into that local storage on the object-present path, so the returned words need not retain their initial values.

The supplied static representation establishes this sequence and its explicit data accesses. It does not establish domain meanings such as object removal, reference counting, resource destruction, notification, or transaction completion.

## Interface

### Inputs

The function accepts four arguments:

| Argument | Width and representation | Established use |
|---|---|---|
| First | 32-bit integer/address-bearing value | Initializes the first local word; participates in an address calculation adding 24 bytes. |
| Second | 32-bit opaque word | Initializes the second local word; also supplies the original second argument to F-494c11edf385. |
| Third | 32-bit opaque word | Initializes the third local word. |
| Fourth | 32-bit opaque word | Initializes the fourth local word. |

Address-bearing values and object-table accesses use the supplied 32-bit representation. A portable interface for wider pointers is not established.

### Local storage

Before lookup, the function initializes four consecutive 32-bit local words in argument order:

| Byte displacement | Initial content |
|---|---|
| 0 | First argument |
| 4 | Second argument |
| 8 | Third argument |
| 12 | Fourth argument |

For this contract:

- **Region A** starts at displacement 0 and includes the first two initialized words.
- **Region B** starts at displacement 8 and includes the last two initialized words.

These labels identify storage locations, not established object types or dependency access bounds. Passing either region’s address does not prove that a dependency accesses exactly eight bytes or cannot affect the other region.

### Return value

The return is a 64-bit composition:

- Bits 0–31 contain the final first local word.
- Bits 32–63 contain the final second local word.

The low word is not sign-extended into the high word.

If lookup returns null, no local-region address has been passed to a dependency, and the return contains the original first and second argument bit patterns.

If lookup succeeds, the return is assembled after the entire object-present sequence. Mutations made through exposed local addresses can therefore affect it. No separate preserved copy of the return words is used.

### Preconditions for normal completion

The function visibly enforces none of the following:

- The first argument’s addition of 24 and the resulting address must be valid for the dependencies receiving it.
- A non-null lookup result must support the required object-table reads and indirect calls.
- F-b9acea4b06ad must return storage permitting a 32-bit write at its beginning.
- Both global manager-reference-cell evaluations, and the later use of the saved cell, must yield valid access chains.
- The manager fields at offsets 1064 and 1060 must permit 32-bit read-modify-write operations.
- The manager field at offset 48 must permit a 32-bit read.
- The object must remain usable for the second indirect call despite all intervening calls.
- Dependencies receiving local addresses must obey whatever lifetime and access constraints their separate contracts require.

These are conditions for safe normal completion, not validation performed by this function.

## Behavior

### 1. Initialize local words and look up the object

The four local words are initialized from the four arguments.

F-0d1bcade3bad is then called with the original first argument plus 24 bytes. The calculation uses the original argument, not a later value of the first local word.

The returned object pointer controls the only explicit branch:

- If null, the function proceeds directly to return-value composition.
- If non-null, the following sequence executes.

Lookup may itself have side effects. Its implementation is not supplied.

### 2. Begin object-present processing

F-bfade2068003 is called with 27 and 0, in that order.

The function then reads the object’s first 32-bit slot as a method-table reference and invokes the callable entry at table byte offset 16. The explicit arguments are:

1. The address of region A.
2. The lookup object pointer.

The object pointer is the second explicit argument. The call’s return value is not used.

The method may affect reachable local storage, object state, or globals; the caller supplies no evidence restricting such effects.

### 3. Obtain and overwrite intermediate storage

F-b9acea4b06ad is called with:

1. The address of region B.
2. The address of region A.

Its result is treated as an address of writable 32-bit storage. After that call returns, the function reads a dedicated global 32-bit word and writes its value to the first word at the returned address.

There is no null check before this write. No additional initialization of the returned storage is visible. Its allocation origin, full layout, and relationship to the local regions are unknown. In particular, disjointness from region A or region B is not established.

F-f13a263e790a is then called with that same returned address and 0. Its result, if any, is unused.

F-289ec4311c64 is subsequently called twice:

1. With the address of region B.
2. With the address of region A.

The order is established. Cleanup or destruction semantics are not.

### 4. Call the object dependency and evaluate manager references

F-6c19e1d86be0 is called with the lookup object pointer. Its result is unused. The function later accesses that same object pointer, so releasing or invalidating the object cannot be assumed to be safe.

After this call, the supplied representation contains **two separate evaluations of the global manager-reference-cell pointer**:

1. It reads the global pointer and saves that cell address for later use.
2. It independently reads the same global pointer again, then dereferences the cell address obtained by this second evaluation to obtain the manager base used for the decrements.

The second operation is not represented as a dereference of the saved cell address. The two cell addresses coincide only if both global evaluations produce the same value.

Using the manager base obtained through the second global evaluation, the function performs these 32-bit operations:

1. Read the field at byte offset 1064, subtract one, and write the result back.
2. Read the field at byte offset 1060, subtract one, and write the result back.

Both updates use the same captured manager base. The higher-offset field is updated first.

There is no visible zero guard, saturation, synchronization, rollback, or intervening dependency call between these updates. For subtraction results representable as signed 32-bit integers, each field decreases by one. The supplied representation does not establish executable behavior for subtraction from the minimum signed 32-bit value.

The separate global evaluations are source-level evidence. Whether the executable performs two distinguishable loads, combines them, or permits external mutation to distinguish them remains unresolved.

### 5. Obtain the object result and submit its words

After both manager updates, the function again reads the object’s first slot as a method-table reference. It invokes the callable entry at byte offset 172 with the object pointer as its sole explicit argument.

This call produces a 64-bit result. The method-table reference is read afresh; the earlier table reference is not visibly reused.

After this call returns, the function dereferences the **saved cell address from the first global evaluation** described above. That dereference obtains a manager base, from which the function reads the 32-bit field at byte offset 48.

F-957fde6b6d36 is called with:

1. The manager field read at offset 48.
2. Bits 32–63 of the indirect result.
3. Bits 0–31 of the indirect result.
4. The value 0.

The high result word precedes the low result word. Each occupies a 32-bit argument representation; its semantic interpretation belongs to the dependency contract.

Three distinctions matter:

- The decremented manager comes through the **second** global-pointer evaluation.
- The later offset-48 read comes through the **saved cell from the first** evaluation.
- The contents of that saved cell are read after the second indirect method, not captured when its address was saved.

Consequently, the decremented manager and the submission manager need not be the same. Even when the initial global evaluations yield the same cell address, a later change to that cell’s contents can redirect the offset-48 read. A later change only to the global cell-pointer value does not redirect the saved-cell dereference.

### 6. Perform final calls and return

F-494c11edf385 is called with:

1. The original first argument plus 24 bytes.
2. The original second argument.

These arguments are not read from the potentially modified local words. The displaced-address expression is evaluated again at this call site.

F-85722f62e20a is then called with 27 and 1000.

Finally, the function composes its 64-bit return from the current first and second local words.

No dependency result is checked for failure beyond the initial lookup’s null/non-null branch.

## Side effects

### Directly established effects

Every invocation initializes the four local words and calls the lookup dependency.

On the object-present path, the function additionally:

- Exposes local-region addresses to dependencies.
- Overwrites one 32-bit word at the address returned by F-b9acea4b06ad.
- Performs two ordered manager-field decrements.
- Reads object method-table entries at two different stages.
- Separately evaluates the global manager-reference-cell pointer twice.
- Later dereferences the cell address saved by the first evaluation.
- Calls the dependencies in the order specified above.

A null lookup skips all these object-present operations, but does not imply a side-effect-free lookup.

### Ownership and lifetime

No allocation, ownership acquisition, ownership transfer, or release is established by this function alone.

Specifically:

- The lookup object is not proven owned or borrowed.
- The intermediate returned storage is not proven newly allocated.
- F-f13a263e790a is not proven to consume that storage.
- F-289ec4311c64 is not proven to destroy either region.
- F-6c19e1d86be0 is not proven to release or preserve the object.

Any dependency contract must remain compatible with the caller’s subsequent accesses, including the later object call and final reads of region A.

Retention of local addresses by dependencies is unknown. Valid use of those addresses after this function returns is not established.

### Errors and partial completion

The function has no visible error-status return, exception handler, compensating action, or rollback.

An invalid intermediate address is still used for the direct write. Invalid manager or object access chains likewise have no visible fallback.

If a dependency fails to return normally, subsequent operations are not established to occur. Direct writes already performed have no visible undo mechanism. The two manager updates are separate operations, not an established atomic transaction.

## Dependencies

| Dependency | Caller-visible role |
|---|---|
| F-0d1bcade3bad | Receives the original first argument plus 24; returns the branch-controlling object pointer. |
| F-bfade2068003 | Receives 27 and 0 before the first indirect call. |
| Indirect site at table offset 16 | Receives region A’s address, then the object pointer. |
| F-b9acea4b06ad | Receives region B’s address, then region A’s address; returns storage overwritten by the caller. |
| F-f13a263e790a | Receives that storage address and 0. |
| F-289ec4311c64 | Receives region B’s address first, then region A’s address in a separate call. |
| F-6c19e1d86be0 | Receives the object before manager processing and the later indirect call. |
| Indirect site at table offset 172 | Receives the object; produces a 64-bit result. |
| F-957fde6b6d36 | Receives the saved-cell-derived manager field, result high word, result low word, and 0. |
| F-494c11edf385 | Receives the original displaced first argument and original second argument. |
| F-85722f62e20a | Receives 27 and 1000 as the final dependency call. |

No opaque target IDs are supplied for the indirect sites. Their targets depend on the table read at each call. No dependency is assumed to preserve globals, local contents, object state, or ownership.

## Acceptance examples

These are conditional static-contract examples, not executed tests.

### Null lookup

Given first and second argument bit patterns 0x00001000 and 0x89ABCDEF, if lookup returns null:

- F-0d1bcade3bad receives 0x00001018, assuming valid address arithmetic.
- No object-present operation occurs.
- The return is 0x89ABCDEF00001000.

Lookup side effects remain unspecified.

### Modified return words

If lookup succeeds and region A’s final words are 0x10203040 and 0x50607080, normal completion returns 0x5060708010203040.

F-494c11edf385 still receives the original displaced first argument and original second argument, regardless of those local changes.

### Stable global pointer, changed cell contents

Suppose both initial global evaluations yield the same cell address. Its manager has values 8 and 3 at offsets 1064 and 1060.

The function writes 7 and 2, in that order. If the second indirect method changes that cell to reference another valid manager, submission uses the replacement manager’s offset-48 field. The completed decrements remain associated with the earlier manager.

### Different results from the two global evaluations

At the supplied representation’s level, suppose the first evaluation yields one valid cell and the second yields a different valid cell:

- The first cell address is saved.
- The manager reached through the second cell supplies both decremented fields.
- After the second indirect call, the first cell is dereferenced for the submission manager.

This illustrates the represented access distinction. Whether a concrete execution can distinguish the two global evaluations is **documented_only**.

### Result-word order

If the second indirect method returns 0x1122334455667788, F-957fde6b6d36 receives 0x11223344 as its second argument and 0x55667788 as its third argument. Its fourth argument is zero.

These are numeric word positions, independent of memory byte order.

### Boundary and invalid states

A manager field containing zero is written as negative one under ordinary representable signed subtraction; there is no zero guard.

A null address from F-b9acea4b06ad is still subjected to the first-word write. No graceful failure result is specified.

Signed-minimum subtraction and overflowing displaced-address calculations are **documented_only**.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of the object, globals, manager, and fields.
- Concrete types, permitted access extents, and aliasing relationships of the local regions and intermediate storage.
- Dependency mutation, ownership, retention, failure, and nonlocal-control-flow behavior.
- Whether the calls using 27 express any shared protocol.
- Object lifetime and table changes across intervening dependencies.
- Signed arithmetic overflow and address-overflow behavior.
- Executable realization of the two source-level global-pointer evaluations.
- Whether external mutation can distinguish those evaluations.
- Thread-safety, reentrancy, and validity under concurrent mutation.
- Individual argument-expression read ordering not established by the supplied representation.
- Memory byte order.

Cleanup, release, removal, and transaction interpretations are possible inferred intent, not acceptance requirements.

## Evidence

This contract uses only the supplied static representation and opaque dependency mapping. Dependency implementations, indirect targets, global-state definitions, and executable-level load behavior were not supplied.

Established static evidence includes argument widths, local word layout, branch condition, explicit call order and arguments, direct overwrite, ordered manager offsets, two separate global-pointer evaluations, later saved-cell dereference, result-word ordering, and final return composition.

The manager access description preserves the distinction between saving the first global-pointer value, independently obtaining the decrement manager through the second evaluation, and later dereferencing the saved cell. Equality of the resulting managers requires additional conditions; it is not unconditional behavior.

No runtime tests have been performed.
