<!-- Trusted provenance: F-e3a1bffa31a1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e3a1bffa31a1

## Purpose

F-e3a1bffa31a1 conditionally stores one 32-bit value into an indexed slot within caller-supplied storage. It accepts unsigned indices from 0 through 127, inclusive. For an accepted index, it writes the value at byte offset 12 within a slot whose stride is 20 bytes, then returns the 32-bit status value 0. For every other unsigned index, it returns the 32-bit status value 2 without accessing the supplied storage.

The established behavior is a bounded field update, not allocation, initialization of an entire slot, or management of the surrounding object. The meaning of the stored value and the broader purpose of the storage are unknown.

## Interface

The interface has three inputs and one result:

| Item | Established representation | Behavioral role |
|---|---|---|
| Storage base | Address-bearing input represented as a signed integer in the supplied evidence | Starting address used to locate the destination |
| Index | Unsigned integer | Selects a slot and determines whether a write occurs |
| Value | 32-bit quantity | Bit pattern written to the selected field |
| Result | 32-bit quantity | Status: 0 after the accepted-path store, or 2 for a rejected index |

The evidence describes an address-oriented interface, but does not independently establish a portable pointer type, calling convention, or address-space width. The signed integer representation of the base must not be interpreted as a requirement that the corresponding address be numerically positive.

The index comparison is unsigned. On a 32-bit unsigned interface, the accepted range is 0 through 127 and the rejected range is 128 through 4,294,967,295. A negative value supplied through another language’s interface is not a separate supported case: its behavior depends on conversion to the actual unsigned argument representation before this function is entered.

The value is treated as an uninterpreted 32-bit payload. There is no visible range check, normalization, conversion, or interpretation of that payload.

For a successful call, the caller must supply a base from which the selected four-byte destination can be legally addressed and written. The function does not establish that precondition itself.

## Behavior

### Range decision

The first relevant operation is the unsigned comparison of the index against 128.

If the index is 128 or greater:

- The function returns status 2.
- It does not read from or write to storage addressed through the base.
- It does not inspect the payload.
- The visible behavior does not depend on the validity of the supplied base.

The function does not clamp, wrap, or otherwise map a rejected index into the accepted range.

### Destination layout

For an accepted index, the destination starts at this byte offset from the supplied base:

**20 × index + 12**

The destination field is four bytes wide. Equivalently, the layout has a 20-byte stride, with the updated field occupying byte offsets 12 through 15 relative to each slot’s beginning.

This describes the addressing pattern, not a complete record definition. The evidence does not identify the contents of the preceding 12 bytes or the following four bytes of a slot. It also does not show that all 128 slots belong to one allocation.

For the accepted range:

| Index | Destination start offset | Destination byte range |
|---|---:|---:|
| 0 | 12 | 12–15 |
| 1 | 32 | 32–35 |
| 127 | 2,552 | 2,552–2,555 |

These offsets are relative to the supplied base, not absolute addresses.

Storage covering 2,556 bytes from the base is sufficient to include all possible destination ranges, provided its addressing and write permissions are otherwise valid. That size is not a demonstrated allocation requirement for every individual call: a call accesses only its selected field.

### Accepted update

When the index is below 128, the function performs one visible 32-bit store of the supplied payload to the calculated destination. The prior destination value is not read or used.

After the store completes normally, the function returns status 0.

The visible implementation does not:

- Initialize the rest of the slot.
- Clear adjacent fields.
- Update a slot count, validity flag, or ownership marker.
- Read back the written value.
- Compare the new value with the old value.
- Return the previous contents.

A payload equal to the existing contents still follows the accepted write path. There is no equality-based suppression of the store.

### Arithmetic boundaries

Within the accepted index range, multiplication by the stride yields values from 0 through 2,540. Adding the field offset yields values from 12 through 2,552. These offset calculations fit within 32-bit signed and unsigned arithmetic.

Adding that offset to the base presents a separate concern. The evidence does not establish address-overflow handling, a wrapping-address contract, or valid behavior when the resulting destination is outside accessible storage. Such inputs must not be treated as guaranteed status-2 failures: status 2 is associated only with index rejection.

