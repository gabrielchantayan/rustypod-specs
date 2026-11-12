<!-- Trusted provenance: F-b1d1c0bd30a2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b1d1c0bd30a2

## Purpose

F-b1d1c0bd30a2 conditionally processes a selected parent record and its referenced child object. It checks eligibility, invokes dependencies on selected-record values, optionally grows parent storage, issues record-addressed external calls, reconciles auxiliary state, and explicitly transfers certain child fields to the parent.

The call pattern suggests flattening a child collection into a parent collection. That intent is **inferred**, not an established end-to-end result. In particular, two unmapped external dependencies receive record addresses and the numeric argument 48; their implementations are unavailable. Their calls do not, by themselves, establish that records are copied or cleared.

The operation is not visibly transactional. Explicit stores and dependency calls precede a possible growth failure. Return value 1 covers both the completed path and that early failure path.

## Interface

The interface has five arguments and a 32-bit result:

| Argument | Established role |
|---|---|
| Context | Opaque 32-bit value passed to dependencies. |
| Parent | Nullable object pointer. |
| Record index | Signed 32-bit index selecting a parent record. |
| Mode A | 32-bit integer distinguished only as zero or nonzero. |
| Mode B | 32-bit integer distinguished only as zero or nonzero. |

Results, assuming the invoked dependencies return normally:

- **0:** null parent or a failed eligibility check.
- **1:** completed processing, or a null result from a parent-growth dependency after mutation has begun.

There is no distinct visible growth-error result.

### Widths and layouts

All offsets are bytes. Words and pointer-bearing fields occupy 32 bits in the supplied representation. Collection counts are signed 16-bit values. Flag accesses are 8-bit accesses.

Relevant object fields:

| Offset | Parent use | Child use |
|---|---|---|
| +0 | Root object; later reconciliation input | Reconciliation input |
| +5 | Eligibility byte; final Boolean destination | Eligibility and final-combination byte |
| +9 | Eligibility byte | Not accessed |
| +12 | Collection pointer | Collection pointer |
| +16 | Primary auxiliary field | Eligibility and auxiliary-selection field |
| +20 | Secondary auxiliary field | Dependency input |
| +24 | Tertiary auxiliary field | Dependency input |
| +28 | Associated-state field | Associated-state field |
| +40 | Eligibility field | Not accessed |
| +44 | Conflict and final-transfer field | Conflict and final-transfer field |
| +48 | Not accessed | Must be zero during eligibility |

A collection has a signed 16-bit count at +0. Record addresses begin at +8 with a 48-byte stride. Record index \(j\) therefore identifies the address at collection base \(+8+48j\).

Within the selected parent record:

| Record-relative offset | Use |
|---|---|
| +0, +4, +8 | Dependency arguments, subsequently explicitly zeroed |
| +12 | Dependency argument, subsequently explicitly zeroed |
| +16 | Child object pointer |
| +21 | Eligibility flag byte; subsequently saved and restored |
| +24 | Saved reconciliation token |

The flag byte at +21 is separate from the child-pointer field at +16.

The root object has a signed 32-bit count at +0 and a table pointer at +12. Table entries have a 12-byte stride; entry +0 is a referenced-object pointer, and entry +4 is a result field. Referenced objects supply words at +28 and +32.

### Preconditions

Except for checking whether the parent is null, the function does not establish pointer validity. It does not check index bounds, reject negative indices, validate allocation extents, or establish alignment or non-aliasing.

A non-null parent causes the parent collection and selected child pointer to be obtained before mode rejection. Consequently, even a request rejected because both modes are nonzero requires the preceding collection and selected-record accesses to be valid.

Address calculations use the represented 32-bit address and integer operations. A defined-use contract requires representable address calculations and valid accessible storage; overflow behavior is **documented_only**. Negative counts are not uniformly rejected, but that does not establish that they are supported collection states.

## Behavior

### Eligibility and captured references

A null parent returns 0 immediately.

Otherwise, the function captures the parent collection pointer and obtains the child pointer from the selected record. It then performs these checks in order. Rejection invokes no dependency and performs no explicit store.

1. **Mode compatibility**
   - Nonzero Mode A with nonzero Mode B rejects.
   - Zero Mode A with nonzero Mode B rejects when the signed parent count is at least 2.
   - Other combinations impose no restriction at this step.

   The count is sign-extended from 16 bits. Its subtraction by 1 is representable in signed 32-bit arithmetic. Counts of 1, 0, and negative values pass this particular count restriction.

