<!-- Trusted provenance: F-bde2e4d6e759; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bde2e4d6e759

## Purpose

F-bde2e4d6e759 coordinates validation, temporary preparation, allocation, an external transfer operation, publication of two outputs, and release of temporary backing storage.

The established behavior is narrower than a complete data-conversion contract: the supplied static evidence establishes call ordering, argument construction, explicit writes, and return values, but not the contents produced by the dependencies.

Successful completion returns a null pointer value. Three detected failure conditions return distinct non-null pointer-shaped status values. These values are not established to identify readable objects.

**Inferred intent:** the function appears to materialize data described by an input record into newly allocated storage, mark the record, and return the resulting address and size through output pointers. The meanings of the record, size, marker, and allocated contents remain dependent on undocumented callees.

## Interface

The function accepts three pointer arguments:

| Argument | Established use |
|---|---|
| Input record pointer | Supplies an interior address at byte offset 4 and a byte at offset 12. Receives a one-byte write at offset 0 on the allocation-success path. |
| Address-output pointer | Receives one 32-bit value representing the successful allocation result. |
| Size-output pointer | Receives one 32-bit value taken from temporary storage after the external transfer call. |

The return is pointer-shaped, with these observed values:

| Return value | Trigger |
|---|---|
| Null | The allocation dependency returned nonzero, and subsequent calls returned control. |
| `0xFFFFFFCE` | At least one argument pointer is null. |
| `0xFFFFFFD5` | All arguments are non-null, but the first dependency returned zero. |
| `0xFFFFFF94` | Initial validation succeeded, but the allocation dependency returned zero. |

Interpreted as signed 32-bit quantities, the three non-null statuses correspond to −50, −43, and −108. That interpretation does not establish named error categories or permit dereferencing these values.

### Widths and record access

The visible representation uses:

- One-byte record reads and writes.
- Four-byte output stores.
- A 32-bit allocation-result value that is subsequently used as an address.
- A four-byte temporary field used as an allocation argument, transfer argument, and published size.
- A temporary handle or descriptor occupying two four-byte words.

The record byte at offset 12 is converted through a character type and then to an integer before being passed to two dependencies. Character signedness is not independently established by the supplied representation. For byte values below `0x80`, the resulting integer is unambiguous. For higher values, signed character interpretation yields −128 through −1, whereas unsigned interpretation yields 128 through 255.

No range validation, clamping, unsigned normalization, or multiplication of that byte is visible.

### Preconditions

Null checking is explicit. Other pointer-validity requirements are not checked locally.

For defined, useful execution, the relevant record locations must be accessible, and both output pointers must support four-byte writes when the allocation-success path is reached. Dependencies may impose stronger requirements, including a larger accessible region beginning at record offset 4.

No non-aliasing requirement is enforced. Overlap among the record, output destinations, and dependency-accessible storage can affect the result.

## Behavior

### 1. Argument rejection

The function checks whether any of the three arguments is null.

If so, it returns `0xFFFFFFCE`. It performs no explicit record-byte read, output write, or dependency call on this path. A non-null argument is not dereferenced merely because another argument is null.

There is no partial-success result for invalid arguments.

### 2. Initial validation

With all arguments non-null, the function reads the record byte at offset 12, performs the character-to-integer conversion, and calls F-bd3e46af741a with:

- The address four bytes beyond the record start.
- The converted byte value.

A zero return produces status `0xFFFFFFD5`. No later dependency is called, and no explicit write to the record or outputs occurs on this path.

Any nonzero return permits execution to continue. The function does not distinguish positive and negative nonzero results.

The dependency’s own reads, writes, and side effects are unknown. In particular, this failure path does not establish that the record or unrelated state remains unchanged.

### 3. Temporary preparation

After successful initial validation, the function reads the record byte at offset 12 again and performs the same conversion. It calls F-b20180f748aa with:

- The start of an 84-byte temporary region.
- The record address plus four bytes.
- The newly obtained converted byte value.

The second read is distinct from the first. If F-bd3e46af741a modifies that record byte, the two dependencies can receive different values.

The return from F-b20180f748aa is not tested.

