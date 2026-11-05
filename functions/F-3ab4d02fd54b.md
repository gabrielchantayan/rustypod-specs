<!-- Trusted provenance: F-3ab4d02fd54b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3ab4d02fd54b

## Purpose

F-3ab4d02fd54b performs an inclusive, range-limited traversal through an associated object. For each visited index, it obtains an entry, invokes a second operation when that entry exists, and then examines the entry’s referenced object. If a field in that referenced object is zero, the function submits the referenced object through an embedded destination.

The established behavior is the sequence of range negotiation, lookup, per-entry operation, post-operation inspection, and conditional submission. Describing the embedded destination as a collection, or the per-entry operation as a removal, would require dependency evidence that is not supplied.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

### Inputs

The function has four inputs, represented in the supplied material as 32-bit integer values:

| Input | Meaning established by use |
|---|---|
| A | Address of an enclosing object or structure. |
| B | Requested inclusive lower index. |
| C | Requested inclusive upper index. Also supplies the initial value of one callback output location. |
| D | Initial value of another callback output location, subsequently treated as a lower bound. |

A is used for address arithmetic and memory access rather than ordinary numeric computation. The exact declared source-language types are not established.

Index and bound comparisons use signed integer ordering in the supplied representation. The distinguished unavailable-bound value is signed 32-bit −1. The function does not generally reject negative indices: negative values other than −1 can participate in the range calculation and traversal.

### Relevant layout

The visible accesses establish these relationships:

| Location | Use |
|---|---|
| A + 0x30 | A 32-bit address-sized field identifying an associated object. |
| First address-sized field of that associated object | Address of a dispatch table. |
| Dispatch table + 0x0C | Callable slot used for initial bound negotiation. |
| A + 0x14 | Address passed directly to F-33f2303f9bf9 as an embedded destination. |
| First 32-bit field of a non-null lookup result | Address of a referenced object. |
| Referenced object + 0x40 | A 32-bit field tested for equality to zero. |

The offset at A + 0x14 is passed as an address. It is not first loaded as a pointer-valued field.

These accesses are consistent with a 32-bit address model. The evidence does not establish complete structure layouts, allocation sizes, alignment guarantees, or the interpretation of fields not listed here.

### Output and completion

The function has no established return value. Successful completion is observable through dependency calls and memory effects, not through a result returned to the caller.

There is no explicit error result, exception translation, retry, or recovery path in the visible behavior. An unavailable bound or a null lookup result is handled by ordinary control flow. Invalid addresses and failures inside dependencies are not visibly handled.

### Preconditions

For defined normal execution:

- A must support the required field accesses and embedded-destination address calculation.
- The initial associated object, its dispatch-table reference, and the callable slot must be valid.
- Each associated-object value subsequently supplied to a dependency must satisfy that dependency’s requirements.
- A non-null lookup result must remain readable after the per-entry operation.
- The object address read from that result must permit the field access at offset 0x40.
- Traversal must avoid an index-increment overflow unless the execution environment supplies a defined behavior for that case.

The function itself does not enforce these conditions.

## Behavior

### 1. Prepare writable callback values

Before negotiating bounds, the function prepares three local 32-bit storage locations initialized from B, C, and D respectively.

The location initialized from D and the location initialized from C are passed by address to the dispatch-table operation. Their argument order is:

1. Associated-object receiver.
2. Address of the value initialized from D.
3. Address of the value initialized from C.

Accordingly, D and C are initial callback values, not necessarily the bounds ultimately used. The operation can replace them through the supplied addresses.

The local storage initialized from B is not an output argument to this initial operation. It is later reused as submission storage.

### 2. Invoke the bound operation unconditionally

The function reads the associated-object field at A + 0x30, obtains its dispatch table, and invokes the slot at offset 0x0C.

This invocation occurs before any range rejection. Even a request whose initial lower index exceeds its initial upper index still reaches this operation.

