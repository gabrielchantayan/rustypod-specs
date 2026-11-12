<!-- Trusted provenance: F-81e0dd8ea0d5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-81e0dd8ea0d5

## Purpose

F-81e0dd8ea0d5 acquires a container from a context-selected source and conditionally rearranges its 32-bit entry references.

Two modes are visible:

- **Individual-entry mode:** exchanges entries using successively smaller prefixes.
- **Grouped mode:** constructs temporary inclusive index ranges from entry metadata, rearranges those range records, optionally exchanges source entries, and reconstructs the original container.

A preferred entry participates in two distinct mechanisms: matching a 32-bit entry field during range construction, and searching for the exact preferred reference near return. Their activation conditions differ.

Randomization, metadata grouping, and preferential placement are inferred intent. Guaranteed outcomes depend on unresolved dependencies, initialization, bounds, and aliasing.

## Interface

The function accepts four 32-bit arguments and returns a 32-bit container reference.

| Argument | Established role |
|---|---|
| Context reference | Supplies source fields, an enable-bearing reference, and a reference used to form addresses for scan-surrounding calls. |
| Preferred entry reference | When nonzero, supplies a matching value and enables the later exact-reference search. |
| Grouping selector | Zero selects individual-entry mode; nonzero selects grouped mode. |
| Within-group selector | In grouped mode, enables attempted source exchanges and permits final exact-reference adjustment. It does not gate that adjustment in individual-entry mode. |

The return value is always the first result from F-e867b748d9b5, including zero on initial acquisition failure. Handled grouped allocation or second-acquisition failure does not produce a distinct error result.

### Accessible representations

References, container counts, and entry slots occupy 32 bits. Address calculations and allocation-size multiplication have no established overflow protection.

| Object | Offset | Access |
|---|---:|---|
| Context | `0x0C` | 32-bit reference; adding `0x1C8` forms a dependency argument. |
| Context | `0x40` | 32-bit reference to an enable-bearing object. |
| Context | `0x3B0` | 32-bit source value for individual-entry mode. |
| Context | `0x3C4` | 32-bit source value for grouped mode. |
| Enable-bearing object | `0x2E` | Unsigned 16-bit enable value. |
| Container | `0x0C` | 32-bit count. |
| Container | `0x10` onward | Contiguous 32-bit entry references. |
| Nonnull entry | `0x10` | 32-bit matching value. |
| Nonnull entry | `0x20` | 32-bit metadata reference. |

Each temporary range record occupies 12 bytes:

- Offset zero: 32-bit inclusive start.
- Offset four: 32-bit inclusive end.
- Offset eight: a 32-bit third word used in marking and record exchanges.

**Mark writes and mark tests have different widths.** Marking writes the single byte value one at record offset eight. Preferential placement later reads the entire 32-bit third word, converts its value to an 8-bit character, and tests the converted value against zero. The test therefore depends on the word’s low-order eight bits, not inherently on its first addressed byte.

On a little-endian target, the written byte at offset eight is the low-order byte tested after conversion. On a big-endian target, the low-order byte is at offset eleven instead. The supplied representation does not independently establish target byte order.

Metadata processing also uses an unsigned 16-bit saved length followed immediately by 510 bytes of saved storage. Neither region is explicitly initialized locally before metadata calls.

Valid execution requires accessible storage for every actual access and sufficient capacities. A nonzero reference alone does not establish validity.

## Behavior

### Acquisition and enable gate

The function first calls F-60950fd9e511 with the context and mode one for individual-entry processing, or mode six for grouped processing. Only afterward does it read the selected context source field and pass that value to F-e867b748d9b5.

A zero acquisition result returns immediately, before enable-field access.

For a nonzero result and nonzero context, the function reads the enable-bearing reference and then its unsigned 16-bit enable field. Zero disables further local processing.

A zero context also selects an early return at this stage. This is not a safe null-context interface: the selected source-field access has already occurred through that context, and the first dependency has already received it.

After processing is enabled, a nonzero preferred reference is dereferenced at offset `0x10` to capture its matching value; otherwise the captured value is zero. This read occurs even in individual-entry mode. The original acquired container’s count is then read.

### Individual-entry mode

