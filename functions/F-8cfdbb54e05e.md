<!-- Trusted provenance: F-8cfdbb54e05e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8cfdbb54e05e

## Purpose

F-8cfdbb54e05e conditionally replaces an object reference held by an owner. It compares a requested 32-bit value with a field in the currently referenced object. If the values match, it leaves the reference unchanged and returns zero. Otherwise, it performs an optional callback, invokes an operation on any object currently held, constructs a replacement through dependencies, stores the returned value, invokes a follow-on dependency, and returns one.

A request of zero has special construction behavior: it selects a value from the owner and one of two global values. The initial comparison nevertheless uses the original zero request, not the owner-derived value.

This is a behavioral contract derived from static evidence only. Descriptions such as “owner,” “replacement,” and “construction” identify observable roles; they do not establish application-level types, allocation ownership, or successful initialization.

## Interface

### Inputs

The function takes three inputs:

| Input | Represented form | Role |
|---|---|---|
| Owner | 32-bit address-bearing value | Identifies storage containing the current-object reference and a default value. |
| Requested value | 32-bit integer | Compared with the current object’s field; zero also selects the default-construction branch. |
| Optional callback object | Nullable pointer | When non-null, supplies a dynamically dispatched operation used during replacement. It is also passed to the follow-on dependency. |

The supplied representation uses 32-bit integers and four-byte pointer-bearing fields. The contract below uses that represented layout. A different native pointer width or binary calling convention is not established.

### Return value

The result is a four-byte value:

- **0:** The owner had a non-null current-object reference whose field at offset 12 equaled the original requested value.
- **1:** The replacement path reached its final return.

The result is a path indicator, not an established success or failure code for allocation, construction, cleanup, or notification. In particular, the replacement dependency’s return value does not determine whether the function returns one.

There is no explicit error return.

### Referenced layouts

Offsets are byte offsets from the indicated base.

| Storage | Offset | Width | Observed interpretation |
|---|---:|---:|---|
| Owner | 24 | 4 bytes | Current-object reference; zero means absent. |
| Owner | 44 | 4 bytes | Default value read only for a zero request on the replacement path. |
| Current object | 0 | 4 bytes | Dispatch-table reference used for its cleanup-like operation. |
| Current object | 12 | 4 bytes | Value compared with the original request. |
| Current object’s dispatch table | 4 | 4 bytes | Target of the cleanup-like indirect call. |
| Callback object | 0 | 4 bytes | Dispatch-table reference. |
| Callback object’s dispatch table | 400 | 4 bytes | Target called with the callback object and zero. |

The dispatch offsets are byte offsets, not source-language method numbers. No complete object layout is established.

### Preconditions

For defined execution, every dereference actually reached must refer to accessible storage of the required width. In particular:

- The owner must support access to its field at offset 24.
- A non-null current reference used in the comparison must support a four-byte read at offset 12.
- A non-null callback object used on the replacement path must provide a usable dispatch table and target at offset 400.
- A current object selected for the cleanup-like call must provide a usable dispatch table and target at offset 4.
- The owner must support a four-byte read at offset 44 when the zero-request construction branch is reached.
- Indirect targets must be compatible with the effective calling convention.

The function contains no owner-null check, object-validity check, dispatch-target check, or allocation-result check. Concurrency, reentrancy, and synchronization requirements are not supplied.

## Behavior

### Initial dependency call and comparison

Every invocation first calls **F-ea223a7ae124** with no explicit arguments and retains its four-byte result for possible use in replacement construction.

This call occurs even when the function ultimately returns zero. The current-object comparison happens after it. Consequently, the comparison observes owner state available after that dependency returns; it is not based on a reference captured before the call.

The function then examines the owner’s current-object field:

- If it is zero, replacement is required, and the object-field comparison is skipped.
- Otherwise, the function reads the current object’s field at offset 12 and compares it with the original requested value.
- Equality selects the zero-return path.
- Inequality selects the replacement path.

The source expression reads the owner’s reference field separately for the non-null test and for locating the object’s comparison field. It does not establish a stable captured reference across those reads. Under stable storage, both reads identify the same object. Behavior under intervening concurrent mutation is not established.

