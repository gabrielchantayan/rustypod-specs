<!-- Trusted provenance: F-f2ca48d02e9d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f2ca48d02e9d

## Purpose

F-f2ca48d02e9d coordinates a count-limited transfer using a mutable context, a caller-supplied count location, and a destination address. It first checks a context flag, then conditionally prepares the context, consumes an apparent buffered region, and may invoke another operation before completing the remaining transfer.

The established behavior is the sequence of field accesses, dependency calls, arithmetic, status propagation, and count updates described below. Interpreting the dependencies as preparation, copying, refill, and direct transfer is inferred intent, not an established guarantee of their implementations.

This is not an all-or-nothing operation. The function can advance a context field and ultimately report a partial amount before returning a failure status. It does not visibly restore earlier state changes.

## Interface

### Inputs

The interface has four arguments:

1. **Context address.** Identifies mutable storage with the following accessed layout.
2. **Count address.** Identifies a readable 32-bit unsigned request count and, on certain paths, the location receiving a result count.
3. **Destination address.** A 32-bit address value passed to transfer dependencies and advanced on one path.
4. **Auxiliary 32-bit unsigned value.** Assigned to temporary storage on the nonzero-request path. No visible operation subsequently consumes that original value before the storage is overwritten. Its intended role is unknown.

The context fields accessed directly are:

| Byte offset | Width | Established use |
|---|---:|---|
| +4 | 1 byte | Zero/nonzero gate |
| +8 | 4 bytes | First argument to F-edf0d9c5fd88 |
| +16 | 4 bytes | Unsigned threshold compared with the remaining request |
| +24 | 4 bytes | Apparent current-position address; read and advanced |
| +32 | 4 bytes | Used with the current-position field to calculate an inclusive span |

The field names describe their use in this function, not independently established context semantics.

### Return value

The return value is a signed status represented in 32 bits:

- **−50** when the gate byte is nonzero.
- **0** when the gate byte is zero and the initial requested count is zero.
- The nonzero status from F-8d88f6e0df46 if that dependency fails.
- The status from F-a78a7790af0e after the initial-region path reaches that dependency.
- If F-a78a7790af0e succeeds and F-edf0d9c5fd88 is selected, the latter dependency’s status replaces it.

The return values, if any, of F-70a021fc7c24 are not inspected.

### Representation and preconditions

The supplied representation uses 32-bit integers and address values, with unsigned comparisons for request sizes and the threshold. The gate access is one byte.

The count address must be readable even when the gate will reject the operation: the count is read first. The context must permit access to the gate byte. Additional context fields and destination storage must be valid as required by the path and dependencies actually reached.

No null checks, range checks, overlap checks, address-overflow checks, or context-consistency checks are visible. Validity requirements imposed by dependencies remain unknown.

## Behavior

### 1. Capture the request and inspect the gate

The function first reads the 32-bit unsigned count through the count address. It then reads the context’s gate byte at offset +4.

If the gate byte is nonzero:

- Return −50.
- Do not explicitly write the count.
- Do not call a dependency.
- Do not explicitly modify the context.

The gate check takes precedence over the zero-request shortcut. A zero request with a nonzero gate still returns −50.

If the gate byte is zero and the captured request is zero:

- Return 0.
- Do not call a dependency.
- Do not explicitly write the count or context.

These paths do not establish that either argument can be invalid: both the count read and the gate read occur.

### 2. Prepare a nonzero request

For a zero gate and nonzero captured request, the function assigns the auxiliary argument to temporary storage, then calls:

- **F-8d88f6e0df46**, with the context address and a zero-valued second argument.

If this call returns nonzero, that status is returned without an explicit result-count write by this function.

If it returns zero, the function proceeds using context fields read after the call. It does not rely on the pre-call values of the current-position and end-related fields.

Preparation may have side effects of its own. A failure here does not imply that the context or other reachable storage is unchanged.

### 3. Calculate the initial span

Following successful preparation, the function calculates an unsigned 32-bit span from the fields at offsets +32 and +24:

**span = end-related value − current-position value + 1**

The addition of one makes the apparent endpoint inclusive. For example, equal field values produce a span of one, not zero.

The captured request is compared with this span using a strict unsigned less-than comparison. Equality takes the initial-region exhaustion path below.

