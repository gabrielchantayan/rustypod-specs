<!-- Trusted provenance: F-63ecaced33d1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-63ecaced33d1

## Purpose

F-63ecaced33d1 adjusts a descriptor’s logical element count and corresponding byte extent, optionally invokes a range-adjustment operation, and conditionally requests a storage-size change.

**Established behavior:** the function performs arithmetic and unsigned boundary checks, may write a status field, invokes up to three opaque dependencies, and commits two descriptor fields only after the applicable storage-request path succeeds.

**Inferred intent:** the descriptor appears to represent a variable-sized collection with a fixed element width. Positive adjustments generally reduce its count; negative adjustments generally increase it. The range-adjustment dependency may move elements, but its actual behavior is not established here.

This is not an atomic operation. A nonzero return can follow descriptor mutations or dependency side effects.

## Interface

### Inputs

The function receives three values:

1. **Descriptor address.** A 32-bit address-like input used to access the fields below.
2. **Adjustment.** A signed 32-bit value. Its effective value may be replaced by a boundary calculation before being negated.
3. **Position.** An unsigned 32-bit value. The exact value `0x7fffffff` has special handling for the initial position check, but remains unchanged in later calculations.

The visible descriptor layout is:

| Byte offset | Width | Contract-facing name | Observed use |
|---|---:|---|---|
| `+0x04` | 32 bits | Element width | Loaded as signed for multiplication |
| `+0x08` | 32 bits | Element count | Compared as unsigned initially; later loaded as signed for addition |
| `+0x0c` | 32 bits | Recorded byte extent | Read during one storage-size calculation; overwritten on commit |
| `+0x10` | 32 bits | Capacity quantity | Compared as unsigned and supplied to storage-size calculations |
| `+0x14` | 32 bits | Status | Early-return gate; explicitly set to one on one path |

These names describe apparent roles, not independently verified meanings. In particular, the capacity quantity is not proven to equal allocated storage size.

The supplied body does not access offset zero directly. Dependencies may access additional fields.

### Output

The return value is unsigned and 32 bits wide:

- A nonzero entry status is returned immediately.
- An effective position greater than the observed count is returned directly.
- A nonzero result from the final storage-size request is propagated.
- Otherwise, the function returns zero.

Zero does not imply that the function changed anything. Conversely, nonzero does not imply that it left the descriptor unchanged.

### Preconditions and arithmetic model

The descriptor must support the accesses performed on the selected path. Dependencies may impose additional requirements that are not visible here.

Unsigned calculations use 32-bit arithmetic, including wraparound. In expressions combining the unsigned position with the signed adjustment, the adjustment is converted to an unsigned 32-bit value before addition. Those comparisons are unsigned.

Some later operations are expressed as signed 32-bit arithmetic:

- Negating the effective adjustment.
- Adding that negation to the count loaded as signed.
- Multiplying the prospective count by the signed element width.
- Multiplying the negated adjustment by the signed element width on the growth path.

For a source-level contract, these operations must remain representable as signed 32-bit values. The supplied representation does not establish a portable result for signed overflow, including negation of the minimum signed 32-bit value. A target-machine wraparound interpretation would require additional evidence.

## Behavior

### 1. Entry status gate

The first observable descriptor access reads status at `+0x14`.

If that value is nonzero, the function reads that field again and returns the second value. It performs no explicit writes and invokes no dependencies on this path.

With stable memory, the returned value equals the nonzero value initially observed. The separate reads matter if another actor can modify the descriptor; no concurrency protocol is established.

### 2. Initial position checks

If entry status is zero, the function derives an **effective position**:

- Ordinarily, it is the supplied position.
- If the supplied position is exactly `0x7fffffff`, it is the count read from `+0x08`.

If the effective position is zero, the function returns zero immediately.

Otherwise, it reads the count from `+0x08` for an unsigned comparison. If that count is less than the effective position, it returns the effective position without an explicit write or dependency call.

The sentinel affects only these preliminary checks. The original supplied position—including `0x7fffffff`—continues to participate in the subsequent boundary arithmetic and dependency arguments. This is not an established “replace position with count everywhere” interface.

For a non-sentinel position of zero, the function can return without reading the count. For the sentinel, obtaining the effective position requires a count read first.

### 3. Adjustment clipping and optional range operation

The count observed for the unsigned position comparison is retained for this stage.

The function adds the supplied position to the adjustment using unsigned 32-bit arithmetic. If the retained count is less than that sum, it replaces the effective adjustment with:

**retained count − supplied position + 1**

The subtraction and addition here use unsigned 32-bit arithmetic. The resulting 32-bit value is assigned back to the signed adjustment quantity. Conversion of an out-of-range unsigned result to signed is a source-model limitation documented below.

The function then recomputes supplied position plus effective adjustment. If this unsigned sum is less than or equal to the retained count, it performs these actions in order:

1. Writes the 32-bit value one to status at `+0x14`.
2. Calls **F-5e96c6f94ff9** with:
   - The descriptor address.
   - The unsigned sum of supplied position and effective adjustment.
   - The retained count.
   - The negation of the effective adjustment.
3. Calls **F-ca132d956acc** with the descriptor address.

The caller does not inspect a result from either dependency.

If the comparison fails, all three actions are skipped.

In ordinary non-wrapping arithmetic, clipping makes the recomputed sum equal to retained count plus one. Consequently, clipping generally suppresses this range-operation branch. At the unsigned boundary, wraparound can change that outcome; the actual 32-bit comparison governs.

The status write precedes both dependencies. There is no explicit restoration or clearing of status later in this function.

### 4. Prospective count and byte extent

After the optional dependencies return, the function negates the effective adjustment. Call this signed result the **count delta**.

It then performs the following observations and calculations:

1. Reads the current count from `+0x08` as signed.
2. Adds the count delta to obtain the **prospective count**.
3. Reads the capacity quantity from `+0x10`.
4. Reads the element width from `+0x04` as signed.
5. Multiplies prospective count by element width to obtain the **prospective byte extent**, stored and subsequently compared as unsigned.

These are fresh observations after the optional dependencies. The prospective count is not necessarily based on the count retained for the earlier boundary checks. Likewise, neither element width nor capacity is assumed to have survived the dependencies unchanged.

There is no explicit rejection of a negative prospective count, a negative element width, or a byte extent whose unsigned representation is large. Behavior is governed by the stated arithmetic preconditions and subsequent unsigned comparisons.

### 5. Capacity below prospective byte extent

If the observed capacity quantity is less than the prospective byte extent, the function computes a growth request.

It reads element width again and multiplies it by the count delta. The result is treated as an unsigned **delta-byte quantity**.

The **growth increment** is selected by three unsigned operations, in this order:

1. Take the larger of delta-byte quantity and observed capacity.
2. Limit that result to at most `0x10000`, or 65,536.
3. Raise it again if necessary so that it is at least the delta-byte quantity.

Equivalently, the increment is the larger of delta-byte quantity and the smaller of 65,536 and the larger of delta-byte quantity and capacity.

Thus, 65,536 is not an absolute upper limit: a larger delta-byte quantity overrides it.

The function calls **F-7eca6144d13b** with:

- The descriptor address.
- Zero as the second argument.
- Observed capacity plus growth increment as the third argument.

That final addition is unsigned 32-bit arithmetic and may wrap.

If this first request returns zero, the function proceeds directly to the commit described below.

If it returns nonzero, that result is not immediately propagated. The function instead calls **F-7eca6144d13b** again with the same descriptor, zero, and the prospective byte extent.

- A nonzero result from this second request is returned.
- A zero result permits the commit.

The second request uses the already computed prospective byte extent. It does not recalculate the extent or reread capacity after the first request.

### 6. Capacity sufficient for prospective byte extent

If observed capacity is greater than or equal to prospective byte extent, the signed count delta determines the next step.

- **Count delta zero or positive:** no storage-size request occurs; the function proceeds directly to commit.
- **Count delta negative:** the function reads recorded byte extent from `+0x0c` and requests:

  **prospective byte extent + observed capacity − recorded byte extent**

This calculation is unsigned 32-bit arithmetic. It appears intended to retain the difference between capacity and the previously recorded extent, but no invariant guaranteeing that interpretation is established.

The request is made through **F-7eca6144d13b**, with the descriptor, zero, and the calculated quantity.

- A nonzero result is returned without the final explicit commit.
- A zero result permits the commit.

### 7. Commit

Every path reaching the commit performs these explicit writes in order:

1. Writes prospective count to `+0x08`.
2. Writes prospective byte extent to `+0x0c`.
3. Returns zero.

These values were calculated before any storage-size request. If that dependency modifies the count or recorded extent, the final writes overwrite those modifications with the previously calculated values.

The function does not explicitly update element width or capacity, and it does not clear status.

## Side effects

The only explicit descriptor writes established by the supplied body are:

- Status set to one before the optional range operation.
- Count written at commit.
- Recorded byte extent written immediately after count at commit.

All dependency side effects remain part of the observable behavior but are unspecified here. They may modify the descriptor, external state, or storage.

In particular:

- The range operation and subsequent maintenance call can run before a later storage-size request fails.
- A failed initial growth request can have side effects before the fallback request.
- Returning an error does not roll back status, dependency effects, or storage changes.
- A storage-size request returning zero is followed by the two explicit commit writes.
- No explicit allocation, deallocation, ownership transfer, or memory release appears outside the dependencies.

The caller’s ownership obligations cannot be inferred from these calls alone. It is not established that any dependency preserves globals, keeps storage addresses stable, or releases memory on failure.

## Dependencies

