<!-- Trusted provenance: F-1ecea458debe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1ecea458debe

## Purpose

F-1ecea458debe applies a caller-supplied callback to an initial object and conditionally processes objects reached through child lists. A zero callback result permits further processing. A nonzero result stops the active traversal and is returned unchanged.

The initial object is processed before its flags are examined. Its bit-0 flag, read only after its callback returns zero, controls whether its child list is traversed.

For each object reached through a child list, the parent examines that child’s bit-0 flag before processing it. A clear bit selects a direct callback. A set bit selects a recursive invocation, which calls the callback before deciding whether to descend further.

Hierarchical traversal is inferred intent. The established behavior does not require a tree, prevent repeated visits, detect cycles, or guarantee termination.

## Interface

### Arguments

| Argument | Established representation | Meaning |
|---|---|---|
| Initial object | 32-bit value used as an object address | Object processed first by this invocation. Zero is rejected. |
| Callback | Callable address | Operation invoked for each processed object. A null address is rejected. |
| Context | Uninterpreted 32-bit value | Forwarded unchanged to callbacks and recursive invocations. |

The context is not established to be a pointer. This function does not dereference it or assign it an ownership meaning.

The caller-supplied callback is designated **F-CB01**. It receives the current object address followed by the context. Its result is consumed as a 32-bit integer:

- Zero permits continuation.
- Any nonzero value stops processing along the active recursive chain.

The exact calling convention and callback-address width remain **documented_only**.

### Result

The result is a 32-bit integer.

| Condition | Result |
|---|---|
| Initial object is zero, callback is null, or both | −50 |
| Traversal completes normally and every callback it undertakes returns zero | 0 |
| A callback or recursive invocation returns nonzero | That nonzero value, unchanged |

All callbacks returning zero does not by itself guarantee a zero return. A cyclic list, unbounded recursion, or a callback that never returns can prevent normal completion.

Positive and negative nonzero results have the same stopping effect. The function does not classify, combine, or translate callback results. A callback-produced −50 is indistinguishable in the return value from the entry-validation error.

### Observable object layout

The function accesses the following fields:

| Byte offset from object address | Access width | Interpretation used here |
|---|---:|---|
| +8 | 32 bits | Next object in the current child list |
| +29 | 8 bits | Flags; only bit 0 affects processing |
| +32 | 32 bits | First child |

A zero first-child address represents an empty list. A zero next-object address terminates a list.

These accesses establish neither a complete object type nor a minimum allocation size for every invocation. Required readable storage depends on the path actually taken. For example, a nonzero initial callback result prevents all field reads by that invocation.

No object-field write is explicit in this function.

### Execution preconditions

For supported execution, the caller must provide:

- A callback compatible with the observed arguments and result use.
- Readable storage for each field when its access is reached.
- Object lifetimes sufficient for accesses pending after callbacks or recursive processing.
- Resources sufficient for the recursion and callback execution that occur.

Only the null initial-object and null-callback conditions receive explicit validation. The other requirements have no dedicated error result in the supplied body.

## Behavior

### Entry validation and initial callback

If the initial object is zero or the callback is null, the function returns −50. No callback or object-field read occurs on this path.

Otherwise, F-CB01 receives the initial object and context before this invocation reads any object field.

If that callback returns nonzero, the function immediately returns the same result. In particular, it does not then read the initial object’s flags, first-child field, or next-object field.

If the callback returns zero, the function reads the flags byte at offset +29:

- If bit 0 is clear, the invocation returns zero without reading the first-child field.
- If bit 0 is set, the invocation reads the first-child address at offset +32 and processes the resulting list.

Other flag bits do not affect this decision. A set bit with a zero first-child address produces successful completion after the initial callback.

The invocation does not follow its initial object’s next-object field. That field matters only when an enclosing invocation processes this object as a child.

### Child-list processing

For each nonzero child address selected from the list, the parent first reads that child’s flags byte.

If bit 0 is clear, the parent invokes F-CB01 directly on the child, passing the same context. This path does not inspect the child’s first-child field.

If bit 0 is set, the parent recursively invokes F-1ecea458debe with the child as its initial object and with the same callback and context.

Continuation depends on the selected operation’s result:

- A nonzero result is returned immediately by the parent.
- A zero result permits the parent to read the processed child’s next-object field at offset +8.
- A zero next-object address ends the list and returns zero.
- A nonzero next-object address selects the next child.

The parent does not prefetch the next-object link before processing the child. Consequently, a nonzero child-processing result suppresses that continuation read.

