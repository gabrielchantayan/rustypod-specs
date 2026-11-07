<!-- Trusted provenance: F-edd7b1e1d592; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-edd7b1e1d592

## Purpose

F-edd7b1e1d592 recursively processes a structure with three outgoing links per node, then conditionally updates shared accounting state and invokes two final dependencies.

The established traversal order is **second link, first link, third link, current node**. A null input ends that branch. After all three recursive invocations return, the current node is passed to F-b1ab9406483c. A zero result from that dependency ends current-node processing without entering the accounting path.

For a nonzero result, the function prepares access to shared state, checks a preceding metadata word, performs seven ordered accounting updates, and calls two dependencies with the captured state base.

**Inferred intent:** this resembles recursive teardown with allocator accounting. The supplied body does not establish that nodes are freed, that the candidate address equals the node address, or that either final dependency releases memory.

## Interface

### Input and node layout

The function accepts one pointer-like argument. Addresses and node fields are represented as 32-bit values.

A zero argument is accepted and returns immediately. A nonzero argument is treated as the address of storage containing at least these fields:

| Offset from node | Width | Use |
|---|---:|---|
| `0` | 32 bits | First recursive link |
| `4` | 32 bits | Second recursive link |
| `8` | 32 bits | Third recursive link |

Each link value is passed directly to a recursive invocation. There is no visible node-kind check, size argument, alignment check, or validation of the linked addresses.

### Output

There is no explicit return value. Normal completion returns control to the caller.

Observable effects include recursive calls, dependency calls, metadata reads, and writes to shared accounting storage. Invalid metadata invokes F-a817d8d58c10, which is represented as non-returning.

### Widths and arithmetic

The supplied representation uses:

- 32-bit addresses and pointer-like results.
- 32-bit metadata words and accounting storage.
- Unsigned 16-bit metadata indices.
- Four-byte strides for all indexed accounting entries.
- A 32-bit signed integer result from F-09dcb55e8e0a as represented at its use.

The masked accounting amount is unsigned and 32 bits wide. Its subtraction from a 32-bit state word produces an unsigned 32-bit result, modulo \(2^{32}\).

The two count decrements are expressed through signed 32-bit arithmetic. There is no explicit underflow check, but the source alone does not establish defined wrapping when decrementing the minimum signed value. Similarly, signed address calculations and the dependency-selected index multiplication do not justify a blanket guarantee of modulo address arithmetic.

Pointer validity, byte order, alignment requirements, and platform-specific arithmetic guarantees remain external preconditions.

## Behavior

### Null handling and traversal

A zero input returns without reading node storage or calling a dependency.

For a nonzero input, the function reads and processes links in this order:

1. Read the second link at offset `4`; recursively process its value.
2. After that invocation returns, read the first link at offset `0`; recursively process its value.
3. After that invocation returns, read the third link at offset `8`; recursively process its value.
4. After all three return, pass the original node address to F-b1ab9406483c.

These link reads are not an entry-time snapshot. An earlier recursive invocation may affect the value read for a later link. Normal execution therefore requires the original node and each subsequently accessed field to remain valid despite earlier recursive activity.

There is no visible cycle detection, visited set, deduplication, or depth limit. Shared substructures can be processed repeatedly. A cycle has no locally defined normal termination mechanism.

If any recursive invocation fails to return, processing does not advance to subsequent links or current-node handling.

### Candidate selection and state capture

F-b1ab9406483c receives the current node address and returns a 32-bit value interpreted as a candidate address.

If this result is zero, processing returns immediately. The three recursive invocations have already completed, but the current node causes no subsequent preparation call, prefix validation, direct accounting writes, or final dependency calls.

For a nonzero result:

1. F-2fd8826557a4 is called without explicit arguments.
2. A shared pointer-bearing location is read to capture the state base.
3. Candidate-prefix validation begins.

The state base is captured after F-2fd8826557a4 returns. It is not an entry-time state snapshot. No preservation of globals or candidate storage by that dependency is established.

Later operations use the captured base; they do not visibly reload it from the shared location. Changes to that location do not themselves replace the captured value, although dependencies may mutate or invalidate the storage to which it refers.

### Candidate prefix and validation

The candidate is interpreted as having an eight-byte prefix:

| Relative position | Width | Use |
|---|---:|---|
| Candidate minus `8` | 32 bits | Validation word and accounting-amount source |
| Candidate minus `4` | Unsigned 16 bits | First accounting-table index |
| Candidate minus `2` | Unsigned 16 bits | Second accounting-table index |

The function first reads the 32-bit prefix word and checks whether it is zero. If zero, it invokes F-a817d8d58c10.

Only if that first check is nonzero does it read the prefix word again and test its least significant bit. If that bit is set, it invokes the same rejection dependency.

The rejection dependency is represented as non-returning. No recovery value, rollback, or subsequent local cleanup is shown.

