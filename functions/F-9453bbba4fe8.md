<!-- Trusted provenance: F-9453bbba4fe8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9453bbba4fe8

## Purpose

F-9453bbba4fe8 conditionally requests growth of a storage-like state object. It compares a stored signed 32-bit quantity with a signed 32-bit request. When the stored quantity is smaller, it either reports a fixed limit condition or invokes F-6dd003795db7 with a growth target derived from the stored quantity.

The request controls whether the growth path is entered. It does **not** determine the target passed to the dependency, and a successful call does not establish that the resulting stored quantity satisfies the request.

The interpretation of the state as a resizable storage descriptor is inferred intent. The directly established behavior is comparison, target calculation, dependency invocation, conditional state modification, and construction of a 64-bit result.

## Interface

### Inputs

The function receives:

1. A reference to a state object containing at least four consecutive 32-bit words.
2. A signed 32-bit request.

The source accesses the state using the following layout:

| Byte offset | Width | Role in this function |
|---|---:|---|
| 0 | 32 bits | Value captured before the dependency call, supplied to that dependency, and subsequently replaced by its return value |
| 4 | 32 bits | Signed stored quantity used for the comparison and growth calculation; conditionally replaced after the dependency call |
| 8 | 32 bits | Not directly accessed |
| 12 | 32 bits | Value supplied as the dependency’s first argument |

The names “stored quantity,” “storage value,” and “context value” describe their uses here, not established data types beyond the visible 32-bit representation. In particular, the word at offset 0 may represent an address or handle, but this function alone does not establish which.

### Result

The result is a 64-bit bit pattern with two 32-bit components:

- Bits 0–31 contain a status value.
- Bits 32–63 contain either the original request or the value captured from state offset 0 before the dependency call.

The upper component depends on the path:

| Path | Low 32 bits | High 32 bits |
|---|---|---|
| Stored quantity is at least the request | 0 | Original request |
| Growth is required and the stored quantity equals 536,870,911 | 64 | Original request |
| Dependency is called | Status held after the dependency returns | Pre-call value captured from offset 0 |

These components are combined by their 32-bit representations. A negative request or negative captured state word therefore contributes its two’s-complement bit pattern to the upper half; the result is not a numeric sum of signed components.

The status is initialized to zero before any dependency invocation. A zero status after the dependency returns causes the stored quantity to be updated. No broader meaning for dependency-produced nonzero statuses is established.

### Preconditions and validity

For ordinary execution, the state reference must support the accesses taken by the selected path:

- Every path reads the word at offset 4.
- The dependency path additionally reads offsets 0 and 12 and writes offset 0.
- That path also writes offset 4 when the final status is zero.

The function contains no explicit null-reference, alignment, object-size, ownership, or lifetime checks. Invalid memory accesses do not have a defined error result in the supplied evidence.

A nonnegative stored quantity within the apparent limit is a plausible intended input domain, but it is not checked. Negative stored quantities and requests participate in signed comparisons.

## Behavior

### Initial decision

A local 32-bit status is initialized to zero. The function then reads the signed stored quantity at offset 4.

If the stored quantity is greater than or equal to the request, the function returns immediately without invoking F-6dd003795db7 and without directly modifying the state object. Its result contains status zero and the original request in the upper half.

Equality therefore does not trigger growth. A negative request also does not trigger growth when the stored quantity is nonnegative.

### Growth target

If the stored quantity is smaller than the request, the function calculates a proposed target from the stored quantity alone.

The visible calculation has two 32-bit stages:

- A growth base formed from the stored quantity plus that quantity shifted right by one bit.
- A tentative target formed by adding four to the growth base.

For nonnegative values where the arithmetic remains representable, the tentative target is:

**stored quantity + floor(stored quantity / 2) + 4**

The request is not used in this calculation. There is no loop, repeated growth, or second comparison against the request.

The supplied source places these calculations before the special limit check described below. For the exact limit value, both calculations remain within signed 32-bit range.

### Exact-limit condition

If growth is required and the stored quantity is exactly **536,870,911**, the function:

- Sets status to **64**.
- Does not invoke the dependency.
- Does not directly write any state field.
- Returns the original request in the upper half of the result.

