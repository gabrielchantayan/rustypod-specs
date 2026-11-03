<!-- Trusted provenance: F-0ec4c847eecb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0ec4c847eecb

## Purpose

F-0ec4c847eecb selects a 32-bit result using one of two paths determined by the first byte of an input record:

- A first byte equal to `0x01` selects a direct-value path. A value in the record supplies the result unless it is zero, in which case a shared fallback supplies the result.
- Every other first-byte value selects a callback-search path. The function obtains an initial object from F-9d0ecc5f6adf, visits linked objects, and returns a separate result value that callbacks may update.

The callback’s return value controls whether traversal continues; it is not itself the result returned by F-0ec4c847eecb.

**Inferred intent:** the callback-search path resembles a chain of handlers or providers queried with a shared context and a record-specific value. Neither those semantic roles nor the meaning of the result is established by the supplied evidence.

## Interface

### Input

The function takes one pointer to a record. The visible behavior uses these locations:

| Byte offset | Width | Role |
|---|---:|---|
| `0x00` | 1 byte | Path selector; only `0x01` selects the direct-value path |
| `0x08` | 4 bytes | Direct result candidate |
| `0x48` | 4 bytes | Opaque value supplied to callbacks |

The evidence represents the direct candidate and function result as 32-bit integers. The callback argument from offset `0x48` is a four-byte quantity with no established signedness or semantic type.

Only fields needed by the selected path are visibly accessed. In particular:

- The direct-value path does not visibly access offset `0x48`.
- The callback-search path does not visibly access offset `0x08`.

The record pointer must be valid for the selected accesses. There is no visible null check, record-size check, or explicit alignment handling. Null, insufficiently accessible, or incorrectly aligned storage has no defined recovery behavior in this function.

### Result

The result is a 32-bit integer:

- On the direct-value path, it is the record’s direct candidate or the shared fallback.
- On the callback-search path, it is the final contents of a local four-byte result slot, initially zero.

No result value is reserved here as a separately reported error. Zero and negative values can be ordinary returned results.

### Representation assumptions

The supplied representation supports a 32-bit object layout: four-byte object fields, a four-byte dispatch-table pointer, and a link at byte offset `0x14`.

Endianness, ABI register assignments, and machine-level access atomicity are not established. Consumers interpreting raw record bytes must obtain those requirements from the surrounding binary or platform contract.

## Behavior

### Path selection

The function first examines the byte at record offset zero.

The comparison is exact. A value of `0x01` selects the direct-value path. Values such as `0x00`, `0x02`, and `0xFF` select the callback-search path. This is not a general zero-versus-nonzero flag test.

There is no visible subsequent recheck of the selector. Once the path is selected, later modifications of that byte do not visibly cause path switching.

### Direct-value path

With stable input storage:

- A nonzero 32-bit candidate at offset `0x08` is returned unchanged.
- A zero candidate causes a shared 32-bit fallback to be read and returned.

The shared fallback is not validated. If it is zero, the function returns zero. If it has a negative signed interpretation, that negative value is returned.

This path does not visibly call F-9d0ecc5f6adf or any object callback, initialize the callback result slot, or read the shared callback argument.

**Access-order qualification:** the supplied representation expresses a candidate read used to retain the prospective return value and another candidate read used for the zero decision. It does not establish whether the underlying machine performs two distinct loads. The storage is not identified as volatile or atomic.

Accordingly, the established ordinary-use contract assumes that the candidate remains stable during this operation. Behavior under concurrent changes, externally changing storage, or memory-mapped side effects is **documented_only**; neither a single-snapshot guarantee nor an exact hardware read count is established.

### Callback-search preparation

For any selector other than `0x01`, the visible ordering is:

1. Call F-9d0ecc5f6adf to obtain the initial object pointer.
2. Read a shared four-byte callback argument.
3. Read the record’s four-byte value at offset `0x48`.
4. Initialize a local four-byte result slot to zero.
5. Begin traversal if the initial object pointer is non-null.

