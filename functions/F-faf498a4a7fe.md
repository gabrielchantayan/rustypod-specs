<!-- Trusted provenance: F-faf498a4a7fe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-faf498a4a7fe

## Purpose

F-faf498a4a7fe conditionally updates links associated with a supplied record, then clears the record’s active key and two link fields. It first captures an owner address and invokes F-c08d1cf25b95. Removal proceeds only if the active key read after that call is nonzero.

The apparent intent is removal from a bucketed doubly linked structure. That interpretation is inferred from the link updates and bucket-entry replacement. The established behavior is the sequence of memory accesses, dependency calls, and stores described below; the supplied body does not establish membership, structural consistency, synchronization, or ownership rules.

## Interface

### Input and result

The function accepts one pointer to mutable record storage. It returns no explicit value and provides no status result.

The supplied representation uses 32-bit integer fields, including integer-held addresses. The field labels below describe their observed roles, not externally established types.

| Record offset | Width | Contract label | Use |
|---|---:|---|---|
| +0x00 | 32 bits | Owner address | Captured before the first dependency call and retained for later owner-relative accesses. |
| +0x04 | 32 bits | Active key | Controls removal eligibility; also participates in bucket selection. |
| +0x08 | 32 bits | Next address | Identifies a possible successor and supplies replacement forward-link or bucket-entry values. |
| +0x0c | 32 bits | Previous address | Identifies a possible predecessor and determines whether a bucket entry is replaced. |

The captured owner address supplies access to these fields:

| Owner offset | Width | Use |
|---|---:|---|
| +0x14 | One byte | A nonzero value enables F-682cc3fccb47. |
| +0x3c | 32 bits | Displacement used to form that dependency’s argument. |
| +0xcc | 32 bits | Mask source for bucket selection. |
| +0xd0 | 32 bits | Integer-held bucket-table base address. |

Bucket entries are four bytes wide. Neighbor records are accessed at +0x08 for a forward link and +0x0c for a backward link.

These widths describe the supplied representation. They do not establish portability to a different integer or pointer model.

### Preconditions

For defined normal completion, each address actually accessed must designate storage valid for the relevant read or write, with any required alignment. The supplied record must remain writable through the final stores when the removal path reaches them.

The owner-address field must be readable before F-c08d1cf25b95 executes. The active-key field must be readable after it returns. Remaining requirements depend on the selected branches and on mutations caused by dependencies or aliased writes.

No visible check validates:

- The input pointer, owner address, or neighbor addresses.
- Reciprocal links, membership, cycles, or non-aliasing.
- Bucket-table capacity or the calculated destination.
- The mask source or optional dependency argument.
- Memory lifetime after either dependency returns.

Unsigned bucket-address wrapping is established behavior, not an address-validity check. Separate signed address calculations also require an execution model in which their evaluation and conversion permit the intended accesses.

## Behavior

### Initial owner capture and unconditional call

The function reads the owner address from record offset +0x00, retains that value, and then invokes F-c08d1cf25b95 with no explicit arguments.

All subsequent owner-relative accesses use the captured address. Replacing the record’s owner-address field during the call does not redirect those accesses. Changes to storage at the captured owner address can affect later reads.

The active key and links are not captured before this call. Their subsequent reads can observe changes made by the dependency. A lack of explicit arguments does not establish that the dependency cannot reach the record, owner, or other relevant state through globals or aliases.

### Removal eligibility

After F-c08d1cf25b95 returns, the function reads the active key.

If it is zero, the function returns without explicit link updates, bucket writes, an optional dependency call, or clearing stores. The initial owner read and unconditional dependency call have already occurred, so this path is not necessarily side-effect-free.

If the active key is nonzero, removal continues. The eligibility decision is not repeated. Later changes to the active key do not independently cancel the remaining operations.

### Successor backward-link update

The function checks the next-address field.

If it is nonzero, the function stores the current previous-address value into the 32-bit field at +0x0c from the next address. If it is zero, this store is skipped.

For ordinary non-overlapping records, this bypasses the supplied record in the successor’s backward link. No reciprocal-link check occurs.

This store precedes the subsequent predecessor-or-bucket decision. If its destination overlaps a field used later, subsequent reads can observe the written value. The next-address check is not a guarantee that an independently changing field will retain the checked value until every later use.

### Predecessor or bucket replacement

The function next checks the previous-address field.

**Nonzero previous address.** The current next-address value is stored into the 32-bit field at +0x08 from the previous address. No bucket-table entry is explicitly written on this branch.

**Zero previous address.** The function calculates a bucket destination using the captured owner and stores the current next-address value there.

