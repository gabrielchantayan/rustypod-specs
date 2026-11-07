<!-- Trusted provenance: F-0edb66a9a85d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0edb66a9a85d

## Purpose

F-0edb66a9a85d selects a record from a forward-linked sequence using two equality keys and a distance from a requested numeric value. If the retained selection is absent or fails a final distance threshold, it instead initializes a record obtained from a context object. It then passes the selected record and context to F-d4c4416ace75 and returns the selected record pointer.

The supplied static evidence does **not** establish an ordinary four-argument, self-contained search. Execution also depends on an incoming current-record pointer, retained-selection pointer, retained distance, candidate value, and condition flags. These are required entry-state inputs, not values initialized by the visible body.

The selection is not necessarily the globally nearest eligible record. A newly improved distance of one terminates traversal immediately, even if a later record would match exactly.

## Interface

### Explicit inputs

The visible interface contains four 32-bit arguments:

| Input | Interpretation established by use |
|---|---|
| First key | A 32-bit equality key. On the fallback path, it is also used as an address from which a 16-bit field is read at byte offset 2. |
| Second key | A 32-bit equality key, also copied into two fallback-record fields. |
| Requested value | An unsigned 32-bit value used in comparisons and distance calculations. It is not explicitly restricted to 16 bits. |
| Context address | A 32-bit address passed to F-d4c4416ace75. On the fallback path, a record pointer is read at byte offset 8. |

The first key has a dual role: opaque comparison value during traversal and address during fallback initialization. A value suitable only for equality comparison is insufficient if fallback can occur.

### Additional entry-state inputs

The following incoming machine state affects behavior:

- Current record pointer.
- Candidate numeric value, represented as an unsigned 32-bit value.
- Retained record pointer.
- Retained distance, represented as an unsigned 32-bit value.
- Carry and zero condition flags.

The initial candidate value is not visibly loaded from the initial current record. The initial current record is also not visibly checked against the two keys or the sentinel before its distance is considered.

A complete callable contract therefore requires either an external entry convention defining this state or documentation of the preceding execution that establishes it. Neither is supplied.

### Return value

The function returns the retained or fallback record pointer selected before the dependency call. The dependency’s return value is not used.

This pointer is not evidence that the record remains unchanged after F-d4c4416ace75. That dependency may modify the record or other state; its behavior is not supplied.

### Record layout

The observed layout uses 32-bit pointers and 32-bit integer fields. Offsets are in bytes from the record address.

| Offset | Width | Observed role |
|---:|---:|---|
| 0 | 32 bits | Next-record pointer, read during traversal |
| 8 | 32 bits | First equality key; written during fallback |
| 12 | 16 bits | Candidate value; signed sentinel check and unsigned numeric read during traversal; cleared during fallback |
| 24 | 32 bits | Second equality key; written during fallback |
| 28 | 32 bits | Cleared during fallback |
| 32 | 16 bits | Cleared during fallback |
| 34 | 16 bits | Set to zero or one during fallback |
| 36 | 32 bits | Copy of second key during fallback |
| 40 | 16 bits | Cleared during fallback |

No accesses to other record bytes are established. In particular, fallback initialization does not visibly clear the entire record or overwrite its next-record pointer.

## Behavior

### 1. Evaluate the incoming candidate

The first action is a distance calculation using the incoming candidate value, requested value, and incoming flags:

- If carry is clear **or** zero is set, subtract the requested value from the candidate value.
- Otherwise, subtract the candidate value from the requested value.
- Retain only the low 16 bits of the subtraction result.

Both subtraction operands are 32-bit unsigned values. Subtraction wraps at 32 bits, and the result is then reduced to the range 0 through 65,535.

The reduced distance is compared with the retained distance using unsigned comparison. If it is strictly smaller:

1. The current record becomes the retained record.
2. The reduced distance becomes the retained distance.
3. If that distance equals one, traversal stops immediately.

A tie does not replace the retained record.

This initial step is materially different from later candidate handling: it does not establish that the current record has matching keys, a non-sentinel value, or a value corresponding to the incoming candidate state.

### 2. Advance through records

Unless stopped by a newly improved distance of one, traversal advances by reading the 32-bit next pointer at offset 0 of the current record.

The function checks the **loaded next pointer** for null. If null, traversal ends. There is no preceding visible null check of the current record pointer before reading its next field.

