<!-- Trusted provenance: F-d93855eeef1a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d93855eeef1a

## Purpose

F-d93855eeef1a conditionally extracts a 32-bit value from a referenced two-word record.

The operation examines the first word of that record. If it equals the discriminator −65,536, the operation copies the record’s second word to a caller-supplied destination and returns zero. Otherwise, it returns 28 without writing to the destination.

This is a discriminator check followed by a conditional copy, not a general record validator. The supplied representation establishes no checks for null addresses, allocation size, address validity, or destination writability.

**Inferred intent:** the first word identifies a particular record form, and the second word contains the value associated with that form. Neither the record’s semantic type nor the meaning of status 28 is established.

## Interface

### Inputs

The operation takes two arguments:

1. **Container address.** An address-bearing argument used to locate a pointer field four bytes after the container’s base address.
2. **Output address.** A pointer to the location that receives one 32-bit value on the matching-discriminator path.

The recovered interface represents the container address as a 32-bit integer-like argument and the output address as a pointer to a 32-bit integer. The address-bearing use of the first argument is established; whether its original source-level type was an integer, a pointer, or another address representation is unknown.

The relevant memory layout is:

| Region | Relative byte offset | Relevant contents |
|---|---:|---|
| Container | 0 | Not accessed by this operation |
| Container | 4 | Pointer to a record |
| Referenced record | 0 | 32-bit discriminator |
| Referenced record | 4 | 32-bit payload |

The layout describes only the locations used here. It does not establish complete object sizes, additional fields, allocation boundaries, or a requirement that the record occupy a separate allocation from the container.

### Return value

The result is a 32-bit value with two normal outcomes:

| Condition | Return value | Output effect |
|---|---:|---|
| Discriminator equals −65,536 | 0 | Writes the selected 32-bit payload |
| Discriminator differs from −65,536 | 28 | No destination write |

The discriminator has the 32-bit bit pattern `0xFFFF0000`. The mismatch result has the value `0x0000001C`.

Both result values are nonnegative and have the same numerical interpretation as signed or unsigned 32-bit values. No symbolic error name is established for 28.

### Preconditions

For an ordinary completed call:

- The container address must permit reading the pointer field at byte offset 4.
- The pointer obtained for the discriminator check must permit reading the record’s first 32-bit word.
- On the matching path, the pointer obtained by the subsequent pointer-field read must permit reading a 32-bit payload at byte offset 4.
- On the matching path, the output address must permit a 32-bit write.
- All accessed addresses must meet the target execution environment’s validity and alignment requirements.

The operation does not enforce these preconditions. A mismatch does not require the output destination to be readable or writable because that path does not dereference it.

## Behavior

### Discriminator check

The operation first reads the record pointer from the container’s field at byte offset 4. It then reads the 32-bit discriminator at the beginning of that referenced record.

The comparison is exact equality against −65,536. It is not a sign test, range test, mask test, or comparison involving the payload.

For example, −65,535 does not match, and neither does −65,537.

### Mismatching discriminator

If the discriminator differs from the required value:

- The operation selects return value 28.
- It does not read the payload word.
- It does not read the container’s pointer field a second time.
- It does not access the output destination.
- It performs no explicit memory write.

Any previous contents of a valid output destination remain unchanged by this operation. A caller must use the return value to distinguish a newly written result from an old destination value.

### Matching discriminator

If the discriminator matches, the operation reads the pointer field at container byte offset 4 again. Through the pointer obtained by this second read, it reads the 32-bit word at record byte offset 4.

It then writes that word to the output address and returns zero.

The payload is copied without numerical transformation. There is no sign extension, truncation from a wider source, scaling, interpretation as an address, or validation of the payload’s value. All 32-bit payload bit patterns are accepted by the visible operation.

The second pointer-field read is significant. The supplied representation does not retain the pointer used for the discriminator check as the sole source of the payload address. With stable memory, both reads yield the same pointer. If the execution environment permits a changed value to be observed between those reads, the payload can come from a different record. The operation does not check that second record’s discriminator.

This is not a guarantee of concurrency safety or atomic snapshot behavior. The legality and visibility of concurrent or device-driven changes are properties of the surrounding execution environment, which is not supplied.

### Access order

The explicit memory accesses occur in this order:

| Order | Access | Condition |
|---:|---|---|
| 1 | Read the pointer field at container byte offset 4 | Always |
| 2 | Read the discriminator through that pointer | Always, after the pointer read |
| 3 | Read the pointer field at container byte offset 4 again | Matching discriminator only |
| 4 | Read the payload at byte offset 4 through the second pointer | Matching discriminator only |
| 5 | Write the payload to the output destination | Matching discriminator only |

This sequence describes the supplied function representation. It does not establish bus transaction widths, compiler-level access preservation, atomicity, synchronization, or memory-ordering guarantees.

### Arithmetic and widths

