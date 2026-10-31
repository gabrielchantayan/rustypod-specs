<!-- Trusted provenance: F-aca60d9c975f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aca60d9c975f

## Purpose

F-aca60d9c975f selects a 32-bit return value through two ordered comparison gates and, if neither gate selects a result, a field-based fallback.

The established selection order is:

1. Obtain a value from F-3ad65987df9a and submit it to a comparison dependency with a fixed comparison operand.
2. If that comparison returns zero, return the first designated global value immediately.
3. Otherwise, obtain another value from F-44362605ab79 using an address derived from the input, and compare that value with a second global comparison operand.
4. If the second comparison returns zero, return a second designated global value immediately.
5. Otherwise, select between a value read from the input-associated storage and a third designated global value.

This is a selection contract, not an established string-lookup or history-management contract. Naming in the supplied material suggests a control-history association, but the function does not independently establish what the selected values represent.

## Interface

### Input

The function accepts one argument represented as a 32-bit integer in the supplied source view. The argument is used as an address base, not merely as a numeric selection key.

Two byte offsets are relevant:

| Offset from input base | Use |
|---|---|
| 40 bytes | Address supplied to F-44362605ab79 |
| 44 bytes | Location of a 32-bit value read directly on the final fallback path |

The supplied material does not establish the complete object layout, allocation size, type, alignment requirements, or lifetime. In particular, passing the address at offset 40 does not prove that this location contains a pointer, a string, or a standalone object. Its interpretation belongs to F-44362605ab79.

The address calculations are represented using integer arithmetic in the supplied source view. The contract does not prescribe wrapping, trapping, or validity behavior if adding an offset exceeds the applicable address range.

### Output

The function returns a value represented as a 32-bit integer. The returned value is selected from:

- A first global value after the first zero comparison result.
- A second global value after the second zero comparison result.
- The first direct read of the 32-bit field at offset 44.
- A third global value when the later test of that field observes zero.

No normalization, range restriction, or interpretation of these selected values is visible. A returned zero is not independently identified as an error or failure.

### Contract labels

The following labels identify data roles without asserting unpublished storage identities:

- **D-first-result:** the global result selected by the first comparison gate.
- **D-second-operand:** the global operand supplied to the second comparison.
- **D-second-result:** the global result selected by the second comparison gate.
- **D-fallback-result:** the global result selected when the final field test observes zero.
- **D-fixed-operand:** the fixed operand supplied to the first comparison.

These are descriptive labels within this document, not additional parameters or inferred data structures.

## Behavior

### First comparison gate

F-3ad65987df9a is called first, with no explicit arguments.

Its four-byte result is then passed as the first argument to the comparison dependency. D-fixed-operand is supplied as the second argument.

If the comparison dependency returns zero, F-aca60d9c975f obtains D-first-result and returns it. On this path, the function does not invoke F-44362605ab79, perform the second comparison, or directly read the field at offset 44.

If the comparison returns any nonzero value, processing continues. No distinction among positive, negative, or other nonzero representations is visible.

The zero result is only an established branch condition. Treating it as “equal strings” would require a contract for the comparison dependency that has not been supplied.

### Second comparison gate

After a nonzero first comparison result, the function computes the address 40 bytes beyond the input base and passes that address to F-44362605ab79.

The four-byte result of F-44362605ab79 becomes the first argument of another call to the same comparison dependency. D-second-operand supplies the second argument.

If this comparison returns zero, the function obtains D-second-result and returns it. The direct field reads at offset 44 are skipped.

If the comparison returns any nonzero value, processing reaches the final selection.

F-44362605ab79 completes before the second comparison operand is used for that comparison. Consequently, this specification does not treat D-second-operand as a value captured at function entry. A dependency may have changed relevant global state before the operand is obtained.

### Final field-based selection

The supplied source view represents two reads of the 32-bit field at offset 44:

1. An initial read supplies a candidate return value.
2. A subsequent read supplies the zero/nonzero condition.

If the second read observes zero, the candidate is replaced by D-fallback-result, and that global value is returned.

If the second read observes nonzero, the candidate from the first read is returned.

For stable storage, this reduces to the familiar result: return the field when it is nonzero; otherwise return D-fallback-result. Stability, however, is a precondition for that simplification, not an established property of the implementation.

The distinction matters if the two observations differ:

- An initial nonzero value followed by zero selects D-fallback-result.
- An initial zero followed by nonzero selects the initial zero.

These are conditional consequences of the represented reads. They do not establish that concurrent modification is supported or that the executable necessarily performs two distinct observable memory transactions.

### Ordering and early exit

The established call sequence is:

- F-3ad65987df9a.
- First comparison.
- Only after a nonzero first comparison: F-44362605ab79.
- Second comparison.
- Only after a nonzero second comparison: final field selection.

The first zero comparison result has priority over every later selection mechanism. The second zero comparison result has priority over the field and its fallback.

No retry, loop, alternate error path, or cleanup operation is visible.

## Side effects

### Direct reads

The function directly obtains the global result associated with the branch it selects. It also obtains D-second-operand when preparing the second comparison.

The field at offset 44 is reached only after both comparisons return nonzero. Its initial read precedes the later zero test in the supplied representation.

There is no evidence of an entry-time snapshot of global data. Result globals are obtained at their respective return branches, after the preceding calls have completed. The specification therefore does not require them to retain values they held before those calls.

