<!-- Trusted provenance: F-c20bd8e1d6d9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c20bd8e1d6d9

## Purpose

F-c20bd8e1d6d9 records a requested 32-bit value in a caller-supplied object, performs an unconditional sequence of dependency calls, obtains or reuses an associated object, and conditionally invokes that associated object with a binary control value.

The control value is zero when a dependency-reported value equals the original requested value. Otherwise, it is one, unless a particular masked flag pattern suppresses the final invocation.

This is a source-facing behavioral contract. The observable ordering and conditions below are established by the supplied static representation. The apparent intent—synchronizing or selecting state across related objects—is inferred, not established. No runtime tests have been performed.

## Interface

### Inputs and result

The function receives:

- **Primary object:** a pointer to writable object storage containing the fields described below.
- **Requested value:** a 32-bit integer bit pattern.

There is no explicit return value. The function can return early after specified dependency results or a flag check. Those returns do not undo earlier writes or calls.

Equality comparisons use the complete 32-bit values. There is no range check, normalization, or arithmetic transformation of the requested value.

### Object layout

The supplied representation uses four-byte words and 32-bit pointer storage. The following byte offsets describe that represented layout; they are not a portable layout for a different ABI.

| Storage | Byte offset | Represented content | Use |
|---|---:|---|---|
| Primary object | `0x000` | Dispatch-table pointer | Resolves calls at slot offset `0x58` |
| Primary object | `0x73C` | 32-bit value | Receives the requested value |
| Primary object | `0x750` | Associated-object pointer | Tested for zero, conditionally assigned, and later reread |
| Primary object | `0x778` | Provider-object pointer | Receives the result of F-1093fa967285 |
| Primary object | `0x88C` | Service-object pointer | Supplies dispatches at slot offsets `0x88` and `0x60` |
| Associated object | `0x000` | Dispatch-table pointer | Resolves the final call at slot offset `0xA0` |
| Associated object | `0x048` | 32-bit flags | Read only on the unequal-value path |
| Provider object | `0x000` | Dispatch-table pointer | Resolves the call at slot offset `0xF8` |

Dispatch offsets are byte offsets into the referenced tables, not slot indices.

Three external four-byte data values also participate. This contract calls them **data A**, **data B**, and **data C**. Their contents, semantic types, mutability, and meanings are not supplied.

### Preconditions

Successful execution requires valid readable or writable storage for each access actually reached:

- The primary object must support all reached field accesses.
- Each reached dispatch-table pointer and function entry must be usable.
- The service-object field must identify suitable storage whenever a service dispatch is resolved.
- An associated object used after the second service call must support its reached dispatch and, on the unequal path, its flag read.
- A nonzero provider result must support its dispatch at `0xF8`.

These conditions are not validated by the function. Null checks exist only for the provider result and for the result of its `0xF8` dispatch during lazy acquisition.

The complete indirect-call ABI is not established. In particular, the two service dispatches have no explicit arguments in the supplied representation. An implicit receiver or additional machine-level argument must not be assumed.

## Behavior

### 1. Record the request and perform unconditional calls

The function first writes the requested value to the primary object at `0x73C`. This write precedes every dependency call.

It then performs these operations in order:

1. Call F-3ac76db3f868 with no explicit arguments.
2. Resolve a service dispatch using the primary object's current pointer at `0x88C`, the dispatch-table pointer stored at that service object's start, and table offset `0x88`.
3. Invoke that entry with no explicit arguments and retain its four-byte result.
4. Call F-a47f88e3d6eb with the primary-object pointer and that result.

The function does not inspect the return of F-3ac76db3f868 or the return of F-a47f88e3d6eb.

These calls precede the test for an existing associated object. They therefore occur even when the associated object is already available or when subsequent acquisition fails.

### 2. Perform two primary-object dispatches

After F-a47f88e3d6eb returns, the function reads data A and retains a four-byte snapshot.

It next invokes the primary object's dispatch at table offset `0x58` twice:

- The first invocation has the explicit arguments: primary-object pointer, a read of data A, and a read of data B.
- The second invocation has the explicit arguments: primary-object pointer and the retained snapshot of data A.

The two calls use the same table offset but have different explicit argument counts. This is an evidence limitation, not a basis for adding a missing argument or declaring two different methods.