| Opaque ID | Invocation and established role |
|---|---|
| **F-5e96c6f94ff9** | Conditional four-argument call after status is written to one. Receives the descriptor, computed boundary, retained count, and negated effective adjustment. Its precise data movement and side effects are unknown. |
| **F-ca132d956acc** | Called with the descriptor immediately after F-5e96c6f94ff9 returns. Its result is not inspected. Its purpose and preservation guarantees are unknown. |
| **F-7eca6144d13b** | Receives the descriptor, zero, and a requested 32-bit quantity. Zero allows progress; nonzero either triggers the growth fallback or becomes the function’s returned result. Actual allocation semantics and failure guarantees are unknown. |

No dependency’s name or call shape establishes ownership or a transactional guarantee.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless stated otherwise, fields remain stable except for the function’s explicit writes, all dependencies return normally, and signed calculations are representable.

### Entry status bypass

With status equal to seven and unchanged between its two reads:

- The function returns seven.
- No other descriptor field is read by this function.
- No dependency is called.
- No explicit write occurs.

### Zero position

With status zero and supplied position zero:

- The function returns zero.
- It does not invoke dependencies or explicitly modify the descriptor.
- It does not need to read the count.

If position is instead `0x7fffffff` and count is zero, the result is also zero, but count must first be read.

### Position beyond count

With status zero, count ten, and supplied position eleven:

- The function returns eleven.
- Adjustment has no effect.
- No dependency or explicit write occurs.

### Ordinary positive adjustment with sufficient capacity

Assume count ten, element width four, recorded extent forty, capacity sixty-four, position three, and adjustment two.

- The boundary is five, which is no greater than ten.
- Status is written to one.
- F-5e96c6f94ff9 receives descriptor, five, ten, and negative two.
- F-ca132d956acc follows.
- If those dependencies preserve the relevant fields, prospective count is eight and prospective extent is thirty-two.
- The storage-size request is fifty-six: thirty-two plus sixty-four minus forty.
- If F-7eca6144d13b returns zero, count becomes eight, recorded extent becomes thirty-two, and the function returns zero.
- If that request returns nine, the function returns nine without its final count and extent writes. The earlier status write and dependency effects remain.

### Clipping suppresses the range operation

Use the same descriptor values, but position nine and adjustment five.

- Fourteen exceeds ten.
- Effective adjustment becomes two.
- The recomputed boundary is eleven, so the range-operation branch is skipped.
- Prospective count is eight and prospective extent is thirty-two.
- The storage-size request is fifty-six.
- On a zero result, count and recorded extent are committed.
- This function does not write status on this path.

### Negative adjustment with growth fallback

Assume count ten, element width four, recorded extent forty, capacity forty, position three, and adjustment negative two.

- The unsigned boundary sum is one.
- The range-operation branch runs with boundary one, retained count ten, and adjustment argument positive two.
- If the dependencies preserve relevant fields, prospective count is twelve and prospective extent is forty-eight.
- Delta-byte quantity is eight.
- Growth increment is forty, so the first storage-size request is eighty.
- If that request returns zero, the function commits twelve and forty-eight.
- If it returns nonzero, a second request for forty-eight follows.
- Only a nonzero result from that second request is propagated.

### Sentinel remains in later arithmetic

Assume count ten, position `0x7fffffff`, adjustment zero, element width zero, capacity zero, and status zero.

- Effective position is ten, so the initial checks pass.
- Later arithmetic still uses `0x7fffffff`.
- Clipping produces an effective adjustment of negative 2,147,483,636 under the documented unsigned-to-signed interpretation.
- The recomputed boundary is eleven, so the range operation is skipped.
- Prospective count is 2,147,483,646; prospective extent is zero.
- The positive count delta permits direct commit without a storage-size request.

This example is conditional on the out-of-range unsigned-to-signed conversion interpretation. It demonstrates why the sentinel must not be specified as a universal substitution with count.

## Uncertainties

### Documented_only: signed representation boundaries

The supplied representation does not establish target-specific results for signed overflow or out-of-range unsigned-to-signed conversion. Cases requiring those results cannot receive an unconditional portable-source acceptance outcome.

### Documented_only: dependency contracts

The internals of all three dependencies are unavailable. Their memory accesses, ownership rules, global effects, error meanings, and preservation guarantees are unknown.

### Documented_only: descriptor invariants

The evidence does not establish valid count ranges, nonnegative element width, consistency between count and recorded extent, or a relationship between recorded extent and capacity.

### Inferred intent

Collection resizing and element shifting are plausible interpretations. Index origin, exact range inclusivity, and the meaning of the extra one in clipping are not established as external API semantics.

## Evidence

This contract derives solely from the supplied static function body and opaque dependency mapping. The field offsets, branch conditions, explicit write order, dependency-call order, unsigned comparisons, and return paths are directly supported by that material.

Collection terminology and storage-management interpretations are labeled as inferred rather than treated as proven dependency behavior.

No implementation access or runtime tests have been performed. Acceptance examples are arithmetic consequences under their stated conditions, not observed executions.