For an ordinary positive count greater than one, the function requests exchange indices from F-bd9ea04b96b3 using prefix lengths equal to the initial count, then one less, continuing through two.

Each result is used directly as an entry index. There is no bounds check. For each exchange:

1. The selected entry reference is read.
2. The final entry of the current prefix is read and written into the selected slot.
3. The saved selected reference is written into the prefix’s final slot.

Selecting the final slot still performs these accesses.

Count one makes no selection call. Count zero is not an empty-container no-op: the decrement-based condition proceeds with a last-position index represented as minus one. Counts outside the ordinary positive signed-32-bit range likewise lack a safe-array guarantee.

Safe conventional use requires a positive representable count, sufficient storage, and selection results within each requested prefix. Uniform distribution is not established.

Individual-entry mode reaches final preferred-reference adjustment regardless of the within-group selector.

### Grouped allocation and acquisition

Grouped mode requests temporary storage from F-858f5b8d5984 using the original count multiplied by 12. The calculation is 32-bit and unchecked.

If allocation returns zero, construction and reconstruction are skipped. Final preferred-reference adjustment remains reachable only when the within-group selector is nonzero.

After successful allocation, F-e867b748d9b5 receives the original acquired reference. If this second acquisition returns zero, temporary storage is passed to F-1137792fba7d before the same selector-dependent final-adjustment decision.

A nonzero second result supplies the grouped source container. Its count is not checked against the original count used to size temporary storage. Equality, adequate capacity, and independent backing storage are not established.

### Metadata scan and boundaries

Before scanning, the function reads context offset `0x0C`, adds `0x1C8`, and passes the resulting address to F-efaaaab570a3.

The source scan visits indices in increasing unsigned order and rereads the second container’s count in its loop condition. Each entry reference is read before deciding whether to process metadata. Null references skip both metadata handling and matching-field marking; they do not themselves create boundaries.

For a nonnull entry at index zero, F-29854e913c2d receives the entry’s metadata reference and the address of the saved 16-bit length. No separate length argument is supplied. Whether it also initializes the following saved bytes, and its write extent, remain unresolved.

For a nonnull entry at a later index, F-7445c028e81c receives the metadata reference and output locations for a metadata address and a 32-bit length. The reported length is normalized:

- Zero through 255 remain unchanged.
- Every unsigned value of 256 or greater becomes 255.

The boundary decision is:

| Condition | Result |
|---|---|
| Normalized length is zero | Create a boundary without an equality read of the saved length. |
| Nonzero normalized length differs from saved length | Create a boundary. |
| Matching nonzero lengths; comparison returns zero | Create a boundary. |
| Matching nonzero lengths; comparison returns any nonzero 32-bit value | Continue the current range. |

The comparison receives the saved-byte address, current metadata address, and twice the normalized length as a byte count. Its maximum requested span is 510 bytes. Every nonzero result, including one with its high bit set, selects continuation. Conventional comparison or equality semantics are not established.

At a boundary, the function writes the preceding index into the current record’s end, advances by 12 bytes, writes the current index into the new record’s start, and stores the normalized length into the saved 16-bit field. For nonzero length, it then calls F-70a021fc7c24 with the current metadata address first and the saved-byte address second, without an explicit length argument.

After metadata handling, every nonnull entry has its matching field read. Equality with the captured preferred matching value writes one byte at the current record’s offset eight. This can occur without a preferred reference, because the captured value is then zero. Preferential range placement nevertheless requires a nonzero captured matching value.

### Missing initial metadata and record initialization

If source index zero is null, initial metadata extraction is skipped. The saved length and saved bytes then have no established initialization.

A later nonnull entry with nonzero normalized length reads the saved length. If equality is observed, comparison also reads saved bytes whose initialization is not established.

A later zero-length entry instead creates a boundary and establishes saved length zero without initializing saved bytes. A subsequent nonzero length differs from zero and creates another boundary without comparison; the ensuing F-70a021fc7c24 call may establish saved bytes, depending on its contract.

Separately, the first temporary record’s start is never explicitly initialized. No record’s third word, or mark byte, is explicitly cleared. Initializing temporary records would not resolve missing initialization of the separate saved metadata state.

### Completing and ordering records

