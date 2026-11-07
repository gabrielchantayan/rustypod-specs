<!-- Trusted provenance: F-5debf13772ea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5debf13772ea

## Purpose

F-5debf13772ea conditionally updates a four-byte result field reached through an object and its associated data. Two indirect predicate calls gate processing. When processing is permitted, the function requests four output words from F-a2e9ccb00bdb, selects an unsigned 32-bit quantity derived from those words, and passes that quantity to F-8c08b0c651a4.

The direct result-field effects distinguish three cases:

- A successful request followed by a conversion result other than all ones: store the conversion result.
- A successful request followed by an all-ones conversion result: store all ones, then store the bit pattern representing −2.
- A failed request: store zero for one distinguished status, or all ones for every other nonzero status.

The domain meaning of the selected quantity, the output words, and the stored result is not established. Describing them as lengths, positions, capacities, or application error codes would require evidence beyond this function.

## Interface

### Entry, return, and representation

The function accepts one object pointer and returns no value.

The supplied representation uses four-byte integers and four-byte pointer-bearing fields. All offsets below are byte offsets. These layouts describe the represented interface, not a portable layout for arbitrary pointer widths.

No explicit validation is present for the object, dispatch targets, associated data, result record, or referenced global input. Defined completion requires every access actually taken to reach valid memory and every called target to support the represented call.

### Object and related layouts

| Location | Established use |
|---|---|
| Object +0 | Pointer to a dispatch table. |
| Object +8 | Pointer to associated data; consulted at multiple stages. |
| Dispatch table +8 | First indirect gate; receives the object pointer and returns a 32-bit value tested against zero. |
| Dispatch table +404, or +0x194 | Second indirect gate; receives the object pointer and returns a 32-bit value tested against zero. |
| Associated data +16, or +0x10 | Pointer to the result-bearing record. |
| Associated data +176, or +0xB0 | Four-byte request input; both its address and its contents are supplied to F-a2e9ccb00bdb. |
| Associated data +180, or +0xB4 | Another four-byte request input. |
| Result-bearing record +36, or +0x24 | Four-byte result field written by this function. |

No nonoverlap or aliasing guarantees are established for these objects.

### Global-referenced request input

The first argument to F-a2e9ccb00bdb is a four-byte value read **through a global-designated pointer or reference**. The pointer/reference and the referenced input value are distinct: the argument is the value obtained by dereferencing that pointer/reference, not merely the pointer value and not an established direct scalar-global read.

The declaration of this global designation is unavailable. Consequently, the evidence does not establish:

- Whether the pointer/reference itself resides in mutable global storage or denotes a fixed reference.
- Its initialization mechanism or lifetime.
- The allocation, ownership, lifetime, or mutability of the referenced memory.
- Whether resolving the pointer/reference requires a separate observable storage read.

The dereference must be valid when the request argument is prepared. No local check enforces that requirement.

### Request output storage

F-a2e9ccb00bdb receives the address of automatic output storage used as four consecutive four-byte words:

| Output offset | Use after a successful request |
|---|---|
| +0 | First component of the primary sum. |
| +4 | Second component of the primary sum. |
| +8 | First component of the secondary sum, or the all-ones disabling sentinel. |
| +12 | Second component of the secondary sum, used only when the sentinel does not disable it. |

This storage is not visibly initialized before the request. On a successful return, the dependency must have supplied valid values for every word subsequently consumed. On request failure, the function does not consume these output words.

## Behavior

### Gate evaluation

The first gate is resolved through the object's dispatch table at offset +8 and called with the object pointer.

- A zero result ends processing.
- A nonzero result permits the second gate to be resolved and called.

The second gate is resolved through the object's dispatch table at offset +404.

- A nonzero result ends processing.
- A zero result permits the request.

Thus, processing requires a nonzero first result and a zero second result. These are zero/nonzero tests, not positive/negative tests.

The second dispatch lookup occurs after the first call. The function does not establish that the first callee preserves the object's dispatch-table pointer, associated data, or any other state.

When either gate rejects processing, this function makes no direct result-field write and calls neither F-a2e9ccb00bdb nor F-8c08b0c651a4. This does not imply that the gate calls themselves have no side effects.

### Request preparation and invocation

After the gates permit processing, the function establishes the address of the word at associated data +176. F-a2e9ccb00bdb receives seven arguments in this positional order:

1. The four-byte value read through the global-designated pointer/reference.
2. The established address of associated data +176.
3. The four-byte value read through that same established address.
4. A zero word.
5. The four-byte value at associated data +180, reached by a separate reference to the object's associated-data field.
6. A zero word.
7. The address of the automatic four-word output storage.

The second and third arguments therefore refer to the same established input location: one is its address and one is its contents. The fifth argument is not specified as a read relative to that captured address; it separately uses the associated-data pointer.

The source establishes argument positions and the earlier establishment of the second-argument address. It does not establish a total evaluation order among all remaining argument expressions. In particular, no reliable ordering is established among the global-referenced value read, the third-argument contents read, and the fifth-argument field read.

