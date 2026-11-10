<!-- Trusted provenance: F-d1dfcd96c031; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d1dfcd96c031

## Purpose

F-d1dfcd96c031 performs an ordered sequence of object-word changes, reference-count-like adjustment, indirect callbacks, collection traversal, and shared-state cleanup. On normal completion, it returns a derived object address, not necessarily the input address.

Teardown is the inferred intent. The established behavior is the sequence of accesses and calls described below. Neither cleanup-looking dependency names nor indirect dispatch slots establish destruction, deallocation, or ownership transfer.

Four object-word writes obtain their values from four separate 32-bit global words. Another 32-bit global word supplies an argument to F-77781c8e0ff9. These are reads at distinct execution points, not established immutable constants. Intervening dependencies may change values that have not yet been read.

Some branches reach F-feee9156a257, identified as non-returning in the supplied representation. There is no separate success or failure result.

## Interface

**Input:** one address identifying writable object storage.

**Normal output:** a derived address designated **View** below.

**Representation:**

- Address-bearing words and scalar fields are 32 bits in the supplied representation.
- All offsets in this contract are byte offsets.
- Address comparisons compare pointer values, not object contents.
- Indirect dispatch obtains a table address from an object's first word, then obtains a callable target at the specified byte offset within that table.
- Local traversal storage comprises 20 bytes of iterator state and two address-sized output entries. Only the first output entry is directly inspected.
- The iterator control argument is the 32-bit pattern `0xFFFFFFFD`, interpreted as −3 if signed.

The following descriptive names identify retained addresses or values, not implementation identifiers:

| Name | Definition |
|---|---|
| **Input** | Caller-supplied address. |
| **Anchor** | Result of F-89958901d617 called with Input + 132. |
| **Link** | Address Anchor − 4; its contents identify an optional referenced object. |
| **Stage** | Result of the unidentified dependency called with Anchor − 16. |
| **Pivot** | Result of F-b7ec5c8c9d08 called with Stage − 76. |
| **View** | Pivot − 40; normal return value. |
| **Record** | Nonzero address read from Pivot − 28 and retained for the Record cleanup block. |
| **Saved token** | Word read from Record + 4 before conditional registration-related cleanup. |

Five separate global words are distinguished by their use:

| Global word | Read point and use |
|---|---|
| **Entry object-word source** | Read before F-8dd08d4a2463; copied to Input's first word. |
| **Intermediate object-word source** | Read after the unidentified dependency returns; copied to Stage − 116. |
| **First-view object-word source** | Read after F-b7ec5c8c9d08 returns; copied to View's first word before F-408b56b8b7c8. |
| **Registration argument source** | Read after the following F-3fdd14ecd250 call; supplied as F-77781c8e0ff9's second argument. |
| **Final-view object-word source** | Read after the optional Record cleanup block; copied to View's first word. |

These names distinguish storage locations and read timing. They do not establish the words' values, semantic types, immutability, or equality with one another.

**Preconditions:**

- Input permits its initial 32-bit write and the subsequent dependency call.
- Every reached dependency-produced address permits the accesses made through it.
- Objects used for indirect dispatch provide readable table pointers and valid callable targets at the selected slots.
- Retained addresses remain usable through subsequent calls that may affect their storage. This routine does not generally validate continued validity.
- The global words are readable when their respective execution points are reached.

No explicit null-input guard, synchronization, error recovery, or allocation-failure handling is visible. Address adjustments are established by their offsets; valid storage relationships and behavior on address overflow are not established.

## Behavior

### 1. Entry transition

The routine reads the 32-bit Entry object-word source and writes that value to Input's first word. It then calls F-8dd08d4a2463 with Input.

After that call returns, it calls F-89958901d617 with Input + 132, obtaining Anchor.

The entry value is sampled before F-8dd08d4a2463. No preservation of Input, globals, or other state by that call is established.

### 2. Referenced-object adjustment

The routine reads the address at Link. If zero, it skips both the count-like adjustment and the explicit Link-clearing write.

For a nonzero referenced-object address, it reads the 32-bit field at that object's offset +4, subtracts one, and writes the result back.

If the result is nonzero, it clears Link and continues.

If the result is zero:

1. It reloads the referenced-object address from Link and reads that object's first word.
2. If that first word is nonzero, it calls F-1b86b75ed2c5, followed by F-feee9156a257.
3. Otherwise, it rereads Link. If Link is nonzero, it calls F-feee9156a257.
4. If execution reaches the common continuation, it clears Link.

Under unchanged Link contents and ordinary memory behavior, a nonzero referenced object whose adjusted field becomes zero therefore reaches a non-returning call. A nonzero first word adds F-1b86b75ed2c5 before that call. The Link-clearing write is not performed before these calls.

The adjustment is 32 bits wide, with no visible underflow guard. At the machine-width level, decrementing zero produces `0xFFFFFFFF`, which is nonzero. Language-level signed-overflow semantics at other boundaries are not independently established.

### 3. Intermediate cleanup

The routine calls the unidentified dependency with Anchor − 16 and retains its result as Stage.

Only after that call returns does it read the Intermediate object-word source and write the sampled 32-bit value to Stage − 116. It does not use an earlier snapshot of this global.

It then processes two object-address fields in order:

- At Stage − 92, a nonzero field causes an indirect call through dispatch slot +4.
- At Stage − 88, a nonzero field causes an indirect call through dispatch slot +40. After that call returns, the routine rereads Stage − 88. If nonzero, it calls dispatch slot +92 on the object then stored there.

The +92 dispatch is not necessarily on the same object as the +40 dispatch. A callback that clears the field suppresses the later dispatch; replacement redirects it.

The representation supplies no explicit argument list for these indirect calls. Receiver conventions and ownership effects remain unresolved. Field tests and subsequent dispatch loads do not establish atomic snapshots.

### 4. Transition to View and registration-related call

The routine calls F-b7ec5c8c9d08 with Stage − 76, obtaining Pivot, and derives View as Pivot − 40.

After F-b7ec5c8c9d08 returns, it reads the First-view object-word source and writes that 32-bit value to View's first word. It then calls F-408b56b8b7c8 with View.

Next it calls F-3fdd14ecd250. Using the returned address, it reads the field at +28. It also reads the Registration argument source, after this accessor call has returned. It calls F-77781c8e0ff9 with:

1. The returned object's +28 field value.
2. The sampled Registration argument source value.
3. View.

The relative evaluation order of the two argument-source reads is not established here; both occur after F-3fdd14ecd250 and before F-77781c8e0ff9. Earlier calls may have changed the registration global.

Results of F-3fdd14ecd250 are called **context results** below. This terminology does not imply a stable singleton.

After F-77781c8e0ff9 returns, the routine reads Pivot − 28. A zero value skips the entire Record cleanup block.

### 5. Conditional Record cleanup

For a nonzero Pivot − 28 field, the routine retains its value as Record and saves the 32-bit word at Record + 4 as Saved token.

It calls F-a1a48bd3fbde with View. A nonzero result enables this sequence:

1. Obtain a context result and test its +40 field.
2. If that field is nonzero:
   - Obtain another context result.
   - Invoke dispatch slot +72 on the object found through this new result's +40 field.
   - After the indirect call, reload Pivot − 28 and read the word at +4 of the object now referenced there.
   - Obtain another context result.
   - Call F-a5514b9f24f7 with this latest result's +40 field and the freshly read word.
3. Call F-65528ded3450 with View and zero.
4. Call F-a82ec4b43e2f with Saved token.

The initial +40 test does not prove that the next context result has a nonzero +40 field. There is no second explicit null guard before the +72 dispatch.

The word passed to F-a5514b9f24f7 need not equal Saved token. Its address is reloaded through Pivot − 28 after a callback. Subsequent Record-relative accesses instead continue to use the retained Record address.

### 6. Collection traversal

Within the Record block, the routine tests Record + 8. If nonzero, it:

1. Calls F-538044b67754 with the local iterator state, the Record + 8 field value read for initialization, and `0xFFFFFFFD`.
2. Repeatedly calls F-ebd94610a438 with the iterator state and two-entry output area.
3. For each nonzero return, examines the first output entry. A nonzero entry causes an indirect call through that object's dispatch slot +4.
4. Stops iteration on a zero return.
5. Calls F-944b595e25e9 with the iterator state.
6. Rereads Record + 8 and, if nonzero, invokes dispatch slot +4 on the object currently stored there.

