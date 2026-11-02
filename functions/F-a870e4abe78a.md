<!-- Trusted provenance: F-a870e4abe78a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a870e4abe78a

## Purpose

F-a870e4abe78a populates three fields of a caller-provided descriptor using a referenced record: a first boundary, a quantity calculated from two boundaries, and an associated value. It then checks whether the captured associated value is zero and whether freshly read boundaries are reversed.

All three descriptor writes precede both checks. A failed check therefore does not prevent publication of the calculated fields. The routine provides no explicit rollback.

The calculation suggests an inclusive range length, but that interpretation is inferred intent. The established behavior is subtraction followed by addition of one, field copying, conditional diagnostic calls, and a zero return if execution reaches the return.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

### Descriptor

The routine accepts one address identifying a descriptor. The supplied representation uses four-byte integer slots.

| Byte offset | Access width | Role |
|---|---:|---|
| `0x00` | 4 bytes | Read once to obtain the address value of the referenced record. |
| `0x10` | 4 bytes | Receives the captured first boundary. |
| `0x14` | 4 bytes | Receives the calculated quantity. |
| `0x18` | 4 bytes | Receives the captured associated value. |

No direct access to descriptor bytes `0x04` through `0x0F` is shown. The layout above identifies accessed locations, not the descriptor’s complete size or type.

### Referenced record

The address value read from descriptor offset `0x00` identifies the base for these accesses:

| Byte offset | Access width | Role |
|---|---:|---|
| `0x174` | 4 bytes | First boundary; initially captured and later read again for comparison. |
| `0x178` | 4 bytes | Second boundary; initially used for calculation and later read again for comparison. |
| `0x17C` | 4 bytes | Associated value; captured, copied, and tested against zero. |

The boundary comparison uses signed 32-bit ordering in the supplied representation. Testing the associated value against zero does not establish its semantic type.

### Conditional global input

A separate global data object supplies the third explicit argument to the first diagnostic call. The routine evaluates that global’s stored value only when the captured associated value is zero.

This is a data read, distinct from supplying the diagnostic-format reference. The evidence does not establish the global value’s width, representation, meaning, or ownership.

### Return value

If execution reaches the return, the routine returns a 32-bit zero.

There is no distinct failure result for either failed check. A diagnostic call may prevent normal continuation, but the supplied body does not establish that it does.

### Preconditions

For completion without an invalid-memory access, the following locations must support their required accesses when reached:

- The descriptor’s initial four-byte address field.
- The three four-byte descriptor output fields.
- The referenced record fields used for copying, calculation, and later comparison.
- The global data object, if the captured associated value is zero.

The routine contains no explicit null-pointer, alignment, object-size, address-validity, or non-overlap checks. Requirements imposed by F-540af0155702 are not established.

The representation carries the record address in a four-byte integer value. Exact address-space rules, byte order, alignment requirements, integer-to-address conversion, and address-offset overflow behavior remain documented_only.

## Behavior

### Initial capture and ordered output writes

The routine first reads the record address from descriptor offset `0x00` and retains it for subsequent record accesses. It does not reload that descriptor field.

The directly established access order is:

1. Read the first boundary at record offset `0x174`.
2. Write that captured value to descriptor offset `0x10`.
3. Read the second boundary at record offset `0x178`.
4. Calculate the second boundary minus the captured first boundary, then add one.
5. Write the calculated result to descriptor offset `0x14`.
6. Read the associated value at record offset `0x17C`.
7. Write that captured value to descriptor offset `0x18`.

The calculation uses the first boundary captured before the first output write. It does not reread the first boundary for this calculation.

These three output writes occur before either diagnostic condition is evaluated. Their presence does not establish successful validation.

### First check and conditional global read

The first check tests the captured associated value against zero. It does not reread the record’s associated field or the descriptor’s copied field.

If the captured value is nonzero, the routine skips the first diagnostic call and the explicit global-data read used to prepare that call.

If the captured value is zero:

- All three descriptor output writes have already occurred.
- The routine reads the value of the separate global data object while preparing the diagnostic call.
- It invokes F-540af0155702 with the diagnostic-format reference, line marker 181, and the evaluated global data value.

The global read occurs before that invocation. The evidence does not require an ordering between evaluation of this global value and preparation of the other call arguments beyond those established boundaries.

