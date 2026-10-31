<!-- Trusted provenance: F-0eeabd45a2a0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0eeabd45a2a0

## Purpose

F-0eeabd45a2a0 conditionally records the address of a caller-supplied reference slot inside the object named by that slot.

The operation has three outcomes:

- A null reference-slot address is rejected without a memory access.
- A slot containing a zero object address, or an object whose eligibility field is zero, is rejected without a memory write.
- A nonzero object address with a nonzero eligibility field causes the reference-slot address to be written into that object.

The successful result is a null pointer. Every rejected case returns the pointer value represented by address `0x84`. That address is returned as a value; this function does not access memory at that address.

“Reference slot,” “object,” and “eligibility field” are descriptive terms for the observed relationships. They do not establish a particular object type, ownership protocol, or lifecycle.

## Interface

### Inputs

The function accepts three arguments:

1. **Reference-slot address.** A pointer to storage containing an integer representation of an object address. A null pointer is an explicitly handled input.
2. **Auxiliary integer.** When the reference-slot address is nonnull, this argument is replaced internally by the value read from the slot. When the reference-slot address is null, its incoming value cannot change the rejection result.
3. **Auxiliary integer.** When a nonnull slot contains a nonzero object address, this argument is replaced internally by the object’s eligibility field. On all other paths, its incoming value cannot change the rejection result.

Consequently, neither auxiliary argument affects the observable result or memory accesses for a fixed reference-slot address and fixed relevant memory contents.

The apparent interface uses a 32-bit address model: integer-held addresses and pointer-valued storage are treated as four-byte quantities. This is consistent with the supplied representation, but an independently verified ABI or type declaration was not provided. Width-dependent statements below use that model.

### Relevant storage layout

| Storage | Location | Meaning | Access |
|---|---|---|---|
| Reference slot | At the first argument’s address | Integer representation of the object address | Read if the first argument is nonnull |
| Registration field | Object address plus `0x58` bytes | Address of the reference slot | Written only on success |
| Eligibility field | Object address plus `0x60` bytes | Zero rejects; any nonzero value permits the write | Read if the slot contains a nonzero address |

Under the apparent 32-bit model, each listed access covers four bytes. The registration and eligibility fields begin eight bytes apart and do not overlap.

No complete object size, alignment requirement, allocation boundary, or meaning for intervening bytes is established.

### Output

The return type is pointer-shaped:

- **Null:** the eligibility test passed and the registration-field write completed.
- **Address `0x84`:** a rejection condition was encountered.

The function does not return the object address, the previous registration-field contents, or an allocated resource. The nonnull result is a fixed result value, not evidence of a dereferenceable object.

### Preconditions for defined memory behavior

A null first argument needs no associated readable storage.

For a nonnull first argument:

- The reference slot must support the required integer read.
- If that read produces a nonzero object address, the eligibility field must support the required integer read.
- If the eligibility field is nonzero, the registration field must support the required pointer-sized write.

Address calculation, alignment, and storage representation must also be valid for the actual execution environment. The function does not validate these conditions.

## Behavior

### Rejection before object access

A null reference-slot address produces the fixed rejection pointer `0x84`. There is no slot read, eligibility-field read, or registration-field write.

With a nonnull reference-slot address, the function first reads the object address from that slot. If the value is zero, it returns `0x84`. It does not access either object-relative field.

These are separate rejection conditions. A valid slot containing zero is read; a null slot address is not.

### Eligibility check

If the slot contains a nonzero object address, the function reads the integer field at byte offset `0x60` from that address.

A zero eligibility value causes rejection. The registration field is neither read nor written on this path.

Any nonzero eligibility value permits registration. The supplied representation does not restrict acceptance to one, positive values, a particular bit pattern, or a range. In particular, a negative integer value is nonzero and passes this check.

### Registration

After a nonzero eligibility value has been obtained, the function stores the original reference-slot address at byte offset `0x58` in the selected object.

The stored value is the address of the slot, not:

- The object address read from that slot.
- The eligibility value.
- Either auxiliary argument.
- The contents previously held in the registration field.

There is no read of the old registration field and no check that it is empty or already refers to this slot. Its previous contents are overwritten.

After the write completes, the function returns null. No further explicit memory access appears in the supplied body.

### Observable access order

For a successful invocation, the source-level memory-access order is:

1. Read the object address from the reference slot.
2. Read the eligibility field from that selected object.
3. Write the original reference-slot address into that object’s registration field.

The slot and eligibility field are each read once. The registration field is written once and is not read.

This ordering describes the supplied function body. It does not establish atomicity, inter-thread synchronization, or ordering guarantees for device memory.

### Address arithmetic and aliasing

The offsets `0x58` and `0x60` are byte offsets applied to the integer-held object address. They are not element counts scaled by the size of an integer.

