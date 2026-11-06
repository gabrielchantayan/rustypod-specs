<!-- Trusted provenance: F-fb5bc2dc656c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fb5bc2dc656c

## Purpose

F-fb5bc2dc656c traverses a counted collection, calls a supplied callback for each entry observed to be nonzero, and then performs a separate sequence of conditional and indirect calls.

For a nonnull collection descriptor, the post-traversal sequence is:

1. Call F-5990a26eaaa3 if the descriptor’s storage field is nonzero when checked.
2. Read a global table reference and retain the resulting address.
3. Independently read that global reference again and use this second observed address to select an optional target at offset 0x30.
4. Invoke the required target at offset 0x18 of the retained table.
5. Independently select and optionally invoke the target currently at offset 0x30 of the retained table.

No dependency call intervenes between the two initial global-reference reads. Nevertheless, equality of their observed values requires a stability assumption not established by the supplied evidence.

The sequence suggests collection cleanup, but neither destruction nor memory release is established. The function’s own body reads collection data and invokes dependencies; the dependencies’ ownership and lifetime effects remain unknown.

## Interface

### Inputs and representation

The recovered interface has four explicit inputs:

| Input | Recovered representation | Established use |
|---|---|---|
| Collection descriptor | 32-bit pointer | Null-tested; otherwise used to read the count and storage address; explicitly passed to two possible cleanup calls |
| Entry callback | Executable target in the supplied 32-bit representation | Invoked when a visited entry is nonzero |
| Third input | 32-bit word | No explicit use is visible |
| Fourth input | 32-bit word | Appears in the recovered argument list of the first optional table call |

The descriptor has this recovered layout:

| Byte offset | Width | Interpretation |
|---|---:|---|
| 0x00 | 32 bits | Signed traversal bound |
| 0x04 | 32 bits | Entry-storage address |

Each entry occupies four bytes and is read as a 32-bit value. Only its equality or inequality to zero is used. The function does not explicitly dereference an entry value or pass it to the callback.

The table references and executable-target fields belong to the supplied 32-bit representation. The relevant table offsets are byte offsets: 0x18 and 0x30. This description does not establish a portable layout for another pointer width.

### Output

No return value or status result is established. The function returns immediately for a null descriptor, or after the reached cleanup operations complete normally.

Dependency return values are not explicitly consumed. There is no explicit aggregation of callback results.

### Preconditions for normal completion

A null descriptor is supported without further validity requirements on the other inputs.

For a nonnull descriptor, every reached memory access and executable target must be valid at the time it is used:

- The descriptor’s fields must remain readable through their last reached read.
- Each reached entry address must support a four-byte read.
- The entry callback must be callable whenever an observed entry is nonzero.
- The global-reference location must support both initial reads.
- The address observed by the second global-reference read must support the initial offset-0x30 target read.
- The retained address must support the later offset-0x18 and offset-0x30 reads, if reached.
- The target selected at offset 0x18 must be callable; it is not null-checked.
- Each nonzero target selected at offset 0x30 must be callable with the execution state supplied at that particular call.

These are preconditions, not validation performed by the function. A zero table reference is not treated as an empty table.

The descriptor itself need not be writable for the body’s explicit operations. Dependencies may impose additional requirements that are not supplied.

## Behavior

### Null descriptor

If the descriptor is null, the function returns without:

- Reading descriptor fields or entry storage.
- Invoking the entry callback.
- Calling F-5990a26eaaa3.
- Reading the global table reference or any table field.

The other explicit inputs do not affect this path.

### Traversal and live reads

Traversal starts at index zero. Before every potential visit, including the check that ends traversal, the function reads the descriptor’s signed count and compares the current index against it.

When the index is less than that observed count, the function:

1. Reads the storage address from descriptor offset 0x04.
2. Reads the four-byte entry at that address plus four times the current index.
3. Invokes the supplied callback if and only if the observed entry is nonzero.
4. Advances the index by one after the entry processing completes normally.

The callback target is the second explicit input. Entry values do not select executable targets.

The recovered callback invocation has no explicit arguments. In particular, no explicit descriptor, entry address, entry value, or index is passed. This does not establish the callback’s complete machine-level calling convention or exclude observations through other execution state.

A null callback is not checked. It causes no attempted call only if traversal never observes a nonzero entry.

An initial count of zero or any negative signed count causes no entry reads and no entry callbacks. Cleanup still follows.

### Mutation during traversal

The count and storage address are not captured as immutable metadata:

- A returning callback that reduces the count can stop subsequent visits.
- A returning callback that increases the count can extend traversal.
- Replacing the storage address changes the base used for the next reached entry read.
- Replacing storage does not reset the index.
- Mutating a later entry can change whether its eventual visit invokes the callback, even if count and storage address remain unchanged.

The function does not explicitly reread the current entry after its callback returns. It also does not clear entries, compact storage, decrement the descriptor count, or record visitation in collection memory.

The ordering of reads is established. Safe concurrent mutation, atomicity, and inter-thread visibility are not established.

