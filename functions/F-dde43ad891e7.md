<!-- Trusted provenance: F-dde43ad891e7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dde43ad891e7

## Purpose

F-dde43ad891e7 partially initializes entry records in a caller-supplied object, derives per-entry quantities through opaque dependencies, and performs checked operations on opaque values.

The established behavior is nontransactional:

- The initial unsigned entry count must be below five.
- The accumulated entry weights must be nonzero.
- Kinds zero, one, and unrecognized kinds take a common processing path.
- Kinds two and three receive only a classification-byte write from this function.
- Zero is returned only when the loop’s live signed-count termination check succeeds.
- Initial rejection or a nonzero result from a checked dependency produces return value three.

Completing an individual entry advances the index; it does not itself return zero. A later entry can still cause return value three.

Weighted distribution is an inferred intent, not an established mathematical contract. The numeric dependencies and opaque-value operations have no supplied behavioral definitions.

## Interface

### Argument and result

The sole argument identifies mutable object storage. The supplied representation uses 32-bit address arithmetic, 32-bit scalar fields, and byte-sized classification fields.

The result is a 32-bit value:

| Result | Established condition |
|---|---|
| 0 | At a loop boundary, the current index is greater than or equal to the live working count under signed comparison. |
| 3 | The initial unsigned count is at least five, the accumulated weight is zero, or a checked dependency returns nonzero. |

No distinct error result identifies which dependency failed. Invalid memory access, arithmetic overflow, or a dependency that does not return has no separately established result.

The function performs no explicit null, extent, alignment, handle-validity, or output-address validation. Storage must be accessible for every reached read and write. Dependencies impose additional unknown preconditions.

### Accessed object layout

Offsets are relative to the argument address. These accesses do not establish a complete object type.

| Offset | Width | Use |
|---|---:|---|
| 0x1060 | 32 bits | Context read for opaque-value dependencies |
| 0x1064 | 32 bits | Initial carried opaque value |
| 0x1068 | 32 bits | Existing opaque value; conditionally replaced |
| 0x1074 | 32 bits | Accumulated quantity; updated before entry processing |
| 0x1080 | 32 bits | Working count; checked live as signed during processing |
| 0x1088 | 32 bits | Source count; initially checked as unsigned |
| 0x1394 | 32 bits | Opaque preliminary-dependency argument |
| 0x13b4 | 32 bits | Weight accumulator and later numeric input |

Input entry index *i* has stride 0x60:

| Offset | Width | Use |
|---|---:|---|
| 0x1090 + 0x60 × *i* | 8 bits | Kind |
| 0x1094 + 0x60 × *i* | 32 bits | Weight |

Output entry index *i* begins at 0x1240 + 0x50 × *i*:

| Relative offset | Width | Use |
|---|---:|---|
| 0x00 | 32 bits | Copy of the carried opaque value |
| 0x04 | 32 bits | Dependency output; subsequently read and overwritten with 0xffffffff |
| 0x08 | 32 bits | Dependency output and later dependency argument |
| 0x0c | 32 bits | Copy of the carried opaque value |
| 0x10 | 32 bits | Cleared |
| 0x14 | 32 bits | Selected quantity; directly written only after all checked entry calls succeed |
| 0x18 | 32 bits | Cleared before numeric calls |
| 0x1c | 32 bits | Copy of the carried opaque value |
| 0x24 | 32 bits | Written as 0xffffffff |
| 0x38 | 8 bits | Written as two |
| 0x3c | 32 bits | Numeric-combination result |
| 0x4e | 8 bits | Classification |

Other output bytes have no established direct initialization.

The initial count permits at most four input entries. It does not permanently bound processing: dependencies may change the working count. Accessible storage requirements therefore depend on the actual live count and dependency behavior.

## Behavior

### Initial writes and rejection

The function reads the source count at 0x1088, writes that captured value to 0x1080, and then clears 0x13b4.

