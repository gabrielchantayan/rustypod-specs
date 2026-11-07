<!-- Trusted provenance: F-7ddd3f3b343e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7ddd3f3b343e

## Purpose

F-7ddd3f3b343e requests a position on an external object, reads a four-byte count into caller-provided output storage, obtains storage for a bounded working number of 12-byte records, reads the payload, and transforms the first two four-byte fields of each admitted record.

Its explicit return values are 32-bit statuses:

- **0:** The payload-read comparison succeeded, all admitted record transformations completed, and the final output-state write occurred.
- **1:** An entry condition or dependency-result check failed.

Failure is not transactional. Position changes, destination writes, published storage, and dependency side effects are not rolled back.

The likely purpose is loading and normalizing a record table. The external-object protocol, record semantics, storage ownership, and field transformation remain unestablished.

## Interface

### Arguments

The supplied representation exposes eight positional arguments in a 32-bit environment.

| Position | Width and interpretation | Established use |
|---|---|---|
| 1 | Opaque 32-bit value | Not directly used. |
| 2 | 32-bit pointer to an external object | Supplies dispatch information for positioning and reading. |
| 3 | Unsigned 32-bit integer | Low word of the requested position. |
| 4 | Signed 32-bit integer | High position word; a negative value rejects the operation. |
| 5 | Opaque 32-bit value | Not directly used. |
| 6 | Opaque 32-bit value | Not directly used. |
| 7 | 32-bit integer selector | Zero selects a 12-byte adjustment; any nonzero value selects 20 bytes. |
| 8 | 32-bit pointer to writable output storage | Receives the count, storage address, and final state word. Null rejects the operation. |

The unused arguments have no direct effect on this function’s visible control flow. Their wider calling-contract significance is unknown.

### Output layout

| Byte offset | Width | Established access |
|---|---:|---|
| 0 | 4 bytes | Destination of the count read; subsequently loaded once as a signed 32-bit count. |
| 4 | 4 bytes | Explicitly written with zero after successful payload processing. |
| 8 | 4 bytes | Explicitly receives the storage dependency’s result, including a null result. |

The function does not initialize the entire output object.

After loading the count, it does not explicitly rewrite the count word or replace it with the bounded working count. **This does not guarantee that the final count word equals the value originally read.** Dependencies may modify it through retained pointers, aliases, or shared state. Such modifications do not change the already-established working count.

The word at offset 4 receives no explicit write on a failure path. The word at offset 8 receives no explicit write until the storage dependency returns. These are restrictions on this function’s own writes, not guarantees about dependency effects.

### Record layout

Each admitted record spans 12 bytes.

| Record-relative offset | Width | Direct processing after the payload read |
|---|---:|---|
| 0 | 4 bytes | Address supplied to F-76bf07f4ac67; its result is written to the corresponding field destination. |
| 4 | 4 bytes | Address supplied to F-76bf07f4ac67; its result is written to the corresponding field destination. |
| 8 | 4 bytes | No explicit transformation or write by this function. |

The final field is not guaranteed unchanged: a dependency may modify it. Field meanings, byte order, and signedness are not established.

### Preconditions

For ordinary operation with distinct storage regions:

- Argument 2 must identify an accessible external object whose first four-byte word identifies usable dispatch data.
- The dispatch entries at byte offsets 16 and 20 must support the observed calls.
- Argument 8 must identify accessible storage covering the output layout.
- A nonnull storage result must support the payload operation and all admitted record accesses.
- Invoked dependencies must return normally for an explicit status return to occur.

Only the high-word sign and output-pointer nullness are checked locally. Invalid objects, invalid nonnull output pointers, insufficient storage, and unusable dispatch entries have no established recovery path.

## Behavior

### Entry rejection

A negative high position word or null output pointer produces status 1.

On this path, the function does not access the external object, invoke a dependency, read a count, request storage, or explicitly write output storage.

A nonnegative high word and nonnull output pointer permit continuation without validating the external object.

### Position adjustment and request

The function selects an adjustment of 12 bytes for a zero selector and 20 bytes otherwise.

The positioning arguments use two-word arithmetic:

- The low word is the low 32 bits of the unsigned low-word addition.
- The high word is increased by the unsigned carry from that addition.

The carry is an unsigned 32-bit addition carry, not a signed-overflow indication. Because the high-word addition involves an unsigned carry value, an initial high word of 0x7fffffff can become the bit pattern 0x80000000. There is no second sign check.

The positioning dependency is selected through dispatch byte offset 20. Its visible argument sequence is:

1. The external-object pointer.
2. The positioning callback address itself.
3. The adjusted low word.
4. The high word plus carry.
5. Zero.