2. **Conflicting +44 fields**
   - Read child +44 and capture the child collection pointer.
   - If the read child +44 is nonzero, read parent +44 and reject if it is nonzero.

3. **Child +48**
   - Reject if child +48 is nonzero.

4. **Parent +40 conflict**
   - Read parent +40.
   - If it and the captured child +44 are both nonzero, read child +28 and reject if it is nonzero.

5. **Child count**
   - Capture the signed 16-bit child count.
   - Reject if it equals zero.
   - This captured count controls later growth and record-call loops; those loops do not refresh it.

6. **Child flag or +44**
   - Read child byte +5.
   - If it is nonzero, or the captured child +44 is nonzero, read the signed parent count.
   - A parent count at least 2 rejects.
   - A parent count less than 2 passes this step only when Mode A is zero.

7. **Parent flag with Mode B**
   - Read parent byte +5.
   - Reject if it and Mode B are both nonzero.

8. **Associated-state conflict**
   - Read parent byte +9.
   - Read parent +28 only if that byte is zero.
   - If either the byte is nonzero or the conditionally read field is nonzero, read child +28 and reject if it is nonzero.

9. **Mode A conflict**
   - If Mode A is nonzero, read child +28 and reject if it is nonzero.

10. **Selected-record flag**
    - Read the selected record’s byte at +21.
    - For a child count at least 2, bit 0x20 must be clear.
    - For a child count less than 2, a clear bit passes; a set bit passes only if child +16 is zero.

Child +44 is captured for the earlier checks, but later processing reads that field again. Other repeated field accesses likewise must not be treated as immutable snapshots.

### Selected-record calls and explicit clearing

After eligibility succeeds, the function saves the reconciliation token and then reloads and saves the complete flag byte.

It invokes:

1. F-da1a289a3277 with selected-record +12.
2. F-94cb53877c15 with selected-record +0.
3. F-94cb53877c15 with selected-record +4.
4. F-94cb53877c15 with selected-record +8.

Each argument is read for its respective call, rather than all four being captured before the first call. Thus an earlier dependency can affect a later argument through memory side effects. Zero arguments do not suppress calls.

After all four calls return, the function explicitly zeros the fields in this order: +12, +0, +4, +8. Each word is cleared through two 16-bit stores, lower-addressed half first.

These stores are established. Whether the preceding calls release resources is not.

### Growth and descending record-address calls

Let \(n\) be the captured child count.

If \(n>1\), F-fc0c338242f8 is called \(n-1\) times unless an earlier call returns null. Each call receives the context, the current working parent collection pointer, and two zeros. Its result becomes the next working pointer.

A null result causes an explicit zero store to parent +12 and immediate return 1. No subsequent record-address calls, flag restoration, reconciliation, or final child call occur.

If every growth call returns non-null, the function explicitly stores the final pointer at parent +12. It does not itself publish each intermediate successful pointer there. Dependencies may independently modify that field.

The function then reads the resulting collection’s signed count. Starting at destination index count minus 1 and descending, it calculates a source index equal to destination index minus \(n-1\). While that source index is at least the selected record index, it calls **F-unmapped-01** with:

- Destination: the working parent collection’s record address at the destination index.
- Source: the same collection’s record address at the calculated source index.
- Numeric third argument: 48.

These calls include a source at the selected index when the bounds reach it. They are not restricted to records strictly after that index.

The order resembles an upward record shift that protects later sources, but neither a shift nor any particular memory change is established without the external dependency’s contract. The function also does not independently establish how growth changes the count or storage.

### Ascending child-record calls and flag restoration

For each integer index from zero through \(n-1\), ascending, the function calls:

1. **F-unmapped-01**, with the working parent record address at selected index plus loop index, the captured child collection’s record address at loop index, and numeric argument 48.
2. **F-unmapped-02**, with that child record address and numeric argument 48.

The second call follows the first for each record, before processing the next record.

This sequence suggests a record transfer followed by source clearing. Only the addresses, argument values, and order are established. The function contains no explicit 48-byte payload copy or clear at these points.