The operation uses fixed byte offsets and a single 32-bit equality comparison. There is no payload arithmetic, length arithmetic, iteration, or narrowing conversion.

The payload lies four bytes after the record base because the represented record elements are 32-bit words. Address validity near the boundary of the address space is not checked. The representation does not establish defined wraparound behavior for invalid address calculations.

## Side effects

The only explicit memory modification is the conditional 32-bit destination write.

There are no visible calls, allocations, deallocations, logging operations, or explicit accesses to named global state. This does not establish that all referenced storage is private: the supplied addresses may refer to shared storage, global storage, or memory with environment-specific behavior.

### Ownership and lifetime

The operation does not visibly acquire ownership of either input object. It does not release, resize, replace, or retain any allocation through an explicit operation.

The caller or surrounding environment must maintain the lifetime of each storage region for the accesses that actually occur. The function provides no lifetime validation.

### Aliasing

No explicit check requires the output destination to be disjoint from the container or the referenced record.

When such aliasing is permitted by the execution environment, the output write happens after the discriminator and payload reads. Consequently, the destination may overwrite storage that was used earlier in the call.

For example, an output destination overlapping the discriminator can replace that discriminator with the payload after the successful check. An output destination overlapping the container’s pointer field can change the pointer used by later calls, provided that write and overlap are valid in the target layout.

These are consequences of the access order, not evidence that the interface was designed for overlapping objects. Original language-level aliasing constraints are unknown.

## Dependencies

No dependency calls are present.

F-d93855eeef1a performs the check and copy directly. Its observable result does not depend on a visible helper’s return value, memory-ownership convention, or preservation of global state.

Platform pointer representation, byte order, alignment rules, and memory-access semantics remain environmental dependencies rather than identified callable dependencies.

## Acceptance examples

The following are conditional acceptance criteria derived from static evidence. They are not reports of runtime tests.

### Matching record with a positive payload

Given a stable, accessible record whose discriminator is −65,536 and whose payload is 123, with a valid output destination:

- The destination becomes 123.
- The return value is zero.

### Matching record with all payload bits set

Given a matching discriminator and payload bit pattern `0xFFFFFFFF`:

- The destination receives `0xFFFFFFFF`.
- The return value is zero.

The payload is not rejected because it represents a negative signed integer.

### Mismatch preserves an existing output

Given a discriminator of zero and an output destination initially containing 77:

- The return value is 28.
- The destination remains 77.
- The payload is not read.

The same mismatch behavior applies to every discriminator bit pattern other than `0xFFFF0000`.

### Mismatch with no usable output destination

Given valid storage for the pointer-field read and discriminator read, and a nonmatching discriminator:

- The output address is not dereferenced.
- The normal result is 28.

This does not make an unusable output address acceptable on the matching path.

### Matching record with an unreadable payload

Given a readable matching discriminator but no readable payload at the address selected by the second pointer-field read:

- The operation attempts the payload read.
- No normal error return for that failure is established.
- A successful destination write and return of zero are not guaranteed.

### Pointer changes between reads

If the execution environment permits the first pointer-field read to select one record and the second to select another:

- The first record supplies the discriminator.
- If that discriminator matches, the second record supplies the payload.
- The second record’s discriminator is not checked.

This example is conditional on such changing reads being meaningful in the target environment; it does not define otherwise-invalid concurrent behavior.

### Destination overlaps the discriminator

Given a valid matching record, a payload of 9, and a valid output destination referring to that record’s discriminator word:

- The check succeeds using the original discriminator.
- The payload is read before the destination write.
- The discriminator word becomes 9.
- The return value is zero.

## Uncertainties

The following items are **documented_only**:

- The original function name, source-level types, and semantic record type.
- The semantic meaning of the discriminator and status 28.
- Complete container and record layouts.
- Exact pointer width and byte order beyond the address-bearing layout visible in the supplied representation.
- Original language-level aliasing restrictions.
- Alignment requirements and behavior of misaligned accesses.
- Behavior on null, dangling, inaccessible, or otherwise invalid addresses.
- Whether invalid accesses trap, terminate execution, invoke an external handler, or have another outcome.
- Atomicity, synchronization, and the visibility of externally changing memory.
- Whether referenced storage has memory-mapped or other special read/write effects.

No fallback, retry, validation error, or recovery behavior should be inferred for these cases.

## Evidence

**Established static evidence:** the supplied representation contains one discriminator comparison, two conditional return outcomes, a second pointer-field read on the matching path, one payload read, and one destination write. It contains no dependency calls or explicit resource-management operations.

**Inferred intent:** a tagged record exposes its second word only when its first word identifies the accepted form.

**Verification limit:** no runtime tests have been performed. The acceptance examples specify consequences of the visible operations under their stated preconditions; they do not establish behavior for invalid memory, concurrent mutation, or unspecified platform properties.
