<!-- Trusted provenance: F-6c73baaa55d3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6c73baaa55d3

## Purpose

F-6c73baaa55d3 advances a state record, decreases a counter, and conditionally performs a transition involving the record’s current descriptor and an auxiliary sequence of 32-bit words.

The established behavior is a sequence of state updates and dependency calls. The apparent roles of “current descriptor,” “saved descriptor,” and “counter” are descriptive interpretations of field usage, not independently established types. In particular, the dependency calls do not establish that an object is destroyed, memory is released, or a resource is owned.

When the transition condition is false, the function performs no further direct work after the initial advancement, decrement, and predicate call. When the condition is true, it consumes one auxiliary word, invokes an action dependency, evaluates the state again, and selects one of two descriptor-replacement paths.

## Interface

### Input and state layout

The function accepts one address identifying a mutable state record. The supplied representation accesses eleven consecutive 32-bit fields, spanning byte offsets 0 through 43.

The following names describe those fields for this contract:

| Byte offset | Width | Contract name | Established use |
|---|---:|---|---|
| 0 | 32 bits | Current position | Advanced by 4; compared with the current boundary; replaced during a transition |
| 4 | 32 bits | Current descriptor word 1 | Replaced during a transition |
| 8 | 32 bits | Current boundary | Compared with the current position; replaced during a transition |
| 12 | 32 bits | Auxiliary position | Interpreted as an address of a 32-bit word; advanced by one word in the transition path |
| 16 | 32 bits | Saved descriptor word 0 | Written and then read in the fallback path |
| 20 | 32 bits | Saved descriptor word 1 | Written and then read in the fallback path |
| 24 | 32 bits | Saved descriptor word 2 | Written and then read in the fallback path |
| 28 | 32 bits | Saved descriptor word 3 | Written and then read in the fallback path |
| 32 | 32 bits | Remaining counter | Decreased by 1 before the first predicate call |
| 36 | 32 bits | Action argument 0 | Passed to the final action in the fallback path |
| 40 | 32 bits | Action argument 1 | Passed to the final action in the fallback path |

The layout establishes storage widths and offsets, not the full semantic types of the fields. Address-valued fields are represented using 32-bit storage in the supplied representation.

The record must be valid for the accesses on the executed path. Additional requirements imposed by dependencies are unknown. No explicit null-address check, record-size check, alignment check, or range check is visible.

### Output

There is no explicit return value. Observable results consist of:

- Direct modifications to the state record.
- Reads from memory addressed by the auxiliary-position field.
- Calls to dependencies, including any effects those dependencies produce.
- Possible failure to complete if a memory access, dependency, or unsupported arithmetic case prevents continuation.

No status value or explicit error result is produced.

### Temporary storage

The function provides two separate temporary regions to dependencies:

- A 16-byte region passed to F-d343b6628b56 at entry.
- A four-word, 16-byte descriptor destination passed to F-6bc2587a4cd2 on either replacement path.

The entry temporary is not subsequently read or explicitly finalized by the visible body. The descriptor destination is read as four 32-bit words after F-6bc2587a4cd2 returns.

The storage has function-local lifetime. Whether dependencies retain its address, and whether such retention is permitted, is not established.

## Behavior

### 1. Entry dependency and initial advancement

The first operation is a call to F-d343b6628b56 with the entry temporary’s address and the state-record address.

Only after that call returns does the function:

1. Read the current-position field, add 4, and write the result to that field.
2. Read the remaining-counter field, subtract 1, and write the result to that field.
3. Call F-ee993636c085 with the state-record address.

The first two updates therefore operate on the state present after F-d343b6628b56 returns. They must not be specified as updates to an immutable entry snapshot.

The additions and subtraction operate on fields represented as 32-bit integers. The supplied evidence does not independently settle signed-overflow semantics. For inputs where the operations remain within the applicable integer range, the results are respectively an increase of 4 and a decrease of 1. Boundary-crossing behavior is documented_only rather than assigned a guessed wraparound rule.

There is no visible check preventing a zero remaining counter from being decreased. There is also no visible check that the current position remains within its boundary after advancement.

### 2. Transition decision

The return value of F-ee993636c085 is tested against zero.

A transition is taken if either:

- That return value is nonzero; or
- That return value is zero and the current-position field equals the current-boundary field.

