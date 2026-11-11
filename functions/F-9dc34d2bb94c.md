<!-- Trusted provenance: F-9dc34d2bb94c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9dc34d2bb94c

## Purpose

F-9dc34d2bb94c performs an ordered, cleanup-like transition on an input object, derives another object address through two dependencies, and performs further transitions and conditional operations on that derived object.

Established behavior includes sequential guards, global-backed word replacements, indirect calls, repeated service lookups, conditional processing of an associated state record, and a final conditional cleanup path. Destruction, deregistration, and ownership release are inferred intent, not established effects of the dependencies.

Ordinary completion returns the derived-object address. It need not equal the input address. Guard-failure paths call a guard-specific dependency followed by F-feee9156a257, whose displayed calls are marked non-returning. No ordinary error return is shown.

## Interface

### Input, result, and memory requirements

The function accepts one input-object address. The supplied representation uses 32-bit words and 32-bit address-bearing values.

There is no input-null check. The initial operation obtains a global-backed word and writes it to the input address. Valid readable global storage and writable input storage are therefore required before any guard can reject the input.

On ordinary completion, the result is the derived-object address. Returning that address does not establish that its storage remains allocated or usable after the intervening dependencies.

All subsequently accessed objects, receiver tables, global cells, and local output values must be valid when used. These are path-dependent requirements; no comprehensive validation occurs.

### Input layout

Offsets are byte offsets from the input address.

| Offset | Access width | Observed role |
|---|---:|---|
| `0x000` | 32 bits | Initial global-backed word replacement |
| `0x0A4` | 32 bits | First nonzero guard |
| `0x0C0` | 32-bit address | Optional indirect-call receiver |
| `0x108` | Embedded address | Address passed to F-03144a4f049b |
| `0x1C0` | 32 bits | Second guard, after optional calls |
| `0x1E8` | 32 bits | Later guard |
| `0x1EC` | 32 bits | Later guard |
| `0x1F0` | 32 bits | Last input guard |
| `0x1F4` | 32 bits | Earlier guard in the final group |
| `0x1F8` | 32 bits | Earlier guard in the final group |
| `0x1FC` | 32 bits | Earlier guard in the final group |

The highest directly accessed input field ends at byte `0x1FF`. This does not establish the complete object size.

### Derived object and associated state

| Derived-object offset | Access width | Observed role |
|---|---:|---|
| `0x00` | 32 bits | Three successive global-backed word replacements |
| `0x04` | 32-bit address | Conditional final receiver |
| `0x08` | 32 bits | Conditional final guard |
| `0x0C` | 32-bit address | Associated state record |

The associated state record supplies a 32-bit token at offset `0x04` and a 32-bit address-bearing field at offset `0x08`.

The iteration path passes an address of 20 bytes of local context storage and an address of a 32-bit output-address slot to dependencies. The supplied body does not establish their initialization guarantees.

### Global sources

The following labels distinguish sources; they are not dependency IDs or claims about semantic type:

| Source label | Observed use |
|---|---|
| **Input-word source** | Supplies the initial 32-bit input-word replacement |
| **First derived-word source** | Supplies the first 32-bit derived-word replacement |
| **Second derived-word source** | Supplies the second derived-word replacement |
| **Third derived-word source** | Supplies the final derived-word replacement |
| **Notification-value source** | Supplies a 32-bit argument to F-77781c8e0ff9 |
| **Shared-cell locator** | Supplies an address through which a shared receiver value is read |

The shared-cell locator and the receiver value stored in the located cell are distinct values. A null receiver skips its call; a null or invalid locator is not checked before dereferencing it.

No definitions establish that any of these global sources is immutable. The word and notification values are read at their respective use sites, not captured together at entry.

### Indirect dispatch

Indirect calls read a table address from the receiver’s first word and obtain a target at table byte offset `0x04`, `0x48`, `0x7C`, or `0x80`.

The `0x80` and `0x7C` calls display explicit receiver arguments. Other indirect calls do not display explicit arguments; that does not establish an argument-free ABI or exclude hidden receiver passing.

## Behavior

### Input transition and ordered guards

The function reads the input-word source and writes that value to input offset `0x00`. It then reads input offset `0x0A4`. A nonzero value calls F-54385ab29a1f and, if that call returns, F-feee9156a257. No ordinary continuation is shown.

