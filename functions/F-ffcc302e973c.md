<!-- Trusted provenance: F-ffcc302e973c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ffcc302e973c

## Purpose

F-ffcc302e973c performs an ordered, teardown-like operation on an input object and several derived or associated regions. It replaces header words, conditionally drains a linked chain, invokes dependencies, processes an optional auxiliary record and collection, conditionally updates context references, and returns a derived address.

**Established static behavior** is the sequence of accesses, calls, branches, and address calculations below. **Inferred intent** is staged destruction or cleanup. Neither that interpretation nor cleanup-shaped calls establish ownership transfer, deallocation, reference-count changes, or continued storage validity.

Two branches invoke a precursor dependency followed by F-feee9156a257, which the supplied representation explicitly marks as nonreturning. There is no ordinary error-result channel.

## Interface

### Arguments and representation

The function accepts four arguments:

| Argument | Established use |
|---|---|
| Input object address | Initial header destination and base for fixed-offset accesses. |
| Auxiliary pointer | Initial contents of a mutable pointer slot. |
| Auxiliary word 1 | Initial contents of a mutable 32-bit slot. |
| Auxiliary word 2 | Initial contents of another mutable 32-bit slot. |

The represented environment uses 32-bit words and addresses. Every offset in this contract is in **bytes**. Null is the zero address. Conditions generally distinguish zero from nonzero without assigning further meaning to nonzero values.

Three temporary slots initially contain the auxiliary arguments. A dependency can access all three by address; a later iteration dependency receives the pointer slot by address. Their contents must therefore be described at the time of use, not assumed to remain equal to entry values.

A separate 20-byte temporary area supports collection processing. Its internal layout is unknown.

### Derived regions and result

The following names denote addresses, not concrete types:

1. F-03144a4f049b receives input +0x144 and returns A1.
2. F-03144a4f049b receives A1 −0xB8 and returns A2.
3. F-c1466ea05103 receives A2 −0x20 and returns A3.
4. F-c1466ea05103 receives A3 −0x14 and returns A4.
5. F-c1466ea05103 receives A4 −0x14 and returns A5.
6. Region **H** is A5 −0x44.
7. Later, F-c1466ea05103 receives A5 −0x2C and returns A6.
8. Region **B** is A6 −0x18.

Ordinary return yields B. There is no established requirement that these dependencies return their input addresses, so B cannot be reduced to a fixed input-relative offset.

The calculations use the represented 32-bit address-width arithmetic. No overflow, underflow, alignment, or allocation-boundary checks are shown. Ordinary return does not establish that B still refers to live storage.

### Global value sources

Five global-value reads supply four header writes and one notification argument. For this contract, their sources are labeled **G-entry**, **G-H**, **G-B**, **G-notify**, and **G-final**.

These labels do not denote immutable constants. Each value is read at its corresponding assignment or call evaluation, as specified below. The representation does not establish immutable storage or stability across dependencies and does not justify caching these values at entry.

### Preconditions

Every directly accessed location must be readable or writable when accessed. This includes input-relative fields, derived regions, nodes, context records, auxiliary records, dispatch-table pointers, dispatch entries, and temporary output locations.

These are execution preconditions, not validations performed by the function. There is no initial input-null check, general dependency-result validation, or comprehensive check that a previously tested pointer remains usable after another read or call.

## Behavior

### 1. Entry header and initial guard

The function reads the 32-bit value from G-entry and writes it to input +0. It then initializes the three auxiliary temporary slots.

F-2be4348b26cc receives input +0x204. After that call returns, the function reads input +0x200.

If that word is nonzero, F-da23f49bf314 is called, followed by nonreturning F-feee9156a257. The header replacement has already occurred. The tested word is its post-call value, not an entry snapshot.

Otherwise, F-b014851c53c0 receives input +0x220.

### 2. Optional chain preparation and draining

The function tests the pointer-valued word at input +0x214. If null, it skips this entire phase, even if input +0x204 contains a nonnull head.

If the branch test is nonnull, the following accesses occur in order:

1. Read input +0x214 again, then load the word at that address +8 into the auxiliary-word-1 slot.
2. Read input +0x214 again into the auxiliary-pointer slot.
3. Call F-211584876e6f with the addresses of:
   - the auxiliary-word-2 slot;
   - the head slot at input +0x204;
   - the auxiliary-word-1 slot;
   - the auxiliary-pointer slot.
