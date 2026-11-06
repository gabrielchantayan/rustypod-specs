<!-- Trusted provenance: F-bdfebf81fa35; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bdfebf81fa35

## Purpose

F-bdfebf81fa35 performs an ordered traversal of fixed-stride records. For each selected record, it counts one linked chain, submits a record-level formatting request, then walks a second linked chain and submits one formatting request per node after transforming a node field through a dependency.

The function periodically queries another dependency and terminates early when that dependency returns either of two specific values. It also executes two software interrupts before traversal and one after each completed record.

**Established static behavior:** record selection, chain traversal, dependency-call order, interrupt placement, and termination comparisons are visible in the supplied source.

**Inferred intent:** the formatting calls and periodic query suggest a listing operation with an opportunity for user cancellation. Actual output destinations, displayed text, and input semantics are not established by the source.

## Interface

### Invocation and result

The function has no explicit parameters and no explicit return value. Its inputs are implicit:

- A global pointer to signed record-count storage.
- A global pointer used as a record-table base.
- A global value used as the base of a 17-unit-per-index address calculation.
- Two fixed formatting descriptors.
- Memory reachable through record and node pointers.
- Results and side effects of its dependencies and software interrupts.

Normal termination provides no explicit indication of whether traversal exhausted the record count or stopped because of the periodic query.

### Representation and widths

The supplied representation uses four-byte integer and payload units. The following layout descriptions use byte offsets consistent with that representation:

- Record stride: **20 bytes**.
- First-chain head: **20 bytes** from the current record pointer.
- Second-chain head: **24 bytes** from the current record pointer.
- Chain link: the first four-byte field of each node.
- Second-chain payload: **32 bytes** from the node address.

The 20-byte stride does not imply that all fields accessed for an iteration fit within a nonoverlapping 20-byte object. In particular, the two head accesses occur at offsets 20 and 24. An implementation must preserve the observed addressing rather than “correcting” those offsets to fit a conventional record layout.

The record index, first-chain count, and query-scheduling counter are represented as signed integers. The source supports a 32-bit interpretation of these integers, but does not independently establish all target ABI rules, pointer widths, or overflow behavior.

### Required accessible storage

For defined traversal behavior, the following storage must remain valid whenever accessed:

- The captured record-count pointer and its signed count value.
- Every selected current-record location and its two chain-head fields.
- Every visited node’s link field.
- Every second-chain node’s payload field.
- Any storage consumed by the formatting dependencies.

The function contains no visible validation of these addresses.

## Behavior

### Initialization and captured state

Before either startup interrupt, the function captures:

1. The global pointer to record-count storage.
2. The global record-table base, retained for later record-level formatting arguments.

It then executes the software interrupt identified by **0x123456** twice, consecutively.

After those interrupts, it initializes:

- The query-scheduling counter to 1.
- The record index to 1.

It then reads the global record-table pointer again and uses this later value as the initial current-record pointer.

This distinction is observable. If an interrupt changes the global record-table pointer, the retained formatting base and the current-record pointer can refer to different storage. The function does not reconcile them.

The record-count pointer itself is not reloaded from its global source during traversal. The value stored through that captured pointer is reread at each iteration boundary.

### Iteration eligibility

At the start of each iteration, the function reads the signed count through the captured count pointer.

If that value is less than the current signed record index, the function returns immediately. No chain-head read, dependency call, or per-record interrupt occurs for that iteration.

The initial index is 1. Consequently, an initial count of zero or a negative count causes termination after the two startup interrupts, assuming those interrupts return normally and do not otherwise prevent execution.

The count is not snapshotted for the whole traversal. Changes to the pointed-to count storage can affect subsequent iteration eligibility.

### Counting the first chain

For an eligible iteration, the function resets a local signed chain count to zero. It reads the first-chain head from offset 20 of the current-record pointer.

For each non-null node:

- It increments the count once.
- It reads that node’s first field to obtain the next node pointer.

A null head produces a count of zero.

No formatting call, periodic query, or per-record interrupt occurs during this walk. A cyclic chain can therefore prevent the function from reaching its next query opportunity. An invalid node can cause an access failure; no recovery behavior is visible.

The count is subsequently passed by address to the record-level formatting dependency.

### Periodic query and early termination

After completing the first-chain count, the function compares the query-scheduling counter with 21.

When the counter is greater than 21:

1. The counter is reset to 1.
2. F-c7198d86625a is called with no visible explicit arguments.
3. Its integer result is compared with **0x58** and **0x78**.
4. Either matching result causes immediate return.

These values coincide with uppercase and lowercase X in ASCII-compatible encodings, but the actual meaning of the returned values is not established.

On this early-return path:

- The first chain for the current iteration has already been counted.
- The record-level formatting call has not occurred.
- The second-chain head has not been read by the visible traversal.
- The per-record interrupt and end-of-iteration increments do not occur.

Under ordinary counter progression, the first query occurs on eligible iteration 22. Subsequent queries occur on eligible iterations 43, 64, and so forth. Thus the spacing is 21 completed iterations between query opportunities, not 22.

This scheduling is based on the internal counter, not the current record-index value.

### Record-level formatting request

If execution continues, the function calls F-88996a971ccf with a fixed record-level formatting descriptor and four further arguments:

1. The address of the current local record index.
2. An address-like value formed from the current index multiplied by **17**, plus the global address-base value.
3. The address of the local first-chain count.
4. An address formed from the record-table base captured before the interrupts, plus **20 times the current index**, plus **16 bytes**.

For an unchanged index of 1, the fourth argument therefore refers to 36 bytes beyond the retained table base.

The 17-unit calculation and the 20-byte table-stride calculation are distinct. They must not be merged into one common stride.

All argument values for this call are established before the dependency begins executing. The source does not establish a precise evaluation order among the argument expressions.

The function passes addresses of two local signed integers. It does not visibly enforce read-only access to either integer. In particular, if the dependency modifies the record index through its supplied address, subsequent loop eligibility and index-derived arguments can change.

The current-record pointer is not recomputed from such an index change.

### Second-chain processing

After the record-level formatting call returns, the function reads the second-chain head from offset 24 of the current-record pointer.

For each non-null node, in order:

1. It reads the four-byte payload at node offset 32.
2. It passes that payload to F-a3fdaba172fb.
3. It takes the returned four-byte value and passes it to F-88996a971ccf with the second fixed formatting descriptor.
4. Only after both calls return does it read the current node’s first field to obtain the next node pointer.

This ordering matters when dependencies mutate shared storage. The next-link value is not saved before the transformation and formatting calls. A change to that link during either call can change which node is visited next.

A null second-chain head produces no transformation or item-level formatting calls.

The first and second chains are independent traversals. The function does not verify that the number of second-chain nodes equals the previously computed first-chain count.

### Completing an iteration

Once the second chain reaches a null pointer, the function:

1. Executes software interrupt **0x123456** once.
2. Increments the query-scheduling counter by 1.
3. Increments the current record index by 1.
4. Advances the current-record pointer by 20 bytes.
5. Returns to the signed record-count comparison.

The current-record pointer advances by one fixed stride regardless of changes to the record index made through a dependency.

There is no extra interrupt on final termination. The last completed record has its per-record interrupt; the following failed eligibility comparison returns directly.

## Side effects

### Directly established effects

The visible function:

- Executes the specified software interrupts.
- Reads global pointer and address-base values.
- Repeatedly reads the captured count storage.
- Reads record chain-head fields, chain links, and second-chain payloads.
- Updates its local counters, index, and traversal pointers.
- Calls the three opaque dependencies in the order described above.

No explicit allocation, deallocation, or write to caller-owned heap storage appears in the function itself.

### Dependency-mediated effects

The function exposes addresses of its local index and chain count to F-88996a971ccf. It also passes values derived from shared storage and, for one argument, a location within the retained table region.

The source does not establish whether any dependency:

- Writes to supplied storage.
- Modifies globals or chain links.
- Performs output or input.
- Allocates or releases memory.
- Retains supplied addresses.
- Returns normally.

Consequently, absence of direct writes in this function is not a guarantee that externally visible memory remains unchanged.

The function does not explicitly transfer ownership of any table, node, or returned value. Nor does it explicitly release them. Ownership and lifetime obligations imposed by dependencies remain documented_only unknowns. Passing a local address does not establish that retaining that address beyond the invocation is safe.

## Dependencies

