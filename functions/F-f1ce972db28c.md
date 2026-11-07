<!-- Trusted provenance: F-f1ce972db28c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f1ce972db28c

## Purpose

F-f1ce972db28c accepts a nonempty input region when its requested length fits within a state object’s recorded remaining space. It invokes an external transfer operation, advances the recorded position, and, when the resulting position reaches or exceeds the recorded endpoint, performs completion processing and clears three state fields.

Completion processing includes a conditionally invoked notification, an unconditional resource-handling call, and destruction or finalization of a temporary helper.

**Established behavior:** the ordinary return value is either 4 for a rejected request or 0 for an accepted request. Acceptance does not necessarily mean the whole accumulated operation is complete.

**Inferred intent:** this is an incremental buffer-fill operation with completion notification and resource cleanup. The meanings of the notification fields, resource, and completion-context values are not established.

## Interface

### Inputs and result

The function has three inputs:

| Input | Representation | Role |
|---|---|---|
| State address | 32-bit address-like value | Locates mutable state and notification fields. |
| Input address | 32-bit address-like value | Passed to the external transfer operation; zero is rejected. |
| Requested length | Unsigned 32-bit value | Passed to the transfer operation and used in position arithmetic; zero is rejected. |

The result is a 32-bit value:

- **4:** the input address is zero, the requested length is zero, or the requested length exceeds the computed remaining space.
- **0:** the request passes those checks and execution returns normally.

No result from a dependency is inspected as a transfer-success indicator. In particular, a normal return of 0 does not independently prove that the external primitive copied the requested bytes.

### State layout

Offsets are byte offsets from the state address.

| Offset | Width | Observed use |
|---|---:|---|
| `0x308` | 8 bits | First notification payload field. |
| `0x30A` | 16 bits | Second notification payload field. |
| `0x30C` | 16 bits | Third notification payload field. |
| `0x310` | 32 bits | Last notification payload field. |
| `0x314` | 32 bits | Fourth notification payload field. |
| `0x318` | 32 bits | Fifth notification payload field and later resource-handling argument. Cleared during completion. |
| `0x31C` | 32 bits | Current transfer position or destination address. Advanced on acceptance and cleared during completion. |
| `0x320` | 32 bits | Endpoint. Cleared during completion. |

The last offset is decimal 800. The fields at `0x31C` and `0x320` are distinct adjacent 32-bit fields.

The function also uses:

- An eight-byte temporary helper containing two 32-bit words.
- An eight-byte output area containing two 32-bit words, explicitly initialized to zero before a helper operation.
- A globally referenced context whose 32-bit field at offset `0x14` participates in the notification decision.

### Preconditions

For defined ordinary operation:

- If both the input address and requested length are nonzero, the state fields needed for the capacity check must be readable.
- An accepted request requires the position field to be writable.
- The external transfer operation’s own address-validity and overlap requirements must hold.
- Completion requires access to the additional state fields, global context, helper objects, and dispatch entries described below.
- Dependencies must return normally for the ordinary result contract to apply.

The function performs no explicit null check on the state address, no explicit validation of the global context, and no explicit check of the object returned for notification dispatch.

## Behavior

### 1. Reject an empty or oversized request

The default result is 4.

The input-address check and nonzero-length check precede the state-dependent capacity check. If either input value is zero, the function returns 4 without explicitly accessing the state object or invoking a dependency.

Otherwise, it obtains the endpoint and current position and compares the requested length against their difference, interpreted as an unsigned 32-bit quantity:

- A length greater than that difference is rejected.
- A length equal to that difference is accepted.
- A nonzero length smaller than that difference is accepted.

The relative load order of the endpoint and position within the subtraction is not established.

There is no separate check that the endpoint is at or above the current position. At the represented 32-bit level, a negative endpoint-minus-position difference becomes a large unsigned value. Consequently, an inconsistent state can pass the capacity check. The mathematical “remaining space” interpretation requires an additional caller-maintained invariant; this function does not enforce it.

### 2. Invoke the transfer operation and advance the position

For an accepted request, the function invokes the external transfer primitive with these arguments, in this order:

1. The current value of the position field.
2. The input address.
3. The requested length.

The call shape suggests a destination, source, and byte count. The primitive’s implementation is not supplied, so exact copy, overlap, failure, and memory-access behavior remain dependency requirements rather than independently established behavior.

After that call returns, the function selects result 0.

