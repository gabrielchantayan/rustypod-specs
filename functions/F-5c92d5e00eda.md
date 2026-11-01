<!-- Trusted provenance: F-5c92d5e00eda; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5c92d5e00eda

## Purpose

F-5c92d5e00eda repeatedly obtains candidate records from F-5e2a2eae4f13 and returns the 32-bit value field of the first candidate that satisfies both of these conditions:

- Its association field equals the target reference selected for that iteration.
- Its value lies within the target’s inclusive, signed 32-bit interval.

Before requesting any candidate, the function initializes a field in the supplied context by copying one 32-bit word through an indirect reference. If the dependency returns zero before an acceptable candidate is found, the function returns the 32-bit pattern `0xffffffff`.

This is the established selection behavior. Interpreting the context field as an iterator cursor, or the dependency’s zero result as exhaustion rather than an error, is inferred intent rather than an established contract.

## Interface

### Inputs

The interface has three arguments:

1. **Selection-object address.** An address represented by a 32-bit integer in the supplied representation. A 32-bit target reference is read at byte offset `0x04` from this address after every nonzero dependency result.
2. **Unused 32-bit argument.** The function does not inspect, forward, or otherwise use this argument in the supplied body. Changing its bits alone does not change the body’s operations.
3. **Context address.** An address represented by a 32-bit integer. It is used for initialization and is the sole argument passed to F-5e2a2eae4f13.

No input structure size is supplied. No buffer length, ownership flag, callback, or explicit termination limit is present.

### Referenced layouts

Only the following fields are established. Offsets are byte offsets; no meaning is assigned to surrounding storage.

| Object | Offset | Width | Established use |
|---|---:|---:|---|
| Selection object | `0x04` | 32 bits | Target reference, reloaded for each nonzero candidate |
| Context | `0x28` | 32-bit address | Reference to the word copied during initialization |
| Context | `0x30` | 32 bits | Destination of the initialization copy |
| Initialization source | `0x00` | 32 bits | Word copied into context offset `0x30` |
| Candidate | `0x04` | 32 bits | Signed comparison value; also the return field |
| Candidate | `0x08` | 32 bits | Association value compared with the target reference |
| Target | `0x40` | 32 bits | Signed upper bound |
| Target | `0x44` | 32 bits | Signed lower bound |

The target reference has two roles: its 32-bit value is compared with the candidate’s association field, and it is used as an address when reading the bounds. Equality is exact 32-bit equality; it does not compare target contents.

### Output

The result is a 32-bit word:

- A zero dependency result produces `0xffffffff`.
- An accepted candidate produces the 32-bit contents read from its offset `0x04` for the return.

The failure-like result is not disjoint from successful results. An accepted value of signed `-1` also has the bit pattern `0xffffffff`. This function provides no separate status output to distinguish those cases.

### Preconditions

For defined execution, each access actually reached must refer to storage readable or writable as required by the execution environment:

- Context offset `0x28` must be readable.
- The address obtained there must identify a readable 32-bit word.
- Context offset `0x30` must be writable.
- After a nonzero dependency result, selection-object offset `0x04` and the candidate association field must be readable.
- Candidate value and target bound fields must be readable when their corresponding comparisons or return are reached.

The body does not validate null references, alignment, allocation size, or lifetime. Target-bound storage is not accessed on every path: an association mismatch skips both bound checks.

## Behavior

### Initialization

Before the first dependency call, the function:

1. Reads the address stored at context offset `0x28`.
2. Reads the 32-bit word at that address.
3. Writes that word to context offset `0x30`.

These accesses have a data-dependent order: the indirect source address must be obtained before its word can be read, and that word must be obtained before it can be stored.

The copy occurs once per invocation, not once per candidate. Its value is copied without signed interpretation or conversion. No later direct write to context offset `0x30` appears in this function.

Aliasing is not prohibited. If the destination overlaps other referenced storage, this initialization can affect subsequent reads. The behavior is defined by the accesses, not by an assumed separation between the context, selection object, source word, targets, and candidates.

### Candidate acquisition

After initialization, the function calls F-5e2a2eae4f13 with the context address. At least one such call is reached on any execution that completes initialization normally.

The dependency result is tested for zero before any selection-object field is read for that iteration.

- If the result is zero, the function immediately returns `0xffffffff`.
- Otherwise, the result is used as the candidate address.

The function then reads the target reference from selection-object offset `0x04`. This read occurs after the dependency call. It is repeated for each nonzero result, rather than being captured once at function entry.

Consequently, changes made by the dependency to the selection object can affect which target is used for the candidate just returned. The function does not establish that the dependency preserves the context, selection object, bounds, or other global or reachable state.

### Filtering

For each nonzero candidate, the function applies these rejection conditions in logical short-circuit order:

1. **Association mismatch.** Reject if the candidate’s 32-bit association at offset `0x08` differs from the target reference selected for this iteration.
2. **Below the lower bound.** If association matches, reject if the candidate’s signed 32-bit value at offset `0x04` is less than the target’s signed 32-bit lower bound at offset `0x44`.
3. **Above the upper bound.** If association matches and the lower-bound check passes, reject if the target’s signed 32-bit upper bound at offset `0x40` is less than the candidate’s signed 32-bit value.

A rejection causes another dependency call with the same context-address argument. There is no explicit rejection count or iteration limit.

A candidate is accepted only when all three rejection conditions are false. With stable field contents, this means:

- Association equals the selected target reference.
- Lower bound ≤ candidate value ≤ upper bound, using signed 32-bit comparisons.

Both endpoints are inclusive. The function does not normalize reversed bounds or interpret them as a wraparound interval.

