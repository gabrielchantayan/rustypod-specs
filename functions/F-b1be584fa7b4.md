<!-- Trusted provenance: F-b1be584fa7b4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b1be584fa7b4

## Purpose

F-b1be584fa7b4 performs unconditional object-related processing, optionally selects one of four actions, performs final processing, and returns the 32-bit value **1** on normal completion.

The action precedence is:

1. Artist selection.
2. Album selection.
3. Compilation selection.
4. Genius start.

Each action is tested only if all preceding tests returned zero. The first nonzero result selects its action and prevents further tests.

**Established static evidence:** The supplied representation establishes the control flow, explicit calls, argument values, memory-access widths, direct writes, and normal return described below.

**Inferred intent:** The visible matching text suggests a media-selection workflow. Descriptions such as “selected status,” “payload,” and “action context” identify apparent roles; they do not establish the semantics of unavailable dependencies.

**Unknown:** Dependency implementations, complete object layouts, referenced global data, and runtime behavior are unavailable. No preservation, ownership, or failure guarantees are inferred from dependency names or apparent roles.

## Interface

### Inputs

**Object address.** A pointer represented in 32-bit storage. The exercised path requires access to:

- A 32-bit dispatch-table pointer at object byte offset 0.
- One writable byte at object byte offset 176, hexadecimal **0xB0**, when action matching is enabled.
- Additional storage required by dependencies receiving the object address.

For artist, album, or compilation selection, the dispatch table must contain a callable entry at byte offset 284, hexadecimal **0x11C**.

**Dispatch-enable value.** A represented 32-bit integer:

- Zero disables action matching and action-specific processing.
- Any nonzero value enables them, including a negative value.

All offsets in this contract are byte offsets. Numeric arguments and record words are 32 bits unless another width is stated explicitly.

### Output

Normal completion returns the 32-bit value **1**. This value does not distinguish disabled dispatch, no match, or a particular selected action. Ignored dependency results are not converted into a different return value.

No other normal return is shown.

### Preconditions and errors

The object, accessed global storage, indirectly loaded addresses, and selected dispatch entry must be valid for the operations reached.

For artist, album, or compilation selection, the first result from F-fdd8589eca2d must identify storage writable for a **16-bit** value at offset **0x80**.

Dependencies must accept the supplied arguments, including addresses of stack-backed temporary storage. There are no explicit null checks, bounds checks, type checks, dependency-failure checks, or local recovery paths.

Final processing and return 1 are conditional on preceding operations completing normally. Behavior following an invalid access, exception, nonlocal transfer, or dependency failure is not established.

## Behavior

### Common processing

Every invocation performs these operations in order:

1. Call F-dafcc25e1374 without explicit arguments.
2. Call F-753f887b468d without explicit arguments, obtaining a 64-bit result.
3. Call F-187151a4b5ba with the object address, the upper 32 bits of that result, and its lower 32 bits.
4. Call F-5eac81954d4b with the lower 32 bits and zero.

The upper half is extracted by an unsigned 64-bit right shift of 32 positions and conveyed as a 32-bit argument. This establishes the transferred bit pattern, not the receiving dependency’s numeric interpretation.

These calls occur even when dispatch is disabled. Their effects may influence later object accesses and global reads.

### Dispatch gate and first test

If the dispatch-enable value is zero, proceed directly to final processing.

Otherwise, perform the following sequence:

1. Write zero to exactly one byte at object offset **0xB0**.
2. Call F-9b9c1c7483f4 with stack-backed primary storage, the text reference for `SelectArtist`, and auxiliary stack-backed storage. Retain its returned argument value.
3. Read the matching-receiver global and save that value for possible later tests.
4. Read the matching-receiver global **again**. Use this second value, together with the constructed argument, in the artist test call to F-8094c3d72052.
5. Call F-8a5aadac202f on the primary temporary storage.
6. Read and retain the separate 32-bit action-context global.
7. Examine the retained artist-test result.

The two matching-receiver reads have different consumers:

- The **first** supplies the saved receiver used by album, compilation, and Genius tests, if reached.
- The **second** supplies the artist test.

No dependency call is shown between these two reads. Nevertheless, their equality is not established without a stability assumption about the global and its environment. The contract does not collapse them into one read.