The checks do not establish a single immutable header snapshot. With changing storage, successful validation establishes that the first read was nonzero and the second read had bit zero clear. It does not establish that both reads saw the same value, or even that the second value was nonzero.

Validation does not check the metadata indices, allocation extent, accounting amount, or continued validity of the prefix.

### Entry into accounting

After successful validation, F-c0fc6d033caa receives the captured state base.

**Inferred intent:** this call may enter a protected state operation. Locking, mutual exclusion, and memory-preservation guarantees are not established.

The indices and accounting amount used for the updates below are read after this call. Values observed during validation cannot substitute for those later reads.

### Ordered accounting updates

The following seven updates occur in order, assuming each preceding operation completes normally. Each update reads a 32-bit state word and stores a 32-bit result. These are ordered operations, not an established atomic transaction.

#### First indexed total

The unsigned 16-bit index at candidate minus `4` is read. Its four-byte-stride entry is based at state offset `0x224`.

The prefix word at candidate minus `8` is then read and masked with `0x3ffffffc`. This produces the **retained amount** used by the first four updates.

The function subtracts that amount from the selected state word and stores the result.

The mask clears bits 0, 1, 30, and 31. Its result is a multiple of four in the inclusive range zero through `0x3ffffffc`.

#### First aggregate total

The word at state offset `0x30c` is read, reduced by the retained amount, and written back.

#### Second indexed total

The unsigned 16-bit index at candidate minus `2` is read. Its four-byte-stride entry is based at state offset `0xe0`.

The selected word is read, reduced by the retained amount, and written back. The accounting amount is not reread for this update.

#### Second aggregate total

The word at state offset `0x220` is read, reduced by the retained amount, and written back.

#### Selected count

F-09dcb55e8e0a is called without explicit arguments.

Its returned value selects a four-byte-stride entry based at state offset `0x310`. The selected 32-bit signed value is read, decremented by one, and written back.

No range check or nonnegative-index check is visible. The call may affect shared memory; no preservation guarantee is supplied.

#### Aggregate count

The 32-bit signed value at state offset `0x390` is read, decremented by one, and written back.

#### Final amount-based total

After the aggregate-count update, the function obtains:

- The 32-bit state word at offset `8`.
- A fresh 32-bit read of the prefix word at candidate minus `8`, masked with `0x3ffffffc`.

**The relative evaluation order of these two reads is unspecified by the supplied source.** Both belong to this final update, after the preceding aggregate-count update and before the final store.

The newly masked amount is subtracted from the state value, and the result is stored at state offset `8`.

This update does not reuse the retained amount from the first four updates. If the prefix changes in the intervening execution, this final subtraction may use a different amount. No validation is repeated on the fresh prefix value.

### Final dependency calls

After the seven accounting stores, the function calls:

1. F-cc6c9b4a0f68 with the captured state base and the address at candidate minus `8`.
2. F-e586bff64fb7 with the captured state base, if the preceding dependency returns.

The address supplied to F-cc6c9b4a0f68 is the prefix address, not the original candidate address.

No subsequent direct node, prefix, or state access is shown after these calls. Nevertheless, F-e586bff64fb7 still receives the captured state value even if the preceding dependency has changed globals or affected the referenced storage.

## Side effects

### Direct writes and aliasing

A normally completed nonzero-candidate accounting path performs seven direct 32-bit stores:

- Two metadata-indexed total updates.
- Two fixed-offset aggregate-total updates.
- One dependency-selected count update.
- One fixed-offset aggregate-count update.
- One final amount-based total update.

These are seven write operations, not necessarily seven distinct locations. No bounds or disjointness checks establish that indexed entries avoid fixed fields, other entries, the node, or the candidate prefix.

For example, first-table index `58` selects state offset `0x30c`. The first indexed update and the following aggregate update then subtract the retained amount from the same word successively.

If an accounting location aliases metadata, earlier stores can affect later metadata reads. In particular, the second index and final prefix amount are not protected by a complete metadata snapshot.

The body contains no explicit store directed at a node link or prefix field. That does not exclude indirect modification through aliasing or dependency effects.

### Ownership and lifetime

No ownership transfer is established.

- Recursive calls are not proven to release their input nodes.
- F-b1ab9406483c may return a distinct address; its relationship to the node is unknown.
- F-cc6c9b4a0f68 is not proven to free, coalesce, enqueue, retain, or invalidate the prefix-addressed object.
- F-e586bff64fb7 is not proven to unlock or finalize a specific ownership protocol.

Every reached access requires valid storage at that point. Calling order alone does not prove that an earlier dependency preserves memory needed later.

### Errors and partial progress

There is no explicit recoverable error result.

The prefix rejection path is the only explicit local error branch. Invalid pointers, recursion exhaustion, invalid computed addresses, signed arithmetic overflow, and dependency failures have no locally defined recovery.