The short-circuit structure establishes which later checks are skipped. It does not establish the relative order of the two operand reads within an individual comparison.

### Successful return

After acceptance, the function reads candidate offset `0x04` for the return.

The supplied representation contains separate value reads for the lower-bound comparison, upper-bound comparison, and return. With ordinary stable storage, these reads yield the same value. A stronger snapshot guarantee is not established: if storage can change between accesses, the returned word is not independently revalidated after its final read.

### Termination and arithmetic

Normal return occurs only when a dependency result is zero or a candidate is accepted. An indefinitely continuing sequence of rejected nonzero candidates can keep the function running indefinitely.

Candidate values and bounds are compared as signed 32-bit integers, with range −2,147,483,648 through 2,147,483,647. No value increment, subtraction, interval-width calculation, or unsigned range transformation is performed.

Address calculations use fixed field offsets. The supplied representation does not justify promises about address wraparound or accesses outside valid objects.

## Side effects

The sole direct write established in this body is the initialization write to context offset `0x30`.

Other effects arise from calls to F-5e2a2eae4f13 and are not defined by this body. In particular, the specification does not assume that the dependency:

- Preserves context fields or globals.
- Leaves the selection object or target bounds unchanged.
- Returns distinct candidates.
- Advances an iterator.
- Allocates, retains, transfers, or releases candidate storage.
- Treats zero specifically as exhaustion rather than failure.

This function itself contains no explicit allocation, deallocation, candidate release, logging, or rollback. Initialization has already occurred if the first dependency call returns zero.

There is no explicit recovery path for invalid memory accesses or dependency failures other than a normal zero return. Behavior when the dependency does not return normally is documented only as unknown.

## Dependencies

### F-5e2a2eae4f13

**Invocation:** Called with the context address as its sole argument.

**Result use:** Zero terminates this function with `0xffffffff`. A nonzero result is treated as a candidate address and inspected at the offsets specified above.

**Call frequency:** Once initially after initialization, then once after each rejected candidate.

**Required storage validity:** A nonzero result must support every candidate-field read reached before the next call or return. The selected target must similarly support any reached bound reads.

**Unknown contract:** No implementation or independent behavioral contract for this dependency is supplied. Its state changes, ownership rules, failure distinctions, and termination guarantees remain documented_only.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Except where noted, assume valid, stable storage and normally returning dependency calls.

1. **Immediate zero result.** Initialization copies the indirect source word into context offset `0x30`. The first dependency call returns zero. The result is `0xffffffff`; selection-object offset `0x04` is not read on this path.

2. **Inclusive endpoints.** The selected target has lower bound `10` and upper bound `20`. A matching candidate with value `10` is accepted and returns `10`. A matching candidate with value `20` is likewise accepted.

3. **Association rejection.** A candidate’s value is within the target interval, but its association differs from the selected target reference. It is rejected without reading target bounds for that candidate, and another dependency call follows.

4. **Lower-bound short circuit.** A matching candidate has value `9`, with lower bound `10`. It is rejected. The upper-bound check is skipped.

5. **Upper-bound rejection.** A matching candidate has value `21`, with lower bound `10` and upper bound `20`. It passes the lower-bound check, fails the upper-bound check, and causes another dependency call.

6. **Signed comparison.** A matching candidate has value `-2`, and the target interval is `[-3, 1]`. The candidate is accepted. Its negative value is not treated as a large unsigned number.

7. **Indistinguishable return patterns.** A matching candidate with value `-1` in interval `[-5, 5]` is accepted and returns `0xffffffff`. The same output bits result from a zero dependency return.

8. **Reversed bounds.** A stable target has lower bound `20` and upper bound `10`. No signed 32-bit value can satisfy both bounds. Matching candidates continue to be rejected; the function returns only if a later dependency result is zero or relevant state changes.

9. **Target changed by dependency.** During a dependency call, the selection object’s target reference changes from one target to another. The nonzero candidate returned by that call is evaluated against the reference read after the call, not the prior reference.

10. **Repeated rejected candidate.** The dependency repeatedly returns the same nonzero candidate, which remains unacceptable. The function continues calling the dependency; it does not detect repetition or impose a retry limit.

## Uncertainties

- **Documented_only — dependency semantics:** Zero means termination here, but whether it represents exhaustion, failure, or another condition is unknown.
- **Documented_only — ownership:** No evidence establishes ownership transfer, retention rights, or release obligations for candidates or targets.
- **Documented_only — execution environment:** Exact ABI, byte order, alignment requirements, memory-fault behavior, and pointer-validity rules are not supplied.
- **Documented_only — changing storage:** Concurrent modification, volatile access semantics, and synchronization guarantees are unspecified. Acceptance of a single stable candidate value must not be generalized into an atomic snapshot guarantee.
- **Inferred intent — traversal initialization:** The initial indirect copy may initialize traversal state, but only the copy itself is established.
- **Inferred intent — target relationship:** The association equality appears to select candidates belonging to a target. No richer identity, type, or lifetime relationship is established.
- **Documented_only — exceptional control flow:** Behavior following a dependency exception, nonlocal exit, or invalid access is not defined by the available evidence.

## Evidence

The supplied static body establishes the initialization copy, dependency-call loop, zero-result return, per-candidate target reload, short-circuit rejection conditions, signed comparisons, and final 32-bit field return.

The widths and offsets above follow the supplied 32-bit field and integer representations. No surrounding declarations, dependency implementation, memory-layout documentation, or runtime traces were provided.

No runtime tests have been performed. The acceptance examples describe consequences of the static evidence under their stated conditions; they are not observed execution results.
