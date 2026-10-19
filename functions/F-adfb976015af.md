<!-- Trusted provenance: F-adfb976015af; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-adfb976015af

## Purpose

F-adfb976015af conditionally stores a pair of 32-bit values in a caller-provided memory region. A signed control value determines both whether the stores occur and the Boolean result:

- A negative control value causes no destination access and returns true.
- A nonnegative control value causes two ordered stores and returns false after those stores complete.

The return value therefore reports whether the control value was negative. It does not independently report whether a destination is valid or whether a memory write succeeded.

The apparent intent is to update two related fields only when the control value is nonnegative. The meanings of those fields, and whether a negative value represents an error, sentinel, or ordinary state, are not established.

## Interface

The interface has four positional inputs and a Boolean output.

| Input | Source-visible representation | Behavioral role |
|---|---|---|
| First | Integer carrying a destination base address | Locates the memory region used by the nonnegative-control path. |
| Second | 32-bit value | Unused by the supplied body. |
| Third | 32-bit value with no established signedness or semantic type | Supplies the bit pattern written to the first destination field. |
| Fourth | Signed integer, treated as 32 bits for this contract | Controls the branch and supplies the value written to the second destination field. |

The source represents the address-bearing input as an integer rather than a typed object pointer. That representation does not establish an object type, allocation size, ownership rule, or permission to access any particular address.

The two destination fields have the following layout relative to the first input:

| Byte offset | Width | Stored content |
|---|---:|---|
| 16 through 19 | 4 bytes | Third input’s 32-bit value |
| 20 through 23 | 4 bytes | Fourth input’s signed 32-bit value |

The fields are adjacent and do not overlap. Their byte order follows the target’s native representation; the supplied material does not independently establish endianness.

The result is a logical Boolean. Its calling-convention representation, including return-register width and padding bits, is not specified by the supplied body.

For a nonnegative fourth input, normal completion requires both destination fields to be writable and their addresses to be valid under the execution environment’s addressing and alignment rules. For a negative fourth input, the body imposes no destination-access requirement because it does not derive or access either destination field on that path.

## Behavior

### Negative control value

The fourth input is interpreted using signed comparison. If it is less than zero:

1. Neither destination store is performed.
2. No destination contents are read.
3. The result is true.

Every negative value takes this path. The behavior is not limited to a distinguished value of negative one. Under the 32-bit signed interpretation, the inclusive negative range is −2,147,483,648 through −1.

On this path, neither the first input nor the third input affects the result. The second input has no effect on either path.

An address that would be unsuitable for writing does not, by itself, prevent this negative-control path from completing: the address is not dereferenced by the supplied body. This is a statement about the body’s accesses, not a guarantee about argument preparation, instrumentation, or other behavior outside it.

### Nonnegative control value

If the fourth input is zero or greater, the body performs two full-width stores in this order:

1. Store the third input’s 32-bit value at destination offset 16.
2. Store the fourth input’s signed 32-bit value at destination offset 20.
3. Return false.

Zero belongs to this write-producing path. There is no stricter positive-only requirement and no upper-bound check within the nonnegative range.

The first store preserves the third input’s 32-bit payload without any visible arithmetic, masking, normalization, or validation. Its interpretation as an integer, address, flags, or another representation is not established.

The second store preserves the fourth input’s value. There is no visible increment, decrement, scaling, narrowing, or conversion to a Boolean before storage.

Under ordinary valid writable-memory conditions, the body does not modify bytes outside offsets 16 through 23. In particular, it does not initialize or inspect the preceding 16 bytes of the destination region.

### Ordering and arithmetic

The supplied body places the offset-16 store before the offset-20 store. A behavioral reproduction should retain this operation order. The evidence does not establish atomic pair publication, synchronization, or a guarantee about what a concurrent observer can see.

The control decision is a signed comparison against zero. Under the 32-bit interpretation, 2,147,483,647 takes the nonnegative path, while the bit pattern representing −2,147,483,648 takes the negative path. An unsigned comparison would not preserve this behavior.

Address calculation uses fixed byte offsets of 16 and 20. The body contains no visible checks for address overflow, alignment, allocation boundaries, or accessibility. The supplied material does not resolve how an implementation should treat an address-bearing integer near the limits of its representation. Normal write-path acceptance examples therefore require valid effective addresses rather than assuming wrapping arithmetic.

