<!-- Trusted provenance: F-a9ddf157cfdb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a9ddf157cfdb

## Purpose

F-a9ddf157cfdb performs an ordered membership search over values obtained from dependency F-93ea0dab0e54. It returns a 32-bit result equal to either zero or one.

The search limit is a signed 32-bit field read from the first input object at entry. For each eligible index, the dependency supplies a signed value. Nonnegative values are compared directly with a field in the third input object. Negative values are first adjusted using another field in that object.

The established behavior is an equality search with conditional adjustment. Interpreting the objects as a particular container, or the negative values as a particular indexing convention, is inferred intent rather than an established interface requirement.

## Interface

The function has three inputs, in this order:

1. **Source-object address.** An address represented by the supplied interface as an integer. The function directly reads a signed 32-bit field at byte offset 8 and passes the address at byte offset 4 to F-93ea0dab0e54.
2. **Opaque 32-bit argument.** The function does not inspect this argument or explicitly forward it to the dependency.
3. **Comparison-object address.** An address represented as an integer. The function reads a signed 32-bit field at byte offset 52 when comparing a candidate. It additionally reads a signed 32-bit field at byte offset 56 when the dependency returns a negative candidate.

The following names describe roles in this contract, not a known public object schema:

| Object | Byte offset | Width | Role |
|---|---:|---:|---|
| Source object | 4 | Address passed onward | Start of the region supplied to the dependency; no direct field read is shown here |
| Source object | 8 | 32 bits | Signed search limit, captured once |
| Comparison object | 52 | 32 bits | Equality target, read separately for each comparison |
| Comparison object | 56 | 32 bits | Adjustment base, read only for negative dependency results |

The output is a 32-bit word:

- **One:** an eligible candidate, after any required adjustment, equals the target read for that comparison.
- **Zero:** the captured search limit is nonpositive, or every eligible candidate fails its comparison.

No other explicit return value or error status appears in the supplied representation.

### Preconditions and representation limits

For a normal return, every direct read that is reached must address valid, readable storage. The initial read at source offset 8 is unconditional. Reads from the comparison object are conditional on at least one eligible iteration and on successful return from the dependency.

The dependency must be callable with the derived source-region address and the current signed index. Any additional requirements it places on that region are unknown.

The supplied representation uses 32-bit integer fields and a 32-bit return word. Pointer encoding, alignment rules, byte order, calling convention, and the behavior of invalid address arithmetic are not established. The address inputs should therefore be understood as object addresses in the represented environment, not as portable integer-to-pointer interfaces.

There is no explicit null check, bounds check on the pointed-to objects, or validation of their field values.

## Behavior

### Entry and search extent

The function first reads the signed 32-bit search limit from source offset 8. That value is retained for the whole invocation.

The search begins at index zero. Before each dependency call, the function compares the current index with the captured limit using signed ordering.

If the captured limit is zero or negative, the function returns zero immediately after the initial field read. It does not call F-93ea0dab0e54 and does not directly read either comparison-object field.

For a positive captured limit, the eligible indices are zero through one less than that limit. They are visited in increasing order. A mismatch advances to the next index; a match returns immediately.

Changes to the source limit field after the entry read do not extend or shorten this invocation’s search. The function does not reload that field.

### Candidate acquisition

For each eligible index, the function calls F-93ea0dab0e54 with:

- the source-object address advanced by four bytes; and
- the current index.

The first dependency argument is a derived address, not a value loaded from source offset 4.

The dependency result is interpreted as a signed 32-bit candidate. Its sign controls the next operation:

- A nonnegative result is used without adjustment.
- A negative result causes a read of the signed 32-bit adjustment base at comparison offset 56.

For a negative result, the compared value is the adjustment base plus the returned candidate plus one. For example, if the adjustment base is ten, a returned candidate of negative one becomes ten, and a returned candidate of negative two becomes nine.

This adjustment is performed once. There is no repeated normalization, clamping, range validation, or rejection merely because the adjusted value remains negative.

### Equality comparison

After obtaining the candidate and completing any required adjustment, the function reads the signed 32-bit target at comparison offset 52.

If the candidate equals that target, the function returns one. Otherwise, it advances the index and checks whether another iteration is eligible.

The target is not captured at entry. Each attempted comparison uses the target read after that iteration’s dependency call. Likewise, the adjustment base is not captured at entry and is read only on iterations whose dependency result is negative.

Consequently, this is not necessarily a search against one stable target or one stable adjustment base. If accessible state changes during dependency calls, different iterations can observe different comparison fields.

### Observable operation order

The established ordering of explicit reads and calls is:

1. Read the search limit at source offset 8.
2. Check whether index zero is eligible.
3. For each eligible index:
   - call F-93ea0dab0e54;
   - if its result is negative, read comparison offset 56;
   - read comparison offset 52;
   - return one on equality, or proceed to the next eligibility check.
4. Return zero when no eligible index remains.

For a negative dependency result, the adjustment-base read precedes the target read. For a nonnegative result, no adjustment-base read is shown.

A successful comparison prevents all later dependency calls and comparison-field reads. Exhaustion performs no final extra dependency call.

### Integer boundaries

The search limit, index, dependency result, adjustment base, and target are treated as signed 32-bit values.

The loop’s own index progression does not require signed overflow. Even with the largest positive signed 32-bit limit, the final eligible index is one less than that limit. Following a final mismatch, incrementing to the limit remains representable, and the next eligibility check terminates the search.

