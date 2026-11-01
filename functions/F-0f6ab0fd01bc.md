<!-- Trusted provenance: F-0f6ab0fd01bc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0f6ab0fd01bc

## Purpose

F-0f6ab0fd01bc obtains an integer result through receiver-based dispatch, then obtains a source address through F-3b5d52fd8e4a. The saved dispatch result selects one of two operations:

- **Result equals 4:** read one 32-bit word from the returned source address and write its bits to the supplied destination.
- **Result differs from 4:** read an indirect callable from a global pointer slot and invoke that callable with the destination address followed by the returned source address.

The receiver-dispatched call precedes the source-provider call. On the non-4 path, the global target-slot read occurs **after F-3b5d52fd8e4a returns**. The wrapper uses the slot’s then-current value; neither preceding dependency is established to preserve it. The direct-transfer path does not perform this target-slot read.

This contract describes the supplied static representation. Only the result-4 branch visibly performs a copy. The non-4 dependency’s behavior is unresolved, so a general copy contract for the entire function would exceed the evidence.

## Interface

### Inputs

The function accepts three arguments:

1. **Receiver address.** Used to resolve the first indirect callable and passed as that callable’s sole explicit argument. The original receiver address is also passed to F-3b5d52fd8e4a.
2. **Opaque 32-bit argument.** Passed to F-3b5d52fd8e4a without a visible transformation. Its meaning is not established: it may represent a number, selector, handle, address representation, or other four-byte value.
3. **Destination address.** On the result-4 path, identifies the word to overwrite. On the non-4 path, becomes the first argument to the callable obtained from the global pointer slot.

### Receiver layout

The visible receiver access establishes this layout and traversal:

- Read the leading integer-represented field at the receiver address.
- Use that value as a base address.
- Obtain a callable from byte displacement 24 relative to that base.
- Invoke the callable with the receiver address.

This is an indirection layout, not a complete receiver type. A method-table interpretation is inferred intent, not an established semantic requirement.

The byte displacement 24 must not be treated as a numbered table slot without knowing the relevant entry width.

### Global target slot

A separate global pointer slot supplies the non-4 callable. It is not obtained from the receiver traversal.

The wrapper reads this slot only on the non-4 path, after the source-provider call has returned. The visible representation shows no earlier capture of its value. Consequently, the target used is the value present at that later read, including changes made by either preceding dependency.

### Widths and arithmetic

The second argument and the directly transferred datum are each four bytes, or **32 bits**.

The receiver-dispatched result is represented as an integer and compared for exact equality with 4. Complete ABI properties, including pointer width and the dispatched integer-return convention, are not independently established by the supplied representation.

The direct transfer performs no visible arithmetic, conversion, byte-order adjustment, or interpretation of the transferred word. It preserves the word’s bits. No arithmetic is applied to the opaque second argument.

The only visible address displacement is the receiver-dispatch lookup at byte offset 24. Valid execution requires that this lookup identify usable storage under the target environment’s address and alignment rules. No overflow or address-range check is shown.

### Output

There is no explicit return value.

If the saved result is 4 and execution completes normally, the destination contains the 32-bit pattern read from the source address after F-3b5d52fd8e4a returns.

If the saved result differs from 4, destination contents and other output effects depend on the callable read from the global slot. The wrapper shows no additional destination write after that invocation.

### Preconditions for normal execution

Common requirements are:

- The receiver supports the initial field read.
- The resulting dispatch base supports the callable lookup at byte offset 24.
- The resolved receiver-dispatched callable supports the observed invocation.
- F-3b5d52fd8e4a accepts the original receiver address and opaque second argument.
- Both preceding calls return normally for the branch-dependent operation to be reached.

For result 4:

- The returned source address supports a 32-bit read.
- The destination supports a 32-bit write.

For a result other than 4:

- The global target slot supports the later pointer read.
- Its then-current value identifies a usable callable.
- The destination and source arguments satisfy that callable’s unknown contract.

The wrapper performs no visible validation of these requirements. The global slot is not a target-resolution precondition on the direct-transfer path.

## Behavior

### Receiver dispatch

The wrapper first resolves the receiver-based callable through the leading field and byte-offset-24 lookup, then invokes it with the receiver address.

The returned integer is saved. This saved value, rather than a later observation of receiver state, controls the eventual branch.

The dispatched call may modify receiver memory, the global target slot, or other state. Its side effects are unknown. Because it precedes F-3b5d52fd8e4a, those effects may influence the source-provider call.

