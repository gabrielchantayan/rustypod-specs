<!-- Trusted provenance: F-0dfab9a8e16d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0dfab9a8e16d

## Purpose

F-0dfab9a8e16d conditionally replaces the storage described by a three-word descriptor when its apparent capacity is below an unsigned requested count. Counts are scaled in eight-byte units.

The established behavior is a capacity check followed, when growth is required, by count selection, an allocation-related call, a relocation-related call, an empty traversal, a release-related call, a second descriptor query, and three ordered descriptor writes. A separate notification precedes the capacity check for requests above a fixed threshold.

The inferred intent is a reserve-style operation for contiguous storage. Neither the public container type nor the element semantics are established.

This contract is based on static evidence only. No runtime tests have been performed. Allocation, content preservation, ownership transfer, and dependency failure behavior remain conditional on contracts not supplied here.

## Interface

### Inputs and descriptor layout

The function receives:

- A pointer to a mutable descriptor.
- An unsigned 32-bit requested count.

The supplied representation uses 32-bit integer words. The descriptor contains at least three consecutive words:

| Byte offset | Contract name | Observed role |
|---|---|---|
| 0 | Start | Used in capacity arithmetic, passed to a dependency, and later replaced |
| 4 | Current end | Passed to a dependency, used as the traversal endpoint, and later replaced |
| 8 | Limit | Used in capacity and release-argument arithmetic, and later replaced |

These names describe usage, not independently verified pointer types. The body does not establish that the three values belong to one allocation or denote valid storage boundaries.

The descriptor pointer must be readable for the capacity check. A normally completing growth path also requires writable descriptor storage. No null check, bounds check, or descriptor validation is present.

For the inferred contiguous-storage interpretation, ordinary valid inputs would have ordered boundaries, an occupied range contained within the available range, and differences compatible with eight-byte units. These are inferred validity conditions, not checks performed by this function.

### Output

There is no explicit return value or status result. Observable results can include:

- Dependency calls and their effects.
- Replacement of all three descriptor words on the completed growth path.
- No direct descriptor writes when growth is not entered.

A notification is not itself an established rejection, exception, or return from this function.

### Arithmetic model

The displayed representation distinguishes signed and unsigned 32-bit arithmetic:

- The request, first query result, growth candidates, and selected count are treated as unsigned.
- Descriptor words, the saved allocation-related result, traversal state, and second query result are represented as signed integers.
- Address differences and traversal increments therefore cannot uniformly be interpreted as checked unsigned address arithmetic.

Unsigned additions and scaling retain their low 32 bits. Signed overflow, right shifts of negative values, and conversions outside the signed representable range require semantics not established by this body. Predictions for those cases remain **documented_only**.

## Behavior

### 1. Notify for an oversized request

Before reading the descriptor for the capacity check, the function compares the request with 0x1fffffff, decimal 536,870,911.

If the request exceeds that threshold, it calls F-5829ec5988c0 with:

1. The integer 8.
2. An opaque static address.
3. The same opaque static address.
4. The original requested count.

A request equal to the threshold does not trigger this call.

The threshold is also assigned to temporary storage on this branch, but this does not clamp the request. If the notification returns normally, the subsequent capacity comparison still uses the original request. If growth is entered, the temporary count is replaced by the growth calculation before selection.

The notification’s return behavior is unknown. It may prevent continuation, but the caller body does not establish that it does. If it returns, its effects on the descriptor or other state are not assumed absent.

### 2. Compare apparent capacity with the request

After any normally returning notification, the function reads the descriptor’s limit and start, subtracts start from limit, right-shifts the signed difference by three, and converts the shifted result to unsigned for comparison with the request.

For an ordinary nonnegative difference divisible by eight, the comparison quantity is the available span in eight-byte units. There is no explicit alignment or divisibility check. For a nonnegative difference that is not divisible by eight, the shift discards the low three bits.

The ordering of the individual start and limit reads within the subtraction is not separately established. Both belong to a check that occurs after the notification and before growth-path dependencies.

If the computed unsigned quantity is at least the request, the function returns without growth. On that path:

- The body performs no descriptor writes.
- It makes no descriptor-query, allocation-related, relocation-related, or release-related calls.
- The earlier notification may still have occurred.

Thus, “no growth” does not guarantee a side-effect-free invocation when the request exceeds the threshold.

### 3. Obtain the first query result and select a count

When apparent capacity is below the request, the function calls F-6e45b75056e5 with the descriptor pointer. Its result is held as an unsigned 32-bit value, denoted \(s\).

Two unsigned candidates are computed:

\[
a=(s+32)\bmod 2^{32}
\]

\[
b=\left(s+\left\lfloor s/2\right\rfloor+
\left\lfloor s/8\right\rfloor\right)\bmod 2^{32}
\]

The two shifted terms in \(b\) are calculated separately. Even without overflow, their sum need not equal a single rounding of \(13s/8\).

