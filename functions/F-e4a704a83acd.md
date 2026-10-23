<!-- Trusted provenance: F-e4a704a83acd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e4a704a83acd

## Purpose

F-e4a704a83acd searches a sequence of linked candidates and returns the address of the first candidate for which dependency F-fca51b715561 returns zero. It returns zero when the current candidate is null or is at or above a previously computed unsigned upper bound.

The established behavior is a bounded-address traversal, not a verified bounded-memory search. The function checks the candidate’s address against an upper bound, but does not establish that the candidate’s contents or successor field lie within valid memory.

An apparent intent is to search records associated with a globally described region. The meaning of a match, the region’s ownership, and the significance of the constant passed to the dependency are not established.

## Interface

### Input

The function accepts one 32-bit argument. It forwards that argument unchanged as the first argument to F-fca51b715561 whenever it examines an eligible candidate.

The supplied evidence does not establish whether this argument represents an address, integer, handle, or another four-byte value. No interpretation, validation, copying of pointed-to data, or ownership transfer is established by this function.

### Output

The function returns a 32-bit unsigned value:

- A nonzero candidate address when F-fca51b715561 returns zero for that candidate.
- Zero when the current candidate is zero or is greater than or equal to the captured upper bound.

Zero is therefore the explicit unsuccessful-search result on normal control flow. It is not an error code for invalid memory, dependency failure, or nontermination.

### Global inputs

The function reads two global 32-bit values, described here by their roles:

- **Starting-address global:** contributes to the upper-bound calculation and is read again to initialize the current candidate.
- **Extent global:** contributes to the upper-bound calculation.

These labels describe their use, not confirmed application-level meanings.

The upper bound is the 32-bit unsigned sum of the values used in the calculation. Arithmetic is modulo \(2^{32}\); overflow is not checked or saturated.

The starting-address global is then read separately to obtain the initial candidate. Consequently, the starting address used in the bound calculation and the initial candidate are not necessarily the same value if the global can change between reads.

### Candidate layout

The only candidate-relative field accessed directly by this function is:

| Offset from candidate address | Width | Use |
|---|---:|---|
| 40 bytes | 32 bits | Address of the next candidate after a nonzero dependency result |

The evidence does not establish a complete record size, field alignment requirement, byte order, or the meaning of any other candidate bytes.

The dependency receives the candidate address itself, not the address of the successor field.

## Behavior

### Initialization

Before examining candidates, the function computes an unsigned 32-bit upper bound from the starting-address global and the extent global. It then reads the starting-address global again and uses that value as the current candidate.

The evidence establishes this separation between calculating the bound and obtaining the initial candidate. It does not establish a left-to-right ordering between the two operand reads within the addition.

The resulting bound is retained for the traversal. The function does not explicitly refresh it after dependency calls or successor reads.

### Eligibility check

Before each possible dependency call, the function examines the current candidate.

It returns zero immediately if either condition holds:

- The current candidate is zero.
- The current candidate is greater than or equal to the captured upper bound, using an unsigned 32-bit comparison.

The upper bound is exclusive. A candidate exactly equal to it is not examined by the dependency.

There is no corresponding lower-bound check. A nonzero successor below the original starting address remains eligible if it is below the captured upper bound.

This check also does not reserve space for candidate contents. In particular, a candidate below the bound can have its successor field at or beyond that bound.

### Dependency call

For each eligible candidate, F-fca51b715561 is called with three arguments, in this order:

1. The function’s original 32-bit input.
2. The current 32-bit candidate address.
3. The constant numeric value 32.

The constant’s units and meaning are unknown. Although it could represent a length, that interpretation is not established by the call alone.

The dependency result is represented as a 32-bit integer and is tested only for equality to zero:

- Zero selects the current candidate.
- Any nonzero result causes traversal to continue.

Positive and negative nonzero results receive the same treatment. The function does not distinguish a mismatch from an error if the dependency uses nonzero values for both.

If the dependency returns zero, the function returns the current candidate without reading its successor field. It performs no final check of the candidate, global values, or memory contents before returning.

### Advancing to the next candidate

After a nonzero dependency result, the function reads a 32-bit successor address from 40 bytes beyond the current candidate address. That value becomes the next candidate, and the eligibility check is repeated.

The successor read occurs **after** the dependency returns. If the dependency changes that field and the memory remains readable, traversal uses the field’s post-call contents. The function does not capture the successor before calling the dependency.

The address used for this field access follows the supplied 32-bit address arithmetic. No overflow check is shown for adding 40 to the candidate address. The function supplies no explicit recovery if that calculation leads to an invalid access.

The successor itself is not dereferenced immediately. It first undergoes the null and unsigned-upper-bound checks on the next iteration.

### Termination and exceptional cases

Normal termination occurs only by:

- Returning a candidate after a zero dependency result.
- Returning zero after a null or out-of-bound candidate is encountered.

There is no iteration limit, cycle detection, or progress requirement. A self-link or cycle of eligible candidates can cause indefinite traversal if every dependency call returns nonzero.

The function provides no explicit handling for an unreadable successor field, a dependency that does not return, or an abnormal exit from the dependency. Those cases are not converted into a normal zero return by any mechanism shown here.

## Side effects

### Directly established effects

The function directly:

- Reads the two global values needed for the upper-bound calculation.
- Reads the starting-address global again for the initial candidate.
- Calls F-fca51b715561 once per eligible candidate reached.
- Reads a successor field after each nonzero dependency result.

No direct global assignment, candidate-field write, allocation, release, or input-buffer modification is shown.

### Effects through the dependency

F-fca51b715561 may have effects not visible in this evidence. In particular, this contract does not assume that it:

- Preserves globals or candidate contents.
- Leaves the successor field unchanged.
- Preserves the lifetime of candidate memory.
- Allocates or releases memory.
- Performs a pure comparison.
- Returns normally for all inputs.

Changes to the source globals do not, by themselves, cause this function to recalculate its captured bound or restart traversal. Changes to the current candidate’s successor field can affect the next step because that field is read after the call.

If the dependency invalidates the current candidate and then returns nonzero, the subsequent successor read may be invalid. No lifetime protection is established.

### Ownership

No ownership transfer is explicitly implemented. The returned nonzero value is the selected candidate address; it is not a newly allocated copy.

Whether callers may retain, mutate, or release the returned object is unknown. Likewise, the evidence does not establish that the input argument or candidate memory remains valid after the dependency executes.

## Dependencies

### F-fca51b715561

**Established call contract at this call site:**

- Receives the original input, the current candidate address, and numeric value 32.
- Is called only for a nonzero candidate strictly below the captured unsigned upper bound.
- Has its result interpreted solely as zero versus nonzero.
- Is called before any direct successor-field read for that candidate.

**Unknown dependency semantics:**

- The meaning of its three arguments.
- The memory ranges it reads or writes.
- Whether zero denotes equality, acceptance, success, or another condition.
- Whether nonzero results include errors.
- Global-state and ownership effects.
- Requirements concerning alignment, readable memory, or object lifetime.

Calling its zero result a “match” is descriptive shorthand for the selection rule, not an independently established semantic guarantee.

## Acceptance examples

These examples specify conditional outcomes from the supplied static evidence. They are not runtime observations. Unless stated otherwise, the global values remain stable during initialization, dependency calls return normally, and all required accesses are valid.

1. **Null initial candidate.**  
   With a starting-address value of zero, the initial candidate is zero. The function returns zero without calling the dependency, regardless of the calculated bound.

2. **Zero extent.**  
   With a starting address of 4,096 and an extent of zero, both the initial candidate and bound are 4,096. The function returns zero without examining that candidate.

3. **Immediate selection.**  
   With starting address 4,096 and extent 256, the bound is 4,352. If the dependency returns zero for candidate 4,096, the function returns 4,096. It does not read the successor field at address 4,136.

4. **Selection after one rejected candidate.**  
   Under the same bounds, suppose the dependency returns nonzero for 4,096, its successor field contains 4,160, and the dependency returns zero for 4,160. The return value is 4,160. The first successor field is read only after the first dependency call.

5. **Exclusive upper bound.**  
   If a rejected candidate’s successor is 4,352 under those bounds, the function returns zero on the next eligibility check. The dependency is not called with 4,352.

6. **No lower-bound enforcement.**  
   With initial candidate 4,096 and bound 4,352, a successor of 2,048 passes the address checks. The dependency is called with 2,048, assuming execution reaches that step.

7. **Post-call successor modification.**  
   If the dependency changes the current candidate’s successor from 4,160 to 4,224 before returning nonzero, and the candidate remains readable, the next candidate is 4,224.

8. **Nonzero negative result.**  
   If the dependency returns a negative 32-bit integer, the function follows the successor exactly as it would for a positive nonzero result. It does not return that dependency result.

9. **Wrapped bound.**  
   With starting address 4,294,967,280 and extent 32, the 32-bit bound is 16. The initial candidate is unsigned-greater-than the bound, so the function returns zero without a dependency call.

10. **Eligible cycle.**  
    If an eligible candidate points to itself and every dependency result is nonzero, traversal does not terminate normally, provided the calls and field reads continue to complete.

11. **Field beyond the bound.**  
    With bound 4,352, candidate 4,336 passes the eligibility check. After a nonzero dependency result, the function attempts its successor read at 4,376. The bound check does not prevent this access.

## Uncertainties

The following are **documented_only** rather than guessed requirements:

- Application-level meanings of the input, globals, candidates, and selection condition.
- Full candidate layout and the interpretation of numeric value 32.
- Byte order, required alignment, and platform behavior for invalid memory accesses.
- Synchronization and visibility guarantees if globals or candidate fields change concurrently.
- Dependency effects on memory, globals, object lifetime, and ownership.
- Caller obligations for retaining or releasing the returned address.
- Whether external invariants guarantee an acyclic list, valid field accesses, or a non-wrapping region.

A nonzero candidate below the captured bound is sufficient for the function’s eligibility check, but not sufficient to establish safe execution. Readability of any memory used by the dependency, and of the successor field when needed, remains an external prerequisite.

## Evidence

The supplied static representation establishes the 32-bit interface, unsigned bound calculation and comparisons, separate initial-candidate read, dependency argument order, zero-result selection rule, and 32-bit successor read at byte offset 40.

It also establishes the absence of explicit lower-bound checking, cycle detection, allocation, release, and direct writes in this function.

The interpretation as a search through region-associated linked records is inferred intent. Dependency semantics and environmental memory guarantees remain unknown.

No runtime tests have been performed.
