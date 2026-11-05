<!-- Trusted provenance: F-a06fe9e0c3a7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a06fe9e0c3a7

## Purpose

F-a06fe9e0c3a7 conditionally returns a 32-bit word from an object-associated table. A dependency result and an unsigned index comparison control eligibility. An eligible request enters an access sequence, obtains and prepares a candidate if the cache is absent, and reads a word using the current cached address.

Zero is not a distinct failure indication. It can represent rejected eligibility, an absent candidate, a cache cleared during preparation, or a retrieved word whose value is zero.

**Established static evidence:** the function performs conditional dependency calls, object-field reads, a conditional cache-slot write, argument reads for preparation, and a final indexed 32-bit read.

**Inferred intent:** the paired access dependencies may delimit synchronized access, and candidate creation plus preparation may establish a usable table. Their names and placement do not establish synchronization, allocation, initialization, or ownership contracts.

The final address calculation uses finite-width arithmetic. Consequently, a selected-entry read can address the table header, even on a path that skips preparation and its separate header-argument read.

## Interface

### Inputs

| Input | Representation | Local use |
|---|---|---|
| Object address | 32-bit address-bearing value | Identifies object storage; zero receives special handling during eligibility checking |
| Index | Unsigned 32-bit integer | Compared with the object’s bound and multiplied by four for lookup |

A nonzero object address is not otherwise validated.

### Object layout

| Byte offset from object base | Width | Observed role |
|---|---:|---|
| 8 | 32 bits | Unsigned eligibility bound |
| 2512 | 32 bits | Argument saved for both access dependencies |
| 2520 | 32 bits | Cached table address |

A zero cache value means absent at the particular check that observes it. A nonzero value is subsequently usable as an address without a local validity or extent check.

### Table interpretation

| Byte offset from table base | Width | Observed role |
|---|---:|---|
| 0 | 32 bits | Header supplied as an argument when preparing a new nonnull candidate |
| 4 | 32 bits | Selected word for index zero, absent address wrap |
| 8 | 32 bits | Selected word for index one, absent address wrap |
| 4 + 4 × index | 32 bits | General selected-word position, subject to 32-bit arithmetic |

The header is not used as a local eligibility bound.

An existing cache skips the **separate header read used to supply preparation’s argument**. It does not necessarily avoid reading the header’s address: wrapped selected-entry addressing can make the final read target byte offset zero.

These are logical storage roles, not a guarantee that all referenced storage is disjoint.

### Preparation’s global input

Preparation receives a value read from a global datum. This is not a fixed literal or a value captured at function entry.

The supplied evidence does not include that datum’s declaration. Its exact storage width, signedness, meaning, and permitted values remain **documented_only**. Its read location in the execution sequence is established.

### Output

The function returns one 32-bit word:

- Zero when the initial dependency returns zero.
- Zero when the unsigned eligibility comparison fails, including for a null object.
- Zero when candidate creation returns a null address.
- Zero when preparation returns and the following cache check observes zero.
- Otherwise, the word obtained by the final selected-entry read.

No interpretation of a returned word as signed data, an address, a status, or another semantic type is established.

### Memory-access preconditions

For each reached path:

- Object storage must support the required field reads and, when applicable, the cache-slot write.
- A nonnull new candidate must support the separate 32-bit header read used for preparation.
- The global datum must be readable when preparation arguments are evaluated.
- The address actually formed for the final lookup must support a 32-bit read.
- Required storage lifetimes and execution-environment alignment constraints must hold across intervening calls.

Passing the eligibility comparison does not prove any of these conditions. Invalid storage or address formation is not converted into a defined zero result.

## Behavior

### Initial gate and eligibility

F-cd52fcefba54 is called first, without explicit arguments.

If it returns zero, the function returns zero. On this path, the function itself does not read object storage, table storage, or the preparation-input global, and calls no other dependency. The gate’s own effects are unknown.

If the gate returns nonzero, the eligibility bound is obtained as follows:

- For a zero object address, the effective bound is zero without dereferencing the object.
- For a nonzero object address, the bound is read as an unsigned 32-bit word at object offset 8.

The unsigned index must be strictly less than the sampled bound. Equality rejects. A zero effective bound rejects every possible index.

Rejection returns zero without reading the access argument or cache slot and without calling the access pair. The bound is sampled once; later changes do not cause another eligibility check.

### Entering the eligible access sequence

After eligibility succeeds, the function reads the 32-bit argument at object offset 2512 and saves that value. It calls F-338d16f9ce72 with the saved argument.

That dependency’s return value is ignored. There is no local success test based on it.

The initial cache-slot read occurs only after F-338d16f9ce72 returns. Changes made during that call can therefore determine whether the function uses an existing cache or requests a candidate.

The saved argument, rather than a later reread of object offset 2512, is used for the exit dependency.

### Existing-cache path

If the initial cache check observes a nonzero address:

- F-f16e3e6dbf6b is not called.
- F-b4b5aa4c2f83 is not called.
- The separate candidate-header read for preparation is skipped.
- The preparation-input global is not directly read.

