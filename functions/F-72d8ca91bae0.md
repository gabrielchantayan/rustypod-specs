<!-- Trusted provenance: F-72d8ca91bae0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-72d8ca91bae0

## Purpose

F-72d8ca91bae0 coordinates emission and delegated processing for a node through a mutable context. An emitter obtained from F-9b3ea8470521 gates all subsequent work.

Three node kinds have dedicated behavior:

- `0x40`: assigns an identifier, starts an aggregate-like emission, processes either a direct list or a referenced object, and normally finalizes the aggregate.
- `0x11` and `0x6e`: prepare a descriptor, replace a referenced-object field, and delegate processing.
- Other kinds: perform no kind-specific work.

Preliminary emission can precede any kind. A retained handle can cause a final emitter-table update unless an item consumes it or delegated processing causes an early return.

“Emitter,” “aggregate,” “descriptor,” and “classification” describe observed roles, not established domain types. The contract below distinguishes explicit behavior from interpretations and unresolved dependency semantics.

## Interface

### Arguments and result

| Position | Established use |
|---|---|
| 1 | Address of a mutable context. |
| 2 | Address of a mutable node. |
| 3 | Unused 32-bit argument. |
| 4 | Incoming 32-bit value overwritten on some paths; no established caller-visible effect depends on its initial value. |

The routine has no explicit return value. Its outcomes are dependency calls, memory mutations, and normal or early return.

Pointer-bearing fields and ordinary words are 32 bits in the supplied representation. This does not establish a portable 64-bit layout.

### Context layout

Offsets are byte offsets from argument 1.

| Offset | Width | Use |
|---|---:|---|
| `0x44` | 32 bits | Identifier counter for kind `0x40`. |
| `0x48` | 32 bits | Incremented for preliminary emission and independently for kinds `0x11` and `0x6e`. |
| `0x54` | 32 bits | Incremented and decremented around each direct-list item-processing call. |
| `0x188` | 32 bits | Suppression value consulted when node flag `0x20` is absent. |

### Node layout

Offsets are byte offsets from argument 2.

| Offset | Width | Use |
|---|---:|---|
| `0x00` | 8 bits | Kind discriminator. |
| `0x02` | 16 bits | Flags; bit `0x20` suppresses preliminary emission. |
| `0x08` | 32 bits | Reference used for classification and derivation. |
| `0x10` | 32 bits | Optional direct-list pointer for kind `0x40`. |
| `0x24` | 32 bits | Assigned identifier, subsequently re-read for emission calls. |
| `0x28` | 32 bits | Written after successful delegation for kinds `0x11` and `0x6e`. |
| `0x38` | 32 bits | Referenced-object pointer; also selects the kind-`0x40` processing path. |

A direct-list header contains a signed 32-bit count at offset `0x00` and a 32-bit entry pointer at offset `0x0c`. Entries have a 12-byte stride. This routine consumes only the first 32-bit word of each entry.

The kind-`0x40` finalization region is 20 bytes. The routine explicitly writes its word at offset `8`, and conditionally its word at offset `16`.

### Preconditions

After successful emitter acquisition, every executed access requires valid storage. No general pointer, allocation-size, or alignment validation is established.

In particular:

- Kinds `0x11` and `0x6e` require a usable referenced object, including its writable field at offset `0x2c`.
- A positive direct-list count requires enough readable entries.
- Selected kind-`0x40` paths dereference the reference at node offset `0x08`.
- Successful referenced-object processing re-reads node offset `0x38` and dereferences the resulting object.
- An eligible final record write requires a valid table allocation, not merely a nonzero table pointer.

Dependency calls may change reachable state. Validity must hold at the actual access, not merely at entry.

## Behavior

### Emitter acquisition and preliminary emission

The first dependency call is F-9b3ea8470521, with no explicit arguments. A zero result causes immediate return, before any explicit context or node access by this routine. The dependency’s own effects remain unconstrained.

For a nonzero emitter, the routine performs two 16-bit reads of node offset `0x02`. The second read determines whether flag `0x20` is absent. Only when that flag is absent does the routine read context offset `0x188`.

Preliminary emission occurs when the flag is absent and that context word is zero. It comprises, in order:

1. Incrementing and storing context offset `0x48`.
2. Calling F-211638bb75da with the emitter, selector `10`, and the incremented value.
3. Calling F-887ba2404a49 with the emitter, selector `0x2f`, and value `1`.

The last call’s 32-bit result becomes the pending handle. Without preliminary emission, that handle is zero.

The node kind is read only after this stage. Preliminary dependencies can therefore affect the kind subsequently observed.

### Kind `0x40`: preparation

For kind `0x40`, these actions precede path selection:

- F-5248ae69747a receives the current node field at `0x08`; its 32-bit result is retained as the classification word.
- The context word at `0x44` is copied into node offset `0x24`.
- Context offset `0x44` is read again, incremented, and written back.
- F-887ba2404a49 receives the emitter, selector `0x70`, the re-read node identifier, and `1`. Its result is retained as the aggregate handle.
- F-EXTERNAL-01 receives the address of a 20-byte temporary region and size `0x14`.
- The routine writes the 32-bit value `1` at temporary-region offset `8`.

F-EXTERNAL-01 has no supplied semantic definition. In particular, zero-initialization of the region is not established.

The routine next reads node offset `0x38`: zero selects direct-list processing; nonzero selects referenced-object processing.

### Kind `0x40`: direct list

The direct-list pointer is read from node offset `0x10`. A null pointer skips all list-specific work and proceeds to aggregate finalization.

For a non-null pointer:

- If the classification word’s low byte is zero, the whole 32-bit word is replaced with `0x00000062`.
- The word at offset `4` of the object currently referenced by node offset `0x08` is copied into temporary-region offset `16`.
- F-e4b8d56f2a5d is called twice with the context. Its two results are retained separately.
- The saved list header supplies its entry pointer first and its signed count afterward.

The entry pointer and count are retained for iteration. Changes to the header during item processing do not cause this routine to reload them. Entry contents are read when their respective iterations begin.

Only a positive signed count produces iterations. For each entry, in order:

1. Its first word is read and retained as the item value.
2. While the pending handle is nonzero, F-010c2096817e receives that item value. A zero result causes F-74a28c0a4d64 to receive the emitter, pending handle minus one, and `2`; the pending handle is then cleared.
3. Context offset `0x54` is incremented.
4. F-c2e9c7166e37 receives the context, item value, and first acquired value.
5. Context offset `0x54` is re-read, decremented, and stored.
6. F-0b0cd8d831b7 receives the emitter, selector `0x55`, first acquired value, `1`, second acquired value, classification-word address, and `1`.
7. F-607c57cec83e receives the context, first acquired value, and `1`.
8. F-887ba2404a49 receives the emitter, selector `0x68`, the node’s re-read identifier at `0x24`, and the second acquired value.

The entry pointer advances by 12 bytes, and the retained count decreases by one.

The decrement of context offset `0x54` does not restore a saved value: it applies to the word present after F-c2e9c7166e37 returns. The classification word is passed by address and is not reset between iterations.

After iteration, including when the initial count is zero or negative, the routine calls F-65af7db35d2d twice:

- First with **the context and first acquired value**.
- Then with **the context and second acquired value**.

These paired calls do not establish deallocation, ownership transfer, or preservation of other state.

### Kind `0x40`: referenced object

F-1e7ea6aa642e receives separate descriptor storage, mode `7`, and the node’s current identifier. The routine then overwrites descriptor byte `1` with the classification word’s low byte. There is no `0x62` substitution on this path.

F-7a1ceda22360 receives the context, the object pointer re-read from node offset `0x38`, the descriptor address, and four zero arguments.

The return is represented as 64 bits, but only its low 32 bits control failure. A nonzero low half causes immediate return, skipping aggregate finalization and final pending-handle processing.

After a zero low half, node offset `0x38` is read again. The first word of the resulting object supplies a list pointer.

If that pointer is nonzero, its signed count is read. Only a strictly positive count causes F-2dc561e1332f to receive:

- The context;
- The current node field at `0x08`;
- The first word of the first entry reached through the list’s offset-`0x0c` pointer.

Its result is stored at temporary-region offset `16`.

A null list or nonpositive count leaves that word without a write by this derivation step. The delegated result’s high half does not change this decision: a null list is excluded, and a non-null list supplies the relevant count.

### Kind `0x40`: finalization