This is an equality check, not a general rejection of stored quantities above the limit.

A state already holding 536,870,911 does not produce status 64 unless the request is strictly greater than that value. Requests equal to or below that stored quantity take the initial no-growth path and return status zero.

### Target adjustment

For growth paths other than the exact-limit condition, the tentative target is replaced by **536,870,911** if either of the following is true:

- The tentative target is smaller than the original stored quantity.
- The tentative target is greater than **536,870,910**, and the growth base is not **536,870,907**.

Otherwise, the tentative target is retained.

The exceptional growth-base value corresponds to a tentative target of 536,870,911 when the addition of four is representable. On such arithmetic, retaining that tentative target and replacing it with the limit have the same numeric outcome.

These rules operate on the visible signed quantities. They are not evidence of a complete or portable overflow policy. In particular, the supplied source does not establish how signed overflow outside the ordinary input domain must behave.

### Dependency invocation and state commitment

After choosing a target, the function captures the 32-bit word at offset 0. This captured value becomes both:

- The fifth argument to F-6dd003795db7.
- The upper 32-bit component of this function’s eventual result.

The function invokes F-6dd003795db7 with six arguments, in this semantic order:

1. The word read from state offset 12.
2. The constant **4**.
3. The original stored quantity read from offset 4.
4. The chosen target.
5. The captured word from offset 0.
6. A reference to the local 32-bit status, initially zero.

The constant 4 is consistent with a four-byte unit size, but that meaning is inferred rather than established by the call alone.

On a normal return from the dependency, its 32-bit return value is written to state offset 0 **regardless of the status value**.

The function then examines the local status:

- If status is zero, it writes the chosen target to offset 4.
- If status is nonzero, it performs no direct write to offset 4.

The function returns the final status in the low half and the captured pre-call offset-0 value in the high half.

Consequently, a nonzero status does not imply that the state is unchanged: the write to offset 0 still occurs. Nor does the upper result half identify the replacement value written there.

## Side effects

### Direct access order

The directly established ordering is:

1. Initialize local status to zero.
2. Read state offset 4.
3. Decide whether growth is required.
4. If the dependency path is selected, determine the target.
5. Read and capture state offset 0.
6. Obtain the offset-12 value used as the dependency’s first argument.
7. Invoke F-6dd003795db7.
8. Write its return value to offset 0.
9. Inspect status and, if zero, write the chosen target to offset 4.
10. Return the packed result.

The evidence supports this ordering at the source-behavior level. It does not establish physical instruction ordering or synchronization properties.

There is no direct access to offset 8. The no-growth and exact-limit paths do not directly access offsets 0 or 12.

### Dependency-visible effects

The dependency receives the captured storage value and context word, not an explicit reference to the complete state object. Nevertheless, those values may provide access to shared or aliased storage. The function’s signature is insufficient to exclude dependency modifications to the state object, other memory, or globals.

If the dependency modifies state through aliases and returns normally:

- The subsequent direct write to offset 0 still installs its return value.
- A zero status causes offset 4 to be overwritten with the previously chosen target.
- A nonzero status causes no direct restoration or write of offset 4.

There is no visible transaction or rollback mechanism.

### Ownership and lifetime

This function contains no explicit allocation, release, copying of storage contents, or ownership transfer. Those operations may occur within F-6dd003795db7, but they are not established here.

Returning the old offset-0 value in the upper result half does not establish that this value remains valid, that the caller owns it, or that it should be released. Likewise, installing the dependency’s return value does not establish its allocation size or lifetime.

The status reference points to local storage. Whether the dependency retains that reference is unknown; safe retention beyond the call is not established.

## Dependencies

### F-6dd003795db7

This is the only visible external dependency.

Its observed interface consists of six arguments and a 32-bit result:

| Argument | Value supplied |
|---|---|
| 1 | State word at offset 12 |
| 2 | Constant 4 |
| 3 | Original signed stored quantity |
| 4 | Chosen signed target |
| 5 | Captured state word at offset 0 |
| 6 | Reference to a zero-initialized 32-bit status |