The second output entry has no established meaning in this routine. Duplicate nonzero yielded addresses cause repeated dispatches; there is no visible deduplication.

The final collection dispatch may target a replacement rather than the object used to initialize iteration. No iterator-finalization guarantee is visible if initialization, advancement, or a callback exits abnormally.

### 7. Shared-state checks and final Record calls

After optional traversal, the routine obtains a new context result. If its +40 field equals View, it calls F-a82ec4b43e2f with Saved token.

This check is independent of the earlier F-a1a48bd3fbde result. Consequently, F-a82ec4b43e2f may receive Saved token twice during one invocation.

The routine obtains another context result and compares its +36 field with View. If equal, it obtains yet another context result and writes a 32-bit zero to that latest result's +36 field.

Thus, the context receiving the zero write need not be the context whose field matched View.

The Record block ends with:

1. F-65528ded3450 called with View and zero.
2. F-8349c252c168 called with the retained Record address and zero.

F-65528ded3450 may be called twice with View and zero. The routine does not itself clear Pivot − 28.

### 8. Final object-word transition and auxiliary cleanup

After the Record block completes, or immediately after it is skipped, the routine reads the Final-view object-word source and writes that sampled 32-bit value to View's first word.

This is a late global read. It is not established to equal any earlier value, and Record-related dependencies may have changed it before it is sampled.

The routine then tests Pivot − 36. If nonzero:

1. Call F-c719314a16bb with View.
2. Reread Pivot − 36; if nonzero, invoke dispatch slot +4 on the object currently stored there.
3. Test Pivot − 32.
4. If nonzero, call F-c8eefdf96f4b, then F-feee9156a257.

The Pivot − 32 test is nested within the initial nonzero Pivot − 36 condition. A nonzero Pivot − 32 alone does not trigger these calls.

Every normal completion returns View.

## Side effects

The directly established object and shared-state writes are:

- Input's first word receives the sampled Entry object-word source.
- A reached referenced object's +4 field receives its 32-bit decrement.
- Link is cleared on paths reaching that write.
- Stage − 116 receives the sampled Intermediate object-word source.
- View's first word receives the First-view object-word source, then later the Final-view object-word source.
- A newly acquired context result's +36 field receives zero when the preceding comparison succeeds.

There are five ordered global-data sampling points: entry object word, intermediate object word, first View object word, registration argument, and final View object word. The routine directly reads, rather than writes, these global words. Their storage is separate; their values and immutability are unestablished.

Dependencies can modify iterator storage and output entries. They may also modify globals, fields, registrations, or memory lifetimes. No general preservation guarantee is supplied.

Dispatch slot +4 does not itself prove deallocation. Neither F-8349c252c168 nor iterator finalization establishes release of its argument. No rollback, thread-safety guarantee, or ownership transfer is visible.

## Dependencies

| Dependency | Established use |
|---|---|
| F-8dd08d4a2463 | Receives Input after the Entry object-word source has been read and written there. |
| F-89958901d617 | Receives Input + 132; returns Anchor. |
| F-1b86b75ed2c5 | Called on one zero-adjusted-count branch before F-feee9156a257. |
| F-feee9156a257 | Identified as non-returning; reached in count and final auxiliary branches. |
| Unidentified dependency; opaque ID unavailable | Receives Anchor − 16; returns Stage before the Intermediate object-word source is read. |
| F-b7ec5c8c9d08 | Receives Stage − 76; returns Pivot before the First-view object-word source is read. |
| F-408b56b8b7c8 | Receives View after its first global-sourced object-word write. |
| F-3fdd14ecd250 | Supplies repeatedly reacquired context results; its first call precedes the Registration argument source read. |
| F-77781c8e0ff9 | Receives context +28, the sampled registration global word, and View. |
| F-a1a48bd3fbde | Gates the early registration-related sequence. |
| F-a5514b9f24f7 | Receives a context +40 value and a freshly reloaded Record-associated word. |
| F-65528ded3450 | Receives View and zero, potentially twice. |
| F-a82ec4b43e2f | Receives Saved token, potentially twice. |
| F-538044b67754 | Initializes iteration with local storage, a collection address, and `0xFFFFFFFD`. |
| F-ebd94610a438 | Advances iteration and supplies output entries. |
| F-944b595e25e9 | Finalizes iterator state after normal traversal. |
| F-8349c252c168 | Receives retained Record and zero before the final global sampling point. |
| F-c719314a16bb | Receives View after the final global-sourced object-word write. |
| F-c8eefdf96f4b | Precedes F-feee9156a257 in the nested auxiliary branch. |

