<!-- Trusted provenance: F-3f8cc4fca4c0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3f8cc4fca4c0

## Purpose

F-3f8cc4fca4c0 determines whether a sequence of candidate records can be accepted against a position-based descriptor, with acceptance influenced by a signed threshold and two opaque dependency checks.

The established behavior is a forward scan over descriptor positions. A successful match consumes one candidate; an ordinary mismatch can advance the descriptor position without consuming that candidate. Some mismatches cause immediate failure instead. Accepted matches also establish an eight-bit XOR value whose consistency is enforced after a threshold.

The function returns a 32-bit Boolean result. On paths reaching normal finalization, it separately writes whether the accumulated XOR value is nonzero. Some immediate-failure paths bypass that write.

The interpretation as sequence compatibility or matching is inferred intent. The observable contract is the scan, dependency calls, state changes, and result rules below.

## Interface

### Arguments

The function receives seven arguments:

1. **Context reference** — a pointer to a structure whose first 32-bit word is treated as a pointer to a root object.
2. **Primary selector** — an unsigned 32-bit value. It is forwarded to F-b205ebbb7297.
3. **Descriptor reference** — a 32-bit address identifying a structure containing the position count, comparison arrays, a distinguished value, and a finalization flag.
4. **Secondary selector** — an unsigned 32-bit value compared with a field of every candidate eligible for matching. It is also forwarded to F-b205ebbb7297.
5. **Candidate-state reference** — a pointer to a structure containing a signed candidate count and a pointer to candidate records.
6. **Threshold** — supplied as an unsigned 32-bit value, but interpreted as a signed 32-bit value for threshold comparisons.
7. **Output reference** — a pointer to a writable 32-bit unsigned word.

The source reflects a 32-bit address and word model. Porting the behavior to another address width requires preserving the stated field offsets, integer widths, and signed comparisons rather than adopting native-width arithmetic implicitly.

### Return value

The returned 32-bit word is:

- **1** when normal finalization accepts the result.
- **0** when an immediate rejection occurs or normal finalization rejects residual candidates.

This is not a count or a dependency status forwarded unchanged.

### Referenced layouts

Only the following portions of the referenced objects are established.

| Object | Byte offset | Interpretation |
|---|---:|---|
| Context | 0 | 32-bit root-object pointer, cached on entry |
| Root object | 44 | 32-bit fallback-object pointer |
| Selected or fallback object | 0 | 32-bit comparison value |
| Candidate state | 0 | Signed 32-bit candidate count, cached on entry |
| Candidate state | 12 | Pointer to candidate-record storage, cached on entry |
| Candidate record | 0 | 32-bit pointer to a candidate object |
| Candidate record | 8 | One-byte attribute |
| Candidate object | 0 | One-byte type marker |
| Candidate object | 36 | Unsigned 32-bit selector |
| Candidate object | 40 | Signed 32-bit identifier |
| Descriptor | 4 | Signed 32-bit position count |
| Descriptor | 8 | Pointer to an array of 32-bit identifiers |
| Descriptor | 16 | Pointer to an object containing the distinguished identifier |
| Distinguished-identifier object | 12 | Signed 32-bit distinguished identifier |
| Descriptor | 24 | One-byte finalization flag; zero and nonzero are distinguished |
| Descriptor | 40 | Pointer to an array of one-byte attributes |
| Descriptor | 44 | Pointer to an array of 32-bit comparison values |

Candidate records have a 12-byte stride. Bytes other than the identified fields are not interpreted directly by this function.

### Preconditions

The caller must provide readable objects and arrays for every access the selected execution path reaches, and a writable output word if normal finalization is reached. No null-pointer, extent, alignment, or object-lifetime validation is visible.

Pointers returned by dependencies, and the fallback pointer when used, must remain valid through subsequent accesses. No ownership transfer is established.

## Behavior

### Entry state and scan bounds

The function initializes the accumulated XOR value to zero and the consumed-candidate count to zero. It caches, in order, the context’s root pointer, the candidate-record base pointer, and the signed candidate count.

Scanning begins at descriptor position zero. An iteration occurs only while both conditions hold:

- The consumed-candidate count is less than the cached candidate count.
- The current descriptor position is less than or equal to the descriptor’s current signed position count.

The first condition is evaluated before reading the position count for the second condition.

The inclusive upper bound permits a special terminal position equal to the descriptor count. That terminal position does not read entries from the descriptor’s position arrays.

The candidate count and record base used by the scan are entry snapshots. The descriptor count is not: it is read again at loop conditions, when choosing between an ordinary and terminal position, and potentially during finalization. Dependency-induced mutation can therefore affect later decisions.

### Candidate eligibility gate

Each iteration reads the candidate-object pointer from the current candidate record, then reads the object’s type byte.

Matching proceeds only if:

- The type byte is exactly `0x95`.
- The unsigned 32-bit selector at candidate-object offset 36 equals the secondary selector.

If the type is different, the selector field is not needed for this gate. Failure of either condition exits the scan without consuming the candidate or incrementing the descriptor position. It does not directly return failure: normal finalization still decides whether the remaining candidates are acceptable.

The primary selector does not participate in this eligibility comparison.

### Selecting the comparison object

For an eligible candidate, the function calls F-4290fdeada83 with the context reference and candidate-object pointer.

- If the returned pointer is nonzero, it is the selected comparison object.
- If the returned pointer is zero, the function reads the fallback-object pointer at offset 44 of the root object cached on entry.

The fallback is read after the dependency returns. Consequently, the root address is cached, but the fallback pointer’s contents are not an entry snapshot.

The function does not validate either selected pointer.

### Obtaining expected values

After selection, the function rereads the descriptor position count.

If the current position is strictly less than that count:

1. It reads the signed identifier from the descriptor’s identifier array at that position.
2. It reads the distinguished identifier through the pointer at descriptor offset 16.
3. If the two identifiers are equal, the expected identifier becomes **−1**; otherwise it remains the array identifier.
4. It reads the one-byte expected attribute from the descriptor’s attribute array.
5. It reads the 32-bit expected comparison value from the descriptor’s comparison-value array.

If the current position is not strictly less than the count:

- The expected identifier is **−1**.
- The expected attribute is zero.
- The expected comparison value is read from offset zero of the selected comparison object.

This second branch normally represents the inclusive terminal position. Because the count can change during dependency execution, it also applies whenever the subsequent count read no longer places the current position below the count.

### Matching and mismatch behavior

The function first compares the candidate object’s signed identifier at offset 40 with the expected identifier.

Only if those identifiers agree does it call F-e6db2171781a. Its arguments are:

1. The current 32-bit value at offset zero of the selected comparison object.
2. The expected comparison value obtained above.

A zero dependency result means this comparison passes. Any nonzero result means mismatch.

For the terminal branch, the expected value is read earlier from the selected object, while the first comparison argument is read later. The contract does not replace these separate reads with an assumed immutable value.

On identifier or comparison mismatch:

- If the signed threshold is less than or equal to the current descriptor position, the function immediately returns **0**, without its final output write.
- Otherwise, it leaves the candidate unconsumed, advances the descriptor position by one, and tries that same candidate against the next position.

Thus positions strictly before the threshold can be skipped after an ordinary mismatch. The threshold position itself cannot.

### Attribute consistency and candidate consumption

After both comparisons pass, the function reads the current candidate record’s attribute byte and XORs it with the expected attribute byte. Both operands are zero-extended, so the result is in the range 0–255.

If the current descriptor position is strictly greater than the signed threshold, this XOR result must equal the previously accumulated XOR value. A difference immediately returns **0**, bypassing the final output write.

At positions less than or equal to the threshold, no such consistency check is imposed.

For an accepted match, the function:

1. Replaces the accumulated value with the newly computed XOR result.
2. Increments the consumed-candidate count.
3. Advances the candidate-record pointer by 12 bytes.

No candidate record or candidate object is directly modified.

### Negative-identifier dependency

After consuming a candidate, if the expected identifier is negative, the function calls F-b205ebbb7297 with:

- The candidate-state reference.
- The primary selector.
- The consumed-candidate count after increment.
- The secondary selector.

This applies to **every negative expected identifier**, not only −1.

The call produces a 64-bit result, of which the low 32 bits govern this decision:

- Low word zero: the scan’s consumed-candidate count is set to the entry-snapshot candidate count, causing the next loop condition to regard all candidates as consumed.
- Low word nonzero: scanning can continue.

The high word is extracted, but no success, failure, or output decision is shown to depend on it.

After a normally completed iteration, including this dependency call when applicable, the descriptor position increments by one.

### Normal finalization

Every path that exits the scan without immediate rejection writes the output word first:

- **0** if the accumulated XOR value is zero.
- **1** otherwise.

The function then checks whether the consumed-candidate count is less than the entry-snapshot candidate count.

If not, it returns **1**, without reading the finalization flag or making a residual-candidate call.

If candidates remain, it evaluates these conditions in order:

1. The descriptor’s finalization flag must be nonzero.
2. The descriptor’s current position count must equal the current scan position.
3. F-b205ebbb7297, called with the same argument roles described above and the current consumed count, must return zero in its low 32 bits.

If any condition fails, the result is **0**. If all pass, the result is **1**.

The flag is read before the position-count equality check. The residual-candidate dependency is called only after both checks succeed.

The output write precedes that dependency call. Its final value on return therefore cannot be guaranteed if the dependency mutates overlapping memory.

## Side effects

The only direct caller-memory write established in the supplied source is the 32-bit Boolean write through the output reference during normal finalization.

