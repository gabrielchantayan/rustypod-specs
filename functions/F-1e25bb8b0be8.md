<!-- Trusted provenance: F-1e25bb8b0be8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1e25bb8b0be8

## Purpose

F-1e25bb8b0be8 searches an indirectly referenced collection in ascending index order and returns the first candidate satisfying three conditions:

1. A comparison involving the candidate’s field at byte offset 12 returns a low 32-bit value of zero.
2. The candidate’s unsigned byte at offset 16 equals the requested signed 32-bit discriminator.
3. If an optional filter is enabled, a comparison involving the candidate’s field at byte offset 4 also returns a low 32-bit value of zero.

On success, the function writes a candidate pointer into the caller’s output cell and returns 1. If the search reaches its initially captured bound without finding a match, it returns 0.

The ordinary interpretation is a filtered lookup. The meaning of the compared fields, the comparison relation, and the collection’s ownership rules are not established. In particular, a zero comparison result must not be described as string equality, identifier equality, or any other specific relation without a separate contract for F-4a6c0bf0740c.

## Interface

### Inputs and result

The interface has four arguments:

| Argument | Established representation | Role |
|---|---|---|
| Optional filter | Signed 32-bit value | Zero disables the final comparison. A nonzero value is passed unchanged as the second argument of the final comparison. |
| Primary comparison value | Opaque 32-bit value | Passed as the second argument of the first comparison for each successfully retrieved candidate. |
| Required discriminator | Signed 32-bit value | Compared against a zero-extended candidate byte. |
| Output cell | Pointer to a writable 32-bit cell | Cleared on entry; receives a candidate pointer on success. |

The return value is 32 bits. The explicit normal return values are 0 and 1; no other normal return value is shown.

The supplied representation uses 32-bit pointers and 32-bit record words. Pointer values written into the output cell occupy one 32-bit word. These widths are part of this source-facing contract, not a claim of portability to another address model.

### Reachable layout

The function obtains a root pointer from global storage. Relative to that root:

- Byte offset 4 contains a 32-bit pointer to a collection object.

Relative to the collection object:

- Byte offset 0 contains a pointer to a dispatch table.
- Byte offset 4 contains a signed 32-bit search bound.

Relative to the dispatch table:

- Byte offset 60 contains the callable entry used to retrieve an indexed candidate.

A retrieved candidate is interpreted using these offsets:

| Candidate offset | Access width | Use |
|---|---:|---|
| 4 | 32 bits | First argument of the optional final comparison |
| 12 | 32 bits | First argument of the primary comparison |
| 16 | 8 bits, unsigned | Discriminator |

No candidate layout beyond those accessed fields is established.

### Preconditions

For an ordinary, defined execution:

- The output cell must be writable for a 32-bit store.
- The root, collection, dispatch table, and required fields must be readable whenever accessed.
- The dispatch entry must be callable with the collection object, a signed 32-bit index, and the address of a writable pointer slot.
- A retrieval reported as successful must leave that pointer slot in a state permitting the subsequent candidate accesses.
- F-4a6c0bf0740c must accept the supplied argument values.

The function performs no explicit null, alignment, bounds-consistency, or memory-lifetime checks.

The output cell is cleared before most collection accesses. If it aliases collection metadata or other reachable state, that write may change the subsequent search. Non-aliasing is therefore a prerequisite for the conventional interpretation of this operation as a read-only collection search, but it is not an enforced restriction.

## Behavior

### Entry and bound capture

The observable order begins with obtaining the root pointer from global storage. The function then writes a 32-bit zero into the caller’s output cell.

Using the captured root pointer, it reads the collection pointer at root offset 4, then reads the signed 32-bit search bound at collection offset 4. That bound is captured once for the entire search.

The function initializes:

- The search index to zero.
- A separate candidate-pointer slot to the address of the caller’s output cell.

The candidate-pointer slot is not the caller’s output cell. The retrieval operation receives the address of this separate slot and can replace its pointer value.

If the captured bound is zero or negative, the function immediately returns 0. No retrieval or comparison is performed in that case.

### Indexed retrieval

Before each retrieval, the function checks whether the current signed index is greater than or equal to the captured signed bound. If so, it returns 0.

Otherwise, it reads the collection pointer again from offset 4 of the captured root. It obtains the dispatch table from that collection object and invokes the entry at byte offset 60 with:

- The current collection object.
- The current index.
- The address of the candidate-pointer slot.

