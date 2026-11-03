<!-- Trusted provenance: F-58f0a7ebbb25; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-58f0a7ebbb25

## Purpose

F-58f0a7ebbb25 updates two adjacent 32-bit fields in a caller-supplied memory object. It combines an incoming adjustment with the field at byte offset 12, selects a dependency argument according to whether that combined value is negative, and uses the dependency’s result to update both fields.

The established behavior is an ordered, in-place state transformation involving two dependency calls. The supplied evidence does not establish the application-level meaning of either field.

A possible intent is to maintain a primary quantity and a residual quantity, with the dependency computing a correction. This interpretation is inferred only. It does not establish division, normalization, rounding, units, bounds, or any invariant that callers may rely on.

## Interface

### Explicit inputs

The supplied representation exposes four inputs:

| Input | Represented width | Established role |
|---|---:|---|
| Object address | 32 bits | Base address for 32-bit field accesses at byte offsets 8 and 12. |
| Primary adjustment | 32 bits, signed interpretation | Added to the field at offset 8 during its final update. |
| Residual adjustment | 32 bits, signed interpretation | Added to the field at offset 12 before branch selection; also forwarded to F-870c4fcf7295. |
| Opaque forwarded word | 32 bits | Forwarded to F-870c4fcf7295 without a visible interpretation in this function. |

The address is represented as an integer rather than a typed object pointer. This does not establish that arbitrary integer values are valid addresses.

### Object layout

Only these fields have established direct uses:

| Byte offset | Access width | Direct use |
|---:|---:|---|
| 8 | 32 bits | Read and replaced with an additive update. |
| 12 | 32 bits | Read to form a retained intermediate value, then replaced using that value and a computed correction. |

No other object fields are directly accessed in the supplied representation. Dependencies may access additional memory; their effects are unknown.

For an ordinary successful execution, both accessed locations must support the represented reads and writes. A contiguous allocation containing the first 16 bytes is one sufficient arrangement, but the evidence does not establish the object’s actual allocation size, type, or alignment requirement.

### Additional incoming context

The call to F-870c4fcf7295 includes four additional 32-bit values obtained from incoming execution context. The function does not visibly initialize these values itself. One is represented as incoming return-link context.

These values are not established as ordinary caller-facing parameters. Their calling-convention meaning, validity, and significance to the dependency are documented_only unknowns.

### Output and ownership

There is no explicit return value. The visible results are the two memory writes and any effects caused by dependencies.

No allocation, deallocation, ownership transfer, or reference-count operation appears directly in this function. This does not establish that either dependency preserves ownership, object lifetime, or memory validity.

## Behavior

### 1. Initial dependency call

F-2f4a6c4b80c2 is called before either object field is visibly read.

No explicit arguments are shown, and no return value is used. Its effects are unknown. In particular:

- The fields must not be assumed to retain their entry values across this call.
- The external values used later must not be assumed to retain their entry values.
- The object must still be valid when subsequent field accesses occur.
- If this dependency does not return normally, the remaining operations are not reached.

Calling it an initializer, lock, validator, or state-preserving helper would exceed the available evidence.

### 2. Formation of the combined value

After the first dependency returns, the function reads the signed 32-bit field at offset 12 and adds the residual adjustment.

Call this retained result the **combined value**. Its sign selects the next operation:

- A value strictly below zero selects the negative branch.
- Zero and positive values select the nonnegative branch.

The comparison is against the combined value, not against either input separately. A positive adjustment may select the negative branch, and a negative adjustment may select the nonnegative branch, depending on the field value.

The function retains this combined value for the final field replacement. It does not visibly reread offset 12 after F-870c4fcf7295 returns.

### 3. Branch-dependent dependency call

F-870c4fcf7295 is called once on either branch, with eight represented argument positions:

1. The combined value.
2. One branch-selected external value.
3. The original residual adjustment.
4. The opaque forwarded word.
5. Through 8. The four incoming context words described above.

