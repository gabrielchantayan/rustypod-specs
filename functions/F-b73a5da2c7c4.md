<!-- Trusted provenance: F-b73a5da2c7c4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b73a5da2c7c4

## Purpose

F-b73a5da2c7c4 conditionally reduces a 32-bit extent recorded in a caller-supplied object. It first performs unconditional initialization, then chooses among an existing-error return, an unchanged-extent return, a flag-controlled direct update, or a prepared update that may invoke an associated object.

On the prepared path, the function can query a two-word quantity and compare it with a signed 64-bit product derived from the requested extent. If those quantities differ, it invokes one of two direction-dependent operations before committing the requested extent.

**Inferred intent:** the object may represent a resource whose logical extent can be reduced, with optional adjustment of an associated resource. The supplied evidence does not establish resource type, measurement units, or the meanings of the dependency operations. Those interpretations are not requirements of this contract.

## Interface

### Inputs and result

The function accepts four 32-bit inputs:

| Input | Representation and role |
|---|---|
| Object address | Address of mutable storage containing the fields described below. |
| Requested extent | Unsigned 32-bit value for extent comparison and assignment. The same bit pattern is interpreted as signed 32-bit when used in the later product. |
| Initial quantity, low word | Unsigned 32-bit initial contents of a temporary quantity’s low word. |
| Initial quantity, high word | Signed 32-bit initial contents of the temporary quantity’s high word. |

The return value is a signed 32-bit integer.

Zero is returned for a successful direct update, a successful prepared update, or an early exit because the recorded extent is already no greater than the requested extent. Nonzero values can come from the object’s stored status or from specified dependency results. The function does not translate those nonzero values into a common error code.

The two initial quantity words are not output parameters. They initialize temporary storage that may be modified by F-da5096269123.

### Object layout

Offsets are relative to the supplied object address.

| Offset | Width | Established use |
|---|---:|---|
| `0x0e` | 8 bits | Unsigned value compared with 3 to determine eligibility for associated-object processing. |
| `0x14` | 8 bits | Flag; any nonzero value selects the direct-update path when an extent reduction is needed. |
| `0x20` | 32 bits | Stored status; nonzero selects an early return. |
| `0x24` | 32 bits | Recorded extent, compared as unsigned and overwritten on a committed update. |
| `0x40` | 32 bits | Signed factor used in the target product. |
| `0x6c` | 32-bit address | Address of an associated object. Its first 32-bit word participates in an eligibility check. |
| `0xe0` | 32 bits | Mutable field decremented after each of two specific dependency calls. |

These names describe observed uses, not independently established application semantics.

### Preconditions

The supplied object must support every read and write reached on the selected path. The function performs no visible null-address, allocation-size, alignment, or access-permission validation.

When associated-object processing is reached, the address at `0x6c` must support reading its first 32-bit word. A zero address is not separately checked before that dereference.

Dependencies may impose additional preconditions. Their requirements are not established here.

## Behavior

### 1. Unconditional initialization and early exits

F-f3192f39e4f6 is called without explicit arguments before any visible object-field access.

After it returns:

1. The function reads the 32-bit status at `0x20`.
2. If that read is nonzero, the function reads the field again and returns the value from the second read.
3. Otherwise, it reads the extent at `0x24` and compares it with the requested extent using an unsigned 32-bit comparison.
4. If the recorded extent is less than or equal to the requested extent, the function returns zero.

Thus, an equal or larger request does not cause a direct extent write or a final notification call. The unconditional initialization call still occurs.

The two status reads are distinct in the supplied representation. Stable storage normally makes their values identical, but the visible behavior does not establish a single saved status value.

### 2. Flag-controlled direct update

If the recorded extent is greater than the requested extent, the function reads the byte at `0x14`.

If that byte is nonzero:

1. It writes the requested extent to `0x24`.
2. It calls F-779292dbb800 with the object address.
3. It returns zero if that call returns normally.

This path does not visibly invoke the preparation dependencies, decrement `0xe0`, inspect the associated object, or calculate the signed product.

The extent write precedes F-779292dbb800. The function does not inspect that dependency’s result or subsequently verify that the field still contains the requested value.

### 3. First preparation stage

If the flag is zero, the function:

1. Calls F-b93e3fe564f6 with the object address.
2. Calls F-eae643b3e145 with the object address and saves its 32-bit result.
3. Reads the field at `0xe0`, subtracts one, and writes the result back.
4. Returns the saved result if it is nonzero.

The decrement therefore occurs after F-eae643b3e145 returns, including when that dependency reports an error. It operates on the field’s post-call value; it is not a restoration of a previously saved value.