The function selects the larger candidate by unsigned comparison, then selects the request if that candidate is less than or equal to the request. The resulting count \(n\) is therefore:

\[
n=\max_{\text{unsigned}}(a,b,\text{request})
\]

The candidate arithmetic wraps before comparison. This is not a maximum over unbounded mathematical sums.

The query is not established to return size or capacity. If it returns the occupied element count, the inferred policy combines fixed and proportional growth based on that count. That interpretation does not establish query purity or descriptor preservation.

No subsequent check requires \(n\) to remain at or below 0x1fffffff.

### 4. Call the allocation-related dependency

The function calls F-871e64ddc6de with:

1. The selected count shifted left by three bits.
2. Zero.

The first argument is:

\[
(8n)\bmod 2^{32}
\]

This is unchecked scaling, not a verified allocation-size calculation. A selected count exceeding 0x1fffffff can produce a wrapped byte argument. This can occur even when the original request did not exceed the threshold, because the growth candidates are not threshold-limited.

The dependency’s result is saved as the prospective new start, denoted \(A\). The body does not test it for zero or another failure indication before using it.

Allocation semantics, alignment, ownership, and failure behavior are unknown.

### 5. Call the relocation-related dependency

Next, F-fd629c44bb6f receives:

1. The descriptor’s current start.
2. The descriptor’s current end.
3. The saved value \(A\).
4. The descriptor pointer.

The start and current-end arguments are read after the first query and allocation-related call. They are not an entry-time snapshot. The relative evaluation order of these two argument reads is not independently established.

The call is consistent with transferring an occupied range into replacement storage, but the body does not establish whether it copies, moves, destroys, transforms, retains, or releases anything. It also does not establish whether the dependency modifies the descriptor.

Subsequent reads use the state left by this call.

### 6. Traverse without explicit element operations

After relocation returns, the function samples the descriptor’s start. Denote this sampled value \(B\).

A traversal begins at \(B\), compares its current position for equality with the descriptor’s current end, and advances by eight while they differ.

The traversal body is empty. It contains no explicit payload read, payload write, destructor invocation, or callback. It must not be described as destruction.

The endpoint expression is part of the repeated condition rather than an explicitly saved endpoint. This does not establish a particular machine-load frequency or safe behavior under concurrent mutation.

Termination uses equality, not a less-than comparison. Ordinary completion requires reaching an equal endpoint through increments whose arithmetic remains valid. No guard detects an incompatible endpoint or signed overflow.

### 7. Call the release-related dependency

After traversal terminates, F-ff1bdc376dcd receives:

1. The sampled start \(B\).
2. The descriptor’s then-current limit minus \(B\), right-shifted by three.
3. Zero.

The second argument is derived through signed subtraction and shifting in the displayed representation. For a valid nonnegative span divisible by eight, it equals the span in eight-byte units. Its meaning to the dependency is unknown.

The first argument is not necessarily the entry-time start, nor necessarily the saved new start \(A\). It is the start sampled after relocation.

The release interpretation is inferred. This body does not prove that the dependency frees memory or that it preserves the descriptor, storage contents, or global state.

### 8. Query again and publish the descriptor

After the release-related call returns, the function calls F-6e45b75056e5 again with the descriptor pointer. It does not reuse the first query result.

Let the second result, as held in the displayed signed representation, be \(t\). The function then performs these writes in order:

1. Current end becomes \(A+8t\).
2. Start becomes \(A\).
3. Limit becomes the 32-bit word produced by \(A+8n\).

The first write uses signed multiplication and addition in the displayed representation. Results requiring signed overflow are not established.

The limit computation incorporates the unsigned selected count. Its unsigned arithmetic wraps to 32 bits; storing that result into the signed descriptor word introduces representation or conversion questions for values outside the signed range.

The function then returns. There is no final consistency check.

These writes are ordered but are not one atomic descriptor replacement. Intermediate states contain:

- The new current end with the previous start and limit.
- The new current end and start with the previous limit.

No synchronization making those states safe for concurrent observers is established.

## Side effects

### Established calls and writes

A normally completing growth sequence makes **five calls across four distinct dependencies**, in this order:

1. F-6e45b75056e5.
2. F-871e64ddc6de.
3. F-fd629c44bb6f.
4. F-ff1bdc376dcd.
5. F-6e45b75056e5 again.

It then writes current end, start, and limit in that order.

F-5829ec5988c0 is separate and conditional. If the request exceeds 0x1fffffff, that notification returns, and growth completes, the entire invocation makes **six calls across five distinct dependencies**.

If growth is not entered, the invocation makes either no dependency calls or only the oversized-request notification.

These counts describe calls made directly by this body, not any calls performed internally by dependencies.

### Ownership and storage lifetime

The body directly reads and writes descriptor fields but performs no explicit element-payload access.

Replacement of backing storage is inferred intent. The following are not established:

- Whether F-871e64ddc6de returns newly owned storage.
- Whether F-fd629c44bb6f consumes or retains either range.
- Whether F-ff1bdc376dcd releases its first argument.
- Whether any dependency preserves descriptor fields or global state.

The release-related call precedes both the second query and final publication. Safe lifetime handling therefore depends on unavailable dependency contracts.

### Errors and partial progress

There is no explicit returned error code, allocation-result check, rollback branch, or cleanup branch.

If a dependency fails to return normally, subsequent operations are not established as occurring. Earlier calls and their effects may already have happened.

The body alone provides no failure-atomicity guarantee and no proof that allocated storage is reclaimed after an interrupted growth sequence.

## Dependencies

| Opaque function ID | Established use | Unresolved contract |
|---|---|---|
| F-5829ec5988c0 | Conditional notification with 8, two identical static addresses, and the original request | Meaning of arguments, diagnostics, return behavior, state changes |
| F-6e45b75056e5 | Descriptor query before count selection and again after the release-related call | Meaning and stability of results, purity, descriptor/global effects |
| F-871e64ddc6de | Receives wrapped byte count and zero; result retained | Allocation semantics, alignment, failure convention, ownership |
| F-fd629c44bb6f | Receives current start, current end, saved result, and descriptor pointer | Content transfer, destruction, ownership, descriptor changes |
| F-ff1bdc376dcd | Receives post-relocation sampled start, shifted span, and zero | Release semantics, metadata requirements, preservation guarantees |

No dependency is assumed pure or assumed to preserve globals merely because of its inferred role.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests.

1. **Sufficient ordinary capacity.**  
   Given a readable descriptor with a valid available span of 80 bytes and a request of 10, growth is not entered. No dependency is called, and this body writes no descriptor fields.

2. **Zero request with zero capacity.**  
   Given equal start and limit values whose subtraction is valid, a request of zero does not enter growth. This does not establish acceptance of a null descriptor pointer.

3. **Fixed-increment growth dominates.**  
   Given that growth is required, a request of 6, and a first query result of 4, the candidates are 36 and 6. The selected count is 36. F-871e64ddc6de receives 288 and zero.

4. **Proportional growth or request dominates.**  
   Given growth and a first query result of 64, the candidates are 96 and 104. A request of 90 selects 104 and yields a byte argument of 832. A request of 120 selects 120 and yields 960.

5. **The second query independently determines current end.**  
   If the selected count is 104, the saved allocation-related result is \(A\), and the second query returns 60, then, provided the relevant signed arithmetic is valid, final values are current end \(A+480\), start \(A\), and limit \(A+832\). Their write order remains current end, start, limit.

6. **Oversized notification does not clamp.**  
   A request of 0x20000000 triggers F-5829ec5988c0. If it returns, growth is required, and the first query returns zero, the selected count is 0x20000000 and the allocation-related byte argument is zero after 32-bit scaling. This does not establish successful zero-byte allocation.

7. **Dependency-call counts distinguish notification from growth.**  
   A completed growth invocation with a request at or below 0x1fffffff makes five direct calls across four distinct dependencies. With an above-threshold request and a normally returning notification, completed growth makes six direct calls across five distinct dependencies.

8. **Relocation changes subsequent release arguments.**  
   If relocation leaves start \(B\), current end \(B+16\), and limit \(B+80\), with valid arithmetic and no intervening mutation, the empty traversal advances twice. The release-related call receives \(B\), 10, and zero, regardless of entry-time boundaries.

9. **An incompatible endpoint does not imply early exit.**  
   If the endpoint cannot be reached by valid eight-byte increments from the sampled start, normal traversal completion is not established. The contract must not replace equality with an ordering comparison or promise wraparound termination for signed traversal arithmetic.

## Uncertainties

The following remain **documented_only**:

- Exact ABI and target-specific signed arithmetic, shift, and conversion behavior.
- Public descriptor and element types.
- Whether either query measures occupied units, and whether the results can differ.
- Notification content and whether it prevents oversized requests from continuing.
- Dependency allocation, relocation, destruction, release, failure, and exception contracts.
- Ownership and content preservation after completion or interruption.
- Behavior for malformed descriptors, inaccessible memory, signed overflow, and concurrent mutation.
- Actual machine-load frequency for the repeated endpoint expression.

Reserve-style growth is inferred intent. It does not override the established call ordering, arithmetic widths, repeated descriptor reads, or unresolved dependency effects.

## Evidence

Established static evidence consists of the supplied function body: the threshold comparison, three-word descriptor accesses, signed capacity expression followed by unsigned comparison, unsigned growth calculations, eight-byte scaling, dependency arguments, empty traversal, repeated query, and ordered descriptor writes.

The supplied dependency mapping establishes the opaque identities used here, not callee behavior.

The growth sequence contains four distinct dependencies and five direct calls because F-6e45b75056e5 is invoked twice. The conditional notification adds one call and one distinct dependency only when its branch is taken.

No runtime execution, dependency inspection, allocation-failure exercise, or concurrency test has been performed.