### Failure behavior

There is no explicit error return for an invalid destination, and no visible recovery or rollback.

The false result describes normal completion of the nonnegative path. It must not be treated as a promise that invalid memory accesses are caught and converted into a Boolean result.

If the first store succeeds but the second cannot complete, the body supplies no restoration of the first field. The precise fault mechanism and resulting process state depend on the execution environment and are not established here.

## Side effects

For a normally completing nonnegative-control invocation, the only explicit side effects are the two destination stores.

The body does not:

- Read the existing contents of either destination field.
- Read memory through the second or third input.
- Allocate or release memory.
- Invoke a dependency.
- Explicitly access a named global state location.
- Perform an explicit bounds, alignment, or permission check.

There is no ownership transfer established by passing the destination address or the third input. The caller remains responsible for supplying suitable storage when the nonnegative path is taken. The function does not establish the storage’s lifetime or release it.

These are direct-body properties. They do not imply that the destination cannot alias other externally meaningful state. For example, caller-selected destination memory could overlap shared state; the function provides no protection against such aliasing.

## Dependencies

No called functions are present in the supplied body. Accordingly, there are no dependency function IDs to list and no callee behavior on which the branch decision or stored values depend.

The function relies on ordinary execution-environment facilities: signed comparison, address calculation, 32-bit memory stores, and Boolean return. These requirements do not establish additional validation, memory ownership, or fault-handling services.

## Acceptance examples

The following examples are conditional behavioral expectations derived from the supplied body. They are not results of runtime tests. Write-path examples assume a valid destination with at least the stated writable fields.

| Conditions | Expected stores | Expected result |
|---|---|---|
| Fourth input is −1 | None | True |
| Fourth input is −2 | None | True |
| Fourth input is −2,147,483,648 | None | True |
| Fourth input is 0 | Third input at offset 16, then 0 at offset 20 | False |
| Fourth input is 1 | Third input at offset 16, then 1 at offset 20 | False |
| Fourth input is 2,147,483,647 | Third input at offset 16, then 2,147,483,647 at offset 20 | False |

Additional boundary and state expectations:

- **Unused argument:** Changing only the second input does not change the branch, stored values, or Boolean result.
- **Payload preservation:** With a nonnegative fourth input and a third-input bit pattern of `0xFFFFFFFF`, the offset-16 field receives that complete 32-bit pattern. No signed interpretation of that payload is required.
- **Negative-path preservation:** With an ordinary valid destination and any negative fourth input, its existing field values remain unchanged by this invocation.
- **Repeated updates:** Two normally completing nonnegative invocations using the same destination leave the two fields containing the later invocation’s payload and control value.
- **Negative invocation after an update:** A subsequent invocation with a negative fourth input leaves the previously stored fields unchanged and returns true.
- **Unsuitable address on the negative path:** If execution reaches the body with a negative fourth input, it performs no destination access even if the first input would not identify writable storage.
- **Unsuitable address on the write path:** No particular Boolean result or completed destination state is guaranteed if either required store cannot execute.

## Uncertainties

- **documented_only — Field meanings:** No domain meaning is established for either stored field.
- **documented_only — Object layout:** Only the two four-byte fields and their offsets are established; the enclosing object’s type and total size are unknown.
- **documented_only — ABI details:** Integer and address conventions are not independently supplied. This contract uses the apparent 32-bit integer interpretation while leaving pointer width and Boolean return encoding unspecified.
- **documented_only — Byte order and alignment:** Native byte order and target-specific alignment requirements are not independently established.
- **documented_only — Exceptional execution:** Fault delivery, invalid-address behavior, and integer-address overflow semantics are unspecified.
- **documented_only — Concurrency:** No atomicity, memory-ordering, or concurrent-access contract is established.
- **Inferred intent:** The two stores appear to form a conditional paired-field update. Their adjacency and shared branch do not prove transactional behavior.

## Evidence

Established static evidence consists of a signed nonnegative branch, two four-byte destination stores at offsets 16 and 20 inside that branch, and a Boolean return that is true exactly when the fourth input is negative.

The supplied body contains no use of the second input, no destination-content reads, no calls, and no explicit allocation, release, validation, or recovery operations.

No runtime tests have been performed. Acceptance examples express source-derived expectations, not observed execution results.