| Dependency | Established interaction | Not established |
|---|---|---|
| F-88996a971ccf | Called once per continuing eligible iteration with a record descriptor and four arguments; called once per second-chain node with an item descriptor and a transformed four-byte value. Its return value is not used. | Exact descriptor bytes, formatting grammar, argument interpretation, output destination, mutation, ownership, and failure semantics. |
| F-a3fdaba172fb | Receives a four-byte payload from node offset 32. Its four-byte result is immediately supplied to the item-level formatting call. | Whether the payload or result denotes a pointer, handle, scalar, or another representation; lifetime and ownership of any referenced object. |
| F-c7198d86625a | Called at the scheduled query point. Results 0x58 and 0x78 cause immediate return; other results continue processing. | Input source, blocking behavior, error conventions, and side effects. |
| Software interrupt 0x123456 | Executed twice before traversal and once after each fully processed record. | Service meaning, register effects, memory effects, failure behavior, and whether execution always resumes. |

No dependency is assumed to preserve globals or reachable memory.

## Acceptance examples

These are conditional static acceptance scenarios, not runtime test results. Unless stated otherwise, dependencies and interrupts return normally without mutating traversal state, all addresses are valid, and arithmetic remains representable.

### Empty selection

If the count read at the first eligibility check is 0:

- Exactly two startup interrupts occur.
- No chain head is read for traversal.
- None of the three opaque dependencies is called.
- The function returns.

A negative initial count has the same outcome under the signed comparison.

### One record with empty chains

If the count remains 1 and both selected chain heads are null:

- Two startup interrupts occur.
- The first-chain count supplied by address is 0.
- One record-level formatting call occurs.
- No query or transformation call occurs.
- No item-level formatting call occurs.
- One per-record interrupt occurs.
- The next eligibility check returns.

### Unequal chain lengths

If the selected first chain contains three nodes and the second contains two:

- The record-level call receives an address to a count of 3.
- F-a3fdaba172fb is called twice.
- Each transformation call is followed by one item-level formatting call.
- No mismatch error or reconciliation is performed.

### Cancellation at the first query

If at least 22 iterations remain eligible and the first query returns 0x78:

- The first 21 iterations complete normally.
- Iteration 22’s first chain is counted.
- F-c7198d86625a is called once.
- Iteration 22 produces neither a record-level formatting call nor second-chain processing.
- No per-record interrupt occurs for iteration 22.

Returning 0x58 produces the same termination behavior.

### Continued query processing

If the first query returns any value other than 0x58 or 0x78:

- The current iteration proceeds to its record-level formatting call.
- Its second chain is processed normally.
- The scheduling counter advances from its reset value of 1 to 2 at completion.
- The next query occurs on eligible iteration 43, provided traversal continues.

### A link changes during node processing

If a dependency changes the current second-chain node’s link before returning, the next-node read observes the changed link, subject to the platform’s memory-access semantics. Acceptance must not require traversal to follow a link value captured before the calls.

### Startup changes the table base

If a startup interrupt changes the global table pointer:

- Chain-head traversal starts from the post-interrupt table pointer.
- The record-level table-derived argument still uses the base captured before the interrupts.

Acceptance must preserve this split rather than require both uses to follow the latest global pointer.

## Uncertainties

The following are documented_only:

- Exact formatting-descriptor contents and their interpretation.
- Concrete meaning of the record-table fields and 17-unit address calculation.
- Full ABI details, including pointer representation and address-arithmetic rules.
- Signed-overflow behavior for chain counts, index increments, and index multiplications.
- Software-interrupt service semantics.
- Dependency mutation, ownership, retention, error, and non-return behavior.
- Concurrency and synchronization guarantees for shared memory.

No explicit handling exists for null count storage, malformed pointers, cycles, concurrent mutation, arithmetic overflow, dependency failures, or non-returning calls. This is an absence of visible handling, not proof of a particular crash or error result.

For an acyclic first chain whose length fits the signed counter, counting terminates. A cycle has no visible termination mechanism. The second chain likewise has no visited-node tracking or traversal bound.

The query counter’s ordinary cycle remains small. That does not make the independently advancing record index or chain count safe from overflow.

## Evidence

This contract is based solely on static inspection of the supplied function representation and the supplied opaque dependency mapping.

Established evidence includes the two startup interrupts; pre-interrupt and post-interrupt pointer captures; signed eligibility comparison; 20-byte current-record stride; chain-head offsets 20 and 24; second-chain payload offset 32; query threshold and accepted termination values; argument-address calculations; dependency-call ordering; and post-record interrupt placement.

The listing and cancellation interpretation is inferred intent, not an established external interface. Descriptor contents and dependency implementations were not supplied.

No runtime tests have been performed.