After the loop, it explicitly writes the saved flag byte to +21 of the selected record in the working parent collection. This store occurs even if \(n\) is negative. For negative \(n\), growth and both record-call loops are skipped; the earlier four field clears and subsequent processing still occur.

No explicit store changes the child collection count.

### Root-table processing and reload behavior

The function captures the root pointer from parent +0 once for this loop. Each loop condition rereads the signed count from that captured root.

For each index, ascending from zero:

1. Read the captured root’s table-pointer field and inspect the indexed entry’s +4 word.
2. If that word is nonzero, advance without a dependency call or result store.
3. Otherwise, obtain the indexed entry’s +0 pointer through the root’s table-pointer field.
4. Read the referenced object’s +28 word.
5. If it is zero, advance without calling the dependency or storing a result.
6. Otherwise, call F-7129d6a8d912 with the context, that +28 value, and the unsigned 32-bit word at referenced-object +32 logically shifted right by one.
7. **After the dependency returns, reread the captured root’s table-pointer field.** Store the returned word at +4 of the entry at the same loop index in this newly addressed table.

The result destination need not be in the table used for eligibility lookup. If the dependency changes the captured root’s table pointer, the result goes to the replacement table, without a second eligibility check there.

Changing parent +0 does not change the root captured for this loop. Changes to the captured root’s count affect subsequent loop conditions. Later reconciliation separately reads parent +0 again.

### Auxiliary reconciliation

The remaining sequence is:

1. Call F-665f1a441b03 with context, current parent +0, saved token, and current child +0.
2. If Mode A is nonzero:
   - Call F-665f1a441b03 with context, parent +20, saved token, and child +0.
   - Call F-cf8121d269c4 with context, parent +24, saved token, and child +0.
3. Read child +28:
   - If nonzero, assign it to parent +28, then explicitly clear child +28.
   - Otherwise, if parent +28 is nonzero, call F-665f1a441b03. Only the context argument is established for this invocation.
4. Read child +16:
   - If zero, select a new auxiliary value of zero.
   - Otherwise, select the return of F-1a5398bcc6b3. Only the context argument is established for this invocation.

For **zero Mode B**:

- Call F-cf8121d269c4 with context, parent +16, saved token, and child +0.
- Call F-8661793c1ec4 with context, the then-current parent +16, and the selected auxiliary value.
- Store its return at parent +16.

For **nonzero Mode B**:

- Capture parent +16, store it at parent +24, and store the selected auxiliary value at parent +16.
- Call F-cf8121d269c4 with context, the captured former parent +16, saved token, and child +0.
- Call F-1a5398bcc6b3 with context and child +24.
- Call F-8661793c1ec4 with context, current parent +24, and the preceding return.
- Store its return at parent +24.
- Call F-1698e90c5d31 with context and child +20.
- Store its return at parent +20.

Reads following dependencies are not guaranteed to reproduce earlier values.

### Final flag and field updates

Read parent byte +5. Only if it is zero, read child byte +5. Write parent byte +5 as Boolean 1 when either tested byte is nonzero, otherwise 0. A preexisting nonzero parent byte is normalized to 1.

Next, inspect the then-current child +44. If nonzero, assign its value to parent +44 and explicitly clear child +44.

Finally, call F-6c7ac9769d41 with the captured child pointer and return 1.

## Side effects

Established direct writes include:

- Four selected-record word clears, implemented as paired 16-bit stores.
- Parent +12 updates on growth success or failure.
- Restoration of the saved record flag byte.
- Root-table result stores through a post-dependency table-pointer reload.
- Auxiliary parent-field assignments.
- Conditional child +28 and +44 clears.
- Parent byte +5 normalization.

Effects of F-unmapped-01 and F-unmapped-02 are not established. In particular, neither successful record insertion nor cleared child payloads can be promised from this representation alone.

The explicit assign-then-clear operations on child +28 and +44 support an **ownership-transfer inference for those fields**. The record-address call pattern suggests similar intent for payloads but does not prove it.