Let L denote the callback-visible value initialized from D, as observed after the operation returns. Let U denote the callback-visible value initialized from C, as observed after the operation returns.

If L equals −1, traversal is skipped.

If L does not equal −1, U is used as the effective upper bound. Traversal is skipped if U equals −1.

The supplied representation also exposes a secondary machine-register value after the call. It does not determine a traversed interval: when L is −1, traversal is rejected; otherwise, U supplies the relevant upper bound. The secondary value is not an established second return value of the callback.

### 3. Restrict the requested interval

For non-sentinel bounds, the function first requires B ≤ U.

It then raises the requested lower index to L when B < L. It also requires L ≤ C. If either interval-overlap requirement fails, no lookup occurs.

The requested upper index is lowered to U when U < C.

Consequently, when traversal occurs, its first index is max(B, L), and its last index is min(C, U), with both endpoints included. The loop executes only while the current index is no greater than the resulting last index.

The function does not separately validate that either input interval is well ordered. Reversed intervals can therefore reach the loop setup but still produce no iterations.

The initial C and D values do not independently constrain the callback’s resulting bounds beyond their role as initial writable values. For example, a callback can replace an initial D of −1 with an ordinary lower bound.

### 4. Process each included index in ascending order

For each current index:

1. Read the associated-object field at A + 0x30.
2. Invoke F-36b67cc7bb38 with that associated object and the current index.
3. If its result is null, skip all remaining per-index work.
4. Otherwise, read the associated-object field at A + 0x30 again.
5. Invoke F-dc07b121bb37 with that newly read associated object and the same index.
6. After that call returns, read the first 32-bit field of the earlier lookup result to obtain an object address.
7. Read the 32-bit field at offset 0x40 from that object address.
8. If the field is zero, read the lookup result’s first field again, copy that value into local submission storage, and invoke F-33f2303f9bf9 with A + 0x14 and the address of that storage.
9. Advance to the next index.

The second operation occurs for every non-null lookup result, regardless of whether the subsequent field test succeeds. A nonzero field suppresses only the submission call.

The first field of the lookup result is not captured before F-dc07b121bb37. Any relevant mutation performed by that dependency can therefore affect the subsequent inspection. The function also does not refresh the lookup result by repeating the lookup after that dependency returns.

The associated-object field is reloaded for each dependency invocation that uses it. The receiver used for F-dc07b121bb37 need not equal the receiver used for the immediately preceding lookup if an intervening dependency changes that field.

### 5. Terminate

Traversal ends when the current index exceeds the clamped upper endpoint.

There is no early-success termination after a submission. Later indices are still processed.

Incrementing a signed 32-bit index at its maximum value cannot be assigned a portable ordinary-integer meaning from the supplied representation. An interval ending at 2,147,483,647 reaches an increment after processing that endpoint. Wrapping, trapping, optimization consequences, or other machine behavior are not established here.

## Side effects

The function directly writes temporary local storage. All visible persistent-state changes are delegated to the initial dispatch operation and the three named dependencies.

Important observable ordering constraints are:

- Bound negotiation precedes every lookup.
- Lookup precedes the per-entry operation at each index.
- The per-entry operation precedes inspection of the referenced object’s field.
- A qualifying submission precedes the next index’s lookup.
- Indices are visited in ascending order, subject to the overflow uncertainty.

No explicit allocation, release, ownership transfer, locking, or global-state access appears in this function. This does not establish their absence inside dependencies.

The function passes addresses of temporary local storage to the bound operation and to F-33f2303f9bf9. Those locations are not caller-owned result buffers and have only invocation-local lifetime. Whether a dependency retains an address is unknown; retaining it does not make the storage valid after this function returns.

The null check protects only the lookup result itself. There is no visible null check for the object address stored inside a non-null result.

## Dependencies