If the captured count is at least five under unsigned comparison, the function returns three. These initial writes precede rejection; no dependency has been invoked.

For a count from zero through four, the function accumulates the corresponding weights into 0x13b4 in ascending entry-index order. Each accumulation reads the current accumulator and the current entry’s 32-bit weight, adds them, and writes the result to the accumulator. The supplied representation does not independently establish an ordering between the two operand reads within an addition.

After accumulation, a zero value at 0x13b4 causes return value three. A source count of zero necessarily takes this rejection path.

There is no positivity check on individual weights or on the aggregate. Negative values and cancellation are not separately rejected. Arithmetic limits are documented under Uncertainties.

### Preliminary dependency stage

For a nonzero aggregate, F-e162eb24ee4a receives:

1. The current context at 0x1060.
2. The constant 0x80.
3. An address for a 32-bit quantity output.
4. An address for a first 32-bit opaque output.
5. An address for a second 32-bit opaque output.
6. The current value at 0x1394.

The three output slots have no visible initialization before this call. The dependency’s direct return value is ignored.

If the quantity output is strictly positive under signed comparison, F-e3a1bffa31a1 is called with the current context, the current value at 0x1068, and the first opaque output.

- A nonzero result immediately produces return value three.
- A zero result is followed by a direct write of the second opaque output to 0x1068.

For a zero or negative quantity, this checked call and the replacement write are skipped.

Next, the function adds the preliminary quantity to the then-current field at 0x1074. It writes the result to 0x1074 and retains the same result as a private working remainder. It then reads 0x1064 to initialize a carried opaque value.

Subsequent remainder reductions do not themselves update 0x1074. Conversely, later dependency changes to 0x1074 do not automatically change the retained remainder.

### Live loop check and kind dispatch

Processing begins with index zero. Before every entry, the function rereads 0x1080 as a signed 32-bit count.

If that count is less than or equal to the index, the function returns zero. Thus, after successful preliminary processing, a dependency-modified count of zero or a negative count can terminate the loop before any entry is processed.

Otherwise, the function reads the current entry’s kind:

| Initial kind | Direct classification write | Subsequent path |
|---|---|---|
| 0 | Zero | Common processing |
| 1 | One | Common processing |
| 2 | Two | Advance index |
| 3 | Three | Advance index |
| Any other byte value | None | Common processing |

For kinds two and three, the function performs no other direct output-record writes and invokes no per-entry dependency. These paths do not change its retained remainder or carried opaque value.

An unrecognized kind does not cause rejection. The function leaves its classification byte untouched at dispatch and proceeds through common processing.

### Numeric processing

The common path first clears output offset 0x18. It then performs the following ordered operations:

1. Call F-1c7710595b35 with the current value at 0x13b4.
2. Call F-1c7710595b35 with the current entry’s weight.
3. Call F-450415463608 with the weight-derived result first and the aggregate-derived result second.
4. Write that result to output offset 0x3c.
5. Call F-1c7710595b35 with the current value at 0x1074.
6. Call F-fdb31af1de99 with this latest conversion result first and the retained F-450415463608 result second.
7. Call F-c9e0501fef5b without explicit arguments and interpret its result as a signed 32-bit quantity.

The second argument to F-fdb31af1de99 is the retained combination result, not a fresh read of output offset 0x3c. If the intervening conversion call modifies that output field, the modification does not replace the retained argument.

No numeric-call result is checked for an error indication. The direct return from F-fdb31af1de99 is unused. Any hidden-state relationship between it and F-c9e0501fef5b is unknown.

A zero result from F-c9e0501fef5b is replaced with one. Negative results are not otherwise rejected.

The function then rereads the entry kind. This read occurs after all numeric calls, so it can differ from the dispatch read.

The following direct writes occur in order:

1. Carried opaque value to output offset 0x00.
2. 0xffffffff to offset 0x24.
3. Carried opaque value to offset 0x1c.
4. Zero to offset 0x10.

