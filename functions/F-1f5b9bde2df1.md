<!-- Trusted provenance: F-1f5b9bde2df1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1f5b9bde2df1

## Purpose

F-1f5b9bde2df1 selects between two address-based record sequences, transfers selected records to a destination, delegates first-sequence remainder processing, and then directly processes the second-sequence remainder.

The directly implemented record stride is 12 bytes. For each directly processed record, the function calls F-46f0b3cb316e and subsequently transfers a four-byte field at offset 8. Its 32-bit result is the destination position maintained after remainder processing, not a record count or an explicit status code.

This structure suggests a merge operation. Sortedness, record semantics, comparison semantics, stability, and ownership behavior are not established.

Descriptions of ordinary sequence traversal require dependencies to preserve live traversal state. The function passes an invocation-local stack address to F-e46aba7df1fa; the supplied evidence does not establish whether that address aliases arguments or otherwise permits changes to positions and endpoints. The operational contract below therefore distinguishes actual state-reading order from outcomes conditional on preservation.

## Interface

The supplied representation exposes five 32-bit arguments and a 32-bit result.

| Argument | Observed role |
|---|---|
| First position | Current address-like position in the first sequence. |
| First endpoint | Equality sentinel used with the first position. |
| Second position | Current address-like position in the second sequence. |
| Second endpoint | Equality sentinel used with the second position. |
| Destination position | Address-like destination used during selection and supplied to the first-remainder dependency. |

These are operational roles, not established semantic types. In particular, endpoints are compared for equality; they are not treated as lengths or ordered bounds.

After selection, F-fae1bf9a30ba returns a 32-bit value that becomes the output cursor for second-remainder processing. The final value of that cursor is returned. It need not equal the original destination plus a locally predictable number of bytes.

### Record layout

| Offset | Directly established behavior |
|---|---|
| 0–7 | No explicit field access by this function. Dependencies receive record base addresses and may access these bytes or other memory. |
| 8–11 | Four-byte source read and destination write during direct transfers. |
| 12 | Displacement used to advance to the next record. |

The four-byte field has no established numerical, pointer, ownership, or encoding interpretation. The function applies no explicit conversion to its value.

### Preconditions and unchecked conditions

For valid ordinary traversal:

- Positions and endpoints must describe ranges terminating under repeated 12-byte advancement.
- Source fields and destination fields must be accessible when actually read or written.
- Storage must satisfy dependency access requirements and any applicable alignment requirements.
- Destination capacity must cover all writes, including those made by dependencies.
- Address calculations must remain valid in the target execution environment.
- Dependencies must return normally for execution to reach subsequent operations.

For the conventional interpretation that each selected record consumes exactly one input element, dependencies must also preserve the relevant live positions, endpoints, destination state, and retained addressing state.

These are conditions on valid use, not checks performed by this function. It does not validate capacities, endpoint ordering, alignment, overlap, ownership, or dependency results.

## Behavior

### Live state and call boundaries

A position supplied to a dependency is the value used at that call. Later operations are not necessarily based on a saved copy of that value.

This distinction matters in the selection phase:

- The positions supplied to F-e46aba7df1fa are obtained before that call.
- The selected source position supplied to F-46f0b3cb316e is obtained after the selection dependency returns.
- The source field address is calculated after F-46f0b3cb316e returns.

If either dependency can modify accessible invocation state, these positions can differ. Normal return alone does not establish preservation. Nonoverlapping record buffers also do not establish preservation of stack-resident traversal state.

The stack address passed to F-e46aba7df1fa has no established object type, extent, initial contents, or alias relationship. No guarantee is made that it is read-only context or independent of the five argument slots.

### Selection eligibility and decision

Selection continues while the first position differs from its endpoint and the second position differs from its endpoint. The first inequality is checked before the second; the second need not be evaluated when the first fails.

When both checks pass, F-e46aba7df1fa is called with:

1. An address of invocation-local stack storage.
2. The current second position.
3. The current first position.

Its represented result is 32 bits wide:

- Zero selects the first-input branch.
- Any nonzero value selects the second-input branch.

Positive and negative nonzero values are not distinguished. There is no separately recognized error result.

The endpoint conditions are not rechecked between this call and the selected transfer. Thus, if the selection dependency changes a live position or endpoint, the branch still proceeds according to the returned zero/nonzero result, using the state subsequently read by that branch. Even a position changed to equal its endpoint is not locally rejected before that transfer.

### Selection transfer ordering

For either selected branch, the established order is:

1. Call F-46f0b3cb316e with the live destination position followed by the live selected-input position.
2. After that call returns, calculate and retain the selected source field address: the then-current selected position plus 8.
3. Advance the selected input position by 12.
4. Read four bytes through the retained field address.
5. Write that value to the destination field at offset 8.
6. Advance the destination position by 12.
7. Return to the selection eligibility checks.

The selected-input advancement therefore occurs **before the explicit source-field read**, not merely before the destination write. The read uses the retained field address rather than an address recalculated from the advanced position.

The function does not explicitly advance the nonselected input during this branch. That is not a guarantee that a dependency leaves it unchanged.

If traversal state is preserved by dependencies and does not alias the accessed fields, the retained address identifies the field of the record just supplied to F-46f0b3cb316e. Without those conditions, the post-call field address can identify a different location. Moreover, if the selected-position update aliases the retained source field, the update itself can affect the value subsequently read.

The source field is not snapshotted before the transfer dependency. Subject to the addressing qualifications above, a change that dependency makes to the source field is visible to the later explicit read. Its write to the destination field can likewise be replaced by the subsequent explicit write.

The relative evaluation timing of the source read and destination-address calculation within the field assignment is not independently established. The source value must be obtained before that value is stored; the selected-input advancement precedes both.

### First-input remainder

When a selection eligibility check fails, the function calls F-fae1bf9a30ba with:

1. The then-current first position.
2. The then-current first endpoint.
3. The then-current destination position.

There is no local empty-range guard around this call. It occurs once on the normal path after selection, including when selection never runs.

The dependency’s returned 32-bit value initializes the output cursor used for the second remainder. The function neither validates that value nor replaces it with a locally calculated first-remainder endpoint.

All first-remainder transfer behavior is delegated. The supplied evidence does not establish what fields are copied, how many records are processed, whether ownership changes, or what an empty range does.

The dependency may also affect accessible state. In particular, second-remainder eligibility is checked after it returns. The second range must not be assumed unchanged merely because its positions were not explicit arguments to this call.

### Second-input remainder

The function then checks the live second position against the live second endpoint. For each successful inequality check, it:

1. Calls F-46f0b3cb316e with the output cursor followed by the second position.
2. After return, reads four bytes from offset 8 of the second position used for the subsequent field access.
3. Writes that value at offset 8 of the output cursor.
4. Advances the output cursor by 12.
5. Advances the second position by 12.
6. Checks the second position against its endpoint again.

Unlike selection-phase handling, second-position advancement occurs **after** the explicit field transfer. There is no further call to F-e46aba7df1fa.

A transfer dependency that changes accessible second-position or endpoint state can affect the field access, subsequent advancement, and next eligibility check. No endpoint recheck intervenes between its return and the explicit field transfer.

When the second-remainder check finds equality, the current output cursor is returned.

### Empty, malformed, and wrapping ranges

With dependencies preserving relevant traversal state:

- An initially empty first range prevents selection.
- An initially empty second range prevents selection.
- Two initially empty ranges still cause the first-remainder call.
- A second range still empty after that call causes no direct second-remainder transfers.

Without preservation, initial emptiness does not determine later second-remainder behavior.

Equality is the only range test. A numerically smaller endpoint does not trigger an error. An endpoint with an incompatible displacement from its position is not detected. Invalid addresses, nontermination, or wraparound are not converted to a local error result.

All represented positions and results are 32 bits. Field offset 8 and stride 12 participate in address-like arithmetic. The supplied representation does not establish portable signed-overflow behavior or valid wraparound addressing. Acceptance examples below exclude overflow and invalid accesses rather than assuming modular traversal.

## Side effects

Direct record-data writes are four-byte writes at destination offset 8. The function also updates invocation-local traversal state. Dependencies may perform additional memory accesses and state changes.

For a selection transfer, the significant ordering is:

**selection dependency → transfer dependency → retain source-field address → advance selected input → read retained source field → write destination field → advance destination.**

For a second-remainder transfer, the ordering is:

**transfer dependency → source-field read and destination-field write → advance output cursor → advance second input.**

The first-remainder dependency runs between the selection phase and the second-remainder phase.

No explicit allocation, deallocation, reference-count operation, source clearing, or global-variable access appears in this function. This does not establish their absence from dependencies. Passing source and destination addresses does not establish copy, move, borrow, retain, or release semantics.

Overlap is not checked. A dependency can change source storage before the explicit read, and earlier destination writes can affect later records or accessible traversal state. Safe in-place operation is not established.

There is no local rollback, recovery branch, or failure cleanup. If a dependency does not return normally or an access faults, earlier effects may remain. No transactional guarantee is established.

## Dependencies

