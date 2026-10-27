<!-- Trusted provenance: F-ffdded7ca165; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ffdded7ca165

## Purpose

F-ffdded7ca165 decreases one signed short-integer field in a globally addressed record region.

A nonnegative selector identifies a record by index. Every negative selector instead identifies record zero; it does not request an update of the entire region. Each successful invocation changes exactly one field.

The selected record depends on both the selector and the global address value used by the selected branch. Negative selectors use the initial global read. Nonnegative selectors use a subsequent read of the same global location.

The field’s application-level meaning is unknown. Its decrement does not establish reference-counting, expiration, resource release, or any other lifecycle behavior.

This is a static behavioral contract. It distinguishes accesses expressed in the supplied representation from guarantees about compiled execution.

## Interface

### Argument and result

The function accepts one signed integer selector by value and exposes no return value.

The selector’s concrete bit width is not established by the supplied material. Its sign determines the selection rule:

| Selector category | Selected record index | Operative base address |
|---|---:|---|
| Negative | 0 | Value from the initial global read |
| Zero | 0 | Value from the second global read |
| Positive | Selector value | Value from the second global read |

A negative selector’s magnitude has no effect on record selection. In particular, the most negative representable selector receives the same treatment as negative one; its magnitude is not negated or used as a record displacement.

There is no record-count argument, direct storage pointer, output parameter, ownership token, or synchronization argument. No status indicates success or failure.

### Global input and record layout

A global location holds an integer-valued base address. The function uses that value to locate record storage; the records are not stored in the global location itself.

The established layout is:

| Property | Value |
|---|---|
| Record stride | 20 bytes |
| Updated field’s offset within a record | 18 bytes |
| Field access type | Signed short integer |
| Concrete field width | Not established |
| Other record contents | Not described or inspected |

If the short-integer storage occupies two bytes, the field occupies offsets 18 and 19. That placement is conditional on the width assumption, not an independent width guarantee.

The stride and offset do not establish byte order, alignment, allocation extent, or the number of records.

### Preconditions for defined completion

A successful update requires:

- Readable global storage for every global access expressed on the selected branch.
- An operative base address supporting the selected field access.
- Valid intermediate address arithmetic and address conversion.
- Readable and writable storage for the complete field.
- Alignment appropriate to the field access.
- Storage lifetime extending through the update.
- Any external synchronization required by the execution environment.

No internal validation establishes these conditions.

For a nonnegative selector, the initial global value need not itself identify usable record storage: it is not used to access a record. The second value must support the update. For a negative selector, the initial value is operative and must support access to record zero.

## Behavior

### Record selection and global observations

The supplied representation expresses an initial read of the global base-address location before the selector’s sign is tested.

On the negative path, that captured value supplies the base for record zero. There is no second global read and no access to another record.

On the nonnegative path, the global location is read again. Only this second value contributes to the selected field address. The initial value is discarded for address selection, even if it differs from the second value.

For valid arithmetic, the selected field’s byte displacement from the operative base is:

- **Negative selector:** 18.
- **Nonnegative selector:** 20 times the selector, plus 18.

These are mathematical descriptions of the intended address, not permission to disregard intermediate arithmetic limits.

Negative selectors and zero therefore select the same record index but do not have identical global-read behavior. They select the same address only when their operative base values agree and the address calculations are valid.

### Access ordering

The representation establishes these shared-data access relationships:

| Path | Global reads before the field access | Field access relationship |
|---|---|---|
| Negative | One initial read | One field read precedes one write to the same address |
| Nonnegative | Initial read, then a second read | One field read precedes one write to the same address |

No additional global read intervenes between the selected field’s read and write. Once selected, the field address is the same for both accesses.

These are representation-level access counts and ordering relationships. The supplied material does not establish that compiled execution retains every expressed global read as a distinct physical load. It also does not establish volatile access, hardware transaction boundaries, or observation by another thread or device.

The possibility of different values from the two global reads is conditional on an execution model that permits those observations. It is not a guarantee that unsynchronized concurrent modification is valid.

### Field transformation

For an ordinary value whose decrement is representable under the applicable arithmetic and storage rules, the stored numerical value is exactly one less than the value read.

Examples of this transformation include:

| Value read | Value stored |
|---:|---:|
| 5 | 4 |
| 1 | 0 |
| 0 | −1 |
| −4 | −5 |

There is no special treatment for zero or negative field values. The function does not clamp, reject, skip, or notify on those values.

A successful invocation expresses exactly one field update, including for every negative selector. It does not traverse the record region, inspect a count or sentinel, or update all records.

Neither the old value nor the new value is returned.

### Integer widths and boundary arithmetic

Address calculation and field-value calculation have separate constraints.

**Address calculation.** The supplied representation uses signed integer quantities for the base and record displacement before accessing memory. Their concrete widths and the intended target’s integer-to-address interpretation are not established.

The indexed displacement is calculated in the represented integer arithmetic, not in an independently established unbounded or pointer-width domain. A nonnegative selector can therefore encounter an arithmetic limit before the field access. Intermediate multiplication and additions must be valid; a mathematically plausible final address alone does not establish validity.

No overflow detection, widened calculation, or checked conversion is visible. Modular address wraparound is not guaranteed.

**Field calculation.** The accessed storage is signed short-integer storage. Arithmetic promotion and conversion back to that storage affect boundary behavior.

At the minimum representable field value, the mathematical decrement is outside the field’s range. Depending on widths and target rules, the subtraction may fit a wider intermediate and then require an out-of-range conversion, or the arithmetic itself may exceed its signed range. The supplied material does not settle the result.