The dispatch is resolved from the primary object for each invocation. The second invocation is not specified to reuse the first invocation's table or entry. Thus, a table change made by the first call can affect the second call.

The second invocation uses the earlier snapshot of data A, not a new read after the first invocation. If the first invocation changes data A, that change does not replace the retained argument.

The supplied representation does not establish a strict order among the independent dispatch-table and argument reads within either call expression.

### 3. Reuse or acquire an associated object

Only after both `0x58` invocations does the function read the primary-object field at `0x750`.

If that field is nonzero, acquisition is skipped. The function does not call F-1093fa967285 and does not directly write the field at `0x778` on this path.

If the field is zero, the following sequence occurs:

1. Call F-1093fa967285 with no explicit arguments.
2. Store its result into the primary-object field at `0x778`.
3. If that result is zero, return immediately.
4. Otherwise, resolve table offset `0xF8` through the returned provider object.
5. Invoke that entry with the provider-object pointer and data C as explicit arguments.
6. Store the returned 32-bit value into the primary-object field at `0x750`.
7. If that returned value is zero, return immediately.

The provider result is stored even when it is zero. The associated-object result is likewise stored before it is tested for zero.

The zero-result branches test the returned values retained by this invocation. They are not additional reads of the corresponding primary-object fields.

A nonzero associated-object field is sufficient to bypass acquisition. No flag, provider identity, or validity check is performed at this stage.

### 4. Query the service and reread the associated object

If execution continues, the function resolves and invokes the service dispatch at table offset `0x60`, again using the primary object's current field at `0x88C`. This call has no explicit arguments in the supplied representation.

After that call returns, the function rereads the associated-object pointer from the primary object at `0x750`.

This ordering matters:

- The second service dispatch need not use the service object used for the earlier `0x88` dispatch.
- The associated object used below need not be the one observed before acquisition or the one returned during acquisition.
- A change to `0x750` made by the `0x60` dependency is observed.
- There is no new null check after this reread.

The returned service value is compared with the original requested argument, not with the current contents of the field at `0x73C`. A dependency's modification of that field does not change this comparison.

### 5. Select or suppress the final invocation

If the service result equals the original requested value:

- Do not read the associated object's flags at `0x48` for this decision.
- Resolve its dispatch at table offset `0xA0`.
- Invoke it with the associated-object pointer and a 32-bit zero.

If the service result differs from the original requested value:

- Read the associated object's 32-bit flags at `0x48`.
- Mask them with `0x1800`.
- If the masked result is exactly `0x0800`, return without invoking `0xA0`.
- Otherwise, resolve its dispatch at table offset `0xA0` and invoke it with the associated-object pointer and a 32-bit one.

Only bits selected by `0x1800` affect suppression:

| Masked flags | Unequal-value behavior |
|---:|---|
| `0x0000` | Invoke `0xA0` with one |
| `0x0800` | Return without invoking `0xA0` |
| `0x1000` | Invoke `0xA0` with one |
| `0x1800` | Invoke `0xA0` with one |

Other flag bits do not affect this decision.

The final entry is invoked indirectly. Its return value is not used, and no subsequent object access is represented. Whether this transfer is implemented as a conventional call or a tail transfer is not established.

## Side effects

The function directly performs these writes:

1. Always, before any call: requested value to primary offset `0x73C`.
2. On the acquisition path: provider result to primary offset `0x778`.
3. After a nonzero provider result: provider-dispatch result to primary offset `0x750`.

No direct allocation, release, reference-count update, flag update, or clearing of the associated object is shown.

Storing a returned pointer does not establish ownership transfer. The function does not visibly release an old provider value before overwriting `0x778`, nor release a returned provider if its `0xF8` dispatch yields zero. Whether either operation is required depends on undocumented dependency contracts.

Every dependency may have additional side effects. No preservation of primary-object fields, dispatch tables, external data, global state, or object lifetime may be assumed. Consequently, the direct writes describe ordered events, not guaranteed final field values after all calls finish.

The function has no explicit error code or diagnostic. Its three conditional early-return reasons are:

- Zero provider result.
- Zero associated-object result during acquisition.
- Unequal service result with masked flags equal to `0x0800`.

Invalid pointers, invalid dispatch entries, dependency failures, and nonlocal control transfer have no recovery behavior specified here.

