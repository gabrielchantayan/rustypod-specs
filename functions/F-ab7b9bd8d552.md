<!-- Trusted provenance: F-ab7b9bd8d552; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ab7b9bd8d552

## Purpose

F-ab7b9bd8d552 selects and returns a 32-bit value using two ordered comparison gates, followed, if necessary, by a field-based fallback.

The first gate compares a value obtained from F-3ad65987df9a against fixed text identified as “CntrlHistoryFn”. A zero comparison result selects a dedicated global return value and ends processing.

If the first gate does not select a result, the function passes the input-derived address at byte offset 40 to F-44362605ab79. It compares that dependency’s result against a separate global comparison operand. A zero result selects a second global return value.

Only if both comparisons return nonzero does the function inspect the 32-bit field at byte offset 44. The supplied representation contains two reads of that field. The first supplies the candidate return value; the second determines whether to replace that candidate with a global default.

This is a selection contract, not an established string-processing or ownership contract. The names and surrounding structure suggest history-related dispatch, but that interpretation is inferred intent rather than a prerequisite for the behavior below.

## Interface

### Input

The function accepts one argument, represented in the supplied evidence as an integer. It is used to form addresses rather than as a direct selection value:

- Input plus 40 bytes is passed to F-44362605ab79.
- Input plus 44 bytes identifies a 32-bit field read on the final fallback path.

These are byte offsets, not indices into an array of 32-bit elements. The two locations begin four bytes apart. The evidence does not establish the size or type of the object beginning at offset 40, so adjacency does not establish that the two locations are independent fields.

The contract requires valid address formation and accessible storage for whichever input-derived accesses are actually reached. The first successful gate avoids all explicit input-derived accesses in this function. The second gate requires the offset-40 address to be acceptable to its dependency, but it avoids this function’s explicit offset-44 reads when selected.

The evidence does not establish a null-input check, bounds check, object-type check, or alignment repair.

### Output

The function returns one 32-bit value, selected from:

1. A global value associated with the first comparison gate.
2. A global value associated with the second comparison gate.
3. The first value read from the field at input offset 44.
4. A global default used when the second read of that field is zero.

No special error result is established. In particular, zero can be returned if the selected source contains zero.

The return declaration is integer-shaped. Some returned values could represent handles, addresses, flags, or ordinary integers, but the evidence does not distinguish those interpretations. Consumers must not infer ownership or pointer validity from the return mechanism alone.

### Width and arithmetic limits

The supplied representation explicitly includes a 32-bit dependency result and integer-shaped field reads and returns. This contract describes the field and selection values as 32-bit quantities.

The input is also represented as an integer used in address arithmetic. The exact address width, signed-overflow behavior, pointer-conversion rules, alignment requirements, and byte order are not independently established. The offsets are small positive constants, but this does not make all possible input values valid.

For acceptance purposes, inputs must permit the reached address calculations without relying on an unspecified overflow or address-conversion rule. This contract does not prescribe wraparound at an address-space boundary.

## Behavior

### Ordered selection

The following labels identify storage roles for this document; they are not implementation identifiers:

| Label | Role |
|---|---|
| First result | Global 32-bit value returned when the first comparison returns zero |
| Second operand | Global value supplied as the second operand of the second comparison |
| Second result | Global 32-bit value returned when the second comparison returns zero |
| Default result | Global 32-bit value used when the final field check reads zero |

Selection proceeds in strict gate order. A later gate cannot override an earlier successful gate.

### First comparison gate

The function first invokes F-3ad65987df9a without explicit arguments.

It then supplies that dependency’s 32-bit result, together with the fixed-text operand identified as “CntrlHistoryFn”, to an unresolved comparison dependency.

If the comparison returns zero:

- The function reads the First result global.
- It returns that value.
- It does not invoke F-44362605ab79.
- It does not perform the second comparison.
- It does not perform either explicit read at input offset 44.

The First result is obtained after the comparison completes. The evidence does not establish that either preceding dependency preserves that global.

If the comparison returns any nonzero value, processing continues. Positive and negative comparison results are not treated differently.

