<!-- Trusted provenance: F-aa5fdc3c4884; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aa5fdc3c4884

## Purpose

F-aa5fdc3c4884 conditionally replaces the backing storage of an object whose storage-related state occupies three consecutive 32-bit fields. The visible procedure uses an eight-byte element stride and requests additional storage when an unsigned query result is smaller than the requested count.

**Inferred intent:** this is a capacity-reservation operation for a contiguous sequence of eight-byte elements. The three fields appear to represent the beginning of storage, the end of live elements, and the end of allocated storage.

**Established behavior:** the function compares a dependency’s result with the request, chooses an allocation count using the request and another dependency’s result, requests storage, invokes an unidentified transfer operation, releases storage through another dependency, and updates the three fields. The meanings “size,” “capacity,” “allocation,” and “release” are supported by this usage but are not independently established contracts of the dependencies.

The function has no explicit return value. Its useful result is the changed object state, together with dependency side effects.

## Interface

### Inputs

1. **Object reference.** A reference to a writable object exposing at least three consecutive 32-bit fields:
   - Offset 0: storage-begin address.
   - Offset 4: live-end address.
   - Offset 8: storage-end address.

   These labels describe the apparent representation. Addresses are carried through 32-bit integer-valued fields in the supplied evidence.

2. **Requested count.** An unsigned 32-bit value, interpreted by the visible allocation arithmetic as a count of eight-byte units.

There is no explicit null check, object-layout validation, alignment check, or consistency check on the three fields.

### Output and ownership

There is no explicit status, count, or address returned to the caller.

On a normally completed growth path, the object contains:
- A beginning field equal to the address returned by the allocation dependency.
- A live-end field computed from that address and a later query result.
- A storage-end field computed from that address and the selected allocation count.

The apparent ownership transition is from old backing storage to replacement backing storage. However, the transfer and release dependencies’ ownership contracts are unavailable. This function alone does not establish whether elements are copied, moved, destroyed, retained elsewhere, or otherwise processed.

### Representation requirements

The visible calculations use a stride of eight bytes and 32-bit quantities. Successful ordinary operation requires a representation in which:
- The object’s address fields identify suitable storage.
- The traversal from the captured beginning toward the repeatedly read live end can terminate by increments of eight.
- The chosen count and resulting addresses are suitable for the allocation and transfer dependencies.
- Dependencies return normally and leave state compatible with subsequent reads, when normal completion is expected.

These are conditional requirements, not validations performed by this function.

## Behavior

### 1. Oversized-request handling

The request is compared, unsigned, against `0x1FFFFFFF`, which is 536,870,911 elements.

For a request greater than that threshold, the function first records the threshold in a temporary and calls F-5829ec5988c0 with four arguments:

1. The integer 8.
2. A reference to a fixed static datum.
3. The same reference again.
4. The original requested count.

The static datum’s contents and the meaning of the integer 8 are not provided.

The visible procedure does **not** explicitly return after this call. It also does not replace the retained request with the threshold. If the dependency returns normally, execution continues with the original requested count.

Consequently, this is not an established saturating or clamping interface. A nonreturning diagnostic, exception, or termination behavior would have to come from F-5829ec5988c0; none is demonstrated here.

### 2. Initial query and growth decision

Next, the function calls F-2b2ef7491758 with the object reference and treats its result as unsigned.

If that result is greater than or equal to the requested count, the function completes without entering the allocation path.

On this path, the function itself performs no explicit writes to the object’s three fields. Dependency side effects remain possible. For an oversized request, the earlier diagnostic call has already occurred before this comparison.

If the result is smaller than the request, the function continues with replacement storage.

### 3. Allocation-count selection

The growth path first calls F-3952b8ae1dfd with the object reference.

It passes that result, followed by the object reference, to F-a406c3310ff3. The latter’s 32-bit result is treated as unsigned.

The selected allocation count is the unsigned maximum of:
- The original requested count.
- The result of F-a406c3310ff3.

Equality selects the request, but this does not change the numerical result.

The visible function does not validate the selected count against `0x1FFFFFFF`. In particular:
- A growth-policy result may exceed that threshold even when the original request does not.
- If an oversized-request diagnostic returns, the original oversized request still participates in selection.
- The temporary threshold value written before that diagnostic is overwritten during the growth path; it does not cap the selected count.

