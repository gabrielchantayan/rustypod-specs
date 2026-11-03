<!-- Trusted provenance: F-6cf9b735ce3f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6cf9b735ce3f

## Purpose

F-6cf9b735ce3f searches an indirectly referenced collection for the first entry whose two 32-bit fields match the corresponding fields of a supplied target.

The function first invokes a prerequisite dependency. If that dependency returns zero, or if the target argument is zero, it returns zero without inspecting the collection. Otherwise, it obtains the collection through the first argument, optionally invokes a dependency when the collection reference is absent, and searches the resulting collection.

For an enabled search with a nonzero target, the result is the zero-based index of the first matching entry, or −1 if no match is found or the collection remains absent.

**Important result ambiguity:** zero denotes both a match at index zero and an early exit that performs no search. The return value alone cannot distinguish these outcomes.

The interpretation of the two compared fields as an identity or lookup key is inferred intent. The established behavior is equality comparison of the fields at the specified offsets; their application-level meanings are unknown.

## Interface

### Arguments and result

The interface has two 32-bit arguments and a 32-bit signed integer result.

| Item | Contract |
|---|---|
| First argument | Address-like value identifying an object containing a collection reference at byte offset `0x900`. |
| Second argument | Address-like value identifying the target record, or zero to request the early-exit behavior. |
| Result `0` | Either an early exit or a match at collection index zero. |
| Result greater than `0` | Index of the first matching entry. |
| Result `−1` | An enabled search did not find a match, including the case where the collection reference remains zero. |

Although the supplied representation declares the arguments as integers, both are used as addresses. The memory accesses establish a 32-bit address/reference representation for this contract. Portability to other pointer widths is not established.

### Relevant memory layout

All offsets below are byte offsets. Each accessed field or reference occupies four bytes.

| Referenced object | Offset | Meaning established by use |
|---|---:|---|
| Object identified by the first argument | `0x900` | Collection reference, with zero meaning absent. |
| Collection | `0x08` | Signed 32-bit entry count used in loop continuation. |
| Collection | `0x0C + 4 × index` | Reference to the entry at that index. |
| Entry record | `0x114` | First field compared for equality. |
| Entry record | `0x110` | Second field compared, only if the first comparison succeeds. |
| Target record | `0x114` | Target value for the first comparison. |
| Target record | `0x110` | Target value for the second comparison. |

The collection’s fields before offset `0x08`, and all other record fields, have no established role in this function. The entry references are stored inline in the collection at four-byte spacing; the records themselves are accessed indirectly.

### Preconditions

No valid first-argument object is needed for the function’s own direct accesses when the prerequisite dependency returns zero or the target is zero. The prerequisite dependency still executes in both cases and has its own unknown requirements.

For a search that reaches collection access:

- The first argument must identify readable memory through the four-byte field at offset `0x900`.
- A nonzero collection reference must identify a readable count field.
- Every reached entry slot must be readable and contain a reference suitable for the record accesses actually performed.
- The target must permit a read at offset `0x114` whenever an entry is examined. It must also permit a read at offset `0x110` whenever the first comparison succeeds.
- Address calculations and accesses must be valid in the execution environment.
- The collection count and reachable storage must be consistent enough to permit the search.

There is no direct validation of these conditions. In particular, there is no check for a zero entry reference.

The ordinary bounded-search interpretation additionally requires that subtracting one from the count and calculating slot addresses remain valid. Behavior outside that arithmetic domain is not established.

## Behavior

### 1. Prerequisite invocation and early exit

The function invokes F-06f4ceb6a4f2 before inspecting either the collection or the target record.

If the dependency returns zero, the function returns zero. If it returns nonzero but the target argument is zero, the function also returns zero.

Consequently:

- A zero target does not suppress the prerequisite call.
- The first argument’s collection-reference field is not directly read on either early-exit path.
- No target-record fields are directly read on either early-exit path.
- All nonzero prerequisite results are treated alike; there is no established interpretation of their individual values.