### Writes and indirect effects

No direct assignment to caller storage or global storage is visible in F-aca60d9c975f.

This does not make the operation pure. F-3ad65987df9a, F-44362605ab79, and the comparison dependency have unspecified side effects. They may change global state or storage reachable through their arguments; the supplied material does not establish otherwise.

The input-derived address is passed to F-44362605ab79. Whether that dependency reads, modifies, retains, or otherwise uses the associated storage is documented_only.

### Ownership and lifetime

No allocation, explicit release, reference-count operation, or explicit ownership transfer is visible in this function.

Neither dependency results nor selected return values may be classified as owned, borrowed, or newly allocated from this evidence alone. If any four-byte value represents an address or handle, the corresponding lifetime and release rules remain unknown.

The function does not visibly release dependency results after comparison. That observation does not establish a leak: the dependencies’ ownership contracts are unavailable.

### Invalid inputs and failures

There is no explicit null check, bounds check, alignment check, or validity check for the input-derived addresses.

The first successful gate can return without direct use of the input-derived addresses. On later paths, address validity depends on the requirements of F-44362605ab79 and, if reached, the direct field reads.

A zero result from F-3ad65987df9a or F-44362605ab79 is still passed to the comparison dependency. The function contains no separate handling for such a result.

There is no visible error translation. Faults, exceptions, nonlocal exits, or other failures originating in dependencies or invalid memory access are not assigned a recovery behavior here.

## Dependencies

| Opaque function ID | Established interaction | Unknown contract |
|---|---|---|
| F-3ad65987df9a | Called first without explicit arguments; its four-byte result is the first comparison’s first operand. | Meaning, ownership, failure representation, and side effects. |
| F-44362605ab79 | Called only after the first comparison returns nonzero; receives the input-derived address at offset 40. Its four-byte result is the second comparison’s first operand. | Required layout, reads and writes, retention, ownership, and failure behavior. |
| F-unmapped-01 | Called for both comparison gates, each time with two operands; a zero result selects that gate’s global result. | Exact comparison semantics, accepted operand types, memory access, side effects, and failure behavior. |

F-unmapped-01 is a document-local opaque identifier for the comparison dependency. No published opaque mapping for that dependency was supplied. It must not be treated as a verified external identity.

The first comparison operand’s source label suggests a control-history-related string. Exact bytes, encoding, termination, and interpretation are not independently established. D-second-operand is likewise not proven to be a string or to have equivalent storage semantics.

## Acceptance examples

These examples are conditional acceptance criteria derived from the static material. They are not reports of executed tests. Calls are assumed to complete normally, and required accesses are assumed valid.

### First gate selects its result

Given that the first comparison returns zero and D-first-result yields 101 at the selected branch:

- The return value is 101.
- F-44362605ab79 is not called.
- The second comparison is not called.
- The field at offset 44 is not directly read by this function.

The contents of the field cannot override this result.

### Second gate selects its result

Given that the first comparison returns nonzero, F-44362605ab79 completes, the second comparison returns zero, and D-second-result yields 202:

- F-44362605ab79 receives the input base plus 40 bytes.
- The return value is 202.
- The final field selection is not performed.

### Stable nonzero field

Given that both comparisons return nonzero and both represented field reads yield 303:

- The return value is 303.
- D-fallback-result is not selected.

### Stable zero field

Given that both comparisons return nonzero, both field reads yield zero, and D-fallback-result yields 404:

- The return value is 404.
- A zero D-fallback-result would instead produce a zero return without further substitution.

### Different field observations

If the execution model permits the represented reads to observe different values:

- First read 505, second read zero: return the value obtained from D-fallback-result.
- First read zero, second read 606: return zero.

These examples preserve the distinction between the candidate value and the later branch condition. They do not authorize unsynchronized concurrent access.

### Changes during dependency calls

Given that the first comparison changes D-first-result and then returns zero, the result is obtained at the selected branch, not from an assumed pre-call snapshot.

Similarly, a change to D-second-operand during F-44362605ab79 is not excluded. The second comparison uses the operand obtained after that dependency completes.

## Uncertainties

The following remain documented_only:

- The original input type and complete object layout.
- Address width and address-overflow behavior beyond the supplied 32-bit integer representation.
- Endianness, alignment constraints, and executable-level memory transaction behavior.
- Whether the two represented field reads survive compilation as distinct accesses.
- Whether concurrent modification, signals, or memory-mapped storage are relevant or supported.
- Exact comparison semantics and the identity of F-unmapped-01.
- Exact contents and types of the comparison operands and result globals.
- Whether selected values are integers, pointers, handles, sentinels, or another four-byte representation.
- Dependency effects on global state, caller storage, ownership, and lifetime.
- Fault, exception, and nonlocal-exit behavior.

The inferred intent is an ordered special-case selection followed by a field-or-default choice. That interpretation is subordinate to the established branch conditions and read ordering.

## Evidence

This contract is based solely on the supplied static function representation and the supplied opaque dependency mappings.

Established evidence supports the two comparison gates, zero-result branch conditions, input offsets of 40 and 44 bytes, four-byte value flow, early returns, and the final candidate-read/condition-read distinction.

The supplied material does not include dependency bodies, complete data definitions, a verified comparison contract, or execution observations. No runtime tests have been performed. Acceptance examples are conditional deductions, not runtime evidence.
