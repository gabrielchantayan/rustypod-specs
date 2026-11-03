<!-- Trusted provenance: F-60d55fcdc4ef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-60d55fcdc4ef

## Purpose

F-60d55fcdc4ef conditionally performs indexed access through an indirectly acquired object. On the accepted path, it writes one 32-bit value to caller-provided storage, then obtains an address and returns the 32-bit value read from that address.

Three conditions gate that path:

1. An operation reached through the input object returns nonzero.
2. F-b3bed2dfc5ec returns a non-null object address.
3. The requested unsigned 32-bit index is strictly below a bound obtained from that object.

If any condition fails, the function returns zero without directly writing to the output location.

The output value and return value come from different operations. They are not established as equal or otherwise related. A successful final read can produce zero, so zero is not an unambiguous rejection indicator.

Indexed access is an inferred intent. The domain meanings of the objects, bound, index, and retrieved values are not established.

## Interface

### Arguments

The function receives four arguments in this order:

| Position | Established role | Width or representation |
|---|---|---|
| First | Input object address, used to resolve the initial indirect operation | Address; exact pointer ABI is not fully established |
| Second | Unused argument | 32-bit value in the supplied representation |
| Third | Index used in the bounds comparison and both indexed calls | Unsigned 32-bit value |
| Fourth | Address receiving the first indexed result | Storage for one 32-bit value |

The second argument has no visible use in the function. No required sentinel, reserved behavior, or ownership role is established for it.

The function does not directly access the output storage on a rejected path. Its validity is therefore required by this function only if execution reaches the output write. This does not constrain what dependencies might access through aliases or other state.

### Return value

The return value is 32 bits:

- Zero if the initial operation returns zero.
- Zero if F-b3bed2dfc5ec returns a null object address.
- Zero if the index is not strictly below the retrieved unsigned bound.
- Otherwise, the 32-bit value read through the address returned by the final indexed operation.

There is no separate status value. No visible mechanism converts invalid accesses, dependency exceptions, or other abnormal completion into a zero return.

### Dispatch layout

Each object used for dispatch has an operation-table address at its beginning. Operations are selected using byte offsets into that table.

| Receiver | Table byte offset | Explicit arguments visible at the call | Use of result |
|---|---:|---|---|
| Input object | 16 | None | Tested for zero; passed to F-b3bed2dfc5ec if nonzero |
| Acquired object | 16 | Acquired object address | Unsigned 32-bit comparison bound |
| Acquired object | 36 | Acquired object address, index | 32-bit value written to output storage |
| Acquired object | 20 | Acquired object address, index | Address used for the final 32-bit read |

These offsets identify dispatch positions, not semantic method names. The shared offset of 16 on two receivers does not establish that the selected operations have the same contract.

The initial invocation shows no explicit arguments. An implicit receiver, hidden arguments, or a particular calling convention must not be assumed.

The representation is consistent with a 32-bit address model, but exact pointer widths, callable-entry representation, alignment rules, and calling conventions remain unresolved. The dispatch offsets alone do not establish those properties.

### Preconditions for ordinary completion

Every reached memory access and call must be valid at its use point. In particular:

- The input object must permit reading its operation-table address and resolving the entry at byte offset 16.
- Reached dispatch entries must be callable under the applicable calling convention.
- A non-null acquired object must support each operation actually reached.
- The output address must permit a 32-bit write when the comparison accepts the index.
- The address returned by the final indexed operation must permit a 32-bit read.

The function does not validate these conditions. A non-null acquired address is not itself proof of a valid object or sufficient lifetime.

## Behavior

### Initial dispatch and gate

The function resolves and invokes the operation at byte offset 16 through the input object. Its result is tested for zero.

If the result is zero, the function returns zero. It does not invoke F-b3bed2dfc5ec, retrieve a bound, perform either indexed operation, or directly access the output storage.

Any nonzero result passes this gate. There is no positive-only comparison. In the represented 32-bit result, a nonzero pattern with the high bit set is not rejected merely because of that bit.

### Acquiring the intermediate object

After a nonzero initial result, the function invokes F-b3bed2dfc5ec with four argument expressions, in these positions:

1. The initial operation’s result.
2. A value read from one global location.
3. A value read from a distinct global location.
4. The fixed literal pattern `0xFFFFFFFF`.

The two global values are distinguished by their argument positions. Their domain meanings are unknown.

The supplied material contains neither declarations for those global locations nor a prototype for F-b3bed2dfc5ec. Consequently, it does **not** establish:

- Either global’s storage type.
- Either global read’s width.
- The widths of the corresponding formal parameters.
- Any conversion applied at the dependency interface.