### 2. Collection acquisition

After passing the gate, the function reads the collection reference at offset `0x900` from the first-argument object.

If that reference is nonzero, the collection-acquisition dependency is skipped.

If the reference is zero, the function invokes F-40f9148e1210 with the first argument. Its return value does not determine the next action. Instead, the function reads the collection-reference field again.

If this second read is still zero, the result is −1. If it is nonzero, the search begins.

Thus, acquisition success is determined by the observed collection-reference field after the dependency returns, not by the dependency’s return value. The acquisition dependency is invoked at most once by this function.

### 3. Search bounds and entry access

The search begins at signed index zero.

Before each attempted entry visit, the function reads the collection reference from the first-argument object, reads that collection’s count at offset `0x08`, subtracts one, and continues only if the current index is less than or equal to the resulting signed bound.

For stable memory and a nonnegative count whose arithmetic is valid, this visits indices zero through count minus one in ascending order:

- Count zero produces no entry visits.
- Count one permits only index zero.
- A positive count permits at most that many visits unless the count changes.

The count is not captured once for the entire search. It is read again for each continuation check, including a final check after the last unsuccessful visit.

After a successful continuation check, the function again reads the collection reference from the first-argument object and obtains the entry reference from offset `0x0C + 4 × index` of that collection. The collection reference used for entry access is therefore a separate observation from the reference used to obtain the count.

No dependency is invoked during the entry-search loop.

### 4. Matching and termination

For each visited entry:

1. The four-byte entry field at offset `0x114` is compared with the four-byte target field at the same offset.
2. Only if those values are equal does the function compare the entry and target fields at offset `0x110`.
3. If both comparisons succeed, the function returns the current index immediately.
4. Otherwise, the search advances to the next index.

The comparison at `0x114` is evaluated before the comparison at `0x110`. The latter comparison is short-circuited when the former fails.

The supplied static representation does not establish which operand of an individual equality comparison is physically read first. It does establish that the second pair of fields is not required unless the first equality succeeds.

Target fields are read during entry comparisons rather than snapshotted before the loop. Entry fields likewise are accessed for each visited record.

The search returns the first matching index. It does not inspect later entries after a match and does not require uniqueness.

If the continuation condition fails without a match, the function returns −1.

## Side effects

### Direct effects

The function contains no explicit writes to the first-argument object, collection, entry records, or target record. Its directly visible data operations are reads, dependency calls, and returning a result.

It contains no explicit allocation, deallocation, ownership transfer, reference-count change, or record update. It returns an integer rather than an owned object or borrowed record reference.

These statements concern only operations visible in this function. They do not establish that invoking it is globally read-only.

### Dependency-mediated effects

F-06f4ceb6a4f2 executes on every invocation. Its effects are unknown.

F-40f9148e1210 executes only when the gate passes and the initially observed collection reference is zero. Its effects are also unknown beyond the caller’s subsequent reread of the collection-reference field.

Neither dependency may be assumed to preserve global state, preserve referenced memory, allocate storage, or release storage. The function does not expose enough evidence to establish those properties.

### Changing memory and errors

Repeated reads of the collection reference and count mean that this function does not establish snapshot consistency. If another actor changes the collection during execution, the count check and subsequent entry access can observe different collections or different bounds.

There is no visible synchronization, retry mechanism, or protection against such changes. Stable, valid storage throughout the reached accesses is therefore a precondition for the straightforward search interpretation, not a guarantee provided by the function.

Invalid addresses, insufficient storage, invalid entry references, and arithmetic outside the established domain have no explicit error handling. They are not converted into −1 by any visible mechanism. Fault behavior is environment-dependent and documented only.

## Dependencies

| Opaque ID | Invocation condition | Inputs | Use of result |
|---|---|---|---|
| F-06f4ceb6a4f2 | Every invocation, before collection access | No explicit arguments | Zero causes immediate result zero; nonzero permits checking whether the target is nonzero. |
| F-40f9148e1210 | Gate passes and the first collection-reference read is zero | First argument | Return value is ignored; the collection-reference field is reread to determine whether searching can proceed. |