4. Call F-3619430fef39 with the head-slot address, a fresh read of input +0x214, and zero.

The branch test, field-load base, pointer-slot assignment, and later call argument are separate reads of input +0x214. The contract does not require them to yield the same address. In particular, the loaded +8 word and the stored pointer need not belong to one retained record.

F-211584876e6f can modify the exposed slots and head. Its unknown contract also does not establish whether it retains those addresses.

The function then repeatedly reads the head at input +0x204. For each nonnull node:

1. Read the node’s +0 word and write it into the head slot.
2. Call F-ff1bdc376dcd with the node’s +8 word, its +4 word, and zero.
3. Call F-ff1bdc376dcd with the node address, one, and zero.

Thus the established node layout is:

| Offset | Use |
|---|---|
| +0 | Replacement head |
| +4 | Second argument of the first per-node call |
| +8 | First argument of the first per-node call |

The head update precedes both calls. The next iteration reloads the head, allowing dependency changes to affect traversal. No relative evaluation order is established between the +8 and +4 argument loads.

There is no cycle detection, traversal bound, or node-validity check. “Payload release” and “node release” are inferred interpretations, not established dependency contracts.

### 3. Intermediate operations and H

F-561858688f94 receives input +0x1FC. The address-producing calls through A5 then occur in the order listed under Interface.

The function reads G-H and writes that current 32-bit value to H +0. This is a new global read after the preceding dependencies.

It next tests H +0x2C. If nonzero, F-58fe1fb670b9 receives H and the **then-current contents** of the auxiliary-pointer, auxiliary-word-1, and auxiliary-word-2 slots, in that order.

These contents are not necessarily the original arguments or the values last written by F-211584876e6f. Dependencies have executed in between, and no preservation guarantee is available for exposed storage.

After the conditional call, H +0x30 is read. If nonzero:

1. F-d0df5eb7e117 is called without explicit arguments.
2. F-a931fcc67fbc receives H.

The second condition is evaluated after any F-58fe1fb670b9 call, not from a saved pair of field values.

### 4. B setup and notification

F-c1466ea05103 receives A5 −0x2C, producing A6 and therefore B.

The function reads G-B and writes its current value to B +0, then calls F-408b56b8b7c8 with B.

F-3fdd14ecd250 is called next. For the subsequent F-77781c8e0ff9 call, the function obtains:

- the word at returned-context +0x1C;
- the current word read from G-notify;
- B.

Both the context-field read and the G-notify read occur after the context resolver returns and before notification is invoked. The representation does not establish a relative evaluation order between these argument reads. G-notify is not an entry-cached constant; preceding dependencies, including the resolver, have already had an opportunity to affect shared state.

There is no null guard for this context access.

### 5. Optional auxiliary record

The function reads B +0x0C and retains that address as **D**. If null, the entire remainder of this phase is skipped.

Otherwise, it saves D +4 as **T**, then calls F-a1a48bd3fbde with B.

#### Predicate-controlled subphase

If the predicate result is nonzero:

1. Resolve a context through F-3fdd14ecd250 and test its +0x28 pointer.
2. If nonnull:
   - Resolve a context again.
   - Through the object obtained from that context’s +0x28 field, invoke the indirect target at dispatch-table byte offset +0x48.
   - Re-read B +0x0C and load the word at the resulting record address +4.
   - Resolve a context again.
   - Call F-a5514b9f24f7 with that context’s +0x28 word and the freshly loaded record word.
3. Call F-65528ded3450 with B and zero.
4. Call F-a82ec4b43e2f with saved T.

The initial context-field check does not validate a later resolver result. The later record read does not reuse D and is not separately null-checked. T remains the earlier saved word.

#### Collection subphase

The function tests D +8 using retained D, regardless of changes to B +0x0C.

If nonzero:

1. Read D +8 again and call F-538044b67754 with the 20-byte temporary area, that value, and the 32-bit pattern 0xFFFFFFFD.
2. Repeatedly call F-ebd94610a438 with the temporary area and the auxiliary-pointer-slot address.
3. For each nonzero iteration result, inspect the current pointer slot. If nonnull, follow its dispatch-table pointer and invoke the target at byte offset +4.
4. Stop on a zero iteration result.
5. Call F-944b595e25e9 with the temporary area.
6. Test D +8 again. If nonnull, obtain the indirect target through the current D +8 object’s dispatch table at byte offset +4 and invoke it.