The supplied representation does not establish the relative evaluation order of the two field reads within this arithmetic expression.

### 4. Request smaller than the initial span

If the captured request is strictly smaller than the span:

1. Call **F-70a021fc7c24** with three explicit arguments:
   - The original destination address.
   - The value read from context offset +24 for this call.
   - The captured request.
2. After the call, read context offset +24 again.
3. Write that newly read value plus the captured request back to offset +24.
4. Write the captured request through the count address.
5. Return 0.

The post-call reread matters. If the dependency changes the current-position field, the explicit increment is applied to the changed value rather than to a saved pre-call address.

The explicit result count is the requested amount regardless of any unobserved return value from F-70a021fc7c24. Whether the dependency actually performs that amount of transfer is outside the supplied evidence.

### 5. Request equal to or larger than the initial span

Otherwise, the function stores the calculated span in temporary count storage and invokes **F-70a021fc7c24**.

This invocation has only **two explicit arguments in the supplied representation**:

- The original destination address.
- The value read from context offset +24.

A third argument must not be invented to make this call resemble the other invocations. Its effective calling convention and transfer extent are unresolved.

After this call, the function:

1. Takes the temporary count as the initial accounted amount.
2. Advances its working destination address by that amount.
3. Rereads context offset +24 and advances that field by the same amount.
4. Subtracts that amount from the captured request.
5. Calls **F-a78a7790af0e** with the context address.

The context advancement therefore precedes F-a78a7790af0e, including when that dependency returns a failure.

If F-a78a7790af0e returns nonzero:

- Do not attempt the remaining-transfer step.
- Write the initial accounted amount through the count address.
- Return the dependency’s failure status.

If it returns zero, continue below.

### 6. Complete the remaining request

After successful F-a78a7790af0e, the function initializes temporary count storage to the remaining request. It then reads the unsigned threshold at context offset +16.

#### Remaining request exceeds the threshold

If the threshold is strictly less than the remaining request, call **F-edf0d9c5fd88** with:

1. The four-byte value read from context offset +8.
2. The address of temporary count storage.
3. The advanced destination address.

The dependency receives a writable count location initialized to the remaining request.

After it returns:

- Use the dependency’s return status as this function’s status.
- Add the resulting temporary count to the initial accounted amount.
- Write that sum through the caller’s count address.
- Return the status.

The count is accumulated and written even when F-edf0d9c5fd88 returns a nonzero status. No visible clamp constrains the returned temporary count to the remaining request.

This branch performs no additional explicit current-position update after F-edf0d9c5fd88. Any context changes performed by that dependency are unknown.

#### Remaining request does not exceed the threshold

If the remaining request is less than or equal to the threshold:

1. Call **F-70a021fc7c24** with:
   - The advanced destination address.
   - The current-position value read from context offset +24.
   - The remaining request.
2. Reread context offset +24.
3. Advance that field by the temporary count, which was initialized to the remaining request.
4. Add that count to the initial accounted amount.
5. Write the total through the caller’s count address.
6. Return 0.

There is no zero-length shortcut here. If the original request exactly equals the initial span, successful F-a78a7790af0e is followed by this branch with a remaining request of zero, including a three-argument F-70a021fc7c24 invocation with a zero count.

## Side effects

### Explicit writes

The function explicitly writes only:

- The four-byte current-position field at context offset +24.
- The caller’s four-byte count location.
- Its own temporary storage.

Destination writes are delegated. The supplied evidence does not independently establish their extent or contents.

On every path reaching successful preparation, the final count write occurs after all selected dependency calls and explicit current-position updates. On gate rejection, a zero request, or preparation failure, this function performs no explicit count write.

### Aliasing and ordering

No disjointness checks exist.

If the count location overlaps context storage, the final count write occurs after the context updates and can overwrite their results. If it overlaps destination storage, the final count write can overwrite destination bytes after dependency activity.

The original request is captured before any dependency call. The function does not reload it from the caller’s count location for later branch decisions. A dependency’s change to that location does not replace the captured request, although the function may later overwrite the location with its result count.

Context values are not generally preserved across calls. In particular:

- The initial span uses values obtained after preparation.
- Explicit current-position increments use post-call rereads.
- The threshold and backend argument are read after successful F-a78a7790af0e.

