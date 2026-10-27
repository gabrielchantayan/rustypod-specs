<!-- Trusted provenance: F-2b3097eaee8e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2b3097eaee8e

## Purpose

F-2b3097eaee8e updates a caller-owned cursor using links stored in a node structure. It returns no value.

**Established behavior:** If the current node has a nonzero link at byte offset 12, the function follows that link and then follows links at byte offset 8 until one is zero. Otherwise, it follows links at byte offset 4 while a relationship involving offset 12 holds, then conditionally updates the cursor.

**Inferred intent:** The offsets are consistent with a parent-linked binary tree: offset 4 is a parent link, offset 8 is a left-child link, and offset 12 is a right-child link. Under an appropriate tree and sentinel convention, the operation resembles advancing an in-order iterator. Those roles are explanatory names, not independently established structure definitions. In particular, the function does not guarantee a conventional successor result for every arbitrary arrangement of these links.

## Interface

### Input and output

The sole argument identifies a writable cursor cell. The cell contains an integer-encoded node address. The function reads that address, examines linked nodes, and may overwrite the cursor cell several times before returning.

The function has no return value, status code, or separate output object. Its explicit result is the final value of the cursor cell.

### Representation and layout

The supplied representation uses integer-valued addresses and integer-sized link fields. It is consistent with a 32-bit address model and 32-bit integer fields:

| Location | Meaning established by use |
|---|---|
| Cursor cell | Address of the currently selected node |
| Node address + 4 bytes | Link followed during upward traversal |
| Node address + 8 bytes | Link followed during downward traversal |
| Node address + 12 bytes | Initial branch link and upward-relationship link |

The offsets are byte offsets, not indices multiplied by a node size. Nothing establishes the meaning of bytes at offset 0, payload layout, total allocation size, or fields beyond those listed.

The supplied text does not independently establish the target ABI. Consequently, the 32-bit interpretation is a representation assumption supported by the source form, not a verified runtime property. An implementation on another data model must preserve the intended address representation and byte layout rather than silently substitute native pointer-width fields.

Zero has two established uses: a zero offset-12 link selects upward traversal, and a zero offset-8 link terminates downward traversal. Zero is **not** handled as a safe terminator for upward traversal.

### Preconditions

For defined operation:

- The argument must identify a live, suitably aligned, readable and writable cursor cell.
- Every cursor value used as a node address must permit the field accesses actually performed.
- Every upward link reached must permit a read at byte offset 12, even before the function decides whether to adopt that link as the cursor.
- Address arithmetic and integer-to-address conversion must be valid under the target execution model.
- Required nodes and the cursor cell must remain alive throughout the operation.
- External mutation must not race with the accesses.

For the usual tree interpretation, the cursor cell should be separate from the traversed link fields. The supplied operation does not check that separation. If the cursor aliases a link field, its writes can change subsequent traversal reads; ordinary tree-successor expectations then do not apply.

Termination additionally requires a finite terminating traversal. Neither cycle detection nor a traversal limit is present.

## Behavior

### Initial branch selection

The function obtains the current node address from the cursor and reads the link at byte offset 12 of that node.

A nonzero link selects downward traversal. A zero link selects upward traversal. There is no initial check for a zero cursor value.

### Downward traversal

When the initial offset-12 link is nonzero, that linked address becomes the first candidate.

For each candidate:

1. The function stores the candidate address into the caller’s cursor cell.
2. It reads the candidate’s link at byte offset 8.
3. If that link is nonzero, it becomes the next candidate. If it is zero, the function returns.

The final cursor therefore identifies the last node reached in this sequence: the initial offset-12 target followed by zero or more offset-8 links.

The cursor write precedes the read of that candidate’s offset-8 field. This matters for partial progress and aliasing. If the following field read cannot complete, the cursor has already been changed to that candidate.

This branch does not inspect parent links. It does not read the initial node’s offset-8 field unless the link structure later leads back to that node.

### Upward traversal

When the initial offset-12 link is zero, the function reads the current node’s offset-4 link as an upward candidate.

It then compares the candidate node’s offset-12 link with the current cursor value.

- If they are equal, it stores the candidate into the cursor, obtains that candidate’s offset-4 link as the next upward candidate, and repeats the comparison.
- If they differ, it stops climbing and performs a final conditional update.

Thus, each successful upward iteration first changes the cursor and then reads the next offset-4 link. The candidate’s offset-12 field must be readable even when that candidate is never stored into the cursor.

Under the inferred tree interpretation, this climbs past ancestors for which the node currently selected by the cursor is the offset-12 child.

### Final conditional update

After upward traversal stops, the function reads the offset-12 link of the node currently selected by the cursor.

- If that link differs from the upward candidate, the function stores the candidate into the cursor.
- If that link equals the candidate, the function returns without that final store.

The comparison that stops upward traversal and this final comparison are different:

- The stopping comparison examines the **candidate’s** offset-12 link and compares it with the cursor.
- The final comparison examines the **cursor node’s** offset-12 link and compares it with the candidate.

The final condition cannot be replaced with an unconditional cursor assignment. Structures with sentinel or reciprocal links can make the no-store outcome significant.

### Read and write ordering

