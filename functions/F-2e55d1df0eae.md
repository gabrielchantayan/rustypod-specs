<!-- Trusted provenance: F-2e55d1df0eae; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2e55d1df0eae

## Purpose

F-2e55d1df0eae initializes selected fields of a caller-supplied record, classifies an unsigned 32-bit input into one of nine numeric categories, and invokes F-be29e3b6fc7e with that category and additional context. If the dependency returns normally, its 32-bit result is stored in the record, and F-2e55d1df0eae returns zero.

The record’s larger purpose, the category meanings, and the interpretation of the dependency’s result are not established. Describing this operation as record preparation followed by delegation is supported by the visible behavior; assigning a domain-specific purpose is not.

## Interface

The interface has eight positional arguments and a 32-bit return value.

| Argument | Contract |
|---|---|
| 1 | Opaque 32-bit value. Written to record offset 0 and passed as the dependency’s fifth argument. |
| 2 | Opaque 32-bit value. Written to record offset 4. |
| 3 | Unused by the visible function body. It is neither inspected, stored, nor forwarded. |
| 4 | Opaque 32-bit value. Passed as the dependency’s third argument. |
| 5 | Opaque 32-bit value. Written to record offset 12. |
| 6 | Unsigned 32-bit classification input. Written unchanged to record offset 8 and separately mapped to a category. |
| 7 | Address of the caller-supplied record. Used for direct writes and passed as the dependency’s fourth argument. |
| 8 | A 32-bit value interpreted as an integer for a zero comparison. A nonzero value is used unchanged; zero selects a value read from shared default storage. The selected value is written to record offset 20. |

The visible record accesses use six consecutive four-byte slots:

| Byte offset | Width | Visible use |
|---|---:|---|
| 0 | 32 bits | Receives argument 1 before delegation. |
| 4 | 32 bits | Receives argument 2 before delegation. |
| 8 | 32 bits | Receives argument 6 unchanged before delegation. |
| 12 | 32 bits | Receives argument 5 before delegation. |
| 16 | 32 bits | Receives the dependency’s return value after delegation. Not initialized by this function before the call. |
| 20 | 32 bits | Receives the selected explicit or default value before delegation. |

These accesses require writable storage through byte offset 23. The function performs no visible null check, capacity check, alignment check, or lifetime check. The dependency may impose additional requirements that cannot be established here.

Two shared 32-bit values are relevant:

- **Default value:** read only when argument 8 is zero.
- **Dependency context:** read to supply the dependency’s first argument.

These labels distinguish their visible roles, not their semantic types. Neither value is established to be a pointer, handle, flag set, or numeric configuration parameter.

The record layout is defined in four-byte offsets and 32-bit values. The supplied evidence does not independently establish byte order or a portable alignment rule.

## Behavior

### Default selection

Before writing the record, the function selects the value for offset 20.

- If argument 8 is nonzero, its original 32-bit value is selected.
- If argument 8 is zero, the function reads the shared default value and selects the value obtained from that read.

A negative argument 8 is nonzero and therefore bypasses the default read. A shared default value of zero is accepted as the selected value: there is no second fallback or rejection.

The shared default is sampled before the record writes and before the dependency call. A later change to that shared storage does not retroactively change the value already selected for the pre-call write.

### Classification

Argument 6 is classified as an unsigned 32-bit value. The resulting category is also passed as a 32-bit value.

| Argument 6 | Category |
|---|---:|
| 0 | 0 |
| 1 through 15, inclusive | 1 |
| 16 | 2 |
| 24 | 3 |
| 32 | 4 |
| 64 | 5 |
| 128 | 6 |
| 192 | 7 |
| 256 | 8 |
| Every other unsigned 32-bit value | 3 |

The listed exceptional values are exact matches, not thresholds. For example, 31 produces category 3, 32 produces category 4, and 33 produces category 3. Values above 256 also produce category 3.

The mapping is not a general division, rounding, saturation, or logarithmic conversion. Although some recognized inputs resemble sizes or bit counts, that interpretation is not established.

There is no visible arithmetic on argument 6 beyond comparison. Consequently, the mapping introduces no addition, multiplication, or shift overflow. Its unsigned interpretation matters: the bit pattern representing the maximum unsigned 32-bit value maps to category 3, not to the category for values below 16.

### Record preparation and delegation

After selecting the default and classifying argument 6, the function performs these direct 32-bit writes in the following order:

1. Argument 1 to offset 0.
2. Argument 2 to offset 4.
3. Argument 6, unchanged, to offset 8.
4. Argument 5 to offset 12.
5. The selected explicit or default value to offset 20.

It does not directly write offset 16 at this stage. With ordinary non-overlapping storage, that slot retains its previous contents until the dependency acts or the function performs its final write.

The function then invokes F-be29e3b6fc7e once, with five arguments in this positional order:

| Dependency argument | Supplied value |
|---|---|
| 1 | Value read from shared dependency-context storage. |
| 2 | Computed category. |
| 3 | Argument 4 of F-2e55d1df0eae. |
| 4 | The caller-supplied record address. |
| 5 | Argument 1 of F-2e55d1df0eae. |

The dependency-context read occurs after the five pre-call record writes. The evidence does not establish a meaningful evaluation order among the call’s individual argument expressions beyond the sequencing of the preceding writes.

If the dependency returns normally, the function writes its returned 32-bit value to record offset 16. It then returns the 32-bit value zero.

### Completion and errors