When the first guard is zero:

- The receiver field at input offset `0x0C0` is tested. If nonnull, the receiver expression is read for dispatch through table offset `0x04`.
- Next, the shared-cell locator is read and dereferenced to test the receiver stored in the located cell. If nonnull, dispatch again obtains the receiver through that global-backed expression and calls table offset `0x04`.

The representation does not save the tested shared receiver for the subsequent dispatch. It does not establish that the locator or located receiver remains unchanged between those accesses. The analogous test and dispatch of input offset `0x0C0` also must not be treated as a validated, permanently saved receiver.

After both optional calls, the function reads input offset `0x1C0`. A nonzero value calls F-282eebd7c413 and then F-feee9156a257.

Next it checks offsets `0x1F4`, `0x1F8`, `0x1FC`, `0x1E8`, `0x1EC`, and `0x1F0`, in that order. Each nonzero value calls F-163ea045cd81 and then F-feee9156a257. Only zero at every reached guard permits derivation.

These are sequential reads. Calls before a guard may change its field. In particular, a mutation made by the first optional call need not survive the second optional call. No direct writes clear the guarded fields or optional receiver fields.

### Derivation and first derived-object transition

F-03144a4f049b receives the address at input offset `0x108`. Its result is reduced by `0x44` bytes and passed to F-282eebd7c413. That result is reduced by `0xC4` bytes to obtain the derived-object address.

These are byte displacements, not counts of 32-bit words. No null, alignment, bounds, or arithmetic-underflow check is shown. The supplied 32-bit address-like arithmetic does not establish safe behavior for invalid or wrapping results.

After both dependency calls, the function reads the first derived-word source and writes its current value to derived offset `0x00`.

### Conditional back-reference operation

F-d277584d672d receives the derived object. If it returns a nonnull receiver, the function calls that receiver’s table entry at `0x80`, explicitly passing the receiver.

If the returned address equals the derived-object address, the function calls the saved receiver’s table entry at `0x7C`, passing that receiver and zero. The table is read again for this second dispatch; the earlier call is not known to preserve it. The saved receiver must remain valid if the second call is reached.

F-a6fdc7050c77 then receives the derived object, zero, and one, whether the receiver was absent, mismatched, or matched, provided preceding calls return normally.

### Second transition and notification

After F-a6fdc7050c77 returns, the function reads the second derived-word source and writes its current value to derived offset `0x00`. F-408b56b8b7c8 then receives the derived object.

The function next calls F-3fdd14ecd250. For the subsequent F-77781c8e0ff9 call, it obtains the 32-bit field at offset `0x1C` of that result and reads the notification-value source. These two values and the derived-object address are passed to F-77781c8e0ff9.

Both argument reads occur after that service lookup and before notification. Their relative machine-level evaluation order is not established here. The notification argument is not an entry-time constant: preceding dependencies may affect its global source.

Only after notification returns does the function read derived offset `0x0C`.

### Associated-state phase

A null associated-state address skips this entire phase. Otherwise, the function saves that address and captures its offset-`0x04` value as the **saved token**. Later fresh reads do not replace the saved address or token.

#### Conditional active-state operations

F-a1a48bd3fbde receives the derived object. A zero result skips this subsection.

For a nonzero result, F-3fdd14ecd250 is called and its offset-`0x28` field is tested. If nonzero:

- A fresh F-3fdd14ecd250 result supplies the offset-`0x28` receiver for dispatch through table offset `0x48`.
- After that call, derived offset `0x0C` is read again, and offset `0x04` of that current record is read.
- Another F-3fdd14ecd250 result supplies its offset-`0x28` field to F-a5514b9f24f7, together with the freshly read record value.

The nonzero check does not validate receivers from later service lookups. The fresh record value need not equal the saved token, and the current associated record need not equal the saved record.

Still within the nonzero F-a1a48bd3fbde branch, F-65528ded3450 receives the derived object and zero, then F-a82ec4b43e2f receives the saved token. These calls occur even if the tested service field was zero.

#### Iteration and source-receiver call

The function next tests offset `0x08` of the saved record. If zero, it skips the iteration, context-finalization call, and post-iteration receiver call.