The equality comparison is short-circuited: it is not required when the predicate result is nonzero. When evaluated, it uses the fields after the predicate call, so changes made by the predicate can affect the decision.

If the predicate returns zero and the two fields are unequal, the function returns without further direct reads, writes, or dependency calls.

The comparison is equality, not a less-than or greater-than bound check. A position that has passed the boundary does not, solely for that reason, trigger this path.

### 3. Auxiliary-word consumption and first action

On the transition path, the function performs these operations in order:

1. Read the auxiliary-position field and retain that address for the next memory read.
2. Advance the auxiliary-position field by one 32-bit word, corresponding to 4 bytes.
3. Read the 32-bit word at the retained, pre-advance address.
4. Call F-6bf88c96e2f7.
5. Call F-ff1bdc376dcd with the consumed word, the result of F-6bf88c96e2f7, and a zero third argument.
6. Call F-ee993636c085 again with the state-record address.

The auxiliary-position update precedes the read from the old address. This matters for aliasing and failure behavior. If the retained address refers to the auxiliary-position field itself, the subsequent word read observes the already-updated field, absent other interference. If the read fails, the pointer-field update has already occurred in the visible sequence.

No claim is made that F-6bf88c96e2f7 is a passive accessor or that F-ff1bdc376dcd preserves state. Neither receives the record address explicitly in this sequence, but either could still affect reachable state or globals. The second predicate therefore examines the state existing after both calls.

### 4. Replacement when the second predicate returns zero

If the second call to F-ee993636c085 returns zero:

1. Read the current auxiliary-position field.
2. Read the 32-bit word at the address represented by that field.
3. Call F-6bc2587a4cd2 with the descriptor temporary’s address and the word just read.
4. Copy the temporary’s four words into record offsets 0, 4, 8, and 12, in that order.

This path does not directly advance the auxiliary position before reading this second word.

Although the second read ordinarily follows the previously consumed word, that adjacency is conditional: intervening dependencies may change the auxiliary-position field. The contract is to use its current value, not an independently retained address plus 4.

The four writes replace the entire current descriptor, including the auxiliary-position field. They use the descriptor temporary produced by the dependency. There is no final action call on this branch.

### 5. Fallback replacement when the second predicate returns nonzero

If the second call to F-ee993636c085 returns nonzero:

1. Call F-6bc2587a4cd2 with the descriptor temporary’s address and two zero-valued arguments.
2. Copy the temporary’s four words into saved-descriptor offsets 16, 20, 24, and 28, in that order.
3. Read those saved-descriptor fields and copy them into current-descriptor offsets 0, 4, 8, and 12, in that order.
4. Read the two action-argument fields at offsets 36 and 40 and call F-ff1bdc376dcd with those values and a zero third argument.

The saved descriptor is observably written before the current descriptor is replaced. This is not specified as one atomic eight-field update.

The last action uses the action-argument fields as they exist after the descriptor dependency and the direct field copies. It does not necessarily use their function-entry values.

The relative evaluation order of the two action-argument reads is not established. Both supply arguments to the same final call.

There is no third predicate call and no further visible correction of the state after that final action.

## Side effects

### Direct state changes

Subject to successful completion of preceding operations:

- The current position is increased by 4 on every path.
- The remaining counter is decreased by 1 on every path.
- The auxiliary position is advanced by 4 bytes only when the transition is taken.
- The current descriptor is replaced only when the transition is taken.
- The saved descriptor is directly replaced only when the second predicate returns nonzero.
- The action-argument fields are not directly written by this function.

These statements describe direct operations. Dependencies may introduce additional changes, including changes that overwrite or counteract these operations.

### External reads and ordering

The transition path reads one word through the old auxiliary address after storing the advanced address. The zero-result replacement path performs another indirect word read after the second predicate call.

Consequently, the function is not transactional. There is no visible rollback if an indirect read or later dependency fails. Intermediate state can differ from both the entry state and the eventual successfully completed state.

### Ownership and resource handling

The function explicitly allocates no heap storage and explicitly releases no memory. Passing a word or pair of words to F-ff1bdc376dcd does not by itself establish resource release or ownership transfer.

The ownership of the state record, indirectly referenced memory, and values returned by dependencies is undocumented. No retention or release guarantees can be assigned to dependencies from this body alone.

## Dependencies

