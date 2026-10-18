<!-- Trusted provenance: F-3e2aefe00c5e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3e2aefe00c5e

## Purpose

F-3e2aefe00c5e conditionally stores a supplied 32-bit unsigned value into a fixed field of a caller-supplied object.

The operation succeeds only when all three conditions hold:

- The object pointer is non-null.
- The object's first 32-bit word equals a reference value read from global storage.
- The supplied value is greater than 2047 under an unsigned comparison.

On success, the function writes the supplied value at byte offset 16 from the object pointer and returns zero. If any condition fails, it returns 26 without performing that field write.

The first-word comparison appears intended to recognize an eligible object, but the meaning of the reference value and the purpose of the updated field are not established. In particular, the value must not be described as a length, capacity, allocation size, or limit without additional evidence.

## Interface

### Inputs

**Object pointer**

A pointer interpreted as the base of an object containing 32-bit words.

The function accesses two locations relative to this base:

| Byte offset | Width | Role |
|---|---:|---|
| 0 | 32 bits | Word compared with a global reference value |
| 16 | 32 bits | Destination of the conditional update |

The intervening bytes are not accessed by this function. No complete structure definition, object type, or allocation size is established.

A null pointer is an explicitly handled input. A non-null pointer is not checked for readability, writability, alignment, lifetime, or sufficient extent.

**Supplied value**

An unsigned 32-bit value. Its representable range is 0 through 4,294,967,295.

The acceptance boundary is strict: 2047 is rejected, and 2048 is accepted if the pointer and first-word checks also pass. Values with the high bit set are not rejected merely because the destination is represented as an integer word.

### Output

A 32-bit result with two explicitly shown outcomes:

| Result | Meaning |
|---|---|
| 0 | All conditions passed and the destination field was assigned |
| 26 | At least one condition failed |

No distinct return values identify which condition failed. The supplied evidence does not associate 26 with a named error category or external error convention.

A fault caused by invalid memory access is not shown as a returned error. A result of 26 must therefore not be interpreted as a general invalid-pointer response.

### Preconditions for defined memory access

For a non-null pointer, the first 32-bit word must be accessible for reading. If the first-word comparison passes and the supplied value exceeds 2047, the 32-bit field at offset 16 must also be accessible for writing.

The global reference storage must be readable whenever the non-null path evaluates the first-word comparison.

Any alignment requirements are those of the target environment's 32-bit integer accesses; no alignment repair or alternate unaligned access path is shown.

## Behavior

### Validation and access order

The decision is short-circuited in this order:

1. Determine whether the object pointer is null.
2. For a non-null pointer, compare its first 32-bit word with the global reference value.
3. Only if that comparison succeeds, determine whether the unsigned supplied value exceeds 2047.
4. Only if all conditions succeed, assign the supplied value to the field at byte offset 16.

This ordering has observable consequences:

- A null pointer does not cause an object-word read or a global reference read through the guarded comparison.
- A first-word mismatch prevents evaluation of the final value condition in the shown expression.
- A small supplied value does not protect a non-null pointer from being dereferenced. The first-word comparison precedes the size-like threshold check.
- The destination field is not read before being written.
- The destination field is not written on any rejecting path.

The comparison requires both the object's first word and the global reference value. The supplied expression does not establish an ordering between those two operand reads. It establishes that the comparison is guarded by the non-null check.

### Successful update

When the pointer is non-null, the first word matches the reference value, and the supplied value is in the inclusive range 2048 through 4,294,967,295:

- The function performs one explicit 32-bit assignment at byte offset 16.
- The assigned value is the supplied 32-bit value, not a value computed from the previous destination contents.
- The function returns zero.

The old destination contents do not affect acceptance or the assigned value. There is no accumulation, rounding, clamping, allocation-size calculation, or comparison against the previous field value.

Assigning a value already present in the destination still takes the success path. Success is not conditioned on whether the stored bits change.

### Rejection

The function returns 26 when:

- The pointer is null.
- The pointer is non-null but the first word differs from the global reference value.
- The pointer is non-null and the first word matches, but the supplied unsigned value is at most 2047.

These paths perform no explicit store.

### Integer boundaries