Every normally completed kind-`0x40` path calls F-2b905b6e3991 with the emitter, retained aggregate handle, and 20-byte temporary-region address. Final pending-handle processing follows.

An internal overwrite of argument 4’s storage with `-6` establishes no separate caller-visible output.

### Kinds `0x11` and `0x6e`

These kinds share the following sequence:

- Save the object pointer from node offset `0x38`.
- Increment context offset `0x48`, independently of any preliminary increment.
- Call F-1e7ea6aa642e with descriptor storage and mode `0`.
- Re-read the node kind and set descriptor byte `0` to `6` for `0x6e`, otherwise `3`.
- Re-read the node kind again and select `0x71` for `0x6e`, otherwise `0x2f`.
- Call F-887ba2404a49 with the emitter, selected selector, zero, and descriptor word at offset `4`.
- Read the saved object’s field at offset `0x2c` and pass it to F-841dc33ee157.
- After that call returns, access the unresolved global expression used as the fifth argument to F-df6b90a68c3a.
- Call F-df6b90a68c3a with the context, `0x7c`, zero, zero, and that expression’s value.
- Write the returned 32-bit value into the saved object’s field at `0x2c`.
- Call F-7a1ceda22360 with the context, saved object, descriptor address, and four zeros.

The global expression is not established as fixed or immutable. Its access follows F-841dc33ee157, and preceding dependencies are not assumed to preserve its value.

The delegated return is assigned to a **32-bit integer**. A nonzero value in those 32 bits causes immediate return. A nonzero high half alone, if the dependency provides a wider result, does not establish failure.

On zero status, descriptor word `4` is copied into node offset `0x28`, then final pending-handle processing follows. This copy observes the descriptor after delegation, not necessarily the value used in the earlier emission call.

The saved object pointer remains the target even if dependencies change node offset `0x38`. Conversely, the tag and selector use later kind reads rather than the initial dispatch value.

### Other kinds and final pending-handle processing

Other kinds perform no kind-specific work and proceed directly here.

A zero pending handle causes return. Otherwise, the routine subtracts one to form a 32-bit record index, interpreted using signed comparisons.

A record write occurs exactly when:

- The index is nonnegative;
- The emitter’s signed 32-bit field at `0x0c` is strictly greater than the index;
- The emitter’s table pointer at `0x14` is nonzero.

**Index zero is eligible.** For nonnegative indices, the comparison with the emitter field supersedes the earlier index-equals-zero condition.

The selected record has a 20-byte stride. Its 32-bit field at offset `8` receives the emitter field at `0x0c`, read again for the write. The table pointer is also re-read for address formation after its nonzero check.

Negative indices, equality with the emitter field, and larger indices produce no write. No dependency call intervenes during this final decision.

## Side effects

Explicit mutations can affect context counters, node identifier and result fields, the saved object’s offset-`0x2c` field, temporary descriptors, and an emitter-table record.

Checked delegated failures do not roll back prior increments, identifier assignments, emissions, or object-field replacement. They also bypass final pending-handle processing.

No direct allocation or deallocation is established. Dependency ownership, release behavior, pointer retention, and global preservation are unknown. Passing temporary addresses establishes availability during the call, not permission for later use.

Arithmetic uses 32-bit storage, with signed comparisons for counts and record eligibility. Flags are 16 bits; kinds and overwritten descriptor tags are 8 bits. No explicit overflow handling or fault conversion appears.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-9b3ea8470521 | Obtains the emitter; zero stops explicit processing. |
| F-211638bb75da | Receives preliminary selector `10` and the incremented counter. |
| F-887ba2404a49 | Handles selectors `0x2f`, `0x70`, `0x68`, and `0x71`; selected results are retained handles. |
| F-5248ae69747a | Supplies the classification word. |
| F-EXTERNAL-01 | Receives a temporary-region address and size `0x14`; effects unresolved. |
| F-e4b8d56f2a5d | Called twice with the context to obtain retained values. |
| F-010c2096817e | Receives an item while the pending handle remains nonzero. |
| F-74a28c0a4d64 | Receives the pending-handle-derived index and `2`. |
| F-c2e9c7166e37 | Processes an item between context-counter updates. |
| F-0b0cd8d831b7 | Receives selector `0x55` and the classification address. |
| F-607c57cec83e | Receives the context, first acquired value, and `1`. |
| F-65af7db35d2d | Receives context and first acquired value, then context and second acquired value. |
| F-1e7ea6aa642e | Prepares descriptor storage in modes `0` or `7`. |
| F-7a1ceda22360 | Delegates processing; only the retained low 32-bit status determines early return. |
| F-2dc561e1332f | Derives a word from the first entry of a positive-count referenced list. |
| F-2b905b6e3991 | Receives the aggregate handle and finalization-region address. |
| F-841dc33ee157 | Receives the object-field value before replacement and before the global-expression access. |
| F-df6b90a68c3a | Produces the replacement field value using an unresolved global-expression argument. |