| Opaque ID | Visible invocation | Established caller behavior |
|---|---|---|
| F-d343b6628b56 | Entry-temporary address, state-record address | Called first; its return value is not used |
| F-ee993636c085 | State-record address | Called once on every normal path and a second time after the first transition action; zero/nonzero controls branches |
| F-6bf88c96e2f7 | No explicit arguments | Called once on the transition path; its 32-bit result becomes the second argument of the first action |
| F-ff1bdc376dcd | Two 32-bit values and zero | Called once on either transition branch, plus once more on the fallback branch; return value is not used |
| F-6bc2587a4cd2 | Descriptor destination plus branch-dependent arguments | Called once on the transition path; four destination words are subsequently consumed |

F-6bc2587a4cd2 appears with different visible argument counts: destination plus one word on the zero-result path, and destination plus two zeros on the fallback path. Its true signature and the significance of that discrepancy are documented_only. The contract preserves the observed distinction rather than inventing an omitted argument or treating the calls as equivalent.

## Acceptance examples

These examples are conditional expectations derived from the supplied static evidence, not runtime test results. Unless stated otherwise, assume valid memory, representable arithmetic, dependencies that return normally, and no dependency mutation beyond the specifically described descriptor output.

### No transition

Given current position 100, current boundary 108, remaining counter 3, and a first predicate result of zero:

- The position becomes 104.
- The counter becomes 2.
- The inequality 104 versus 108 prevents transition.
- No auxiliary word is read, and neither action nor descriptor-replacement dependency is called.

### Equality-triggered transition

Given current position 100, current boundary 104, and a first predicate result of zero:

- The position becomes 104 before the predicate decision.
- Equality triggers the transition.
- The auxiliary position is advanced before the old-address word is read.
- The second predicate, not the equality comparison, determines which replacement path follows.

### Predicate-triggered transition without equality

If the first predicate returns nonzero, transition occurs even when the position and boundary differ. The equality comparison is skipped.

### Position beyond boundary

Given current position 104, boundary 104, and a first predicate result of zero, advancement produces 108. With no intervening mutation, the fields are unequal and no transition occurs. Equality at entry does not suffice.

### Zero-result replacement

Suppose the auxiliary position initially identifies address 4096. The word at 4096 is 17, the word at 4100 is 23, and F-6bf88c96e2f7 returns 9.

If transition occurs and the second predicate returns zero:

- The auxiliary position is first stored as 4100.
- The first action receives 17, 9, and 0.
- F-6bc2587a4cd2 receives its destination and 23.
- If that dependency supplies descriptor words 200, 201, 220, and 5000, those values replace record offsets 0, 4, 8, and 12 respectively.
- No final action using offsets 36 and 40 occurs.

### Nonzero-result fallback

If the second predicate returns nonzero and F-6bc2587a4cd2 supplies four zero words:

- Saved-descriptor offsets 16 through 28 are written with zeros first.
- Current-descriptor offsets 0 through 12 are then written from those saved fields.
- A final action receives the current values at offsets 36 and 40, followed by zero.

The all-zero descriptor in this example is a stipulated dependency output, not a guaranteed consequence of passing zero arguments.

## Uncertainties

- **documented_only — Dependency semantics:** Validation, allocation, cleanup, ownership, exceptions, nonlocal exits, and global effects are not established.
- **documented_only — Arithmetic boundaries:** Signedness and overflow behavior are not sufficiently established to specify boundary-crossing results.
- **documented_only — Pointer validity:** The extent, alignment, and provenance of indirectly addressed memory are unknown.
- **documented_only — Descriptor completeness:** Correct continuation requires usable values in the four temporary words subsequently read; the dependency’s write guarantees are unavailable.
- **documented_only — Signature discrepancy:** The two visible invocation shapes of F-6bc2587a4cd2 cannot be reconciled from this body alone.
- **documented_only — Concurrency:** No synchronization or atomicity guarantee is visible.
- **Inferred intent:** The field pattern suggests advancement within a descriptor followed by descriptor rollover or reset. That interpretation does not establish container type, element size, or resource policy.

## Evidence

The supplied static representation establishes field accesses, 32-bit storage layout, explicit update order, branch conditions, dependency invocation order, and the absence of an explicit return value.

The field names and transition terminology in this contract are explanatory labels, not recovered type declarations. Dependency internals and runtime behavior are unavailable. No runtime tests have been performed.