The function proceeds to the final selected-entry access. That access reads the cache slot again to obtain the address used in its address expression.

The permitting nonzero check does not validate readiness, extent, generation, or header contents. Nor does skipping the preparation-header read guarantee that the final selected-entry read avoids the header’s address. For example, the index 1,073,741,823 produces a net zero displacement after 32-bit wrapping.

No dependency call intervenes between the initial nonzero cache check and final address formation on this path. Nevertheless, the supplied body does not establish atomicity or stability against concurrent or externally mediated changes.

### Absent-cache path

If the initial cache check observes zero, F-f16e3e6dbf6b is called with the object address.

The returned candidate address is written to object offset 2520 before it is tested for zero. This explicit store overwrites any value the dependency may itself have left in that slot.

If the returned address is zero:

- The explicit store leaves the cache slot zero.
- Preparation is skipped.
- No separate candidate-header read or preparation-input global read occurs.
- No selected-entry read occurs.
- F-1333331b2da1 is called with the saved access argument.
- The function returns zero.

If the returned candidate is nonzero, F-b4b5aa4c2f83 is called with four arguments:

1. The address four bytes after the returned candidate’s base.
2. The candidate’s first 32-bit word.
3. The value read from the preparation-input global.
4. Zero.

The first two arguments derive from the returned candidate, not from a fresh cache-slot read. A dependency-induced cache replacement does not retroactively change which candidate supplies these arguments.

The candidate-header and global reads occur after candidate creation returns and after the caller stores the candidate in the cache slot. Both occur before preparation is invoked. Their relative evaluation order is not established by the supplied evidence.

Earlier dependencies may change the global. Preparation therefore receives the value read during argument evaluation, not a value presumed to have existed at entry.

Preparation’s return value is ignored. After it returns, the function checks the cache slot:

- If the slot is zero, the function calls F-1333331b2da1 with the saved argument and returns zero without a selected-entry read.
- If the slot is nonzero, the function proceeds to lookup.

Preparation can leave, clear, or replace the cached address. The caller does not restore the original candidate or derive the lookup decision from preparation’s return value.

### Final lookup and exit

The final lookup forms an address from:

- A fresh read of the cached table address.
- The index multiplied by four.
- An additional displacement of four bytes.

It reads one 32-bit word from the resulting address, saves that word, calls F-1333331b2da1 with the saved access argument, and returns the saved word.

The selected word is read before the exit call. Changes made by that dependency do not trigger another selected-entry read or alter the already saved return value.

The cache read used for final address formation is distinct from the earlier nonzero check. The supplied body provides no guarantee that the two reads observe the same value.

Every visible normal-return path that calls F-338d16f9ce72 subsequently calls F-1333331b2da1 exactly once with the same saved argument. This establishes normal control-flow pairing, not exception-safe cleanup or synchronization semantics.

### Arithmetic and header aliasing

The index and bound are unsigned 32-bit values. Multiplication by four is unsigned 32-bit arithmetic, wrapping modulo 4,294,967,296.

The represented final address calculation also combines the cached address and displacements in a 32-bit address model without a local overflow check.

Two important boundary cases are:

- **Index 1,073,741,823:** the product is 4,294,967,292, or 0xFFFFFFFC. Adding the four-byte displacement wraps the net displacement to zero. Under the represented 32-bit address computation, the selected-entry read addresses the cached table base—the header location.
- **Index 1,073,741,824:** the product wraps to zero. The additional four-byte displacement places the selected-entry read at table offset four, provided the address is valid.

Either index can pass eligibility if the sampled unsigned bound is greater than it. The function imposes no smaller index ceiling.

A passing comparison therefore does not guarantee nonwrapping arithmetic, an address within a table allocation, distinct addresses for distinct indices, or avoidance of the header. Exact execution-environment consequences of invalid or overflowing address formation remain **documented_only**; a defined lookup requires valid storage at the address actually used.

## Side effects

### Direct reads and their ordering

Depending on the path, the function directly reads:

1. The eligibility bound, after a nonzero gate result and only for a nonnull object.
2. The access argument, after eligibility succeeds.
3. The cache slot, after the entry dependency returns.
4. A new candidate’s header and the preparation-input global, after storing a nonnull candidate and before preparation.
5. The cache slot after preparation, if preparation occurred.
6. The cache slot used in final address formation.
7. The final selected 32-bit word, before the exit dependency.

The relative order of the two preparation-argument reads is not established.

The final selected-word read can coincide with the header location through wrapped addressing. Listing it separately from the preparation-header read identifies its purpose and control-flow position, not necessarily a different address.

### Direct writes

The sole explicit memory write is storing the candidate-creation result into the cache slot. This includes storing zero.

There is no separate explicit write to the bound, access-argument field, global datum, header, or table entries. This describes logical access roles; aliasing can make a cache-slot store affect storage also serving another role.

### Dependency effects and ownership

