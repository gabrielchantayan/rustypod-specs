<!-- Trusted provenance: F-6b96c9e5bbe4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6b96c9e5bbe4

## Purpose

F-6b96c9e5bbe4 copies a four-word descriptor into receiver state, invokes two receiver-dispatched operations, determines a signed 32-bit selection, and supplies that selection to two further receiver-dispatched operations.

The default selection comes from descriptor word 2. A nonzero descriptor word 1 enables a search over receiver-defined indices. The first matching index replaces the default selection; a nonnegative descriptor word 3 then supplies an additional adjustment.

These are established structural roles, not domain interpretations. Whether the descriptor represents a position, identifier, navigation request, or another object is **documented_only**. Descriptor contents, receiver fields, and global values are read at different stages; the function does not operate entirely on an immutable entry snapshot.

## Interface

### Inputs and representation

The function accepts two addresses:

- **Receiver:** supports the field accesses, writes, and indirect dispatches specified below.
- **Descriptor:** supports reads of four consecutive 32-bit words.

The supplied representation uses 32-bit words and 32-bit addresses. Byte order and alignment tolerance are not established.

Descriptor words are numbered from zero:

| Word | Byte offset | Established role |
|---|---:|---|
| 0 | 0 | Copied into receiver state; reread for the second dispatched operation. |
| 1 | 4 | Copied into receiver state; later enables searching and supplies candidate comparison values. |
| 2 | 8 | Copied into receiver state; later supplies the default selection. |
| 3 | 12 | Copied into receiver state; on a match, tested as signed and conditionally added to the matching index. |

These roles refer to values at the specified read times, not necessarily to values present at function entry.

### Receiver layout

All offsets below are byte offsets from the receiver address.

| Offset | Width | Established use |
|---|---:|---|
| 0 | 32 bits | Dispatch-table address, read separately for successive dispatched operations. |
| 56 | 32 bits | First argument to lookup acquisition. |
| 256 | 32 bits | Signed starting index. |
| 260 | 32 bits | Signed exclusive search limit. |
| 264 | 32 bits | Destination for descriptor word 0. |
| 268 | 32 bits | Destination for descriptor word 1. |
| 272 | 32 bits | Destination for descriptor word 2. |
| 276 | 32 bits | Destination for descriptor word 3. |

The four dispatch-table entry offsets used, in invocation order, are 348, 416, 400, and 372 bytes.

### Global inputs

Two distinct fixed global words are relevant on the search path:

- **G-L:** the lower-addressed 32-bit global word.
- **G-H:** the higher-addressed 32-bit global word, located four bytes after G-L.

Their values and meanings are unknown. These labels identify storage locations, not semantic types.

**Load order and argument order differ:** G-H is read first, then G-L. Candidate lookup subsequently receives the captured G-L value as argument 2 and the captured G-H value as argument 3.

### Output and preconditions

There is no explicit return value. Observable results are receiver writes, dependency invocations, and effects produced by those dependencies.

The caller must provide valid memory for every access actually reached, a usable dispatch table, and callable targets with compatible argument conventions. No explicit checks establish descriptor length, pointer validity, bounds safety, alignment, or callable-target validity.

Normal return requires all invoked operations to return normally. Invalid memory, invalid dispatch, or violations of dependency preconditions have no established recovery behavior.

## Behavior

### Initial descriptor transfer

Before its first receiver write, the function reads descriptor words in this order:

1. Word 1.
2. Word 2.
3. Word 3.
4. Word 0.

It then writes the captured values in descriptor-word order to receiver offsets 264, 268, 272, and 276.

All four descriptor reads therefore precede all four destination writes. Overlap cannot make one of these initial destination writes alter a descriptor value still awaiting its initial capture. This ordering does not establish that overlapping arguments are permitted by the surrounding API, nor does it protect later descriptor rereads.

The copied values are unchanged 32-bit bit patterns. No validation or interpretation precedes these writes.

### Initial dispatched operations

After completing the four writes, the function resolves and invokes the dispatch-table entry at byte offset 348 with the receiver as its sole visible argument.

After that operation returns, it resolves the entry at byte offset 416 and invokes it with:

1. The receiver.
2. A fresh read of descriptor word 0.

The second argument may differ from the value initially stored at receiver offset 264. The first operation may have modified descriptor storage, directly or through aliasing.

The dispatch-table address is obtained again for the second invocation. A change to that address during the first operation can affect the second target. No preservation guarantee is assumed.

### Default selection and search decision

After the second initial operation returns, the function reads descriptor word 2 and retains its signed 32-bit value as the default selection.

It then reads descriptor word 1:

- Zero bypasses lookup acquisition and searching.
- Nonzero enters the search path.

These reads occur after both initial dispatched operations. Changes made by those operations can therefore affect both the default and the decision to search.

