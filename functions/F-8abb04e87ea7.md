<!-- Trusted provenance: F-8abb04e87ea7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8abb04e87ea7

## Purpose

F-8abb04e87ea7 updates a caller-supplied position cell by following links in an externally owned, word-addressed structure. It returns no explicit result; the observable result is the value left in that cell.

Two traversal modes are established by the supplied static evidence:

- If the starting record’s link at byte offset 12 is nonzero, follow that link and then repeatedly follow links at byte offset 8, publishing each visited record into the position cell.
- Otherwise, ascend through links at byte offset 4 while a relationship involving the candidate ancestor’s link at byte offset 12 holds. A final comparison determines whether to publish the candidate ancestor.

**Inferred intent:** the link pattern resembles in-order successor traversal in a binary tree, potentially with a sentinel or header record. That interpretation is not necessary to specify the observed behavior. In particular, the evidence does not establish a tree invariant, identify a sentinel value, or define an abstract “end” position.

This contract describes the function’s own accesses and updates. It does not assume that a surrounding container validates links, prevents concurrent mutation, or repairs invalid structure.

## Interface

### Input and output

The function accepts one pointer to a mutable position cell.

The cell initially contains an integer-valued record address. During execution, the function may replace that address with addresses obtained from record links.

There is no explicit return value, status result, or error indicator.

### Representation and layout

The supplied listing represents both record addresses and record links as integer values. The apparent target representation is a 32-bit address stored in a 32-bit integer word. This is a target interpretation, not a portable guarantee derived from an independently supplied ABI.

Under that interpretation, the relevant layout is:

| Location | Width | Observed use |
|---|---:|---|
| Position cell | 32 bits | Current record address; repeatedly readable and writable |
| Record + 4 bytes | 32 bits | Ascending link |
| Record + 8 bytes | 32 bits | Descending link used after taking a nonzero offset-12 link |
| Record + 12 bytes | 32 bits | Initial branch-selection link, ascending relationship link, and final comparison link |

Offsets are byte offsets. The record’s contents at byte offset 0 are not inspected by this function.

The function does not perform ordering comparisons on addresses. It tests links for zero and compares address values for equality or inequality.

### Preconditions for a defined, terminating invocation

The caller must provide:

- A valid, suitably aligned position cell that remains readable throughout the invocation.
- A writable position cell whenever the selected path performs a store.
- Valid, suitably aligned storage for every link field actually read.
- Address values for which the required field-address calculations are valid in the target execution model.
- A structure whose selected traversal terminates, if completion is required.

The position cell is dereferenced immediately. Its initial value is not checked for zero before a field at offset 12 is read.

No disjointness check exists between the position cell and record storage. The ordinary traversal interpretation assumes the position cell does not overlap fields whose contents govern traversal. If overlap is permitted by the execution model, position-cell stores can alter subsequent link reads; the ordered accesses below, rather than an idealized tree traversal, determine the result.

## Behavior

### Initial branch selection

The function obtains the starting record address from the position cell and reads that record’s offset-12 link.

This initial read occurs before any position-cell store.

A nonzero value selects descending traversal. A zero value selects ascending traversal. The function does not subsequently reconsider this initial branch decision.

### Descending traversal

The first destination is the nonzero offset-12 link obtained during branch selection.

For each destination:

1. The destination address is stored into the position cell.
2. The destination record’s offset-8 link is read.
3. A zero link ends the function. A nonzero link becomes the next destination.

Consequently:

- Every visited destination is published to the position cell before its offset-8 link is inspected.
- A one-record descent still performs a position-cell store.
- The terminating zero link is never stored into the position cell.
- The final position is the last visited record whose offset-8 link is zero.
- Offset-4 links are not read on this path.
- After the initial branch-selection read, offset-12 links are not read on this path.

The function does not merely compute a final destination and store it once. Intermediate stores are part of its behavior.

If a destination address is invalid, the function can already have published that address before attempting the invalid offset-8 read. There is no rollback.

### Ascending traversal

This path is selected only when the initial offset-12 link is zero.

The function first reads the starting record’s offset-4 link to obtain a candidate ancestor. It then reads that candidate’s offset-12 link and compares it with the address currently held in the position cell.

When those values are equal:

- The candidate address is stored into the position cell.
- The candidate’s offset-4 link is read to obtain the next candidate.
- The relationship comparison is repeated for the new candidate and the updated position cell.

This continues while each candidate’s offset-12 link equals the current position.