Immediate rejection due to a threshold-controlled mismatch or inconsistent XOR bypasses that write. This guarantees absence of the function’s own final write, not absence of writes by dependencies already called.

The function directly maintains only local scan state. It advances a local candidate-record pointer; it does not visibly store that pointer or the consumed count into candidate state.

Dependencies can introduce additional effects. Their behavior is not supplied, so this contract does not claim they:

- Preserve globals or argument-reachable memory.
- Avoid allocation or release.
- Preserve the candidate-state count, descriptor arrays, or selected object.
- Leave the output location unchanged through aliases.
- Maintain returned-object lifetime.

The source shows sequential accesses, not an atomic snapshot or synchronization protocol.

## Dependencies

| Opaque function ID | Invocation | Established use |
|---|---|---|
| F-4290fdeada83 | Context reference, candidate-object pointer | Returns the selected-object pointer; zero selects the root’s fallback |
| F-e6db2171781a | Selected object’s first word, expected comparison word | Zero is a match; nonzero is a mismatch |
| F-b205ebbb7297 | Candidate-state reference, primary selector, consumed count, secondary selector | Low 32-bit zero signals acceptance/completion in the applicable branch |

The dependency names, argument roles, and return-value use do not establish their internal semantics. In particular, equality, suffix validation, lookup, and ownership behavior remain undocumented beyond the caller’s observed use.

## Acceptance examples

These examples are conditional static consequences, not executed tests. Unless stated otherwise, all accessed memory is valid, dependencies preserve relevant memory, and arithmetic does not overflow.

1. **No candidates.**  
   The cached candidate count is zero. The loop does not run. The function writes output **0** and returns **1**. The descriptor position count need not be read by the loop guard.

2. **Negative cached candidate count.**  
   The initial consumed count, zero, is not less than a negative count. The scan is skipped, output becomes **0**, and the function returns **1**. No explicit malformed-count rejection exists.

3. **Skip before the threshold.**  
   At position zero with threshold one, an identifier mismatch leaves the candidate unconsumed and advances to position one. If the candidate matches there and exhausts the candidate count, the function returns **1**.

4. **Mismatch at the threshold.**  
   At position one with threshold one, an identifier or comparison mismatch immediately returns **0**. The final output write does not occur.

5. **Inconsistent attribute difference after the threshold.**  
   A previously accepted match establishes XOR value 3. A later otherwise matching position strictly beyond the threshold produces XOR value 5. The function immediately returns **0**, without its final output write.

6. **Successful comparison at the threshold.**  
   A match exactly at the threshold may replace the accumulated XOR value with a different value. Consistency is enforced only at strictly greater positions.

7. **Negative interpretation of the threshold.**  
   Threshold bit pattern `0xFFFFFFFF` is interpreted as −1. At position zero, an ordinary mismatch rejects immediately. A successful comparison also requires XOR zero, because position zero is greater than −1 and the initial accumulator is zero.

8. **Eligibility failure can still accept.**  
   An ineligible candidate stops scanning. If residual candidates remain, the finalization flag is nonzero, the current descriptor count equals the stopped position, and the final F-b205ebbb7297 call returns low-word zero, the function returns **1**.

9. **Final rejection still writes the output.**  
   The scan exits normally with candidates remaining and accumulated XOR 7. If the finalization flag is zero, output is written as **1**, then the function returns **0**.

10. **Negative expected identifier completes early.**  
    An accepted match has a negative expected identifier. If F-b205ebbb7297 returns low-word zero, the scan treats all cached candidates as consumed. Normal finalization returns **1**, irrespective of whether every record was individually visited.

## Uncertainties

The following are **documented_only**:

- Business meaning of the marker, selectors, identifiers, attributes, and descriptor flag.
- Full dependency contracts, including mutation, global effects, ownership, allocation, and release.
- Required object alignment and behavior for invalid or inaccessible pointers.
- Intended behavior if scan indices or signed increments overflow. The source uses 32-bit quantities; no overflow guard is visible.
- Intended handling of negative descriptor counts beyond the established signed comparisons and finalization rules.
- Whether mutable or aliased inputs are supported intentionally.
- Whether the reconstructed signedness and pointer interpretation exactly match the original machine-level interface.

No global-preservation guarantee, memory-management guarantee, or concurrency guarantee is inferred.

## Evidence

Established static evidence consists solely of the supplied function body and opaque dependency mapping. It supports the layouts, access order, branch conditions, signed threshold interpretation, 12-byte candidate stride, dependency arguments, return decisions, and distinction between normal-finalization writes and immediate-rejection paths.

The description of the function as a sequence-matching operation is inferred intent, not an additional requirement.

No implementation access, dependency-body analysis, runtime execution, or runtime tests have been performed. The acceptance examples are conditional deductions from static control flow.
