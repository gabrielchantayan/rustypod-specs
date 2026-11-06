<!-- Trusted provenance: F-84aceaa3f40b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-84aceaa3f40b

## Purpose

F-84aceaa3f40b coordinates an operation using two caller-supplied context words and a caller-supplied output address. It requests initial storage, initializes metadata, alternates polling with state advancement, and conditionally requests two additional storage values for final processing. It publishes the second additional storage value before calling final processing.

The established behavior is orchestration: call ordering, explicit reads and writes, branching, and status selection. “Storage,” “initialization,” “advancement,” “processing,” and “release” describe inferred dependency roles, not established application semantics.

A zero return does not independently establish that the published value identifies valid storage or usable output. Neither the processing contract nor the output’s ownership and layout is supplied.

## Interface

### Arguments

The function accepts:

1. **First context word:** a 32-bit value forwarded to initialization, polling, advancement, and final processing.
2. **Second context word:** a 32-bit value forwarded to initialization and advancement.
3. **Output address:** a location to which the function explicitly writes one 32-bit value whenever it reaches final processing.

The function does not directly dereference either context word. Their interpretation and permitted values are unknown.

The output address receives no explicit null, alignment, accessibility, or overlap check. It must permit a four-byte write if publication is reached. The function does not explicitly read its previous contents.

No non-aliasing guarantee is supplied. In particular, the contract cannot assume that a publication leaves other observable storage unaffected, or that later dependency activity leaves the output word unchanged.

### Return value

The result is a 32-bit status:

| Condition | Result |
|---|---|
| Initial storage request returns zero | 1 |
| Initialization returns nonzero | That status |
| Advancement returns nonzero | That status |
| Final processing returns nonzero | That status |
| Common cleanup finishes with current status zero and state equal to 1 | 0 |
| Common cleanup finishes with current status zero and state unequal to 1 | 7 |

The state used for the last two cases is examined **after** the initial storage value has been passed to the release dependency.

The polling result is not assigned to the current status. A nonzero polling result therefore does not become the return value merely because it caused termination.

Numeric results do not identify their origin. A dependency can independently return 1 or 7.

### Local data and widths

The visible operation uses:

- A 32-bit state slot, explicitly initialized to zero.
- Two 32-bit metadata slots.
- A 32-bit length slot.
- Two separate four-byte temporary regions.
- A first storage request of 512 bytes.
- 32-bit storage values returned by the storage-request dependency.

The metadata slots, length slot, and temporary regions have no explicit initialization before their first receiving calls. No check verifies that a dependency writes required outputs.

The supplied representation uses 32-bit integer length and status values. It does not supply a complete external size contract or establish valid allocation sizes.

## Behavior

### Initial storage and initialization

The first dependency call is F-d33f6d597e5d with size 512.

If it returns zero, F-84aceaa3f40b returns 1 immediately. No initialization, polling, advancement, processing, or release call follows. No explicit output write occurs.

Otherwise, the returned value is retained as the initial workspace value. The function writes zero to the state slot and calls F-4008d9582460 with these arguments, in order:

1. First context word.
2. Second context word.
3. Address of the initialization temporary region.
4. Address of the first metadata slot.
5. Address of the second metadata slot.

The returned status becomes the current status.

A nonzero initialization status leads directly to common cleanup. A zero status permits entry into polling. Metadata validity is not checked.

### Polling and advancement

Every entry into the inner phase begins with F-e27353dd9a60, receiving the first context word.

The result is handled as follows:

- **Nonzero polling result:** proceed directly to common cleanup without assigning that result to the current status.
- **Zero polling result:** read the state slot. A nonzero state leads to common cleanup; a zero state permits advancement.

Thus, a nonzero state does not prevent the polling call. When polling itself returns nonzero, the poll-exit decision does not require a state read; common cleanup may still examine state later.

When advancement is permitted, F-fd20b5807437 receives:

1. First context word.
2. Second context word.
3. The first metadata word.
4. Address of the length slot.
5. Address of the advancement temporary region.
6. Initial workspace value.
7. Address of the state slot.

The first metadata word is read for this call after the preceding polling call has returned. It is not a permanently captured initialization result. A dependency that retained its address could have changed it before this read.

The advancement result replaces the current status:

- A nonzero status immediately selects common cleanup, regardless of state.
- A zero status is followed by a state examination.
- State 1 selects the processing-storage stage.
- Any other state repeats the inner phase, starting with another poll.

Consequently, successful advancement leaving state zero can repeat indefinitely. There is no explicit timeout, retry limit, or progress counter.

Successful advancement leaving a state other than zero or one does not immediately terminate. Another poll occurs first. If the state remains nonzero when a subsequent zero poll result permits its examination, cleanup follows. If the poll is nonzero, cleanup follows without that intermediate state examination.

