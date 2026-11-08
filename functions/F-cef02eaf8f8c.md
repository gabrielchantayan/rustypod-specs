<!-- Trusted provenance: F-cef02eaf8f8c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cef02eaf8f8c

## Purpose

F-cef02eaf8f8c attempts to load a bounded window of fixed-size records into an existing buffer, transform two fields of each loaded record, and publish the requested starting index in associated metadata.

The established record size is 12 bytes. On the ordinary, non-overflowing path, a request selects the remaining records beginning at the supplied index, capped at 32,768 records. This interpretation requires a nonnegative index no greater than the stored record count; the function does not enforce that condition.

The function returns the 32-bit value 0 only after both I/O callbacks report the expected results and all selected field transformations finish. It returns the 32-bit value 3 when the initial state pointer is null, the positioning callback reports a nonzero result, or the transfer callback reports a result different from the requested byte count.

“Positioning,” “transfer,” and “record count” describe inferred roles. Their exact external meanings are not established by the supplied body.

## Interface

### Inputs

The function accepts two 32-bit inputs:

- **Context address:** the address of an object containing state and callback-object pointers.
- **Requested starting index:** a value used in signed comparisons and in 32-bit address and byte-count arithmetic.

The context address is dereferenced without a null check. A valid call therefore requires readable context storage and valid reachable objects for every path actually taken.

### Relevant layouts

All offsets below are byte offsets. The supplied representation uses 32-bit pointers, integers, and transferred words.

| Object | Offset | Established use |
|---|---:|---|
| Context | 8 | Pointer to the callback object |
| Context | 440 | Pointer to the loader state; zero selects the immediate failure path |
| Loader state | 20 | Pointer to record metadata |
| Loader state | 96 | Low 32-bit portion of the base position |
| Loader state | 100 | High 32-bit portion of the base position |
| Record metadata | 0 | Signed value used to determine the selected record count |
| Record metadata | 4 | Destination for the requested starting index on success |
| Record metadata | 8 | Address of the transfer and transformation buffer |
| Callback object | 0 | Pointer to a callback table |
| Callback table | 16 | Transfer callback |
| Callback table | 20 | Positioning callback |

A record occupies 12 bytes:

| Record-relative offset | Size | Treatment by this function |
|---|---:|---|
| 0 | 4 bytes | Replaced with the result of F-76bf07f4ac67 called with this field’s address |
| 4 | 4 bytes | Replaced with the result of F-76bf07f4ac67 called with this field’s address |
| 8 | 4 bytes | No direct transformation or write by this function |

The final four bytes may still be changed by the transfer callback or by a dependency with broader side effects.

### Output

The direct result is a 32-bit status:

- **0:** the callback success conditions were met, the selected transformation loop completed, and the metadata index store was performed.
- **3:** one of the explicit failure conditions occurred.

No separate count, byte count, or underlying callback error is returned.

There is no explicit validation-error result for an invalid index, invalid pointer, insufficient buffer, arithmetic overflow, or malformed callback table. Such conditions are outside the demonstrated safe operating domain.

## Behavior

### 1. Resolve initial state

The function first reads the loader-state pointer at context offset 440.

If that pointer is zero, it returns 3. On this branch, it does not explicitly access the callback object, record metadata, or transfer buffer.

Otherwise, it follows the pointer at loader-state offset 20 and reads the signed 32-bit value at metadata offset 0.

### 2. Select the record count

The supplied body selects the number of records using a signed comparison against the requested index plus 32,767:

- If the stored count is no greater than that threshold, the selected count is the stored count minus the requested index.
- If the stored count is greater than that threshold, the selected count is 32,768.

For ordinary signed arithmetic without overflow, and with the requested index between zero and the stored count inclusive, this selects the lesser of the remaining count and 32,768.

Important boundaries:

- Exactly 32,767 remaining records selects 32,767.
- Exactly 32,768 remaining records selects 32,768.
- More than 32,768 remaining records also selects 32,768.
- An index equal to the stored count selects zero.
- An index greater than the stored count can select a negative count; there is no lower clamp.
- A negative index is not rejected.

The signed threshold addition and subtraction require special care near the limits of a signed 32-bit integer. The supplied body does not establish a portable overflow policy for those expressions. The ordinary count interpretation must not be extended across those boundaries without additional evidence.

### 3. Construct the requested position

The function constructs a two-word position from the initial loader state and the requested index.

The established arithmetic is:

- Form the 32-bit product of the requested index and 12.
- Add that 32-bit product to the low base-position word, retaining the low 32 bits.
- Add 16 to that intermediate low word, again retaining the low 32 bits.
- Form the high result from the stored high word plus:
  - one carry if adding the product to the low base word overflows 32 bits;
  - one carry if adding 16 to the intermediate low word overflows 32 bits.

The second carry occurs when the intermediate low word is at least `0xfffffff0`.

This is not established as a full-width multiplication of the index by 12 followed by a full-width addition. In particular, the visible high-word expression does not incorporate a high word from the multiplication itself. A consumer must not assume that positions for very large or negative indices follow unrestricted 64-bit mathematical arithmetic.

For nonnegative indices whose multiplication by 12 fits in 32 bits, the calculation has the expected interpretation of the base position plus 12 bytes per preceding record plus a 16-byte prefix, with carries propagated between the two position words.

### 4. Invoke the positioning callback

The function resolves the callback-table entry at offset 20 through the context’s callback object.

It invokes that callback with five arguments, in this logical order:

1. The callback-object pointer.
2. The resolved positioning-callback address itself.
3. The computed low position word.
4. The computed high position word.
5. Zero.

The second argument is visibly the callback address, not an independently established position or mode argument. Its intended ABI meaning is unknown.

If the callback returns a nonzero result, the function returns 3. It does not invoke the transfer callback or perform its own record transformations or final metadata-index store on that path.

The ordering of individual memory reads used to assemble arguments is not fully established by the supplied expression representation. The callback occurs after the needed argument values have been obtained.

### 5. Invoke the transfer callback

After a zero positioning result, the function resolves the transfer callback at callback-table offset 16.

This operation uses fresh pointer reads through the context. The buffer address is also obtained through fresh reads of the current loader state and metadata, rather than being guaranteed to use the initial state and metadata objects.

The transfer callback receives four arguments:

1. The callback-object pointer.
2. The current metadata buffer address.
3. The selected record count multiplied by 12, represented through the visible 32-bit arithmetic.
4. The literal value 2.

The last argument’s meaning is unknown.

The transfer result must equal the same selected-count-times-12 expression. A zero result is therefore sufficient only when the requested byte count is zero.

A mismatch returns 3. No rollback is shown: any buffer contents, object changes, or external-position changes already produced by the callbacks remain possible.

### 6. Transform loaded fields

Only an exact transfer-result match enters the transformation phase.

For every signed loop index beginning at zero and less than the selected record count, the function:

- Calls F-76bf07f4ac67 with the address of the record’s first 4-byte field.
- Stores the returned 32-bit value into that field.
- Calls F-76bf07f4ac67 with the address of the record’s second 4-byte field.
- Stores the returned 32-bit value into that field.

Records are processed in ascending index order. Within each record, the first field’s call and store precede the second field’s call and store.

The converter receives a field address, not merely the field’s existing 32-bit contents. Whether it reads, decodes, mutates, or otherwise interprets memory at that address is not established here.

The destination pointer chain is re-read after each converter call before the returned value is stored. Consequently, if that dependency changes the reachable state, metadata, or buffer pointer, the destination of the store can differ from the address supplied to the converter. A stable-buffer interpretation requires pointer stability across these calls; this function does not enforce it.

For a selected count of zero or less, the signed loop condition admits no iterations.

### 7. Publish the requested index

After the transformation loop, the function follows the current context-to-state-to-metadata chain and writes the requested index to metadata offset 4. It then returns 0.

This store is after all transformation calls and stores on a normal positive-count path. It is not accompanied by an explicit count update, completion flag, allocation, or buffer replacement.

## Side effects

### Directly established effects

The function directly:

- Calls one positioning callback when initial state is nonzero.
- Calls one transfer callback only after positioning returns zero.
- Calls F-76bf07f4ac67 twice per selected record when the selected count is positive and transfer succeeds exactly.
- Writes a returned 32-bit value to each of the first two fields of every processed record.
- Writes the requested index to metadata offset 4 on its success path.

It performs no explicit allocation, deallocation, or ownership transfer.

The interface is address-based. Valid backing storage must exist throughout every access. Nothing establishes that any dependency preserves objects, pointers, global state, or ownership.

### Failure and partial effects

Returning 3 does not mean that nothing changed:

- A failed positioning callback may already have changed external or shared state.
- A transfer-result mismatch may follow a partial or otherwise mutating transfer.
- Dependency effects are not undone.

There is no shown recovery mechanism for an invalid dereference or a dependency that fails to return. Such events do not have an established status-code mapping.

### Observability and synchronization