## Side effects

The sole explicit externally visible mutation is the four-byte destination store on the accepted path. There are no explicit data reads from the target storage and no explicit writes to neighboring bytes.

The observable order established by the function is:

1. Decide whether the unsigned index is accepted.
2. On acceptance, write the payload to the selected field.
3. Return the corresponding status.

The rejected path omits the write entirely.

No allocation, release, callback, synchronization operation, logging operation, or explicit global-state update appears in the supplied evidence. Ownership of the supplied storage is not visibly transferred. The function neither acquires storage nor releases it.

Those observations do not establish broader environmental guarantees. In particular:

- A 32-bit store is not, by itself, evidence of atomicity.
- No inter-thread visibility or memory-ordering contract is shown.
- Alignment requirements are not specified.
- The behavior of writes to device-backed or otherwise special memory is unknown.
- Aliasing with other program state is not prohibited or analyzed by the function.

If the accepted destination is invalid, read-only, improperly aligned for the execution environment, or otherwise unsuitable, normal completion is not established. No explicit recovery path or alternate status for such a failure is visible.

## Dependencies

No invoked function dependencies are visible for F-e3a1bffa31a1. Its established behavior consists of an unsigned range decision, destination-address calculation, one conditional store, and status return.

Consequently, there is no callee ownership policy or callee global-preservation behavior to assume.

The function does rely on execution-environment facilities whose details are not provided: integer representation, address formation, writable memory, and the semantics of the resulting 32-bit store. These are environmental prerequisites, not identified opaque-function dependencies.

## Acceptance examples

The following are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

### Lowest accepted index

Given a valid writable destination at base plus 12, an index of 0 and a payload of hexadecimal `89ABCDEF` must cause that 32-bit payload to replace the field at offsets 12–15. The function must then return 0.

The byte ordering of that payload in memory depends on the target’s endianness and is not established here.

### Adjacent slot selection

Given valid storage for both relevant fields, index 1 must select offsets 32–35, rather than offsets 16–19 or 20–23.

An ordinary-memory observation must show no explicit modification by this function to the field at offsets 12–15.

### Highest accepted index

Given a valid writable destination at offsets 2,552–2,555, index 127 must write the payload there and return 0. This case is accepted; the boundary is not exclusive at 127.

### First rejected index

Index 128 must return 2 without accessing the supplied storage. Existing storage contents must remain unchanged by this function.

A valid base is not needed to satisfy the function’s visible rejected-path behavior, because that path does not dereference it.

### Large unsigned index

On a 32-bit unsigned interface, index 4,294,967,295 must return 2 without a destination access. It must not select an earlier slot through offset wraparound.

### Payload independence

For an accepted index and valid destination, payloads consisting entirely of zero bits or entirely of one bits must both be stored without payload-based rejection. Both calls return 0 after normal completion.

### Invalid accepted destination

An index of 0 with an invalid destination does not have a guaranteed normal return. Requiring status 2 in this case would add address validation absent from the evidence.

## Uncertainties

The following remain **documented_only**:

- The semantic identity of the surrounding storage and updated field.
- The intended interpretation of status values beyond the observed accepted/rejected distinction.
- The complete argument ABI, including independent confirmation of address width and index width.
- Target endianness and alignment requirements.
- Behavior when base-plus-offset arithmetic cannot form a valid destination.
- Fault handling for inaccessible, read-only, or unsuitable storage.
- Atomicity, concurrency safety, and inter-thread ordering.
- Whether callers allocate all 128 slots together or supply storage valid only for particular indices.

A plausible intent is to update a field in one entry of a fixed-capacity table. That interpretation is inferred from the bound and stride; it is not required to explain the established behavior.

## Evidence

This contract is based solely on the supplied static function body: an unsigned comparison against 128, a conditional four-byte store using a 20-byte stride and 12-byte field offset, and two explicit return values.

The field offsets and maximum accessed extent are arithmetic consequences of those constants. Ownership, synchronization, complete object layout, and runtime fault behavior are not established by that evidence.

No runtime tests have been performed.