### Processing-storage requests and publication

Once a successful advancement is followed by a state read equal to 1, the function performs this sequence:

1. Reads the length slot and calls F-d33f6d597e5d with that value.
2. After the first request returns, reads the length slot again and calls F-d33f6d597e5d with that newly read value plus four.
3. Writes the second returned storage value to the output address.
4. Reads the length and both metadata slots for final processing.
5. Calls F-924f4530474b.
6. After final processing returns, passes the first processing-storage value to F-5e3309c9d083.

These are separate reads across dependency calls, not a shared snapshot.

In particular:

- The second request need not use the first request’s size plus four.
- Final processing need not receive the length used for either request.
- Final processing need not receive the metadata values originally written by initialization or previously passed to advancement.

Initialization receives the metadata addresses, and advancement receives the length address. Their retention rules are unknown. Other dependencies may therefore have access to those slots through retained addresses or shared state.

Both storage requests occur before publication. Publication occurs before the reads supplying the length and metadata arguments to final processing. Consequently, permitted aliasing involving the output write can also affect those later reads.

The evidence establishes this ordering across calls and publication, but does not establish an order among the individual length and metadata argument reads within preparation of the final-processing call.

Neither processing-storage result is checked for zero. The second request is still made if the first returns zero. The second result is written to the output address even if it is zero.

The second request’s size expression uses a 32-bit length plus four. There is no visible overflow, negativity, maximum-size, or capacity check. A portable result for signed overflow is not established; neither mathematical widening nor a particular wraparound policy is promised.

### Final processing and continuation

F-924f4530474b receives:

1. First context word.
2. Length read after both storage requests and publication.
3. Second metadata word read after those events.
4. First metadata word read after those events.
5. First processing-storage value.
6. Second processing-storage value.

Its result becomes the current status. The function then calls F-5e3309c9d083 with the first processing-storage value, including when that value is zero or final processing returned an error.

A nonzero processing status leads to common cleanup without another poll.

A zero processing status re-enters the polling phase. There is no explicit state reset. The state may nevertheless have changed through retained addresses during storage requests, final processing, or release.

If those calls preserve state at 1, the next poll is followed by cleanup: a nonzero poll selects cleanup directly; a zero poll selects cleanup after reading the nonzero state. If state has instead become zero, a zero poll permits further advancement and possibly another processing stage.

The function does not revalidate state between selecting the processing-storage stage and calling final processing.

### Common cleanup

Every ordinary return after a successful initial storage request passes the original workspace value to F-5e3309c9d083 once.

After that release call returns:

- A nonzero current status is returned unchanged.
- A zero current status causes the state slot to be examined.
- State 1 yields zero.
- Any other state yields 7.

The final state read follows the release call. No dependency contract establishes that release preserves state accessible through retained addresses.

Nontermination or abnormal dependency termination can prevent cleanup and return.

## Side effects

### Explicit writes and read ordering

The function explicitly initializes state to zero and publishes one four-byte storage value on each reached processing stage. It does not initially clear the output address or explicitly clear it on error.

The following read boundaries are significant:

- State is examined after a zero poll result.
- The first metadata word is read for each permitted advancement call.
- State is examined after a successful advancement.
- Length is read separately for each processing-storage request.
- Length and both metadata words are read after publication for final processing.
- State is examined after workspace release when current status is zero.

These boundaries do not imply preservation between reads.

Dependencies receive writable addresses for state, metadata, length, and temporary storage. Their actual write extents and retention behavior are unknown. A four-byte temporary region does not itself establish that the receiving dependency respects that bound.

### Storage and ownership

The visible lifecycle is:

| Storage role | Visible handling |
|---|---|
| Initial workspace | Requested once; a nonzero result is passed to release on every subsequent ordinary return path |
| First processing storage | Requested at each processing stage; passed to release after final processing returns |
| Second processing storage | Requested at each processing stage; published before final processing; never explicitly passed to release by this function |

This establishes calls, not ownership semantics. It does not prove that request results are allocations, that release accepts zero, or that final processing neither retains nor releases storage.

A previous output value is overwritten without explicit disposition. If processing occurs repeatedly, each publication replaces the output word without an explicit release of an earlier published value.

After publication, this function performs no further explicit output write on a processing-error path. That does **not** establish persistence of the published value: processing, release, shared-state effects, or aliases may modify the location.

### Invalid inputs and failures

No recovery behavior is established for an invalid output address, invalid storage values, unwritten required slots, dependency memory corruption, unsupported sizes, or dependencies that do not return normally.

There is no supplied output layout beyond publication of one 32-bit value. The extra four requested bytes do not establish a header, trailer, terminator, or alignment scheme.

