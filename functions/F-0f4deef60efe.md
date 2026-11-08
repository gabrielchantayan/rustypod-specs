<!-- Trusted provenance: F-0f4deef60efe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0f4deef60efe

## Purpose

F-0f4deef60efe conditionally transfers a source object’s element range into a destination object and returns the destination object’s address. Each object exposes three 32-bit fields representing a storage start, an active-range end, and a storage boundary. Element-related address calculations use a stride of 16 bytes.

The established behavior has three non-self branches:

- Allocate replacement storage when the destination’s recorded capacity is below a source-size query.
- Reuse storage and extend the active range when a destination-size query is below a source-size query.
- Reuse storage and process a trailing destination range otherwise.

The last branch explicitly sets both the destination’s active end and its storage boundary to the same computed address. That write is part of the contract; it must not be replaced with an assumed conventional container-assignment rule.

Interpreting this operation as assignment of a dynamically allocated sequence is **inferred intent**. The actual element operations, allocation semantics, and range-processing semantics depend on opaque functions whose implementations are not supplied.

## Interface

### Inputs and result

The function receives four arguments:

| Argument | Established representation | Role |
|---|---|---|
| Destination | Pointer to an object containing at least three consecutive 32-bit fields | Object whose metadata may be changed |
| Source | Pointer to an object containing at least three consecutive 32-bit fields | Object consulted for source bounds and size queries |
| Additional scalar A | Unsigned 32-bit value | Copied to temporary storage on non-self calls, but overwritten before it could affect an observed decision |
| Additional scalar B | Unsigned 32-bit value | Copied to temporary storage on non-self calls, but overwritten before it could affect an observed decision |

The result is the destination pointer, unchanged as a pointer value, on every shown normal-return path. This is not an independently allocated result object.

The additional scalar arguments have no established effect on the shown branch decisions, dependency arguments, metadata writes, or returned value.

### Object layout

The visible layout consists of:

| Byte offset | Field name used in this contract | Established use |
|---|---|---|
| 0 | Start | Base address for element operations and address calculations |
| 4 | Active end | Upper bound passed to range-processing dependencies |
| 8 | Storage boundary | Used with Start to derive a capacity-like quantity |

These fields have 32-bit representations in the supplied evidence. The supplied declarations treat their contents as signed integers in several expressions, even though they are also used as addresses.

A 16-byte stride is established by both the four-bit right shift used for capacity calculations and multiplication or left shift used for byte extents. The internal contents and alignment requirements of an individual element are unknown.

### Preconditions and validity limits

Except for the self-identity path, valid operation requires accessible object metadata and ranges acceptable to the invoked dependencies. The function performs no explicit checks for:

- Null object pointers.
- Invalid, reversed, or misaligned ranges.
- Arithmetic overflow.
- Allocation failure.
- Overlapping element storage.
- Dependency failure.

A well-formed sequence interpretation would normally require ordered bounds and a byte extent divisible by 16. Those are **inferred preconditions for meaningful container behavior**, not validations established here.

If the two object pointers compare equal, the function returns immediately without reading their metadata or calling any dependency. The shown path does not require dereferencing the shared pointer.

## Behavior

### Arithmetic and observation rules

Size queries are made through F-f40dec111e7a. Query results are not cached globally: the function may query the same object repeatedly, and different query results can affect different parts of one operation.

The initial capacity comparison uses the destination’s Storage boundary minus Start, shifted right by four, and then interpreted as an unsigned 32-bit value. The subtraction and shift are shown with signed operands before conversion. For malformed bounds or overflow cases, the supplied representation does not establish a portable mathematical capacity calculation.

Growth selection uses unsigned 32-bit arithmetic. If the queried destination size is \(n\), the two candidate capacities are:

- \(n + 32\).
- \(n + \lfloor n/2 \rfloor + \lfloor n/8 \rfloor\).

Each addition is a 32-bit unsigned addition, not an unbounded integer operation. The selected capacity is the unsigned maximum of those two computed candidates and a separately queried source size. Thus overflow can make the selected value smaller than the corresponding mathematical growth expression.

The requested allocation byte count is the selected capacity shifted left four bits in a 32-bit unsigned value. High bits can be discarded. The function does not verify that the byte count represents the selected capacity without overflow.

Other address calculations use 32-bit field values and multiplication by 16. The evidence does not justify extending these calculations to wider arithmetic or defining safe behavior for every signed-overflow case.

Dependency calls are sequencing boundaries. Unless stated otherwise below, fields used after a call are read from the objects at that later point; the function does not preserve an initial metadata snapshot. Argument expressions within a single call have no established left-to-right evaluation order.

### 1. Self-identity

The destination and source object pointers are compared first.

If they are equal:

- No metadata fields are read or written by the function.
- No dependency is called.
- The destination pointer is returned.

This check concerns object-pointer identity only. Distinct objects sharing some or all backing storage do not take this path.

