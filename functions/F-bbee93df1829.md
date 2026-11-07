<!-- Trusted provenance: F-bbee93df1829; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bbee93df1829

## Purpose

F-bbee93df1829 obtains a byte sequence through one of two dependencies and, only when the dependency returns zero, copies that sequence into a caller-supplied destination.

A control word in the supplied context selects between:

- A direct-byte path that passes a context-held value to F-8364fe87b70d and clears that context field following a zero result.
- A transformed-byte path that passes the context itself to F-0b6f5f32b71e and applies a fixed per-byte transformation to successful output.

The function returns the selected dependency’s status without translating it. A null destination instead causes an external 32-bit value to be returned, without examining the context or calling either dependency.

These are static observations. The semantic meaning of the context, the produced bytes, and the returned statuses is not established.

## Interface

### Arguments

**Context address**

The first argument is represented as a 32-bit integer used as an address. For a non-null destination, it must identify storage supporting the field accesses described below.

| Byte offset from context base | Access width | Established use |
|---|---:|---|
| 92, or 0x5c | 32 bits | Input value for F-8364fe87b70d; overwritten with zero after that dependency returns zero |
| 96, or 0x60 | 32 bits | Unsigned byte count, read after the selected dependency returns zero |
| 100, or 0x64 | 32 bits | Control word used to choose the dependency |

This is an accessed-field layout, not a complete context definition. The evidence does not establish the meanings of other bytes, the context’s total allocation size, or additional layout requirements imposed by F-0b6f5f32b71e.

**Destination address**

The second argument is either null or a pointer to writable byte storage.

For a successful non-null operation, the destination must accommodate the byte count read from context offset 96. There is no destination-capacity argument and no capacity check.

A null destination selects a distinct return-only path. It is not passed to either dependency.

### Return value

The result is an unsigned 32-bit value:

- With a null destination, it is the value read from an external 32-bit datum.
- With a non-null destination and a dependency that returns normally, it is exactly that dependency’s returned value.

Zero is the only dependency result that enables copying. Every nonzero value suppresses this function’s copy operation. This zero/nonzero distinction is established by the control flow; descriptions such as “success” and “failure” do not establish a broader status-code convention.

### Representation and preconditions

The supplied representation uses 32-bit status values, field accesses, and counts, together with byte-addressed character buffers. The byte-transformation contract below uses eight-bit bytes. Endianness is not established and must not be inferred from the field offsets.

For defined, bounded execution on the non-null path:

- Required context accesses must be valid.
- The selected dependency’s own preconditions must hold.
- The dependency must not overrun the temporary region it receives.
- Every temporary byte subsequently read by this function must be available and initialized.
- Every destination byte subsequently written must be writable.

No validation enforces these conditions.

## Behavior

### Null-destination path

When the destination is null, the function reads the external 32-bit datum and returns it.

On this path, it does not:

- Read any context field.
- Invoke either dependency.
- Allocate a result for the caller.
- Copy bytes.
- Clear the context field at offset 92.

Consequently, this function’s own accesses do not require a valid context address on the null-destination path. The external datum’s meaning, mutability, and initialization requirements remain unknown.

### Dependency selection

For a non-null destination, selection proceeds through two short-circuited control-word reads:

1. Read the 32-bit control word at context offset 100 and test whether its low eight bits equal 2.
2. Only if that test passes, read the control word again and test whether bit 0x200 is set.

The direct-byte path is selected only when both tests pass. Otherwise, the transformed-byte path is selected.

The two tests are represented by separate reads; the contract does not replace them with a single snapshot. If the underlying storage can change between reads, the tests can observe different values. The source does not establish synchronization or permit a claim of thread safety.

Bits outside the low-byte equality test and the 0x200 mask do not otherwise participate in this function’s selection logic.

### Direct-byte path

This path uses a temporary region beginning one byte into a 20-byte local array. The remaining array-backed capacity from that passed address is therefore 19 bytes.

