<!-- Trusted provenance: F-69facf68ac4d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-69facf68ac4d

## Purpose

F-69facf68ac4d searches a dependency-supplied sequence of record addresses and returns the first record that satisfies a key comparison and a qualifying status observation. It returns zero if the sequence ends before such a record is found.

The established selection rule is narrow: a record must have the requested 32-bit key at byte offset 8, and its 32-bit field at byte offset 12 must be observed as either −1 or zero under the ordered checks described below. A matching key with any other stable status does not qualify.

The sequence’s organization is not established. In particular, the available evidence does not show a linked list, an array, a tree, or a container-owned iterator. Describing the operation as a search expresses its observed control flow, not a guarantee about the underlying data structure.

There is no effective fallback selection. If no qualifying record is returned immediately, exhaustion produces zero, even if earlier records had matching keys.

## Interface

### Inputs

The function accepts two values:

- **Sequence argument:** an opaque 32-bit value passed unchanged to F-63a239e63e43. This function does not directly interpret or dereference it. Its meaning, validity rules, and ownership are dependency-defined.
- **Requested key:** a signed 32-bit integer used for equality comparison with the record field at byte offset 8. All 32-bit key patterns are representable; the function applies no range restriction or special treatment to zero or negative keys.

The visible interface does not accept a sequence length, a separate iterator object, an output buffer, or an error-output parameter.

### Result

The result is a 32-bit integer carrying one of two meanings:

- **Zero:** the initial or subsequent sequence dependency returned zero before a qualifying record was selected.
- **Nonzero:** the address value supplied for the first qualifying record.

The result is not a Boolean and not a key or status value. Its nonzero case represents a record address, although the visible return type is an integer rather than a separately typed pointer.

Consequently, a nonzero result need not be positive under signed interpretation. The caller must not treat every negative result as an error: the function contains no such error convention. Zero alone is the visible no-result sentinel.

### Record layout accessed by this function

For each nonzero record address, the function uses the following fields:

| Byte offset from record address | Width | Interpretation | Access condition |
|---|---:|---|---|
| 8 | 32 bits | Signed integer key | Every visited nonzero record |
| 12 | 32 bits | Signed integer status | Only when the key equals the requested key |

The fields occupy byte ranges 8–11 and 12–15 respectively. No meaning is established for bytes 0–7, any trailing bytes, or the record’s total allocation size.

The status sentinel −1 is the 32-bit all-ones value. Zero is the 32-bit all-zero value. Comparisons are equality tests; there is no signed ordering comparison.

### Preconditions for defined completion

The dependencies must be callable under the applicable calling convention. Every nonzero address they supply must permit the accesses required for that record:

- The 32-bit key access at offset 8 must be valid.
- If that key matches, the required 32-bit status accesses at offset 12 must be valid.
- Address formation and alignment must satisfy the execution environment’s requirements.
- The record must remain accessible while those accesses occur.

The function performs no visible validation of these conditions. The evidence does not establish any safe recovery behavior for invalid addresses, unreadable memory, or dependency failures.

## Behavior

### Sequence initialization

The function calls F-63a239e63e43 once, passing the sequence argument unchanged. The dependency’s 32-bit result becomes the first candidate record address.

No record access precedes this call. If its result is zero, the function returns zero without calling F-bbc912f1c01e and without reading either record field.

The function has no visible special handling for a zero sequence argument. Whether that argument is valid is a property of F-63a239e63e43, not a condition checked here.

### Candidate inspection

For each nonzero candidate address, the function first reads the 32-bit key at offset 8 and compares it with the requested key.

If the key does not match, the candidate is skipped. Its status field is not read by this function.

If the key matches, status evaluation occurs in this order:

1. The function reads the 32-bit status at offset 12 and compares that observation with −1.
2. If that observation equals −1, the function immediately returns the candidate address.
3. Otherwise, the function reads the status at offset 12 again and compares this second observation with zero.
4. If the second observation equals zero, the function immediately returns the candidate address.
5. Otherwise, the candidate is skipped.

This ordering does not establish a preference for all −1 records over all zero-status records. Selection is first qualifying record in the supplied sequence. A zero-status record encountered earlier is returned without looking for a later −1 record.

The represented behavior includes two possible status observations rather than a guaranteed single snapshot. With a stable status field, a matching record qualifies exactly when its status is −1 or zero. If the field changes between observations, qualification follows the separate checks:

- A first observation of −1 qualifies immediately, regardless of a later value.
- A first observation other than −1 followed by a second observation of zero qualifies.
- A first observation of zero followed by a second observation other than zero does not qualify merely because the earlier observation was zero.
- A second observation of −1 does not qualify unless it is also zero, which a 32-bit value cannot be.

These distinctions describe the access order represented by the supplied material. They do not establish a concurrency model, atomicity guarantee, or permission to mutate the field concurrently.

### Advancement and exhaustion

After skipping a candidate, the function calls F-bbc912f1c01e with no explicit arguments. Its result becomes the next candidate address.

A zero result ends the search and produces zero. A nonzero result is inspected using the same rules as the previous candidate.

The current record address is not explicitly passed to F-bbc912f1c01e. Any connection between successive results therefore depends on state or behavior outside this function’s visible argument passing. The contract does not assume that the dependency preserves, advances, or owns any particular global iterator representation.

There is no visible iteration bound, duplicate-address check, or cycle detection. If the dependency supplies an unending sequence of nonqualifying nonzero records and all operations continue normally, this function does not terminate. Returning the same nonqualifying address repeatedly is sufficient to prevent completion.