### 2. Initial capacity decision

For distinct object pointers, the function first queries the source through F-f40dec111e7a.

It then compares that result against the unsigned capacity-like quantity derived from the destination’s current Start and Storage boundary.

- If the capacity-like quantity is smaller, execution takes the replacement-storage branch.
- Otherwise, execution takes a reuse-storage branch.

The initial source-size result establishes this comparison only. Subsequent branches perform additional queries rather than relying on it throughout.

### 3. Replacement-storage branch

The following operations occur in order:

1. Query the source again.
2. Query the destination.
3. Compute the unsigned growth candidates and selected capacity described above.
4. Call F-871e64ddc6de with the selected capacity shifted left four bits and a second argument of zero.
5. Pass the source’s then-current Start and Active end, the allocation result, and the destination object pointer to F-3789da823c71.
6. Pass the destination object pointer and its then-current Start and Active end to F-660635c2f259.
7. Pass the destination’s then-current Start, its then-current Storage-boundary-minus-Start extent shifted right four bits, and zero to F-ff1bdc376dcd.
8. Write the allocation result into destination Start.
9. Query the source again.
10. Write destination Active end as the then-current destination Start plus 16 times that final source-size result.
11. Write destination Storage boundary as the then-current destination Start plus 16 times the selected capacity.
12. Return the destination pointer.

The range-transfer call precedes the call that processes the destination’s existing active range. The release-like call follows both. The destination’s Start is not explicitly replaced until after these three calls.

There is no explicit check of the allocation result before it is passed to F-3789da823c71. There is also no explicit rollback path if an intervening dependency does not complete normally.

The final active extent uses the last source-size query, whereas allocation sizing uses an earlier source-size query. If dependencies change the source or its reported size, the two values need not agree.

### 4. Reuse-storage decision

When the initial capacity comparison permits reuse, the function queries:

1. The destination.
2. The source.

It compares these two results as unsigned 32-bit values.

If the destination result is smaller, it takes the extending-reuse branch. Otherwise, it takes the non-extending-reuse branch.

The function does not repeat the initial capacity check after these queries.

### 5. Extending reuse

This branch performs the following sequence:

1. Query the destination again to obtain a prefix count.
2. Call F-3f0b6cad06f7 with:
   - The source’s current Start.
   - That Start plus 16 times the prefix count.
   - The destination’s current Start.
3. Query the source again.
4. Write destination Active end as the destination’s then-current Start plus 16 times that source-size result.
5. Call F-3789da823c71 with:
   - The source’s then-current Start plus 16 times the saved prefix count.
   - The source’s then-current Active end.
   - The destination’s then-current Start plus 16 times the saved prefix count.
   - The destination object pointer.
6. Return the destination pointer immediately.

The return value from F-3f0b6cad06f7 is ignored in this branch.

The Active end write occurs **before** the trailing F-3789da823c71 call. This ordering is observable to that dependency and matters if it fails or inspects the destination metadata.

There is no explicit write to destination Storage boundary on this branch. That does not establish that the field remains unchanged: a dependency may modify it.

### 6. Non-extending reuse

This branch performs the following sequence:

1. Call F-3f0b6cad06f7 with the source’s current Start, the source’s current Active end, and the destination’s current Start.
2. Pass that call’s returned 32-bit value as the beginning bound to F-660635c2f259, together with the destination object pointer and its then-current Active end.
3. Query the source again.
4. Compute a new end address from the destination’s then-current Start plus 16 times the returned source size.
5. Write that address to destination Active end.
6. Write the same address to destination Storage boundary.
7. Return the destination pointer.

The returned value from F-3f0b6cad06f7 is operationally significant here, unlike in extending reuse.

This branch invokes neither the allocation dependency nor the release-like dependency directly. Nevertheless, it explicitly replaces the recorded Storage boundary with the new Active end, even when the original storage boundary was farther away.

## Side effects

### Explicit writes

The function itself writes only destination metadata in the shown normal execution:

- Replacement storage: Start, then Active end, then Storage boundary.
- Extending reuse: Active end only.
- Non-extending reuse: Active end, then Storage boundary.
- Self-identity: no writes.

No direct source metadata write is shown.

### Dependency-mediated effects

Dependencies receive source bounds, destination addresses, and, in some calls, the destination object pointer. They may therefore affect element storage and potentially metadata or other state.

Their ownership behavior is not established. In particular:

- The allocation-like result is subsequently used as a storage address and installed as destination Start, but its ownership and failure representation are unknown.
- F-ff1bdc376dcd receives the old storage-like address and a count-like extent, but actual deallocation semantics are not proven.
- F-660635c2f259 receives a range and destination context, but destruction or cleanup semantics are inferred rather than established.
- No claim is made that any dependency preserves globals, source metadata, destination metadata, or unrelated memory.