A retrieval result of zero causes the current index to be skipped without examining candidate fields. Any nonzero result permits candidate evaluation.

The function does not reset the candidate-pointer slot between retrievals. Consequently, its value can persist across failed retrievals or retrievals that do not replace it. A nonzero retrieval result is not independently checked for a null or valid candidate pointer.

The collection pointer is reloaded for each retrieval, but the bound is not. A dependency can therefore change which collection object supplies later candidates without changing the number of indices governed by the original bound.

Replacing the global root pointer itself does not redirect this invocation: the root pointer was captured before the output clear.

### Primary comparison

For a nonzero retrieval result, the function reads the candidate’s 32-bit field at offset 12 and calls F-4a6c0bf0740c with:

1. That field value.
2. The caller’s primary comparison value.

Only the low 32 bits of the returned value determine whether the candidate proceeds.

- A nonzero low word rejects the candidate.
- A zero low word permits discriminator evaluation.

The visible representation of this call includes a 64-bit result, but its upper 32 bits do not affect acceptance. In particular, a result with a zero low word and a nonzero upper word still passes this stage.

When the low word is nonzero, the candidate’s discriminator is not read by this function, and the optional final comparison is not called.

### Discriminator check

After the primary comparison passes, the function obtains the candidate pointer and reads one byte at candidate offset 16. That byte is zero-extended before comparison with the required signed 32-bit discriminator.

The discriminator can therefore match only values from 0 through 255 inclusive. For an ordinary execution, negative requested values and values greater than 255 cannot match any candidate.

The discriminator is read after the primary dependency returns. The function does not establish that the dependency preserves candidate storage or globally reachable state.

The intermediate representation combines the candidate pointer and the discriminator into a wider value. The established behavioral consequence is a pointer-width candidate value accompanied by an unsigned byte discriminator; there is no evidence that this wider value is itself stored into a candidate or returned to the caller.

### Optional final comparison

If the discriminator matches:

- When the optional filter is zero, no final comparison is performed.
- When the optional filter is nonzero, the function reads the 32-bit field at offset 4 of the candidate pointer captured for the discriminator stage and calls F-4a6c0bf0740c with that field and the optional filter.

The final comparison passes only when its low 32-bit result is zero. A nonzero low word rejects the candidate.

A zero optional filter is a control value that disables this comparison. It is not passed to the dependency as a filter value.

### Success and exhaustion

When all enabled checks pass, the function writes the candidate-pointer slot’s current value into the caller’s output cell and returns 1 immediately. No later index is retrieved.

Otherwise, it increments the index by one and repeats.

The signed index begins at zero and advances only while it is below the captured positive bound. Even when the bound is the maximum signed 32-bit value, the visible progression reaches that bound without requiring signed overflow.

Under noninterfering dependency behavior, the search examines indices in ascending order and returns the earliest accepted candidate.

The function explicitly clears the output cell at entry and explicitly writes it again only on success. Thus, in an ordinary noninterfering execution, return 0 leaves the output as zero. That is not an unconditional postcondition if dependencies or aliases can modify the output cell.

## Side effects

### Direct writes

The function directly performs:

1. A 32-bit zero store to the output cell before reading the search bound.
2. A 32-bit candidate-pointer store to the output cell on success.

Its index, captured bound, and candidate-pointer slot are local bookkeeping. The retrieval dependency can write the candidate-pointer slot through the address supplied to it.

No explicit candidate mutation, collection mutation, allocation, deallocation, reference-count adjustment, or release operation appears in the visible body.

### Dependency effects and lifetime

No preservation guarantee is established for either dependency. They may affect memory or global state unless constrained by contracts not supplied here.

In particular:

- The primary comparison occurs before the discriminator read.
- The optional comparison occurs before the successful output write.
- The pointer written on success is read from the candidate-pointer slot at that time.

If a dependency can retain or otherwise reach the candidate-pointer slot, later changes to that slot can affect subsequent reads or the returned pointer. The visible function does not enforce that the pointer written on success is identical to the pointer whose fields were examined earlier.

Returning a pointer does not establish ownership transfer. Whether the returned candidate is borrowed, retained, newly owned, stable across mutations, or subject to external invalidation is unknown.

### Errors

There is no explicit error-reporting channel distinct from the Boolean-style return value.

A retrieval result of zero is treated as a skipped index, not an immediate function-level error. Exhaustion returns 0 whether all retrievals failed, all candidates were rejected, or the initial bound was nonpositive.