Once captured, the default selection is not refreshed during searching. The search-enabling test is also not repeated.

### Lookup acquisition and global capture

On the search path, the function calls **F-682adfd5a561** with:

1. The current 32-bit receiver field at byte offset 56.
2. The integer value 6.

The returned 32-bit value becomes the lookup token used by subsequent candidate calls. Whether it is an address, handle, or another representation is **documented_only**.

After acquisition returns, the function performs two global reads in this order:

1. Read G-H and retain its 32-bit value.
2. Read G-L and retain its 32-bit value.

Both reads precede handling of a zero acquisition result. Thus, even when acquisition returns zero, the represented execution reads G-H and then G-L before invoking the failure dependency, provided those accesses complete.

If the acquisition result is zero, the function invokes **F-a817d8d58c10** without visible arguments. Static evidence marks that dependency as nonreturning. No normal continuation, returned error value, or cleanup path is established for this condition.

Otherwise, searching proceeds using the nonzero acquisition result and the two captured global values. Candidate calls do not reload either global word. Later changes to global storage do not themselves change the retained arguments.

### Candidate search

The function reads receiver offset 256 once to obtain the signed 32-bit starting index. This read occurs after acquisition, both global reads, and the zero-result check.

Before each candidate call, the current index is compared against a fresh signed 32-bit read of receiver offset 260. A candidate is admitted only when the index is strictly less than that exclusive limit.

For every admitted candidate, **F-9dfe5f6048f1** receives these arguments:

| Argument | Value |
|---:|---|
| 1 | The retained nonzero result from F-682adfd5a561. |
| 2 | The value captured from G-L, the lower-addressed global word read second. |
| 3 | The value captured from G-H, the higher-addressed global word read first. |
| 4 | The current signed 32-bit candidate index. |

After the candidate call returns, its 32-bit result is compared with a fresh read of descriptor word 1. Equality is equality of the represented 32-bit values; signedness does not change that test.

On a mismatch, the index increases by one and the next limit check occurs.

On a match:

- The selection becomes the matching index.
- Descriptor word 3 is read and tested as signed.
- If the tested value is nonnegative, word 3 is read again and added to the matching index.
- If the tested value is negative, the selection remains the matching index.
- Searching stops immediately.

The sign test and addition use separate descriptor reads. The representation does not substitute one captured adjustment value for both accesses.

The first match in actual execution order wins. Because the exclusive limit is reread and the comparison word is read after each candidate call, this is not necessarily a search over a fixed range for a fixed key.

An empty or reversed range performs no candidate calls, but acquisition and both global reads have already occurred. Exhaustion without a match preserves the previously captured default selection.

### Selection delivery

After searching is skipped, exhausted, or stopped by a match, the function invokes the dispatch-table entry at byte offset 400 with:

1. The receiver.
2. The selected value.
3. The integer value 1.

After that operation returns, it resolves and invokes the entry at byte offset 372 with:

1. The receiver.
2. The same selected value.

The dispatch-table address is obtained for each invocation. Changes made by the first delivery operation may affect the second target, but the function does not recompute the selection between them.

The representation establishes the final operation’s target selection and arguments. Its precise high-level control-transfer convention remains **documented_only**.

### Integer boundaries

The starting index, exclusive limit, selection, and adjustment are treated as signed 32-bit values where ordering or arithmetic is involved. Negative starting indices are not explicitly rejected.

The adjustment is applied only when its sign-test read is nonnegative. A negative adjustment suppresses addition; it does not request subtraction.

No explicit overflow check or saturation is present for matching-index-plus-adjustment arithmetic. For a mathematically representable result, the selection is that sum. Overflow behavior is **documented_only**; wrapping is not asserted.

An admitted candidate is strictly below a signed 32-bit limit and therefore cannot equal the largest signed 32-bit value. Incrementing such a candidate by one remains within the signed range before the next comparison.

## Side effects

The function directly writes four 32-bit receiver fields at offsets 264, 268, 272, and 276. These writes precede every dependency invocation.

All other state changes depend on invoked operations. No dependency contract supplied here guarantees preservation of descriptor contents, receiver fields, dispatch-table state, global storage, lookup state, or other externally visible state.

The visible body performs no explicit allocation, release, or reference-count operation. This does not establish that dependencies perform none, or that passing an argument cannot transfer ownership under an unknown dependency contract.

The lookup token is retained and passed to candidate calls without a visible release. Its ownership, lifetime, and validity requirements are **documented_only**.

No rollback is visible. A nonreturning operation or other interruption does not cause the function to undo its earlier writes or dependency effects.

No synchronization is visible. The ordered reads describe the supplied representation; they do not establish behavior under unsynchronized concurrent mutation.