This ordering matters because F-9d0ecc5f6adf runs before the shared argument and record value are captured. Its side effects may therefore influence those later reads.

The shared callback argument and record value are captured once in the visible preparation. They are not visibly reloaded for each object. Later callbacks changing their original storage do not, by this function’s own actions, replace the captured argument values for later calls.

An empty chain does not bypass preparation: even when F-9d0ecc5f6adf returns null, the shared argument and record value are visibly read before the zero result is returned.

### Object and callback layout

For every non-null current object, traversal uses this layout:

| Location | Role |
|---|---|
| Object offset `0x00` | Dispatch-table pointer |
| Dispatch-table offset `0x64` | Callable entry |
| Object offset `0x14` | Next-object pointer, read only after a zero callback status |

The entry at dispatch-table byte offset `0x64` is invoked with four arguments, in this order:

1. The current object pointer.
2. The captured shared four-byte argument.
3. The captured four-byte value from record offset `0x48`.
4. A pointer to the local result slot.

The callback’s concrete identity is not supplied. Its opaque dependency designation here is **F-unknown-dispatch-0064**; the actual target can differ between objects.

The function does not visibly validate the dispatch-table pointer or callback entry before calling it.

### Traversal and stopping

Each callback returns a status interpreted through comparison with zero:

- A zero status means continue traversal.
- Any nonzero status means stop traversal immediately.

Positive and negative nonzero statuses are equivalent for this decision. The status is not copied into the returned result.

After a zero status, the function reads the current object’s next pointer at offset `0x14`. Because this read follows the callback, callback changes to that link can determine which object is visited next.

After a nonzero status, that next pointer is not visibly read.

Traversal also stops when the current object pointer becomes null. There is no visible visit limit, cycle detection, or repeated-object suppression. A cycle whose callbacks always return zero can prevent the function from returning.

### Result-slot persistence

The result slot is initialized to zero once, before traversal. It is not reset between callbacks.

Consequently:

- An empty chain returns zero.
- Callbacks that leave the slot untouched preserve its previous contents.
- A callback can update the result while returning zero, allowing later traversal.
- A stopping callback can return nonzero without updating the slot; the existing slot value is then returned.
- Exhausting the chain returns the slot’s current value, even if no callback returned nonzero.

A callback may write zero into the result slot. There is no distinction between zero written by a callback and the initial zero.

The supplied representation reserves local storage larger than the single initialized four-byte result slot. It does not establish a supported output structure beyond that slot. Adjacent storage is not initialized by the visible preparation and must not be treated as an additional defined output.

### Arithmetic and value handling

No arithmetic is visibly performed on the candidate, fallback, captured arguments, or result slot. In particular, there is no increment, scaling, saturation, or range normalization.

Offsets in this contract are byte offsets. The next-object field is at `0x14`, not at byte offset five. The callable entry is at dispatch-table byte offset `0x64`.

## Side effects

### Directly established effects

F-0ec4c847eecb visibly:

- Reads the selector and path-specific fields.
- Reads shared state as required by the selected path.
- Calls F-9d0ecc5f6adf on the callback-search path.
- Initializes a local result slot and exposes its address to callbacks.
- Reads object dispatch and link fields during traversal.

It does not visibly write to the input record, shared state, or object fields itself. It contains no explicit allocation, release, reference-count adjustment, or ownership transfer.

These observations do not constrain what dependencies do.

### Dependency-mediated effects and lifetime

F-9d0ecc5f6adf and F-unknown-dispatch-0064 may have side effects not shown in the supplied evidence. Their preservation of shared state, input storage, object contents, and memory ownership is unknown.

Safe traversal requires each current object and its dispatch information to remain usable for the accesses performed. In particular, a zero-returning callback must leave the current object’s next-link field readable when traversal continues, or otherwise ensure that the subsequent access is valid.