The visible temporary layout places the 84-byte region immediately before a two-word temporary descriptor and a further four-byte field. Subsequent operations use the descriptor and field, although no explicit assignments initializing them appear in the supplied function body.

Consequently, the function relies on an unstated initialization mechanism or on details absent from the supplied representation. It is not established that F-b20180f748aa initializes only the 84-byte region, nor is it established that writing the adjacent fields is a valid part of its interface.

### 4. Allocation attempt

The function reads the four-byte temporary field and passes its value to F-d33f6d597e5d.

The dependency returns a 32-bit value:

- Zero selects the allocation-failure path.
- Any nonzero value selects the success path and is subsequently treated as an address.

There is no local check for a negative value, alignment, requested-size range, or destination capacity. A zero requested size is not handled specially by this function.

#### Allocation failure

When F-d33f6d597e5d returns zero:

1. The function selects return status `0xFFFFFF94`.
2. It calls F-cc6f52873d53 with the address of the temporary two-word descriptor.
3. It subtracts `0x54`, or 84 bytes, from that dependency’s returned value.
4. It passes the adjusted value to F-a19c2808de87.
5. If those calls return control, it returns the selected status.

There are no explicit record-marker or output stores on this path. Temporary cleanup still occurs.

### 5. Success-side transfer and publication

When F-d33f6d597e5d returns nonzero, the function performs these operations in order:

1. Stores the one-byte value `1` at record offset 0.
2. Calls an external dependency with three arguments: the successful allocation result, the first four-byte word of the temporary descriptor, and the current value of the four-byte temporary field.
3. Reads the temporary field again and stores that value through the size-output pointer.
4. Stores the successful allocation result through the address-output pointer.
5. Calls F-cc6f52873d53 with the temporary descriptor’s address.
6. Subtracts 84 bytes from the returned value and passes the result to F-a19c2808de87.
7. Returns null.

No opaque identifier for the external dependency was supplied. Its implementation and semantics are therefore documented_only rather than assigned a fabricated identity.

The external call’s return value is not checked. The visible function contains no transfer-failure branch, marker rollback, or conditional output publication after this call.

The temporary field is read at multiple stages. The allocation argument, transfer argument, and published size are not proven equal if an intervening dependency can modify that storage.

### 6. Cleanup arithmetic

The cleanup argument is the result of F-cc6f52873d53 minus `0x54`.

This adjustment is performed on the shown 32-bit value. No lower-bound, null, or validity check precedes it. Under ordinary 32-bit machine arithmetic, an input below 84 wraps to a high unsigned bit pattern; for example, zero becomes `0xFFFFFFAC`.

The adjustment suggests a payload-to-backing-allocation relationship, but that ownership model is inferred, not established.

## Side effects

### Explicit writes

The function’s explicit caller-visible writes occur only after a nonzero allocation result:

1. One byte containing `1` at record offset 0.
2. Four bytes through the size-output pointer.
3. Four bytes through the address-output pointer.

These are the ordering guarantees of the visible operations, not unconditional guarantees about final memory contents. Calls after or between these stores may have additional effects.

The output pointers need not be distinct according to the visible checks. If they designate the same four-byte destination, the address-output store occurs last and overwrites the preceding size-output store, provided both stores are valid and subsequent dependencies leave that memory unchanged.

Likewise, outputs overlapping the record can overwrite the marker or other record bytes. No overlap protection is visible.

### Allocation and ownership

The function requests storage through F-d33f6d597e5d and publishes its nonzero result. It does not visibly release that returned allocation.

The temporary backing object is routed through F-cc6f52873d53 and F-a19c2808de87 on both allocation outcomes after preparation.

These observations do not establish:

- Which dependency owns the temporary object.
- Whether the cleanup dependency releases memory.
- Whether the external dependency copies, moves, retains, or consumes its source.
- Which operation a recipient must use to release the published allocation.
- Whether any dependency preserves globals or other caller-visible memory.

### Errors and interrupted execution

The three explicit error returns cover null arguments, zero from initial validation, and zero from allocation.

No local recovery is visible for dependency faults, non-returning behavior, invalid memory accesses, or failures reported through mechanisms other than the tested zero returns. In particular, a record marker may already have been written when the external transfer call fails to return.