The function uses the object address obtained by its initial slot read; it does not reread the slot before writing. Aliasing can therefore cause the final store to modify storage that was read earlier.

For example, if the reference slot itself occupies the object’s registration field, the initial read can select that object, and the successful store then replaces the slot’s contents with the slot’s own address. There is no alias-exclusion check in the supplied body.

The source does not establish behavior for address arithmetic that exceeds the applicable integer range. Modular wrapping must not be assumed.

## Side effects

The only explicit memory mutation is the successful registration-field write.

On rejection, the function performs no explicit memory write. A rejection after the eligibility check can nevertheless have performed both required reads.

The supplied body contains no allocation, release, external call, logging operation, or explicit global-variable access. The fixed return address does not imply a global-data read.

No ownership transfer is established. Recording a slot address does not by itself prove that the object owns the slot or that either storage region may be released. Conversely, this function does not release either region or clear a previous relationship.

**Inferred intent:** the stored slot address may serve as a reverse reference or registration link.

**Not established:** how long the slot must remain alive, who later consumes the stored address, or whether an existing link requires separate cleanup.

## Dependencies

No callee dependencies appear in the supplied body. F-0eeabd45a2a0 performs its decisions and memory accesses directly.

Its environmental dependencies are:

- The integer and pointer representations used by the actual ABI.
- Readable storage for the reference slot and, conditionally, the eligibility field.
- Writable storage for the registration field when eligibility is nonzero.
- Valid address arithmetic and access alignment.
- Any caller-provided synchronization needed when these locations are shared.

No behavior of another function is needed to explain the observed paths. No external ownership or global-state preservation guarantee is assumed.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Except where stated otherwise, referenced storage and address calculations are assumed valid.

| Situation | Required result | Required accesses |
|---|---|---|
| The reference-slot address is null; auxiliary arguments have arbitrary values | Return pointer value `0x84` | No memory access |
| A valid reference slot contains zero | Return pointer value `0x84` | Read the slot only |
| A valid slot selects an object whose eligibility field is zero | Return pointer value `0x84` | Read the slot, then eligibility; do not write registration |
| A valid slot selects an object whose eligibility field is one | Store the slot address in registration; return null | Read slot, read eligibility, write registration |
| Eligibility is any other representable nonzero integer, including a negative value | Same success behavior | Same ordered accesses |
| Registration already contains a different value, while eligibility is nonzero | Overwrite registration with the supplied slot address; return null | Do not read or validate the old registration value |

Additional boundary conditions:

- **Auxiliary-argument independence:** Changing either auxiliary argument while keeping the first argument and relevant memory fixed must not change the result or explicit memory-access sequence.
- **No access through a zero object address:** A valid slot containing zero must not trigger an access at address `0x58` or `0x60`.
- **Conditional write requirement:** An object with zero eligibility need not have a writable registration field for this function’s explicit path, because that field is not accessed.
- **Aliased slot and registration:** If the reference slot occupies the selected object’s registration field and eligibility is nonzero, the final write stores that slot’s own address into the shared location. The result remains null if the write completes.
- **Invalid storage:** An unreadable nonnull slot, unreadable eligibility field, or unwritable required registration field has no established normal-return behavior. Such cases are not specified as returning `0x84`.

## Uncertainties

The following remain **documented_only**:

- **Verified ABI:** Four-byte integers and pointers are the apparent representation, not independently confirmed runtime facts.
- **Return-value meaning:** The operational distinction between null and `0x84` is established. A named error, status enumeration, or broader convention is not.
- **Object semantics:** The eligibility field’s domain meaning and the registration field’s consumers are unknown.
- **Ownership and lifetime:** No retention, transfer, release, or cleanup protocol is established.
- **Exceptional accesses:** Fault handling and recovery for invalid addresses are not shown.
- **Arithmetic limits:** Behavior for an unrepresentable offset calculation is not established.
- **Concurrency:** No locking, atomic operation, race handling, or stability guarantee is shown.
- **Special memory:** Behavior for volatile storage, memory-mapped devices, or externally changing memory is not established.
- **Caller obligations:** Whether surrounding code imposes stronger nonaliasing, initialization, or lifecycle rules is unknown.

## Evidence

Established static evidence consists of the supplied body for F-0eeabd45a2a0:

- The first argument is checked for null before its storage is read.
- A nonnull slot supplies the object address.
- A zero object address prevents object-relative accesses.
- The field at byte offset `0x60` is tested for zero.
- A nonzero field permits a store of the original slot address at byte offset `0x58`.
- Successful completion returns null; rejection returns address `0x84`.
- No calls or explicit resource-management operations appear.

The reverse-reference interpretation is inferred intent, not an established lifecycle contract. No runtime tests have been performed.
