<!-- Trusted provenance: F-5dbde81efa3b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5dbde81efa3b

## Purpose

F-5dbde81efa3b returns the directed difference between two adjacent signed integer values, reduced by a four-bit right shift. The first element supplies the starting quantity; the second supplies the ending quantity.

For a nonnegative, representable difference, the result is the number of complete groups of 16 in that difference. The operation is not an absolute distance, and it does not reduce the two endpoints separately.

**Established static evidence:** subtraction precedes the right shift. The resulting signed integer is returned directly, without an explicit validation or error-reporting path.

**Inferred intent:** the function may express a difference in units of 16. The application meaning of the values and the significance of that unit are not established.

This is a static behavioral contract. No runtime tests have been performed.

## Interface

### Input

The function accepts one pointer to signed integer storage and accesses two consecutive elements:

| Storage position | Arithmetic role |
|---|---|
| Element at the supplied address | Starting quantity |
| Immediately following element | Ending quantity |

There is no length argument, output buffer, configuration argument, or error-reporting parameter.

For supported ordinary-storage behavior, both accesses must be valid. The storage must have suitable alignment, contain readable initialized values, remain accessible during the required reads, and satisfy the applicable object-access and concurrency rules. An array containing at least two elements of the required type can satisfy the layout requirement.

A null pointer, insufficient accessible storage, or an address that does not permit the second-element access violates these preconditions. Numerical adjacency of unrelated objects alone does not establish that accessing them as consecutive elements is valid.

The mathematical difference between the ending and starting quantities must also be representable in the signed arithmetic type before the shift occurs.

### Width and layout

The two input elements, the subtraction result, and the return value have the same nominal signed integer type. No explicit widening conversion or wider intermediate calculation is visible.

The supplied evidence does not establish:

- Exact integer width or representable range.
- Element size in bytes.
- Required alignment.
- Byte order.
- Signed integer representation.

The second element begins one element-storage size after the first. An unconditional byte offset cannot be specified.

**Conditional layout example:** if the applicable ABI uses signed 32-bit integers occupying four eight-bit bytes, the two elements occupy eight bytes, and the second begins four bytes after the first. Under that condition, the return value is also signed 32-bit. The shift count does not establish this ABI or require 16-byte alignment.

### Output and ownership

The function returns one signed integer directly. There is no separate success flag, and no numerical return value is designated as an error indicator.

**Ownership is unknown.** The supplied function does not establish who owns the storage. The caller may own it or may merely have permission to access it.

No allocation, release, ownership-transfer operation, or retention of the supplied pointer is visible. Valid access during the required reads is necessary regardless of ownership. The function introduces no visible requirement that the storage remain available after those accesses have completed.

## Behavior

### Input observations

Both adjacent elements contribute to the calculation. Their positions establish their arithmetic roles, not their temporal read order.

The supplied representation does not establish:

- Which element is read first.
- An atomic snapshot of the pair.
- A particular physical load count or load width.
- Volatile or device-access semantics.
- Synchronization with another execution context.

Stable ordinary storage is therefore the clearest supported case. Concurrent modification must independently satisfy the applicable memory model. The function provides no visible synchronization that would make an otherwise invalid concurrent access valid.

### Directed difference and reduction

The function forms the ending quantity minus the starting quantity. It then applies a right shift of four bit positions to that completed difference and returns the result.

For a nonnegative, representable difference, the result is the whole-number quotient by 16:

| Difference | Return value |
|---:|---:|
| 0–15 | 0 |
| 16–31 | 1 |
| 32–47 | 2 |
| 48–63 | 3 |

Neither endpoint nor the difference must be divisible by 16. A remainder does not cause rejection, upward rounding, or a distinct error result.

Equal values produce zero, including equal negative values. Negative endpoint values do not necessarily produce a negative result: an ending quantity of −4 and a starting quantity of −20 produce a positive difference of 16.

A zero return does not prove that the endpoints are equal. Every nonnegative difference smaller than 16 also produces zero.

Reducing each endpoint separately is not equivalent to the supported operation. For example, starting at 15 and ending at 16 produces a difference of one and therefore returns zero. Separately reducing those endpoints would produce a different result.

### Negative differences

An ending quantity smaller than the starting quantity is not explicitly rejected. If its difference is representable, that negative difference reaches the same four-bit right shift.

The supplied evidence does not establish the target’s treatment of negative signed right shifts. No unconditional numerical return is therefore specified for a negative difference.

**Conditional on arithmetic, sign-extending right shift**, the result equals division by 16 rounded toward negative infinity:

| Difference | Conditional return value |
|---:|---:|
| −1 | −1 |
| −15 | −1 |
| −16 | −1 |
| −17 | −2 |
| −32 | −2 |

Rounding toward zero is not an interchangeable interpretation. For example, it would disagree with arithmetic right shift for a difference of −1.

Reversing the endpoints need not negate the return value. Under arithmetic right shift, a difference of one returns zero, whereas a difference of negative one returns negative one.

The absence of an ordering check does not establish that reversed endpoints are intended application inputs.

### Arithmetic limits

Representability is required for the subtraction itself, before reduction. A mathematical quotient that would fit in the return type does not make an overflowing intermediate subtraction supported.

No explicit range check, saturation, widening step, or overflow-reporting path is visible. This contract promises no numerical result when the difference lies outside the signed arithmetic range.

Under ordinary source-level signed arithmetic rules, an overflowing subtraction does not define a usable result. Any claim about wrapping, trapping, saturation, or successful machine-level completion would require additional authoritative evidence about the execution model.

### Invalid inputs and errors

There is no explicit validation of:

- Pointer validity or alignment.
- Availability of the second element.
- Arithmetic range.
- Endpoint ordering.
- Divisibility by 16.