### Flag timing and callback mutation

A child selected for recursion has an unconditional parent-side flag observation: the parent reads bit 0 and finds it set.

A second flag observation occurs inside the recursive invocation **only if its initial callback returns zero**. If that callback returns nonzero, the recursive invocation returns immediately without reading the child’s flags again.

When both observations occur, they need not agree:

- If the parent sees bit 0 set, and the child’s callback clears it before returning zero, recursive processing does not read the child’s first-child field.
- If the parent sees bit 0 clear, it selects a direct callback. Setting the bit during that callback does not trigger descent during the same processing step. The parent does not repeat its processing-mode decision.

For an invocation’s initial object, there is no pre-callback flag read within that invocation. Its descent decision uses the flag value observed after its callback returns zero.

### Processing order

With stable relevant fields, normally returning callbacks, and finite reachable structure, processing is callback-first and depth-first within enabled child lists:

1. Process the invocation’s initial object.
2. If its post-callback bit 0 is set, select its first child.
3. Process that child directly or recursively according to the parent-side flag read.
4. Complete any selected recursive processing before following the child’s next-object link.

List order is determined by the link values actually read. The function does not sort, deduplicate, or snapshot the structure.

A nonzero first-child field alone does not enable descent. The initial object’s bit 0 must be set when read after its successful callback.

### Failure and termination

A nonzero callback result propagates unchanged through active enclosing invocations. No remaining sibling is processed after the failure reaches its parent. No cleanup, rollback, or compensating callback is explicit.

The function has no visible iteration limit, recursion-depth limit, visited-object tracking, or structural progress check.

Repeated references can produce repeated callbacks. A cyclic next-object chain can continue indefinitely while every callback returns zero. Enabled child cycles can cause unbounded recursive processing. A callback may stop either pattern by returning nonzero, but no such eventual result is guaranteed.

### Widths and address arithmetic

The object and link representations used here are 32 bits. Context and results are also 32 bits. The flag read is one byte, and the field displacements are byte offsets +8, +29, and +32.

The function does not perform checked address arithmetic or report displacement overflow. The evidence does not establish a portable wraparound rule or a recoverable error for an address whose required displacement is invalid.

Invalid addresses, insufficient readable storage, incompatible callbacks, and exhausted recursion resources have no specified outcome in this contract.

## Side effects

### Explicit effects

The function explicitly performs callback invocations, conditional object-field reads, and recursive invocations.

The supplied body contains no explicit:

- Object-field writes.
- Global-variable accesses.
- Allocation or deallocation.
- Retain or release operations.
- Logging or synchronization.

These observations apply to this function’s own operations. They do not constrain effects of F-CB01. The callback may modify memory or globals, invoke other operations, or affect resource ownership.

### Visibility of mutations

Relevant object state is read at the control-flow points described above, not captured at entry. Callback changes can therefore affect:

- The initial object’s post-callback flags.
- Its first-child address, if descent is enabled.
- The corresponding fields of recursively processed children.
- A child’s next-object address read after successful processing.
- Objects that have not yet been examined.

Changes made during descendant processing can affect the enclosing parent’s later read of the processed child’s next-object link.

A child address already selected for processing is not retroactively replaced by a later change to the link from which it was obtained. Subsequent link reads can nevertheless redirect later processing.

No atomicity, locking, or safe concurrent-mutation guarantee is established.

### Ownership and lifetime

No ownership transfer is explicit. The function does not establish ownership of objects or context, and it does not establish that the callback preserves either.

Returning zero often leaves accesses pending:

- After the initial callback, the invocation reads the initial object’s flags.
- After a direct child callback, the parent reads that child’s next-object field.
- After successful recursive processing, the parent reads the recursively processed child’s next-object field.

Releasing or invalidating such storage requires an independent guarantee that these pending reads remain valid.

A nonzero callback result suppresses subsequent reads along the failing path. That fact does not establish a release protocol or permission for the callback to invalidate arbitrary storage.

## Dependencies

| Opaque function ID | Role | Established behavior |
|---|---|---|
| F-CB01 | Caller-supplied callback | Receives the current object address and unchanged 32-bit context. A zero 32-bit result permits continuation; nonzero stops processing. Other effects are unknown. |
| F-1ecea458debe | Recursive processing | Invoked for a child whose parent-side bit-0 read is set, using the same callback and context. |

No other function dependency is visible in the supplied body.

F-CB01’s global-state preservation, memory ownership, lifetime effects, reentrancy, and termination behavior remain **documented_only**. Forwarding the same context value does not imply that memory it might designate remains unchanged.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Except where nontermination is the subject, assume all reached accesses and calls are valid and return as specified.