## Dependencies

| Opaque ID | Visible contract boundary |
|---|---|
| **F-d33f6d597e5d** | Receives a size and returns a 32-bit storage value. The initial request uses 512. Later requests use separate length reads, with four added for the second. Only the initial result is checked for zero. |
| **F-4008d9582460** | Receives both contexts, a temporary-region address, and two metadata addresses. Returns initialization status. |
| **F-e27353dd9a60** | Receives the first context. A nonzero result selects cleanup without becoming current status. |
| **F-fd20b5807437** | Receives both contexts, the currently read first metadata word, writable length and temporary-region addresses, workspace, and writable state. Returns advancement status. |
| **F-924f4530474b** | Receives the first context, freshly read length and metadata values, and both processing-storage values. Returns processing status. |
| **F-5e3309c9d083** | Receives storage values at the release points. Its return does not supply the function’s status. Ownership and other effects are unknown. |

No supplied dependency contract guarantees global preservation, absence of retained pointers, exclusive ownership, reentrancy, or thread safety.

## Acceptance examples

These are conditional consequences of static control flow, not executed tests. Each example assumes normal dependency returns, valid accessed storage, and no unmentioned aliasing or retained-address mutations.

1. **Initial request failure.**  
   The request for 512 returns zero. Result: 1. No subsequent dependency call or explicit output write occurs.

2. **Initialization failure.**  
   The initial request succeeds and initialization returns 12. The next dependency call releases the workspace. Result: 12. This function performs no output write.

3. **Polling stops pending work.**  
   Initialization returns zero. The first poll returns 23, and state remains zero through cleanup. Advancement is not called. Result: 7, not 23.

4. **Pending advancement becomes ready.**  
   Polls return zero. One advancement returns zero with state zero; a later advancement returns zero with state 1. Processing-storage requests occur only after the latter. Each advancement receives the first metadata word read for that particular call.

5. **Unexpected state.**  
   Advancement returns zero with state 2. Another poll occurs. If state remains 2 through final selection, cleanup returns 7. No processing-storage request occurs.

6. **Advancement error overrides ready state.**  
   Advancement writes state 1 but returns 9. The workspace is released and 9 is returned. No processing storage, publication, or final processing occurs.

7. **Preserved values on a successful path.**  
   Advancement returns zero with state 1 and length 20. Assume all subsequent dependencies preserve length, metadata, state, and the output word except for the specified publication. Requests use 20 and 24. Final processing receives length 20 and the preserved metadata. After processing returns zero, the first processing-storage value is released, another poll occurs, and the workspace is released. Result: zero.

8. **Length changes between requests.**  
   Advancement leaves state 1 and length 20. The first processing-storage request receives 20 and changes the retained length slot to 40 before returning. The second request receives 44. If that request changes length to 60, and publication does not affect it, final processing receives 60—not 20 or 40.

9. **Metadata changes before processing.**  
   A dependency changes either retained metadata slot during a processing-storage request. With no later change, final processing receives that updated word. It need not receive the initialization-time value.

10. **Processing failure after publication.**  
    Final processing returns 6. The function releases the first processing-storage value and then the workspace, and returns 6. It performs no subsequent explicit output write and no explicit release of the published value. Persistence of that value requires dependencies and aliases not to modify the output location.

11. **Unchecked processing-storage failure.**  
    Either processing-storage request returns zero. That alone selects no error branch. The second result is still published and final processing is still called with both results. Further behavior depends on dependency contracts.

## Uncertainties

The following remain **documented_only**:

- Application meanings of contexts, metadata, temporary regions, states, and output.
- Named meanings, if any, of statuses 1 and 7.
- Storage-request size signedness, accepted range, layout, and failure conventions beyond the initial zero check.
- Arithmetic behavior when length plus four exceeds the signed 32-bit range.
- Whether dependencies retain addresses or mutate retained slots during later calls.
- Relative ordering among final-processing argument reads.
- Output validity, initialized extent, ownership, and usability after errors.
- Whether dependencies retain, release, or otherwise consume passed storage.
- Consequences of aliasing and dependency writes outside intended regions.
- Behavior when required outputs remain unwritten.
- Concurrency, reentrancy, and abnormal termination guarantees.

## Evidence

Established statements derive from static inspection of the supplied function representation and opaque dependency mapping. Dependency implementations and external interface contracts were not supplied.

Static evidence establishes explicit initialization and publication, separate length reads across storage requests, final argument reads after publication, branch conditions, call ordering, unchecked processing-storage results, release placement, and final status selection.

Application-level role names are inferred intent only. Preservation, ownership, and output validity are not established by those names.

No runtime tests have been performed. Acceptance examples are conditional static cases, not observed executions.
