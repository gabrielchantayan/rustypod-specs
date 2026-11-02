<!-- Trusted provenance: F-6a2a5ddf37ed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6a2a5ddf37ed

## Purpose

F-6a2a5ddf37ed conditionally invokes a storage-management dependency when a requested bit count, converted to bytes, exceeds the byte count derived from a stored capacity field.

The established behavior is narrower than “ensure capacity.” The function compares converted byte counts, rounds a growth target to a multiple of eight bytes, calls F-6dd003795db7, stores that dependency’s returned 32-bit value, and conditionally updates the capacity field according to a separately reported status.

Interpreting the record as a resizable bit-oriented buffer is **inferred intent**. The visible operations do not establish that the dependency allocates memory, preserves existing contents, or returns a usable address.

The function returns a 64-bit composite value. Its low half is a status value. Its high half depends on whether the growth branch executes; it is not uniformly a pointer, count, or status.

## Interface

### Inputs

The function receives three arguments:

1. **Record address:** a 32-bit address identifying a writable record with relevant 32-bit fields at byte offsets 4 and 8.
2. **Requested count:** a 32-bit value used as a bit count for the capacity comparison. Its original bit pattern also supplies the high half of the return value when growth is skipped.
3. **Dependency context:** an opaque 32-bit value passed unchanged to F-6dd003795db7 when growth occurs.

Although the first two arguments are represented as signed 32-bit values in the supplied material, the count-conversion arithmetic uses unsigned 32-bit semantics. Negative-looking input bit patterns are therefore not rejected as negative counts.

### Relevant record layout

| Byte offset | Width | Established use |
|---|---:|---|
| 4 | 32 bits | Read as the stored count; overwritten with a rounded bit capacity after a zero dependency status. |
| 8 | 32 bits | Read only on the growth branch; supplied to the dependency and replaced by its returned 32-bit value. |

The function does not directly access the field at offset 0. No other record layout is established.

The field at offset 8 is plausibly a storage pointer or handle, but its actual type and ownership are unknown.

### Output

The result is 64 bits, composed as follows:

- **Low 32 bits:** the local status, initialized to zero and exposed to the dependency through an address.
- **High 32 bits:**
  - When growth is skipped, the original requested-count bit pattern.
  - When growth occurs, the value read from record offset 8 before the dependency call.

The dependency’s direct return value is stored into the record, not placed directly into either half of the function’s return value.

### Preconditions

For ordinary defined execution, the record address must permit the required 32-bit accesses. Offset 4 must be readable on every invocation. When growth occurs, offset 8 must be readable and writable; offset 4 must also be writable if the dependency reports zero status.

The third argument and existing storage value must satisfy whatever requirements F-6dd003795db7 imposes. Those requirements are not supplied.

There is no visible validation of null addresses, alignment, record extent, requested-count range, or dependency-context validity.

## Behavior

### Count conversion and comparison

The function first reads the 32-bit stored count at record offset 4.

It independently converts that stored count and the requested count to byte counts. Each conversion:

- Adds 7 using unsigned 32-bit arithmetic.
- Discards overflow beyond 32 bits.
- Performs a logical right shift by 3.

For inputs from 0 through 4,294,967,288, this is the usual ceiling conversion from bits to bytes. For the seven larger input bit patterns, the addition wraps and the resulting byte count is zero.

Both converted byte counts lie between 0 and 536,870,911 inclusive. Their comparison is unsigned.

The local status begins at zero. The dependency is called only when the converted stored byte count is strictly less than the converted requested byte count.

Consequently:

- Equal converted byte counts do not trigger growth.
- A larger requested bit count can still skip growth if both counts convert to the same byte count.
- The stored count is not normalized or rounded when growth is skipped.
- There is no shrink operation.

### No-growth branch

If the converted stored byte count is at least the converted requested byte count:

- Record offset 8 is not read by this function.
- F-6dd003795db7 is not called.
- Neither record field is written by this function.
- The returned low half is zero.
- The returned high half is the original requested-count bit pattern.

A zero low half in this branch means that the initialized status remained zero. It does not demonstrate that any storage-management operation succeeded, because none was attempted.

### Growth target

If growth is required, the function reads the existing 32-bit value at record offset 8.

It rounds the converted requested byte count upward to a multiple of eight bytes by adding 7 and clearing the lowest three bits. This rounding addition does not overflow: the converted byte count is at most 536,870,911, so the largest intermediate sum is 536,870,918.

A growth target is therefore an eight-byte multiple between 8 and 536,870,912 inclusive. This corresponds to a nominal granularity of 64 bits.

The comparison uses the unrounded converted requested byte count. The dependency receives the rounded target.

### Dependency call

F-6dd003795db7 receives six arguments in this order:

1. The opaque third input.
2. The constant value 1.
3. The converted stored byte count.
4. The rounded target byte count.
5. The value read from record offset 8.
6. The address of the local 32-bit status.

The visible function does not pass the record address explicitly. That does not establish that the dependency cannot access the record through another alias or external state.

The local status is zero when its address is supplied. If the dependency does not change it, it remains zero.

### Post-call updates

After the dependency returns normally, the function stores its direct 32-bit return value at record offset 8. This write occurs regardless of the status.

Only after this write does the function inspect the local status:

- If status is zero, it writes the rounded target byte count shifted left by 3 into record offset 4.
- If status is nonzero, it performs no post-call write to offset 4.

The capacity update retains only 32 bits. For the maximum rounded target of 536,870,912 bytes, shifting left by 3 produces zero in the stored 32-bit field.

The unconditional offset-8 write is significant on failure: a nonzero status does not preserve the old storage value. There is no visible rollback or cleanup call.