The negative-candidate adjustment has a separate arithmetic risk. Its mathematical result can fall outside the signed 32-bit range. The supplied representation does not establish whether such cases wrap, trap, or have another execution-level consequence. This contract therefore specifies the adjusted numeric value only where the relevant signed additions are representable. Overflow cases remain documented_only.

### Errors and abnormal completion

The function has no explicit error branch. In particular:

- a negative search limit produces zero, not a distinct error;
- a negative dependency result is a candidate requiring adjustment, not an error indicator;
- an adjusted candidate outside some application-defined logical range is still compared;
- dependency failure, invalid memory access, and arithmetic faults have no shown recovery path.

Normal completion is conditional on the dependency returning normally and on all reached memory operations and arithmetic being valid in the execution environment.

## Side effects

No direct memory write, allocation, release, or ownership transfer is shown in F-a9ddf157cfdb. Its explicit effects are field reads, dependency calls, and its return value.

That does not establish that the invocation is side-effect-free. F-93ea0dab0e54 may modify the source region, shared state, or storage reachable through aliases. No preservation guarantee is available for globals, object contents, or memory ownership.

The captured search limit is insulated from later changes to its backing field. The comparison fields are not: they are read after dependency execution on each applicable iteration.

The source and comparison addresses are not required by the visible logic to designate disjoint storage. If they overlap, the same ordering rules apply. Whether such aliasing is valid under the dependency’s contract is unknown.

The function itself neither retains nor releases either input address explicitly. This observation must not be extended into a guarantee about what the dependency retains, mutates, allocates, or frees.

## Dependencies

### F-93ea0dab0e54

This is the only explicit dependency.

**Inputs supplied by this function:**

- source-object address plus four bytes;
- a signed 32-bit index, beginning at zero and increasing by one after each mismatch.

**Output consumed by this function:**

- a signed 32-bit value, interpreted as either a direct candidate or a negative candidate requiring adjustment.

**Call-count constraints for normal execution:**

- No calls when the captured limit is nonpositive.
- One call per visited eligible index.
- At most the positive captured limit in calls.
- No calls after the first successful comparison.

Its internal representation, valid input domain, failure behavior, state preservation, and memory-management behavior are unknown. In particular, there is no established basis for assuming that it merely reads an element from an immutable collection.

## Acceptance examples

These are conditional acceptance criteria derived from the supplied representation, not runtime test results. Unless stated otherwise, assume valid storage, normal dependency returns, and representable arithmetic.

1. **Zero limit.**  
   With a captured search limit of zero, the result is zero. There are no dependency calls and no direct comparison-object reads.

2. **Negative limit.**  
   With a captured search limit of negative three, the result is zero under the same read and call restrictions as a zero limit.

3. **Immediate direct match.**  
   With a positive limit, a first dependency result of seven and a subsequently read target of seven produce one. The adjustment base is not read, and index one is not visited.

4. **Direct mismatch followed by exhaustion.**  
   With a limit of one, a dependency result of seven and a target of eight produce zero after exactly one dependency call. There is no call for index one.

5. **Negative-one adjustment.**  
   With a limit of one, a dependency result of negative one, an adjustment base of twelve, and a target of twelve produce one. The adjustment-base read precedes the target read.

6. **Adjusted value remains negative.**  
   With a limit of one, a dependency result of negative five and an adjustment base of two produce an adjusted candidate of negative two. A target of negative two therefore produces one; the function does not reject the candidate for remaining negative.

7. **Later match.**  
   With a captured limit of three, suppose index zero yields a mismatch and index one yields a match. The result is one, calls occur for indices zero and one only, and index two is not visited.

8. **Search-limit mutation.**  
   Suppose the captured limit is two and the first dependency call changes the source limit field to zero. If the first comparison fails, index one is still visited. This assumes the mutation does not invalidate other required state.

9. **Target mutation during a call.**  
   Suppose a dependency call returns nine and changes the comparison target to nine before returning. The ensuing comparison uses the target read after that call and can return one, regardless of the target’s earlier value.

10. **Nonnegative candidate bypasses adjustment.**  
    With a candidate of zero and a target of zero, the function returns one without directly reading the adjustment base. That field’s numeric contents do not affect this comparison.

11. **Overflow boundary.**  
    With an adjustment base equal to the minimum signed 32-bit value and a dependency result of negative one, the represented addition reaches an out-of-range intermediate before the final addition of one. This contract does not require a particular return value for that execution.

## Uncertainties

The following are documented_only:

- **Object meaning.** Field roles follow their use; no complete object layout or application-level type is established.
- **Dependency semantics.** Element retrieval is a plausible interpretation, not a guaranteed dependency contract.
- **Negative-value intent.** End-relative indexing is plausible, but only the stated adjustment is established.
- **Overflow semantics.** Signed adjustment overflow has no confirmed machine-level outcome.
- **Execution model.** Alignment, byte order, pointer width, address overflow, volatile behavior, and concurrent-memory guarantees are unspecified.
- **Lifetime and ownership.** The direct function shows no ownership operation, but dependency effects and retention are unknown.
- **Abnormal completion.** Faults, nonlocal exits, and dependency nontermination have no specified translation into a return value.

## Evidence

The supplied static representation establishes the entry-time limit read, signed loop comparison, ordered dependency calls, sign-dependent adjustment, per-iteration target read, early success return, and exhaustion return.

It also establishes the absence of an explicit use of the second argument and the absence of direct stores or explicit memory-management operations in this function.

The search and adjustment descriptions are behavioral abstractions of that representation. Application-level meanings, dependency purity, and arithmetic-overflow outcomes are not established.

No runtime tests have been performed. The acceptance examples are conditional consequences of the static evidence, not observed executions.