There is no visible rollback. Growth failure follows dependency calls and explicit clearing. Other dependency results are not visibly checked for failure. Continued use of parent, child, collections, and auxiliary objects requires compatible dependency lifetime behavior; preservation, allocation, release, and global-state effects must not be assumed.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-da1a289a3277 | Receives selected-record +12 before explicit clearing. |
| F-94cb53877c15 | Receives selected-record +0, +4, and +8 in separate ordered calls. |
| F-fc0c338242f8 | Returns working collection pointers; null selects the early return-1 path. |
| F-7129d6a8d912 | Supplies a root-table result; may affect state before destination reload. |
| F-665f1a441b03 | Reconciliation calls, including one with incompletely established arguments. |
| F-cf8121d269c4 | Receives auxiliary values, the saved token, and child +0. |
| F-1a5398bcc6b3 | Supplies auxiliary values; one invocation has incompletely established arguments. |
| F-8661793c1ec4 | Supplies values explicitly stored in parent auxiliary fields. |
| F-1698e90c5d31 | Supplies parent +20 in the nonzero Mode B branch. |
| F-6c7ac9769d41 | Receives the child at the end of the completed path. |
| F-unmapped-01 | Receives destination address, source address, and 48. |
| F-unmapped-02 | Receives child-record address and 48. |

The last two IDs are contract-local opaque labels for distinct unmapped external dependencies, not supplied mappings. Their names do not imply memory semantics.

## Acceptance examples

These are conditional static expectations, not executed tests. Assume valid storage, representable arithmetic, normal dependency returns, and preservation of objects needed by subsequent accesses unless an example specifies another effect.

1. **Null parent:** returns 0 without collection or child accesses.

2. **Both modes nonzero:** returns 0 after obtaining the selected child pointer. No dependency is invoked and no explicit store occurs.

3. **Mode B count boundary:** with Mode A zero and Mode B nonzero, parent count 2 rejects. Count 1 passes this check, subject to later eligibility.

4. **Zero child count:** rejects after preceding eligibility reads. A negative child count is not rejected by this check alone.

5. **Flag boundary:** child count 1 with selected-record bit 0x20 set passes the last check only when child +16 is zero. Child count 2 with that bit set rejects regardless of child +16.

6. **Growth failure:** for child count 2, a null growth result follows the four selected-record calls and explicit clears. Parent +12 becomes zero and the result is 1. Neither unmapped dependency nor later reconciliation is reached.

7. **Two-record call sequence:** if growth succeeds with the expected expanded count, descending parent-address calls precede two ascending pairs of child-record calls. Each pair is F-unmapped-01 followed by F-unmapped-02, with numeric argument 48. Actual copying or clearing is an additional dependency-contract condition, not an acceptance claim established here.

8. **Negative child count:** if all checks pass, neither growth nor record-address calls occur. The four explicit clears, saved-flag store, and remaining full-path processing still occur.

9. **Redirected root-table store:** eligibility finds index 3 in table A with +4 zero and a referenced-object +28 nonzero. F-7129d6a8d912 changes the captured root’s table pointer to table B and returns a value. That value is stored at index 3, +4, in table B—even if that destination was already nonzero. Replacing parent +0 alone does not redirect this store.

10. **Unsigned shift:** a referenced-object +32 value of 5 supplies 2 to F-7129d6a8d912. A value of 0x80000000 supplies 0x40000000.

11. **Final updates:** parent byte +5 equal to 7 becomes 1 without reading child byte +5 for that combination. Nonzero child +44 at the final check is assigned to parent +44 and then cleared before F-6c7ac9769d41.

## Uncertainties

The following remain **documented_only**:

- Application meanings of records, modes, flags, and auxiliary fields.
- Caller invariants for negative counts and indices.
- Implementations and memory effects of both unmapped dependencies, including whether they copy, clear, tolerate overlap, or modify other state.
- Growth allocation, relocation, count updates, and ownership behavior.
- Dependency preservation of globals, reachable objects, and lifetimes.
- Complete arguments for the two incompletely represented invocations.
- Failure conventions beyond the explicitly checked null growth result.
- Whether the final child dependency releases, destroys, or updates the child.
- Behavior under invalid storage, aliasing, concurrent mutation, or overflowing address calculations.
- Byte order and any portable serialization interpretation.

## Evidence

This contract uses static inspection of the supplied function representation and opaque dependency mapping. Prior prose, embedded names, and comments are not treated as independent dependency contracts.

Established evidence covers access widths, offsets, branch conditions, captured references, call arguments and ordering, explicit stores, loop bounds, table-pointer reloads, and return sites. Flattening, release, and ownership interpretations are separately identified as inferences.

No runtime tests have been performed. No external dependency implementation, allocator contract, or caller invariant has been independently verified.