Invalid storage is not guaranteed to receive a safe rejection. No specific signal, exception, diagnostic, fallback value, or termination behavior is established.

Zero is an ordinary numerical result. A negative result can also be ordinary under the applicable negative-shift semantics. Neither is an established error sentinel.

## Side effects

For valid, stable ordinary storage and a representable subtraction, the visible behavior consists of reading the two values and returning the calculated result.

The supplied function contains no explicit:

- Store to either input element.
- Store to other memory.
- Global-state access.
- Function call.
- Allocation or release.
- Ownership-transfer operation.
- Logging or diagnostic operation.
- Retention of the input pointer.

The result is not written back into the pair. Without external modification, ordinary input storage remains unchanged.

These observations describe operations visible in the function, not ownership of the accessed objects. They neither identify an owner nor grant the caller permission to release, modify, or retain the storage.

The absence of explicit writes also does not characterize every environmental consequence of a read. Invalid accesses can fault, and specially mapped storage can attach effects to reads. Those consequences are outside the established ordinary-storage behavior.

No assumptions about unknown callees are necessary because no function call is visible.

## Dependencies

No function dependencies are visible. There are no dependency function IDs to enumerate.

The following environmental properties affect the behavioral contract:

| Property | Consequence |
|---|---|
| Signed integer width and range | Determines valid endpoint values and representable differences |
| Element size and alignment | Determines concrete storage layout and valid addresses |
| Object-access rules | Determines whether both consecutive-element accesses are permitted |
| Memory-model rules | Determines whether concurrent access is valid |
| Negative signed-right-shift semantics | Determines numerical results for negative differences |
| Authoritative overflow semantics | Determines whether behavior beyond the supported subtraction range can be described |
| Storage access permissions and lifetime | Determines whether the reads are valid, independently of ownership |

A familiar platform ABI or presumed machine instruction sequence cannot substitute for missing evidence.

## Acceptance examples

These are derived static expectations, not recorded executions. Unless a violated precondition is being illustrated, assume valid, stable ordinary storage and a representable subtraction.

| Starting quantity | Ending quantity | Difference | Expected return |
|---:|---:|---:|---|
| 0 | 0 | 0 | 0 |
| 7 | 22 | 15 | 0 |
| 7 | 23 | 16 | 1 |
| 7 | 38 | 31 | 1 |
| 7 | 39 | 32 | 2 |
| −20 | −4 | 16 | 1 |
| −20 | −20 | 0 | 0 |
| 15 | 16 | 1 | 0 |
| 16 | 0 | −16 | −1, conditional on arithmetic right shift |
| 1 | 0 | −1 | −1, conditional on arithmetic right shift |
| 17 | 0 | −17 | −2, conditional on arithmetic right shift |

Additional acceptance conditions:

- **Reduction follows subtraction.** Starting at 15 and ending at 16 returns zero. Reducing each endpoint independently does not satisfy this case.
- **Remainders are permitted.** Starting at 7 and ending at 38 returns one; the difference of 31 need not be a multiple of 16.
- **Endpoint signs do not determine the result sign.** Starting at −20 and ending at −4 returns one.
- **Equality includes negative values.** Equal negative endpoints return zero.
- **Subtraction range matters before reduction.** Conditional on signed 32-bit arithmetic, starting at −2,147,483,648 and ending at zero produces an unrepresentable positive difference. No return value is promised, even though the mathematical quotient after division by 16 would fit.
- **Access permission does not require caller ownership.** If the caller has valid access to two suitable elements owned by another party, ownership alone adds no restriction visible in this function. All storage and arithmetic preconditions still apply.
- **Insufficient storage is unsupported.** Availability of only the first element does not require a sentinel return or clean failure.
- **Read order is not an acceptance requirement.** Requiring either element to be observed first would add an unsupported guarantee.
- **Ordinary input storage is preserved.** With valid storage and no external writer, the function changes neither element. This does not establish who owns either element.

## Uncertainties

The following remain **documented_only**:

- Exact integer width, signed representation, element size, alignment, and byte order.
- Target-specific numerical results for negative signed right shifts.
- Machine-level consequences of an overflowing subtraction.
- Physical load count, access widths, and ordering between the two observations.
- Fault behavior for invalid, inaccessible, or misaligned storage.
- Effects of reads from specially mapped memory.
- Ownership of the accessed storage and the origin of the caller’s access permission.
- Application meaning, measurement units, and intended restrictions on the endpoint values.

**Inferred intent:** counting groups of 16 in a directed difference is consistent with the established nonnegative behavior. It does not establish that the inputs are addresses, that the result counts objects, or that any object occupies 16 bytes.

Missing validation does not broaden the supported input domain. Invalid storage and overflowing arithmetic remain outside the promised behavior. Conversely, negative endpoint values alone do not violate the contract when access is valid and the subtraction is representable.

Unknown ownership does not prevent specifying the calculation. The relevant access precondition is permission to perform the required reads, not proof that the caller owns the storage.

## Evidence

The supplied static representation establishes:

- One pointer parameter identifying signed integer storage.
- Reads of the element at that address and the immediately following element.
- Subtraction of the first element’s value from the second element’s value.
- A four-bit right shift applied to the completed difference.
- Direct return of the resulting signed integer.
- No explicit calls, stores, validation branches, allocation, release, ownership-transfer operations, pointer retention, or widening conversions.

The access expressions establish arithmetic roles but not a temporal order between the reads. The pointer parameter establishes neither ownership nor the caller’s authority to access the storage; valid access is a precondition.

Concrete ABI properties, negative-shift results, and behavior after an out-of-range subtraction remain unresolved by this evidence.

The acceptance examples are deductions under their stated conditions. No runtime tests, target-instruction checks, or measured observations have been performed.