## Dependencies

| Dependency | Established interaction | Unresolved contract |
|---|---|---|
| **F-682adfd5a561** | Search-path acquisition using receiver offset 56 and the value 6. Its result is checked against zero after the global reads. | Semantic purpose, result type, ownership, mutations, and failure details. |
| **F-9dfe5f6048f1** | One invocation per admitted candidate until match or exhaustion. Arguments are acquisition result, captured G-L, captured G-H, and index. | Lookup semantics, result domain, state preservation, ownership effects, and additional failures. |
| **F-a817d8d58c10** | Invoked without visible arguments when acquisition returns zero; statically marked nonreturning. | Whether it terminates, throws, blocks, transfers control, or has other effects. |

Four additional dependencies are selected through receiver dispatch-table offsets 348, 416, 400, and 372. Their opaque function IDs are unavailable. Their arguments and invocation order are established above; their implementation contracts are not.

Neither G-L nor G-H is established to be a string, type identifier, immutable constant, address, or ownership-bearing value. Only their 32-bit widths, relative locations, read order, and argument mapping are established.

## Acceptance examples

These are conditional acceptance criteria, not executed tests. Unless stated otherwise, memory is valid, dependencies return normally, and relevant state remains unchanged.

### Search disabled

If descriptor word 1 is zero and word 2 is −7 at the selection stage:

- The initial copy and both initial dispatched operations occur.
- No acquisition, global capture, or candidate lookup occurs.
- Both delivery operations receive −7; the first also receives 1.
- Word 3 does not affect selection.

### Global load order versus argument order

Suppose acquisition returns a nonzero token, G-H contains 90, G-L contains 40, and index 2 is admitted:

- G-H is read before G-L.
- The candidate call receives the token as argument 1, 40 as argument 2, 90 as argument 3, and 2 as argument 4.

The global read first is therefore supplied third, not second.

If that candidate call changes G-L to 41 and G-H to 91, a later admitted candidate still receives 40 and 90 in arguments 2 and 3. The function does not reload the global words.

### No match

With comparison word 42, default selection 9, and stable bounds 2 and 5:

- Successful acquisition is followed by candidate calls at indices 2, 3, and 4.
- If none returns 42, both delivery operations receive 9.
- A change to descriptor word 2 during searching does not refresh that default.

### Match and adjustment

If the first match is at index 3:

- Stable word 3 equal to 4 produces selection 7.
- Word 3 equal to −1 at the sign test leaves selection 3.
- No later candidate is examined.

The addition case assumes both adjustment reads yield 4 and the sum is representable.

### Empty range

With a nonzero search-control word and start greater than or equal to limit:

- Acquisition still occurs.
- G-H and then G-L are still read.
- A nonzero acquisition result leads to no candidate calls and delivery of the default.
- A zero result invokes F-a817d8d58c10 instead.

### Mutation by dependencies

If the first dispatched operation changes descriptor word 0 from 10 to 20, the second receives 20. The initial transfer wrote 10 to receiver offset 264, absent a separate later change to that field.

If a candidate call changes descriptor word 1 and returns the newly stored value, the subsequent comparison can match that new value.

If a candidate call changes the exclusive limit, the current candidate has already been admitted. The changed limit affects the next condition check if searching continues.

### Acquisition failure

With searching enabled and acquisition returning zero:

- The four receiver writes and two initial dispatched operations have already occurred.
- G-H is read, then G-L.
- F-a817d8d58c10 is invoked.
- Normal selection delivery is not established.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of the receiver, descriptor words, lookup token, and selection.
- Concrete identities and contracts of the four indirect targets.
- Numeric values, semantic types, and external writers of G-L and G-H.
- Overflow behavior for the adjustment addition.
- Dependency ownership, lifetime, mutation, exception, and termination guarantees.
- Whether overlapping receiver and descriptor storage is permitted by the surrounding API.
- Behavior after invalid access, invalid dispatch, or violated dependency preconditions.
- Behavior under unsynchronized concurrent mutation.
- The exact control-transfer mechanism of the final indirect operation.
- The mechanism by which F-a817d8d58c10 prevents normal return.

## Evidence

The established claims derive from static inspection of the supplied function representation: 32-bit accesses, receiver and dispatch offsets, dependency arguments, read and write ordering, signed comparisons, arithmetic, and branches.

In particular, the representation reads the higher-addressed global word before the lower-addressed word, while supplying the lower-addressed captured value as argument 2 and the higher-addressed captured value as argument 3 to F-9dfe5f6048f1. Both global reads precede zero-result handling.

The nonreturning characterization of F-a817d8d58c10 is static metadata, not runtime confirmation. Dependency implementations were not supplied.

No runtime tests have been performed. The acceptance examples specify conditional expected behavior only.