The first dependency appears to provide an availability or eligibility gate. The second appears to obtain or prepare the collection. Those descriptions are inferred intent, not established dependency contracts.

No ownership or lifetime guarantee follows from either call’s presence.

## Acceptance examples

These are conditional acceptance cases derived from static evidence. They are not reports of runtime execution. Unless stated otherwise, memory is stable, addresses are valid, and arithmetic is within the established domain.

1. **Prerequisite rejects the operation.**  
   F-06f4ceb6a4f2 returns zero. Expected result: zero. No collection-reference or record-field access is performed directly by this function, regardless of the argument values.

2. **Zero target with an affirmative prerequisite.**  
   The prerequisite returns nonzero and the target is zero. Expected result: zero. The prerequisite has executed, but collection acquisition and searching do not occur.

3. **Existing collection bypasses acquisition.**  
   The gate passes and the collection-reference field is already nonzero. F-40f9148e1210 is not invoked. Searching uses the referenced collection.

4. **Acquisition leaves the collection absent.**  
   The initial collection reference is zero, F-40f9148e1210 is invoked, and the subsequent reference read is still zero. Expected result: −1, irrespective of that dependency’s return value.

5. **Acquisition makes a collection available.**  
   The initial reference is zero, but the post-call read is nonzero. The function searches that collection. A dependency result that might conventionally signify failure does not prevent searching if the field is nonzero.

6. **Empty collection.**  
   The gate passes and a valid collection has count zero. Expected result: −1. No entry or target-record fields are read.

7. **First entry matches.**  
   Count is positive, and the entry at index zero matches both target fields. Expected result: zero. Later entries are not inspected. This result is numerically indistinguishable from either early-exit case.

8. **First comparison fails.**  
   A visited entry’s `0x114` field differs from the target’s corresponding field. The `0x110` comparison is skipped for that entry, and the search continues.

9. **Second comparison fails.**  
   A visited entry matches at `0x114` but differs at `0x110`. That entry is not a match. A later entry matching both fields may still be returned.

10. **Duplicate matches.**  
    Entries at indices two and four both match both target fields, while earlier entries do not. Expected result: two. Index four is not examined after the match.

11. **No matching entry.**  
    Every entry permitted by a stable valid count fails at least one comparison. Expected result: −1.

12. **Negative count within the subtraction domain.**  
    The count is negative but not the minimum signed 32-bit value. At initial index zero, the signed continuation condition fails. Expected result: −1 without entry-field comparisons.

## Uncertainties

The following remain **documented_only**:

- The application-level meanings of the object, collection, count, and compared fields.
- Dependency side effects, failure conventions, memory ownership, and lifetime rules.
- Whether callers can externally distinguish an early-exit zero from a matching-index zero.
- Intended behavior for a count of `−2,147,483,648`, where subtracting one exceeds the signed 32-bit range.
- Address-arithmetic behavior outside valid storage, including oversized indices and wrapped address calculations.
- Byte order, alignment requirements, volatile behavior, and atomicity of the four-byte reads.
- Exact physical ordering of the two operand reads within each equality comparison.
- Whether concurrent mutation is permitted by an external synchronization contract.
- Whether any surrounding runtime intercepts invalid-memory or arithmetic faults.

None of these gaps is filled by assuming conventional container, initializer, or ownership semantics.

## Evidence

The established contract comes solely from static inspection of the supplied function representation and the supplied opaque dependency mapping.

Static evidence supports the unconditional prerequisite call, early-exit ordering, four-byte field layout, optional acquisition call, post-call reference reread, repeated bound reads, ascending index traversal, short-circuit field comparisons, immediate first-match return, and absence of explicit direct writes or memory-management operations.

The collection-search purpose and the descriptive roles assigned to the dependencies are inferred intent. Dependency internals and environmental guarantees are unknown.

No runtime tests have been performed.