If the reread kind is zero, the candidate quantity is incremented by one. The function then writes the carried opaque value to offset 0x0c.

The candidate is capped using signed comparison with the retained remainder: if the remainder is smaller, the remainder becomes the selected quantity. This is an upper cap, not a lower-bound clamp. Zero and negative selected quantities are possible.

Finally, the function writes byte value two to output offset 0x38.

### Checked opaque-value operations

F-d31de91d1e1b receives:

1. The current context.
2. The selected quantity minus one.
3. The carried opaque value.
4. The address of output offset 0x08.

A nonzero result returns three immediately.

On success, F-8fce2b40d724 receives the current context, the then-current value at output offset 0x08, and the address of output offset 0x04. A nonzero result returns three immediately.

On that call’s success, the function reads output offset 0x04 into its carried opaque value, then directly overwrites offset 0x04 with 0xffffffff.

It next calls F-e3a1bffa31a1 with two explicit arguments: the current context and the then-current value at output offset 0x08. A nonzero result returns three.

Only after this final checked call succeeds does the function subtract the selected quantity from its retained remainder and directly write the selected quantity to output offset 0x14. It then advances the index and repeats the live count check.

The sentinel write to offset 0x04 occurs before the final call. It does not guarantee that the field still contains the sentinel after that call: the dependency may modify object storage. Likewise, failure prevents this function’s selected-quantity write to offset 0x14, but does not prevent dependencies from having written that field.

## Side effects

The function directly modifies caller-supplied storage in the order described. It performs no compensating restoration on any checked failure.

This lack of restoration is distinct from a guarantee about final contents. Dependencies may overwrite earlier fields, including fields in already processed entries. Consequently, earlier direct writes establish events in the execution, not immutable post-call storage values.

Important boundaries are:

- Count rejection occurs after copying the source count and clearing the weight accumulator.
- Zero-sum rejection occurs after the accumulation writes and before any dependency call.
- Failure of the preliminary checked call prevents this function’s replacement of 0x1068 and its subsequent update of 0x1074. The failing dependency’s own effects remain unknown.
- Entry failures can occur after classification, numeric-result, sentinel, and carried-value writes.
- Dependencies receiving output addresses may write through them before reporting failure.
- Before the final entry call, this function writes 0xffffffff to offset 0x04.
- If that final call fails, this function does not perform the selected-quantity write to offset 0x14 or the remainder subtraction.
- A later failure does not trigger restoration of earlier entries, but dependencies may already have changed them.

The supplied body establishes no direct allocation, deallocation, ownership transfer, or lifetime extension. Opaque values must not be characterized as owned or released solely from these calls. In particular, neither invocation form of F-e3a1bffa31a1 establishes a release operation.

Context values are reread for successive opaque-value calls. Numeric inputs, kinds, and the live count are also read at their described points rather than treated as a complete initial snapshot.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-e162eb24ee4a | Context, 0x80, three 32-bit output addresses, and an opaque field; direct result ignored |
| F-e3a1bffa31a1 | Three explicit arguments during preliminary processing; two during entry processing; nonzero result causes return three |
| F-1c7710595b35 | One 32-bit input; returns a 32-bit value consumed by numeric dependencies |
| F-450415463608 | Two numeric-dependency results; returned value is retained and written to the output record |
| F-fdb31af1de99 | Converted accumulated quantity and retained combination result; direct result unused |
| F-c9e0501fef5b | No explicit arguments; result used as a signed 32-bit quantity |
| F-d31de91d1e1b | Context, selected quantity minus one, carried value, and output address; nonzero result causes return three |
| F-8fce2b40d724 | Context, current opaque field, and output address; nonzero result causes return three |

The differing explicit argument counts for F-e3a1bffa31a1 are preserved without inventing a unified signature. Argument listings establish positional roles, not an independently proven evaluation order among argument reads.

No dependency contract guarantees preservation of globals, object fields, output slots, or external state.