When that equality first fails, the candidate remains available for a separate final decision. The function reads the current position record’s offset-12 link:

- If that link differs from the candidate address, the candidate is stored into the position cell.
- If that link equals the candidate address, the position cell is left unchanged.

The final comparison is distinct from the comparison that ended ascent:

- The ascent comparison examines the **candidate’s** offset-12 link.
- The final comparison examines the **current position’s** offset-12 link.

Substituting one comparison for the other would change behavior.

### Important ascending-path consequences

A candidate is dereferenced before it can be selected by the final store. In particular, a zero offset-4 link does not cause a clean return or an ordinary zero-position result: the next relationship check attempts to read at offset 12 relative to that zero value.

The function does not write the candidate into the position cell merely because it was obtained from an offset-4 link. Publication occurs either when the relationship comparison succeeds or when the final inequality succeeds.

If no ascent iteration succeeds, the final comparison rereads the starting record’s offset-12 link. For stable, non-overlapping storage, that link is still zero. In a valid execution with a nonzero candidate, the final inequality therefore selects that candidate.

After one or more successful ascent iterations, the final comparison can preserve the last published position instead of selecting the candidate that ended ascent.

### Read and write ordering

The established ordering is:

- Initial offset-12 read before any store.
- On descent, each position-cell store before the corresponding offset-8 read.
- On ascent, each successful relationship comparison before its position-cell store.
- Each successful ascent store before reading that ancestor’s offset-4 link.
- The failed ascent comparison before the final offset-12 read.
- The final offset-12 read before any final candidate store.

The source representation does not establish a machine-level total order between independent operand reads within an equality comparison. It also does not establish that all logical reads correspond to separate physical memory transactions in an optimized binary.

No atomic-access, synchronization, or volatile-access guarantee is present.

### Arithmetic and bounds

The only address arithmetic shown is addition of the fixed byte offsets 4, 8, and 12. There is no traversal counter, length limit, allocation-size check, or address-range check.

For a 32-bit target interpretation, each field address must remain valid under that target’s address arithmetic and memory model. The supplied evidence does not establish a portable wraparound rule for invalid addresses or overflowing signed address calculations.

Traversal is not bounded by the function. Cyclic links can prevent termination when the selected conditions continue to hold.

## Side effects

### Writes

Every explicit write targets the caller-supplied position cell.

There are no explicit stores to record fields. If the cell overlaps record storage, however, a cell store can also change a field used by a later read.

Descending traversal performs one store per visited destination.

Ascending traversal performs:

- One store per successful relationship comparison.
- At most one additional store after the relationship comparison fails.

An ascending invocation may therefore perform no stores, intermediate stores only, or intermediate stores followed by a final candidate store, depending on the observed links and storage relationships.

### Ownership and lifetime

The function performs no visible allocation, deallocation, ownership transfer, reference-count update, or destruction.

It retains no explicit persistent reference outside the position cell. Record storage and the position cell must remain valid for their respective accesses.

The evidence establishes no ownership convention for the enclosing structure.

### Errors and partial progress

There is no explicit validation or error-reporting path.

Invalid, unmapped, misaligned, or otherwise impermissible accesses have consequences determined by the target execution environment. The function does not translate them into a documented status or sentinel value.

Stores completed before a later invalid access are not undone. A failure during traversal can therefore leave the position cell changed.

Concurrent mutation is not coordinated by this function. The evidence does not establish a supported concurrent-use contract or a coherent snapshot of the structure.

## Dependencies

No calls to other functions are present in the supplied body. There are therefore no callee dependencies to identify by opaque function ID.

The behavioral dependencies are external memory conditions:

- The position cell contains an address in the expected representation.
- Each selected link field is readable.
- Each required position-cell store is permitted.
- The selected path eventually reaches its termination condition when completion is expected.

No allocator, callback, comparison routine, or cleanup routine is invoked.

## Acceptance examples

These examples are conditional static acceptance cases, not reports of execution.

Unless otherwise stated, all listed addresses identify valid, aligned, non-overlapping records; the position cell is separate, writable storage; field contents remain stable; and address arithmetic is valid. Field values not needed by the selected path are intentionally unspecified.

### 1. Single descending destination

Initial position: 0x1000.

Relevant fields:

| Record | Offset | Value |
|---|---:|---:|
| 0x1000 | 12 | 0x2000 |
| 0x2000 | 8 | 0 |

Expected behavior:

