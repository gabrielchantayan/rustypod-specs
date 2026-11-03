<!-- Trusted provenance: F-8c9ebb9ecebf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8c9ebb9ecebf

## Purpose

F-8c9ebb9ecebf performs a bounded transfer from a byte-addressed region described by a context object. It selects an offset either directly or relative to a stored position, may reduce a caller-supplied transfer count, invokes an external operation, and then writes a new position into the context.

The established behavior is narrower than a general-purpose read API:

- Only two origin selectors are accepted.
- An offset at the region’s recorded length is rejected, including for a zero-count request.
- An offset with its high bit set is rejected after any relative-position addition.
- The count-bound calculation uses 32-bit modular arithmetic. It does not reliably reject or clamp every mathematically oversized count.
- The final position uses the count read **after** the external operation returns, not necessarily the count passed to that operation.
- The external operation’s return value does not determine the reported status.

A memory-copy or stream-read interpretation is inferred intent. The external operation’s actual semantics are not established.

## Interface

### Inputs

The recovered interface has five arguments:

| Argument | Role | Width and interpretation |
|---|---|---|
| Context address | Identifies an object containing a source address, region length, and current position | Address-sized value represented in the recovered 32-bit model |
| Transfer operand | Passed unchanged as the first argument to the external operation | Opaque 32-bit value |
| Count address | Identifies caller-accessible count storage | Address of a 32-bit quantity |
| Origin selector | Selects direct or current-position-relative addressing | 32-bit integer |
| Requested offset | Supplies the direct offset or relative displacement bit pattern | 32-bit unsigned quantity; subsequently checked as signed |

The count storage is shown as an integer object, but its participation in arithmetic with unsigned offsets causes its bit pattern to be interpreted as unsigned in those expressions. This distinction matters for counts whose high bit is set.

No validation of the transfer operand, context address, count address, or source allocation is visible.

### Context layout

Only these fields are established:

| Byte offset from context address | Width | Meaning in this function |
|---|---|---|
| +8 | 32 bits | Source-base address representation |
| +12 | 32 bits | Recorded region length, compared as unsigned |
| +16 | 32 bits | Stored current position |

The function does not establish the context’s full size, alignment requirements, other fields, allocation type, or ownership.

The source address supplied to the external operation is formed by adding the effective offset to the source-base representation. In the recovered arithmetic model, this addition is 32-bit modular arithmetic. No check establishes that the resulting address belongs to a valid allocation.

### Outputs

If execution returns normally, the function returns one of three 32-bit status values:

| Status | Established condition |
|---|---|
| 0 | The external operation returned and the final position write completed |
| 5 | The effective offset is greater than or equal to the recorded region length |
| 6 | The origin selector is unsupported, or the effective offset has its high bit set |

These meanings describe this function’s branches, not a verified mapping to a wider error-code system.

The function can also modify:

- The 32-bit count object, if its clamp condition is satisfied.
- The context position at byte offset +16, after the external operation returns.
- Any state modified by the external operation.

No transferred-byte count is returned separately.

## Behavior

### 1. Select the effective offset

Origin selector **0** selects direct addressing. The effective offset initially equals the requested offset. The stored current position is not read for this selection.

Origin selector **1** selects relative addressing. The function first reads the 32-bit current-position field at context offset +16 and adds it to the requested offset. The resulting effective offset is the low 32 bits of that sum.

Every other origin selector returns status 6 immediately. This rejection occurs before any explicit context-field access, count access, or external invocation.

The relative calculation is not a checked mathematical addition. Overflow does not itself produce an error. For example, adding the bit pattern representing −1 to a stored position of 10 produces an effective offset of 9.

### 2. Reject effective offsets with the high bit set

The effective offset is interpreted as a signed 32-bit value for this check. If it is negative under that interpretation, the function returns status 6.

Equivalently, only effective offsets from 0 through 2,147,483,647 pass this check.

For direct addressing, this rejection requires no context-field read. For relative addressing, the current-position read has already occurred.

This check examines the result after wrapping arithmetic. It does not establish that the original requested offset represented a nonnegative displacement.

### 3. Check against the recorded region length

The function reads the 32-bit length field at context offset +12 and retains that value for the remaining local bounds decisions.