The action-context read occurs after the artist test and its temporary cleanup, even if the artist test fails. Changes made during those calls can therefore affect the captured context.

### Remaining tests and precedence

A nonzero artist result selects the artist branch. Otherwise, test the following actions in order:

| Order | Matching text reference | Nonzero result selects |
|---|---|---|
| 2 | `SelectAlbum` | Album selection |
| 3 | `SelectCompilation` | Compilation selection |
| 4 | `StartGenius` | Genius start |

Each reached test:

- Constructs an argument using F-9b9c1c7483f4 and primary and auxiliary stack-backed storage.
- Calls F-8094c3d72052 with the **saved first receiver value** and the constructor’s returned argument.
- Calls F-8a5aadac202f on that test’s primary temporary storage before entering its branch or proceeding to another test.

The result is a zero/nonzero predicate; negative and positive nonzero results both select the action.

The action context is not reread for later tests or branches. The matching-receiver global is also not reread for these later tests.

If all four results are zero, no action-specific processing occurs. The byte clear and all four construction, matching, and cleanup sequences have still occurred.

### Four-word action records

Every matched action uses a contiguous stack-backed record containing four 32-bit words, at relative offsets **0, 4, 8, and 12**.

F-78801ca166f2 receives the record’s address before explicit assignments. The function then assigns all four words, with branch-specific values and write order described below.

After those assignments, F-fdd8589eca2d runs **before** the words are consumed as arguments to F-8fa83bbc2366. That call receives:

1. The result of the intervening F-fdd8589eca2d call.
2. The previously captured action context.
3. The then-current first record word.
4. The then-current second record word.
5. The then-current third record word.
6. The then-current fourth record word.

The assigned record contents and the arguments ultimately consumed are distinct facts. F-78801ca166f2 has received the record address; no non-retention guarantee or dependency body is supplied. Consequently, the intervening F-fdd8589eca2d call cannot be assumed to preserve that exposed storage.

The explicit assignments establish the contents immediately after those writes. They establish the later argument values only **if the record remains unchanged until argument consumption**. The relative evaluation order of the individual argument loads is not established here.

### Artist, album, and compilation selections

These branches share this call-level sequence:

1. Call F-fdd8589eca2d.
2. Write the 16-bit value **1** at offset **0x80** from its returned address.
3. Call F-6a21f4f07f67.
4. Call F-aaf1683df92f with that result, **128**, and the branch-associated global payload read for this call.
5. Call F-78801ca166f2 with the branch’s four-word record address.
6. Perform the branch-specific record assignments.
7. Call F-fdd8589eca2d again.
8. Call F-8fa83bbc2366 with that second result, the captured context, and the then-current record words.
9. Call F-c7dac3ee6298 with the object and the branch-specific seven-value tuple.
10. Construct a selected-status argument using F-9b9c1c7483f4.
11. Read the object’s dispatch-table pointer and call its entry at offset **0x11C**, passing the object and the constructor’s returned argument.
12. Call F-8a5aadac202f on the selected-status primary temporary.

The two F-fdd8589eca2d calls are distinct. Their results need not be equal. The explicit 16-bit write uses the first result; F-8fa83bbc2366 receives the second.

The dispatch-table lookup occurs after selected-status construction, not at function entry. Earlier dependencies, including that constructor, may affect the pointer or entry subsequently read.

#### Artist values

The payload supplied to F-aaf1683df92f is the artist-associated global payload.

The record assignments occur in this order:

1. Second word: captured action context plus **4**.
2. First word: zero.
3. Third word: zero.
4. Fourth word: zero.

Thus, immediately after assignment, the record is ordered as **zero, context plus 4, zero, zero**, subject to the arithmetic limitation below.

F-c7dac3ee6298 receives, after the object:

**17, 0, 1, 4, 2, 0, 0**

The selected-status constructor uses the artist-associated text reference.

The addition is represented as signed 32-bit integer arithmetic. For contexts from −2,147,483,648 through 2,147,483,643, the mathematical sum fits. For larger signed contexts, the representation does not establish portable wrapping or another defined overflow behavior. No 64-bit widening or local overflow check is shown.

#### Album values

The payload supplied to F-aaf1683df92f is the album-associated global payload.

The record assignments occur in this order:

1. First word: zero.
2. Third word: zero.
3. Fourth word: zero.
4. Second word: the album-associated 32-bit global value, read at this point.

Immediately after assignment, the record is **zero, album-associated value, zero, zero**.

F-c7dac3ee6298 receives, after the object:

**17, 0, 1, 5, 3, 0, 0**

The selected-status constructor uses the album-associated text reference.

#### Compilation values

The payload supplied to F-aaf1683df92f is the compilation-associated global payload.

The record assignments occur in this order:

1. First word: zero.
2. Third word: zero.
3. Fourth word: zero.
4. Second word: the compilation-associated 32-bit global value, read at this point.

Immediately after assignment, the record is **zero, compilation-associated value, zero, zero**.

F-c7dac3ee6298 receives, after the object:

**17, 0, 1, 10, 8, 0, 0**

The selected-status constructor uses the compilation-associated text reference.

For all three selected-status constructions, the supplied representation specifies an address **one byte beyond** its associated named text location. The underlying bytes are unavailable. Exact selected-status strings, including their initial characters, are **documented_only**.

### Genius start

The Genius branch performs a different sequence:

1. Call F-78801ca166f2 with a four-word record address.
2. Assign zero to the first word, then the third word, then the fourth word.
3. Read the Genius-associated 32-bit global value and assign it to the second word.
4. Call F-fdd8589eca2d.
5. Call F-8fa83bbc2366 with that result, the captured context, and the then-current four record words in record order.
6. Call F-6a21f4f07f67.
7. Call F-aaf1683df92f with that result, **128**, and the Genius-associated global payload read for this call.
8. Call F-66c31f8db1bf with the object and zero.

Immediately after assignment, the record is **zero, Genius-associated value, zero, zero**. As in every other branch, preservation until consumption is not established.

Unlike the selection branches, Genius processing has no explicit 16-bit write at offset 0x80, no F-c7dac3ee6298 call, and no selected-status construction or dispatch-table callback.

Its F-8fa83bbc2366 call precedes F-aaf1683df92f; the three selection branches use the opposite order.

### Final processing

After disabled dispatch, no match, or completion of a selected branch:

1. Read a global pointer.
2. Read the 32-bit value at the address identified by that pointer.
3. Call F-b002ce492389 with the loaded value.
4. Return the 32-bit value **1**.

These reads occur at the end. Neither the pointer nor its pointed-to value is specified as an entry-time snapshot.

## Side effects

### Direct writes and reads

The function explicitly writes:

- One zero byte at object offset **0xB0**, only when dispatch is enabled.
- One 16-bit value of 1 at offset **0x80** from the first F-fdd8589eca2d result in each selection branch.
- Four 32-bit record words for every matched action.

The one-byte and 16-bit stores do not establish clearing or replacing adjacent bytes.

Read timing is behaviorally significant:

- Matching-receiver storage is read twice before the artist test, with different consumers.
- Action context is captured after the first matching cleanup.
- Branch payloads and record-source globals are read at their respective use points.
- Record arguments are consumed after the intervening accessor call.
- The selected-status dispatch target is resolved after status construction.
- The final indirect value is loaded during final processing.

Moving these reads to function entry or substituting assigned record values for later loads requires additional preservation assumptions.

### Dependency effects and ownership

Dependencies may change objects, globals, exposed stack storage, or external state. Their bodies are unavailable; this contract supplies no general preservation guarantee.

Each text construction on a normally completing path is followed by F-8a5aadac202f on its primary storage after use. This establishes call pairing, not proof of allocation, destruction, release, or ownership transfer.

Record addresses are exposed to F-78801ca166f2. F-8fa83bbc2366 receives word values, not the record address through those four arguments. No explicit record cleanup is shown.

Stack storage exists within this invocation. Whether dependencies retain its address, and whether any such retention remains valid, is **documented_only**. No ownership or lifetime extension is implied.

## Dependencies