- Publish 0x2000.
- Read its zero offset-8 link.
- Return with position 0x2000.

No offset-4 read is required.

### 2. Multiple descending destinations

Initial position: 0x1000.

Relevant links:

- 0x1000 at offset 12 contains 0x4000.
- 0x4000 at offset 8 contains 0x3000.
- 0x3000 at offset 8 contains 0x2000.
- 0x2000 at offset 8 contains zero.

Expected position-cell store sequence: 0x4000, then 0x3000, then 0x2000.

The final position is 0x2000. An implementation that stores only 0x2000 does not preserve the evidenced intermediate-write behavior.

### 3. Select the immediate ancestor

Initial position: 0x2000.

Relevant links:

- 0x2000 at offset 12 contains zero.
- 0x2000 at offset 4 contains 0x3000.
- 0x3000 at offset 12 contains 0x4000.

The candidate’s offset-12 link differs from the current position, so no ascent iteration publishes 0x3000. The final comparison reads the current record’s zero offset-12 link, which differs from 0x3000.

Expected result: one store of 0x3000, followed by return.

### 4. Ascend before selecting a later ancestor

Initial position: 0x1000.

Relevant links:

- 0x1000 at offset 12 contains zero.
- 0x1000 at offset 4 contains 0x2000.
- 0x2000 at offset 12 contains 0x1000.
- 0x2000 at offset 4 contains 0x3000.
- 0x3000 at offset 12 contains 0x4000.

The first relationship comparison succeeds and publishes 0x2000. The next comparison fails because 0x4000 differs from 0x2000. The final comparison examines 0x2000’s offset-12 link, containing 0x1000, and finds it different from candidate 0x3000.

Expected store sequence: 0x2000, then 0x3000. Final position: 0x3000.

### 5. Preserve the last ascended position

Initial position: 0x1000.

Relevant links:

- 0x1000 at offset 12 contains zero.
- 0x1000 at offset 4 contains 0x2000.
- 0x2000 at offset 12 contains 0x1000.
- 0x2000 at offset 4 contains 0x1000.

The first comparison succeeds and publishes 0x2000. The next candidate is 0x1000. Its offset-12 link is zero, so ascent stops. The final comparison finds that the current record’s offset-12 link equals candidate 0x1000.

Expected result: retain 0x2000; do not publish 0x1000.

This establishes the final equality exception without assuming that either record is a sentinel.

### 6. Failure after a descending store

Suppose the initial offset-12 link is nonzero but identifies storage whose offset-8 field cannot legally be read.

Expected access ordering: the nonzero address is stored into the position cell before the attempted offset-8 read.

No successful return is required for this invalid input. No rollback behavior is established.

### 7. Zero ascending candidate

Suppose the initial offset-12 link is zero and the initial offset-4 link is also zero.

Expected behavior: attempt the candidate’s offset-12 read. A clean zero-position return is not established.

On an environment where that address is inaccessible, the invocation encounters an invalid access before any position-cell store.

## Uncertainties

### Documented only: target representation

The listing exposes integer-valued addresses and four-byte-spaced fields, consistent with a 32-bit target. An independent ABI, instruction listing, and memory-layout declaration were not supplied. Exact target integer semantics, alignment requirements, and exceptional-access behavior remain unverified.

### Documented only: abstract data structure

Binary-tree successor traversal is a strong structural interpretation, not a separately established API guarantee. Record roles, ordering invariants, sentinel construction, root handling, and abstract end-state meaning are unknown.

### Documented only: aliasing and concurrency

No explicit non-aliasing or synchronization contract accompanies the body. The simple graph examples assume stable, separate storage. Other legal storage relationships must be evaluated using the actual read/write ordering.

### Documented only: external observability

The supplied source representation establishes logical accesses and updates. Exact instruction order, optimization effects, memory-bus transactions, and observation by concurrent actors are not established.

## Evidence

The supplied body for F-8abb04e87ea7 is the sole evidence.

Established static observations include:

- One pointer argument and no explicit return value.
- An initial offset-12 link test.
- A nonzero branch that publishes destinations before following offset-8 links.
- A zero branch that follows offset-4 links while comparing candidate offset-12 links with the current position.
- A separate final comparison that can suppress the final candidate store.
- Writes directed only to the supplied position cell.
- No visible function calls, allocation, deallocation, validation, or error reporting.

The successor-traversal interpretation and apparent 32-bit target model are identified above as interpretations rather than independent runtime facts.

No runtime tests have been performed.
