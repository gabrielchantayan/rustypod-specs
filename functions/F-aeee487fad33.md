<!-- Trusted provenance: F-aeee487fad33; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aeee487fad33

## Purpose

F-aeee487fad33 processes an ordered sequence of length-and-offset descriptors. It combines contiguous descriptors into pending runs, invokes an indirect positioning operation for each flushed run, and invokes an indirect transfer operation with successive addresses derived from the caller-supplied buffer base.

The function returns a 32-bit status distinguishing normal completion, a dependency-result failure, and observed cancellation.

**Established static behavior:** the length pattern `0xFFFFFFFF` identifies a marker rather than an ordinary descriptor. Every other 32-bit length pattern can participate in a run, including zero and patterns whose signed interpretation is negative.

**Inferred intent:** combining contiguous descriptors appears intended to reduce dependency calls while processing selected ranges. The evidence does not establish the underlying resource, the units of the offsets and lengths, or the direction of transfer. In particular, passing a buffer address does not establish whether the dependency reads from it, writes to it, or uses it in another way.

## Interface

### Inputs

The interface has six arguments:

| Argument | Width and interpretation |
|---|---|
| Context address | 32-bit address of an object containing an indirect-object pointer at byte offset `8`. |
| Buffer base | 32-bit address used to derive the address supplied to the transfer dependency. |
| Descriptor count | Signed 32-bit iteration bound. A nonpositive value processes no descriptors. |
| Length-table base | 32-bit address of a table of 4-byte length entries. |
| Offset-table base | 32-bit address of a table of 4-byte offset entries. |
| Cancellation address | Address of a byte compared with exactly `0x01` before each descriptor is processed. |

For zero-based descriptor index \(j\), the corresponding table entry is addressed at the table base plus \(4j\), using 32-bit address arithmetic.

Length and offset entries are read as 32-bit patterns. Their interpretations differ:

- A length equal to `0xFFFFFFFF` on the initial length read selects marker handling.
- Other lengths contribute their raw patterns to the accumulated transfer count.
- Lengths contribute their signed 32-bit values to endpoint arithmetic.
- A run-start offset is zero-extended to form a two-word position.
- The accumulated count is interpreted as signed only for marker and final-flush eligibility.

### Indirect-object layout

The visible access chain is:

| Location | Contents |
|---|---|
| Context plus byte offset `8` | 32-bit indirect-object pointer |
| Indirect object plus byte offset `0` | 32-bit dispatch-table pointer |
| Dispatch table plus byte offset `0x14` | Positioning-function pointer |
| Dispatch table plus byte offset `0x10` | Transfer-function pointer |

These are byte offsets, not table indices. The concrete dependency targets are not supplied.

### Return value

| 32-bit value | Meaning |
|---|---|
| `0` | Processing completed under the function’s flush rules. |
| `1` | Positioning returned nonzero, or transfer returned a 32-bit value unequal to the requested count. |
| `2` | A cancellation check observed exactly `0x01`. |

Normal completion does not imply that every non-marker descriptor resulted in a transfer. A zero or signed-negative pending count is not flushed at the end.

### Preconditions and ownership

The caller must provide readable storage for every access actually reached:

- The cancellation byte at the start of each processed iteration.
- Each processed length entry, including repeated reads where applicable.
- Each offset entry that the selected path reads.
- The context, indirect object, dispatch table, and function-pointer entries whenever a dependency call is reached.

The buffer address and requested count must satisfy the selected transfer dependency’s requirements. Those requirements, including capacity, access direction, and alignment, are not established.

The function contains no explicit validation of null pointers, table sizes, buffer capacity, address wrapping, or arithmetic overflow. Invalid accesses are not explicitly translated into one of its three return statuses.

No allocation, deallocation, ownership transfer, or pointer retention is directly established for this function. Dependency ownership and retention behavior remain unknown.

## Behavior

### Initial state and empty input

The initial processing state contains:

- An accumulated transfer count of zero.
- A buffer displacement of zero.
- An inactive pending position whose low and high words are both `0xFFFFFFFF`.
- No pending run.

If the descriptor count is zero or negative, no iteration occurs. The final-flush condition rejects the zero accumulated count, and the function returns `0`.

This path reads neither the cancellation byte nor either descriptor table, and it does not traverse the indirect-object chain.

### Cancellation

At the beginning of each iteration, before reading that descriptor’s length, the function reads the cancellation byte.

If its value is exactly `0x01`, the function immediately returns `2`:

- The current descriptor is not read.
- Any pending run is not flushed.
- Earlier dependency effects are not undone.

Other byte values do not trigger cancellation.

There is no cancellation check between positioning and transfer, and no separate check before the final flush. Cancellation changed during a dependency call can therefore remain unobserved if no later iteration begins.

These are the checks visible in the supplied representation. Atomicity, volatile access, cross-thread visibility, and synchronization are not established.

### Marker handling

The first length read in an iteration determines whether that descriptor is a marker. If this read yields `0xFFFFFFFF`, its offset entry is not read.

A marker flushes the pending run only when the accumulated count, interpreted as signed 32-bit, is strictly positive.

For an eligible marker flush:

1. Invoke the positioning dependency with the pending position.
2. If positioning returns nonzero, return `1` without invoking transfer.
3. Otherwise invoke the transfer dependency with the current derived buffer address and pending count.
4. If its 32-bit result differs from the requested count, return `1`.
5. On exact success, increase the buffer displacement by the pending count using 32-bit arithmetic.
6. Set the pending count to zero and restore the inactive position sentinel.

If the accumulated count is zero or signed-negative, the marker does nothing. It does not clear an active run.

A marker is therefore not an unconditional boundary for all possible input patterns.

### Starting a run

For a non-marker descriptor, an inactive pending position causes a new run to be initialized:

1. Read the descriptor’s offset as the pending position’s low word.
2. Set the pending position’s high word to zero.
3. Read the descriptor’s length again as the pending transfer count.
4. Compute the endpoint from that offset and length.

Only the position with both words equal to `0xFFFFFFFF` is inactive. An offset of `0xFFFFFFFF` produces an active position because its high word is zero.

The initial marker-classification read and the later count read are separate. The function does not snapshot the length table. If the contents change between those reads, the later value determines the count and endpoint; the marker classification is not repeated.

### Endpoint and count arithmetic

The endpoint is represented by two 32-bit words.

At run start, it is the zero-extended offset plus the sign-extended 32-bit length. For each contiguous extension, the new length is added in the same signed manner:

- The endpoint low word receives the low-word sum modulo \(2^{32}\).
- The endpoint high word receives the previous high word, the length’s sign contribution, and the carry from low-word addition.
- The high word also retains only 32 bits.

The length’s sign contribution is zero when its most significant bit is clear and `0xFFFFFFFF` otherwise.

Examples:

- Offset `0xFFFFFFFE` and length `2` produce endpoint high word `1`, low word `0`.
- Offset `0` and length `0xFFFFFFFE`, interpreted as −2, produce endpoint high word `0xFFFFFFFF`, low word `0xFFFFFFFE`.

The accumulated transfer count is separate from the endpoint. It is the sum of the raw length words modulo \(2^{32}\), without overflow rejection.

### Contiguous extension

When a run is active, the current non-marker descriptor extends it only if both conditions hold:

- The endpoint high word is zero.
- The current offset equals the endpoint low word.

If the endpoint high word is nonzero, the offset comparison is skipped.

On a match, the function reads the current length again, adds it to the accumulated count modulo \(2^{32}\), and advances the endpoint using signed-length arithmetic. The original run-start position remains unchanged. No dependency is called for that iteration.

A zero-length descriptor can extend a run without changing its count or endpoint.

### Noncontiguous transition