The explicit callback-address argument is part of the supplied representation. Its ABI meaning is unresolved; it must not be silently replaced by a conventional positioning signature.

Only a return of exactly zero permits continuation. Any other return produces status 1 without a count read or storage request by this function.

### Count read and working-count selection

After successful positioning, the function selects the read dependency through dispatch byte offset 16 and supplies:

1. The external-object pointer.
2. The output pointer, targeting offset 0.
3. A requested size of four bytes.
4. Selector value 1.

Only a return of exactly 4 permits continuation. Any other return produces status 1. Destination or external-object changes already made by that callback are not restored.

After an exact return, the function loads the output’s first four-byte word and interprets it as a signed 32-bit integer. It does not invoke F-76bf07f4ac67 on that word.

The working count is selected as follows:

- A signed count greater than 32,767 becomes 32,768.
- Every other signed count is used unchanged.

Thus positive counts are capped, but zero and negative counts are not rejected.

The working count is retained independently of the output count word. Later dependency changes to that word do not alter the requested payload size or loop bound. Conversely, the function’s lack of a later count write does not establish preservation of the output word.

### Storage request and publication

The storage-size expression is the working count multiplied by 12.

For working counts from zero through 32,768, the expression ranges from zero through 393,216 and fits in signed 32-bit arithmetic.

Negative counts also reach this expression. Some negative products are representable, including −12 for a count of −1. More negative values can exceed the signed 32-bit range. The supplied representation does not establish a portable result for overflowing multiplication or the dependency’s interpretation of a negative size. No general wraparound contract is inferred.

F-fab3799b3190 receives the size expression. Its returned 32-bit pointer-like value is immediately written at output offset 8, before the null check.

- A null result overwrites that output word with zero and produces status 1.
- A nonnull result permits the payload read.

There is no explicit release of any previous value overwritten at offset 8.

### Payload read

The function selects the read dependency again through dispatch byte offset 16. The earlier callback address is not established as permanently cached.

The payload call receives:

1. The external-object pointer.
2. The nonnull storage result.
3. The working count multiplied by 12.
4. Selector value 2.

Continuation requires the callback result to compare equal to the requested size expression. Otherwise, the function returns 1.

For a zero working count, both the storage request and payload call still occur. Success therefore requires a nonnull zero-size storage result and a payload return equal to zero.

A failed payload comparison does not trigger deallocation, output restoration, or record transformations. The storage result has already been published, although dependencies may subsequently change that output word. Partial payload writes and external-object changes are not undone.

### Record transformation and completion

After a successful payload comparison, records are processed in ascending index order, starting at zero and continuing while the index is less than the signed working count.

For each admitted record:

1. The first field’s address is passed to F-76bf07f4ac67.
2. Its returned four-byte value is written to that field’s destination.
3. The second field’s address is passed to F-76bf07f4ac67.
4. Its returned four-byte value is written to that field’s destination.

The first field’s write precedes the second field’s dependency call.

The payload base is read through output offset 8 for field-address calculations. It is read again to calculate the write destination after each transformation call. Consequently, the destination need not equal the address originally passed to the dependency if that dependency changes the published pointer.

The same caution applies across fields and records: later accesses can use a changed published base. No dependency purity, global-state preservation, or alias preservation is assumed.

A zero or negative working count admits no record iterations. A negative count can reach this point only if the storage and payload dependencies satisfy the preceding checks.

After the loop, the function explicitly writes zero at output offset 4 and returns 0. It does not perform a final validation of the count word or published pointer.

## Side effects

The observable sequence on a complete ordinary path is:

1. Positioning request.
2. Four-byte count-read request.
3. Load and selection of the working count.
4. Storage request.
5. Publication of the storage result.
6. Payload-read request.
7. Ordered transformation calls and field writes.
8. Final zero write at output offset 4.

Each failed check prevents later steps, but does not reverse earlier effects.

There is no explicit deallocation, rollback, position restoration, error-detail write, or whole-output initialization.

**Ownership is documented_only.** Publication of a storage address does not establish transfer of ownership, a required release operation, or even that F-fab3799b3190 is a conventional allocator. Dependencies may perform undisclosed ownership operations; none are assumed.

**Aliasing behavior is documented_only.** Output storage, payload storage, external-object storage, and dispatch data are not checked for overlap. Direct writes or dependency effects can therefore alter later reads. Non-overlapping output and payload storage alone do not prevent dependencies from retaining the output address and modifying it later.

In particular, a callback can modify the output count after the function has captured its working count without changing the published pointer. The original loop bound remains in effect, while the final count word may differ.

## Dependencies