On the zero-return path, this function performs no callback dispatch, cleanup-like dispatch, allocation request, replacement construction, current-reference write, or follow-on notification. The initial dependency call has already occurred, and its own side effects remain possible.

### Optional callback before cleanup

On the replacement path, a non-null callback object receives an indirect call through the dispatch target at byte offset 400 of its dispatch table. The explicit arguments are:

1. The callback object itself.
2. A zero-valued argument.

A null callback object skips this indirect call.

The callback runs before the function checks which current object to clean up. The function does not retain the comparison-time current reference for that purpose. If the callback changes the owner’s current-object field, the subsequent cleanup stage observes the field again.

The callback’s result, if any, does not control the visible branch or return value.

### Cleanup-like dispatch and clearing

After the optional callback, the function reads the owner’s current-object reference again.

If it is non-null, the function resolves and invokes the indirect target at byte offset 4 of that object’s dispatch table. The supplied call representation shows **no explicit arguments** for this invocation. It therefore does not establish an explicit receiver argument or a specific source-language destructor signature.

Only after that call returns does the function write four zero bytes to the owner’s current-object field.

This ordering matters:

- The cleanup-like call is entered while the field has not yet been cleared by this function.
- If that call changes the owner’s current-object field, the subsequent zero write overwrites that field change.
- If the call does not return normally, the later zero write is not reached.
- If the post-callback field is already zero, both the indirect call and this clearing write are skipped.

The indirect operation’s name, ownership consequences, and ability to free memory are unknown.

### Replacement inputs and allocation request

The original requested value determines which construction branch is selected.

For an original request of zero:

1. **F-8157a180d901** is called with the size argument **36 bytes**.
2. The owner’s four-byte field at offset 44 is read as the effective construction value.
3. One branch-specific global four-byte value is read as a construction argument.

For an original request other than zero:

1. **F-8157a180d901** is called with the size argument **36 bytes**.
2. The original requested value remains the effective construction value.
3. A different branch-specific global four-byte value is read as a construction argument.

The default field and selected global value are read after the allocation dependency returns. They must not be treated as values captured earlier in the operation. The allocation dependency’s effects on shared state are unspecified.

The two global values are distinct source locations. Their values and semantic roles are not supplied. They may happen to contain equal values; the evidence establishes branch-dependent selection, not inequality of their contents.

### Construction, publication, and follow-on call

The function calls **F-326b1fb00786** with five explicit arguments, in this order:

1. The four-byte result of the 36-byte allocation request.
2. The four-byte result retained from the initial F-ea223a7ae124 call.
3. The effective construction value.
4. The selected global four-byte value.
5. Zero.

There is no local null-result check between allocation and this call.

The four-byte result from F-326b1fb00786 is written directly to the owner’s current-object field. The function does not check whether it is zero or whether it identifies valid storage.

After this write, **F-eddad957e718** is called with:

1. The owner.
2. The original callback-object argument, including null if that was supplied.
3. The value one.

Its result is not used to determine the function’s return. If this dependency returns normally, the function returns one.

The publication write precedes the follow-on call, but the published reference is not guaranteed to remain present afterward: the follow-on dependency may change shared state, and this function performs no restoring write after it.

## Side effects

The direct writes established by the supplied body are limited to the owner’s current-object field:

- A zero write after a non-null cleanup-like call returns.
- A replacement-result write after F-326b1fb00786 returns.

The first write is conditional; the second occurs on every replacement path that reaches that stage.

All called dependencies and indirect targets may have additional side effects. The evidence does not establish that they preserve globals, owner fields, referenced objects, or memory ownership.

The function requests a 36-byte allocation-like operation but contains no explicit deallocation operation whose semantics are known. It is not established that:

- The returned allocation is fresh or non-null.
- F-326b1fb00786 takes ownership of the allocation.
- Its returned value equals the allocation result.
- The cleanup-like dispatch frees the previous object.
- A failed construction releases any storage.
- The initial dependency’s result is a borrowed or owned resource.