The displayed fixed argument establishes the supplied literal pattern, not a domain-specific sentinel meaning or the dependency’s formal parameter width.

Both global argument values are obtained after the initial operation has completed and before F-b3bed2dfc5ec is invoked. The initial operation is not established to preserve either location. The relative order of the two global reads is not established.

If F-b3bed2dfc5ec returns null, the function returns zero without invoking operations on an acquired object and without directly writing to the output location.

### Bounds decision

For a non-null acquired object, the function invokes the operation at table byte offset 16, explicitly passing that object address.

The result is used as an unsigned 32-bit bound. The function accepts the index only when the unsigned 32-bit index is strictly less than that bound.

Established boundary behavior:

- A zero bound rejects every index.
- An index equal to the bound is rejected.
- An index above the bound is rejected.
- A bound with its high bit set is treated as a large unsigned value, not a negative value.
- The maximum unsigned 32-bit index, `0xFFFFFFFF`, is rejected for every possible 32-bit bound.

The function obtains the bound once. It does not visibly refresh or recheck it after subsequent calls.

No index increment, decrement, scaling, or other index arithmetic is visible. Both indexed operations receive the original 32-bit index unchanged. Any address calculation or overflow inside those operations is outside the established contract.

### First indexed result and output write

After the comparison accepts the index, the function invokes the acquired object’s operation at table byte offset 36 with the object address and index.

When that operation returns, the function writes its 32-bit result to the output address.

This is the only direct caller-storage write visible in the function. It occurs before resolution and invocation of the final indexed operation.

There is no visible rollback. If later processing fails, the output write may already be observable.

### Final indexed operation and return read

After the output write, the function resolves the acquired object’s operation at table byte offset 20 and invokes it with the same object address and index.

The result is treated as an address. The function reads one 32-bit value from that address and returns it.

There is no visible check that the address:

- Is non-null.
- Refers to accessible storage.
- Belongs to the acquired object.
- Corresponds to the previously obtained bound.
- Differs from the output location.
- Remains valid independently of dependency effects.

A successful bounds comparison therefore does not establish final-address safety.

### Observable ordering

On an ordinarily completing accepted path, the established sequence is:

1. The initial operation completes.
2. The two global argument values are obtained, without an established relative order.
3. F-b3bed2dfc5ec completes.
4. The bound operation completes.
5. The unsigned comparison accepts the index.
6. The operation at offset 36 completes.
7. The 32-bit output write occurs.
8. The operation at offset 20 is resolved and invoked.
9. The returned address is read for 32 bits.
10. That value is returned.

Dispatch information is not established as a single snapshot taken before all calls. Later dispatches use object and table state available when those dispatches are reached.

### Aliasing and mutation

The visible function imposes no non-aliasing restriction.

If output storage is the same location later returned by the final operation, the final read occurs after the output write and any intervening dependency effects. If nothing intervening changes that location, the return value equals the written value.

If the output write overlaps object metadata or dispatch-table storage, it can affect the subsequent dispatch or make it invalid. Likewise, a dependency can change object state, table state, global state, or memory consulted by a later operation.

The function supplies no visible mechanism that makes these operations immune to such changes.

## Side effects

### Direct effects

Depending on the path taken, the function directly performs:

- Object and dispatch-table reads needed for reached indirect calls.
- Reads from two distinct global locations before the call to F-b3bed2dfc5ec.
- One 32-bit write to caller-provided output storage.
- One 32-bit read through the final returned address.

The widths of the two global reads are not established. They must not be grouped with the established 32-bit output write or final read as though all accesses had the same width.

Rejected paths contain no direct output write. This is weaker than a guarantee that output storage remains unchanged: reached dependencies may independently modify aliased storage.

### Dependency effects

No reached dependency is established as pure, read-only, or preserving any global or object state. Returning a bound or value does not establish that the operation only reads memory.

No synchronization, transaction, rollback, or state-preservation mechanism is visible in this function.

### Ownership and lifetime

No explicit allocation, release, retain, or ownership transfer is visible in the function itself.

This does not establish whether any dependency allocates, releases, retains, transfers, or invalidates memory. Ownership responsibilities for the input object, acquired object, final returned address, and output storage remain unknown.

Ordinary completion requires that each used address remain valid until its last actual access. The function does not visibly acquire a lifetime guarantee or release one afterward.

### Errors and partial completion

The three gated rejection paths explicitly return zero. Other failures are not specified as graceful rejection.

Invalid dispatch data, inaccessible output storage, an invalid final address, dependency exceptions, and non-returning dependencies have no visible handling here.

