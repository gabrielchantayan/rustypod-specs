<!-- Trusted provenance: F-af55eff9cfe2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-af55eff9cfe2

## Purpose

F-af55eff9cfe2 updates an object’s stored cursor to the next position whose inspected 32-bit field is not the all-ones value. The search proceeds toward increasing indices through index 17, inclusive.

If the search reaches the upper limit without finding an eligible position, the function stores the all-ones cursor value, 0xFFFFFFFF. That value also has a special effect when supplied as the initial cursor: incrementing it with 32-bit unsigned arithmetic produces zero, allowing a subsequent invocation to begin its search at index 0.

The established behavior is a bounded forward search followed by one cursor write. Interpreting the inspected fields as availability markers, or the cursor as a selection, is inferred intent rather than a demonstrated application-level meaning.

## Interface

**Input:** One address-like argument identifying the base of a mutable object.

**Output:** No return value. The result is communicated by overwriting a 32-bit field at byte offset 0x28C from the object base.

The relevant layout is:

| Object-relative location | Width | Role |
|---|---:|---|
| 0x28C | 32 bits | Stored cursor, interpreted as unsigned |
| 0x24 + 0x24 × index | 32 bits | Inspected field for a candidate index |
| Candidate index range | 0 through 17 inclusive | Positions the function may inspect |

The candidate fields are spaced 0x24 bytes, or 36 bytes, apart. Only the four-byte field at each candidate location is relevant to this function. The contents and meanings of the intervening bytes are not established.

The first candidate field is at offset 0x24. The last is at offset 0x288, occupying bytes through 0x28B. The cursor begins immediately afterward at offset 0x28C; it does not overlap any inspected candidate field.

The contract uses 32-bit unsigned arithmetic for the cursor and candidate index. The supplied representation identifies these operations as unsigned integer operations and uses a 32-bit all-ones sentinel. Exact target ABI details are not independently available.

**Memory preconditions:**

- The input must identify memory for which the cursor read and cursor write are valid.
- Each candidate field actually reached by the search must be readable as a 32-bit integer.
- Accesses must satisfy the target platform’s alignment and memory-access requirements.
- Object-relative address calculations must resolve to the intended storage.

A conventional contiguous object containing the entire relevant layout needs accessible storage through offset 0x28F. A particular invocation may touch much less of that storage, because some initial cursor values cause no candidate reads.

There is no demonstrated null check, allocation-size check, alignment check, or pointer validation.

## Behavior

### Cursor interpretation and search bounds

The function first reads the stored cursor once. That value determines the first candidate:

- Initial cursor values 0 through 16 begin the search at the following index.
- Initial cursor value 17 produces the failure sentinel without inspecting any candidate.
- Initial cursor values 18 through 0xFFFFFFFE also produce the failure sentinel without inspecting any candidate.
- Initial cursor value 0xFFFFFFFF begins the search at index 0 because unsigned increment wraps to zero.

Consequently, the currently stored position is not reconsidered when its value is within 0 through 17. The search is strictly after that position.

The sentinel is the only initial value that starts a search at index 0. Arbitrary out-of-range values are not reduced modulo 18, clamped into the candidate range, or treated as equivalent to the sentinel.

### Eligibility

A candidate is rejected exactly when its inspected 32-bit field compares equal to signed −1. In the specified 32-bit model, that is the bit pattern 0xFFFFFFFF.

Every other field value is eligible. In particular:

- Zero is eligible.
- Positive signed values are eligible.
- Negative signed values other than −1 are eligible.
- No demonstrated validity rule examines other fields in the candidate’s 36-byte region.

The first eligible candidate encountered in increasing index order becomes the stored cursor. No later candidates are inspected.

### Exhaustion and restart behavior

If all remaining candidates are rejected, the stored cursor becomes 0xFFFFFFFF.

The function does not wrap from index 17 to index 0 during the same invocation. Instead, exhaustion stores the sentinel and ends the invocation. If a later invocation observes that sentinel, that later invocation starts at index 0.

This distinction is observable. With a starting cursor of 16, a rejected candidate at index 17 causes the sentinel to be stored even if index 0 is eligible.

### Observable access order

For an invocation that completes normally, the access order is:

1. One 32-bit read of the cursor at offset 0x28C.
2. Zero or more 32-bit candidate-field reads in strictly increasing index order.
3. One 32-bit write to the cursor at offset 0x28C.

Each candidate is read at most once during an invocation. At most 18 candidate fields are read: this maximum occurs when the initial cursor is 0xFFFFFFFF and all indices 0 through 17 must be examined.

The upper-bound check occurs before reading a candidate field. The function therefore does not inspect index 18, nor does it use an arbitrary out-of-range initial cursor to address a candidate field.

No intermediate cursor values are written while the search proceeds. The final cursor write occurs even when it writes the same bit pattern that was initially present, such as an initial sentinel followed by complete exhaustion.

### Stability assumptions

For a fixed initial cursor and stable candidate fields, the result is deterministic.

The function does not reread the cursor during its search. A change to that field after its initial read does not alter the search’s already established starting point; the final store may overwrite such a change.