The global value is not established as constant. Earlier descriptor writes may affect it through aliasing. Consequently, the argument is determined by the global read at this point, not by an assumed value at routine entry. Supplying the global’s value is also not equivalent to merely passing the address of that global object; the value itself could have an address-like representation, but that is unknown.

If F-540af0155702 does not return normally, the remaining behavior is not reached through normal continuation. If it returns normally, the routine proceeds to the boundary comparison.

### Second check using fresh boundary reads

The routine reads the second boundary at record offset `0x178` and the first boundary at record offset `0x174` again, comparing them with signed ordering.

These reads occur after the first check and any normal return from its diagnostic call. The evidence does not establish which comparison operand is read first.

If the freshly read second boundary is less than the freshly read first boundary, the routine invokes F-540af0155702 with the diagnostic-format reference and line marker 182.

This second call has two explicit arguments. It has no corresponding explicit global-data argument read in the supplied body. This does not establish that the dependency itself avoids reading global state.

The check does not compare the values previously used for the calculation. Output fields and the later comparison can therefore reflect different record states.

If normal execution continues through this check, the routine returns zero.

### Arithmetic boundaries

The calculated quantity has the mathematical form:

\[
\text{second boundary} - \text{captured first boundary} + 1.
\]

For cases in which both the subtraction and subsequent addition are representable as signed 32-bit values, this determines the stored result.

The evidence does not establish the applicable overflow semantics. It therefore does not justify assuming modulo-\(2^{32}\) wrapping, a trap, saturation, or another specific result when an arithmetic step overflows. Such cases remain documented_only.

The later ordering check does not protect the calculation:

- It occurs after the result has been written.
- It uses fresh reads.
- Ordered signed boundaries can still produce a mathematical count outside the signed 32-bit range.
- Representability of the final mathematical expression alone does not establish representability of each intermediate operation.

### Overlap and changing memory

No non-overlap requirement is enforced.

A descriptor output write may alter a record field that has not yet been read or will be reread. For example, the write to descriptor offset `0x10` precedes the initial read of the second boundary. If those locations overlap, the later read can observe the write.

Likewise, an output write may affect the global data object. On the zero-associated-value path, the global argument is read after all three writes and can reflect their effects.

Changing descriptor offset `0x00` through aliasing or dependency activity does not cause the routine to reload the record address. Subsequent accesses use the retained address.

No explicit synchronization or snapshot mechanism appears. The source does not establish safe concurrent-mutation conditions, atomicity guarantees, or a coherent relationship among the values copied, calculated, read globally, and later checked.

## Side effects

### Direct memory effects

On execution reaching the first check, the routine has written four-byte values to descriptor offsets `0x10`, `0x14`, and `0x18`.

No other direct stores appear in the supplied body. There is no explicit restoration of the previous contents.

Aliasing can make those stores affect other logical objects, including record fields or the global diagnostic datum. The source does not rule out such overlap.

### Diagnostic calls and continuation

The routine may make:

- One call for a captured associated value of zero.
- One call for freshly observed reversed boundaries.
- Both calls, in that order, if the first returns normally and the later comparison fails.

Only the first call requires the explicit global-data read described above. The second has no such explicit argument read.

The dependency’s output, memory effects, global-state effects, and control-flow behavior are unknown. It must not be assumed to terminate execution, preserve globals, preserve either object, or leave the output fields unchanged.

The routine does not itself recopy or recompute the output fields after either diagnostic call. Nevertheless, the fields’ values at return cannot be guaranteed against unknown dependency effects.

### Ownership

The supplied body contains no explicit allocation, deallocation, reference-count operation, or ownership transfer.

Neither the record address, the associated value, nor the global diagnostic value establishes ownership. In particular, copying a four-byte associated value does not establish that it is a pointer or resource handle.

Any allocation, release, or ownership effects inside F-540af0155702 remain documented_only.

## Dependencies

### F-540af0155702

This is the only called dependency shown.

| Trigger | Explicit arguments | Required preceding events |
|---|---|---|
| Captured associated value equals zero | Diagnostic-format reference; line marker 181; value read from a separate global data object | All three descriptor writes, followed by evaluation of the global data value before invocation |
| Fresh second boundary is signed-less-than fresh first boundary | Diagnostic-format reference; line marker 182 | First check and any normal return from its call, followed by fresh boundary reads |

The first call supplies three explicit arguments; the second supplies two. The evidence does not establish the dependency’s true signature or explain this difference.