The caller uses the dependency’s return value as the replacement for offset 0. It uses the status as the condition for committing the chosen target to offset 4.

The following dependency properties are **documented_only**:

- Whether it allocates, reallocates, frees, or copies memory.
- Whether it interprets either word-sized argument as an address.
- Whether it always writes the status.
- Which nonzero statuses it can produce.
- Whether its return value is meaningful on nonzero status.
- Whether it modifies globals or memory reachable through its arguments.
- Whether it can fail to return normally.
- Whether it preserves existing storage on failure.

None of these properties may be assumed merely from the growth-like call shape.

## Acceptance examples

These are conditional acceptance examples derived from static evidence. No runtime tests have been performed. Unless specified otherwise, values are signed 32-bit quantities and arithmetic is representable.

### Request already satisfied

Given stored quantity 12 and request 12:

- No dependency call occurs.
- The function performs no direct state writes.
- Low result half is 0.
- High result half is 12.

The same no-growth behavior applies to stored quantity 12 and request 5, except that the high half is 5.

### Growth from zero

Given stored quantity 0 and request 1:

- The growth base is 0.
- The chosen target is 4.
- The dependency receives original quantity 0 and target 4.

If the dependency returns replacement value R and leaves status zero, offset 0 becomes R and offset 4 becomes 4. The upper result half is the offset-0 value captured before the call, not R.

### Request larger than the chosen target

Given stored quantity 10 and request 1,000:

- The chosen target is 19.
- The dependency is called once.
- If status remains zero, offset 4 becomes 19.

This outcome is accepted by the visible behavior even though 19 is smaller than the request. A contract requiring capacity to reach 1,000 would contradict the supplied evidence.

### Nonzero dependency status

Given stored quantity 10 and request 11, suppose the dependency returns R and sets status to 7:

- Offset 0 is written with R.
- The function does not directly write offset 4.
- The low result half is 7.
- The high result half is the captured pre-call offset-0 value.

If the dependency itself changes offset 4 through an alias, this function does not restore the original value.

### Exact limit

Given stored quantity 536,870,911 and request 536,870,912:

- No dependency call occurs.
- No direct state write occurs.
- Low result half is 64.
- High result half is 536,870,912.

With the same stored quantity and request 536,870,911, the low half is instead zero because growth is not required.

### Clamped growth

Given stored quantity 400,000,000 and request 400,000,001:

- The growth base is 600,000,000.
- The tentative target is 600,000,004.
- The chosen target is clamped to 536,870,911.
- The dependency receives that clamped target.

### Negative request

Given stored quantity 0 and request −1:

- Growth is not required under the signed comparison.
- No dependency call occurs.
- The result’s high half is the 32-bit representation of −1.
- The low half is zero.

## Uncertainties

The following remain **documented_only**, not behavioral guarantees:

- The intended meanings of the state words and of the constant 4.
- The valid application-level range of stored quantities and requests.
- Whether negative stored quantities are admissible.
- The exact overflow behavior of the growth arithmetic outside representable signed 32-bit calculations.
- The right-shift behavior for negative stored quantities. For nonnegative quantities, the shift supplies floor division by two; extending that interpretation to negative inputs is not justified here.
- Any concurrency, atomicity, or reentrancy guarantee.
- The behavior of invalid, overlapping, volatile, or concurrently modified state storage.
- The dependency’s ownership, preservation, and failure contracts.
- A semantic name for status 64 beyond its exact triggering condition.
- Whether callers interpret the 64-bit result as a scalar, paired values, or another representation.

The apparent intent is bounded geometric storage growth. That inference does not strengthen the established behavior into a promise of allocation success, request satisfaction, or failure atomicity.

## Evidence

The supplied static body establishes the state offsets, signed comparison, zero-initialized status, growth calculation, numeric limit checks, single dependency call, unconditional offset-0 replacement after normal dependency return, conditional offset-4 update, and two-part result construction.

No dependency implementation, caller contract, runtime trace, or test result was supplied. No runtime tests have been performed. Allocation semantics, ownership rules, and behavior outside the representable arithmetic domain therefore remain explicitly unresolved.
