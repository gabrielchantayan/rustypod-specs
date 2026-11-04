<!-- Trusted provenance: F-487d7eebdf4a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-487d7eebdf4a

## Purpose

F-487d7eebdf4a conditionally dispatches a record through an object-associated table of callable entries.

Two record kinds receive special handling: numeric values 19 and 20. For either kind, the function first invokes a kind-specific preliminary dependency. A nonzero result from that dependency is returned unchanged. A zero result permits further dispatch, using the record’s kind as observed after the preliminary dependency returns.

A fallback dependency is available for each supported kind. Calling either fallback causes this function to return 1, provided that the fallback returns normally. If neither fallback kind matches, the function returns 0.

This is a static behavioral contract. Descriptions such as “object,” “record,” “dispatch table,” and “preliminary” identify structural roles; they do not establish an application-level meaning, ownership model, or class hierarchy.

## Interface

### Inputs

The function receives two arguments:

- **Object address:** an address whose first machine-address field supplies the base address of a dispatch table.
- **Record address:** an address used to read a kind field and, when a fallback is selected, additional four-byte payload fields.

The supplied representation uses integer and address operations characteristic of a 32-bit interface. The kind, preliminary return value, function result, and payload words are treated here as 32-bit quantities. The exact calling convention, pointer representation, alignment requirements, and native source-language declarations are not independently established.

### Record layout

Offsets are byte offsets from the record address.

| Offset | Width | Observed role |
|---|---:|---|
| +0 | Not established | Not read directly by this function |
| +4 | 4 bytes | Kind discriminator, potentially read multiple times |
| +8 | 4 bytes | First fallback payload word |
| +12 | 4 bytes | Second fallback payload word |
| +16 | 4 bytes | Third payload word, used only by the kind-20 fallback |

The payload words are forwarded without visible arithmetic or conversion in this function. Their semantic types are unknown. A word may encode an integer, address, flags, or another representation; none is established by the supplied evidence.

The record’s total size is unknown. A path using the third payload word requires readable storage through byte offset +19. This does not establish that every record must have that extent: unsupported-kind paths do not directly access the payload.

### Object and dispatch-table layout

The object’s first address field identifies a dispatch table. Relevant table entries are addressed at these byte offsets:

| Table offset | Role | Arguments |
|---|---|---|
| +92, hexadecimal +0x5C | Kind-19 fallback | Object address, payload word at +8, payload word at +12 |
| +120, hexadecimal +0x78 | Kind-20 fallback | Object address, payload words at +8, +12, and +16 |
| +144, hexadecimal +0x90 | Kind-19 preliminary dependency | Object address, record address |
| +148, hexadecimal +0x94 | Kind-20 preliminary dependency | Object address, record address |

These are entry offsets, not table indices. The evidence does not independently establish the size or meaning of other entries.

### Result

The visible result is a 32-bit integer:

- A nonzero preliminary result is returned unchanged.
- A normally returning fallback is followed by a result of 1.
- A path selecting no fallback returns 0.

Consequently, the result is not restricted to Boolean values. Negative preliminary results, if produced, are also propagated unchanged.

### Preconditions

For a normally completing execution, every address actually accessed must be valid for the corresponding read, and every selected entry must be callable with the arguments supplied.

There is no visible null check, bounds check, table validation, or callback-signature validation. Record or object validity is path-dependent: an unsupported record can return 0 without directly reading the object or its table.

No ownership transfer is established by passing either argument.

## Behavior

### Initial kind classification

The function first reads the four-byte kind at record offset +4 and compares it with 19.

If that observation equals 19, the function obtains the object’s current table address, loads the callable entry at table offset +144, and invokes it with the object and record addresses.

If the first observation does not equal 19, the function reads the kind again and compares this new observation with 20. If this second observation equals 20, it obtains the object’s current table address, loads the callable entry at table offset +148, and invokes it with the same two addresses.