### Arithmetic boundaries

The count and traversal index are recovered as signed 32-bit integers. Entry displacement is four times the index, and the storage address is represented by a 32-bit field.

For each reached body iteration, the index is nonnegative and below the observed signed count. Because a signed 32-bit count cannot exceed 2,147,483,647, the greatest index admitted into a body iteration is 2,147,483,646. Incrementing that index once does not itself exceed the signed 32-bit maximum.

This does not establish valid entry-address arithmetic. At index 536,870,912, the mathematical byte displacement is 2,147,483,648, outside the positive signed 32-bit range. Smaller displacements may also produce an invalid address when added to a particular storage base.

The supplied evidence does not settle overflow, wrapping, or fault behavior for unsupported displacement or address calculations. Normal-completion requirements therefore include valid arithmetic and readable resulting addresses, not merely a nonnegative count.

### Conditional helper

After traversal terminates, the function reads the storage field again.

If that newly observed field is nonzero, it calls F-5990a26eaaa3. Otherwise, it skips the helper. This decision uses the cleanup-time field value, not an earlier storage address.

Consequently, a nonzero storage field triggers the helper even when the count is zero or negative. A callback that changes the storage field can alter whether the helper is reached.

No explicit arguments are recovered for this call. In particular, the source does not establish that the storage address is passed to the helper. Release behavior, descriptor mutation, global mutation, and return behavior are unknown.

### Two initial global-reference reads

After the helper returns, or immediately if it is skipped, the function performs two distinct global-reference reads:

1. The first observed address is retained for the later required call and final optional call.
2. The second observed address is used to read the initial optional target at byte offset 0x30.

No dependency call intervenes between these reads. They are nevertheless separate reads in the supplied body.

Changes made by entry callbacks or F-5990a26eaaa3 before these reads can affect table selection. If the global reference remains stable across both reads, both observed addresses identify the same table. Without that stability assumption, the first optional target need not come from the retained table.

After these reads, later table accesses use the retained address rather than rereading the global reference.

### First optional table call

The function reads the target at offset 0x30 of the address obtained by the second global-reference read.

A zero target skips this call. A nonzero target is invoked with eight recovered argument positions:

| Position | Recovered value |
|---|---|
| 1 | Descriptor pointer |
| 2 | Zero-valued word |
| 3 | The selected executable target itself |
| 4 | Fourth explicit input |
| 5–7 | Three unrecovered execution-state words |
| 8 | An unrecovered word associated with return-link state |

The recovered word representation is 32 bits. The last four positions do not have established application-level meanings. This argument list is not sufficient evidence for a complete public ABI.

After normal return, execution continues using the address retained from the first global-reference read. The first optional call may mutate that table even when its own target came from a different table; no preservation guarantee is supplied.

### Required table call

The function next reads the target at offset 0x18 of the retained table and invokes it with the descriptor pointer as the sole explicitly recovered argument.

This target read occurs after the first optional call, if that call occurs. A mutation of the retained table’s offset-0x18 field by that dependency can therefore change the required target used in this invocation.

There is no null check, fallback, or explicit error return for an unusable required target.

### Final optional table call

After the required call returns normally, the function reads offset 0x30 of the retained table.

If this newly observed target is zero, the function returns. Otherwise, it invokes the target with two explicitly recovered word arguments: zero followed by one.

This selection is independent of the initial optional-target selection:

- The two target fields may belong to different tables if the two initial global-reference reads differed.
- Even when they belong to the same table, intervening dependencies may change the slot.
- Skipping the first optional call does not imply that the final call is skipped.
- Performing the first optional call does not imply that the final call occurs.

The exact final machine-level transfer form is unresolved: the evidence does not conclusively distinguish an ordinary call followed by return from a tail transfer.

All later operations are conditional on earlier dependencies returning normally. A fault, nonreturning dependency, or nonlocal transfer can prevent the remaining sequence.

## Side effects

The function contains no explicit writes to the descriptor, entry storage, global table reference, or table fields. Its directly established externally relevant operations are the ordered reads and dependency invocations described above.

Dependency effects remain unconstrained by the supplied body. In particular:

- Nonzero entries are tested, not explicitly released.
- The entry callback receives no explicitly recovered entry argument.
- The helper receives no explicitly recovered storage argument.
- Passing the descriptor to a dependency does not establish ownership transfer.
- No descriptor reset or entry clearing is performed explicitly.
- Successful return does not prove that any memory was freed, retained, or remains usable.

Callees must not be assumed to preserve globals, table contents, collection metadata, entry contents, or memory lifetime. They must also not be assumed to invalidate them. Their actual effects must leave every later reached access valid for normal completion.

Replacing the global table reference after the two initial reads does not by itself redirect later accesses. Invalidating the retained table can still make those accesses invalid.

No explicit synchronization, rollback, fault recovery, allocation, or allocation-failure handling is present.

## Dependencies

### F-5990a26eaaa3

