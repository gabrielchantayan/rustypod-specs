<!-- Trusted provenance: F-ca7cd7c393a5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ca7cd7c393a5

## Purpose

F-ca7cd7c393a5 answers whether a requested indexed entry passes two checks: the index is below a bound supplied by a dependency, and the entry’s leading byte does not equal the exclusion marker.

The result is a four-byte value containing exactly 0 or 1. A result of 1 means both checks passed. A result of 0 means either the index was outside the reported bound or the entry’s leading byte matched the marker.

**Established static evidence:** The function calls one dependency, performs an unsigned bounds comparison, and conditionally examines one byte of an entry.

**Inferred intent:** The object appears to contain a descriptor for a collection of 16-byte entries. The leading byte appears to indicate whether an entry is eligible or present. Those semantic labels are explanatory only; the supplied evidence does not establish a collection type, marker name, or lifecycle.

## Interface

### Inputs

The function accepts two inputs:

- **Object base:** A recovered 32-bit signed integer used as an address. The function derives the address 12 bytes beyond this base.
- **Index:** A 32-bit unsigned integer selecting an entry.

The address at object-base offset 12 has two distinct roles:

1. Its address is passed to F-b75d62a4795b.
2. After that dependency returns, and only if the bounds check succeeds, its contents are read as a 32-bit entry-storage address.

The first role does not imply that the caller loads the contents before calling the dependency. Passing the field’s address and passing the stored entry-storage address are observably different operations.

### Output

The return value is four bytes wide:

- **1:** The unsigned index is below the dependency’s returned bound, and the selected entry’s leading byte differs from the exclusion marker.
- **0:** At least one of those conditions is false.

No other ordinary return value is present in the supplied body. No separate error value, diagnostic output, or status object is established.

### Relevant layout

Only the following layout facts are supported:

| Location | Interpretation used by this function |
|---|---|
| Object base plus 12 bytes | Address supplied to F-b75d62a4795b |
| Four-byte field at object base plus 12 bytes | Entry-storage address, loaded on the successful bounds branch |
| Entry-storage address plus 16 times the index | Address of the selected entry’s leading byte |

The function does not inspect the remaining 15 bytes of an entry. Their meaning and required contents are unknown.

### Preconditions

For ordinary completion:

- F-b75d62a4795b must accept the derived address and return normally.
- If the returned bound exceeds the index, the four-byte storage-address field must be readable after the dependency returns.
- On that same branch, the computed entry-byte address must be readable.
- Address calculations and accesses must have valid behavior under the actual target’s execution model.

A failing bounds check avoids the caller’s storage-address and entry-byte reads. It does not establish that an invalid object base is safe: the dependency is called first and receives an address derived from that base.

## Behavior

### Dependency invocation and bound

The function first derives the address at object-base offset 12 and calls F-b75d62a4795b with that address as its sole visible argument.

The dependency’s returned value is held as a 32-bit unsigned bound. The function calls this dependency once per invocation, including when the index is very large or will ultimately fail the bounds check.

The dependency’s internal behavior is not supplied. In particular, the word “bound” describes how its result is used here, not a proven promise that it returns an actual allocation size or entry count.

### Unsigned bounds decision

The index is compared with the returned bound using unsigned ordering.

If the index is greater than or equal to the bound, the function returns 0. On that path, the supplied caller body does not load the entry-storage address and does not read an entry byte.

Consequences include:

- A bound of zero rejects every possible index.
- An index equal to the bound is rejected.
- The largest 32-bit unsigned index cannot pass this comparison, because no 32-bit unsigned bound is greater than it.
- A value interpreted elsewhere as negative is not treated as a negative index here. If supplied through its 32-bit unsigned representation, it participates as a large unsigned value.

There is no second bound calculation or revalidation before the selected byte is examined.

### Conditional storage lookup

When the index is below the bound, the function loads the four-byte entry-storage address from object-base offset 12.

This load occurs **after** F-b75d62a4795b returns. The caller does not visibly cache this field before the call. Consequently, if the dependency changes the field and returns normally, the subsequent lookup uses the field’s post-call contents.

The selected byte is located at the loaded storage address plus an offset of 16 times the index. The function then reads that one byte.

The 16-byte spacing is established; the existence of a complete readable 16-byte record at every selected position is not. This function itself reads only the first byte.

### Marker decision

The selected byte is interpreted through a character load and compared against negative one:

- If it compares equal to negative one, the function returns 0.
- If it compares unequal to negative one, the function returns 1.

Under a signed eight-bit character interpretation, the excluded byte pattern is **0xFF**. All other byte patterns pass this second check.

The recovered character type does not independently establish the target’s character signedness. Therefore, identifying 0xFF as the exclusion marker is conditional on the signed eight-bit interpretation. The negative-one comparison is the directly supplied evidence.

### Width and arithmetic boundaries

The recovered interface and accesses use 32-bit quantities for the object-base integer, unsigned index, returned bound, storage-address load, and result.

The index multiplication uses unsigned 32-bit arithmetic under the recovered types. Its offset therefore wraps modulo 2³² if the mathematical product exceeds the representable range. No explicit overflow check appears.