The observable sequence is:

1. Read the 32-bit value at context offset 92.
2. Call F-8364fe87b70d with that value as its first argument and the temporary-region address as its second argument.
3. Retain the dependency’s unsigned 32-bit result.
4. If the result is nonzero, skip the field clear, count read, and copy.
5. If the result is zero, write a 32-bit zero to context offset 92.
6. After that write, read the unsigned 32-bit byte count from context offset 96.
7. Copy that many bytes from the temporary region to the destination, using the direction rule below.
8. Return the retained dependency result.

The field clear precedes both the count read and all destination writes. It occurs even when the count is zero.

The dependency receives the value loaded before the call. The subsequent clear does not establish that the value denotes an owned resource, that the dependency released it, or that clearing it transfers ownership.

### Transformed-byte path

This path uses a temporary region beginning one byte into a 21-byte local array. The remaining array-backed capacity from that address is 20 bytes.

The observable sequence is:

1. Call F-0b6f5f32b71e with the temporary-region address as its first argument and the context address as its second argument.
2. Retain the dependency’s unsigned 32-bit result.
3. If the result is nonzero, skip the count read and copy.
4. If the result is zero, read the unsigned 32-bit byte count from context offset 96.
5. Transform and copy that many bytes, using the direction rule below.
6. Return the retained dependency result.

For each source byte with numeric value \(b\), the stored destination byte is:

\[
(36 - 75b)\bmod 256.
\]

Signed versus unsigned interpretation of an eight-bit source character does not change this final eight-bit result: interpretations of the same byte differ by 256, and the corresponding products differ by a multiple of 256. The intermediate arithmetic remains within a signed 32-bit range for either eight-bit interpretation.

This path contains no explicit write to context offset 92. That does not imply that the dependency, or an aliased destination, leaves the field unchanged.

### Copy direction and extent

In either successful path, the function compares the destination address with that path’s temporary-source address:

- If the destination address is lower, indices are processed from zero upward.
- Otherwise, indices are processed from count minus one downward to zero.

At each processed index, the source byte is read before the corresponding destination byte is written. The transformed path performs its byte transformation between those operations.

A zero count causes no source-byte reads and no destination-byte writes. It does not undo earlier dependency activity or the direct path’s field clear.

The count is the full unsigned 32-bit field value. It is not restricted to a byte-sized length, masked, clamped to temporary capacity, or checked against destination capacity. It is sampled once after the dependency returns zero; later writes that alias its storage do not cause the loop bound to be reread.

The function does not append a terminator, report a separate output length, or preserve destination contents beyond what follows from the actual count and aliasing.

## Side effects

### Explicit effects

For a non-null destination, this function explicitly performs:

- Control-word reads before dependency selection.
- One dependency call.
- A count read only after a zero dependency result.
- Bytewise destination writes only after a zero dependency result.
- A 32-bit zero write to context offset 92 only after a zero result from F-8364fe87b70d.

For a nonzero dependency result, this function performs no copy and no direct-path field clear. Dependency side effects are not rolled back.

### Dependency and aliasing effects

Neither dependency is shown in the supplied evidence. Either may affect state beyond its nominal output buffer. In particular, the contract does not assume either preserves global state.

F-0b6f5f32b71e receives the context directly and may alter its count before this function reads it. F-8364fe87b70d could also affect reachable state through mechanisms not shown here. Accordingly, the copy count is the value observed after the call and, on the direct path, after the explicit field clear.

No destination/context alias check exists. If the destination overlaps the context, destination writes can overwrite context bytes. The copy still uses the already sampled count, subject to the underlying memory operations remaining valid.

### Lifetime and ownership

Temporary output storage is local to this invocation. The function does not return its address.

The caller’s destination remains caller-supplied storage. No explicit allocation or deallocation operation appears in this function.