After scanning, the function freshly reads context offset `0x0C`, adds `0x1C8`, and calls F-2a517f9b9ef3. Intervening dependencies may have changed that context field; the two surrounding calls are not proven to receive the same address.

It then rereads the second container’s count and writes count minus one into the current record’s end.

F-870c4fcf7295 receives the 32-bit byte distance between the first and current record addresses, together with 12. Its result plus one becomes the record count. Treating that result as the number of constructed records requires an unresolved arithmetic contract.

If the resulting unsigned count exceeds one, shrinking-prefix exchanges rearrange complete 12-byte records using F-bd9ea04b96b3. For each exchange, the final record’s three words are read, followed by the selected record’s end and third word. The selected start is read before replacement of the final record’s start. The final record’s end and third word are then replaced, followed by the selected record’s start, end, and third word. All bytes of the third words move, not only the byte written by marking.

When the captured matching value is nonzero, preferential placement scans records in increasing order. At each candidate it performs a **32-bit read at offset eight**, converts that word to an 8-bit character, and tests the converted value. The first nonzero result causes an exchange of that complete record with record zero, then terminates the search. A match at record zero still performs the exchange accesses.

This does not guarantee selection of the range containing the exact preferred reference. Shared matching values, uninitialized third words, dependency mutations, and target byte order affect the result.

### Optional source exchanges and reconstruction

Records are processed in their resulting order. Each supplies unsigned 32-bit start and end indices.

A nonzero within-group selector enables attempted exchanges when the signed interpretation of the wrapped 32-bit inclusive size, end minus start plus one, exceeds one. This is not an unlimited mathematical size calculation.

During exchanges, the high index decreases while it exceeds the unsigned start. F-bd9ea04b96b3 receives the remaining inclusive length. Its result is used as an **absolute source index**: the range start is not added. A range beginning above zero can therefore exchange an entry with a position preceding that range.

Each source exchange reads the selected reference, reads the high-position reference, writes the latter into the selected slot, and writes the saved reference into the high slot.

After optional exchanges, source entries from start through end, inclusively, are copied into successive original-container slots. The destination index starts at zero and continues across records. Each source read precedes its destination write.

No coverage, disjointness, capacity, bounds, or storage-independence validation occurs. Start greater than end skips copying. End `0xFFFFFFFF` creates increment-wrap hazards rather than a reliable finite traversal. Neither container count is explicitly updated.

Temporary storage is then passed to F-1137792fba7d, followed by the nonzero second container reference. Only afterward is the grouped final-adjustment gate applied.

### Final preferred-reference adjustment

This phase is reached after individual-entry processing, or after grouped processing and handled failures when the within-group selector is nonzero.

A zero preferred reference causes no search. Otherwise, the original acquired container is scanned in increasing unsigned index order, rereading its count in each loop condition.

At the first exact-reference match at index \(k\), F-70a021fc7c24 receives the entry-array base, the address one entry later, and \(4k\) calculated with 32-bit arithmetic. The preferred reference is then written into entry zero, and the function returns. Index zero still causes the dependency call with byte count zero.

An overlap-safe transfer from the first address to the second would shift preceding references right. That direction and overlap behavior are not established. If no match is found, no local final-adjustment write occurs.

## Side effects

Local writes affect entry arrays, temporary records, saved metadata state, and dependency output locations.

No dependency is assumed to preserve globals, context fields, counts, entries, or referenced storage. Repeated reads may therefore observe changes. The preferred matching value is captured once; scanned matching fields are read after their metadata calls.

There is no distinct return status for grouped allocation failure, second-acquisition failure, or completed processing. Invalid references, out-of-range selection results, insufficient capacity, arithmetic wrap, and malformed ranges have no local recovery path.

Ownership remains unresolved. The original acquired reference is not directly passed to F-1137792fba7d, but the second reference may alias it or share storage. Passing that second reference to the dependency does not establish continued validity or independent ownership of the returned object.

## Dependencies

