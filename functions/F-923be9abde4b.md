<!-- Trusted provenance: F-923be9abde4b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-923be9abde4b

## Purpose

F-923be9abde4b obtains a record through F-3877d9acff71 and conditionally copies two 32-bit fields from that record into caller-supplied destinations.

Its observable result has two branches:

- A nonzero result from F-3877d9acff71 selects the successful-record branch. The function performs whichever output transfers were requested and returns the 32-bit value zero.
- A zero result from F-3877d9acff71 selects the unavailable-record branch. The function performs no output-field transfers and returns a statically referenced 32-bit value whose numeric contents are not supplied.

The apparent intent is to expose two properties of a record obtained from global state. The meanings of the record, the properties, and the unavailable-record return value are unknown. In particular, zero is established as the return value for the successful-record branch, but the evidence does not establish a named error convention for the other branch.

## Interface

### Inputs

The supplied representation exposes two pointer-sized values in incoming stack locations rather than an explicit parameter declaration. This contract identifies them by their incoming order:

| Input | Role | Permitted null behavior |
|---|---|---|
| First destination | Receives the 32-bit field at record offset `0x98` | A null value suppresses this transfer |
| Second destination | Receives the 32-bit field at record offset `0x90` | A null value suppresses this transfer |

These names identify interface positions, not execution order. The second destination is processed before the first destination.

Each non-null destination must designate storage capable of accepting a 32-bit write if the successful-record branch is reached. The function contains no visible destination validation beyond testing for null.

Both destination values may be null. This does not bypass the global-state lookup or the call to F-3877d9acff71.

The exact external calling convention, register assignments, and any formal source-language prototype are not established by the supplied representation.

### Global input structure

The dependency argument is obtained through a global pointer chain. For this contract:

1. The **global anchor** is a referenced global location containing a base address.
2. The **context pointer** is the 32-bit value read at byte offset `0x30` from that base address.
3. The **lookup input** is the 32-bit value read at byte offset `0xF60` from the context pointer.

The lookup input is passed as the sole visible argument to F-3877d9acff71.

The offsets are byte displacements. The evidence supports 32-bit loads and a 32-bit pointer-like address model; it does not establish the complete layouts or allocation sizes of the enclosing objects.

### Return value

The function returns one 32-bit value:

- Zero after a nonzero dependency result and completion of all requested output transfers.
- The statically referenced unavailable-record value after a zero dependency result.

No numeric value, signedness-based interpretation, or symbolic meaning is established for the unavailable-record value. It must not be assumed to differ from zero.

## Behavior

### 1. Resolve the dependency argument

Before testing either output destination, the function resolves the global pointer chain:

1. Read the base address from the global anchor.
2. Read the context pointer at base offset `0x30`.
3. Read the lookup input at context offset `0xF60`.

These accesses are unconditional on both destination values.

No null check, range check, or validity check is visible for the base address or context pointer. For ordinary completion, the corresponding addresses must support the required reads. Null destinations do not make an invalid global chain safe.

The visible arithmetic consists of adding fixed byte offsets to address-like values. There is no visible multiplication, indexing by an input count, or explicit bounds calculation. Behavior at address-space boundaries, including wrapped address calculations, is not established independently of the target execution environment.

### 2. Invoke F-3877d9acff71

The function invokes F-3877d9acff71 once with the resolved 32-bit lookup input.

The result is tested only for equality to zero:

- Zero selects the unavailable-record branch.
- Any nonzero value selects the successful-record branch and is subsequently treated as a record address when an output transfer is requested.

There is no visible distinction between positive and negative nonzero values, no tag removal, and no record-address validation.

The global lookup precedes the call. The field reads described below follow the call. Consequently, the field contents used by this function are those available when those later reads occur, not necessarily contents that existed before the dependency ran.

### 3. Handle an unavailable record

If the dependency returns zero, the function returns the statically referenced unavailable-record value.

On this branch, the function itself:

- Does not read either record field.
- Does not write through either output destination.
- Does not initialize, clear, or otherwise normalize output storage.
- Does not make a second dependency call.

These are limits on this function’s direct operations. They do not establish that F-3877d9acff71 left caller-visible memory or globals unchanged.

### 4. Transfer the field at offset `0x90`

If the dependency returned nonzero, the function first considers the second destination.

If that destination is non-null:

1. Read the 32-bit field at byte offset `0x90` from the returned record address.
2. Write that 32-bit value through the second destination.

If the destination is null, neither this field read nor its corresponding destination write is performed.

The transfer preserves the 32-bit value without a visible conversion, mask, comparison, or arithmetic transformation. The field’s interpretation as signed, unsigned, floating-point, pointer-like, or otherwise encoded data is unknown.

### 5. Transfer the field at offset `0x98`

After the preceding transfer has completed or been skipped, the function considers the first destination.

If that destination is non-null:

1. Read the 32-bit field at byte offset `0x98` from the returned record address.
2. Write that 32-bit value through the first destination.

If the destination is null, neither this field read nor its corresponding destination write is performed.

The second field is not visibly prefetched before the first destination write. This ordering matters when a destination aliases record storage.

### 6. Return from the successful-record branch

After both conditional transfers, the function returns zero.

A nonzero dependency result therefore suffices for this branch even when both destinations are null. In that case, this function performs no record-field reads.