The bucket calculation has established **unsigned 32-bit arithmetic throughout its mask, index, scaling, and base-addition operations**:

- The mask source is treated as unsigned 32-bit for subtraction of one.
- Subtraction wraps modulo \(2^{32}\).
- The active key is converted to unsigned 32-bit for the bitwise intersection.
- Multiplication of the resulting index by four is unsigned 32-bit and wraps modulo \(2^{32}\).
- The table-base integer is converted to unsigned 32-bit for addition to the scaled index.
- That addition also wraps modulo \(2^{32}\), before the resulting integer is converted to a pointer.

With \(U_{32}\) denoting conversion to an unsigned 32-bit value, the mathematical relationships are:

\[
\begin{aligned}
M &= \bigl(U_{32}(\text{mask source})-1\bigr)\bmod 2^{32},\\
I &= U_{32}(\text{active key})\mathbin{\&}M,\\
S &= (4I)\bmod 2^{32},\\
A &= \bigl(U_{32}(\text{table base})+S\bigr)\bmod 2^{32}.
\end{aligned}
\]

The four-byte store targets the pointer obtained from \(A\). This is not an unbounded mathematical address calculation: both scaling and base addition can discard high bits.

The active key used here is read for this operation and need not equal the earlier value that admitted removal. No check requires a positive mask source, a power-of-two size, or an in-range index. A zero mask source produces `0xffffffff`.

The arithmetic result is established; the target environment’s integer-to-pointer conversion and the validity of the resulting destination remain separate requirements.

### Optional dependency

After the successor update and predecessor-or-bucket update, the function reads the byte at +0x14 from the captured owner.

If the byte is zero, F-682cc3fccb47 is skipped.

If it is nonzero, the function reads the 32-bit displacement at captured-owner offset +0x3c and invokes F-682cc3fccb47 with one argument formed from the record pointer converted to an integer, plus the displacement, plus 0x38.

The fixed displacement is 56 bytes. This calculation is separate from the unsigned bucket calculation. Its additions are represented as signed integer arithmetic; portable wrapping is not established. The record-pointer-to-integer conversion, possible signed overflow, and interpretation of the resulting argument depend on the target environment.

The call occurs after the explicit structural updates and before the final clearing stores. With non-overlapping storage and no relevant mutation, it therefore runs after unlinking while the supplied record still contains its prior active key and links.

### Final clearing

If the optional call is skipped or returns normally, the function issues three separate 32-bit zero stores in this order:

1. Active key at record offset +0x04.
2. Previous address at record offset +0x0c.
3. Next address at record offset +0x08.

These stores occur even if the optional dependency changed those fields. They are not an atomic reset.

There is no direct store to the record’s owner-address field. Aliased structural writes or dependency effects are not constrained by that observation.

### Read timing and expression ordering

The record is not treated as an entry-time snapshot. Apart from the captured owner address, relevant values are read as their operations are reached.

The body establishes the ordering between the major operations above. It does not establish an additional evaluation order among operands within one expression. This distinction matters for external mutation, unusual memory mappings, and aliases involving later-read fields.

## Side effects

### Explicit effects

On paths that reach the corresponding operations, the function can:

- Invoke F-c08d1cf25b95 once.
- Write a successor’s backward-link field.
- Write either a predecessor’s forward-link field or one calculated bucket entry.
- Invoke F-682cc3fccb47 once.
- Clear the supplied record’s active key and links.

The body does not explicitly allocate or release storage, update a count, return an error, or perform rollback. This does not constrain indirect effects of either dependency.

The signed additions used for neighbor-field and owner-field addresses must not be conflated with the established unsigned bucket construction. The supplied body does not justify a general claim that every address addition wraps modulo \(2^{32}\).

### Ownership and lifetime

No ownership transfer is established by the body. In particular, F-682cc3fccb47 cannot be characterized as destruction, release, or deallocation solely from its position and argument.

The supplied record remains a write target after that call. If a dependency invalidates the record’s storage, later clearing accesses cannot be assumed valid. Neither dependency is assumed to preserve globals, fields, or object lifetime.

### Errors and partial completion

There is no explicit error-handling path, dependency-result check, or transactional guarantee.

A failure or nonreturning dependency can prevent later operations. An invalid access after an earlier successful store can leave partially changed links or bucket state. Final clearing is not guaranteed after such an interruption.

## Dependencies