| Dependency | Established invocation contract | Not established |
|---|---|---|
| Associated object’s dispatch slot at offset 0x0C | Called once before range filtering, with the receiver and two writable 32-bit locations initialized from D and C, in that order. | Method identity, complete calling convention, mutation scope, lifetime behavior, and failure semantics. |
| F-36b67cc7bb38 | Called once per visited index with the current associated-object field and that index. Its result is treated as nullable addressable storage containing an object address. | Lookup semantics, ownership, result lifetime, and other side effects. |
| F-dc07b121bb37 | Called once for each non-null lookup result, before dereferencing that result. Receives a freshly read associated-object field and the same index. Its return value is not used. | Whether it removes, invalidates, replaces, releases, or otherwise mutates entries or objects. |
| F-33f2303f9bf9 | Called after a zero field test, with A + 0x14 and an address containing the then-read object address. Its return value is not used. | Whether it copies, retains, consumes, or mutates the supplied value or destination. |

No dependency is assumed to preserve global state, the associated-object field, entry contents, or ownership.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. They assume valid memory, normally returning dependencies, and no relevant arithmetic overflow.

1. **Callback replaces initial bounds.**  
   B = 2, C = 9, and D = −1. The bound operation writes L = 4 and U = 7. Lookups occur at indices 4, 5, 6, and 7. The initial D sentinel does not prevent traversal once replaced.

2. **Unavailable lower bound.**  
   The bound operation leaves or writes L = −1. It is still called once, but no lookup, per-entry operation, or submission follows.

3. **Unavailable upper bound.**  
   L = 0 and U = −1 after the operation. No traversal occurs.

4. **Disjoint ranges.**  
   B = 10, C = 20, L = 0, and U = 9. No lookup occurs because B exceeds U. Similarly, B = 0, C = 5, L = 6, and U = 10 produces no lookup.

5. **Single inclusive endpoint.**  
   B = 5, C = 5, L = 0, and U = 10. Exactly index 5 is visited.

6. **Null result at one index.**  
   The effective interval is 3 through 5, and the lookup at 4 returns null. Lookups still occur at all three indices. Index 4 triggers neither F-dc07b121bb37 nor F-33f2303f9bf9.

7. **Post-operation state controls submission.**  
   A lookup returns a valid entry whose referenced object initially has a nonzero field at offset 0x40. F-dc07b121bb37 changes that field to zero without invalidating the entry. The later zero test qualifies the object for submission. A change in the opposite direction suppresses submission.

8. **Negative ordinary index.**  
   B = −3, C = −2, L = −3, and U = −2. Indices −3 and −2 are visited. Negative values are not categorically rejected.

9. **Receiver mutation is observable.**  
   A lookup changes the field at A + 0x30 and returns a valid entry. The following F-dc07b121bb37 invocation receives the updated field value, not a cached original receiver.

## Uncertainties

The following remain **documented_only**:

- Domain meaning of the negotiated bounds and the field at offset 0x40.
- Whether the embedded destination represents a collection, queue, registry, or another abstraction.
- Ownership and lifetime of lookup results and their referenced objects.
- Whether F-dc07b121bb37 preserves the result sufficiently for the subsequent reads.
- Dependency failure behavior, callbacks, reentrancy, and global mutations.
- Concurrency guarantees and whether the repeated entry-field reads can observe different values.
- Exact behavior of signed index overflow and invalid address arithmetic.
- Full ABI details and original declared types.

The inferred intent is to process the overlap of a requested interval and an associated object’s reported interval, then submit selected objects. The observable contract does not depend on assigning stronger domain meanings.

## Evidence

The supplied static body establishes the call order, layout offsets, sentinel comparisons, signed range comparisons, inclusive ascending traversal, null-result skip, post-operation field inspection, repeated receiver reads, and conditional submission through temporary storage.

The dependency map establishes the opaque identities used above. Dependency implementations, surrounding type definitions, and runtime observations were not supplied.

No runtime tests have been performed. The acceptance examples are consequences of the visible control flow under their stated preconditions, not reports of observed executions.