## Acceptance examples

These are conditional static expectations, not executed tests. Except where mutation is explicitly introduced, assume dependencies initialize consumed outputs, preserve other object storage, and return as specified. Arithmetic examples avoid overflow.

1. **Initial count five.**  
   The function writes five to 0x1080, clears 0x13b4, and returns three without invoking a dependency.

2. **Initial count zero.**  
   The function writes zero to both initial destination fields and returns three because the weight accumulator remains zero.

3. **Cancelling weights.**  
   Count two with weights seven and negative seven produces a zero aggregate and return three before preliminary dependency processing.

4. **Only kinds two and three.**  
   With a nonzero aggregate and successful preliminary processing, the function writes classification bytes two and three. With the working count unchanged at two, the next live count check returns zero. No common-path entry call occurs.

5. **Kind zero with zero numeric result.**  
   With both kind reads equal to zero, numeric result zero, and remainder ten, zero becomes one and the kind-zero increment makes the candidate two. The first checked entry call receives one as its quantity argument. If all checked entry calls succeed, the function writes two to offset 0x14 and reduces its remainder to eight.

6. **Kind one with zero numeric result.**  
   With both kind reads equal to one and remainder ten, the selected quantity is one. The first checked entry call receives zero. Successful completion writes one to offset 0x14.

7. **Zero remainder.**  
   A positive candidate is capped to zero. The first checked entry call receives negative one. No local check rejects this argument.

8. **Unrecognized kind.**  
   Stable kind four takes the common path without a classification write and without the kind-zero increment.

9. **Successful entry followed by failure.**  
   With working count two, successful completion of entry zero advances the index to one. If a checked call for entry one returns nonzero, the function returns three, not zero. There is no compensating restoration.

10. **Final entry call fails without mutating storage.**  
    After the first two checked entry calls succeed, the function captures offset 0x04 and writes the sentinel there. If the final call returns nonzero without changing storage, the sentinel remains, and offset 0x14 retains its previous contents.

11. **Final entry call fails and mutates storage.**  
    If that final call writes 0x04 or 0x14 before returning nonzero, those writes are not undone. The unconditional guarantees are the earlier sentinel-write event, absence of this function’s selected-quantity write after failure, and return value three—not particular final field contents.

12. **Live count changed before processing.**  
    If preliminary dependency activity changes 0x1080 to zero while allowing preliminary processing to complete, the first loop check returns zero without processing an entry.

## Uncertainties

### documented_only: Arithmetic boundaries

Count eligibility is unsigned. Preliminary positivity, loop termination, and quantity capping use signed comparisons. Relevant quantities and fields are represented as 32 bits; no widening arithmetic is established.

Overflow behavior is not independently resolved for weight accumulation, accumulated-quantity addition, kind-zero increment, selected quantity minus one, remainder subtraction, index advancement, or address calculations. Saturation and guaranteed modular arithmetic must not be assumed.

### documented_only: Dependency semantics

Conversion, ratio formation, multiplication, and integer extraction are suggested by call structure but are not established operations. Precision, rounding, exceptional values, and hidden-state communication remain unknown.

The preliminary outputs are not visibly initialized before the dependency call. A checked return does not validate them, because that call’s direct result is ignored. Behavior when a consumed output was not supplied is unresolved.

Dependencies may change live counts, kinds, weights, context, numeric fields, and output records. Enlarging the live count can exceed the initial four-entry bound. Aliasing, retained output addresses, ownership obligations, and cleanup requirements are unknown.

## Evidence

Established static evidence consists of the supplied body’s widths, offsets, explicit accesses, branches, call order, positional arguments, checked results, and return paths.

Weighted distribution and handle-management roles are inferred intent only. Dependency mathematics, resource semantics, calling-convention resolution, overflow behavior, and storage preservation across dependencies remain documented-only unknowns.

No implementation access, runtime execution, or runtime tests were used.