Invalid pointers, unusable dispatch entries, or dependency failures have no visible recovery path in this body. Their precise effects are outside the established contract.

## Dependencies

### F-4a6c0bf0740c

This opaque dependency is used for both comparison stages.

Established usage:

- It receives two 32-bit arguments.
- The primary call receives the candidate field at offset 12 and the primary comparison value.
- The optional call receives the candidate field at offset 4 and the nonzero optional filter.
- A low 32-bit result of zero is treated as a match.
- No comparison is called for a retrieval returning zero.
- The optional call is made only after the primary comparison and discriminator both pass.

Unknown properties include comparison semantics, accepted argument domains, memory effects, failure behavior, and ownership behavior.

### Indirect retrieval dependency — ID documented_only

No opaque function ID is supplied for the dispatch-table target, so none is invented.

Established usage:

- Dispatch-table byte offset: 60.
- Arguments: collection object, current signed index, address of the candidate-pointer slot.
- Zero result: skip candidate evaluation.
- Nonzero result: evaluate the pointer slot as a candidate.

The meaning of the bound, candidate lifetime, slot-write guarantees, and behavior under concurrent collection changes remain undocumented.

## Acceptance examples

These examples are conditional specifications derived from the visible control flow, not executed tests. Unless stated otherwise, assume valid memory and dependencies that do not modify unrelated state.

1. **Nonpositive bound.**  
   With a captured bound of 0 or −1, the output is cleared, the function returns 0, and neither retrieval nor comparison is called.

2. **Skipped retrieval followed by success.**  
   With bound 2, retrieval at index 0 returns zero. Retrieval at index 1 supplies a valid candidate whose primary comparison has a zero low word and whose discriminator matches. With the optional filter disabled, the function writes that candidate pointer and returns 1.

3. **First match wins.**  
   With two acceptable candidates at indices 0 and 1, index 0 is returned. Index 1 is not retrieved.

4. **Primary mismatch short-circuits later checks.**  
   A successful retrieval followed by a primary result whose low word is 7 rejects the candidate. The function does not read its discriminator or perform the optional comparison.

5. **Upper return bits do not reject a primary match.**  
   A primary result of hexadecimal 00000001_00000000 passes the primary stage because its low 32 bits are zero. Acceptance still depends on the discriminator and any enabled final comparison.

6. **Unsigned byte boundary.**  
   A candidate byte of 255 can match a requested discriminator of 255. The same byte does not match −1. A requested discriminator of 256 cannot match any candidate byte.

7. **Disabled optional filter.**  
   With optional filter zero, a passing primary comparison and matching discriminator suffice. The field at offset 4 is not read for a final comparison.

8. **Enabled optional filter rejects.**  
   With a nonzero optional filter, a candidate passing the first two checks is rejected if the final comparison’s low word is nonzero. Search continues at the next index.

9. **Captured bound, changing collection.**  
   If a dependency changes the collection pointer stored in the captured root, the next retrieval uses the replacement collection. It still uses the original captured bound.

10. **Exhaustion output qualification.**  
    If no candidate passes and no dependency or alias modifies the output cell, return 0 leaves that cell zero. If a dependency modifies it, this function performs no final clear to restore zero.

## Uncertainties

The following are documented_only rather than assumed:

- Semantic names and types of the candidate fields.
- Whether F-4a6c0bf0740c implements equality or another zero-indicating relation.
- Whether the collection bound is a count, an exclusive limit, or another field used as a limit.
- Whether retrieval always writes its output slot on success.
- Whether dependencies preserve the output cell, candidate slot, candidate fields, collection, or global state.
- Ownership and lifetime of retrieved candidates.
- Thread safety, locking, reentrancy, and mutation guarantees.
- Whether callers permit aliasing between the output cell and reachable search state.
- Any independent error semantics behind a zero retrieval result.
- The indirect retrieval target’s opaque function ID.

The lookup interpretation is inferred intent. The accesses, comparisons, branching conditions, and explicit writes described above are established static behavior.

## Evidence

This contract is based solely on the supplied static function body and dependency mapping.

The static evidence establishes the argument widths, output clearing, one-time signed bound capture, ascending indexed retrieval, dispatch offset, candidate field offsets, short-circuit order, low-word comparison rules, unsigned discriminator range, and explicit return values.

No implementation access, dependency bodies, runtime traces, tests, or external ownership documentation were supplied. No runtime tests have been performed.