| Opaque ID | Established use and unresolved contract |
|---|---|
| F-60950fd9e511 | Receives context and mode before source-field selection; context effects unknown. |
| F-e867b748d9b5 | Supplies original and optional second containers; copying, aliasing, ownership, and lifetime unknown. |
| F-bd9ea04b96b3 | Supplies exchange indices; bounds and distribution unknown. |
| F-858f5b8d5984 | Receives temporary byte-size request; initialization and zero-size behavior unknown. |
| F-29854e913c2d | Receives initial metadata and saved-length address; write extent unknown. |
| F-7445c028e81c | Receives metadata and output locations for an address and 32-bit length. |
| F-70a021fc7c24 | Has two-argument metadata and three-argument array-transfer call sites; interface, direction, bounds, and overlap contract unresolved. |
| F-870c4fcf7295 | Receives record-byte distance and 12; arithmetic contract unresolved. |
| F-efaaaab570a3 | Receives the context-derived address before scanning. |
| F-2a517f9b9ef3 | Receives a freshly evaluated context-derived address after scanning. |
| F-1137792fba7d | Receives temporary storage and any acquired second container; release semantics unknown. |

The comparison dependency has no supplied opaque ID. Its established local result rule is zero for boundary formation and any nonzero 32-bit result for continuation when nonzero lengths match.

## Acceptance examples

These are conditional static expectations, not executed tests. Outcomes assume valid storage and no additional dependency mutations unless stated otherwise.

1. **Initial acquisition failure:** a zero first acquisition returns zero before enable access or local rearrangement.

2. **Disabled processing:** zero enable returns the acquired reference without reading the preferred entry.

3. **Individual exchanges:** `[A, B, C]`, count three, and selection result zero for requests three and two produce `[B, C, A]` before any final adjustment.

4. **Count boundary:** individual count one makes no selection request. Count zero does not satisfy an empty-container no-op contract.

5. **Grouped allocation failure:** zero within-group selector prevents final preferred-reference searching; a nonzero selector leaves that search reachable.

6. **Comparison behavior:** with initialized saved metadata and both lengths three, comparison requests six bytes. Result zero starts a new range; result one or `0x80000000` continues the range.

7. **Zero metadata length:** a later nonnull entry reporting zero starts another range even if saved length is already zero, without comparison.

8. **Null first entry:** if index zero is null and index one reports nonzero length, no deterministic grouping outcome is established without additional saved-state initialization evidence.

9. **Word-width mark test:** a third word numerically equal to `0x00000100` converts to zero for the test, despite being a nonzero 32-bit word. A word equal to `0x00000001` tests nonzero.

10. **Byte-order-dependent marking:** with an initially zero third word, writing one at offset eight produces a nonzero test on a little-endian target. On a big-endian target it produces word `0x01000000`, whose converted low-order byte is zero. These outcomes require the stated byte order and no intervening mutation.

11. **Range-crossing exchange:** range `[2, 4]` and initial selection zero exchange source positions zero and four, not two and four.

12. **Conditional preferred movement:** for `[A, B, P]`, an exact search for `P` produces `[P, A, B]` only if the final transfer dependency performs the required overlap-safe first-address-to-second-address movement.

## Uncertainties

The following remain **documented_only**:

- Target byte order and consequent correspondence between the offset-eight byte write and the tested low-order byte.
- Initial first-record start, third-word contents, and mark state.
- Initial metadata write extent and saved-state initialization.
- Defined grouping after skipped initial metadata extraction.
- Comparison meaning beyond its observed branch effect.
- Second-container count, capacity, aliasing, independence, and lifetime.
- F-870c4fcf7295 arithmetic semantics.
- Both F-70a021fc7c24 call contracts.
- Selection bounds, distribution, and other dependency effects.
- Synchronization semantics and identity of surrounding-call targets.
- Ownership and release behavior.
- Safe outcomes for empty grouped containers, arithmetic overflow, invalid ranges, or aliased reconstruction.

## Evidence

Established behavior derives solely from the supplied static representation and dependency mapping: access widths, offsets, arithmetic, branch conditions, call order, writes, and return paths.

The marking operation is an explicit one-byte write. Preferential placement instead performs a 32-bit third-word read followed by character conversion and a zero test. Equating the written byte with the tested low-order byte requires established target byte order.

Grouping, randomization, synchronization, transfer, and release terminology describe inferred roles where dependency contracts are unavailable. Conditional examples do not turn those inferences into guarantees.

No runtime tests have been performed.
