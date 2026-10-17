<!-- Trusted provenance: F-05ca8baffd5f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-05ca8baffd5f

## Purpose

F-05ca8baffd5f performs a fixed, in-place update to two adjacent fields in caller-supplied storage:

- It replaces the low five bits of a 16-bit field with the value 1, preserving that field’s upper eleven bits.
- It writes the value 5 to the following 8-bit field.

The field names, enclosing object type, and meanings of the values 1 and 5 are unknown. The evidence establishes a bit-field update and a separate byte assignment, not a particular application-level state transition.

This contract describes the behavior established by the supplied static evidence. No runtime tests have been performed.

## Interface

### Input

The function accepts one argument, used as a base address for memory accesses. The supplied representation declares that argument as an integer rather than a typed object pointer.

The required storage layout is:

| Offset from base | Width | Access | Required behavior |
|---|---:|---|---|
| 0x1C, decimal 28 | 16 bits | Read, then write | Replace the low five bits with 1; preserve the upper eleven bits |
| 0x1E, decimal 30 | 8 bits | Write only | Store 5 |

The offsets identify adjacent, non-overlapping fields: the 16-bit field occupies offsets 0x1C and 0x1D, and the byte field occupies offset 0x1E.

The function does not read an object pointer from the supplied storage. It uses the argument itself as the base for both field locations. No length, capacity, type discriminator, or separate output address is supplied.

### Preconditions

For normal completion:

- The base must identify storage whose 16-bit field at offset 0x1C is readable and writable.
- The byte at offset 0x1E must be writable.
- The effective addresses must be valid under the execution environment’s address arithmetic and memory-access rules.
- The 16-bit access must satisfy the target’s alignment requirements, if any.
- The storage must remain valid throughout the operation.

There is no explicit validation of these conditions.

The evidence does not establish the width of the address space, the calling convention, or the conversion rules between the integer argument and an address. In particular, the argument must not be assumed to support arbitrary host-width pointers.

### Output

The function returns no value. Its intended observable output is the mutation of the two fields.

There is no status result, error code, returned pointer, or separately written success indicator.

## Behavior

### Ordered field update

The established operation order is:

1. Read the 16-bit value at base offset 0x1C.
2. Write a replacement 16-bit value to that same location.
3. Write the 8-bit value 5 at base offset 0x1E.
4. Return normally, provided the accesses complete.

This ordering describes the supplied operation sequence. It is not a guarantee of atomicity, synchronization, processor memory ordering, or visibility to concurrent observers.

The previous value of the byte at offset 0x1E is not read by the function. Its replacement is unconditional once execution reaches that write.

### Sixteen-bit transformation

The first field is treated as an unsigned 16-bit quantity. Its bits are numbered from bit 0, the least significant bit, through bit 15, the most significant bit.

The replacement has these properties:

- Bit 0 is set.
- Bits 1 through 4 are cleared.
- Bits 5 through 15 retain their values from the initial read.

Equivalently, if the initial 16-bit value is \(w\), the replacement is:

\[
(w \mathbin{\mathrm{AND}} \text{0xFFE0}) \mathbin{\mathrm{OR}} \text{0x0001}
\]

The mask 0xFFE0 preserves exactly eleven upper bits and clears exactly five lower bits. The final operation sets only bit 0. Every result therefore has low five bits equal to binary 00001.

The result always fits within 16 bits. There is no addition, subtraction, saturation, carry propagation, or signed comparison in this field transformation. The supplied representation permits intermediate integer promotion, but the resulting stored 16-bit value is fully determined for every possible initial 16-bit value.

Preservation concerns the value captured by the initial read. The function does not reread the field before writing. No mechanism is shown for retaining an intervening change made by another actor.

### Byte assignment

After the 16-bit write, the function stores the unsigned 8-bit value 5 at offset 0x1E.

This is a complete byte replacement, not a masked update. All previous bits of that byte are discarded.

### Repeated invocation

For stable, ordinary storage, applying the function again produces the same two field values as applying it once. After the first completed call, the 16-bit field already has the required low-five-bit pattern, and the byte already contains 5.

This value-level idempotence does not establish that subsequent invocations omit accesses. The described operation still includes a 16-bit read, a 16-bit write, and an 8-bit write. If accesses themselves have external significance, repeated invocation need not be equivalent to a single invocation.

### Boundary and exceptional cases

The function contains no conditional branches based on field contents. All 65,536 possible initial 16-bit values receive the same transformation, and every previous byte value is replaced with 5.

There is no explicit null check, bounds check, alignment adjustment, recovery path, or rollback.