### Absence of fallback behavior

The supplied control flow includes a conditional path that could appear to retain a matching candidate. Its enabling condition depends on a local control value that is initialized to a nonzero value and never changed. That path is therefore unreachable in the established flow.

As a result:

- A matching key with a stable status other than −1 or zero is not retained for later return.
- Encountering one or more such records does not alter the exhaustion result.
- Returning zero after exhaustion is not evidence that no key matched; it means no candidate satisfied the effective qualification checks.

### Arithmetic and representation

Field addresses use fixed byte offsets of 8 and 12. The function performs no visible key arithmetic, status arithmetic, record-count arithmetic, or length calculation.

The evidence uses 32-bit address-valued results and 32-bit integer fields. It does not provide a portable rule for address overflow or misaligned access. Valid field address formation is a precondition, not a failure case that the function detects.

## Side effects

The function performs no explicit writes to record memory and does not visibly modify the key or status of a candidate. It constructs no replacement record and copies no record contents to an output object.

Its directly represented memory activity consists of the conditional field reads described above. Dependency calls may have additional effects.

In particular, the evidence does **not** establish that either dependency:

- Preserves global state.
- Leaves record contents unchanged.
- Avoids allocation or deallocation.
- Preserves addresses returned earlier.
- Provides isolated state for nested or concurrent searches.

The function makes no visible allocation, release, retain, reference-count adjustment, or ownership-transfer call. A successful result is the same address value supplied by a dependency, not a newly created object. The caller’s right to retain, mutate, or release that address remains unspecified.

On immediate success, no further advancement call occurs. On exhaustion, no explicit finalization or cleanup call occurs. Any resource or iterator lifecycle requirements must therefore come from the dependencies’ contracts or surrounding usage.

## Dependencies

| Opaque function ID | Visible invocation | Result consumed here | Limits of established knowledge |
|---|---|---|---|
| F-63a239e63e43 | Once, with the opaque 32-bit sequence argument | Initial record address, or zero for termination | Input meaning, side effects, ownership, and sequence setup are unknown |
| F-bbc912f1c01e | With no explicit arguments after each rejected candidate | Next record address, or zero for termination | Progression mechanism, state location, ordering, and side effects are unknown |

For a normally terminating search that succeeds on the first candidate, the visible dependency pattern is one initialization call and no advancement calls.

For a normally terminating search that rejects several candidates before succeeding, each rejected candidate is followed by one advancement call. The selected candidate is not followed by an advancement call.

For exhaustion after a nonempty sequence, the last rejected candidate is followed by an advancement call returning zero.

Neither dependency’s zero result is interpreted as a distinct error. It is treated as sequence termination regardless of the dependency’s internal reason for returning it.

## Acceptance examples

These examples are conditional contracts, not reports of executed tests. Unless otherwise stated, all addresses and field accesses are valid, field values remain stable during inspection, and dependencies return normally.

1. **Empty sequence.**  
   F-63a239e63e43 returns zero. The result is zero. No candidate fields are read, and F-bbc912f1c01e is not called.

2. **First candidate has a matching key and status −1.**  
   The candidate address is returned after the first status observation. There is no second status observation and no advancement call.

3. **First candidate has a matching key and status zero.**  
   The first status comparison does not match −1. The second status observation matches zero, and the candidate address is returned.

4. **Nonmatching key with an otherwise qualifying status.**  
   A candidate’s key differs from the requested key. It is skipped without reading its status. A status of −1 or zero cannot compensate for the key mismatch.

5. **Matching key with a nonqualifying status, then exhaustion.**  
   A candidate has the requested key and a stable status of 7. Both status checks fail. The advancement dependency returns zero. The function returns zero rather than the rejected candidate.

6. **Earlier zero-status match, later −1-status match.**  
   The earlier candidate is returned. The later candidate is not requested or inspected by this function.

7. **Negative requested key.**  
   The requested key is −23, and a candidate’s 32-bit key equals −23. It proceeds to status evaluation exactly as a positive-key match would.

8. **Status changes between represented observations.**  
   For a matching candidate, the first status observation is 5 and the second is zero. The candidate is returned. If those observations are instead zero and 5, it is skipped. This example is conditional on the execution environment permitting the changing observations.

9. **Repeated nonqualifying candidate.**  
   The advancement dependency continually supplies the same valid address with a nonmatching key. The function continues calling the dependency; it has no visible repeated-address termination rule.

## Uncertainties

The following matters are **documented_only** because the supplied evidence does not resolve them:

- The semantic meaning of the sequence argument and the record fields.
- The data structure, traversal order, and state used by the dependencies.
- Dependency error reporting beyond the zero value consumed as termination.
- Record ownership, lifetime after return, and release obligations.
- Thread safety, reentrancy, synchronization, and mutation rules.
- Endianness, alignment behavior, and invalid-memory consequences.
- Whether the represented repeated status accesses correspond to distinct observable machine reads in the execution environment.
- Any guarantees that a sequence is finite or advances without repetition.

The likely intent is to locate a record by key while accepting two distinguished status values. That interpretation is inferred; the precise field reads, comparisons, early returns, and dependency invocation order are the established behavior.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping.

Established evidence supports the two 32-bit inputs, address-valued 32-bit result, field offsets and widths, ordered equality checks, conditional dependency calls, immediate success returns, zero termination handling, and unreachable fallback-selection path.

No dependency implementations, record type declarations, caller contracts, or runtime observations were supplied. No runtime tests have been performed. The acceptance examples are consequences of the stated static behavior under explicit assumptions, not empirical validation.