If the recorded length is less than or equal to the effective offset, the function returns status 5.

Consequences:

- The effective offset must be strictly less than the recorded length.
- A zero-length region has no accepted offset.
- An offset exactly at the end is rejected even when the requested count is zero.
- This return path does not read or write the count object.
- This return path does not invoke the external operation or explicitly update the position.

The length comparison is unsigned. A length with its high bit set is not rejected merely for that reason.

### 4. Conditionally reduce the count

After the offset passes the length check, the function reads the count object and adds its 32-bit bit pattern to the effective offset. This addition wraps modulo 4,294,967,296.

The function compares that wrapped sum against the cached length. If the length is strictly smaller than the sum, it writes the remaining length—recorded length minus effective offset—into the count object.

Because the offset is already known to be strictly below the length, this subtraction produces a nonzero unsigned remainder without unsigned underflow.

The condition is specifically a comparison against a **wrapped sum**, not an overflow-safe comparison of count against remaining length. Therefore:

- A request whose mathematical end exceeds the region can escape clamping if the addition wraps to a sufficiently small value.
- A count with its high bit set is not explicitly rejected.
- A sum exactly equal to the length does not cause a count write.
- A zero count does not cause a count write when the count remains unchanged during this phase.

The function does not otherwise normalize or validate the count.

### 5. Invoke the external operation

The function invokes F-dependency-01 with three arguments:

1. The transfer operand, unchanged.
2. The source-base representation read from context offset +8, plus the effective offset, using the recovered 32-bit address arithmetic.
3. The current contents of the count object.

The count is read again for this invocation. It is not simply the earlier count value retained from the clamp comparison.

The source-base read occurs after any local clamp write. This ordering matters when the count object aliases the source-base field.

The supplied representation does not establish a relative evaluation order between the source-base read and the count read used to prepare the call arguments. A contract must not require one of those two reads to precede the other.

The external operation’s return value is ignored. No local branch interprets it as success, failure, or a byte count.

### 6. Update the position and return

After F-dependency-01 returns, the function reads the count object again. It adds that value to the effective offset modulo 4,294,967,296 and writes the result to context offset +16.

The effective offset used here is the value calculated before the external invocation. The count, however, is read after that invocation.

Thus, if the external operation changes the count object, the final position reflects the changed count. If it changes the stored position, the explicit final position write replaces that change when execution reaches the write.

The function then returns status 0.

There is no post-call recheck against the recorded length, no reread of the length for validation, and no comparison between the count supplied to the external operation and the count used for the final position.

## Side effects

### Explicit memory-access order

For normally accessible, nonvolatile storage, the established local sequence is:

1. Read the current position only for origin selector 1.
2. Reject a high-bit-set effective offset if applicable.
3. Read and cache the region length.
4. Reject an offset at or beyond that length if applicable.
5. Read the count for the clamp decision.
6. Write a reduced count if the clamp condition holds.
7. Read the source base and read the count to prepare the external invocation; their relative order is not established.
8. Invoke F-dependency-01.
9. Read the count after the invocation returns.
10. Write the final position.
11. Return status 0.

Unsupported-origin rejection precedes all these memory accesses.

### Aliasing

No non-aliasing guarantee is visible.

If the count object overlaps context fields, ordinary “input count” and “context state” roles are not independent:

- A clamp write can modify the source base before its call-time read.
- A clamp write can modify the stored length, but the local comparison logic continues using the previously read length.
- A count object overlapping the position field can be affected by the final position write.
- Relative addressing reads the old position before any count clamp.

These are consequences of the visible access sequence, not evidence that overlapping objects are supported by the surrounding API.

### Ownership and external effects

No explicit allocation, deallocation, ownership transfer, or resource release appears in the local behavior.

That absence does not establish what F-dependency-01 does. Its treatment of the transfer operand, source region, count storage, context, global state, and resource ownership remains unknown.

The function supplies no local rollback if the external operation changes state and then fails to return normally. It also supplies no visible exception, fault, or nonlocal-exit handling.

## Dependencies

### F-dependency-01

This is a document-local opaque function ID for the sole external call. Its canonical identity is not supplied.

**Established call contract:**