For each non-null record reached this way, eligibility is evaluated with short-circuit reads:

1. Read the 32-bit field at offset 8.
2. If it differs from the first key, skip this record without reading its second key or candidate field.
3. Otherwise, read the 32-bit field at offset 24.
4. If it differs from the second key, skip this record without reading its candidate field.
5. Otherwise, read the 16-bit field at offset 12 as signed.
6. If that signed value is −1, skip this record.

Thus the bit pattern `0xFFFF` is a sentinel and is excluded from ordinary candidate processing.

Skipping a record resumes by reading that record’s next pointer. The visible body has no cycle detection or traversal bound.

### 3. Handle an eligible successor

For an eligible record, the 16-bit candidate field is read again, this time as unsigned, and zero-extended to 32 bits.

These are two distinct source-level reads: the sentinel check and the numeric load. The evidence does not establish that they are atomic together or that concurrent mutation is supported.

If this unsigned candidate value equals the full 32-bit requested value:

- The retained distance becomes zero.
- The current record becomes the retained record.
- Traversal ends immediately.

Otherwise, the function establishes comparison flags from the relationship between this candidate value and the requested value, then repeats distance evaluation. For these later candidates, the chosen subtraction direction produces the absolute numerical difference before retaining only its low 16 bits.

For requested values above 65,535, this low-16-bit reduction can make a large numerical difference appear small. It is not a saturating distance and is not necessarily a full-width absolute distance.

### 4. Apply the final selection threshold

After traversal stops, the function uses fallback if either condition holds:

- The retained record pointer is null.
- The requested value is unsigned-less-than the retained distance.

Equality passes the threshold. A non-null retained record with retained distance equal to the requested value is accepted.

The threshold uses the full unsigned 32-bit requested value and the retained unsigned 32-bit distance. The incoming retained distance is not masked merely because later calculated distances are 16-bit-reduced.

An exact eligible successor match always passes this threshold because it establishes a non-null record and distance zero.

### 5. Initialize the fallback record

When fallback is required, the function reads a 32-bit record pointer from context byte offset 8 and makes that pointer the selected record.

It then performs the following writes in the visible order:

1. Write the first key as 32 bits at record offset 8.
2. Write the second key as 32 bits at record offset 24.
3. Write zero as 16 bits at record offset 12.
4. Write zero as 32 bits at record offset 28.
5. Write zero as 16 bits at record offset 32.
6. Write the second key as 32 bits at record offset 36.
7. Write zero as 16 bits at record offset 40.

After those writes, it reads a signed 16-bit field from first-key address plus 2. If that field equals `0x482B`, it writes the 16-bit value one at selected-record offset 34. Otherwise, it writes zero there.

The ordering matters if these memory regions alias. In particular, the first-key-address read occurs **after** the seven preceding fallback writes, so it may observe memory changed by those writes.

There is no visible null check of the fallback pointer and no allocation in the supplied body.

### 6. Invoke the dependency and return

On every normally completing path, the function calls F-d4c4416ace75 with:

1. The context address.
2. The selected record pointer.

It then returns the selected pointer. If the dependency does not return normally, the visible body establishes no alternative completion behavior.

## Side effects

### Memory access and mutation

Traversal reads next pointers, equality keys, and candidate fields. It performs no visible record writes unless fallback is selected.

Fallback mutates only the listed fields before invoking the dependency. Stores of 16 bits affect only their addressed two-byte fields; they do not establish that neighboring bytes are cleared.

The context field at offset 8 is read only on fallback. The first-key address plus 2 is also read only on fallback.

F-d4c4416ace75 is invoked even when the function reuses an existing retained selection without fallback.

### Ownership and lifetime

No ownership transfer, allocation, release, reference-count adjustment, or lifetime extension is established by the visible body.

The fallback record is obtained from the context, not visibly created. It may be shared or preexisting. The dependency’s ownership and lifetime effects are unknown; callers must not infer that the selected pointer remains valid after the call without an additional contract.

### Preconditions and failure behavior

For defined completion, every memory location actually accessed must be valid for the width and access type used. This includes:

- Each current record whose next pointer is read.
- Each reached record field that passes the preceding eligibility checks.
- The context pointer field and fallback record fields when fallback occurs.
- The first-key-address field when fallback occurs.