There is no visible rollback mechanism. If the first executed destination write succeeds and a later read or write cannot complete, the earlier write has already occurred in the shown operation order. The evidence does not define fault recovery or whether control returns normally after an invalid access.

## Side effects

### Direct memory effects

The visible body performs:

- Reads through the global anchor chain.
- One dependency call.
- Up to two 32-bit record-field reads.
- Up to two 32-bit destination writes.

It contains no explicit allocation, release, record update, global assignment, lock operation, or retry.

This absence applies only to operations visible in this function. F-3877d9acff71 may have additional effects that are not supplied.

### Aliasing and write order

No destination-disjointness check is visible.

If both destinations identify the same ordinary writable 32-bit location, the field at offset `0x90` is written there first, followed by the field at offset `0x98`. Assuming the intervening write does not affect the later source read, the final stored value is the latter field.

A destination may also overlap a record field. In particular, if the second destination identifies the record’s field at offset `0x98`, the first transfer overwrites that field before the later field read. The first destination then receives the overwritten value.

A behavioral implementation must therefore preserve the visible read/write ordering rather than unconditionally snapshotting both source fields before writing either destination.

Partially overlapping destinations, unusual alignment, and device-backed memory require target-specific semantics not established here.

### Ownership and lifetime

The function does not visibly acquire, transfer, retain, or release ownership of:

- The global objects.
- The dependency result.
- Either destination.

It copies field values; it does not visibly manage resources that those values might represent.

For ordinary completion, the global chain must remain readable during argument resolution. Any requested record field must remain readable until its read, and each requested destination must remain writable until its write. No internal lifetime protection or synchronization is shown.

No preservation, ownership, or release guarantee is inferred for F-3877d9acff71.

## Dependencies

### F-3877d9acff71

**Visible interface:** Receives one 32-bit value obtained through the global chain and produces a result tested for zero.

**Use of result:** Zero selects the unavailable-record return. Nonzero is treated as a record address for requested field reads.

**Established invocation count:** One invocation on each ordinary execution reaching the call. Destination nullness does not suppress it.

**Unknown behavior:** Its lookup rules, errors, memory ownership, global mutations, synchronization, callbacks, and lifetime guarantees are not supplied.

The calling function does not visibly retry, translate multiple dependency error categories, or release the returned record.

### Data dependencies

The function also depends on:

- A readable global anchor and two-level offset-based lookup.
- Readable record fields at offsets `0x90` and `0x98` when their respective transfers are requested.
- A statically referenced 32-bit unavailable-record value.

The numeric contents of the last dependency are not supplied.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless stated otherwise, assume ordinary readable and writable memory, stable field values, and normal return from the dependency.

1. **Two distinct outputs.**  
   If the dependency returns a valid record whose fields at `0x90` and `0x98` contain `0x11223344` and `0xA1B2C3D4`, respectively, the second destination receives `0x11223344` before the first destination receives `0xA1B2C3D4`. The return value is zero.

2. **Only the first destination is supplied.**  
   With a nonzero dependency result, a null second destination, and a valid first destination, only the field at `0x98` is read and transferred. The field at `0x90` is not read by this function.

3. **Only the second destination is supplied.**  
   With a nonzero dependency result, a valid second destination, and a null first destination, only the field at `0x90` is read and transferred.

4. **Neither destination is supplied.**  
   The global lookup and dependency call still occur. If the dependency returns nonzero, neither record field is read and the function returns zero.

5. **No record is returned.**  
   If the dependency returns zero, neither destination is written by this function. The returned bits equal the unavailable-record value, whose numeric contents remain unspecified.

6. **Both destinations identify the same location.**  
   For stable, non-overlapping source fields, the shared destination receives the `0x90` field first and the `0x98` field second. Its final value is the latter.

7. **The earlier write changes the later source.**  
   If the second destination points exactly to the record field at `0x98`, the transfer from `0x90` overwrites that field. A subsequent transfer to a distinct first destination reads the updated field, not a saved original value.

8. **Global lookup is invalid despite null destinations.**  
   Both destinations being null does not authorize bypassing the global reads. The evidence provides no defined fallback return for an invalid global chain.

## Uncertainties

The following are **documented_only**, not inferred guarantees:

- Exact calling convention and formal parameter declaration.
- Semantic identities of the global objects, lookup input, record, and copied fields.
- Numeric contents and error meaning of the unavailable-record value.
- Dependency ownership, lifetime, synchronization, and mutation behavior.
- Target endianness and precise alignment requirements.
- Fault, signal, exception, or recovery behavior for invalid accesses.
- Address-overflow behavior beyond the visible fixed-offset calculations.
- Concurrency guarantees, memory barriers, and atomicity of the 32-bit accesses.
- Whether unusual overlapping or device-memory accesses are supported.

The description of the dependency as obtaining a record is inferred intent. The established behavior is narrower: its zero result selects one return path, while its nonzero result supplies the base address for conditional field reads.

## Evidence

This contract is based solely on the supplied static representation of F-923be9abde4b.

Established evidence includes the global pointer-chain offsets, one visible dependency call, a zero/nonzero result test, conditional 32-bit transfers from offsets `0x90` and `0x98`, their ordering, and the two return-value sources.

No implementation of F-3877d9acff71, complete object layouts, numeric unavailable-record value, or external interface declaration was supplied. No runtime tests have been performed.