| Opaque function ID | Observed interface and use | Unresolved boundary |
|---|---|---|
| F-e46aba7df1fa | Receives a stack-storage address, second position, and first position. Its 32-bit zero/nonzero result chooses a branch. | Comparison meaning; stack-object extent and aliasing; permitted mutation; globals; failure behavior. |
| F-46f0b3cb316e | Receives destination then source for every directly transferred record. Its result is unused. | Actual transfer semantics; accessed extent; state mutation; allocation, release, and ownership behavior. |
| F-fae1bf9a30ba | Receives first position, first endpoint, and destination once after selection. Its 32-bit result becomes the output cursor. | Remainder processing, empty-range behavior, state preservation, ownership effects, and validity of the returned position. |

Call-count statements apply to execution reaching those points. Dependency-internal calls are outside this contract.

## Acceptance examples

These are conditional obligations derived from static evidence, not runtime results.

### One record in each input; zero selection result

Assume valid nonoverlapping record storage, valid arithmetic, normal dependency returns, and preservation of all relevant live traversal state and addressing state. Each endpoint is exactly 12 bytes beyond its initial position.

If F-e46aba7df1fa returns zero:

- Its input-position arguments are ordered second, then first.
- F-46f0b3cb316e receives the initial destination and first record.
- The first source field address is retained after that call.
- The first position advances to its endpoint before the field is read.
- The field value is then written to the initial destination, which advances by 12.
- Selection exits because the first range is exhausted.
- F-fae1bf9a30ba receives an empty first range and the advanced destination.
- The second record is transferred at the position returned by that dependency.
- The final result is that returned position plus 12.

The second record need not be placed at the original destination plus 12: the empty-range dependency result is not constrained to equal its destination argument.

### One record in each input; nonzero selection result

Under the same preservation and validity conditions, any nonzero selection result causes the second record to be transferred first.

The second position advances to its endpoint before that record’s explicit source-field read. After the destination advances, selection ends. F-fae1bf9a30ba receives the still-unconsumed first range.

Because the second range remains exhausted under the stated preservation condition, no direct second-remainder transfers occur. The function returns F-fae1bf9a30ba’s result unchanged.

### Both inputs initially empty

Initial equality of both ranges prevents any selection call. F-fae1bf9a30ba is still called.

If that dependency preserves second-position and second-endpoint equality, no direct transfer calls occur and its result is returned unchanged.

If it changes accessible second-range state so those positions differ, second-remainder processing can occur. Initial emptiness alone does not exclude that outcome.

### Selection dependency changes traversal state

Suppose an eligibility check passes, but F-e46aba7df1fa changes accessible first-position storage and returns zero.

The function does not recheck the first endpoint before the selected transfer. F-46f0b3cb316e receives the first position subsequently read by the branch, not necessarily the position supplied to the selection dependency.

Whether this mutation is possible through the supplied stack address is documented_only. If it is possible, the stated operational consequence follows from the absence of an intervening endpoint check.

### Transfer dependency changes the source field

Assume stable traversal state, no alias between traversal state and record fields, and otherwise valid accesses. If F-46f0b3cb316e changes the selected source’s offset-8 value, the later explicit read observes the changed value.

In the selection phase, the selected position advances before this read, but the retained field address continues to identify that source field.

### Equivalent records

If an external comparison contract establishes that equivalent records produce zero, and dependencies preserve traversal state, this function chooses the first input for that comparison.

This does not independently prove stable merging. Input ordering, comparison semantics, and first-remainder behavior remain necessary external conditions.

## Uncertainties

The following remain documented_only:

- Record and field semantic types.
- The stack object passed to F-e46aba7df1fa, including its initial contents, extent, and alias relationship to argument storage.
- Which live positions, endpoints, or other invocation state dependencies can modify.
- Comparison relation, sorted-input requirements, and stability.
- Copying, moving, retaining, releasing, or other ownership effects.
- First-remainder processing and empty-range behavior.
- Safe overlap configurations.
- Target alignment and address-validity requirements.
- Dependency effects on globals, shared memory, and concurrency.
- Fault, exception, nonlocal-exit, and recovery behavior.
- Arithmetic behavior outside valid nonoverflowing address calculations.

The merge interpretation is inferred intent. Dependency order, zero/nonzero branching, explicit field width and offset, differing advancement order between phases, unconditional first-remainder delegation, and return-value flow are established static observations.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent behavioral evidence.

The representation establishes that selection retains the post-transfer source-field address, advances the selected position, and only then reads the retained field. It also establishes that remainder eligibility is evaluated after the first-remainder dependency returns. These facts require preservation conditions on ordinary traversal examples rather than unconditional claims about exhaustion.

Dependency implementations, independent interface documentation, target arithmetic specifications, and runtime observations were not supplied. No runtime tests have been performed.