No synchronization is shown. If another actor changes candidate fields while they are being examined, the function’s decisions depend on the values observed by the individual reads. The supplied evidence does not establish atomicity, a coherent snapshot, or valid concurrent use under any particular language or platform memory model.

## Side effects

The sole established application-memory write is the final 32-bit cursor store at offset 0x28C.

The function does not modify candidate fields or other object bytes in the supplied behavior. It performs no demonstrated allocation, deallocation, ownership transfer, external call, logging, or error reporting.

The object remains caller-provided storage. This function neither acquires ownership of it nor releases it.

There is no explicit error return or recovery path. Invalid or inaccessible memory can prevent normal completion, but the precise fault behavior is platform-dependent and is not established here. In particular, the normal-completion guarantee of one final cursor write does not apply if an earlier access faults.

The access counts above describe accesses represented by the supplied source, not a guarantee about processor instructions, bus transactions, compiler transformations, or volatile-memory semantics.

## Dependencies

No callable dependencies appear in the supplied function. There are therefore no additional opaque function IDs to enumerate.

The behavior depends on:

- The object-relative memory layout described above.
- A 32-bit unsigned cursor arithmetic model.
- Recognition of the candidate-field all-ones value as signed −1.
- Valid memory access under the target execution environment.

No unknown callee behavior is needed to explain the result. In particular, this contract does not rely on another function preserving global state or managing memory ownership.

## Acceptance examples

These are conditional acceptance cases derived from static evidence. They have not been executed. Unless stated otherwise, assume valid memory, 32-bit unsigned arithmetic, and stable candidate fields during the invocation.

| Initial cursor and relevant fields | Required cursor result | Required candidate reads |
|---|---|---|
| Cursor is 4; field at index 5 is zero | 5 | Index 5 only |
| Cursor is 4; fields at indices 5 and 6 are 0xFFFFFFFF; field at index 7 is 12 | 7 | Indices 5, 6, and 7, in that order |
| Cursor is 16; field at index 17 is 0xFFFFFFFE | 17 | Index 17 only; this negative signed value is not the rejection marker |
| Cursor is 16; field at index 17 is 0xFFFFFFFF; field at index 0 is zero | 0xFFFFFFFF | Index 17 only; no same-invocation wrap |
| Cursor is 17; candidate fields have arbitrary values | 0xFFFFFFFF | None |
| Cursor is 18; candidate fields have arbitrary values | 0xFFFFFFFF | None |
| Cursor is 0xFFFFFFFE; candidate fields have arbitrary values | 0xFFFFFFFF | None |
| Cursor is 0xFFFFFFFF; field at index 0 is zero | 0 | Index 0 only |
| Cursor is 0xFFFFFFFF; indices 0 through 16 contain 0xFFFFFFFF; index 17 contains 1 | 17 | All 18 candidate fields, in increasing order |
| Cursor is 0xFFFFFFFF; every candidate field contains 0xFFFFFFFF | 0xFFFFFFFF | All 18 candidate fields, in increasing order |

Additional acceptance conditions:

- With initial cursor 0, an eligible field at index 0 must not cause index 0 to be selected. The search begins at index 1.
- If an eligible candidate is found, later fields must not be inspected by this function’s search.
- Changing bytes outside the cursor and inspected candidate fields must not affect the source-level result, provided memory validity and execution conditions otherwise remain unchanged.
- After one invocation exhausts the remaining range and stores 0xFFFFFFFF, a subsequent invocation must begin at index 0 if it observes that stored value. This is a two-invocation transition, not circular search within one invocation.

## Uncertainties

**Established by static evidence:**

- Cursor and candidate-field offsets and access widths.
- Candidate stride and inclusive upper bound.
- Increment-before-inspection behavior.
- Exact rejection comparison.
- First-match selection, exhaustion behavior, and final cursor store.
- Absence of calls in the supplied function.

**Inferred intent:**

- The object contains a fixed sequence of 18 logical entries.
- The cursor represents a current selection or iteration position.
- The all-ones candidate field denotes an unavailable or inactive entry.

These interpretations are not required for the mechanical contract.

**documented_only:**

- The complete object type, allocation size, and meanings of other fields.
- The application-level meaning of an eligible candidate.
- The actual target ABI, byte order, and precise address-arithmetic fault or overflow behavior.
- Whether callers guarantee valid cursor values beyond the sentinel convention.
- Whether concurrent access is permitted or externally synchronized.
- Whether any relevant memory is volatile, memory-mapped, or subject to external mutation.
- Whether callers depend on the final store when its value is unchanged.

No unspecified ownership, synchronization, validation, or error-handling guarantees are assumed.

## Evidence

This contract is based solely on the supplied static representation of F-af55eff9cfe2.

The decisive evidence is the initial unsigned cursor load at offset 0x28C, unsigned increment before each bound check, inclusive candidate limit of 17, 36-byte candidate spacing beginning at offset 0x24, comparison of each candidate field against −1, and one final cursor store.

No runtime tests, execution traces, caller analysis, or independent platform verification have been performed. The acceptance examples are derived expectations, not observed results.