Indirect dependencies occupy dispatch slots +4, +40, +72, and +92. Their identities and effects are unknown. No listed dependency is assumed to preserve global words.

## Acceptance examples

These are conditional criteria, not executed tests. Unless stated otherwise, reached calls return normally and accessed storage remains valid.

1. **Minimal optional state.** If Link, Pivot − 28, and Pivot − 36 are zero when tested, count adjustment, Record cleanup, and auxiliary cleanup are skipped. Intermediate processing still occurs. All five global reads occur in the specified order, and the routine returns View.

2. **Global changes before later sampling.** If the unidentified dependency changes the Intermediate object-word source before returning, Stage − 116 receives the changed value. An entry-time snapshot would not satisfy this behavior.

3. **Registration global changed by accessor.** If the first F-3fdd14ecd250 call changes the Registration argument source, F-77781c8e0ff9 receives its subsequently read value, not an assumed constant or earlier snapshot.

4. **Late final value.** If Record cleanup changes the Final-view object-word source, the final View write uses the value read after that cleanup. It need not match the first View write.

5. **Count remains nonzero.** With a stable Link and an initial referenced count of 2, the routine writes 1 and clears Link without either count-branch non-returning call.

6. **Count becomes zero.** With a stable Link and count 1, it writes zero and reaches F-feee9156a257. A nonzero referenced first word causes F-1b86b75ed2c5 first. Link is not cleared before these calls.

7. **Intermediate field cleared.** If the +40 callback clears Stage − 88, no +92 callback follows. If it replaces the field with another valid object, the later dispatch uses that object.

8. **Early Record cleanup disabled.** A zero F-a1a48bd3fbde result skips the early F-65528ded3450 and token call, but does not skip traversal, later context checks, or the final Record calls.

9. **Repeated token action.** A nonzero F-a1a48bd3fbde result plus a later context +40 equal to View causes two F-a82ec4b43e2f calls with Saved token.

10. **Iterator outputs.** A nonzero continuation with a null first output causes no dispatch. Repeated nonnull addresses cause repeated +4 dispatches. A zero continuation ends traversal without processing that output.

11. **Changing context identity.** If the +36 comparison succeeds on one context and the next accessor returns another, the zero write targets the latter's +36 field.

12. **Nested auxiliary condition.** An initially zero Pivot − 36 suppresses the Pivot − 32 test. If the outer condition is nonzero and Pivot − 32 is nonzero when reached, F-c8eefdf96f4b precedes F-feee9156a257.

## Uncertainties

**Inferred intent:** teardown, reference-count maintenance, collection cleanup, registration removal, and dispatch-state transitions. These interpretations do not add ownership or dependency guarantees.

**documented_only:**

- Exact values and semantic meanings of the five global words.
- Whether any global word is immutable in the larger program; the supplied function does not establish immutability.
- Relative evaluation order of the context +28 read and registration-global read used for the same call.
- Identity and behavior of the dependency whose opaque ID is unavailable.
- Allocation boundaries, overlap among derived object views, and relationships to global storage.
- Ownership, deallocation, and implicit receiver conventions.
- Context stability and effects of callbacks on retained addresses.
- Reentrancy, concurrent mutation, atomicity, exceptions, and abnormal-exit cleanup.
- Whether non-returning calls terminate, throw, or transfer control otherwise.
- Signed-overflow and address-overflow semantics beyond the represented widths and operations.

## Evidence

Established claims derive from the supplied static function representation and dependency-ID mapping: access widths, offsets, five distinct global-data reads, their placement relative to calls, direct writes, retained values, rereads, branches, dispatch slots, and normal return value.

Dependency implementations were not supplied. Prior prose and reviewer feedback are not independent evidence of dependency behavior.

No runtime tests have been performed. Acceptance examples are conditional behavioral requirements, not observed execution results.