The collection test does not establish that the separately read initialization argument is identical. Likewise, the post-finalization operation uses D +8 again rather than a saved collection address.

The selector’s signed 32-bit interpretation is −3; only its bit pattern is established. Its semantic meaning is unknown. Indirect calls have no explicit arguments in the supplied representation; implicit calling-convention details are unknown.

#### Context comparisons and record completion

Still within the nonnull-D branch:

1. Resolve a context. If its +0x28 word equals B, call F-a82ec4b43e2f with saved T.
2. Resolve a context again. If its +0x24 word equals B, resolve a context once more and write zero to the **last returned context’s** +0x24 field.
3. Call F-65528ded3450 with B and zero.
4. Call F-8349c252c168 with retained D and zero.

Thus F-65528ded3450 and F-a82ec4b43e2f can each be called twice. Their first occurrences depend on the predicate; their later occurrences follow the separate conditions above.

The function does not directly clear B +0x0C.

### 6. Final header and conditional fatal check

The function reads G-final and writes its current value to B +0. This fourth header write uses a fresh global read after the preceding processing.

It then tests B +4. If initially null, it skips the entire final conditional block, including the B +8 check.

If initially nonnull:

1. Call F-c719314a16bb with B.
2. Re-read B +4. If nonnull, invoke the indirect target at dispatch-table byte offset +4 through the current object referenced there.
3. Read B +8. If nonzero, call F-c8eefdf96f4b, followed by nonreturning F-feee9156a257.

If execution continues normally, return B. A nonzero B +8 does not independently select the fatal path when the first B +4 test was null.

## Side effects

Established direct external writes are:

- Four header writes: input +0, H +0, B +0, and B +0 again.
- Head-slot replacement before each node’s two dependency calls.
- Conditional zeroing of a freshly resolved context’s +0x24 field.

The header writes use four separately timed global-value reads. The fifth global read supplies the notification argument. Neither header-like use nor global storage establishes immutability.

Temporary writes include auxiliary-slot initialization, chain-preparation assignments, and dependency-mediated updates through exposed addresses.

Regions may alias. No distinct-allocation guarantee is established for the input, H, B, nodes, records, contexts, or dependency-returned addresses.

Every dependency and indirect call may have additional effects not specified here. Global preservation, context stability, storage lifetime, and preservation of exposed temporary contents are unknown. Repeated resolutions and field reads must not be replaced contractually by cached entry values.

Ownership and release semantics are **documented_only**. No allocation origin, transfer rule, reference-count rule, or deallocation guarantee is established.

There is no explicit rollback. If a later access fails or a dependency does not return, earlier writes and calls remain part of the executed prefix.

## Dependencies

| Opaque ID | Established role |
|---|---|
| F-2be4348b26cc | Initial operation on input +0x204. |
| F-da23f49bf314 | Early fatal-path precursor. |
| F-feee9156a257 | Explicitly nonreturning operation on both fatal paths. |
| F-b014851c53c0 | Operation on input +0x220. |
| F-211584876e6f | Receives three temporary-slot addresses and the head-slot address. |
| F-3619430fef39 | Receives head-slot address, freshly read input +0x214, and zero. |
| F-ff1bdc376dcd | Two calls per processed node. |
| F-561858688f94 | Operation on input +0x1FC. |
| F-03144a4f049b | First two address-producing calls. |
| F-c1466ea05103 | Four subsequent address-producing calls. |
| F-58fe1fb670b9 | Conditional H operation with current auxiliary-slot contents. |
| F-d0df5eb7e117 | Precedes the second conditional H operation. |
| F-a931fcc67fbc | Conditional operation on H. |
| F-408b56b8b7c8 | Operation after B’s first header replacement. |
| F-3fdd14ecd250 | Repeated context resolver. |
| F-77781c8e0ff9 | Notification using context +0x1C, freshly read G-notify, and B. |
| F-a1a48bd3fbde | Predicate for an auxiliary-record subphase. |
| F-a5514b9f24f7 | Receives context +0x28 and a freshly loaded record word. |
| F-65528ded3450 | Receives B and zero, once or twice for nonnull D. |
| F-a82ec4b43e2f | Receives saved T, conditionally once or twice. |
| F-538044b67754 | Initializes temporary collection state. |
| F-ebd94610a438 | Advances iteration and receives the pointer slot by address. |
| F-944b595e25e9 | Finalizes temporary collection state. |
| F-8349c252c168 | Final operation on retained D. |
| F-c719314a16bb | Operation selected by the initial final-phase B +4 test. |
| F-c8eefdf96f4b | Final fatal-path precursor. |