The sequencing above is established at the level of the supplied operation: candidate selection precedes its cursor write, and that write precedes the next downward or upward link read.

An expression comparing a field read with a cursor read does not establish which operand is evaluated first. This contract therefore does not promise a finer order between those reads, physical bus accesses, atomicity, or concurrent observability.

## Side effects

The only explicit writes are assignments to the caller-provided cursor cell. There are no explicit writes through node addresses.

The cursor is not updated transactionally:

- Downward traversal writes once for every candidate visited.
- Upward traversal writes once for every successful climb.
- The final upward update is conditional.

A completed call can perform several writes even if only the final cursor value is relevant to an ordinary caller. A call that encounters invalid memory after an earlier write has no rollback mechanism.

No allocation, release, ownership transfer, payload modification, callback invocation, logging, or error reporting is present in the supplied body. The function uses existing storage; callers retain responsibility for its lifetime.

There is no explicit access to a named global. This does not establish preservation of all globally reachable memory: a caller can supply a cursor located in global storage or aliased with other reachable state.

Invalid addresses, unsuitable alignment, invalid address arithmetic, and data races are outside the defined operating preconditions. The supplied body provides no recoverable error behavior for them.

## Dependencies

No function calls are present. There are therefore no callee dependencies to identify by opaque function ID, and no unknown callee whose ownership or global-state behavior must be assumed.

The operation depends on the execution environment’s address model and on the caller-provided node-link convention. In particular, a tree or sentinel representation must ensure that every upward candidate inspected by the function is readable. A null parent convention alone does not satisfy that requirement.

## Acceptance examples

These examples are conditional on valid storage, the stated representation, stable links, and a cursor cell separate from node fields. Unmentioned fields are irrelevant unless traversal reaches them.

### Direct downward selection

Initially, the cursor identifies node A. A’s offset-12 link identifies B, and B’s offset-8 link is zero.

**Required result:** The cursor becomes B and the function returns. It writes B before reading B’s offset-8 link. No parent field is needed.

### Multiple downward steps

A’s offset-12 link identifies B. B’s offset-8 link identifies C. C’s offset-8 link identifies D, and D’s offset-8 link is zero.

**Required result:** Cursor stores occur in the order B, C, D. The final cursor is D.

### Upward candidate accepted immediately

Initially, the cursor identifies A. A’s offset-12 link is zero, and its offset-4 link identifies B. B’s offset-12 link does not identify A.

**Required result:** No climbing iteration occurs. The final comparison examines A’s offset-12 link, which is zero. If B is a nonzero valid address, that comparison requires the cursor to become B.

### Upward climbing followed by selection

A’s offset-12 link is zero, and A’s offset-4 link identifies B. B’s offset-12 link identifies A, so the first climb stores B. B’s offset-4 link identifies C, but C’s offset-12 link does not identify B.

If B’s offset-12 link also differs from C, the final update stores C.

**Required result:** The cursor is first B and then C. This outcome follows from both upward comparisons, not merely from following parent links.

### Final update deliberately omitted

A’s offset-12 link is zero and its offset-4 link identifies B. B’s offset-12 link identifies A, causing a climb to B. B’s offset-4 link identifies C. C’s offset-12 link differs from B, ending the climb. B’s offset-12 link equals C.

**Required result:** The cursor remains B. The final assignment is omitted. These relationships imply C and A identify the same node in this example; they are not independent distinct-node assumptions.

### Zero upward link

A’s offset-12 link is zero, and its offset-4 link is zero.

**Required interpretation:** The function attempts a field read using the zero upward candidate plus byte offset 12. It does not return a null cursor as an explicitly supported “no successor” result. Without an execution environment making that access valid, the preconditions fail.

### Cyclic downward links

The downward branch reaches nodes whose nonzero offset-8 links cycle indefinitely.

**Required interpretation:** With stable accessible storage, the function does not return and continues updating the cursor around that cycle. No cycle error or iteration limit is specified.

## Uncertainties

**Inferred intent:** The traversal resembles in-order successor selection for a parent-linked binary tree. The final conditional assignment suggests compatibility with a special boundary or sentinel arrangement.

**documented_only — structure convention:** No structure definition establishes the semantic names of the links, the root representation, sentinel layout, or the meaning of an end cursor.

**documented_only — complete successor semantics:** The source alone does not establish that all valid application states produce a mathematical successor, particularly at the maximum element or when the input cursor is already a sentinel.

**documented_only — ABI and integer arithmetic:** The apparent 32-bit model is not independently verified. Overflow, address wrapping, alignment, and integer-to-pointer behavior depend on the target model. Wrapping is not a promised traversal feature.

**documented_only — exceptional execution:** No runtime evidence establishes fault delivery, instrumentation behavior, or externally visible intermediate stores under a particular compiler.

## Evidence

The established contract comes solely from static inspection of the supplied body for F-2b3097eaee8e: one mutable cursor argument, field accesses at byte offsets 4, 8, and 12, a downward branch, an upward loop, and a conditional final cursor assignment.

No external definitions, caller behavior, ABI documentation, or sentinel specification were supplied. No runtime tests have been performed. The acceptance examples are behavioral consequences under their stated conditions, not observed executions.