## Dependencies

| Opaque ID | Role established by this caller | Unknown contract |
|---|---|---|
| F-bd3e46af741a | Receives record offset 4 and the converted offset-12 byte; zero rejects further processing. | Validation meaning, accessible extent, mutation, and other side effects. |
| F-b20180f748aa | Receives temporary-region start, record offset 4, and a fresh converted byte value. | Initialization extent, descriptor construction, size production, and failure behavior. |
| F-d33f6d597e5d | Receives the temporary four-byte field; zero means failure to this caller. | Allocation policy, argument interpretation, ownership, alignment, and other effects. |
| F-cc6f52873d53 | Receives the temporary descriptor address during cleanup; returns a value adjusted by −84. | Descriptor semantics, mutation, and meaning of the returned address-like value. |
| F-a19c2808de87 | Receives the adjusted cleanup value on both allocation outcomes. | Whether it releases storage and what validity conditions it requires. |

The external three-argument dependency is an additional dependency whose opaque ID is unavailable. Its apparent transfer role is inferred from argument shape and surrounding operations; neither byte-copy semantics nor a byte count is independently established.

## Acceptance examples

These examples specify conditional observations, not runtime test results.

### Null argument

Given any one null argument, the function returns `0xFFFFFFCE`, makes none of the listed dependency calls, and performs none of its explicit record or output stores.

### Validation rejection

Given three non-null arguments and a zero result from F-bd3e46af741a, the function returns `0xFFFFFFD5`. It does not invoke preparation, allocation, transfer, or cleanup. Any mutation performed by the validation dependency remains outside the local guarantee.

### Byte conversion boundary

Given an offset-12 byte of `0x7F`, both conversion sites produce 127 if the byte remains unchanged between reads.

Given `0xFF`, the dependency argument is −1 under signed character interpretation or 255 under unsigned character interpretation. A specification consumer must resolve the target character signedness before selecting one expectation.

### Validation changes the input

Given that the initial offset-12 byte is 3 and F-bd3e46af741a changes it to 5 before returning nonzero, F-b20180f748aa receives 5, not a cached 3.

### Allocation failure still cleans up

Given successful validation and preparation followed by zero from F-d33f6d597e5d, the function invokes the cleanup pair and returns `0xFFFFFF94`. It does not explicitly set the record marker or publish either output.

### Successful publication

Assume preparation establishes a size field of 16 and a valid descriptor; allocation returns a nonzero address; all later calls return normally; and dependencies leave the relevant temporary fields and output locations unchanged.

The observable local sequence is marker write, external call with size argument 16, size-output store of 16, address-output store of the allocated address, cleanup, and null return.

The allocation’s contents are not established by this example without the external dependency’s contract.

### Output aliasing

Under the preceding success conditions, if the two output pointers designate the same valid four-byte location, that location contains the allocated-address value after the explicit stores. This result is conditional on cleanup not modifying the location.

## Uncertainties

The following are documented_only:

- The semantic format and full required extent of the input record.
- Character signedness for offset-12 values at or above `0x80`.
- How the temporary descriptor and four-byte field become initialized.
- Whether the apparent 84-byte temporary region is part of a larger valid object.
- The external dependency’s identity and transfer semantics.
- Whether the four-byte field is a byte count, element count, capacity, or another quantity.
- Dependency mutation of temporary storage, record data, outputs, or globals.
- Allocation and temporary-storage ownership rules.
- Semantic names for the three encoded error statuses.
- Behavior when cleanup receives an invalid adjusted value.
- Synchronization or concurrency guarantees; none are visible locally.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency map. Established claims are limited to visible argument checks, accesses, conversions, calls, branches, explicit stores, arithmetic, and return values.

Descriptions of validation, allocation, transfer, descriptor construction, and cleanup express inferred roles where the callee contracts are unavailable. Adjacent temporary storage is reported as visible layout evidence, not proof of valid cross-object initialization.

No implementation access or runtime tests have been performed. No claim is made that the supplied representation fully captures the original types, object boundaries, or dependency contracts.