For example, an index of 2²⁸ produces a mathematical byte offset of 2³²; its unsigned 32-bit offset is zero. If a returned bound permits that index, the bounds check alone does not prevent offset wraparound.

The object-base adjustment and final address formation also have no explicit overflow checks. Exact overflow and address-validity behavior depends on the target execution model; the evidence does not establish a portable rule for every such case.

## Side effects

### Direct effects

The supplied caller body contains no explicit memory writes.

Its visible actions are:

1. Call F-b75d62a4795b with the derived field address.
2. Use the returned unsigned bound.
3. Conditionally read the four-byte storage-address field.
4. Conditionally read one byte from entry storage.
5. Return 0 or 1.

These are caller-level ordering facts. They do not describe machine speculation, hardware bus transactions, or reads performed inside the dependency.

### Indirect effects

F-b75d62a4795b may have side effects that are not visible in the supplied evidence. This contract does not assume that it preserves:

- The storage-address field.
- Other object state.
- Global state.
- Allocation lifetime.
- External resources.

If the dependency invalidates storage that the later branch needs, ordinary completion is not established.

### Ownership and lifetime

The function body does not explicitly allocate, release, retain, or transfer ownership of memory. It does not return a pointer or expose an entry object.

No ownership conclusion can be drawn for the dependency. Callers must satisfy whatever lifetime requirements that dependency imposes, as well as the readability requirements of the conditional lookup.

### Failure handling

There is no visible recovery path for an invalid address or an abnormal dependency outcome. A memory-access failure or dependency failure is not converted into a defined return value by the supplied body.

A normal result of 0 is therefore a failed eligibility check, not a general-purpose indication that all input or memory errors were handled.

## Dependencies

### F-b75d62a4795b

**Invocation:** Exactly one visible call, before the bounds decision.

**Argument:** The address 12 bytes beyond the object base.

**Consumed result:** A 32-bit unsigned value used as an exclusive upper bound.

**Unknown contract:** The supplied material does not establish its own input validation, reads, writes, ownership effects, failure behavior, synchronization, or relationship between its return value and the size of entry storage.

No other explicit function dependency appears.

## Acceptance examples

The following are conditional acceptance cases derived from static evidence, not executed tests. Except where stated otherwise, they assume normal dependency completion, valid address calculations, readable accessed storage, and signed eight-bit interpretation of the entry byte.

| Conditions | Required result and access behavior |
|---|---|
| Dependency returns 0; index is 0 | Return 0. The caller does not load the storage-address field or entry byte. |
| Dependency returns 3; index is 3 | Return 0 because equality fails the exclusive bound. No caller entry lookup occurs. |
| Dependency returns 3; index is 4 | Return 0. The dependency is still called. |
| Dependency returns 3; index is 2; byte at storage plus 32 is 0x00 | Return 1. A zero byte is not rejected by this marker check. |
| Dependency returns 3; index is 2; byte at storage plus 32 is 0xFF | Return 0. |
| Dependency returns 3; index is 2; byte at storage plus 32 is 0x80 | Return 1. A negative signed character other than negative one still passes. |
| Index is the maximum 32-bit unsigned value | Return 0 after the dependency returns, regardless of its 32-bit bound. |

Additional ordering and boundary cases:

- **Dependency replaces storage:** If the dependency changes the storage-address field and returns a bound greater than the index, the caller reads the replacement address. The outcome depends on the byte reached through that replacement.
- **Rejected index with unusable storage:** If the dependency returns a bound no greater than the index, the caller does not dereference the stored entry address. This is conditional on the dependency itself completing normally.
- **Offset wraparound:** If the bound permits index 2²⁸, the recovered unsigned multiplication produces a zero offset. Subject to valid target address behavior, the examined byte is at the storage base, not at an unbounded mathematical offset of 2³² bytes.

## Uncertainties

The following remain **documented_only**:

- The semantic type of the object and the field at offset 12.
- Whether the dependency returns a count, capacity, or another limit.
- The target’s character signedness and confirmation that 0xFF is the intended exclusion marker.
- The meaning of the leading byte beyond this comparison.
- The actual size and lifetime of the entry allocation.
- Exact target behavior for overflowing address calculations.
- Dependency mutations, resource effects, and failure behavior.
- Concurrency guarantees and behavior under concurrent mutation.

The function does not visibly create a synchronized snapshot. Its dependency result, storage-address field, and selected byte are obtained at distinct stages. No atomicity or consistency guarantee across those stages is established.

## Evidence

This contract is based solely on the supplied static function body and opaque dependency mapping.

Established evidence includes the single dependency call, offset-12 argument address, unsigned strict bounds comparison, conditional four-byte address load, 16-byte entry spacing, one-byte marker check, and four-byte 0-or-1 return.

Collection terminology is inferred intent, not an additional behavioral promise. Dependency internals, target signedness details, memory validity, and synchronization remain unknown where noted.

No implementation execution, runtime tests, or observed acceptance results have been performed.