Indirect targets are identified only by dispatch-table offsets +4 and +0x48. Their concrete identities, argument conventions, effects, and return behavior remain unknown.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

1. **Early fatal condition.** If input +0x200 is nonzero after F-2be4348b26cc returns, the G-entry-derived header write has already occurred. F-da23f49bf314 precedes F-feee9156a257; the input +0x220 operation is not reached.

2. **Separately timed global reads.** If an intervening dependency changes G-H and execution reaches the H assignment, that assignment uses G-H’s value when read there, not an entry snapshot. If the notification’s resolver changes G-notify, notification likewise uses the post-resolver read.

3. **Skipped chain.** A null branch-test read of input +0x214 skips chain processing even with a nonnull head.

4. **Distinct preparation reads.** If the branch test is nonnull, the next input +0x214 read yields record R, and the following read yields record S, the word slot receives R +8 while the pointer slot receives S, provided those accesses are valid. R and S need not match.

5. **Single-node draining.** If the prepared head identifies a node with a zero next word and dependencies leave the head unchanged, the head becomes zero before either per-node call. The field-based call precedes the node-address call.

6. **Exposed-slot mutation.** If F-211584876e6f writes particular auxiliary-slot values, no intervening operation changes those slots, and H +0x2C is later nonzero, F-58fe1fb670b9 receives those values. Without the no-intervening-modification condition, it receives whatever contents are current at its call.

7. **Null D.** A null B +0x0C read skips the predicate, collection processing, token operations, record-specific context comparisons, and F-8349c252c168. The G-final read and final header write still occur if execution reaches that phase.

8. **Two token calls.** With nonnull D, a nonzero predicate result and a later context +0x28 equal to B cause two calls using saved T, assuming intervening calls return.

9. **Null iteration item.** A nonzero iteration result with a null pointer-slot value causes no item dispatch and proceeds to another iteration call.

10. **Changed context identity.** If the +0x24 comparison succeeds on one context but the next resolver returns another, zero is written to the latter context’s +0x24 field.

11. **Nested final fatal check.** A null first B +4 read skips the fatal check despite nonzero B +8. With a nonnull first read, execution reaching a nonzero B +8 read invokes the fatal precursor and nonreturning dependency.

## Uncertainties

- **Inferred intent:** staged teardown across related object regions.
- **documented_only:** concrete types, inheritance relationships, and meanings of header values and the notification value.
- **documented_only:** mutability guarantees, synchronization, and stability of the five global sources.
- **documented_only:** ownership, allocation provenance, storage release, and lifetime after calls.
- **documented_only:** dependency effects on shared state and exposed temporary storage.
- **documented_only:** whether repeated context resolutions or pointer-field reads return identical values.
- **documented_only:** indirect-call conventions and target semantics.
- **documented_only:** dependency failures, exceptions, and nonreturning behavior beyond the explicitly marked fatal dependency.
- **documented_only:** behavior under invalid storage, cyclic chains, concurrent mutation, or callbacks invalidating later accesses.
- **documented_only:** address-producing dependency semantics beyond their observed inputs and use of their results.

## Evidence

This contract uses the supplied static representation and opaque dependency mapping. Established evidence comprises 32-bit accesses, byte-offset relationships, five separately situated global-value reads, ordered assignments and calls, repeated pointer reads, exposed temporary addresses, branches, indirect dispatch offsets, and the returned derived address.

Argument positions do not establish an otherwise unspecified order among sibling argument loads. Semantic labels such as “header,” “record,” “token,” and “collection” organize observed behavior without supplying missing dependency contracts.

No runtime tests have been performed.