For clarity, the external value selected by the negative branch is called the **negative selector**, and the one selected by the nonnegative branch is called the **nonnegative selector**.

These names express selection only. Neither selector’s contents, mathematical meaning, relationship to the other, or stability across calls is known.

The selected value is obtained for the dependency invocation after F-2f4a6c4b80c2 has returned. The evidence does not establish a snapshot of external values at function entry.

The dependency supplies a result treated here as a signed 32-bit value:

- On the nonnegative branch, that result becomes the **correction** unchanged.
- On the negative branch, the correction is the negation of the dependency result plus one:  
  \[
  \text{correction}=-(\text{dependency result}+1).
  \]

The negative transformation is not merely a sign reversal. For example, a dependency result of zero becomes −1, not zero.

No success indicator is checked. Every normally returned result proceeds into the arithmetic updates, subject to the arithmetic limitations described below.

### 4. Update of the field at offset 8

After the second dependency returns and the correction is determined, the function reads the field at offset 8.

It writes back the sum of:

- That newly read field value.
- The correction.
- The primary adjustment.

Thus, absent arithmetic overflow:

\[
\text{new field at offset 8}
=
\text{field at offset 8 read after the dependency}
+\text{correction}
+\text{primary adjustment}.
\]

This timing is significant. If F-870c4fcf7295 changes the field at offset 8, the update incorporates the changed value rather than a value captured before that call.

### 5. Replacement of the field at offset 12

After the offset-8 write, the function writes the field at offset 12 using:

- The retained combined value.
- A third external value, called the **scale factor**.
- The correction.
- The literal multiplier 128.

Absent arithmetic overflow, the replacement is:

\[
\text{new field at offset 12}
=
\text{combined value}
+\text{scale factor}\times\text{correction}\times128.
\]

This is a replacement, not an addition to the current contents of offset 12. If the second dependency changes that field, the dependency’s replacement is not reread for this calculation.

The scale factor is referenced in the final update, after the dependency call and after the offset-8 write. It must not be assumed to equal its entry-time value. If it aliases memory written earlier, that relationship may affect the value observed; the available evidence does not exclude aliasing.

### Arithmetic boundaries

The fields, explicit integer adjustments, combined value, and dependency result are represented as 32-bit quantities with signed uses. No widened intermediate calculation is shown. The literal 128 does not establish wider arithmetic.

Potential overflow points include:

- Adding the residual adjustment to the offset-12 field.
- Adding one to the dependency result on the negative branch.
- Negating that incremented result.
- Adding the correction and primary adjustment to offset 8.
- Multiplying the scale factor by the correction and then by 128.
- Adding the resulting product to the combined value.

The supplied representation alone does not establish an authoritative overflow policy. In particular, wraparound, trapping, saturation, and checked-error behavior must not be invented.

The acceptance examples below keep every intermediate within the signed 32-bit range, −2,147,483,648 through 2,147,483,647. Overflow cases remain documented_only until authoritative execution semantics are available.

## Side effects

### Established direct ordering

For a normally completing invocation, the visible sequence is:

1. Call F-2f4a6c4b80c2.
2. Read offset 12.
3. Select a branch-specific external value and call F-870c4fcf7295.
4. Read offset 8.
5. Write offset 8.
6. Obtain the scale factor for the final expression and write offset 12.
7. Return without an explicit result.

This establishes ordering between the represented operations. It does not establish hardware memory barriers, volatile semantics, synchronization, or atomicity.

### Partial updates and failure

The two writes are not presented as a transaction. The offset-8 write precedes the offset-12 write, and no rollback is shown.

Consequently, if execution fails after the first write but before the second completes, the representation provides no mechanism that restores the earlier state. No error return, exception handling, null check, bounds check, or arithmetic-error check is visible.

Dependency effects may occur before any direct object write. A failing invocation therefore must not be assumed side-effect-free.

### Concurrency and aliasing

No locking or synchronization mechanism is established. The first dependency could provide one, but that is unknown rather than a supported guarantee.