### Source acquisition

Next, the wrapper calls F-3b5d52fd8e4a with:

1. The original receiver address.
2. The original opaque 32-bit argument.

The result is treated as a source address.

This call occurs on both paths. It is not skipped when the saved result is 4, and the non-4 callable is not invoked before it returns.

The provider may modify the receiver, source storage, the global target slot, or other state. No preservation guarantee is established. The wrapper does not validate the returned address or establish its ownership.

### Result-4 path

When the saved result equals 4:

1. Read one 32-bit word from the returned source address.
2. Write those bits to the destination address.
3. Complete the wrapper.

The source read precedes the destination write. There is no visible decoding, arithmetic, or deep-copy operation.

This path does not read the global slot to resolve the non-4 target and does not invoke that target. This is a statement about the wrapper’s explicit target-resolution operation; it does not prohibit dependencies or aliased source storage from accessing the same physical memory.

### Non-4 path

When the saved result differs from 4:

1. Read the indirect callable from the global pointer slot, after F-3b5d52fd8e4a has returned.
2. Invoke the callable obtained by that read.
3. Supply the destination address as the first explicit argument.
4. Supply the returned source address as the second explicit argument.

No result from this invocation is visibly used. The supplied representation then shows completion of the wrapper, subject to the unresolved control-transfer details described below.

The target is not established to be stable across the two earlier calls. In particular:

- A value present at function entry is not necessarily the value invoked.
- A value installed by the receiver-dispatched call may be replaced by F-3b5d52fd8e4a before the slot is read.
- The wrapper does not visibly choose or cache this target before calling the source provider.

Every integer result unequal to 4 selects this same visible path. There is no separate handling for zero, negative results, or other positive results.

### Ordering and aliasing

For normally returning dependencies, the established order is:

1. Receiver-based lookup and invocation.
2. Saving the returned integer.
3. Source-provider invocation.
4. Branch selection using the saved integer.
5. Either the direct word transfer or the global target-slot read and indirect invocation.

Changing receiver state during the provider call does not trigger a fresh receiver dispatch. Changing the global target slot during that call can affect the non-4 target because its read occurs afterward.

No explicit non-aliasing restriction is shown between receiver storage, source, destination, and global storage.

On the direct path, identical source and destination addresses still express a read followed by a write. If the destination overlaps receiver-related storage, its write occurs after both preceding calls. Only one word transfer is established; this is not a general overlapping-region copy contract.

On the non-4 path, the indirect dependency determines any additional aliasing requirements.

These are source-level ordering claims, not guarantees about instruction scheduling, atomicity, synchronization, or concurrent modification.

## Side effects

### Visible wrapper operations

The wrapper visibly performs:

- Reads needed for receiver-based dispatch.
- One receiver-dispatched invocation.
- One invocation of F-3b5d52fd8e4a.
- On result 4, one 32-bit source read and one 32-bit destination write.
- Otherwise, a read of the global callable slot followed by an invocation through the value read.

There is no visible wrapper-owned allocation, deallocation, reference-count adjustment, explicit global assignment, logging, retry, or error-code construction.

These absences do not constrain dependency side effects. In particular, either preceding dependency may change the global slot even though the wrapper does not explicitly assign to it.

### Ownership and lifetime

The source address is used for the selected operation and is not visibly retained or released by the wrapper. This does not establish whether the address denotes borrowed storage, receiver-owned storage, or separately managed storage.

For the direct transfer, the source must remain readable through the word read. For the non-4 invocation, lifetime requirements depend on the unresolved target.

Copying a 32-bit representation does not itself establish resource duplication, ownership transfer, or reference retention. If the word represents a pointer or resource reference, no corresponding management operation is visible.

The indirect dependencies may allocate, retain, release, or transfer ownership. Their contracts are unknown.

### Errors and abnormal completion

No explicit checks are shown for invalid receiver storage, dispatch entries, source addresses, destinations, global-slot contents, or dependency-reported failures.

There is no visible fallback value, rollback, exception translation, or status return.

If either preceding call fails to return normally, later operations are not reached through the visible flow. If the non-4 slot read or indirect invocation cannot complete normally, the wrapper shows no recovery path.

Effects already performed are not visibly undone. Exact fault, exception, and nonlocal-control-transfer behavior remains environment-dependent.

## Dependencies

The indirect IDs below distinguish call roles; they do not assert recovered function identities.