For example, a 16-bit field initially containing −32,768 becomes 32,767 only if separately established arithmetic and storage rules support that outcome. The function performs no explicit minimum-value check, but absence of a check is not evidence of guaranteed wraparound.

### Completion and failure

Under the required storage and arithmetic conditions, the function completes after the selected field write.

There is no explicit error-result path, bounds rejection, null-address rejection, permission check, or recovery action. Invalid access or arithmetic cannot be assigned a particular exception, signal, stored result, or normal-return behavior from this material.

The number of accesses completed before a fault or other invalid execution outcome is not generally specified.

## Side effects

The sole explicit data mutation is the write to the selected short-integer field.

No write is expressed to:

- The global base-address location.
- Another field of the selected record.
- Any other record.

This describes accessed storage locations, not guaranteed semantic independence between them. The supplied material does not establish that the selected field cannot overlap the global location or other application data.

The update is expressed as a read followed by a write. It is not established as an atomic decrement. No lock, memory barrier, synchronization protocol, or inter-thread visibility guarantee is present.

No allocation, deallocation, callback, logging, or external notification is visible.

The function neither establishes nor transfers ownership of the record region. The allocator, owner, release mechanism, and lifetime policy are unknown. Decrementing the field does not itself release any resource.

## Dependencies

No called-function dependency is visible. There are therefore no dependency function IDs to enumerate and no callee behavior on which to rely.

The function depends on:

- A readable global location containing the base-address value.
- Accessible selected-field storage.
- The target’s integer widths, promotions, overflow rules, and narrowing conversions.
- The target’s interpretation of integer-valued addresses.
- Field width, representation, byte order, and alignment.
- Externally maintained storage lifetime and synchronization.

No unseen helper is assumed to validate indices, preserve globals, maintain ownership, or handle arithmetic boundaries.

## Acceptance examples

These are conditional acceptance criteria, not executed tests. Unless a case states otherwise, assume valid arithmetic, suitable alignment, accessible storage, and a representable decrement.

### Zero selector with a stable base

Both global reads yield the same base, and record zero’s selected field contains seven.

The required represented behavior is two global reads followed by an update of the field at offset 18 from the second value. That field becomes six. No other field is explicitly accessed, and no status is returned.

### Positive selector

The selector is two, the operative base identifies valid storage, and record two’s selected field contains twelve.

The updated field is 58 bytes from the second base value: two record strides plus the field offset. Its value becomes eleven. Records zero and one are not visited.

### Negative selector

The selector is negative one. Record zero’s selected field contains three, and record one’s corresponding field contains nine.

Only one global read is expressed. Record zero’s field becomes two. Record one is not accessed by the function. Any other representable negative selector has the same selection behavior.

### Different global values

Under an execution model permitting distinct observations, the initial global read yields one region’s base and the second read yields another region’s base.

For selector zero, only the first-record field in the second region is selected. The first value does not cause a record access.

A negative-selector invocation instead uses its initial value and performs no second global read. Thus equal selected indices do not imply equal selected addresses.

This case does not require distinct hardware loads or authorize a data race.

### Unusable discarded value

A nonnegative invocation obtains an unusable record address from its initial global read and a valid address from its second read.

If both global reads and all subsequent operations are valid, the unusable first value does not by itself invalidate a record access: it is not used for one.

The same reasoning does not apply to a negative invocation, whose initial value supplies the selected address.

### Nonpositive field values

A selected field containing zero becomes negative one. A selected field containing negative four becomes negative five.

Neither value causes a separate branch, rejection, or release action.

### Minimum field value

Assume a signed 16-bit field initially contains −32,768.

The function has no explicit boundary guard. The exact resulting value or execution outcome remains **documented_only** until the relevant promotion, arithmetic, and storage-conversion rules are established. A result of 32,767 must not be required solely from the supplied representation.

### Inaccessible indexed storage

A nonnegative selector identifies storage outside the accessible region.

The function provides no explicit bounds error or clean failure return. The resulting execution outcome is not specified by this contract.

## Uncertainties

The following remain **documented_only**:

- Concrete selector, base-address, and intermediate arithmetic widths.
- Concrete short-integer width and representation.
- Boundary subtraction and out-of-range storage behavior.
- Integer-to-address conversion and address-overflow behavior.
- Byte order and alignment requirements.
- Record count, allocation extent, initialization, lifetime, and ownership.
- Whether selected storage can overlap the global address location.
- Whether distinct global values can be observed in a defined execution.
- Physical load counts, compiler transformations, and hardware ordering.
- Concurrency semantics and invalid-access outcomes.
- The field’s application-level meaning.
- The intended significance of negative selectors.

A homogeneous record region is a plausible interpretation of the regular stride and fixed field offset. It does not establish the region’s extent or the meaning of any record field.

The negative path’s single-record behavior is established. Whether that behavior reflects the intended application contract remains unknown.

## Evidence

**Established static evidence:** The supplied representation accepts a signed integer selector, expresses a global address read before its sign test, and uses either that initial value or a second global read according to the selector category. Record selection uses a 20-byte stride and an 18-byte field offset. Each successful path reads and decrements one signed short-integer field. No calls, validation, ownership operation, or error-result mechanism are visible.

**Inferred intent:** The layout suggests repeated records, and the negative-selector branch suggests a special selection convention. Neither inference establishes traversal semantics, field meaning, or resource-management behavior.

**Unknown execution properties:** Concrete widths, boundary outcomes, physical memory transactions, address validity, and concurrency guarantees require evidence beyond the supplied representation.

**Verification status:** Static analysis only. No runtime tests, memory traces, or target-execution checks have been performed.