If nonzero, that field is read again for a call to F-538044b67754. The other displayed arguments are the local context address and the 32-bit pattern `0xFFFFFFFD`. This pattern represents −3 under signed 32-bit interpretation; its semantic role is unknown.

The function repeatedly calls F-ebd94610a438 with the context address and the output-slot address:

- A zero result ends iteration without inspecting the output slot for that result.
- A nonzero result is followed by a read of the output slot.
- A null output receiver skips dispatch but does not end iteration.
- A nonnull output receiver is dispatched through table offset `0x04`.

Passing these addresses does not prove that either dependency initializes the context or writes a fresh output value. The caller performs no visible initialization of the output slot before the loop. A defined value is required whenever the caller reads it; dependency guarantees supporting that requirement remain unknown.

After a zero iterator result, F-944b595e25e9 receives the context address. The function then tests saved-record offset `0x08` again. If nonnull, it reads that receiver expression for a table-offset-`0x04` call. This need not be the original iteration source. No direct write clears the field.

#### Shared-reference reconciliation and record handoff

F-3fdd14ecd250 is called. If its offset-`0x28` field equals the derived-object address, F-a82ec4b43e2f receives the saved token.

A separate F-3fdd14ecd250 call supplies an offset-`0x24` comparison. If that field equals the derived-object address, another service lookup occurs, and zero is written to offset `0x24` of this latest result. The compared object and written object need not be identical.

Finally, F-65528ded3450 receives the derived object and zero, followed by F-8349c252c168 with the saved record address and zero.

Thus F-65528ded3450 and F-a82ec4b43e2f can each occur twice during one invocation.

### Final transition, conditional guard, and return

After the associated-state phase, or immediately after skipping it, the function reads the third derived-word source and writes its current value to derived offset `0x00`. This read follows all reached state-processing dependencies; it is not an earlier cached value.

Derived offset `0x04` is then tested:

- If zero, the function returns the derived-object address without reading offset `0x08`.
- If nonzero, F-c719314a16bb receives the derived object. Offset `0x04` is tested again; if still nonnull, its receiver is obtained for dispatch through table offset `0x04`.
- After that optional dispatch, derived offset `0x08` is read. Nonzero calls F-c8eefdf96f4b followed by F-feee9156a257. Zero permits return.

The final guard remains reached even if F-c719314a16bb clears the receiver. It is not reached when the first offset-`0x04` test is zero.

## Side effects

### Established direct writes

The visible externally addressed writes are:

- One input-word replacement using the input-word source.
- Three derived-word replacements, each using its distinct global source when its phase is reached.
- A conditional zero write at offset `0x24` of a freshly obtained service object.

These sources are not established constants, dispatch tables, or lifecycle markers. Those interpretations remain inferred.

Local bookkeeping also occurs, and addresses of local context and output storage are passed to dependencies. Dependency writes, initialization, and freshness guarantees are not established by those calls.

### Ownership, lifetime, and failure

No dependency may be assumed to preserve global sources, object fields, service identities, receiver tables, or saved storage.

Table-offset-`0x04` calls may release, destroy, notify, or do something else. F-8349c252c168 may or may not deallocate the saved record. Allocation provenance, ownership transfer, and reference-count effects are unknown.

Every later access requires its target to remain valid despite earlier calls. No rollback or compensating action is shown if a dependency terminates, fails to return, or invalidates required storage. Iteration termination is also dependency-controlled; no bound is shown.

## Dependencies

| Opaque ID | Established role |
|---|---|
| F-54385ab29a1f | First input-guard failure call |
| F-feee9156a257 | Marked non-returning on displayed guard-failure paths |
| F-282eebd7c413 | `0x1C0` failure call and address-derivation dependency |
| F-163ea045cd81 | Failure call for six later input guards |
| F-03144a4f049b | Receives input embedded-storage address |
| F-d277584d672d | Returns optional back-reference receiver |
| F-a6fdc7050c77 | Receives derived object, zero, and one |
| F-408b56b8b7c8 | Receives derived object after second word replacement |
| F-3fdd14ecd250 | Repeatedly supplies addresses for service-field accesses |
| F-77781c8e0ff9 | Receives service field, current global-backed notification value, and derived object |
| F-a1a48bd3fbde | Controls active-state branch |
| F-a5514b9f24f7 | Receives freshly fetched service field and freshly read record value |
| F-65528ded3450 | Receives derived object and zero |
| F-a82ec4b43e2f | Receives saved token |
| F-538044b67754 | Receives context address, current source field, and `0xFFFFFFFD`; initialization unknown |
| F-ebd94610a438 | Receives context/output addresses; result controls continuation; output writes unknown |
| F-944b595e25e9 | Receives context after iteration ends; effects unknown |
| F-8349c252c168 | Receives saved record and zero |
| F-c719314a16bb | Precedes final receiver recheck |
| F-c8eefdf96f4b | Final derived-guard failure call |