If a run is active and the current non-marker descriptor does not satisfy the contiguous conditions, the existing run is flushed before the new run is initialized.

This flush has no signed-positive-count guard. An active run with count zero or with its most significant bit set still causes positioning and, after successful positioning, transfer.

Positioning and transfer have the same result checks as a marker flush. Failure returns `1` immediately.

After exact transfer success:

- The buffer displacement increases by the old pending count modulo \(2^{32}\).
- The current descriptor’s offset is read to initialize a new position.
- Its length is read to initialize the new count and endpoint.

These initialization reads occur after the dependency calls. They can observe dependency-induced changes to the tables. If the offset was already read for a failed contiguity comparison, it is read again rather than reused.

### Final flush

After the loop, the accumulated count is interpreted as signed 32-bit.

If it is less than one, the function returns `0` without a final dependency call. This includes both zero and all patterns with the most significant bit set.

Otherwise, the function positions to the pending start and requests transfer of the pending count:

- Nonzero positioning result produces `1` and suppresses transfer.
- A transfer result unequal to the requested 32-bit count produces `1`.
- Zero positioning status followed by an exact transfer result produces `0`.

There is no subsequent transfer, so the final successful call does not advance the buffer displacement.

### Failure and partial progress

Every reached transfer must return exactly the requested 32-bit count. Smaller values, larger values, and error-like patterns receive no distinct treatment: inequality produces status `1`.

The function performs no retry, partial-count adjustment, remaining-count request, or additional error query.

Earlier calls are not rolled back. Effects performed by a failing dependency are likewise outside this function’s control.

## Side effects

No direct writes through the supplied arguments are visible in the function body. The body reads caller-accessible memory and invokes dependencies that may have additional effects.

For each flushed run, the established call order is:

1. Positioning.
2. Transfer only if positioning returned zero.
3. Subsequent descriptor processing, if any.

The transfer address is the buffer base plus the accumulated displacement, calculated using 32-bit arithmetic. A successful non-final flush increases that displacement by its requested count modulo \(2^{32}\). Neither this progression nor the name “transfer” establishes memory-access direction.

The context’s indirect-object pointer and the dispatch information are accessed again for later calls. The function does not establish a stable object or dispatch-table snapshot across a positioning/transfer pair. A positioning dependency that changes dispatch state can affect the subsequent transfer target or object argument.

The supplied representation does not establish every constituent memory-read order within an individual indirect-call expression. It does establish the sequencing between dependency calls and subsequent descriptor initialization.

Wrapping addresses, overlapping argument storage, and dependency-induced changes to tables or context are not rejected. The resulting observations depend on the reads actually reached.

No assumption is made that either dependency preserves globals, leaves inputs unchanged, retains no pointers, or refrains from allocating or releasing memory.

## Dependencies

The following opaque IDs identify call roles, not proven concrete implementations.

### F-opaque-001 — positioning role

Resolved through dispatch-table byte offset `0x14`.

The observed argument order is:

1. Indirect-object pointer.
2. Resolved positioning-function pointer itself.
3. Pending position low word.
4. Pending position high word.
5. Zero.

The result is tested against zero. Any nonzero result causes F-aeee487fad33 to return `1`.

The function-pointer argument is part of the observed interface. Its purpose is **documented_only**; it must not be omitted or assigned an unsupported meaning.

### F-opaque-002 — transfer role

Resolved through dispatch-table byte offset `0x10`.

The observed argument order is:

1. Indirect-object pointer.
2. Buffer base plus cumulative displacement.
3. Pending 32-bit count.
4. The value `2`.

The result is compared with the requested count as a 32-bit value. Exact equality is the sole success criterion visible here.

The meaning of `2`, transfer direction, units, error representation, and memory ownership behavior are **documented_only**.

## Acceptance examples

These are conditional consequences of static inspection, not executed tests. Unless stated otherwise, storage is valid and stable, cancellation is not `0x01`, positioning returns zero, and transfer returns exactly the requested count.