- Called only after origin, sign, and offset-versus-length checks pass.
- Receives the unchanged 32-bit transfer operand.
- Receives the computed source address.
- Receives the call-time 32-bit count bit pattern.
- Has its return value ignored.
- Must return normally for the local final-position write and status-0 return to occur.

**Unknown contract:**

- Whether it copies data, and in which direction.
- Whether its third argument is interpreted as signed or unsigned.
- Whether zero-count invocation accesses either operand.
- Whether overlapping source and destination regions are supported.
- Whether it preserves context fields, count storage, globals, or other reachable state.
- Whether it allocates, releases, retains, or transfers ownership of memory.
- How it reports failures or whether it can terminate execution nonlocally.

A conventional transfer operation is plausible inferred intent, not an established dependency guarantee.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, assume valid accessible storage, no overlapping objects, no concurrent mutation, and an external operation that returns without changing the count or context.

### Direct transfer within the region

Given length 100, origin selector 0, requested offset 20, and count 10:

- The effective offset is 20.
- The count remains 10.
- The external operation receives the source base plus 20 and count 10.
- The final position becomes 30.
- The return status is 0.

### Clamp at the end

Given length 100, direct offset 90, and count 20:

- The count is changed to 10 before the external invocation.
- The invocation receives count 10.
- The final position becomes 100.
- The return status is 0.

### Exact end rejected, including zero count

Given length 100 and direct offset 100:

- The function returns 5.
- The count object is not accessed.
- No external invocation or explicit position write occurs.

This remains true when the supplied count would have been zero.

### Zero count at an accepted offset

Given length 100, direct offset 20, and count 0:

- No clamp write occurs.
- The external operation is still invoked with count 0.
- If it leaves the count unchanged, the final position becomes 20.
- The return status is 0.

No claim is made that the external operation itself treats this as a no-op.

### Relative negative displacement represented as bits

Given position 10, origin selector 1, requested-offset bit pattern 4,294,967,295, length 100, and count 2:

- The relative addition wraps to effective offset 9.
- The request passes the sign check.
- The invocation receives source base plus 9 and count 2.
- The final position becomes 11.

### High-bit-set effective offset

Given direct offset 2,147,483,648:

- The function returns 6 before reading context fields or count storage.

For relative addressing that produces the same effective offset, the position field has already been read before rejection.

### Wrapped count bypasses clamping

Given length 100, direct offset 20, and count bit pattern 4,294,967,286:

- The count-plus-offset sum wraps to 10.
- The clamp condition is false.
- The external operation receives the unchanged count bit pattern.
- If that count remains unchanged, the final position becomes 10.

This example establishes arithmetic behavior, not that such an invocation is safe.

### External modification of count

Given length 100, direct offset 20, and initial count 10, suppose the external operation changes the count object to 4 before returning:

- The call was supplied count 10.
- The final position becomes 24.
- The function returns 0 regardless of the external operation’s ordinary return value.

## Uncertainties

- **documented_only — Machine representation:** The contract uses the recovered 32-bit integer and address model. An independent ABI description, endianness, alignment rules, and original type declarations are unavailable.
- **documented_only — External semantics:** Transfer direction, failure behavior, memory ownership, and state preservation are not established.
- **documented_only — API validity rules:** The surrounding API may prohibit aliasing, negative count bit patterns, or unusual context values, but no such restrictions are visible here.
- **documented_only — Memory validity:** Recorded length is not proof of allocation size or address accessibility. Pointer arithmetic can wrap, and no local check prevents that.
- **documented_only — Concurrency:** No synchronization or atomicity contract is established. The ordered description does not promise coherent behavior under concurrent mutation.
- **Inferred intent:** The context fields resemble a source region, its length, and a current position. The count reduction resembles a bounded-read clamp.
- **Not established:** Status 0 does not prove successful data transfer, correct destination contents, or preservation of any state by the external operation.

## Evidence

The evidence consists solely of the supplied recovered function representation. It establishes the branch conditions, field offsets, local arithmetic, explicit accesses, external call arguments, ignored external return value, and three normal return values described above.

No dependency implementation, caller contract, original declarations, executable observations, or runtime tests were provided. No runtime tests have been performed. Acceptance examples are deductions from the static representation under their stated assumptions.
