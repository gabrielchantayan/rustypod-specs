<!-- Trusted provenance: F-42dffdf1b496; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-42dffdf1b496

## Purpose

F-42dffdf1b496 performs a bounded search through a linked sequence selected from a table associated with an input object. It returns the first visited node whose comparison dependency returns zero. It returns a null pointer when the table is absent, the selected sequence ends before a match, or the search bound is exhausted.

The comparison dependency is selected by the object’s leading byte: value 3 selects one target; every other byte value selects another.

**Established static behavior:** table selection, callback selection, argument positions, zero-as-match handling, traversal, and the returned node identity are visible in the supplied source representation.

**Inferred intent:** the selected table entry appears to describe a bucket containing a count and a linked-list head. “Bucket,” “node,” and “comparison” are descriptive terms, not established application-level types.

**Unknown:** the meanings of the two query words, the node payload words, and the comparison routines are not supplied. Equality, ordering, string interpretation, allocation, and memory-release behavior must not be inferred.

## Interface

### Inputs

The function accepts four inputs:

| Input | Contract |
|---|---|
| Object address | Address of storage containing at least the fields accessed at byte offsets 0 and 16 when their branches are reached. |
| First query word | An uninterpreted 32-bit value forwarded to the comparison dependency. |
| Second query word | An uninterpreted 32-bit value forwarded to the comparison dependency. |
| Entry index | An integer used to select an eight-byte table entry. No range check is visible. |

The supplied representation uses four-byte words for table fields, node fields, and query values. It also reconstructs addresses through integer-sized fields. This supports a 32-bit address model for the described layout, but does not independently establish the target ABI, byte order, alignment requirements, or machine overflow rules.

### Object layout

Only these object fields are used:

| Byte offset | Access | Meaning |
|---|---|---|
| 0 | One byte | Comparison-target selector. Exactly 3 selects the alternate target. |
| 16 | Four-byte address-bearing field | Table base. Zero means no table. |

No requirements on other object bytes are established.

### Table-entry layout

The selected entry starts at the table base plus eight times the entry index.

| Entry-relative byte offset | Access | Meaning |
|---|---|---|
| 0 | Four-byte integer | Traversal counter. |
| 4 | Four-byte address-bearing field | Initial node address. |

The representation interprets the counter as an integer. For ordinary nonnegative counter values, it bounds the number of comparisons by that value.

### Node layout

For each node that reaches the comparison call:

| Node-relative byte offset | Access | Meaning |
|---|---|---|
| 0 | Four-byte address-bearing field | Successor node, read only after a nonmatching comparison returns normally. |
| 12 | Four-byte word | First comparison argument. |
| 16 | Four-byte word | Second comparison argument. |

Bytes at offsets 4 through 11 have no established role in this function.

A node need not have a readable successor field for this function to return it as a match: the successor is not read on that path. Conversely, every nonmatching comparison followed by normal continuation requires a readable successor field, including the last permitted comparison.

### Output

The result is either:

- The exact address of the first visited node whose comparison result is zero; or
- A null pointer.

The function does not return a copy, a payload address, a predecessor, a comparison result, or a count of visited nodes.

No ownership transfer, reference-count adjustment, or lifetime extension is visible. A returned node is an existing address obtained from the selected sequence. Whether it remains valid after return depends on external state and dependency behavior.

### Preconditions for defined ordinary use

Callers must provide readable storage and valid address arithmetic for every access reached by the chosen path. In particular:

- The object must support the table-field read.
- A nonzero table base must identify a valid selected entry.
- The index must select an accessible entry without relying on unspecified overflow behavior.
- Each compared node must support its two payload reads.
- Each normally returning nonmatching comparison must leave its current node’s successor field readable.
- The selected comparison target must be callable with the four forwarded words.

The function does not validate these conditions.

## Behavior

### 1. Table presence gates all later work

The table field at object offset 16 is tested for zero.

If it is zero, the function returns null without selecting an entry, examining the object’s selector byte, reading node data, or invoking a comparison dependency.

On the nonzero path, the table field is used to derive the selected entry address. The source representation contains a further use of that field for address construction; it does not establish an atomic snapshot of mutable object storage.