It then **reloads** the position field, adds the requested length, and writes the resulting 32-bit value back to the position field. Thus, the value advanced is not necessarily the position used in the capacity check or passed to the transfer primitive. Any change made through an alias during the transfer call can affect this update.

The address-sized addition has 32-bit behavior; it is not a widened address calculation.

The function next reloads the endpoint and performs an unsigned comparison against the newly computed position:

- If the endpoint is greater than the new position, the function returns 0 without completion processing.
- If the endpoint is less than or equal to the new position, it enters completion processing.

This comparison admits overshoot. It is not an equality-only completion check.

Under the additional assumptions that the position and endpoint remain unchanged during the transfer call, the address range does not wrap, and the endpoint is not below the original position, the initial capacity check prevents overshoot. Those assumptions are not enforced across the external call.

### 3. Obtain a completion-context value

On completion, the function performs these operations in order:

1. Calls F-7a117b899bf6 with the address of the eight-byte temporary helper.
2. Sets both 32-bit words of the separate output area to zero.
3. Invokes the temporary helper’s dispatch entry at table byte offset `0x0C`, passing the helper address and output-area address.
4. Calls F-bf18a6a8124b with the output-area address.
5. Reads the globally referenced context’s 32-bit field at offset `0x14`.
6. Computes an unsigned 32-bit difference between the value returned by F-bf18a6a8124b and that global field.

The global field is read after F-bf18a6a8124b returns. Its earlier value, if any, is not captured for this decision.

The helper initialization call precedes zeroing the output area. The output area is therefore guaranteed to start as two zero words immediately before the dispatch operation, not before helper initialization.

### 4. Conditionally dispatch a notification

A notification is eligible exactly when the unsigned difference described above is at most `0x3A98`, inclusive.

The represented condition splits this range around `0x3800`, but its accepted set is continuous:

- `0x00000000` through `0x000037FF`: accepted.
- `0x00003800` through `0x00003A98`: accepted.
- `0x00003A99` through `0xFFFFFFFF`: rejected.

This is an unsigned modular-difference test, not a signed-distance or absolute-distance test.

If the difference is eligible:

1. F-3d00a9979740 is called with no explicit arguments.
2. Its returned address is treated as an object with a dispatch-table pointer.
3. The dispatch entry at table byte offset `0x30` is invoked.

The notification receives the returned object as receiver, followed by these state payloads in order:

| Argument after receiver | State offset | Width read |
|---:|---|---:|
| 1 | `0x308` | 8 bits |
| 2 | `0x30A` | 16 bits |
| 3 | `0x30C` | 16 bits |
| 4 | `0x314` | 32 bits |
| 5 | `0x318` | 32 bits |
| 6 | `0x310` | 32 bits |

These fields are read after F-3d00a9979740 returns. They are not snapshots taken before helper processing. Their relative evaluation order within the dispatch call is not established.

The precise signedness and argument-extension rules for the narrow fields are not established by their storage widths alone.

If the difference is outside the accepted range, both F-3d00a9979740 and the notification dispatch are skipped.

### 5. Handle the resource and clear completion state

Whether or not notification occurs, completion continues in this order:

1. Reload the 32-bit field at `0x318` and pass it to F-5e3309c9d083.
2. After that call returns, write zero to `0x318`.
3. Write zero to `0x31C`.
4. Write zero to `0x320`.
5. Call F-ec22bd8215d3 with the temporary helper’s address.
6. Return 0 if execution continues normally.

The resource argument is reloaded after any notification. A notification that changes `0x318` can therefore change the value passed to F-5e3309c9d083.

There is no explicit nonzero check before that call: a zero value is passed if the field contains zero.

The three zero writes occur after the resource-handling call and before temporary-helper finalization. They do not establish an unconditional final-memory guarantee against later writes by F-ec22bd8215d3 or other aliasing activity.

## Side effects

### Rejection

A rejected request causes no explicit state writes and no dependency calls. For nonempty requests, rejection can still involve reads of the position and endpoint.

### Accepted, incomplete request

The explicit effects are:

- One external transfer call.
- One write to the position field.

The function does not explicitly clear the resource or endpoint and does not construct the completion helper.

### Accepted, complete request

In addition to the transfer and position update:

- Temporary helper state is initialized and later finalized.
- The output area is zeroed and exposed to helper dependencies.
- A global context field is read.
- A notification may run.
- F-5e3309c9d083 is called with the then-current resource field.
- Resource, position, and endpoint fields are explicitly zeroed, in that order.

