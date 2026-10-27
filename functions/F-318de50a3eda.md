<!-- Trusted provenance: F-318de50a3eda; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-318de50a3eda

## Purpose

F-318de50a3eda invokes an operation through an object’s dispatch table, stores that operation’s reported result in caller-provided storage, and returns a separate status indicating whether the reported result exactly matches the requested value.

An exact match produces status **0**. A mismatch triggers a second dispatched operation whose result selects status **34** or **5**.

This is an equality-based completion check. The supplied evidence does not establish that the requested value or reported result represents bytes, elements, a position, or any other particular unit. It also does not establish that either nonzero status belongs to a particular error-code namespace.

## Interface

The function has four inputs and one direct return value.

| Input | Established role | Width and interpretation |
|---|---|---|
| Object reference | Supplies the dispatch-table reference and is passed to both dispatched operations. | Address of storage whose first word supplies an address-bearing dispatch-table value. The supplied representation treats that word as a 32-bit integer; the native pointer ABI is not independently established. |
| Requested value | Passed to the first dispatched operation and compared with its result. | 32-bit integer. No range restriction or unit is established. |
| Operation argument | Passed unchanged to the first dispatched operation. | Opaque 32-bit value. It may encode an address or another kind of value, but its meaning is unknown. |
| Result destination | Receives the first dispatched operation’s reported result. | Address of writable storage for one 32-bit integer. |

The direct return value is a 32-bit status:

| Status | Selection condition |
|---|---|
| **0** | The first operation’s reported result equals the requested value. |
| **34**, hexadecimal **0x22** | The results differ, and the second operation returns zero. |
| **5** | The results differ, and the second operation returns any nonzero value. |

These are all explicit return values in the supplied body. The function does not directly return either dispatched operation’s raw result.

### Validity preconditions

For the ordinary return behavior described here:

- The object reference must permit the initial dispatch-table-reference read.
- The selected dispatch table must permit reading a callable entry at byte offset **48**, hexadecimal **0x30**.
- That entry must be callable with the object reference, opaque operation argument, and requested value, in that order.
- The result destination must permit a 32-bit write after the first operation returns.
- On the mismatch path, the object must again permit reading its dispatch-table reference. The table then obtained must permit reading a callable entry at byte offset **24**, hexadecimal **0x18**.
- The mismatch-path entry must be callable with the object reference as its sole explicit argument.
- Each dispatched operation must return normally for the wrapper to proceed beyond that call.

There is no explicit null, range, alignment, destination-capacity, or dispatch-target validation. Invalid accesses or incompatible call targets have no established recovery behavior in this function.

## Behavior

### Initial dispatch

The function first reads the dispatch-table reference from the beginning of the object. It then selects the callable entry at byte offset **48** from that table.

The selected operation receives three arguments, in this observable order:

1. The original object reference.
2. The opaque 32-bit operation argument.
3. The requested 32-bit value.

The order is significant: the requested value is the function’s second input but the dispatched operation’s third explicit argument.

The first operation’s return is treated as a 32-bit integer. This contract calls that value the **reported result**.

No meaning beyond that role is established. In particular, the wrapper does not distinguish a negative result from a nonnegative result by sign, and it does not interpret any particular result as an error sentinel.

### Result publication

After the first operation returns, the function writes the reported result to the result destination.

This write precedes both:

- Selection of the success or mismatch path.
- Any invocation of the second operation.

The write occurs on every path that reaches this point, including paths that later return a nonzero status. There is no explicit rollback or replacement write by F-318de50a3eda.

Consequently, the destination is an output for the first operation’s report, not an output for the final status. The returned status and stored report must be consumed separately.

The value written is exactly the first operation’s reported 32-bit result. The wrapper does not clamp it, convert it to the requested value, or replace it with the second operation’s return.

### Exact-match path

The function compares the reported result with the requested value.

If they are equal, it returns **0** immediately. The second dispatch-table entry is not read or called on this path.

The comparison is exact equality, not an ordered or magnitude comparison. A reported value greater than the request is not accepted merely because it exceeds the request. Likewise, matching negative values satisfy the equality test; the wrapper itself imposes no nonnegative-value precondition.

The requested value used for this comparison is the function’s scalar input. No memory-backed request value is reread after the first operation.

### Mismatch path

If the reported result differs from the requested value, the function reads the dispatch-table reference from the object again. It then selects the callable entry at byte offset **24** from that table and invokes it with the object reference.

This is a fresh object-field read after both the first call and the result-destination write. The second dispatch does not necessarily use the table observed for the first dispatch.

The second operation’s result is classified only as zero or nonzero:

- Zero selects return status **34**.
- Any nonzero value selects return status **5**.

A negative nonzero value and a positive nonzero value select the same status. The magnitude of this result is not otherwise used.

The wrapper does not explicitly write the second operation’s result to the result destination. However, the second operation may have its own memory effects, so preservation of the destination through that call is not established.

### Ordering and aliasing

The established sequence on a normally returning mismatch path is:

1. Read the object’s dispatch-table reference.
2. Read the first callable entry at byte offset 48.
3. Invoke the first operation.
4. Write its reported result to the result destination.
5. Determine that the report differs from the request.
6. Reread the object’s dispatch-table reference.
7. Read the second callable entry at byte offset 24.
8. Invoke the second operation.
9. Return status 34 or 5 according to its zero/nonzero result.

No disjointness requirement between the object and the result destination is shown.