A zero result is the operative selection condition. Describing it as textual equality would require a contract for the comparison dependency that is not supplied here.

### Second comparison gate

After the first comparison returns nonzero, the function forms the address at input plus 40 bytes and passes it to F-44362605ab79.

The resulting 32-bit value becomes one operand of the same unresolved comparison dependency. The other operand comes from the Second operand global.

The Second operand is used after F-44362605ab79 has returned. It must not be assumed to retain a value observed before that call: no preservation guarantee for the dependency is available.

If this comparison returns zero:

- The function reads the Second result global.
- It returns that value.
- It performs neither explicit read at input offset 44.

The Second result is read after this comparison completes. A dependency’s possible changes to that storage cannot be excluded by the available evidence.

If the comparison returns nonzero, execution reaches the final field-based selection.

### Final field-based selection

After both comparisons return nonzero, the supplied representation performs two reads of the same 32-bit field at input plus 44 bytes.

Their roles differ:

1. The first read obtains the candidate return value.
2. The second read determines whether the Default result global must replace that candidate.

If the second read is zero, the function reads and returns the Default result global.

If the second read is nonzero, the function returns the value obtained by the first read.

This is not equivalent, under changing storage, to reading once and returning that value unless it is zero. In particular:

- A nonzero first read followed by a zero second read selects the Default result.
- A zero first read followed by a nonzero second read returns zero.
- Two different nonzero reads return the first value, not the second.

No dependency call is shown between these two reads. Nevertheless, the supplied representation distinguishes them. Whether the original executable guarantees two separate observable memory transactions, and whether external modification between them is supported, require evidence beyond this representation.

For stable ordinary memory, the behavior simplifies: a nonzero field value is returned, while a zero field value selects the Default result.

### Completion and failures

All successful paths return immediately after selecting their result. No retry, cleanup call, logging call, or explicit error translation is shown.

The evidence does not specify behavior when:

- A dependency fails to return normally.
- An input-derived address is invalid.
- A dependency cannot interpret an operand.
- Global storage is inaccessible or corrupted.
- Address arithmetic exceeds the applicable machine or language limits.

These conditions must not be assigned a fabricated fallback result. The Default result is selected only by the explicit zero check on the second field read, not by a demonstrated general failure handler.

## Side effects

No direct memory write is shown in F-ab7b9bd8d552. Its explicit operations consist of dependency calls, address formation, reads, comparisons against zero, and returning a selected value.

That does not establish that the function is pure. Any invoked dependency may have side effects not documented in the supplied evidence, including changes to globals or input-associated storage.

The relevant observable ordering is:

- F-3ad65987df9a precedes the first comparison.
- The first comparison precedes any read of the First result used for return.
- A nonzero first comparison precedes F-44362605ab79.
- F-44362605ab79 precedes use of the Second operand in the second comparison.
- The second comparison precedes any read of the Second result used for return.
- Two nonzero comparisons precede the two explicit offset-44 reads.
- A zero second field read precedes the read of the Default result.

Unselected return globals are not explicitly read by this function’s corresponding return paths. Dependencies may access them independently; their implementations are unavailable.

No allocation, release, reference-count adjustment, or ownership transfer is explicit. Passing an input-derived address to F-44362605ab79 does not prove that the dependency borrows it, owns it, retains it, mutates it, or releases it. Likewise, returning a global or field value does not prove transfer of any resource represented by that value.

## Dependencies

| Opaque function ID | Established role | Unestablished behavior |
|---|---|---|
| F-3ad65987df9a | Called first, without explicit arguments; supplies a 32-bit operand for the first comparison | Meaning of the result, failure behavior, ownership, and effects on state |
| F-44362605ab79 | Called only after a nonzero first comparison; receives input plus 40 bytes and supplies a 32-bit operand for the second comparison | Required storage layout, reads and writes, retention or release of memory, and failure behavior |
| F-unresolved-01 | Document-local opaque ID for the same unresolved comparison dependency used at both gates; a zero result selects the corresponding gate | Published identity, operand types, comparison semantics, null handling, ownership, and side effects |