### 2. Entry selection is unchecked

The entry address is formed using the supplied index and an eight-byte stride.

There is no visible check for:

- Negative indices;
- Indices beyond the allocated table;
- Multiplication or address overflow;
- A table-size field;
- Alignment;
- Consistency between a count and its list.

A negative index is not a special “not found” request. It participates in address calculation. Its safety depends on the resulting address and the target’s arithmetic behavior.

The initial node address is obtained from the selected entry before traversal begins.

### 3. The comparison target is selected once

The default comparison-target slot is consulted. If the object’s leading byte equals 3, the alternate slot supplies the selected target instead.

Every selector byte other than 3 follows the default path; no other selector cases are visible.

Selection occurs before iteration and is retained for the entire invocation. Consequently, a comparison dependency changing the object’s selector or either target slot does not, by itself, cause a later iteration to select a different target.

The source-level order places the initial head read before target selection, and target selection before the initial counter read.

### 4. The counter is decremented before deciding whether to compare

Before each potential comparison, the local counter is reduced by one. Comparison proceeds only when:

- The reduced value is not minus one; and
- The current node address is nonzero.

For an initial nonnegative counter \(N\), at most \(N\) comparisons occur.

Consequences include:

- Counter zero causes no comparison, even with a nonnull head.
- A positive counter with a null head causes no comparison.
- An earlier match stops immediately.
- An earlier null successor stops traversal.
- The stored counter is not written back.

The counter is captured before iteration. Later changes to the table entry’s stored counter do not change that captured bound.

Negative initial counters are not explicitly rejected. They do not have the ordinary nonnegative-count meaning. Their behavior is constrained by integer-width and overflow uncertainties described below.

### 5. Comparison arguments are positional and uninterpreted

For a visited node, the selected dependency receives four arguments in this order:

1. The four-byte word at node offset 12;
2. The four-byte word at node offset 16;
3. The first query word;
4. The second query word.

The source establishes argument positions, but not the relative evaluation order of the two node-field reads.

The function makes no visible conversion of the query words and does not inspect the node payload words itself.

A comparison result equal to zero causes immediate return of the current node address. Any nonzero result takes the same continuation path. Positive and negative nonzero results do not select different behavior.

### 6. A miss follows the successor after the dependency returns

After a nonzero comparison result, the function reads the successor field at offset 0 of the current node and uses that address for the next loop condition.

This read occurs after the comparison call. Therefore:

- A dependency’s modification of that successor field can affect traversal immediately.
- The successor is not a pre-call snapshot.
- The function does not visibly protect against the dependency invalidating the current node.
- A successor read still occurs after the final allowed comparison returns nonzero, before the next counter check ends the search.

On a zero result, the function returns immediately without reading the successor.

The entry’s original head is not reread during traversal. Mutating that stored head during a comparison does not redirect the current traversal.

## Side effects

The function contains no explicit writes to:

- The object;
- The table;
- The selected entry;
- Nodes;
- The comparison-target slots.

It also contains no explicit allocation, release, insertion, removal, reference-count operation, or persistent counter update.

Its direct memory effects are reads. Its indirect effects are whatever the selected dependency performs.

No preservation guarantee is established for global state, node contents, ownership, or memory lifetimes across a comparison call. In particular, dependencies must not be assumed pure merely because their results are used as comparisons.

The function retains its selected target, captured counter, current node address, and query arguments across calls. That retained local state does not imply the referenced storage remains unchanged or valid.

There is no visible synchronization or concurrency protocol. Stable reads under concurrent modification are not guaranteed by this contract.

Errors are not reported through a separate status value. Null represents ordinary search failure, not a documented conversion of invalid memory, invalid targets, arithmetic failures, or dependency failures. No visible handler translates such failures into null.

## Dependencies

Two unresolved callable dependencies are involved. The following opaque IDs are document-local labels, not published identity claims:

| Opaque ID | Selection | Established call contract |
|---|---|---|
| F-000000000001 | Object selector byte is not 3 | Receives the two node words followed by the two query words; zero means match to this caller. |
| F-000000000002 | Object selector byte is 3 | Receives the same argument arrangement and uses the same zero/nonzero interpretation. |