### Invalid entry

If either the initial object is zero or the callback is null, the result is −50. No callback occurs and no object field is read. This includes both arguments being null.

### Initial callback failure

If the initial callback returns 7, the function returns 7 without reading the initial object’s flags or either link field.

The same rule applies to a negative nonzero result such as −9.

### Disabled initial descent

If the initial callback returns zero and the subsequent flags read finds bit 0 clear, the function returns zero without reading the first-child field.

A stored nonzero first-child address and set flag bits other than bit 0 do not change this outcome.

### Enabled empty list

If the initial callback returns zero, the subsequent bit-0 read is set, and the first-child address is zero, only the initial callback occurs. The function returns zero.

### Stable nested processing

Assume:

- The initial callback returns zero, and the initial object’s bit 0 is set afterward.
- Its child list contains A followed by B.
- A’s bit 0 is set, and its child list contains only C.
- B’s and C’s bit 0 values are clear.
- All callbacks return zero and preserve relevant fields.
- The described lists terminate.

The callback order is initial object, A, C, B. Traversal completes and returns zero.

### Failure within a subtree

Use the preceding arrangement, including the enabled initial object, but let C’s callback return −9.

The callback order is initial object, A, C. The overall result is −9. B is not visited.

C’s next-object field is not read after its failing callback. A’s next-object field is not read by the enclosing invocation after A’s recursive processing returns −9.

### Recursive callback prevents the second flag read

Suppose an enabled parent reaches A and reads A’s bit 0 as set. The parent selects recursive processing.

If A’s callback returns 7, the recursive invocation returns 7 without performing its post-callback read of A’s flags or reading A’s first-child field. The parent propagates 7 without reading A’s next-object field.

Only the parent-side flag observation is required for this visit.

### Recursive callback disables descent

Suppose the parent reads A’s bit 0 as set, selecting recursion. If A’s callback clears that bit and returns zero, the recursive invocation reads the cleared flag and returns zero without reading A’s first-child field.

The parent then reads A’s next-object field.

### Direct callback enables a flag too late

Suppose the parent reads A’s bit 0 as clear, selecting a direct callback. If that callback sets the bit and returns zero, A’s descendants are not processed during this visit. The parent proceeds to A’s next-object field without reconsidering the flag.

### Mutation redirects continuation

Suppose A’s next-object link initially names B. During successful processing of A, that link changes to C and remains C until the parent reads it.

The parent continues at C, not B through the replaced link. B may still be reachable through another path.

### All-zero callbacks without normal completion

Suppose an enabled initial object has one child A, A’s bit 0 remains clear, and A’s next-object link points back to A. Every callback returns zero and the relevant fields remain unchanged.

The parent repeatedly invokes the callback on A. Under these conditions, traversal does not normally complete and no zero return is produced merely because every callback returns zero.

### Initial next-object link is not followed

If the initial callback returns zero and its post-callback bit 0 is clear, the invocation returns zero without following the initial object’s nonzero next-object link.

## Uncertainties

The following remain **documented_only**:

- Exact ABI, calling convention, callback-address width, byte order, and alignment requirements.
- Complete object layout and meanings of flag bits other than bit 0.
- Caller-provided structural guarantees, including acyclicity and link stability.
- Callback side effects, global-state behavior, ownership conventions, reentrancy, and termination.
- Fault behavior for invalid storage, incompatible callbacks, address arithmetic limits, or recursion exhaustion.
- Concurrency and synchronization requirements.
- Any broader semantic meaning of −50.

Hierarchical traversal is inferred intent, not proof of a tree invariant. These unknowns do not justify adding validation, cycle detection, cleanup, ownership transfer, or fault recovery to the established behavior.

## Evidence

The supplied static body establishes the entry checks, callback argument order, 32-bit context forwarding and result use, field offsets, access widths, bit-0 decisions, recursive selection, and unchanged propagation of nonzero results.

Its control flow establishes that a recursive visit’s second flag read occurs only after that invocation’s initial callback returns zero. A nonzero result bypasses that read.

Normal zero returns occur when descent is disabled or a selected child list reaches its zero terminator. There is no unconditional termination guarantee merely from all callback results being zero.

The body contains no explicit object writes, global accesses, allocation, release, synchronization, or cycle detection. Those observations do not constrain the unknown callback.

No runtime tests, callback executions, or memory traces have been performed. The acceptance examples specify conditional, source-supported behavior rather than measured outcomes.