Indirect targets have no supplied opaque IDs. Their behavior is documented_only.

## Acceptance examples

These are conditional static examples, not executed tests. Required accesses must remain valid and preceding calls must return normally unless stated otherwise.

1. **First guard failure.** A nonzero input field at `0x0A4` causes the initial global-backed write, F-54385ab29a1f, and F-feee9156a257. Neither optional receiver call is reached.

2. **Mutation must survive until observation.** If the first optional receiver call sets input `0x1F4` nonzero, failure at that guard follows only if preceding guards permit progress and the field is nonzero when actually read. The second optional call may clear it before then.

3. **Distinct shared indirection.** A valid shared-cell locator pointing to a cell containing zero skips shared dispatch. A zero locator is not equivalent: dereferencing it is not protected by a locator-null check.

4. **Late global-backed value.** If F-a6fdc7050c77 changes the second derived-word source, the following replacement uses its value at that later read, not its entry value.

5. **Back-reference mismatch.** A nonnull receiver whose `0x80` call returns another address does not receive the `0x7C` call. F-a6fdc7050c77 still follows.

6. **No associated state.** A null derived `0x0C` after notification skips token handling, iteration, and F-8349c252c168. The final global-backed word replacement still occurs.

7. **Iterator output requirements.** A nonzero iterator result with a defined null output value skips dispatch and continues. A nonzero result alone does not establish a fresh output value. A zero result ends iteration without reading that output.

8. **Initially absent iteration source.** Zero at the saved record’s first `0x08` test skips F-538044b67754, F-ebd94610a438, F-944b595e25e9, and the source-receiver call. Shared-reference reconciliation still follows.

9. **Repeated token handling.** A nonzero active-state result and a later matching service `0x28` comparison cause two calls to F-a82ec4b43e2f with the same saved token.

10. **Changed final receiver.** If the first derived `0x04` test is nonzero but F-c719314a16bb clears it, receiver dispatch is skipped while the `0x08` guard is still evaluated.

11. **Conditional exit guard.** If the first derived `0x04` test is zero, even a nonzero `0x08` does not select the displayed final failure path; that guard is not read.

## Uncertainties

- **documented_only:** Dependency bodies, indirect target identities, hidden arguments, and complete calling conventions.
- **documented_only:** Global-source definitions, numeric values, mutability rules, and semantic types.
- **documented_only:** Context initialization, output-slot initialization, and whether any iterator result guarantees a fresh output write.
- **documented_only:** Complete object layouts, byte order, alignment, and arithmetic behavior for invalid or wrapping addresses.
- **documented_only:** Ownership, deallocation, synchronization, reentrancy, and concurrent-mutation rules.
- **documented_only:** Whether F-feee9156a257 terminates, traps, throws, or otherwise transfers control. Its displayed calls are marked non-returning.
- **Inferred intent:** Word replacements and ordered cleanup-like calls resemble layered destruction; language-level destructor semantics are not established.
- **Established constraint:** Saved addresses and tokens coexist with fresh reads. Repeated lookups and global reads cannot be collapsed into entry-time snapshots.

## Evidence

This contract uses the supplied static function representation and opaque dependency map. Established claims concern visible access widths and offsets, global-backed reads, direct writes, branch conditions, call ordering, displayed arguments, address comparisons, and the ordinary returned address.

Dependency effects are not inferred from names or call placement. No dependency bodies, runtime observations, or runtime tests were provided or performed. Acceptance examples express conditional consequences of the visible control flow, not empirical results.