The explicit success-index store follows the per-record stores in the presented execution flow. There are no visible atomic operations, locks, memory barriers, or concurrency guarantees. The index store must not be treated as a cross-thread publication protocol based on this evidence alone.

## Dependencies

### F-76bf07f4ac67

Established interface at this call site:

- Input: an address corresponding to a 4-byte record field.
- Output: a 32-bit value, stored by the caller.
- Invocation order: first field, then second field, for each record in ascending order.

Its transformation, byte-order behavior, error behavior, pointer stability, memory ownership, and other side effects are unknown. Describing it as a particular endian conversion would exceed the evidence.

### Indirect positioning callback

No opaque function ID is supplied for this dynamic target.

Established contract at this call site:

- Resolved at callback-table offset 20.
- Receives the five arguments described above.
- A zero result permits the transfer attempt.
- Any nonzero result causes status 3.

Whether this operation is a seek, whether zero denotes success under a wider API, and what the final zero argument means remain inferred or unknown.

### Indirect transfer callback

No opaque function ID is supplied for this dynamic target.

Established contract at this call site:

- Resolved at callback-table offset 16.
- Receives the callback object, buffer address, requested byte count, and literal 2.
- Its result must exactly match the requested byte count.

Its use is consistent with a byte-count-returning read operation, but the direction of transfer, interpretation of 2, and complete error convention are not independently established.

## Acceptance examples

These are conditional static acceptance examples, not runtime test results. Unless stated otherwise, assume valid storage, stable pointer chains, ordinary non-overflowing signed arithmetic, and returning dependencies.

1. **Absent state**  
   If context offset 440 contains zero, the result is 3. Neither callback nor F-76bf07f4ac67 is invoked.

2. **Small successful window**  
   With stored count 10 and requested index 3, seven records are selected. The position is the base plus 52 bytes, subject to the described two-word arithmetic. A positioning result of zero and transfer result of 84 cause 14 converter calls, ordered by record and field. The requested index 3 is then stored, and the result is 0.

3. **Maximum-sized window**  
   With stored count 100,000 and requested index 0, 32,768 records are selected. The requested transfer size is 393,216 bytes. Exact callback success leads to 65,536 converter calls before the index store.

4. **Boundary below the cap**  
   With stored count 32,767 and requested index 0, the selected count is 32,767, not 32,768.

5. **Empty window**  
   With stored count 10 and requested index 10, the selected count is zero. Positioning still occurs, followed by a zero-byte transfer request. If both return zero, no converter calls occur; index 10 is stored and the result is 0.

6. **Positioning failure**  
   If positioning returns a nonzero value, the result is 3. There is no transfer call or direct metadata-index update by this function.

7. **Short transfer**  
   If seven records were selected but transfer returns 83 instead of 84, the result is 3. There are no converter calls or direct final index store. Transfer-produced buffer changes are not reverted.

8. **Low-position carry**  
   With a zero index and low base word `0xfffffff8`, the computed low position is 8 and the computed high position is the stored high word plus one, modulo its 32-bit representation.

9. **Index beyond the stored count**  
   With stored count 10 and requested index 11, the selected count is negative one. This is not rejected before the callbacks. The transfer byte-count expression has the 32-bit representation of negative 12. If the callback returns that matching representation, the loop executes zero times and the function can store index 11 and return 0. This illustrates missing validation, not an endorsed valid-input case.

## Uncertainties

The following are documented_only:

- The external meaning of metadata offset 0 and the final index field.
- Required buffer capacity and how it is allocated or retained.
- Whether callbacks preserve the context, loader state, metadata, or buffer pointers.
- The exact positioning-callback ABI, especially its self-address argument.
- The transfer direction and meaning of its literal argument 2.
- The transformation performed by F-76bf07f4ac67.
- Exact machine behavior for signed-overflow cases not resolved by the supplied representation.
- Evaluation order among argument-producing reads within a single call expression.
- Concurrency, reentrancy, cancellation, exception, and nonlocal-exit behavior.
- Whether invalid pointers or inaccessible storage produce a recoverable external error.

The ordinary “load and normalize a record window” interpretation is inferred intent. The explicit pointer accesses, callback gates, arithmetic widths, transformation order, and final store are the established behavioral basis.

## Evidence

This contract is derived solely from the supplied static function body and the supplied opaque dependency mapping.

The static evidence establishes a 12-byte record stride, a 32,768-record upper selection branch, two-word position construction, two indirect callback gates, two field-conversion calls per processed record, a final index store, and direct status values 0 and 3.

No dependency implementations, runtime traces, ABI documentation, or external format specification were supplied. No runtime tests have been performed.