1. **Nonpositive descriptor count.** A count of `0` or `−1` returns `0` without reading cancellation, either table, or the indirect-object chain.

2. **Immediate cancellation.** With a positive descriptor count and cancellation byte `0x01`, the result is `2`. Neither descriptor table is read.

3. **Contiguous descriptors.** Offsets `100` and `103`, with lengths `3` and `2`, produce one final positioning call for low word `100`, high word zero, followed by one transfer request of `5` using the original buffer base.

4. **Separated descriptors.** Offsets `100` and `200`, each with length `3`, produce two positioning/transfer pairs. The transfer-address arguments are the buffer base and base plus `3`. This does not establish that either address is written.

5. **Positive run followed by marker.** A length-`4` descriptor followed by a marker flushes the four-unit run at the marker. The marker’s offset is not read. With no further descriptors, no final transfer occurs.

6. **Cancellation with pending work.** If the first descriptor starts a run and cancellation is observed before the second descriptor, the function returns `2` without flushing that run.

7. **Positioning failure.** Any nonzero positioning result produces status `1` and prevents the corresponding transfer.

8. **Transfer mismatch.** A transfer request for `5` returning `4` produces status `1`. No request for the remaining unit follows.

9. **Zero-count noncontiguous transition.** Offset `10`, length `0`, followed by offset `20`, length `1`, causes a zero-count positioning/transfer pair before the second run starts. That zero-count transfer must return zero to permit continuation. A lone zero-length descriptor instead returns `0` without dependency calls.

10. **Endpoint crossing the low-word boundary.** Offset `0xFFFFFFFE`, length `2`, followed by offset `0`, length `1`, does not merge. The first endpoint has high word `1`, so matching its low word is insufficient.

11. **Signed-negative final count.** A lone length of `0x80000000` returns `0` without a final dependency call.

12. **Marker retaining a zero-count run.** Offset `10`, length `0`, then a marker, then offset `20`, length `1`, retains the first run across the marker. The last descriptor triggers a zero-count flush.

13. **Maximum offset remains active.** Offset `0xFFFFFFFF`, length `0`, followed by offset `0`, length `1`, triggers a zero-count flush. The first run’s high position word is zero, so its low word alone does not make it inactive.

14. **Changes during a flush.** If a dependency changes the current noncontiguous descriptor before new-run initialization, the subsequent offset and length reads use the changed contents. This example is conditional on those changes and continued valid storage.

## Uncertainties

- **Documented_only:** concrete identities and external contracts of both indirect dependencies.
- **Documented_only:** transfer direction, buffer capacity requirements, alignment, count units, and the meaning of the constant `2`.
- **Documented_only:** dependency allocation, release, pointer retention, global-state changes, input mutation, and dispatch mutation.
- **Documented_only:** concurrency guarantees and visibility of cancellation or table changes.
- **Documented_only:** exact constituent read order within individual indirect-call expressions.
- **Documented_only:** behavior following invalid memory access or an invalid indirect target; no explicit recovery path is supplied.
- **Inferred intent:** ordinary positive lengths and nonwrapping address calculations likely describe a useful operating domain, but the function does not enforce it.
- **Established arithmetic, unknown dependency meaning:** zero, signed-negative, and wrapped counts have defined caller-side control-flow consequences here. Whether a dependency accepts such requests is not established.

## Evidence

This contract is based solely on static inspection of the supplied source representation and dependency map. The prior draft and reviewer feedback are not independent behavioral evidence.

The supplied body establishes the signed loop bound, cancellation comparison, marker classification, repeated table reads, active-position sentinel, endpoint arithmetic, count accumulation, flush guards, dispatch offsets, dependency argument shapes, call ordering, and return decisions.

The body establishes a derived buffer-address argument, not a transfer direction. No dependency bodies or runtime observations were available. No runtime tests have been performed.