| Identity | Established interaction | Unresolved contract |
|---|---|---|
| F-fab3799b3190 | Receives the count-times-12 expression; returns a 32-bit pointer-like value that is published and null-checked. | Size interpretation, storage mechanism, zero-size behavior, ownership, release protocol, and other side effects. |
| F-76bf07f4ac67 | Receives a field address and returns the four-byte value subsequently written to a freshly calculated destination. Called twice per admitted record. | Transformation meaning, byte order, memory writes, global-state effects, ownership effects, and preservation of output fields. |
| Indirect dependency at dispatch offset 20 | Receives the positioning arguments above; zero permits continuation. | ABI details, position interpretation, trailing-zero meaning, and external-state effects. |
| Indirect dependency at dispatch offset 16 | Receives a destination, size, and selector 1 or 2; its return is compared with the requested size. | Selector semantics, partial-read behavior, cursor effects, retained pointers, and other side effects. |

No opaque function IDs were supplied for the indirect dependencies. Their dispatch locations distinguish them without inventing identities.

## Acceptance examples

These are conditional static criteria, not executed tests. Unless stated otherwise, assume accessible, non-overlapping storage, normally returning dependencies, representable arithmetic, and no dependency-induced change to the published payload pointer. **These assumptions do not imply preservation of the output count word.**

1. **Rejected high word.** A negative argument 4 produces status 1 without a positioning or read call.

2. **Null output.** A null argument 8 produces status 1 without accessing output storage or the external object.

3. **Adjustment with carry.** For low word 0xfffffff8 and high word 2, a zero selector supplies adjusted words 4 and 3. A nonzero selector supplies words 12 and 3.

4. **High-word boundary.** For initial high word 0x7fffffff and a low-word addition that carries, the supplied high-word bit pattern is 0x80000000. This function does not reject that adjusted value before calling the positioning dependency.

5. **Positioning failure.** A nonzero positioning return produces status 1 without a count-read request. Any positioning effect is not rolled back.

6. **Incomplete count read.** A count-read return of 3 produces status 1 without calling F-fab3799b3190. Any bytes already written into the output remain unrestored.

7. **One record.** If the captured count is 1, the storage request is 12 bytes. A nonnull result and payload return of 12 admit two transformation calls, first-field write before second-field processing, followed by the final zero write and status 0. There is no explicit write to the record’s third field.

8. **Upper bound without a count-preservation promise.** If the captured count is 32,769, the storage and payload requests use 393,216 bytes and successful processing admits 32,768 records. This function does not explicitly rewrite the count to 32,768. A final count of 32,769 is guaranteed only if dependencies do not modify that word after it is captured.

9. **Later count mutation.** Suppose the captured count is 32,769, but a later dependency changes output offset 0 to 7 while leaving the published pointer unchanged. If all required returns and accesses succeed, the working count still admits 32,768 records. The function does not restore the output count to either 32,769 or 32,768.

10. **Zero count.** A null zero-size storage result produces status 1. A nonnull result followed by a payload return of zero permits status 0 without transformation calls.

11. **Payload failure after publication.** With captured count 2, a payload return of 12 rather than 24 produces status 1 without transformation calls or explicit deallocation. Publication has already occurred; its subsequent preservation depends on dependency behavior.

12. **Negative count.** A captured count of −1 is not rejected or capped. Its size expression is −12. If the storage result is nonnull and the payload return compares equal to −12, no records are transformed, output offset 4 is cleared, and status 0 follows. Whether the dependencies support these arguments is documented_only.

## Uncertainties

- **Inferred intent:** The external object supports positioning and reading; the payload represents records; the field dependency performs normalization. These meanings are not established contracts.
- **documented_only:** Exact calling convention and significance of the explicit callback-address argument.
- **documented_only:** Count encoding and byte order. The count is directly interpreted as a signed word after the read.
- **documented_only:** Negative-size interpretation and overflowing signed multiplication. No safe general negative-count protocol is established.
- **documented_only:** Ownership, release operations, retained pointers, dependency mutation of output fields or global state, aliasing, concurrency, and reentrancy.
- **documented_only:** Faults, exceptions, nonlocal exits, or nonreturning dependencies. No visible mechanism converts them to status 1.

## Evidence

The established statements derive from the supplied static representation: argument widths, entry conditions, dispatch offsets, call arguments, return comparisons, count selection, publication order, field-address recalculation, signed loop condition, and final output write.

The output count is loaded before the storage and payload dependencies and is not explicitly rewritten afterward. That establishes an independent working count—not preservation of the memory word across dependency calls.

No dependency implementations, caller contracts, or runtime observations were supplied. **No runtime tests have been performed.** The acceptance examples are conditional consequences of the visible behavior, not evidence that unknown dependencies support every described input.