No rollback is visible. If execution stops after some accounting stores, the function provides no local mechanism to reverse them.

## Dependencies

| Function ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-edd7b1e1d592 | Recursive processing of second, first, then third link | Cycle safety, recursion bounds, ownership |
| F-b1ab9406483c | Receives current node; returns candidate address or zero | Mapping rules, memory effects, ownership |
| F-2fd8826557a4 | Called before state-base capture | Initialization, synchronization, global preservation |
| F-a817d8d58c10 | Invoked on either prefix rejection condition; represented as non-returning | Diagnostics and termination mechanism |
| F-c0fc6d033caa | Receives captured state before accounting metadata reads | Locking, mutation, lifetime guarantees |
| F-09dcb55e8e0a | Returns the index for the table based at `0x310` | Range, sign constraints, side effects |
| F-cc6c9b4a0f68 | Receives captured state and candidate-minus-eight after accounting | Release semantics and ownership |
| F-e586bff64fb7 | Receives captured state after the preceding call returns | Unlocking, completion, memory effects |

## Acceptance examples

These are conditional static expectations, not runtime test results. Normal-completion examples require valid accesses, defined arithmetic, and returning dependencies unless stated otherwise.

1. **Null input.** A zero argument returns without reading node fields or invoking dependencies.

2. **Three null links and no candidate.** The recursive invocations receive zero in second-first-third order and return immediately. F-b1ab9406483c then receives the node. If it returns zero, current-node accounting is skipped.

3. **Mutation of a later link.** If second-link processing changes the original node’s first-link field and returns, the next recursive invocation uses the newly read first-link value. Capturing all three links on entry would not preserve this behavior.

4. **Zero validation word.** With a nonzero candidate whose first validation read is zero, F-2fd8826557a4 and state-base capture have already occurred. F-a817d8d58c10 is invoked without the second validation read. Accounting entry and direct accounting stores are not reached.

5. **Odd validation word.** If the first validation read is nonzero and the second read has its least significant bit set, the rejection dependency is invoked before accounting entry.

6. **Valid word with zero amount.** A stable prefix word of `2` passes validation and masks to zero. The five amount-based updates subtract zero. The selected and aggregate counts still decrement, and both final dependencies are reached if execution otherwise completes.

7. **Stable ordinary amount.** With a stable prefix word of `0x42`, the masked amount is `0x40`. If all seven destinations are distinct, relevant storage is unchanged by dependencies, each amount-based total starts at `0x100`, and both counts start at three, the totals become `0xc0` and the counts become two.

8. **Different final amount.** If the retained amount is `0x40` but the fresh prefix read for the final update masks to `0x80`, the first four amount-based updates subtract `0x40`; the update at state offset `8` subtracts `0x80`. This example imposes no order between that update’s state read and prefix read.

9. **Zero count.** Decrementing a signed count of zero produces minus one, represented by the 32-bit pattern `0xffffffff`. This does not establish wrapping at the minimum signed value.

10. **Aliased aggregate.** With first metadata index `58`, the first two accounting updates target the same word at offset `0x30c`. Absent other effects, they subtract the retained amount twice.

11. **Final-read ordering.** For the final update, an observation placing the state read before the prefix read, or the prefix read before the state read, is not excluded by the supplied expression. Both reads must precede that update’s store; neither ordering is a required contract.

## Uncertainties

The following remain **documented_only**:

- Semantic identities of nodes, candidate objects, tables, and counters.
- Concrete pointer ABI, byte order, alignment guarantees, and valid address regions.
- Valid ranges for both unsigned 16-bit indices and the dependency-selected signed index.
- Synchronization, reentrancy, and concurrent-mutation guarantees.
- Whether the input must be a tree, acyclic graph, or exclusively owned structure.
- Whether node, prefix, and accounting storage are guaranteed disjoint.
- Allocation, release, retention, and invalidation effects of dependencies.
- The rejection dependency’s diagnostics and termination mechanism.
- Platform guarantees for signed arithmetic overflow and address formation.

The final update’s two operand reads have unspecified relative order in the supplied source. Selecting one order would add a guarantee unsupported by the evidence.

## Evidence

Established claims derive from static inspection of the supplied function body: recursive-call sequencing, field offsets, branch conditions, short-circuit validation, dependency arguments, represented widths, unsigned metadata indices, masks, accounting-store order, and the represented non-returning rejection call.

The final subtraction contains both a state-word read and a prefix-word read without establishing their relative evaluation order. Its placement establishes their relationship to the preceding completed update and the final store, not an order between the reads themselves.

Allocator-like teardown and synchronization roles are inferred intent only. Dependency implementations, caller contracts, complete type definitions, and platform guarantees were not supplied.

**No runtime tests have been performed.**