No visible branch returns a nonzero status. No classification input is rejected: every unsigned 32-bit argument 6 has a category.

The dependency’s result is stored rather than propagated as the function’s return value. A result that a caller might interpret as failure does not, by itself, change this function’s zero return.

This normal-return contract does not cover invalid memory access, dependency non-return, or other abnormal control transfer. There is no visible recovery, rollback, cleanup handler, or validation path for such conditions.

## Side effects

The function directly writes six 32-bit record fields across normal execution: five before delegation and one after it. It directly reads the shared dependency context and conditionally reads the shared default.

The dependency can observe the prepared record because it receives the same record address after the five initial writes. In particular, it can observe the unchanged classification input at offset 8, not merely the derived category.

No guarantee is established that the dependency preserves the record or either shared value. Therefore:

- The pre-call writes are guaranteed events, but their values are not unconditional post-return invariants.
- If the dependency changes offsets 0, 4, 8, 12, or 20, this function does not restore them.
- If the dependency returns normally and the final write succeeds, offset 16 contains its returned 32-bit value, replacing whatever was there immediately beforehand.
- Changes made by the dependency to globals or other memory are not visibly undone.

No allocation, deallocation, reference-count operation, or ownership transfer is visible in this function. Passing the record address to the dependency does not establish whether that dependency retains it, releases associated resources, or requires exclusive access. Normal completion requires the record address to remain valid for the final write after the call.

Aliasing can make the read/write order significant. If the record overlaps shared dependency-context storage, a pre-call write may change the value subsequently read as the dependency context. If it overlaps shared default storage, the selected default has already been sampled before the writes begin. No non-aliasing guarantee is supplied.

No synchronization mechanism is visible. Concurrent modification of the record or shared values has no established coherent-snapshot contract.

## Dependencies

**F-be29e3b6fc7e**

- Called exactly once on the ordinary execution path.
- Receives five arguments in the order specified above.
- Receives access to the caller’s record after partial initialization.
- Supplies the 32-bit value written to offset 16.
- Has no established error convention, mutation restrictions, ownership rules, retention policy, or termination guarantee in the supplied evidence.

The pre-call contents of offset 16 may be relevant to this dependency because the function does not initialize that field first. Whether the dependency reads it is unknown.

The two shared-value reads are additional environmental dependencies. Their initialization, validity, update policy, and semantic meanings are not established.

## Acceptance examples

These examples are conditional behavioral criteria, not reports of executed tests. Unless stated otherwise, assume valid writable record storage, no overlapping shared storage, no concurrent mutation, and a dependency that returns normally.

1. **Explicit selection and recognized category.**  
   Given argument 6 equal to 32 and argument 8 equal to 9, the dependency receives category 4. Immediately before the call, offset 8 contains 32 and offset 20 contains 9. The shared default is not read by this function.

2. **Default selection preserves zero.**  
   Given argument 8 equal to zero and shared default equal to zero, offset 20 is written with zero before delegation. There is no further fallback.

3. **Negative explicit value bypasses defaulting.**  
   Given a negative 32-bit argument 8, its bit pattern is written to offset 20. It is not replaced merely because it is negative.

4. **Low-input boundary.**  
   Inputs 0, 1, 15, 16, and 17 produce categories 0, 1, 1, 2, and 3, respectively. In each case, offset 8 receives the original input.

5. **Exact-match behavior.**  
   Inputs 191, 192, and 193 produce categories 3, 7, and 3. Input 256 produces category 8; input 257 produces category 3.

6. **Unsigned upper boundary.**  
   An argument 6 of 4,294,967,295 produces category 3 and is written unchanged to offset 8.

7. **Unused third argument.**  
   Changing only argument 3 does not change this function’s visible classification, selected value, direct writes, or dependency arguments.

8. **Dependency result and record mutation.**  
   If the dependency changes offset 0 and returns the 32-bit value 37, this function does not restore offset 0. It writes 37 to offset 16 and returns zero. This criterion assumes the dependency leaves the record writable.

9. **Previous result slot.**  
   If offset 16 initially contains 11, this function performs no pre-call write to replace it. Whether the dependency observes or changes 11 depends on the dependency’s behavior.

10. **Abnormal dependency completion.**  
    If the dependency never returns, no final write to offset 16 or zero return from this function is established.

## Uncertainties

The following are **documented_only**:

- Semantic names and domain meaning for the record fields, categories, and shared values.
- The dependency’s behavior, including errors, record mutations, global mutations, retention, and resource ownership.
- Whether callers initialize offset 16 for use by the dependency.
- Whether callers guarantee non-aliasing, alignment, thread exclusion, or a particular memory lifetime.
- Target byte order and complete machine-level calling convention.
- Behavior when the record is null, undersized, inaccessible, or invalidated during delegation.
- Any intended relationship between the recognized classification inputs and physical sizes, capacities, or formats.

An inferred intent is that argument 8 supports a default configuration value and that the category adapts argument 6 for the dependency. These interpretations are not additional requirements.

## Evidence

The supplied static representation establishes the zero-triggered default selection, exhaustive unsigned classification, five ordered pre-call writes, dependency argument order, post-call result write, and constant zero normal return.

It also establishes the absence of visible use of argument 3 and the absence of a direct pre-call write to offset 16.

No implementation beyond the supplied function and dependency identity was available. No runtime tests have been performed. Acceptance examples are deductions from the supplied static behavior, not observed execution results.