| Opaque dependency | Established use |
|---|---|
| F-dafcc25e1374 | First unconditional call. |
| F-753f887b468d | Supplies the 64-bit result split into halves. |
| F-187151a4b5ba | Receives object, upper half, and lower half. |
| F-5eac81954d4b | Receives lower half and zero. |
| F-9b9c1c7483f4 | Receives primary storage, text reference, and auxiliary storage; returns an argument used by a matcher or callback. |
| F-8094c3d72052 | Supplies each zero/nonzero matching result. |
| F-8a5aadac202f | Receives primary text storage after use. |
| F-fdd8589eca2d | Supplies action-processing values or addresses; also intervenes between record assignments and consumption. |
| F-6a21f4f07f67 | Supplies the first argument to F-aaf1683df92f. |
| F-aaf1683df92f | Receives that result, 128, and an action-associated payload. |
| F-78801ca166f2 | Receives the record address before explicit assignments. |
| F-8fa83bbc2366 | Receives an accessor result, captured context, and four then-current record words. |
| F-c7dac3ee6298 | Receives object and the selection-specific tuple. |
| F-66c31f8db1bf | Receives object and zero for Genius start. |
| F-b002ce492389 | Receives the final indirectly loaded value. |
| Object dispatch entry at 0x11C | Receives object and constructed selected-status argument. |

An ignored return value does not establish that a dependency cannot fail.

## Acceptance examples

These are conditional acceptance criteria, not executed tests. Normal completion of the relevant dependencies is assumed.

1. **Disabled dispatch.** With an enable value of zero, common and final processing occur. No matching construction occurs, and this function performs no explicit byte clear at object offset 0xB0. It returns 1.

2. **Nonzero enable.** A negative enable value takes the same dispatch gate as a positive nonzero value.

3. **Different receiver observations.** If the first matching-receiver read yields one value and the second yields another, the artist test receives the second. If artist misses, later reached tests receive the first. Equality is not required.

4. **Context timing.** If artist-test cleanup changes the action-context global, the subsequent capture observes that changed value. Later changes to that global do not replace the captured context.

5. **Artist precedence.** Any nonzero artist result prevents the other three tests. The artist branch performs its specified writes, uses tuple **17, 0, 1, 4, 2, 0, 0**, and performs status construction, callback, and cleanup.

6. **Album record preservation condition.** After artist misses and album matches, explicit assignment produces **zero, album-associated value, zero, zero**. If the exposed record remains unchanged until consumption, those are the four arguments to F-8fa83bbc2366. Otherwise, the consumed arguments reflect the record contents at their respective reads, not necessarily the assignments.

7. **Record mutation condition.** If F-78801ca166f2 retains access to the record and the subsequent F-fdd8589eca2d changes its second word before consumption, F-8fa83bbc2366 receives that changed word. This criterion does not assert that either dependency actually behaves this way.

8. **Compilation precedence.** If artist and album miss and compilation matches, Genius is not tested. The compilation tuple is **17, 0, 1, 10, 8, 0, 0**.

9. **Distinct accessor results.** If the two selection-branch accessor calls return different addresses, the 16-bit write targets the first; F-8fa83bbc2366 receives the second.

10. **Genius ordering.** After three misses and a Genius match, F-8fa83bbc2366 precedes F-aaf1683df92f and then F-66c31f8db1bf. No selected-status callback occurs.

11. **No match.** Four zero matching results produce four matching constructions and cleanups, no action record processing, final processing, and return 1.

## Uncertainties

The following remain **documented_only**:

- Complete object, dispatch-table, and global-data layouts.
- Exact selected-status text bytes.
- Global payload meanings and dependency-level types.
- Stability of the matching-receiver global between its two reads.
- Dependency retention of record addresses and preservation of record contents.
- Individual argument-load ordering not fixed by the supplied representation.
- Allocation, release, retention, and ownership-transfer rules.
- Dependency failure conventions and abnormal control flow.
- Signed-overflow behavior of the artist context addition.
- Reentrancy, synchronization, and thread safety.

These uncertainties do not erase the established call order, explicit writes, separate receiver reads, or the distinction between record assignment and later consumption.

## Evidence

This contract uses only the supplied static representation and opaque dependency mapping.

Static evidence establishes the dispatch gate, test precedence, two matching-receiver reads and their consumers, context-capture timing, record layout and assignment order, intervening calls, then-current record consumption, explicit access widths, callback lookup placement, final indirect load, and normal return value.

Action text supports the stated intent inference but does not establish hidden dependency semantics. No dependency bodies, complete referenced data, ABI documentation, or runtime observations were supplied.

**No runtime tests have been performed.**