No atomicity or synchronization is visible. Concurrent modification is not supported by an established contract.

### Ownership and lifetime

No explicit allocation or release occurs in this function. That does not establish allocation, release, or ownership behavior inside dependencies.

F-edf0d9c5fd88 receives an address to temporary storage used during this invocation. Whether it retains that address is unknown; the supplied function provides no persistent storage for it.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| **F-8d88f6e0df46** | Called once for an accepted nonzero request, with the context and zero. Nonzero return terminates the path before an explicit result-count write. |
| **F-70a021fc7c24** | Invoked with three explicit arguments on the short-initial-region and final buffered paths. Invoked with only two explicit arguments on the initial-region exhaustion path. Return status is not inspected. |
| **F-a78a7790af0e** | Called after the initial span has been accounted for and the current-position field advanced. Its return controls whether remaining work is attempted. |
| **F-edf0d9c5fd88** | Selected only after successful F-a78a7790af0e when the remaining request exceeds the threshold. Receives a writable temporary count; both its resulting count and status are consumed. |

No dependency is assumed to preserve globals, context fields, destination contents, or other reachable memory. Their names here imply no ownership guarantees.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Gate rejection precedes the zero shortcut.**  
   Given a readable count containing zero and a nonzero gate byte, return −50. Make no dependency calls and perform no explicit count write.

2. **Accepted zero request.**  
   Given a zero gate and a captured request of zero, return 0 without preparation or transfer calls.

3. **Preparation failure.**  
   Given a zero gate, a nonzero request, and preparation returning status 7, return 7 without an explicit result-count write. Do not require the context to be unchanged.

4. **Request strictly inside the initial span.**  
   Suppose preparation succeeds, the current-position value is 1000, the end-related value is 1009, and the request is 4. The calculated span is 10. Invoke the three-argument transfer dependency with destination, source value 1000, and count 4. If that dependency leaves the current-position field unchanged, write 1004 there, write 4 to the count location, and return 0.

5. **Equality triggers the next-stage operation.**  
   With the same span of 10 and a request of 10, take the two-explicit-argument invocation path, advance the current-position field by 10, and call F-a78a7790af0e. If it succeeds, invoke the final three-argument transfer with count zero. If dependencies leave the relevant values unchanged, report 10 and return 0.

6. **Failure after initial accounting.**  
   Given span 10, request 14, and F-a78a7790af0e returning −3, report 10 and return −3. The explicit current-position advancement has already occurred.

7. **Backend partial result with failure.**  
   Given span 10, request 25, successful F-a78a7790af0e, and threshold 8, invoke F-edf0d9c5fd88 with temporary count 15. If it changes that count to 6 and returns −9, report 16 and return −9.

8. **Threshold equality uses the buffered path.**  
   If the remaining request and threshold are both 8, select the three-argument F-70a021fc7c24 path, not F-edf0d9c5fd88.

9. **Dependency mutation affects cursor advancement.**  
   On a three-argument transfer path with count 4, if the dependency changes the current-position field to 2000, the subsequent explicit update writes 2004, assuming no further intervening modification.

## Uncertainties

- **documented_only — Two-argument invocation:** The effective transfer size and calling convention of the initial-region exhaustion invocation of F-70a021fc7c24 are unresolved.
- **documented_only — Dependency semantics:** Actual transferred bytes, error conventions, ownership, retention, allocation, and global mutations are unknown.
- **documented_only — Auxiliary argument:** Its original value has no established observable use in the visible path.
- **documented_only — Arithmetic extremes:** The representation combines signed-looking field arithmetic with unsigned counts. Unsigned count addition and subtraction are 32-bit modular operations. Exact machine behavior for overflowing signed-looking address expressions is not independently verified. A portable signed-overflow contract must not be inferred.
- **documented_only — Context invariants:** No evidence establishes valid field ordering, a minimum span, or sufficient storage after the next-stage operation.
- **documented_only — Invalid accesses:** No defined recovery behavior is shown for invalid addresses or dependency failures that do not return normally.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes branch conditions, field offsets and widths, explicit argument lists, count propagation, status propagation, and the ordering of calls and writes described above. Buffered-transfer terminology and apparent cursor/end roles are inferred intent.

No dependency bodies, runtime observations, executable checks, or memory-layout documentation were supplied. No runtime tests have been performed.