If neither comparison matches, no preliminary dependency is called.

The two initial comparisons are separate reads in the supplied representation. They must not be described as a single immutable snapshot. With a stable kind field, they simply distinguish 19, 20, and all other values. The behavior of externally concurrent modifications, including their language-level validity and visibility, is not established.

### Preliminary result handling

After a preliminary dependency returns:

- A nonzero result immediately becomes this function’s result.
- A zero result continues to fallback selection.

There is no visible normalization, masking, sign change, or range restriction applied to a nonzero result.

The preliminary dependency receives the original object and record addresses, not a payload copy. Its effects on reachable state are unknown. In particular, the contract does not assume that it preserves the record kind, payload fields, object’s table address, table entries, global state, or allocation lifetimes.

If the preliminary dependency does not return normally, no subsequent fallback or normal result is established.

### Fallback selection

Fallback selection occurs after either:

- A preliminary dependency returns zero; or
- Initial classification selects no preliminary dependency.

The function reads the kind again and first compares it with 20.

If this observation equals 20:

- It resolves the fallback entry through the object’s then-current table address, using table offset +120.
- It reads the four-byte payload words at record offsets +8, +12, and +16.
- It invokes the selected entry with the object address followed by those three words in that argument order.
- If the call returns normally, this function returns 1.

If that kind observation is not 20, the function reads the kind once more and compares this new observation with 19.

If this latter observation equals 19:

- It resolves the fallback entry through the object’s then-current table address, using table offset +92.
- It reads the four-byte payload words at record offsets +8 and +12.
- It invokes the selected entry with the object address followed by those two words in that argument order.
- If the call returns normally, this function returns 1.

If neither fallback comparison matches, the function returns 0.

The fallback’s own return value, if any, is not used. Returning 1 therefore establishes that a fallback returned normally, not that the fallback reported application-level success.

### Ordering and state sensitivity

The following ordering is established:

1. Initial kind classification precedes any preliminary call.
2. A selected preliminary call completes before its result is examined.
3. A zero preliminary result precedes fallback-kind reads.
4. A selected fallback call completes before this function returns 1.

Within a fallback call expression, the exact relative evaluation order of the table-entry lookup and individual payload reads is not established. The logical argument positions are established; the sequence of those reads is not.

The object’s table address is resolved again for fallback dispatch. It is not visibly retained from preliminary dispatch. A preliminary dependency that changes that address can therefore change which fallback entry is used.

Similarly, fallback selection uses fresh kind reads. A zero-returning preliminary dependency can change which fallback runs, or prevent a fallback altogether. A change from kind 19 to kind 20 can route execution through the kind-19 preliminary dependency followed by the kind-20 fallback.

At most one preliminary dependency and at most one fallback dependency are called by a single normally executing invocation.

## Side effects

No direct writes to the object, record, dispatch table, global storage, or caller-provided output storage are visible in the supplied function.

The function directly performs:

- Kind-field reads.
- Conditional object-header and dispatch-entry reads.
- Conditional payload reads.
- Up to two indirect calls.
- Production of an integer result.

All additional state changes are attributable to indirect dependencies or exceptional execution effects. Their behavior is unknown. They may mutate reachable storage, change global state, allocate or release memory, or fail to return; this function provides no visible containment or rollback.

No direct allocation, release, retention, reference-count adjustment, or cleanup is shown. Passing an address does not establish that a dependency borrows, retains, or consumes the referenced storage.

If a preliminary dependency invalidates storage needed by a subsequent fallback, there is no visible validity check before reuse. Safe normal completion requires that any subsequently accessed storage remain valid.

## Dependencies

The following opaque IDs are local labels for indirect-call roles in this contract, not recovered identities of concrete implementations. Different roles may resolve to the same implementation, and a role’s target may change with table state.