F-EXTERNAL-01 is a document-local opaque identifier, not a supplied mapped identity.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Dependencies are assumed to return normally and preserve unrelated state only where an example requires it.

1. **Emitter absent:** A zero result from F-9b3ea8470521 causes no subsequent explicit context or node access by this routine.

2. **Suppression:** An unrecognized kind with flag `0x20` set performs neither preliminary emission nor final table writing, because the pending handle remains zero.

3. **Independent increments:** Kind `0x11`, with preliminary emission enabled, increments context offset `0x48` twice: once before dispatch and once in its dedicated path.

4. **Classification width:** A non-null direct list with classification `0x12340000` changes the entire classification to `0x00000062`. Referenced-object processing instead places zero into descriptor byte `1`.

5. **Empty direct list:** A non-null list with count zero or negative still obtains two values and calls F-65af7db35d2d with `(context, first value)`, then `(context, second value)`. It performs no item iteration.

6. **Consumed handle:** Pending handle `5` and a zero item-classification result cause F-74a28c0a4d64 to receive index `4` and value `2`. Later iterations skip pending-handle classification; final table writing is skipped.

7. **Delegated status width:** A delegated result with low 32 bits zero and only its high half nonzero does not trigger the checked failure return on either delegated path.

8. **Replacement survives failure:** For kind `0x6e`, nonzero retained 32-bit delegated status leaves the replacement at object offset `0x2c` in place and prevents this routine’s write to node offset `0x28`.

9. **Record zero:** Pending handle `1`, emitter field `0x0c` equal to `5`, and a valid nonzero table pointer cause record `0`, offset `8`, to receive `5`. Handle `3` under the same conditions writes `5` to record `2`.

10. **Record boundary:** Handle `6` with emitter field `5` produces index `5` and no write, because equality fails the strict comparison.

11. **Mutable global argument:** If F-841dc33ee157 changes the value supplied by the unresolved global expression, the subsequent F-df6b90a68c3a argument is obtained after that change, not from a value cached before F-841dc33ee157.

## Uncertainties

- **documented_only — temporary initialization:** F-EXTERNAL-01’s byte effects and descriptor-constructor guarantees are unavailable. Untouched temporary fields cannot be assumed zero.
- **documented_only — global expression:** Its declaration, width, value, constancy, ownership, and interpretation are unresolved. Preceding dependencies may affect it.
- **documented_only — dependency effects:** No dependency is assumed to preserve globals or reachable memory, release resources, or refrain from retaining pointers.
- **documented_only — arithmetic extremes:** Exact signed-overflow, address-wrap, and extreme-handle behavior require target-level evidence. No explicit handling is supplied.
- **documented_only — memory model:** Endianness, alignment rules, concurrency guarantees, aliasing restrictions, and exception behavior are not established.
- **Inferred intent:** Context offset `0x54` appears nesting-related; handles and final record writes appear to support deferred emission bookkeeping. These interpretations add no requirements.

## Evidence

The supplied static representation establishes explicit accesses, field widths, branch conditions, call ordering, arguments, status truncation, writes, and returns. The supplied dependency map establishes the mapped opaque function IDs.

The final-record conditions follow the effective condition after the nonnegative-index comparison, which permits index zero. The paired cleanup calls explicitly include the context. The global-expression access occurs after F-841dc33ee157. Both delegated paths base failure on 32 bits, despite the wider represented return on the kind-`0x40` path.

No dependency implementation or runtime observation is supplied. No runtime tests have been performed. Unresolved behavior remains documented rather than replaced with assumptions about initialization, immutability, ownership, or recovery.