| Opaque function ID | Provenance and role | Established interaction |
|---|---|---|
| F-indirect-A | Callable obtained through the receiver’s leading indirection at byte offset 24 | Invoked first with the receiver; its saved integer result selects the later path. |
| F-3b5d52fd8e4a | Source-address provider | Invoked after F-indirect-A with the original receiver and opaque 32-bit argument; returns the address used by the selected operation. |
| F-indirect-B | Callable read from a global pointer slot on the non-4 path | Resolved after F-3b5d52fd8e4a returns; invoked with destination first and returned source address second. |

F-indirect-B identifies a role whose actual target depends on the global slot’s value at the read. It does not identify one immutable function.

No dependency contract establishes global preservation, purity, ownership, bounds checking, or error recovery. The supplied representation also marks the non-4 indirect control transfer as unresolved and presents it as a call. A complete machine-level call-versus-tail-transfer and return-path contract is not established.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests.

### Direct transfer

Given valid inputs, a saved dispatch result of 4, and a provider returning a word containing hexadecimal 89ABCDEF:

- Receiver dispatch precedes the provider call.
- The provider receives the original receiver address and second argument.
- The destination receives the same 32-bit pattern.
- The wrapper performs no global target-slot read for F-indirect-B and does not invoke it.

### Ordinary non-4 result

Given a saved result of 3, a returned source address, and a usable callable in the global slot when it is read:

- The global-slot read follows the provider’s return.
- The callable read from that slot receives destination first and source second.
- No particular destination contents may be required without that callable’s contract.

### Target changed by the provider

Suppose the global slot initially identifies one callable, but F-3b5d52fd8e4a replaces it with another before returning.

For a saved result other than 4, the wrapper reads and invokes the replacement. Using a target cached before the provider call would not satisfy the established ordering.

### Both preceding calls change the slot

Suppose F-indirect-A installs one callable and the provider subsequently installs a different callable.

For a non-4 result, absent another intervening change, the provider’s replacement is the value read and invoked. The wrapper must not assume either dependency preserved the entry value.

### Unusable target on the direct path

Suppose the global slot contains an unusable callable value, but the saved result is 4 and all direct-path accesses are valid.

That unusable value does not itself prevent the wrapper’s direct transfer: the wrapper does not read or invoke it on this path. This example assumes the preceding dependencies do not themselves require that slot to be usable.

### Saved-result stability

Suppose F-indirect-A returns 4, then the provider changes receiver state so that a hypothetical later dispatch would return 3.

The wrapper still performs the direct transfer. No later dispatch is visible.

### Invalid source

For result 4 and an unusable source address, graceful failure is not specified. Acceptance must not require successful completion, a defined error result, rollback, or unchanged destination storage.

For a non-4 result, the wrapper passes the returned address onward without a visible word read; whether that address is acceptable depends on F-indirect-B.

## Uncertainties

### Inferred intent

The receiver traversal suggests dynamic dispatch. The saved result may distinguish representations or transfer modes. Neither interpretation is required by the visible behavior.

The constant 4 is established as a comparison value, not as a byte count, type identifier, or size declaration.

### Documented-only unknowns

The following remain **documented_only**:

- Complete target ABI, pointer widths, integer-return width, and alignment rules.
- Receiver type and dispatch-table meaning.
- Meaning of the opaque second argument.
- Source ownership, extent, and validity duration.
- Global-slot initialization, permitted writers, and synchronization.
- Actual identities and semantics of the indirect targets.
- Dependency side effects, including ownership and global-state changes.
- Complete machine-level semantics of the unresolved non-4 control transfer.
- Atomicity, thread safety, and behavior during concurrent mutation.

The global origin and post-provider ordering of the non-4 target read are established, not unknown. The target’s contents and dependency behavior remain unknown.

## Evidence

The supplied static representation establishes:

- Receiver-based callable lookup at byte offset 24.
- A saved integer result from that invocation.
- A subsequent call to F-3b5d52fd8e4a.
- An exact comparison of the saved result with 4.
- A 32-bit source read and destination write on the equal path.
- On the unequal path, a callable read from a global pointer slot after the provider returns, followed by an indirect invocation with destination and source.
- No corresponding target-slot read on the direct-transfer path.
- No visible local validation, cleanup, or explicit return value.

No runtime tests, dependency inspection, ABI verification, or implementation execution have been performed. The acceptance examples are conditional specifications, not observed results.