Dependency effects are not defined by this body. No dependency is assumed to preserve globals, object fields, table contents, storage validity, or unrelated state.

There is no explicit deallocation. Storing a candidate does not establish ownership transfer. If preparation clears or replaces the cache, the caller performs no visible release of the original candidate. Ownership, retention, and release obligations remain unknown.

### Errors and abnormal completion

Normal zero-result paths have no separate diagnostic or error code.

If a dependency does not return normally, or a required memory access faults, recovery is not established. The normal-path access pairing does not prove that the exit dependency runs after interruption.

## Dependencies

| Opaque function ID | Caller-visible use |
|---|---|
| F-cd52fcefba54 | First call, without explicit arguments. Zero rejects; nonzero permits eligibility checking. |
| F-338d16f9ce72 | Called after eligibility succeeds with the saved 32-bit access argument. Return value ignored. |
| F-f16e3e6dbf6b | Called with the object address when the initial cache check observes zero. Its result is stored before null testing. |
| F-b4b5aa4c2f83 | Called for a nonnull new candidate with its payload address, header word, current global argument value, and zero. Return value ignored. |
| F-1333331b2da1 | Called once on each normal exit from the entered access sequence, using the saved access argument. Return value ignored. |

These descriptions specify invocation conditions and result use, not internal dependency contracts.

## Acceptance examples

These examples are conditional consequences of static evidence, not observed executions. Unless stated otherwise, assume valid storage, normal dependency returns, and stable relevant storage except for the specified changes.

1. **Gate rejection.** If F-cd52fcefba54 returns zero, the result is zero. No other dependency is called, and this function performs no direct object, table, or preparation-global access.

2. **Null object.** With a nonzero gate result and a null object, every unsigned index fails against the effective zero bound. The object is not dereferenced.

3. **Exact bound.** With bound three and index three, the function returns zero before reading the access argument or cache slot.

4. **Ordinary existing-cache lookup.** With bound three, index one, and a stable valid cache address, the function reads the word at table offset eight and returns it after the exit call. Candidate creation, preparation, the separate preparation-header read, and the global read are skipped.

5. **Existing-cache lookup reaches the header.** With bound 1,073,741,824 and index 1,073,741,823, eligibility succeeds. With a stable valid cache address, wrapped addressing selects the word at the table base. The header value is returned after the exit call, despite preparation and its separate header-argument read being skipped.

6. **Null candidate overwrites an internal change.** If candidate creation changes the cache slot but returns zero, the caller overwrites that change with zero. Preparation and final lookup are skipped; the exit call precedes the zero return.

7. **Current global input.** If candidate creation changes the preparation-input global from A to B and returns a nonnull candidate, preparation receives B if no intervening change occurs. It does not receive an entry-time snapshot of A.

8. **Argument-read ordering.** A nonnull candidate requires both the header value and global value for preparation. Acceptance must not require either read to occur first.

9. **Preparation clears or replaces the cache.** Clearing causes a zero return after the exit call without a selected-entry read. Replacing it with another stable valid table address makes the final lookup use that replacement, although preparation’s candidate-derived arguments came from the original candidate.

10. **Saved values remain authoritative.** If the access argument is saved as A and the selected word as X, later changes to their source storage do not change the exit argument A or returned word X.

11. **No bound recheck.** Reducing the bound after eligibility does not itself reject the request. Defined execution still requires a valid final read address.

12. **Wrapped stride and zero-valued success.** Index 1,073,741,824 selects offset four under the stated arithmetic when eligible. A retrieved zero word returns zero with no indication distinguishing it from rejection or absence.

## Uncertainties

The following remain **documented_only**:

- Gate meaning and rejection reasons.
- Synchronization, reentrancy, and failure semantics of the access pair.
- Candidate allocation, ownership, lifetime, and extent.
- Header meaning and relationship to the object’s bound.
- Preparation-global declaration, width, interpretation, values, and writers.
- Relative ordering of the header and global argument reads.
- Whether preparation intentionally communicates through cache clearing or replacement.
- Storage-aliasing restrictions, concurrency rules, atomicity, and memory ordering.
- Alignment, byte order, and fault handling.
- Semantic interpretation of returned words.
- External index restrictions that might exclude arithmetic wrap.
- Execution-environment consequences of invalid or overflowing address formation.

No runtime tests have been performed.

## Evidence

The supplied static body establishes the gate-first call, unsigned eligibility comparison, object-field offsets, saved access argument, cache checks and assignment, conditional preparation, final indexed 32-bit read, and paired normal exits.

The preparation-global read occurs on the nonnull-new-candidate path after candidate creation and cache assignment, before preparation. Its declaration and its read order relative to the candidate header are not supplied.

The unsigned 32-bit stride establishes the header-aliasing case: index 1,073,741,823 yields 0xFFFFFFFC, and the additional four-byte displacement wraps to zero. Thus, skipping preparation’s separate header read does not establish that lookup avoids the header location.

Dependency internals, ownership, synchronization, and storage validity are not established by the caller’s willingness to continue using state after dependency calls.