The returned status is interpreted as a 32-bit value:

- Zero selects successful-request processing.
- −42170, represented by `0xFFFF5B46`, selects the distinguished failure.
- Every other nonzero value selects the general failure.

### Successful request: quantity selection

The primary quantity is derived by adding output words +0 and +4.

Output word +8 is then tested against the all-ones bit pattern:

- If it is `0xFFFFFFFF`, the primary quantity is selected. Output word +12 is not consumed.
- Otherwise, output words +8 and +12 form a secondary sum. The smaller candidate is selected using unsigned 32-bit comparison.

For equal candidate values, the represented branch selects the secondary candidate, but the argument value supplied to F-8c08b0c651a4 is unchanged by that choice.

The disabling sentinel applies to word +8 itself. It does not apply to the completed secondary sum or to the primary sum. In particular, an enabled secondary sum may equal all ones without being treated as disabled.

The selected 32-bit unsigned quantity is passed as the single argument to F-8c08b0c651a4.

### Successful request: result writes

After F-8c08b0c651a4 returns, the function performs the following externally relevant sequence:

1. Resolve the current associated-data pointer from object +8.
2. Resolve its result-record pointer at +16.
3. Store the conversion's four-byte return value at result record +36.
4. Resolve the result-record pointer again through the object's associated data.
5. If the conversion return is not all ones, return without another result-field store.
6. If the conversion return is all ones, store `0xFFFFFFFE`, representing −2, at +36 of the record obtained by the second lookup.

The second destination-pointer lookup occurs even when the conversion return is not all ones. Therefore, a non-sentinel conversion result does not eliminate that access or its validity requirement.

For an all-ones conversion result, the first store is not omitted or replaced by a single −2 store. The direct store sequence is all ones followed by −2. Because the destination is resolved again, the two stores are not established to target the same record if intervening state changes or aliasing affect the pointer chain.

There is no call between these two stores, but the source supplies no synchronization or general guarantee against externally changed or aliased state.

### Distinguished request failure

A request status of −42170 causes one direct store of zero to the result field reached through the current object-associated pointer chain.

F-8c08b0c651a4 is not called. The request output words are not consumed. The function returns after the store.

### Other request failures

Any other nonzero request status causes one direct store of `0xFFFFFFFF` to the result field reached through the current object-associated pointer chain.

F-8c08b0c651a4 is not called. The request output words are not consumed.

### Arithmetic interpretation and limits

The source representation expresses the component additions using signed 32-bit operands, while the candidate selection uses unsigned 32-bit values.

When a component sum is representable as a signed 32-bit integer, its resulting bit pattern is interpreted as unsigned for selection and conversion. For example, −2 plus 5 equals 3 without signed overflow; no wrapping assumption is needed for that calculation.

For a mathematical sum outside the signed 32-bit range, the source-like representation alone does not establish portable source-language behavior. Under a 32-bit wrapping interpretation of the represented machine arithmetic, only the low 32 bits are retained before unsigned comparison.

No saturation, explicit overflow rejection, or wider intermediate arithmetic is visible. Overflow-boundary examples below are conditional on that wrapping interpretation, not assertions of portable signed-overflow behavior.

## Side effects

### Direct effects

Outside its automatic working storage, the function explicitly writes only the four-byte result field reached through the object-associated pointer chain.

The number of direct result-field stores is:

| Path | Stores |
|---|---|
| Either gate rejects | None |
| Distinguished request failure | One zero store |
| Other request failure | One all-ones store |
| Successful request, conversion not all ones | One conversion-result store |
| Successful request, conversion all ones | All-ones store followed by −2 store |

The function shows no direct allocation, deallocation, reference-count operation, global-input write, or explicit initialization of the request output storage.

### Dependency effects and ownership

The indirect gates and named dependencies may have additional effects. Their preservation of globals, dispatch information, object fields, associated memory, or resource ownership is not established.

F-a2e9ccb00bdb receives an address into associated data and an address into automatic storage. No explicit ownership transfer appears at the call site. That absence does not establish that the dependency preserves the associated data, refrains from retaining pointers, or refrains from releasing resources.

The automatic output storage has the caller's activation lifetime. Safe use after the call requires that any retained address not be used beyond that lifetime; actual retention behavior is unknown.

Result destinations are resolved after the relevant dependency calls rather than fixed before them. Consequently, dependency changes to the object-associated pointer chain can affect the eventual destination, provided subsequent accesses remain valid.

No atomicity or synchronization guarantee is visible. In particular, the two-store conversion-sentinel path is not established as one indivisible update.

## Dependencies