If the result destination aliases the object’s dispatch-table-reference storage, the publication write can change the table reference used on the mismatch path. Similarly, an aliased destination can alter storage holding the later callable entry. Such effects are not repaired or prevented.

Aliasing alone is not established as invalid. Its consequences depend on the resulting storage contents and whether all subsequent reads and calls remain valid. A success path performs no further dispatch-table read after publishing the result, even if that write changes the object.

## Side effects

The wrapper itself explicitly performs one caller-visible memory write: publication of the first operation’s reported result to the result destination.

It also invokes:

- Exactly one dispatched operation on the exact-match path.
- Two dispatched operations, in the stated order, on the mismatch path.

These call counts describe paths where the required accesses and calls complete normally.

No explicit allocation, deallocation, retry, logging, synchronization, or resource-release operation appears in the wrapper. This does **not** establish that its dependencies lack those effects.

The first operation may alter the object, dispatch tables, destination storage, global state, or other reachable state before returning. The wrapper’s publication write happens afterward. The second operation may likewise alter state after publication.

Ownership is not transferred or discharged by any explicit wrapper operation. Whether either dependency acquires, retains, consumes, or releases resources is unknown. Callers must not infer memory ownership from the opaque operation argument or from the wrapper’s return status.

A nonzero status does not imply that the first operation had no effect. The first call has already completed and its report has already been written before the mismatch status is selected.

## Dependencies

The following opaque IDs are local contract labels for unresolved dynamic dependencies. They are not recovered global identities and do not imply that the two entries must resolve to distinct implementations.

| Opaque ID | Resolution | Established interaction |
|---|---|---|
| **F-D01** | Entry at byte offset 48 in the dispatch table read before the first call. | Receives the object reference, opaque 32-bit argument, and requested 32-bit value; supplies the reported 32-bit result. |
| **F-D02** | Entry at byte offset 24 in the dispatch table reread after a mismatch and after result publication. | Receives the object reference; supplies a result tested only for zero versus nonzero. |

Neither dependency’s implementation, side effects, ownership behavior, or error conventions are supplied.

F-D02 is not established to be a pure predicate or an error-query operation. Those interpretations might fit the control flow, but the only established use of its result is the selection between statuses 34 and 5.

The supplied dependency mapping identifies F-318de50a3eda itself; it does not identify either indirect target.

## Acceptance examples

These are conditional behavioral examples, not executed tests. Each assumes valid required accesses, compatible call targets, and normal returns.

### Exact positive match

Given a requested value of **12**, if F-D01 reports **12**:

- The result destination is written with **12**.
- F-D02 is not invoked.
- The function returns **0**.

### Zero request and zero report

Given a requested value of **0**, if F-D01 reports **0**:

- The destination is written with **0**.
- The function returns **0** without invoking F-D02.

A zero request does not bypass the first operation.

### Matching negative values

Given a requested value of **−1**, if F-D01 reports **−1**:

- The destination is written with **−1**.
- The exact-match path returns **0**.

This example establishes the wrapper’s comparison behavior only. Whether F-D01 accepts such a request remains unknown.

### Short report with zero secondary result

Given a request of **12**, an F-D01 report of **7**, and an F-D02 result of **0**:

- The destination is written with **7** before F-D02 is called.
- The function returns **34**.

If F-D02 leaves the destination unchanged, it still contains **7** on return.

### Excess report with nonzero secondary result

Given a request of **12**, an F-D01 report of **13**, and an F-D02 result of **−3**:

- The report is a mismatch despite exceeding the request.
- The destination is written with **13** before the second call.
- The function returns **5**.

A positive nonzero F-D02 result would select the same status.

### Dispatch-table mutation

If F-D01 changes the object’s dispatch-table reference and then reports a mismatch, the second operation is resolved using the reference reread after publication, not necessarily the original table.

If the publication write also changes that reference through aliasing, the post-write contents govern the second lookup.

### Secondary mutation of the destination

If F-D02 can reach and modify the result destination, its write may supersede the wrapper’s earlier publication. The wrapper still selects its return status from F-D02’s returned zero/nonzero value and performs no explicit restoration.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning and units of the requested value and reported result.
- Whether the opaque 32-bit argument represents data, an address, a handle, or something else.
- The actual implementations selected by either dispatch entry.
- Whether dispatch-table contents remain stable during either call.
- The full pointer ABI and callable-entry width.
- Required alignment and platform-specific behavior for invalid accesses.
- Whether either dependency preserves globals, object fields, or destination storage.
- Any allocation, release, ownership-transfer, or resource-lifetime behavior inside the dependencies.
- Whether statuses 34 and 5 have external symbolic names or standardized meanings.
- Concurrency, reentrancy, and visibility guarantees beyond the stated evaluation order.
- Behavior when a dependency does not return normally.

The apparent intent is to distinguish exact completion from two kinds of mismatch. That is an inference from the comparison and status selection, not proof of a particular I/O or resource-management API.

## Evidence

Established static evidence consists of the supplied function representation: two indirect-call sites at byte offsets **48** and **24**, an unconditional post-first-call 32-bit destination write, an equality comparison against the requested value, and explicit return statuses **0**, **34**, and **5**.

The second table reference is read only on mismatch and only after result publication. The representation shows no arithmetic on the request or report, no retry, and no explicit ownership operation.

The 32-bit scalar descriptions follow the supplied type representation. Native ABI details and dependency semantics are not independently verified.

No runtime tests have been performed. The acceptance examples are conditional consequences of the static behavior, not observed executions.