The result-slot pointer refers to local storage whose lifetime does not extend beyond the invocation. No durable ownership of that storage is established. Retaining and later using that pointer is unsupported by the visible lifetime.

No synchronization, lock acquisition, fault recovery, or exception handling is visible. Dependency failures and invalid memory accesses have no specified conversion into a normal error result.

## Dependencies

| Opaque ID | Established interaction | Unknowns |
|---|---|---|
| F-9d0ecc5f6adf | Called once on the callback-search path, with no explicit arguments in the supplied representation; its result becomes the initial object pointer | Object provenance, allocation, ownership, side effects, failure modes, and stability |
| F-unknown-dispatch-0064 | Indirect callback selected from the current object’s dispatch table; receives the object, two captured four-byte arguments, and the result-slot pointer | Concrete identity, argument meanings, write extent, ownership behavior, termination, and other side effects |

The shared callback argument and direct-path fallback are distinct shared storage locations. Their initial values, writers, and relationship—if any—are not established.

## Acceptance examples

These are conditional static expectations, not executed tests. They assume valid storage, compatible callbacks, and stable values except where a stated callback changes them.

| Scenario | Expected observable behavior |
|---|---|
| Selector is `0x01`; candidate is `27` | Return `27`; no dependency call is required by this path |
| Selector is `0x01`; candidate is zero; fallback is `-8` | Return `-8` |
| Selector is `0x01`; candidate and fallback are zero | Return zero |
| Selector is `0xFF`; initial object is null | Capture the shared argument and record value, then return zero without an object callback |
| One object writes `42` and returns nonzero | Return `42`; do not read that object’s next link |
| One object writes nothing and returns `-1` | Stop and return the initialized zero; `-1` is only the stop status |
| First object writes `12` and returns zero; second writes nothing and returns nonzero | Return `12`, demonstrating persistence across callbacks |
| An object writes `31`, returns zero, and has a null next link | Return `31` after chain exhaustion |
| First object writes `12`; a later stopping object writes zero | Return zero |
| A zero-returning callback replaces its own next link with another valid object | Follow the replacement link read after the callback |
| A nonzero-returning callback leaves its next link inaccessible | This function does not subsequently read that link |
| A callback changes the original shared argument storage | Later callbacks still receive the previously captured value, absent other interference |
| F-9d0ecc5f6adf changes the record value at offset `0x48` before returning | The subsequent capture observes that changed value, assuming ordinary stable storage afterward |
| All callbacks return zero while the links form a cycle | No normal return is guaranteed |

A non-`0x01` selector with a null initial object still requires the record’s offset `0x48` to be readable. A null chain is not a supported way to avoid that input requirement.

## Uncertainties

The following remain **documented_only**:

- Semantic names for the record, selector, shared values, objects, and returned integer.
- Concrete callback identities and their full contracts.
- Whether any dependency allocates, retains, releases, or invalidates objects.
- The required byte order, calling convention, and machine-level alignment rules.
- Exact load counts and behavior under concurrent mutation.
- Whether callbacks may access more than the initialized result slot, and whether any such access is valid.
- Whether callbacks throw, terminate execution, re-enter this function, or otherwise fail to return normally.
- Any higher-level error convention encoded in returned integers.

The apparent handler-chain design is inferred intent, not an additional guarantee. In particular, a nonzero callback status does not establish that the result slot was written or that a meaningful result was produced.

## Evidence

This contract is based solely on the supplied static representation of F-0ec4c847eecb and the supplied dependency mapping.

Established evidence covers the exact selector comparison, path-specific field offsets, shared-value reads, initial dependency call, indirect callback layout, persistent zero-initialized result slot, zero-versus-nonzero traversal decision, post-callback next-link read, and final return source.

No implementation access, runtime execution, or tests were performed. The acceptance examples are deductions under their stated preconditions, not observed execution results.