The threshold comparison is unsigned and 32-bit. Consequently, all of the following exceed the threshold:

- 2048
- 2,147,483,647
- 2,147,483,648
- 4,294,967,295

There is no addition, subtraction, multiplication, or scaling of the supplied value before assignment, so no arithmetic overflow boundary is involved in accepting it.

The destination is represented in the supplied material as a 32-bit integer slot, while the input is unsigned. The contract records the full-width 32-bit assignment; it does not infer a portable signed numeric interpretation for destination values above 2,147,483,647.

## Side effects

The only explicit write is the successful assignment to the 32-bit destination at byte offset 16.

The function contains no explicit allocation, release, ownership transfer, callback, logging operation, or synchronization operation. It neither returns an object pointer nor replaces the caller's pointer.

The visible accesses are limited to:

- Reading the object's first word on the non-null path.
- Reading the global reference value for that comparison.
- Writing the destination word on success.

No other object fields are explicitly accessed.

There is no stated non-aliasing restriction between the object and global storage. Therefore, “only the destination is written” is an address-based statement, not proof that all globally named state remains unchanged. If the destination aliases global storage, the successful assignment affects that storage through the alias. No reference-value reload or post-write validation is shown.

The function provides no established thread-safety or atomicity guarantee. Concurrent modification of the object or global reference is outside the demonstrated sequential behavior.

## Dependencies

There are no visible function calls and therefore no callee dependencies to identify by opaque function ID.

The operation depends on a 32-bit reference value in global storage. The following are not established:

- Its initial value.
- How or when it is initialized.
- Whether other operations modify it.
- Whether it identifies an object type, state, generation, or another property.

Acceptance depends on the value read during the comparison, not on a documented constant.

No hidden validation, memory management, or preservation behavior is attributed to another function.

## Acceptance examples

These examples are conditional expectations derived from the supplied static evidence, not executed test results. Unless stated otherwise, the required memory is valid and no concurrent modification occurs.

| Conditions | Expected result | Expected destination effect |
|---|---:|---|
| Null pointer; supplied value 2048 | 26 | No write |
| Non-null pointer; first word mismatches; value 2048 | 26 | No write |
| Non-null pointer; first word matches; value 0 | 26 | No write |
| Non-null pointer; first word matches; value 2047 | 26 | No write |
| Non-null pointer; first word matches; value 2048 | 0 | Assigned 2048 |
| Non-null pointer; first word matches; value 2,147,483,648 | 0 | Assigned the supplied 32-bit value |
| Non-null pointer; first word matches; value 4,294,967,295 | 0 | Assigned the supplied 32-bit value |

Additional boundary cases:

- **Small value with an invalid non-null pointer:** a clean rejection is not guaranteed. The function attempts the first-word comparison before applying the threshold condition.
- **Readable first word but inaccessible destination:** a mismatch or a value at most 2047 avoids the destination write. A matching first word and a larger value require that write; no recovery path is shown.
- **Destination already equals the input:** the function still performs the successful assignment and returns zero.
- **Reference changes between separate calls:** an unchanged object can pass one call and fail another if the reference values read by those calls differ.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning of the object, reference value, and destination field.
- The complete object layout and its ownership or lifetime rules.
- The target's detailed alignment, fault, and memory-access semantics.
- Whether callers prohibit aliasing with the reference storage.
- Any external synchronization requirements.
- Whether the return value 26 has a broader application-specific meaning.
- The interpretation of the destination as a signed quantity after storing a value above the signed 32-bit maximum.

The first-word check suggests object recognition or eligibility validation. That is inferred intent, not an established type-safety guarantee. Matching one word does not prove that the destination exists or is writable.

## Evidence

**Established static evidence:** a null guard; a first-word comparison against global storage; a strict unsigned threshold of 2047; a conditional 32-bit assignment at byte offset 16; and return values zero and 26. No calls or additional explicit stores are present.

**Inferred intent:** recognition of an eligible object followed by an update of one associated field.

**Unknowns:** domain meaning, surrounding object invariants, reference initialization, memory ownership, concurrency rules, and platform behavior beyond the represented accesses.

No runtime tests have been performed. The acceptance examples specify expected behavior from static evidence and do not constitute observed execution results.