There are no visible error codes or explicit error-result branches. Exceptional control flow, traps, nonlocal exits, and dependency failure effects are **documented_only**.

## Dependencies

| Opaque function ID | Established use | Unestablished interpretation |
|---|---|---|
| F-f40dec111e7a | Receives an object pointer; returns a 32-bit value used for counts, comparisons, or offsets | Likely a size query; purity and stability are unknown |
| F-871e64ddc6de | Receives a 32-bit byte-count expression and zero; returns a value used as an address | Likely allocation; alignment, ownership, and failure semantics are unknown |
| F-3789da823c71 | Receives source beginning, source end, destination beginning, and destination context | Likely transfers or constructs a range; overlap and failure behavior are unknown |
| F-3f0b6cad06f7 | Receives source beginning, source end, and destination beginning; its result is sometimes used as a range boundary | Likely transfers into existing elements; returned-boundary semantics are unknown |
| F-660635c2f259 | Receives destination context and two range boundaries | Likely processes or destroys a range; exact effects are unknown |
| F-ff1bdc376dcd | Receives a storage-like address, a shifted extent, and zero | Likely releases storage; ownership requirements and count units are unknown |

There is no recursive call to F-0f4deef60efe in the supplied body.

## Acceptance examples

These are conditional contract examples, not runtime results. Unless otherwise specified, they assume stable size queries, no dependency-induced metadata changes, valid non-overlapping storage, normal dependency returns, and arithmetic without overflow.

### Identical object pointers

Given equal source and destination object pointers, acceptance requires returning that pointer without dependency calls or metadata access.

The contents of the three metadata fields must not affect this path.

### Replacement driven by source size

Given destination size 8, destination capacity 10, and source size 12:

- Replacement storage is selected.
- The growth candidates are 40 and 13.
- The selected capacity is 40.
- The allocation request is 640 bytes with a second argument of zero.
- Transfer precedes existing-range processing, which precedes the release-like call.
- On normal completion, the recorded active extent is 192 bytes and the recorded storage extent is 640 bytes from the replacement Start.

### Source dominates the growth candidates

Given destination size 8, destination capacity 10, and source size 100, the selected capacity is 100 and the allocation request is 1,600 bytes. The growth heuristic does not impose a lower capacity than the source-size candidate.

### Extending within existing capacity

Given destination size 3, destination capacity 8, and source size 5:

- F-3f0b6cad06f7 receives the first three source elements and destination Start.
- Destination Active end is written to Start plus 80 bytes.
- F-3789da823c71 then receives the remaining source range and destination Start plus 48 bytes.
- There is no direct Storage boundary write and no direct allocation or release-like call.

### Shortening within existing storage

Given destination size 5, destination capacity 8, and source size 3:

- F-3f0b6cad06f7 receives the complete source range.
- Its returned value becomes the beginning bound passed to F-660635c2f259.
- Both Active end and Storage boundary are finally written to destination Start plus 48 bytes.

Preserving the original eight-element recorded capacity would not match this branch.

### Equal sizes

Given destination size 4, capacity 8, and source size 4, non-extending reuse is selected. Both recorded end fields become Start plus 64 bytes. The range-processing dependency is still called; the function does not suppress that call merely because the sizes compare equal.

### Empty source

Given a stable source size of zero and a valid destination, non-extending reuse is selected. After its dependency calls, both destination end fields become destination Start. Empty-range behavior inside the dependencies remains unknown.

### Changing query results

If a dependency changes the source before a later size query, acceptance must follow the later returned value where the behavior specifies a fresh query. An implementation that unconditionally snapshots one source size for the entire operation would not preserve the established call and observation structure.

## Uncertainties

The following remain **documented_only**:

- Exact element type, payload layout, alignment, and lifecycle.
- Whether F-f40dec111e7a is a pure size accessor.
- Whether any dependency mutates globals or either object.
- Allocation failure representation and handling.
- Ownership transfer and actual release behavior.
- Partial-transfer, exception, and rollback guarantees.
- Safety for overlapping storage held by distinct object records.
- Defined behavior for invalid bounds, negative shifted extents, or overflowing signed address calculations.
- Whether setting Storage boundary equal to Active end in non-extending reuse is intentional policy or an artifact of the supplied representation.

The observable boundary write is established despite uncertainty about its intent.

## Evidence

This contract is based solely on static inspection of the supplied function representation and the provided opaque dependency mapping. No dependency implementation, runtime trace, or external behavioral document was supplied. No runtime tests have been performed.

Established evidence includes the identity guard, three-field metadata layout, 16-byte stride, unsigned branch comparisons and growth selection, dependency call ordering, repeated queries, explicit destination writes, early return in extending reuse, and destination-pointer return.

Descriptions involving assignment, allocation, construction, destruction, capacity, or release are interpretive labels. They do not establish undocumented guarantees of the opaque dependencies.