Whether either dependency retains the temporary address, consumes the context-held value, or manages resources is unknown. Safe retention of the temporary address beyond this invocation cannot be inferred.

### Errors and unsafe extents

There is no explicit error result for:

- An invalid context address.
- An invalid destination address.
- Insufficient destination capacity.
- An excessive count.
- Uninitialized temporary bytes.

A count above 19 on the direct path or above 20 on the transformed path exceeds the respective visible array-backed source capacity. The evidence does not define valid output for such executions. Memory faults or other invalid-memory consequences must not be presented as supported error handling.

## Dependencies

| Opaque ID | Argument order | Established use |
|---|---|---|
| F-8364fe87b70d | 32-bit context-held value, temporary output address | Called on the direct-byte path; zero enables field clearing and untransformed copying |
| F-0b6f5f32b71e | Temporary output address, context address | Called on the transformed-byte path; zero enables transformed copying |

Neither call includes an explicit output capacity. Their output sizes, status meanings, state changes, resource behavior, and guarantees on error are not established.

An external 32-bit datum supplies the null-destination return value. Its relationship to either dependency’s status domain is unknown.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless otherwise stated, all memory accesses and dependency operations satisfy their preconditions.

1. **Null destination.** If the external datum contains 9, the function returns 9 without accessing the context or calling either dependency.

2. **Direct-byte selection.** If both control-word reads observe 0x202, F-8364fe87b70d is selected. If it returns zero, supplies bytes 0x11 and 0x7e, and the subsequently read count is 2, those bytes are copied unchanged. The offset-92 field is cleared before copying, and the function returns zero.

3. **Missing selection bit.** If the low byte is 2 but the second control-word read has bit 0x200 clear, F-0b6f5f32b71e is selected.

4. **Low-byte mismatch.** If the first control-word read has low byte 3, the transformed path is selected without the second control-word read.

5. **Transformation.** If F-0b6f5f32b71e returns zero, the count is 3, and the temporary bytes are 0x00, 0x01, and 0xff, the destination receives 0x24, 0xd9, and 0x6f respectively.

6. **Nonzero dependency result.** If the selected dependency returns 1 or 0xffffffff, that exact value is returned. This function performs no copy, count read, or field clear. Any dependency side effects remain.

7. **Successful zero-length direct result.** If F-8364fe87b70d returns zero and the subsequently read count is zero, the offset-92 field is still cleared. No output bytes are accessed.

8. **Post-call count.** If F-0b6f5f32b71e changes the count from 5 to 2 before returning zero, this function uses 2, not the earlier value.

9. **Direction.** For a positive count, a destination below the temporary-source address is written in increasing index order. A destination at or above it is written in decreasing index order.

10. **Capacity boundary.** Counts of 19 on the direct path and 20 on the transformed path fit the respective visible temporary regions, provided the dependency initializes every byte read and the destination is large enough. Larger counts have no defined safe-output contract from this evidence.

## Uncertainties

The following are **documented_only**:

- The context’s semantic type and complete layout.
- The external datum’s purpose.
- Dependency status meanings beyond zero-gated copying.
- How dependencies determine or limit output size.
- Whether dependencies initialize all bytes covered by the eventual count.
- Resource ownership associated with the offset-92 value.
- Dependency effects on globals, context fields, and other reachable memory.
- Platform endianness, alignment requirements, and pointer-comparison semantics outside the represented target model.
- Behavior after invalid memory accesses or dependency overruns.

The direction rule suggests overlap-aware intent, but that inference does not establish that callers can validly reference the invocation’s temporary storage. The field clear may suggest consumption or invalidation of a handle-like value; neither interpretation is established.

## Evidence

This contract is based solely on the supplied static function representation: its branches, field offsets, access widths, local array extents, dependency arguments, status gating, arithmetic, and byte-access order.

Established behavior is separated above from inferred intent and unavailable dependency semantics. No dependency implementation, external datum definition, or broader context definition was supplied. No runtime tests have been performed.