The evidence also does not establish that the object fields, external selectors, and scale factor occupy disjoint storage. The ordering above matters if they alias. Independent-field interpretations require an explicit non-aliasing assumption.

## Dependencies

| Opaque function ID | Established use | Unknown contract |
|---|---|---|
| F-2f4a6c4b80c2 | Called first, without visible explicit arguments; result unused. | Memory effects, ownership effects, synchronization, failure behavior, and return guarantees. |
| F-870c4fcf7295 | Called once with the combined value, a branch-selected external value, the residual adjustment, the opaque word, and four context words; result determines the correction. | Mathematical operation, argument interpretation, memory effects, ownership effects, failure behavior, and valid input domain. |

The three external data dependencies are the negative selector, nonnegative selector, and scale factor. Their numeric contents and mutability are not supplied. They must not be assigned names implying divisors, moduli, or conversion units.

No assumption that either dependency preserves globals or object storage is part of this contract.

## Acceptance examples

These are conditional behavioral examples, not observed runtime results. Unless stated otherwise, assume valid memory, normal dependency returns, no relevant aliasing, a scale factor of 1, and no dependency mutation of the object.

### Nonnegative combined value

Suppose the field at offset 12 is 10 after the initial dependency, the residual adjustment is 5, and the primary adjustment is 7. The field at offset 8 is 100 when read.

The combined value is 15, so F-870c4fcf7295 receives 15 with the nonnegative selector. If it returns 2:

- The correction is 2.
- Offset 8 becomes 109.
- Offset 12 becomes 271: 15 plus 2 multiplied by 128.

### Negative combined value

Suppose offset 12 is −10 after the initial dependency, the residual adjustment is 3, and the primary adjustment is 4. Offset 8 is 100 when read.

The combined value is −7, so the negative selector is used. If the dependency returns 2:

- The correction is −3.
- Offset 8 becomes 101.
- Offset 12 becomes −391.

This distinguishes the negative transformation from a simple negation.

### Exact zero boundary

Suppose offset 12 is −5 and the residual adjustment is 5. The combined value is zero, so the nonnegative selector is required.

If the dependency returns zero, offset 12 becomes zero, and offset 8 changes only by the primary adjustment. Zero does not select the negative branch.

### Zero dependency result on the negative branch

Suppose the combined value is −1 and the dependency returns zero. The correction is −1.

With a primary adjustment of zero, offset 8 decreases by one, and offset 12 becomes −129. Treating a zero dependency result as “no correction” would contradict the established negative-branch behavior.

### Dependency mutations

Suppose the combined value has already been formed as 4. During F-870c4fcf7295, the dependency changes offset 8 to 200, offset 12 to 999, and the scale factor to 3, then returns 1. Let the primary adjustment be 2.

The resulting fields are:

- Offset 8: 203, because its read occurs after the dependency.
- Offset 12: 388, because the function uses retained 4 plus 3 multiplied by 1 multiplied by 128, not the dependency-written 999.

## Uncertainties

The following remain documented_only:

- The application meaning, units, and invariants of the two fields.
- The contents and storage relationships of the three external values.
- Dependency argument semantics and memory or ownership effects.
- The provenance and ABI meaning of the four incoming context words.
- Overflow behavior and target-specific arithmetic details.
- Address validity, required alignment, and behavior on invalid accesses.
- Concurrency guarantees, asynchronous observation, and memory-ordering semantics.
- Whether abnormal dependency exits or memory faults have any externally defined recovery behavior.

None of these unknowns is resolved by inferred intent.

## Evidence

The supplied static representation establishes the two calls, sign-dependent selection, negative-result transformation, field offsets, represented widths, arithmetic expressions, and direct access ordering described above.

The normalization interpretation is inferred intent only. The dependency internals, external data contents, and authoritative overflow semantics are absent.

No runtime tests have been performed. The acceptance examples are conditional calculations derived from the visible behavior, not execution evidence.