If an access fails, the evidence does not establish how failure is reported or whether execution can resume. In particular, the update is not transactional: failure at the byte write does not imply that the preceding 16-bit write is undone.

## Side effects

### Direct memory effects

The only explicit memory modifications are:

- A 16-bit store at base offset 0x1C.
- An 8-bit store at base offset 0x1E.

No other field, global location, allocation, release, logging action, or external operation is explicitly accessed.

The 16-bit store targets the whole field even though its upper eleven logical bits are preserved. The preservation claim is therefore a claim about the computed value, not a claim that only selected physical bits are written.

### Ownership and lifetime

The function operates on existing storage. It does not allocate memory, free memory, return ownership, or record the base for later use in the supplied body.

The caller remains responsible for providing storage with a suitable lifetime and access permissions. The mutation alone does not establish a broader ownership-transfer convention.

### Endianness and concurrency

The bit transformation is defined on the numeric 16-bit field. The byte encoding of that field depends on target endianness, which is not specified. The separate byte at offset 0x1E receives the value 5 regardless of the field’s endianness.

No lock, atomic operation, memory barrier, or concurrency protocol is present in the evidence. Concurrent access requires an external contract. A concurrent writer’s changes may be overwritten because the replacement is based on the earlier read.

Behavior for memory-mapped devices or other access-sensitive storage is not established.

## Dependencies

No called functions are present in the supplied body. There are therefore no dependency function IDs to enumerate and no unknown callee effects to incorporate.

Execution relies on the environment’s integer-to-address interpretation, effective-address calculation, memory-access semantics, and return mechanism. These are environmental prerequisites, not identified function dependencies.

## Acceptance examples

The following are conditional acceptance criteria derived from static evidence, not observed test results. Unless otherwise stated, they assume valid, stable, ordinary writable storage and successful completion.

| Initial 16-bit field | Initial byte | Required final 16-bit field | Required final byte |
|---:|---:|---:|---:|
| 0x0000 | 0x00 | 0x0001 | 0x05 |
| 0x001F | 0xFF | 0x0001 | 0x05 |
| 0x0020 | 0x01 | 0x0021 | 0x05 |
| 0xABCD | 0x7E | 0xABC1 | 0x05 |
| 0xFFFF | 0xA5 | 0xFFE1 | 0x05 |
| 0xFFE1 | 0x05 | 0xFFE1 | 0x05 |

Additional conditional criteria:

- **Upper-bit preservation:** For any initial 16-bit value, masking the initial and final values with 0xFFE0 yields equal results.
- **Low-bit replacement:** For any initial 16-bit value, masking the final value with 0x001F yields 0x0001.
- **Byte independence:** Changing only the previous byte at offset 0x1E does not change either required final field value.
- **Unrelated storage:** In ordinary memory without external interference, bytes outside offsets 0x1C through 0x1E are not directly modified by this function.
- **Repeated calls:** Two successful calls on unchanged storage yield the same final field values as one successful call.
- **Access sequence:** Where the environment permits observing the described accesses, the 16-bit read precedes its replacement write, which precedes the byte write. This does not imply an inter-thread visibility guarantee.
- **Invalid storage:** No particular return value, diagnostic, or recovery behavior is required by this evidence when an address is invalid. Such behavior remains documented_only rather than an invented error contract.

## Uncertainties

### Inferred intent

The low-five-bit replacement is consistent with selecting a packed state or mode while retaining unrelated flags. The byte assignment is consistent with setting a second state-related field. Neither interpretation is established.

### Unknowns — documented_only

The supplied evidence does not determine:

- The enclosing object’s type, size, or field names.
- The semantic meanings of the assigned values.
- Address width, argument ABI width, or pointer provenance requirements.
- Endianness and alignment rules.
- Effective-address overflow or wrapping behavior.
- Fault handling for invalid, inaccessible, or misaligned storage.
- Atomicity and cross-thread visibility of the accesses.
- Whether actual callers provide shared memory or access-sensitive storage.
- Any caller-level ownership or synchronization convention.

These gaps do not alter the established field transformation, but they limit portability and environmental guarantees.

## Evidence

The supplied static body establishes one input used as a base address, one 16-bit read-modify-write at offset 0x1C, one subsequent byte write at offset 0x1E, and a return without a value.

The mask and assigned constants establish the exact bit-level results described above. The body contains no calls, validation branches, explicit allocation or release, or error-reporting operation.

No runtime tests, access traces, ABI verification, or target-environment checks have been performed. The acceptance examples are derived requirements, not execution evidence.