The diagnostic-format reference indicates a failed-assertion reporting context. It is separate from the evaluated global value supplied only at the first call site. The global value’s meaning and representation are unknown; it must not be characterized as a constant or a mere reference to its storage.

The dependency’s formatting behavior, output destination, return convention, and effects remain documented_only.

## Acceptance examples

These are conditional static expectations, not observed executions. Unless stated otherwise, they assume valid memory, non-overlapping storage, unchanged inputs, and representable signed 32-bit arithmetic.

### Ordered boundaries

With first boundary 10, second boundary 14, and associated value 7:

- Descriptor offsets `0x10`, `0x14`, and `0x18` receive 10, 5, and 7, respectively.
- Neither diagnostic call occurs.
- The routine does not perform the explicit global-data read associated with the first diagnostic call.
- The routine returns zero.

### Equal or negative ordered boundaries

With both boundaries equal to 23 and a nonzero associated value, the calculated quantity is 1 and neither diagnostic call occurs.

With boundaries −4 and −2 and a nonzero associated value, the calculated quantity is 3 and neither diagnostic call occurs. No explicit check rejects boundaries merely for being negative.

Both cases return zero on normal completion.

### Zero associated value

With boundaries 10 and 14 and associated value zero:

- The outputs first receive 10, 5, and zero.
- The routine then reads the separate global data value.
- F-540af0155702 receives the diagnostic-format reference, line marker 181, and that evaluated value.
- If the dependency returns normally without changing relevant state, the later boundary check passes and the routine returns zero.

The global read is not an entry-time snapshot.

### Reversed boundaries

With first boundary 9, second boundary 4, and a nonzero associated value:

- The outputs receive 9, −4, and the associated value.
- The first diagnostic call and its explicit global-data read are skipped.
- The boundary comparison triggers the call with line marker 182.
- If that call returns normally, the routine returns zero.

The negative calculated quantity is stored before the reversal is diagnosed.

### Both checks fail

With reversed boundaries and associated value zero:

- All output writes occur first.
- The global data value is then read for the line-181 call.
- If that call returns normally and the freshly read boundaries remain reversed, the line-182 call follows.
- No corresponding explicit global-data argument read accompanies the second call.
- Zero is returned only if execution reaches the return normally.

### Aliasing affects the diagnostic argument

Suppose a descriptor output write overlaps the global data object and the captured associated value is zero.

The first diagnostic call receives the value obtained by reading that global after all three output writes. An acceptance expectation based solely on the global’s entry value is invalid. The exact resulting argument depends on the overlap, representation, and stored bytes, which are not supplied.

### Dependency changes the record

Suppose the first diagnostic call occurs and returns after changing the record’s boundaries.

The later comparison uses fresh boundary reads at the retained record address. The routine does not recompute the count or recopy the first boundary. Diagnostic success or failure can therefore concern different values from those used to populate the descriptor.

### Count exceeds signed 32-bit range

With boundaries zero and 2,147,483,647, the mathematical inclusive count is 2,147,483,648.

The boundaries are ordered, but the addition exceeds the signed 32-bit range. The exact stored result and continuation behavior are documented_only. An assumed wrapping result is not an established acceptance criterion.

## Uncertainties

The following remain documented_only:

- Complete object types, sizes, alignment, and semantic roles.
- Arithmetic overflow behavior and address-arithmetic rules.
- Byte order and integer-to-address interpretation.
- The global diagnostic datum’s value, width, representation, meaning, and ownership.
- Whether overlapping storage is supported by the surrounding system.
- Concurrency and memory-model guarantees.
- Fault behavior for inaccessible or misaligned memory.
- The dependency’s signature, formatting, output, return behavior, and memory or ownership effects.

Inclusive-range intent is inferred. Conditional evaluation of a global data value for the first diagnostic call is established; its interpretation is not.

## Evidence

The supplied static body establishes:

- Four-byte descriptor and record accesses at the documented offsets.
- A retained record address.
- Ordered publication of three output fields before checking.
- A zero check using the captured associated value.
- A global data read only on that zero-value path, after all output writes and before the first dependency invocation.
- Separate diagnostic-format and global-value arguments at the first call.
- Fresh signed boundary comparison and a second call without a corresponding explicit global-data argument read.
- A 32-bit zero return when reached.

The supplied material does not include the dependency implementation, callers, complete object definitions, global datum definition, or runtime observations.

No runtime tests have been performed.