On the growth branch, the high return half remains the pre-call value read from offset 8. The low half is the final local status.

### Errors and abnormal execution

The only explicit error-like mechanism is the 32-bit status exposed to the dependency. Zero selects the capacity update; every nonzero bit pattern suppresses that update.

No meaning is established for individual nonzero values. They are returned without visible translation.

Invalid memory access, dependency failure to return, and other abnormal execution have no visible local handling. A normal composite return is not guaranteed in those cases.

## Side effects

The direct record-access sequence is:

1. Read offset 4.
2. On growth only, read offset 8.
3. Invoke F-6dd003795db7.
4. After normal dependency return, write its returned value to offset 8.
5. Read the local status.
6. If status is zero, write the computed capacity to offset 4.

This is the observable ordering represented by the supplied operations, not a claim about machine-instruction scheduling, memory barriers, or inter-thread visibility.

No direct record writes occur on the no-growth branch.

A nonzero dependency status leaves offset 4 unwritten by this function after the call. It does **not** prove that offset 4 is unchanged overall: the dependency may have access to aliased state, and its implementation is unknown.

The function neither directly allocates nor directly releases storage. It delegates whatever storage-related activity may occur to F-6dd003795db7. There is no established ownership transfer for the old value, the returned value, or memory that either might identify.

The dependency receives an address to invocation-local status storage. The visible function provides no post-return lifetime guarantee for that address. Whether the dependency retains it is unknown.

No synchronization, atomicity, or concurrent-access protocol is established.

## Dependencies

### F-6dd003795db7

**Established interface use:** called only on growth, with six arguments consisting of a context value, constant 1, old byte count, rounded target byte count, old storage value, and status address.

**Established result handling:** its direct 32-bit return value always replaces record offset 8 after normal return. The status reachable through its final argument determines whether offset 4 is updated and supplies the low return half.

**Inferred intent:** the size arguments and old storage value suggest an allocation or resizing service.

**Unknown contract:** allocation strategy, copying behavior, initialization of newly available storage, release of old storage, failure ownership, valid context values, meaning of constant 1, status meanings, global-state changes, and aliasing effects.

No other explicit dependency call appears in the supplied function.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless stated otherwise, assume valid record accesses, normal dependency return, and no dependency-induced changes to the record other than the specified status behavior.

1. **Same byte count despite a larger request.**  
   Stored count is 1 bit; requested count is 8 bits. Both convert to 1 byte. There is no dependency call or direct record write. The returned high half is 8 and the low half is zero.

2. **Growth to the minimum rounded target.**  
   Stored count is 8 bits; requested count is 9 bits. The converted counts are 1 and 2 bytes. The dependency receives old size 1 and target size 8. If it reports zero status, offset 4 becomes 64 bits, and offset 8 receives its direct return value.

3. **Failure still replaces the storage field.**  
   For the same 8-bit-to-9-bit request, suppose the old storage value is 0x12345678, the dependency returns zero, and it writes status 5. Offset 8 becomes zero. This function does not write offset 4 after the call. The composite return is 0x1234567800000005.

4. **No growth does not require reading storage.**  
   Stored count is 64 bits; requested count is 63 bits. Both convert to 8 bytes. Offset 8 is not accessed by this function. The composite return is 0x0000003F00000000.

5. **A smaller stored bit count need not cause growth.**  
   Stored count is 9 bits; requested count is 16 bits. Both convert to 2 bytes. Growth is skipped, and the stored count remains 9.

6. **Wrapped requested-count conversion.**  
   Requested count has bit pattern 0xFFFFFFFF. Adding 7 wraps to 6, which shifts to zero bytes. No possible converted stored byte count is less than zero, so growth is skipped. The composite return is 0xFFFFFFFF00000000.

7. **Maximum rounded target wraps the capacity write.**  
   Stored count is zero; requested count is 0xFFFFFFF8. The request converts to 536,870,911 bytes and rounds to 536,870,912 bytes. If the dependency returns normally with zero status, the capacity field is written as zero because the bit conversion overflows 32 bits.

8. **An untouched status selects the update.**  
   On any growth case, if the dependency leaves the supplied status storage unchanged, the status remains zero. The function updates offset 4 without independently validating the returned storage value.

## Uncertainties

The following are **documented_only**:

- The record’s public type, total size, and other fields.
- Whether offset 4 denotes capacity, length, or another bit-oriented quantity.
- Whether offset 8 is an address, handle, or other 32-bit token.
- The intended interpretation of the branch-dependent high return half.
- The dependency’s ownership, memory-preservation, and error contracts.
- Whether unusually large count bit patterns are valid caller inputs.
- Whether callers rely on or prevent the documented arithmetic wraparound.
- Dependency access to globals, aliases, or retained argument addresses.
- Alignment requirements, byte order, and concurrent-use guarantees.
- Whether external calling conventions assign additional meaning to the 64-bit composite result.

The apparent storage-growth purpose does not justify assuming any of these contracts.

## Evidence

**Established static evidence:** the supplied function shows two wrapping bit-to-byte conversions, an unsigned comparison, a conditional read of the old storage value, eight-byte target rounding, one dependency call, an unconditional post-call storage-field write, a status-conditional capacity write, and a 64-bit concatenated return.

**Inferred intent:** bit-capacity management through a storage-resizing dependency. This interpretation explains the sizes and rounding but is not required to describe the visible behavior.

**Unknowns:** the dependency implementation, callers, broader record definition, and execution environment were not supplied.

No runtime tests have been performed. The acceptance examples are arithmetic and control-flow consequences of the supplied static evidence, not observed execution results.