| Dependency | Established interaction | Unresolved contract |
|---|---|---|
| First indirect gate; opaque function ID unavailable | Resolved at dispatch-table +8; called first with the object; zero stops processing. | Identity, predicate meaning, side effects, ownership behavior, and abnormal completion. |
| Second indirect gate; opaque function ID unavailable | Resolved at dispatch-table +404 after a nonzero first result; nonzero stops processing. | Identity, predicate meaning, state preservation, and abnormal completion. |
| F-a2e9ccb00bdb | Receives the seven arguments described above; returns a 32-bit status; supplies consumed output words on success. | Input/output semantics, global effects, pointer retention, ownership, and other status meanings. |
| F-8c08b0c651a4 | Receives the selected unsigned 32-bit quantity; returns the four-byte value initially stored; all ones triggers a second store of −2. | Conversion semantics, accepted domain, side effects, ownership behavior, and meanings of other results. |

These interactions describe this caller's use of the dependencies, not their complete contracts.

## Acceptance examples

These examples are conditional consequences of static evidence, not executed tests. Unless otherwise stated, assume valid accesses, returning callees, stable pointer chains, and no relevant dependency side effects.

1. **First gate rejects.**  
   If the first gate returns zero, the second gate and both named dependencies are not called. There is no direct result-field write.

2. **Second gate rejects.**  
   If the first gate returns 7 and the second returns −1, both named dependencies are skipped. A negative nonzero second result rejects processing just as a positive nonzero result does.

3. **First request argument is a dereferenced value.**  
   If the global-designated pointer/reference identifies valid memory containing `0x12345678`, the first request argument is `0x12345678`, not the pointer/reference's address or pointer bit pattern. This example does not prescribe how that pointer/reference is stored or initialized.

4. **Secondary candidate disabled.**  
   Successful output words of 100, 20, all ones, and an unspecified fourth word cause F-8c08b0c651a4 to receive 120. The fourth word is not consumed. If conversion returns 9, one result-field store writes 9, followed by the required destination-pointer lookup.

5. **Secondary candidate smaller.**  
   Successful output words of 100, 20, 30, and 40 yield candidates 120 and 70. Conversion receives 70.

6. **Unsigned comparison.**  
   Successful output words of 2147483647, 0, −2, and 0 yield candidate bit patterns `0x7FFFFFFF` and `0xFFFFFFFE`. Neither addition overflows signed 32-bit arithmetic. Conversion receives `0x7FFFFFFF`, because comparison is unsigned.

7. **Completed sum is not the disabling sentinel.**  
   Successful output words of −2, 0, −2, and 1 yield unsigned candidates `0xFFFFFFFE` and `0xFFFFFFFF`. The secondary candidate is enabled because its first component is −2, not −1. Conversion receives `0xFFFFFFFE`.

8. **Conversion sentinel causes two stores.**  
   If conversion returns all ones, the direct store sequence is `0xFFFFFFFF` followed by `0xFFFFFFFE`. With a stable pointer chain, both affect the same field. The destination lookup between them must not be omitted.

9. **Distinguished request failure.**  
   A request return of −42170 causes one zero store, no conversion call, and no consumption of request output words.

10. **Other request failure.**  
    A request return of 1 or −1 causes one all-ones store, no conversion call, and no consumption of request output words.

11. **Ordinary negative-component addition.**  
    With successful primary components −2 and 5 and the secondary candidate disabled, conversion receives 3. This does not depend on wrapping arithmetic.

12. **Actual overflow boundary, conditionally interpreted.**  
    With successful primary components 2147483647 and 1 and the secondary candidate disabled, a 32-bit wrapping interpretation produces `0x80000000` for conversion. The mathematical sum exceeds the signed 32-bit range, so this result is conditional on the arithmetic interpretation.

13. **Dependency redirects the destination.**  
    If conversion changes object +8 to identify different valid associated data, the subsequent result write follows that current pointer chain rather than a destination captured before conversion.

## Uncertainties

The following remain **documented_only**:

- Domain meanings and lifetimes of the object, associated data, result record, selected quantity, and result values.
- Declaration, storage, initialization, and ownership of the global-designated pointer/reference and its referent.
- Identities and full contracts of the indirect gates.
- Dependency side effects, global preservation, pointer retention, and resource ownership or release.
- Ordering among request argument-expression reads where the representation does not establish one.
- Behavior after invalid accesses, invalid dispatch targets, incomplete successful output, nonreturning callees, or dependency failures outside the represented status protocol.
- Memory properties such as volatility, device backing, sharing, synchronization, and concurrent mutation.
- Portable behavior of additions that exceed the signed 32-bit range.

Selecting the smaller of two quantities while permitting one to be disabled suggests a bounded-quantity calculation. That is inferred intent, not an established application-level purpose.

## Evidence

The supplied static representation establishes the dispatch and field offsets, gate order, seven request argument positions, dereference of the global-designated pointer/reference, status branches, output-word usage, unsigned comparison, sentinel checks, and ordered result stores.

It also establishes the repeated destination lookup after the first successful-request store, including when no second store follows.

The declaration of the global designation and the implementations of all callees are unavailable. No external interface specification, execution trace, or runtime observation was supplied. No runtime tests have been performed. All acceptance examples are conditional static deductions rather than observed outcomes.