Called at most once, after traversal, if the storage field is nonzero when checked.

Its arguments, outputs, ownership rules, global effects, and failure behavior are **documented_only** unknowns. Its placement before both initial global-reference reads is established.

### Caller-selected indirect dependency

The callback’s opaque function identity is unavailable; its target is supplied by the second explicit input.

It is invoked for entries observed to be nonzero. No explicit arguments or consumed return value are recovered.

### Table-selected indirect dependencies

The selected targets have no supplied opaque function IDs.

Their selection and recovered argument shapes are distinct:

| Role | Selection source | Explicitly recovered arguments |
|---|---|---|
| First optional target | Offset 0x30 using the second global-reference observation | Eight positions described above |
| Required target | Offset 0x18 using the retained first observation | Descriptor pointer |
| Final optional target | Offset 0x30 using the retained first observation, read after the required call | Zero, one |

A uniform signature, common identity, or shared ownership policy must not be inferred from the shared offset used by the optional selections.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Except where an example specifies otherwise, reached memory accesses are valid and dependencies return normally.

1. **Null descriptor.**  
   A null descriptor produces no dependency calls and no descriptor, entry, global-reference, or table reads.

2. **Stable mixed entries.**  
   With count four and entry values zero, nonzero, nonzero, zero, exactly two entry-callback invocations occur if the count and storage address remain unchanged and the values observed at subsequent reads retain that zero/nonzero pattern. Merely preserving metadata is insufficient. Cleanup follows.

3. **Entry mutation without metadata mutation.**  
   With the same initial pattern, if the callback at the second entry clears the third entry and leaves the fourth entry zero, only one entry callback occurs. Count and storage address may remain unchanged.

4. **Negative count and nonzero storage.**  
   No entries are read. The helper is called because of the cleanup-time nonzero storage field, before either global-reference read.

5. **Zero count and zero storage.**  
   Traversal and the helper are skipped. Both initial global-reference reads and the table-call sequence are still reached.

6. **Traversal shortened by a callback.**  
   If the callback for index zero changes the count to one, the next bound read ends traversal. No entry at index one is read.

7. **Storage replaced by a callback.**  
   If the callback for index zero replaces storage while the count remains greater than one, the next entry read uses the replacement base plus four bytes. Traversal does not restart.

8. **Helper changes table selection.**  
   If the helper replaces the global reference and that reference then stays stable across both initial reads, both reads observe the replacement table.

9. **Distinct initial observations.**  
   If the execution environment permits the first global-reference read to observe table A and the second to observe table B, the initial optional target is read from B at offset 0x30. The required and final target reads use A. This is conditional on such observations being permitted; it is not a concurrency guarantee.

10. **First optional call replaces the required target.**  
    If that call modifies offset 0x18 of the retained table, the subsequent required-target read observes the replacement, subject to the applicable memory-access semantics.

11. **Required call enables the final call.**  
    If the initial optional target is zero and the required call installs a nonzero target at offset 0x30 of the retained table, the first optional call is skipped and the final call occurs with explicit arguments zero and one.

12. **Global replacement after selection.**  
    If the first optional dependency replaces only the global reference, subsequent table reads still use the retained address. The replacement alone does not redirect them.

## Uncertainties

### documented_only: complete calling conventions

The full signatures of F-5990a26eaaa3 and the indirect targets are unresolved. Recovered argument positions must not be treated as verified complete ABIs. Meanings of the four execution-state words in the first optional call remain unknown.

### documented_only: ownership and cleanup intent

Teardown, destruction, notification, and deallocation are possible interpretations, not established effects. No ownership-transfer or post-return lifetime guarantee is available.

### documented_only: arithmetic outside valid ranges

The recovered widths, signed comparison, and four-byte stride are established. Defined wrapping, overflow handling, and resulting-address behavior outside valid ranges are not.

### documented_only: concurrency and access semantics

The two initial global-reference reads are distinct and have no intervening dependency call. Their equality is not guaranteed without stability. No concurrency model, atomicity guarantee, visibility rule, or safe-mutation protocol is supplied.

### documented_only: faults and final transfer

No exception, fault-recovery, or nonlocal-control-transfer contract is supplied. The exact final indirect transfer form remains unresolved.

## Evidence

**Established static evidence:** null early return; two-field descriptor layout; signed live count checks; per-visit storage reads; four-byte entry reads; nonzero-entry callback condition; cleanup-time storage check; conditional F-5990a26eaaa3 call; two separately expressed initial global-reference reads; retained first address for the required and final target reads; second observed address for the initial optional target read; independently timed target selections; and the recovered argument positions.

**Inferred intent:** the sequence resembles collection cleanup with surrounding dispatch hooks. This inference does not establish release behavior, dependency signatures, or ownership.

**Unknowns:** dependency implementations, complete ABIs, execution-state meanings, unsupported arithmetic behavior, concurrency semantics, and the final transfer form.

No runtime tests have been performed. The acceptance examples are static criteria under their stated preconditions.