| Opaque ID | Invocation and position | Established relationship | Unresolved behavior |
|---|---|---|---|
| F-c08d1cf25b95 | No explicit arguments; after owner capture and before the active-key check. | Runs even when the subsequent active key is zero, provided execution reaches the call. | Global effects, synchronization, ownership, mutation, failure, and return behavior. |
| F-682cc3fccb47 | One derived integer/address-valued argument; after structural updates and before final clearing. | Runs only when the captured owner’s +0x14 byte is nonzero. | Argument interpretation, mutation, ownership, failure, and return behavior. |

No preservation or memory-management guarantees are imported from these dependencies.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless an example states otherwise, assume valid non-overlapping storage, supported address conversions, defined address calculations, normal dependency returns, and no relevant dependency mutation.

### Inactive record

If the active key is zero when checked after F-c08d1cf25b95:

- The unconditional dependency has executed.
- No explicit link or bucket write follows.
- F-682cc3fccb47 is not called.
- Existing nonzero link fields are not cleared.

### Middle record

For a nonzero active key and nonzero next and previous addresses:

- The successor’s +0x0c field receives the previous address.
- The predecessor’s +0x08 field receives the next address.
- No bucket entry is explicitly written.
- The optional-call decision follows those writes.
- Final clearing writes the active key, previous address, and next address to zero, in that order.

### Bucket-head removal

For active key 13, mask source 8, zero previous address, and a nonzero next address:

- The successor’s backward link receives zero.
- The mask is 7 and the index is 5.
- The scaled offset is 20.
- The destination integer is the unsigned table-base value plus 20, reduced modulo \(2^{32}\).
- The selected four-byte destination receives the next address.

### Single-record bucket and tail

For a nonzero active key with both links zero, no successor write occurs and the calculated bucket destination receives zero.

For a nonzero active key, zero next address, and nonzero previous address, no successor write occurs and the predecessor’s forward link receives zero. No bucket entry is explicitly written.

Both cases continue to the optional-call decision and final clearing.

### Zero mask source

For zero previous address, active key 3, and mask source zero:

- The mask becomes `0xffffffff`.
- The index is 3 and the scaled offset is 12.
- The destination integer is the unsigned table-base value plus 12 modulo \(2^{32}\).

Zero is not rejected as a mask source.

### Wrapping index multiplication

For zero previous address, active-key bits `0x40000001`, and mask source zero:

- The index is `0x40000001`.
- Multiplication by four wraps to `0x00000004`.
- With table-base bits `0x00001000`, the destination integer is `0x00001004`.

The store is conditional on conversion of that integer to a valid writable pointer. The function does not instead use the unbounded product.

### Wrapping table-base addition

For zero previous address, active key 4, mask source 8, and table-base bits `0xfffffff8`:

- The index is 4 and the scaled offset is `0x10`.
- Unsigned addition wraps to destination integer `0x00000008`.

If that integer converts to a valid writable pointer, that location receives the next-address value. Otherwise, normal completion is not established.

### Dependency mutations

If F-c08d1cf25b95 changes the active key from nonzero to zero, removal is skipped. A change from zero to nonzero can enable removal.

If it replaces the record’s owner-address field, later owner-relative accesses still use the address captured before the call.

If F-682cc3fccb47 writes nonzero active-key or link values while preserving record validity, the final clearing stores overwrite them with zero.

## Uncertainties

The following remain **documented_only**:

- The actual purpose, synchronization guarantees, and effects of either dependency.
- The meaning and ownership of the optional dependency’s argument.
- Caller guarantees concerning membership, link consistency, non-aliasing, and table capacity.
- Integer-to-pointer conversion for the calculated bucket destination.
- Pointer-to-integer conversion for the optional-call argument.
- Target-specific behavior of separate signed address calculations, including overflow.
- Concurrency rules and external synchronization.
- Dependency failure, nonreturn, and memory-lifetime effects.
- Resource lifecycle outside this function.

Bucket-mask subtraction, index multiplication by four, and table-base addition are not unresolved overflow questions: each uses unsigned 32-bit arithmetic with modulo-\(2^{32}\) reduction. Those established rules do not resolve pointer conversion or unrelated signed arithmetic.

## Evidence

The supplied static body establishes the field offsets, 32-bit field accesses, byte flag, branch conditions, dependency positions, structural writes, and final clearing order.

The unsigned operand in mask subtraction establishes an unsigned mask and index. That unsigned index makes scaling unsigned; the resulting unsigned value makes table-base addition unsigned as well. Both arithmetic stages therefore wrap modulo \(2^{32}\) before pointer conversion.

Removal from a bucketed doubly linked structure is inferred intent, not an independently verified type or lifecycle contract.

No runtime tests have been performed. The acceptance examples describe conditional static behavior, not observed executions.