If the output write succeeds but the final call or final read fails, the function may have changed caller storage without returning normally. No transactional guarantee is established.

## Dependencies

### F-b3bed2dfc5ec

This dependency is called once only when the initial indirect result is nonzero.

Its four supplied arguments are the initial result, values read from two distinct global locations in their second- and third-argument roles, and the fixed literal pattern `0xFFFFFFFF`. Its result is interpreted as an object address and checked for null.

The dependency’s prototype, formal parameter widths, internal behavior, error conventions, ownership rules, and state-preservation guarantees are not supplied. In particular, neither global value nor its corresponding formal parameter is established as 32-bit.

### Unresolved indirect dependencies

Four indirect call sites are visible: one through the input object and three through the acquired object.

Their runtime target identities are unknown; no additional opaque function IDs are available for them. Their dispatch locations, explicit argument roles, and result uses are specified above. Those observations do not establish their complete interfaces or side effects.

## Acceptance examples

These are conditional consequences of the static representation, not runtime test results. Unless an example explicitly concerns failure, it assumes valid reached accesses and dependencies that return normally.

### Initial result is zero

If the initial operation returns zero:

- F-b3bed2dfc5ec is not called.
- Neither global argument value is read by this function.
- No acquired-object operation is called.
- No direct output write occurs.
- The function returns zero.

### Initial operation changes a global

Suppose the initial operation returns nonzero and changes the global location used for the second dependency argument. If no intervening effect changes it again, F-b3bed2dfc5ec receives the value obtained from that location after the initial operation, not a value snapshotted before it.

This example does not assign a width or type to that value.

### Acquisition returns null

If the initial result is nonzero but F-b3bed2dfc5ec returns null:

- The two global values have been supplied in their respective argument positions.
- No bound operation or indexed operation occurs.
- No direct output write occurs.
- The return value is zero.

### Index equals the bound

With index 7 and retrieved bound 7, the comparison rejects the index. Neither indexed operation occurs, no direct output write occurs, and the function returns zero.

### Distinct output and return values

With bound 8 and index 7, suppose the operation at offset 36 returns `0x12345678`, and the address supplied by the final operation contains `0xAABBCCDD` when read.

The function writes `0x12345678` to output storage before invoking the final operation, then returns `0xAABBCCDD`.

### Accepted access returns zero

If the comparison accepts the index and the final address contains zero when read, the output write still occurs and the function returns zero. The return value alone does not distinguish this path from rejection.

### Unsigned boundary

With bound `0xFFFFFFFF` and index `0xFFFFFFFE`, the comparison accepts the index.

With both bound and index equal to `0xFFFFFFFF`, it rejects the index.

With bound zero, it rejects index zero and every other index.

### Output aliases the final read location

If the output address equals the final returned address, and the final operation does not change the stored value, the returned value equals the value written to output storage.

If that operation changes the location before returning, the function instead reads the value present after that change.

### Final address is null

If the final operation returns null, the output write has already occurred. The function attempts the final read without a visible null check. No normal return value is established for that invalid access.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of both objects, the index, bound, and successful-path values.
- Runtime identities and complete contracts of the indirect dependencies.
- Calling conventions, hidden arguments, and any implicit receiver for the initial call.
- Exact pointer ABI, alignment requirements, and byte order.
- Storage types and read widths of both global locations.
- Formal parameter types, widths, and conversions at F-b3bed2dfc5ec.
- Relative evaluation order of the two global reads.
- Global mutability, concurrency properties, and synchronization requirements.
- Ownership, retention, release, and lifetime responsibilities for all addresses.
- Whether dependencies preserve globals, object fields, dispatch tables, or output storage.
- Whether the previously retrieved bound remains meaningful after later mutations.
- Behavior under invalid memory, concurrent mutation, exceptions, or non-returning dependencies.
- Any caller-level means of distinguishing accepted zero results from rejection.

These unknowns are not resolved by assuming conventional container behavior or by inferring types from the global locations’ addresses.

## Evidence

The supplied static representation establishes the conditional acquisition call, four indirect call sites, dispatch offsets, unsigned strict bounds comparison, unchanged index forwarding, 32-bit output write, output-before-final-dispatch ordering, final 32-bit dereference, and zero-return rejection paths.

It also establishes two distinct global argument expressions. Without their declarations or the acquisition dependency’s prototype, it does not establish their storage widths, read widths, or corresponding formal parameter widths.

Indexed-access intent and a 32-bit address model are interpretations supported by the representation, not complete domain or ABI contracts.

No implementation execution, dependency inspection, or runtime testing has been performed. The acceptance examples express conditional static behavior, not observed executions.