The initial current record must be readable if execution reaches traversal, even when its incoming candidate does not improve the retained distance.

No explicit error result, recovery path, bounds check, or memory-fault handling is visible. Invalid addresses may cause platform-dependent failure rather than a returned error.

A cyclic chain without an earlier terminating condition may not terminate.

## Dependencies

### F-d4c4416ace75

Established interface at this call site:

- Receives the context address and selected record pointer, in that order.
- Is called once on each normally completing execution.
- Runs after all visible selection and fallback-initialization work.
- Has no return value used by F-0edb66a9a85d.

Its effects on records, context, globals, ownership, memory validity, and failure propagation are **documented_only**. No preservation guarantees are established.

No other called dependency is visible.

## Acceptance examples

These examples are conditional predictions from static evidence, not runtime observations. Unless stated otherwise, memory is valid, relevant reads are stable, and F-d4c4416ace75 returns normally.

### Exact eligible successor

A successor has both matching keys and candidate value 27; the requested value is 27.

That record becomes the selection with retained distance zero. Traversal stops before reading its next pointer. Fallback is not used. The dependency receives that record.

### Distance-one early termination

The retained distance is 10. An eligible successor has candidate value 26 and the requested value is 27.

Its distance of one strictly improves the retained distance, so traversal stops. A later record with candidate value 27 is not examined.

### Equal distances preserve the earlier selection

The retained record is non-null with retained distance 3. The requested value is 20. An eligible successor has candidate value 23.

The calculated distance is 3, so the retained selection is unchanged. If traversal then ends, fallback is not required because 20 is not less than 3.

### Sentinel rejection

A reached record has both matching keys and candidate bits `0xFFFF`.

It is skipped at the signed sentinel check, including when the requested value is 65,535. It cannot become an exact successor match through this path.

### Short-circuit key rejection

A reached record’s first key differs from the requested first key.

The second-key field and candidate field are not read for eligibility. Its next pointer is still read to continue traversal.

### Threshold equality

Traversal ends with a non-null retained record, retained distance 5, and requested value 5.

Fallback is not used. Changing the requested value to 4 while holding that final retained state fixed makes fallback required.

### Full-width request with reduced distance

The requested value is 65,537, the retained distance is 10, and an eligible successor has candidate value zero.

The full numerical difference is 65,537, but its low 16 bits equal one. The record therefore becomes the retained selection and triggers early termination. It is not an exact match.

### Initial zero distance does not itself terminate

The incoming candidate and flags produce reduced distance zero, which improves the retained distance.

The initial current record becomes retained, but execution continues to its next pointer because only an improved distance of one triggers the distance-loop shortcut. Exact-match termination is a separate check performed on eligible successors.

### Fallback field population

Fallback is required, the context contains a valid record pointer, and the 16-bit value read at first-key address plus 2 equals `0x482B`.

The specified fallback fields receive their listed values, and the 16-bit field at record offset 34 becomes one. If that final comparison does not match, the field becomes zero instead. The dependency is then called with the fallback record.

## Uncertainties

### Established static behavior

The supplied body establishes field widths and offsets, short-circuit eligibility checks, distance masking, strict-improvement replacement, early exits, fallback writes, and the dependency call.

### Inferred intent

The structure suggests choosing a nearby compatible record and preparing a context-supplied replacement when necessary. Field meanings beyond those operations, including why distance one is sufficient, are not established.

The magic-value comparison appears to classify the first-key object, but no domain meaning is assigned here.

### Documented-only unknowns

- How the incoming current record, retained selection, retained distance, candidate value, and flags are established.
- Whether entry is intended only after earlier execution has validated the initial candidate.
- Whether callers restrict requested values to 16 bits.
- Whether record chains are acyclic or contain a guaranteed eligible record.
- Alignment requirements, byte order, and memory-mapped access semantics.
- Whether aliasing between context, records, and the first-key object is permitted.
- Whether concurrent mutation is possible or supported.
- F-d4c4416ace75’s state, ownership, lifetime, and failure effects.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. The evidence supports a 32-bit pointer/integer model with explicitly observed 16-bit field operations.

No runtime tests have been performed. Acceptance examples are derived consequences under their stated preconditions, not observed executions. No external dependency implementation or complete entry-state convention was supplied.