There is no local rollback, retry, or error recovery. If execution fails after clearing the current reference but before publishing a replacement, the supplied body provides no restoration path. Whether a callee can throw, terminate, transfer control, or report failure is undocumented.

## Dependencies

| Dependency | Established use | Unknown contract |
|---|---|---|
| **F-ea223a7ae124** | Called first, with no explicit arguments; its result is forwarded to construction if replacement occurs. | Meaning of result, ownership, failures, and shared-state effects. |
| **F-8157a180d901** | Called once on the replacement path with a 36-byte size argument. | Allocation guarantees, initialization, alignment, failure behavior, and ownership. |
| **F-326b1fb00786** | Receives allocation result, initial dependency result, effective value, selected global value, and zero; its result is published. | Initialization semantics, input validity requirements, failure behavior, and ownership transfer. |
| **F-eddad957e718** | Called after publication with owner, original callback object, and one. | Notification or update semantics, state mutation, and failures. |

The two indirect operations have no supplied opaque function IDs. They are identified solely by their dispatch-table offsets and observed argument forms. No concrete target identity is established.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless stated otherwise, assume valid storage, normal returns, and no dependency-driven state changes beyond those described.

### Existing value matches a nonzero request

The owner references an object whose offset-12 field is 7, and the request is 7.

Expected behavior:

- F-ea223a7ae124 is called.
- The comparison matches.
- The result is zero.
- No replacement-path calls or direct owner-field writes occur.

A non-null callback argument does not change this outcome because its dispatch is not reached.

### Empty owner with a nonzero request

The current-object field is zero, and the request is 7.

Expected behavior:

- The optional callback dispatch occurs if a callback object was supplied.
- Assuming the field remains zero, cleanup-like dispatch is skipped.
- A 36-byte allocation request occurs.
- Construction receives 7 and the nonzero-request global value.
- The construction result is stored, the follow-on dependency is called, and the result is one.

### Zero request uses the owner default only during construction

The current object’s comparison field is 7, the request is zero, and the owner’s default field is 7.

Expected behavior:

- The comparison is against zero, so replacement occurs despite the default matching the current object’s value.
- Construction receives the owner default, 7, and the zero-request global value.

Conversely, if the current object’s comparison field is already zero, a zero request returns zero without reading the default field for construction.

### Callback changes the cleanup target

The comparison selects replacement with object A currently referenced. The callback changes the owner’s reference to object B.

Expected behavior:

- The cleanup stage uses the subsequently read reference to B.
- This function does not separately clean up A using a saved comparison-time reference.
- After B’s cleanup-like call returns, the owner’s reference is cleared.

### Construction returns zero

F-326b1fb00786 returns zero normally.

Expected behavior:

- Zero is stored in the owner’s current-object field.
- F-eddad957e718 is still called.
- If it returns normally, F-8cfdbb54e05e returns one.

### Follow-on dependency changes the published reference

Construction returns a nonzero reference, which is stored. The follow-on dependency then clears that field.

Expected behavior:

- The function returns one after the dependency returns.
- The field remains as left by that dependency; this function does not republish the construction result.

## Uncertainties

The following are **documented_only**:

- Application-level meaning of the compared value and owner default.
- Signed interpretation of nonzero values. The observed decisions use zero testing and 32-bit equality; no signed ordering or arithmetic is present.
- Native ABI details beyond the represented four-byte fields and explicit argument lists.
- Concrete identities and signatures of indirect targets.
- Contents and meaning of the two global construction arguments.
- Allocation, destruction, reference-counting, and ownership guarantees.
- Dependency behavior on invalid inputs or resource exhaustion.
- Thread safety, synchronization, and legal reentrant state transitions.
- Whether the final owner field denotes a successfully initialized object.

The inferred intent is a cached-object replacement operation with default selection. That interpretation does not strengthen any dependency guarantee or redefine the return value as success.

## Evidence

Established static evidence consists of the supplied function body and opaque dependency mapping. It establishes the branch conditions, field offsets, four-byte values, dependency argument ordering, allocation-size argument, direct writes, and return constants described above.

No runtime tests have been performed. The acceptance examples express conditional consequences of the static control flow, not observed execution results.