No matching increment is visible in this function. Whether either dependency increments the field, and whether the field represents nesting or a reference count, is unknown.

### 4. Second preparation stage

If the first preparation result is zero, the function:

1. Calls F-b93e3fe564f6 again with the object address.
2. Calls F-41010b2f183d with the object address and the 32-bit value 4.
3. Decrements the current 32-bit field at `0xe0`.
4. Returns the saved F-41010b2f183d result if it is nonzero.

Again, the decrement occurs before examining the result for an early return. A failure here does not undo the earlier decrement or any dependency side effects.

### 5. Temporary quantity and associated-object eligibility

After both preparation results are zero, the function initializes a contiguous two-word temporary quantity:

- Low word: the third input’s 32-bit pattern.
- High word: the fourth input’s 32-bit pattern.
- Combined value: those words interpreted as a signed 64-bit quantity, with the high word supplying the sign.

It then checks associated-object eligibility:

1. Read the unsigned byte at `0x0e`.
2. If that value is at most 3, skip associated-object processing.
3. Otherwise, read the address at `0x6c` and dereference its first 32-bit word.
4. If that word is zero, skip associated-object processing.
5. Otherwise, perform the query and comparison below.

The test is of the associated object’s first word, not merely of whether its address is nonzero.

### 6. Query and target calculation

When eligible, the function calls F-da5096269123 with:

- The associated-object address obtained through `0x6c`.
- The address of the temporary two-word quantity.

After that call returns, the function reads the signed 32-bit factor at `0x40` and calculates the target as:

**signed 32-bit factor × signed interpretation of the requested extent**

Both operands are widened to signed 64-bit before multiplication. The full product is retained. Every product of two signed 32-bit values fits in signed 64-bit storage.

Only after calculating this target does the function examine the saved F-da5096269123 result. If that result is nonzero, it returns the result without a direct extent write or final notification.

Consequently, the factor read and target calculation occur even when the query reports failure. No comparison-driven associated operation occurs on that failure path.

On query success, the comparison uses the temporary quantity as it exists after the query. The function does not enforce that the dependency writes both words; any word left unchanged retains its initialized input value.

### 7. Direction-dependent associated operation

The target and post-query quantity are compared as signed 64-bit values.

- **Equal:** no adjustment dependency is called.
- **Target greater than quantity:** call F-87fe98a22e06.
- **Target less than quantity:** call F-1d494943708d.

The greater-than call receives:

1. A newly read associated-object address from `0x6c`.
2. The address of a fixed data object.
3. The 32-bit value 1.

The fixed data object’s contents, extent, and interpretation are not supplied.

The less-than call receives:

1. A newly read associated-object address from `0x6c`.
2. A 32-bit high-word subtraction result.
3. The target’s low 32-bit word.
4. The target’s high 32-bit word.

For that subtraction result, the function subtracts the quantity’s high word from the target’s high word, then subtracts one additional unit if the target’s low word is less than the quantity’s low word under unsigned comparison. The result is retained as a 32-bit word. It is not the entire difference, and the separately passed low word is the target’s low word, not the difference’s low word.

The signed direction decision accounts for subtraction overflow. A wrapped high-word subtraction’s sign alone does not define the comparison.

If the selected adjustment dependency returns nonzero, the function returns that value immediately. Otherwise it continues to commit the extent.

### 8. Commit

The prepared path reaches commit when:

- Associated-object processing was skipped;
- The target equaled the queried quantity; or
- The selected adjustment operation returned zero.

Commit consists of:

1. Writing the requested extent to `0x24`.
2. Calling F-779292dbb800 with the object address.
3. Returning zero if that call returns normally.

There is no second check of the stored status, recorded extent, or flag after preparation.

## Side effects

### Direct writes

The function visibly writes only:

- The extent field at `0x24`, on either commit path.
- The field at `0xe0`, once after each reached preparation-result call.
- Its own temporary two-word quantity during initialization.

F-da5096269123 receives a writable address to that temporary quantity.

The decrements are 32-bit operations. The evidence establishes the decrement and its placement, but not a valid application range for the field or an intended signed-underflow policy. A counter-like interpretation must not introduce saturation, clamping, or conditional suppression.

### Dependency effects and ownership

Every dependency may have additional effects not visible in the supplied body. In particular, this contract does not assume that dependencies preserve object fields, globals, associated-object addresses, or ownership.

No allocation, release, or explicit ownership transfer is visible in F-b73a5da2c7c4 itself.

The temporary quantity has function-local lifetime. Its address is passed to F-da5096269123, but the evidence does not establish whether that dependency retains the address. Safe retention beyond the call is not an established property.

The associated-object address is reloaded for an adjustment call after the query. Therefore, the query and adjustment are not guaranteed by this body alone to receive the same address.