No other state fields are explicitly modified by this function. Dependencies may have additional effects; none may be assumed to preserve globals, preserve the state object, or refrain from reentrant calls.

### Ownership and failures

Passing the input address to the transfer primitive does not itself establish ownership transfer.

The call involving `0x318`, followed by clearing that field, suggests resource disposal or relinquishment. Actual release, reference-count, ownership, and zero-value semantics are unknown.

No dependency error is converted into a distinct status code. There is no visible recovery path for faults, failed dispatch, exceptions, nonlocal exits, or dependencies that fail to return. For example, if resource handling does not return, the subsequent three zero writes are not reached.

## Dependencies

| Dependency | Established use | Not established |
|---|---|---|
| External transfer primitive; opaque ID unavailable | Called once for each accepted request with position, input address, and length. | Exact transfer semantics, overlap rules, return convention, failure behavior, and mutation of shared state. |
| F-7a117b899bf6 | Receives the temporary helper address at completion entry. | Exact initialized layout, ownership, and global effects. |
| Helper dispatch at byte offset `0x0C` | Receives the helper and zero-initialized output area. | Output meaning and additional effects. |
| F-bf18a6a8124b | Receives the output area and returns a 32-bit value used in the range decision. | Units, interpretation, ownership, and global effects. |
| F-3d00a9979740 | Supplies the notification object on the eligible branch. | Object lifetime, nullability, ownership, and shared-state effects. |
| Notification dispatch at byte offset `0x30` | Receives six payload fields in the specified argument order. | Callback semantics, return meaning, and mutations. |
| F-5e3309c9d083 | Receives the reloaded field at `0x318` on every completed request. | Whether it frees, releases, unregisters, or performs another operation. |
| F-ec22bd8215d3 | Receives the temporary helper after the three zero writes. | Finalization details and effects on other memory or globals. |

## Acceptance examples

These examples are conditional static predictions, not executed tests. Unless stated otherwise, dependencies return normally and do not modify relevant state.

1. **Zero input address.**  
   A zero input address and any length produce result 4. No explicit state access or dependency call occurs.

2. **Zero length.**  
   A nonzero input address and zero length produce result 4, likewise without explicit state access.

3. **Oversized request.**  
   With position `0x1000`, endpoint `0x1010`, and length 17, the result is 4. No transfer occurs and neither field is written.

4. **Partial fill.**  
   With position `0x1000`, endpoint `0x1010`, and length 8, the transfer receives destination `0x1000` and length 8. Position becomes `0x1008`; result is 0. Completion processing does not run.

5. **Exact fill.**  
   With the same initial fields and length 16, position is first written as `0x1010`. Completion runs. Resource handling precedes zeroing the resource, position, and endpoint fields.

6. **Notification boundaries.**  
   Completion differences `0`, `0x37FF`, `0x3800`, and `0x3A98` invoke the notification path. Differences `0x3A99` and `0xFFFFFFFF` do not. Resource handling and the three zero writes occur in either case.

7. **Notification replaces the resource.**  
   If notification changes `0x318` from one value to another, the later resource-handling call receives the replacement value, not a cached pre-notification value.

8. **Transfer changes the position.**  
   If a request passes the initial check but the transfer dependency changes the position field, the subsequent increment uses that changed field. Completion uses the resulting position and the endpoint reloaded afterward.

9. **Inconsistent bounds.**  
   Position `0x2000` and endpoint `0x1000` yield an unsigned 32-bit difference of `0xFFFFF000`. A small nonzero length therefore passes the capacity test. This is not evidence that the destination region is valid.

## Uncertainties

The following remain **documented_only**:

- The external transfer primitive’s precise behavior.
- The semantic types and units of notification fields and completion-context values.
- Narrow-argument signedness and extension.
- Ownership obligations for the input, resource, helper outputs, and notification object.
- Whether dependencies mutate relevant state or global data.
- Thread safety, synchronization, and reentrancy guarantees.
- Recovery behavior for dependency failures.
- Required alignment and validity rules beyond the observed accesses.
- Behavior intended for wrapped address arithmetic or inconsistent position/endpoint state.

The buffer-fill interpretation is inferred intent. The field widths, branch conditions, call ordering, reload points, and explicit writes are established static observations.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping. The supplied material supports the two return values, state offsets and widths, acceptance checks, unsigned range boundary, dependency sequence, payload order, reload points, and completion writes described above.

No dependency implementations or runtime observations were supplied. No runtime tests have been performed.