| Opaque dependency ID | Resolution | Observable contract used here |
|---|---|---|
| F-1f0d40a9c271 | Table offset +144 | Receives object and record addresses; supplies an integer result tested against zero |
| F-b3e78216d04a | Table offset +148 | Receives object and record addresses; supplies an integer result tested against zero |
| F-8ca41f72b903 | Table offset +92 | Receives object address and two four-byte payload values; result not consumed |
| F-6d209b83e15f | Table offset +120 | Receives object address and three four-byte payload values; result not consumed |

For every dependency, memory ownership, global-state preservation, error behavior, reentrancy, and lifetime guarantees remain **documented_only**.

No assumption that a dependency preserves state is necessary for the dispatch contract. Acceptance conditions that require stable state must state that condition explicitly.

## Acceptance examples

These are conditional static expectations, not executed tests. Each assumes valid accessed storage and normally returning dependencies unless stated otherwise.

### Stable kind 19, preliminary decline

Given a kind that remains 19 and a zero result from F-1f0d40a9c271:

- The preliminary role at table offset +144 is called with the object and record addresses.
- The kind-19 fallback is subsequently called with the object address and the payload words at +8 and +12.
- The function returns 1.
- The payload word at +16 is not directly read by this function.

### Stable kind 20, preliminary decline

Given a kind that remains 20 and a zero result from F-b3e78216d04a:

- The preliminary role at table offset +148 is called.
- The kind-20 fallback receives the payload words at +8, +12, and +16 in that argument order.
- The function returns 1.

### Nonzero preliminary result

Given either supported initial kind and a preliminary result of −7:

- The function returns −7 unchanged.
- No fallback is called.
- This function performs no fallback payload reads.

This does not constrain reads performed inside the preliminary dependency.

### Unsupported stable kind

Given a kind that remains 21:

- No dependency is called.
- The function returns 0.
- No object-header, dispatch-entry, or payload read is required by this function.

### Preliminary dependency changes kind

Given an initial kind of 19, and a preliminary dependency that sets the kind to 20 before returning zero:

- The kind-19 preliminary role is used.
- The kind-20 fallback role is then selected.
- The third payload word is required.
- Normal fallback completion produces 1.

If that dependency instead leaves the kind at an unsupported value, fallback selection returns 0 without calling a fallback.

### Preliminary dependency changes table or payload

Given a zero-returning preliminary dependency that replaces the object’s table address and changes payload words:

- Fallback dispatch uses the replacement table address.
- Payload arguments come from the fields read during fallback preparation, not from a copy captured before the preliminary call.

This expectation assumes the replacement table and updated record remain valid.

### Fallback reports failure through its own result

If a selected fallback returns normally with a result its implementation considers failure, this function still returns 1. No interpretation of that result is visible here.

## Uncertainties

The following remain **documented_only**:

- Exact target ABI, pointer width, byte order, alignment, and native declarations.
- Whether record kinds have application-specific names or meanings.
- Whether the object is a class instance or the table follows any standard object model.
- Concrete identities and full contracts of indirect dependencies.
- Signedness and interpretation of payload words.
- Exact ordering among reads used to prepare a fallback call.
- Validity and observability of concurrent modification.
- Fault, exception, cancellation, and nonlocal-exit behavior.
- Whether dependencies may release or replace input storage.
- Behavior after invalid memory access or an invalid indirect target.

“Inferred intent” is limited to a preliminary-handling stage followed by fallback dispatch. That interpretation does not strengthen the observable contract into a promise of successful processing.

## Evidence

The supplied static representation establishes comparisons against 19 and 20, four dispatch-table offsets, two preliminary argument layouts, two fallback argument layouts, repeated kind reads, propagation of nonzero preliminary results, and fixed returns after fallback selection.

Four-byte payload accesses support the stated payload widths. The 32-bit integer interpretation is consistent with the supplied representation; full ABI details are not independently verified.

No dependency bodies, runtime observations, or independent interface declarations were supplied. No runtime tests have been performed.