### 4. Storage request

The function calls F-871e64ddc6de with:
- The selected count shifted left by three bits.
- A second argument of zero.

The shift operates on an unsigned 32-bit count. Its result is the low 32 bits of eight times that count.

For selected counts at or below `0x1FFFFFFF`, this calculation does not discard high bits. At the threshold, the byte count is `0xFFFFFFF8`.

Above the threshold, the calculation can wrap. For example, a selected count of `0x20000000` produces a byte-count argument of zero.

The returned 32-bit value is retained as the replacement-storage address. No explicit allocation-failure check appears before it is passed onward or used in address calculations.

### 5. Transfer invocation

After the storage request returns, the function invokes an unidentified transfer dependency with four arguments:

1. The object’s then-current beginning field.
2. The object’s then-current live-end field.
3. The returned replacement-storage address.
4. The object reference.

These object fields are read after allocation returns. They are not snapshots taken before the allocation call.

The available dependency map does not assign an opaque function ID to this transfer operation. Its semantics are therefore **documented_only**. In particular, this contract does not assume that it:
- Preserves the object fields.
- Preserves global state.
- Leaves the old elements untouched.
- Transfers exactly the apparent live range.
- Acquires or releases ownership.
- Cannot fail or terminate nonlocally.

There is no explicit function-local recovery sequence around this call.

### 6. Post-transfer traversal

After transfer returns, the function rereads the object’s beginning field and retains that value.

Starting from this captured address, it advances a traversal value by eight until it equals the object’s live-end field. The live-end field is consulted as the loop condition is evaluated; it is not separately captured as a fixed endpoint.

The loop body contains no visible element access, destructor call, storage write, or other dependency call.

This distinction is significant: the function does not visibly destroy each element in this traversal. Any element-level processing would have to be supplied by another dependency.

Termination depends on reaching the live-end value by repeated eight-byte increments. A representation with an incompatible endpoint is not explicitly rejected. Exact overflow behavior for the signed address-valued traversal is not established by the supplied representation.

### 7. Storage-release invocation

After the traversal finishes, the function calls F-ff1bdc376dcd with:

1. The beginning address captured after transfer.
2. The difference between the then-current storage-end field and that captured beginning address, shifted right by three.
3. Zero.

The second argument is calculated in eight-byte units rather than directly in bytes.

For an ordinary valid layout with a nonnegative, eight-byte-aligned storage extent, this is the apparent allocated element count. The source representation uses signed integer arithmetic for this subtraction and shift. Overflowing subtraction and negative extents do not have a fully established portable meaning here; they must not be described as a validated unsigned capacity calculation.

The released address is specifically the beginning field read **after** transfer. It need not equal the address that was present on entry if a dependency changed the object.

### 8. Final query and field writes

After the release call returns, F-3952b8ae1dfd is called again with the object reference.

This second result, not the earlier result used for growth selection, determines the replacement live-end address.

The function then performs three field writes in this order:

1. Live end becomes the replacement-storage address plus eight times the second query result.
2. Beginning becomes the replacement-storage address.
3. Storage end becomes the replacement-storage address plus eight times the selected allocation count.

There is no intervening dependency call between these writes. They are separate writes, not an atomic object replacement.

No explicit check ensures that the second query result is no greater than the selected allocation count. Thus, without stronger dependency contracts, the visible function does not independently guarantee that the final live end lies within the replacement storage extent.

The exact overflow semantics of the signed address and live-count arithmetic are unresolved. Ordinary in-range examples below assume representable results.

## Side effects

The established call order on a normally returning growth path is:

1. Optional oversized-request diagnostic.
2. Initial unsigned query.
3. First live-count-like query.
4. Growth-policy computation.
5. Storage request.
6. Unidentified transfer operation.
7. Empty traversal.
8. Storage-release operation.
9. Second live-count-like query.
10. Three object-field writes.

The no-growth path omits steps 3 through 10.

The function performs no explicit writes to element contents in the supplied body. It explicitly writes the three object fields only at the end of the growth path.

All dependency effects remain part of the observable operation. The evidence does not justify assuming that dependencies preserve globals or object fields. This matters particularly because the function reads object state after allocation, after transfer, and after release rather than relying entirely on entry-state snapshots.