F-unresolved-01 is a document-local designation, not a supplied provenance mapping.

Neither its call shape nor the fixed text proves a particular library comparison API. Only the caller’s use of zero versus nonzero is established.

## Acceptance examples

These are conditional acceptance cases derived from static evidence. They have not been executed.

### First gate takes precedence

Assume F-3ad65987df9a returns an operand and F-unresolved-01 returns zero for the first comparison. If First result contains 23 when subsequently read, the function returns 23.

F-44362605ab79 and the second comparison are not reached. This function performs no explicit input-derived field read. Even a nonzero value at offset 44 cannot override the first gate.

This case does not establish that arbitrary input values are safe if a dependency independently accesses related state.

### First result is read after the comparison

Assume the first comparison returns zero and changes First result from 23 to 91 before returning. Provided no other change intervenes, the function returns 91.

Returning a previously cached value of 23 would not match the shown ordering.

### Second gate takes precedence over the field

Assume the first comparison returns nonzero. F-44362605ab79 receives the address 40 bytes beyond the input. The second comparison returns zero, and Second result contains 37 when read.

The function returns 37 without either explicit offset-44 read. A zero or nonzero field value does not affect this selection.

### Second operand reflects the preceding dependency

Assume F-44362605ab79 changes Second operand before returning. The second comparison uses the global operand obtained after that call, not a value assumed to have been preserved from entry.

The comparison’s outcome remains dependent on the unresolved dependency’s actual semantics.

### Stable nonzero field

Assume both comparisons return nonzero and both offset-44 reads yield 12. The function returns 12 and does not explicitly read Default result.

The signs and magnitudes of the nonzero comparison results do not change this outcome.

### Stable zero field

Assume both comparisons return nonzero, both field reads yield zero, and Default result contains 64 when read. The function returns 64.

If Default result instead contains zero, the function returns zero. No additional fallback is shown.

### Different field observations

Conditionally, if the execution environment permits distinct observations between the two field reads:

| First read | Second read | Required selection |
|---:|---:|---|
| 12 | 0 | Default result |
| 0 | 12 | Zero from the first read |
| 12 | 19 | 12 from the first read |

These cases distinguish the supplied representation from a single-read fallback rule. They do not assert support for unsynchronized concurrent mutation or any particular memory model.

## Uncertainties

The following items are **documented_only**:

- **Comparison semantics:** zero triggers selection, but equality, encoding, case sensitivity, operand representation, and invalid-operand handling are unknown.
- **Fixed-text representation:** the supplied data identifies “CntrlHistoryFn”; its complete storage representation and comparison interpretation are not independently established.
- **Object layout:** only the two byte offsets and the 32-bit field access are available. A complete object schema cannot be inferred.
- **Address model:** exact pointer width, conversion rules, arithmetic-overflow behavior, alignment requirements, and byte order are unavailable.
- **Dependency effects:** no dependency is established to preserve globals or input storage, or to follow any ownership convention.
- **Double-read observability:** two reads appear in the supplied representation; original instruction behavior and supported inter-read mutation require further evidence.
- **Global contents and mutability:** the source provides storage references, not established values, initialization rules, or synchronization guarantees.
- **Failure behavior:** no explicit validation or recovery contract is shown.
- **Domain intent:** history-related selection is suggested by the text operand, but is not necessary to the established selection behavior.

## Evidence

The evidence consists solely of the supplied function representation and the supplied mappings for F-3ad65987df9a, F-44362605ab79, and F-ab7b9bd8d552.

Established static evidence supports the ordered calls, zero/nonzero gates, byte offsets 40 and 44, selected global reads, two represented reads of the final field, and absence of explicit writes or resource-management operations in this function.

Interpretations involving textual equality, resource ownership, object identity, concurrency guarantees, or domain-specific meaning are not established by that evidence.

No implementation access, dependency inspection, executable inspection, or runtime tests have been performed. The acceptance examples are conditional behavioral obligations, not reported test results.