Their addresses come from distinct target slots in the supplied representation. It is not established that those slots always contain different addresses.

For both dependencies, the following are **documented_only**:

- Semantic interpretation of arguments;
- Valid payload and query domains;
- Return values beyond zero versus nonzero;
- Global-state preservation;
- Memory ownership and release behavior;
- Mutation of nodes or table storage;
- Reentrancy and thread safety;
- Failure, exception, or nonlocal-exit behavior.

If a dependency does not return normally, the supplied function establishes no subsequent traversal or return behavior.

## Acceptance examples

These examples are conditional consequences of static evidence, not executed tests. Unless stated otherwise, all reached storage is valid, arithmetic remains within its supported range, and dependencies return normally.

### Absent table

**Given:** the object’s table field is zero.

**Required:** return null. No comparison occurs, and the selector byte need not be read on this path.

### Zero counter with a nonnull head

**Given:** a valid selected entry has counter zero and a nonnull head.

**Required:** perform entry and target-selection setup, but invoke neither comparison dependency. Return null without reading node payload or successor fields.

### Null head with a positive counter

**Given:** the selected entry has counter five and a null head.

**Required:** return null without invoking a comparison dependency or reading node fields.

### First-node match

**Given:** the counter is at least one, the head is nonnull, and the selected dependency returns zero for that node.

**Required:** call the dependency with the two node words followed by the query words. Return the head address. Do not read that node’s successor.

### Bound reached before a later match

**Given:** the counter is one; the head’s comparison returns nonzero; its successor points to another node that would match.

**Required:** read the head’s successor after the nonzero result, then return null without comparing the second node.

### Early end of sequence

**Given:** the counter is four; the first comparison returns nonzero; the first node’s successor is null.

**Required:** make one comparison, read that null successor, and return null.

### Selector distinction

**Given:** the selector byte is 3 and at least one comparison is reached.

**Required:** invoke F-000000000002.

**Given instead:** the selector byte is any other value.

**Required:** invoke F-000000000001.

### Successor mutation during comparison

**Given:** a comparison changes the current node’s successor to another valid node, returns nonzero, and leaves the current node readable; sufficient counter allowance remains.

**Required:** traversal follows the successor value read after the call, not its previous value.

### Cyclic sequence with a finite ordinary bound

**Given:** the counter is three, every comparison returns nonzero, and successors form a readable cycle.

**Required:** make three comparisons, including repeated visits where the cycle requires them, then return null. No cycle detection is performed.

## Uncertainties

- **Arithmetic model — documented_only.** The representation uses integer counter arithmetic and integer-based address calculation. Exact machine overflow behavior is not supplied. Negative counts, counter underflow, index multiplication overflow, and address overflow must not be assigned portable outcomes.
- **Negative-count intent — inferred only.** Nonnegative values behave as bounded counts. No evidence establishes that negative values are valid inputs or intentional special cases.
- **ABI and storage encoding — documented_only.** Four-byte field accesses are visible, but byte order, alignment tolerance, calling convention, and a complete pointer ABI are not independently established.
- **Concrete instruction-level reads — documented_only.** The source representation establishes logical access dependencies, not an exact machine load trace, volatile semantics, or atomicity.
- **Dependency identities and behavior — documented_only.** Only target selection, argument arrangement, and zero/nonzero use are established.
- **Lifetime and ownership — documented_only.** No local release operation is visible, but dependencies may affect storage. A returned address is not proof of continued validity.
- **Application meaning — inferred only.** Bucket lookup and equality search are plausible descriptions. No payload schema or application-specific equality rule is established.

## Evidence

This contract is based exclusively on the supplied static source representation of F-42dffdf1b496.

The evidence establishes the two object offsets, eight-byte entry stride, entry-field arrangement, three accessed node fields, selector value 3, four comparison argument positions, zero-as-match branch, pre-comparison decrement, post-miss successor read, immediate matching return, and null fallback.

No dependency implementations, runtime traces, binary instruction listing, allocator contract, synchronization contract, or surrounding type definitions were supplied. No runtime tests have been performed.