## Dependencies

| Dependency | Explicit inputs | Result used | Position |
|---|---|---|---|
| F-3ac76db3f868 | None | None | Immediately after recording the request |
| Service dispatch `0x88` | None shown | Four-byte value | Before F-a47f88e3d6eb |
| F-a47f88e3d6eb | Primary pointer, service result | None | Before the two primary dispatches |
| Primary dispatch `0x58`, first invocation | Primary pointer, data A, data B | None | Before associated-object test |
| Primary dispatch `0x58`, second invocation | Primary pointer, retained data A | None | Immediately after first invocation |
| F-1093fa967285 | None | Provider pointer | Only when `0x750` is zero |
| Provider dispatch `0xF8` | Provider pointer, data C | Associated-object value | Only after nonzero provider result |
| Service dispatch `0x60` | None shown | Value compared with original request | After reuse or successful acquisition |
| Associated dispatch `0xA0` | Associated pointer, zero or one | None | Final action unless suppressed |

The named dependencies use the supplied opaque identifiers. Indirect dependencies have no supplied opaque function identities and are identified only by their dispatch locations.

## Acceptance examples

These examples are conditional static expectations, not runtime observations. Unless stated otherwise, assume valid storage, normally returning dependencies, and no dependency mutation of relevant state.

### Existing association, equal service result

Given a nonzero associated-object field and a service `0x60` result equal to the requested value:

- The request write and unconditional calls occur first.
- Acquisition is skipped.
- The final associated dispatch receives zero.
- Associated flags are not read by this function, even if their masked value would be `0x0800`.

### Existing association, unequal result with suppression

Given a nonzero associated-object field, an unequal service result, and flags whose `0x1800` mask equals `0x0800`:

- The unconditional sequence and service query occur.
- The flags are read.
- The function returns without resolving or invoking the final `0xA0` entry.

For example, flags `0x0801` suppress the call; flags `0x1801` do not.

### Provider acquisition fails

Given a zero associated-object field and a zero result from F-1093fa967285:

- Zero is written to `0x778`.
- No provider `0xF8` dispatch occurs.
- No service `0x60` dispatch or final associated dispatch occurs.
- Earlier calls and the request write remain observable.

### Associated-object acquisition fails

Given a zero associated-object field, a nonzero provider result, and a zero provider-dispatch result:

- The provider is stored at `0x778`.
- Zero is stored at `0x750`.
- Execution returns before the second service query.
- No release operation is directly performed.

### Dependency replaces the association

Suppose the `0x60` service dependency changes `0x750` from one valid associated object to another. The final flags and dispatch accesses use the replacement object.

If it instead clears `0x750`, the function has no protective null check before the later associated-object dereference. Safe behavior is not established for that case.

### Saved request field changes

Suppose a dependency overwrites `0x73C`. Equality still uses the original requested argument. The replacement field value does not control whether the final argument is zero or one.

### Data A changes during the first primary dispatch

Suppose the first `0x58` invocation changes data A. The second invocation still receives the snapshot taken before the first invocation.

## Uncertainties

The following remain **documented_only**:

- The semantic purpose of the request, objects, external data, flags, and binary control argument.
- Exact indirect-call signatures, including the inconsistent explicit argument counts at `0x58`.
- Any implicit receiver convention for the service dispatches.
- Ownership, allocation, retention, and release obligations for returned objects.
- Dependency side effects and lifetime guarantees.
- Concurrency, synchronization, volatile-access semantics, and atomicity.
- Behavior on invalid memory, failed dependencies, exceptions, or other abnormal transfers.
- Whether the final indirect transfer has ordinary call or tail-transfer mechanics.

The 32-bit layout is established for the supplied representation. Compatibility with a different pointer width or object ABI is not established.

## Evidence

The supplied static representation establishes the direct field accesses, dispatch offsets, explicit argument lists, branch conditions, result assignments, and call ordering described above.

Its substantive numerical operations are four-byte indexing, 32-bit equality, zero tests, and a bitwise mask with `0x1800`. No request-value arithmetic, bounds validation, retry, rollback, or explicit resource release is shown.

The behavioral interpretation is independent of semantic names not established by those operations. No runtime execution, dynamic ABI validation, or dependency implementation inspection has been performed.