### Failure and rollback

The function performs no visible rollback.

A nonzero preparation, query, or adjustment result prevents this function’s later extent assignment and final notification. It does not reverse prior decrements or dependency effects. A dependency could itself have modified the extent before returning failure; the caller cannot infer unchanged object state merely from a nonzero result.

Faults, exceptions, nonlocal exits, or nonreturning dependencies have no visible recovery handling.

## Dependencies

| Opaque ID | Visible interaction |
|---|---|
| F-f3192f39e4f6 | Unconditional entry call without explicit arguments; no result is inspected. |
| F-b93e3fe564f6 | Called before each reached preparation-result call; receives the object address. |
| F-eae643b3e145 | First preparation result; nonzero propagates after one decrement. |
| F-41010b2f183d | Second preparation result; receives the object address and 4; nonzero propagates after the second decrement. |
| F-da5096269123 | Receives an associated-object address and writable two-word temporary; nonzero propagates after target calculation. |
| F-87fe98a22e06 | Used when the signed target is greater than the signed quantity; receives the associated-object address, fixed-data address, and 1. |
| F-1d494943708d | Used when the signed target is less than the signed quantity; receives the associated-object address and three specified 32-bit words. |
| F-779292dbb800 | Called after each direct extent assignment; no result is inspected. |

Names such as “query,” “preparation,” and “adjustment” describe call placement and data flow, not independently proven dependency semantics.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Unless specified otherwise, dependencies return normally and do not make additional relevant changes.

1. **Stored status takes precedence.**  
   After entry initialization, the status field remains −7 across both reads. The function returns −7 without reading the extent or taking a preparation path.

2. **No enlargement.**  
   Status is zero, recorded extent is 12, and requested extent is 12 or 20. The function returns zero without directly writing the extent or calling F-779292dbb800.

3. **Nonzero flag permits direct commit.**  
   Recorded extent is 20, request is 12, and the flag byte is 128. The function writes 12, calls F-779292dbb800, and returns zero. The flag need not equal 1.

4. **First preparation failure still decrements.**  
   F-eae643b3e145 returns 9 and leaves `0xe0` equal to 6. The function writes 5 to that field and returns 9. The second preparation stage is not reached.

5. **Second preparation failure preserves earlier effects.**  
   The first result is zero and the second result is −3. Both decrement sites execute. The function returns −3 without its own extent commit.

6. **Eligibility threshold skips the query.**  
   Both preparation results are zero and the byte at `0x0e` is 3. The function does not read the associated-object address for the eligibility check. It commits the requested extent and invokes F-779292dbb800.

7. **Equal quantity avoids adjustment.**  
   Associated-object processing is eligible, factor is 4, request is 5, and a successful query leaves quantity 20. Neither adjustment dependency is called; commit follows.

8. **Greater target selects F-87fe98a22e06.**  
   With target 20 and queried quantity 10, F-87fe98a22e06 is selected. Its zero result permits commit; a nonzero result is returned without this function’s commit.

9. **Less target includes low-word borrow.**  
   With target 20 and quantity 30, both high words are zero, but the low-word subtraction borrows. F-1d494943708d receives a high-word subtraction result with bit pattern `0xffffffff`, target low word 20, and target high word zero.

10. **Request interpretation changes at the sign boundary.**  
    A request with bit pattern `0xffffffff` is 4,294,967,295 for the unsigned extent comparison. No unsigned 32-bit recorded extent exceeds it, so the ordinary path exits before product calculation. If a product is reached for another request with its top bit set, that request is interpreted as negative for multiplication, not as its unsigned magnitude.

11. **Query failure still precedes a factor read.**  
    F-da5096269123 returns 11. The function reads the factor and calculates the product before returning 11. It does not perform an adjustment or its own extent commit.

## Uncertainties

The following remain **documented_only**:

- The resource type and units represented by the extent and factor.
- The operational meaning of either adjustment dependency.
- The fixed data object’s contents and required lifetime.
- The meaning and valid range of the field at `0xe0`.
- Dependency mutations, ownership rules, synchronization, and memory retention.
- Whether external synchronization guarantees stable repeated reads.
- Invalid-address behavior beyond the absence of visible validation.
- Application-level guarantees that prevent counter underflow or negative target products.

## Evidence

Established behavior comes solely from the supplied function body: visible field widths and offsets, call ordering, branches, direct writes, result propagation, and two-word arithmetic.

The signed 64-bit comparison description follows from the displayed low-word borrow and signed-overflow-aware high-word comparison. Resource-oriented terminology is inferred intent only.

No dependency implementations or runtime observations were supplied. No runtime tests have been performed.