No rollback, explicit cleanup handler, or error return is visible. If a dependency does not return normally, later steps are not established to occur. The supplied evidence is insufficient to assign strong exception safety, leak freedom on failure, or ownership restoration guarantees.

## Dependencies

| Opaque ID | Visible role and limits |
|---|---|
| F-5829ec5988c0 | Called for requests above `0x1FFFFFFF`; receives 8, the same static-data reference twice, and the original request. Diagnostic and control-flow semantics are unknown. |
| F-2b2ef7491758 | Receives the object; its unsigned result controls whether growth occurs. Interpreting it as current capacity is inferred intent. |
| F-3952b8ae1dfd | Receives the object twice on the growth path. The first result feeds growth policy; the second determines the final live end. Interpreting it as element count is inferred intent. |
| F-a406c3310ff3 | Receives the first query result and the object. Its unsigned result is combined with the request by maximum selection. |
| F-871e64ddc6de | Receives a 32-bit byte-count calculation and zero; returns the value used as replacement storage. Allocation semantics are inferred from usage. |
| F-ff1bdc376dcd | Receives the post-transfer beginning address, an extent expressed in eight-byte units, and zero. Release semantics are inferred from usage. |

One additional transfer dependency is present but has no supplied opaque ID. Its four-argument invocation is described above without assigning an invented identity.

## Acceptance examples

These examples are conditional specifications, not observed executions. Unless stated otherwise, dependencies return normally, do not introduce unrelated state changes, and all address arithmetic is representable.

### Request already covered

Given a request of 12 and an initial unsigned query result of 12 or greater:

- The initial query occurs.
- No growth-policy, allocation, transfer, or release call occurs.
- The function makes no explicit object-field writes.

### Growth selects the request

Given:
- Request: 12.
- Initial query result: 8.
- Growth-policy result: 10.
- Second live-count-like query result: 5.
- Replacement-storage address: \(B\).

The selected count is 12, and the storage-request byte argument is 96. After the intervening operations complete, the field writes occur in this order:

1. Live end: \(B + 40\).
2. Beginning: \(B\).
3. Storage end: \(B + 96\).

### Growth selects a larger policy result

With request 12, initial query result 8, and growth-policy result 20, the selected count is 20 and the storage-request byte argument is 160. The final storage end uses 20, not 12.

### Zero request

A request of zero never enters growth because an unsigned initial query result cannot be less than zero. The query still occurs.

### Oversized request whose diagnostic returns

For request `0x20000000`, the diagnostic call occurs before the initial query.

If the diagnostic returns, the initial query is smaller than the request, and the growth-policy result is no larger than the request, the selected count remains `0x20000000`. The storage-request byte argument is zero after the 32-bit left shift. The function does not visibly substitute the threshold or stop.

### State changed by transfer

If transfer changes the beginning and storage-end fields, the subsequent release call uses the beginning captured after transfer and the storage-end field read for that release calculation. Acceptance must not require release of the entry-time beginning address without an additional transfer-preservation contract.

### Query results differ

If the first live-count-like query returns 5 and the second returns 7, growth policy receives 5, but the final live end uses \(B + 56\). The first result is not reused for the final live-end calculation.

## Uncertainties

The following remain **documented_only**:

- The precise semantic identities of the three fields and query results.
- The transfer dependency’s opaque identity, behavior, and ownership effects.
- Whether the oversized-request dependency returns, throws, terminates, or changes state.
- Allocation failure signaling and zero-byte allocation behavior.
- Element construction, destruction, copying, or moving.
- Dependency effects on globals and the referenced object.
- Signed-overflow behavior, negative right shifts, and invalid-address handling.
- Thread safety, reentrancy, and external observation during the final separate writes.
- Exception unwinding or cleanup not visible in the supplied function body.

These unknowns do not justify silently imposing conventional container guarantees.

## Evidence

This contract derives from static inspection of the supplied function representation and the supplied opaque dependency mapping.

Directly established evidence includes the threshold comparison, unsigned growth decision, maximum-based count selection, eight-byte allocation calculation, dependency-call order, post-transfer rereads, empty traversal, release-argument calculation, second query, and final field-write order.

The contiguous-sequence interpretation and conventional allocation, transfer, and release terminology are inferred intent. Dependency internals and runtime outcomes are not established.

No runtime tests have been performed.
