<!-- Trusted provenance: F-2ef7dbdf62d6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2ef7dbdf62d6

## Purpose

F-2ef7dbdf62d6 resolves an object through a receiver, searches that object’s collection, constructs a new entry using one of two descriptor-selected paths, and invokes collection-update operations. It then performs conditional notification and a separately gated sequence involving a stack-local address.

**Established static behavior:** descriptor accesses, search criteria, explicit record writes, dependency arguments, call ordering, conditional gates, and a packed 64-bit return value.

**Inferred intent:** the two collection-update paths appear to distinguish insertion from replacement. Some dependencies appear to construct entries or propagate changes. These interpretations do not establish their ownership, copying, initialization, destruction, or notification contracts.

The description below specifies the visible caller behavior. Unavailable dependency behavior remains **documented_only**.

## Interface

### Inputs

The function receives:

1. An opaque 32-bit argument, unused by the visible body.
2. A receiver pointer supporting an indirect operation at dispatch-table byte offset `0xF4`.
3. A descriptor pointer with the following observed accesses.

| Descriptor byte offset | Access width | Use |
|---|---:|---|
| `0x00` | 32 bits | Argument to receiver dispatch operation `0xF4`. |
| `0x04` | 32 bits | Entry-form selector; reread for F-283eda65a14c on the nonzero-selector path. |
| `0x08` | 32 bits | Search key and record-field input on the nonzero-selector path. |
| `0x0C` | 32 bits | Comparison and construction input on the zero-selector path; also interpreted as an address for a separate byte read. |
| `0x10` | 32 bits | Flags combined with `0x39530000` by bitwise OR. |
| `0x14` | 32 bits | Additional zero-selector construction argument. |
| `0x18` | 8 bits | Argument to the new entry’s dispatch operation `0x3C`. |

A contiguous readable descriptor region of **25 bytes**, from offset `0x00` through offset `0x18` inclusive, covers the extent of all direct descriptor accesses. The function does not establish a requirement for 28 readable bytes or a complete seventh 32-bit word. Individual paths access only their applicable fields; valid alignment, pointer representations, and dependency-specific requirements remain necessary.

The descriptor is not snapshotted. Reads occur throughout execution, including after dependency calls. Aliasing or dependency side effects can therefore change later inputs.

### Output

On normal completion, the function returns a 64-bit packed value:

- Bits `31:0`: the 32-bit representation of the resolved-object pointer.
- Bits `63:32`: the 32-bit representation of the address at byte offset `0x80` within that same object.

The high half identifies an embedded location, not a separately allocated result. There is no explicit success flag or error code.

### Preconditions and failure exposure

The visible body assumes valid dispatch tables, objects, collection metadata, encountered entries, and memory used by subsequent operations. It does not check the resolved object or allocation results for null before using them.

The zero-selector path additionally reads one byte through the current address represented by descriptor offset `0x0C`. Comparison-dependency requirements for that value are unknown.

On the nonzero-selector search path, each nonnull adapted candidate must provide a readable 32-bit pointer at object offset `0x10`, leading to a readable 32-bit field at record offset `0x18`.

Normal continuation after dependencies requires addresses and objects used afterward to remain valid. That requirement is not evidence that dependencies preserve them. No local error recovery, rollback, or explicit deallocation is visible.

## Behavior

### Target resolution and captured search state

The receiver’s dispatch operation at byte offset `0xF4` is invoked with the receiver and the descriptor word at `0x00`. Its result is used as the resolved object.

The function then:

- Reads a 32-bit collection pointer from resolved-object offset `0x1E0`.
- Reads a pointer from collection offset `0x04`.
- Reads a signed 32-bit count from offset `0x04` within that pointed-to structure.

The collection pointer and count are captured before the selector at descriptor offset `0x04` determines the construction path. Neither is refreshed during searching.

Both searches examine indices in ascending order, beginning at zero, while the signed index is less than the captured signed count. Zero or negative counts produce no iterations. A positive count is not checked against actual collection capacity.

The first match ends the search. Exhaustion leaves no selected candidate and a replacement index of `−1`; the last examined candidate is not retained as a fallback. Index advancement for a positive signed 32-bit count reaches at most that count, without requiring an increment beyond the maximum signed count.

### Zero-selector path

This path is selected when the initially tested descriptor word at `0x04` equals zero.

#### Search

For each visited index:

1. F-71f95996d3e8 receives the captured collection pointer and index.
2. The returned entry’s dispatch operation at offset `0x18` produces an operand.
3. An unmapped comparison dependency receives that operand and the current descriptor word at `0x0C`.

A zero comparison result selects the entry and its index. The supplied evidence does not establish comparison rules such as encoding, case sensitivity, termination, or treatment of invalid operands.

#### Entry construction

F-8157a180d901 is first called with size `0x40`.

A signed 16-bit value is then obtained:

- Without a selected entry, it is the low 16 bits of the current descriptor word at `0x0C`, interpreted as signed.
- With a selected entry, it is the 16-bit result of that entry’s dispatch operation at offset `0x54`.

The eventual argument is sign-extended to 32 bits. In the unmatched case, this value comes from the address representation itself, not from memory at that address.

Three other inherited values default to zero. With a selected entry, its dispatch operations are called in this order:

1. Offset `0x50`.
2. Offset `0x2C`.
3. Offset `0x28`.

F-893c3f343b70 then receives the allocation result followed by:

| Argument after storage | Value |
|---:|---|
| 1 | Current descriptor word at `0x0C`, interpreted as an address. |
| 2 | One byte read through the descriptor-supplied address at `0x0C`. |
| 3 | Result from dispatch `0x28`, or zero. |
| 4 | Zero. |
| 5 | Zero. |
| 6 | Result from dispatch `0x2C`, or zero. |
| 7 | Zero. |
| 8 | Current descriptor flags at `0x10` OR `0x39530000`. |
| 9 | `4`. |
| 10 | Current descriptor word at `0x14`. |
| 11 | Result from dispatch `0x50`, or zero. |
| 12 | The sign-extended 16-bit value. |

The dependency result becomes the new-entry pointer.

Allocation precedes the selected-entry property calls. The supplied representation does not establish an evaluation order among the individual argument expressions of F-893c3f343b70. In particular, descriptor stability must not be assumed merely because several arguments refer to the same field.

### Nonzero-selector path

This path is selected when the initially tested descriptor word at `0x04` is nonzero.

#### Search

For each visited index, F-71f95996d3e8 supplies an entry to F-682adfd5a561 with discriminator `0x16`.

A null adaptation result is skipped. For a nonnull result, the function follows the pointer at adapted-object offset `0x10` and compares the 32-bit field at the resulting record’s offset `0x18` with the current descriptor word at `0x08`.

The first equal bit pattern selects the adapted candidate and index. Signed interpretation does not affect equality. Exhaustion clears the candidate even if the last adaptation was nonnull but its key differed.

#### Record allocation and unmatched writes

F-8157a180d901 is called with size `0x34`.

Without a selected candidate, the function performs these explicit writes to the returned storage, in order:

| Order | Byte offset | Width | Value |
|---:|---|---:|---|
| 1 | `0x00` | 8 bits | `4` |
| 2 | `0x04` | 32 bits | Zero |
| 3 | `0x0C` | 32 bits | Zero |
| 4 | `0x08` | 32 bits | Descriptor flags OR `0x39530000` |
| 5 | `0x10` | 32 bits | Zero |
| 6 | `0x14` | 16 bits | Zero |
| 7 | `0x18` | 32 bits | Descriptor key at `0x08` |
| 8 | `0x20` | 32 bits | Zero |
| 9 | `0x1C` | 32 bits | A second descriptor-key read |
| 10 | `0x24` | 32 bits | Zero |
| 11 | `0x28` | 32 bits | Zero |
| 12 | `0x2C` | 32 bits | Zero |
| 13 | `0x30` | 32 bits | Zero |

The flags read occurs after the write at `0x04` and before the write at `0x0C`. The second key read occurs after the write at `0x18` but before the write at `0x20`. Thus the two key fields need not agree if the descriptor changes between reads, including through aliasing.

Bytes `0x01–0x03` and `0x16–0x17` receive no explicit writes in this branch. Their contents are not established.

#### Selected-candidate dependency invocation

With a selected candidate:

1. Its dispatch operation at offset `0x60` is invoked.
2. An unidentified external dependency receives the allocated record pointer, that dispatch result, and the numeric argument `0x34`.
3. Record offset `0x18` is explicitly overwritten with the current descriptor key.
4. Record offset `0x08` is explicitly overwritten with the current descriptor flags OR `0x39530000`.

The external invocation is established; copying is not. The argument `0x34` does not independently prove a 52-byte transfer, a source-readable extent, preservation of other record bytes, or preservation of unrelated state.

There is no explicit write to record offset `0x1C` on this path. Its value after the external invocation remains unspecified by this function. Only an independently established dependency contract could justify describing it as a preserved copied field.

#### New-entry dependency sequence

After either record-preparation branch:

1. F-8157a180d901 receives size `0x2C`.
2. F-283eda65a14c receives a fresh read of descriptor offset `0x04`.
3. F-682adfd5a561 receives that result and discriminator `5`.
4. F-e768f9f56408 receives the `0x2C` allocation result, the discriminator-`5` result, and the record pointer.

The final result becomes the new-entry pointer. The selector passed here is not guaranteed to equal the earlier branch-selection value.

### Common entry and collection operations

The new entry’s dispatch operation at offset `0x3C` receives the byte read at descriptor byte offset `0x18`. This is the byte at that address, not an established numeric low byte of a 32-bit seventh word.

Collection operations then use the originally captured collection pointer:

- Without a match: F-eaccb645fcdf receives the collection and new entry.
- With a match: F-14d5f1ca2fa8 receives the collection and new entry, followed by F-791a9cdc71ef receiving the collection and matched index.

The second matched-path call uses the captured index without adjustment or a new search. Actual insertion, removal, ordering, release, and ownership semantics depend on these dependencies.

### Conditional notification

The function rereads resolved-object offset `0x1E0`. If that fresh collection-field value is nonzero:

1. F-004c5654fd0a is invoked.
2. Its result is passed to the resolved object’s dispatch operation at offset `0x16C`.

The reread serves only as a gate. It does not replace the collection pointer previously used for searching and updates.

### Final gated sequence

F-3fac671648b8 is invoked unconditionally after the notification stage.

Further work requires, in short-circuit order:

1. A nonzero result from F-3fac671648b8.
2. A zero value in the low eight bits of the resolved object’s 32-bit field at offset `0xA0`.
3. A nonzero byte reached through an external global pointer.

If an earlier gate fails, later gate reads do not occur.

When all gates pass:

1. The 32-bit value at resolved-object offset `0x34` is captured.
2. F-e57b05520b65 receives a stack-local address and, as its next two arguments, the same resolved-object address at offset `0x80`.
3. If the captured value is nonzero, F-ee8733a08210 receives the stack-local address and an address formed by adding `0xD0` bytes to the captured value.
4. F-ffe0d92c57b7 receives the resolved object.
5. If that result is nonzero, F-c9b5d17d8d1e receives it and the stack-local address, followed by F-4d037ecf0f82 with the same two argument values.

The captured offset-`0x34` value is not refreshed after F-e57b05520b65. The final target is not refreshed between the last two calls.

The invocation of F-e57b05520b65 does **not** establish that it initializes the stack-local region. Its contents, extent, representation, and validity for later consumers depend on unavailable contracts. No explicit local initialization or destruction operation is shown.

## Side effects

Direct writes are established for the allocated nonzero-selector record. Other effects occur through dependencies and indirect operations.

The descriptor is not explicitly written, but aliasing can expose it to direct record writes or dependency effects. The function imposes no visible protection against such overlap.

No explicit deallocation, reference-count adjustment, or rollback appears. This does not establish absence of those effects inside dependencies. In particular:

- Allocation initialization and failure behavior are unknown.
- Entry-producing dependencies may retain, transform, or release supplied resources.
- Collection operations may transfer ownership or affect lifetimes.
- The unidentified external dependency may modify record storage or other state.
- No dependency is assumed to preserve globals, descriptor contents, or object fields.
- Passing a stack-local address does not establish that retaining it is safe.

Pointer offsets and packed return components use the shown 32-bit address representation. Validity of address formation, including addition of `0x80` or `0xD0`, is a precondition for ordinary address-based interpretation; overflow behavior outside that domain is not established.

## Dependencies

| Opaque ID | Established call-site role |
|---|---|
| F-71f95996d3e8 | Receives a collection and index; its result is used as an entry. |
| F-8157a180d901 | Receives storage sizes `0x40`, `0x34`, or `0x2C`. |
| F-893c3f343b70 | Receives zero-selector storage and construction inputs; returns the new entry. |
| F-682adfd5a561 | Receives values with discriminator `0x16` or `5`. |
| F-283eda65a14c | Receives the reread nonzero-path selector. |
| F-e768f9f56408 | Receives object storage, an adapted value, and the record pointer; returns the new entry. |
| F-eaccb645fcdf | Unmatched collection-path operation. |
| F-14d5f1ca2fa8 | First matched collection-path operation. |
| F-791a9cdc71ef | Second matched operation, receiving the captured index. |
| F-004c5654fd0a | Supplies the conditional dispatch-`0x16C` argument. |
| F-3fac671648b8 | Supplies the first final-sequence gate. |
| F-e57b05520b65 | Receives a stack-local address and two identical embedded-object addresses; initialization is unknown. |
| F-ee8733a08210 | Receives that stack-local address and the optional captured-value-plus-`0xD0` address. |
| F-ffe0d92c57b7 | Supplies the target for the last two calls. |
| F-c9b5d17d8d1e | First final-target operation. |
| F-4d037ecf0f82 | Second final-target operation. |

No opaque IDs were supplied for the comparison dependency, the external three-argument dependency, or indirect dispatch targets. Their identities and contracts remain **documented_only**; no invented mapping is implied.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

1. **Descriptor extent.** A valid, appropriately aligned 25-byte descriptor region covers all direct descriptor access extents. This does not satisfy separate requirements for the address stored at `0x0C`, resolved objects, or dependency inputs.

2. **Empty or negative count.** No entry lookup occurs. Construction follows the unmatched branch, and F-eaccb645fcdf is used.

3. **First match.** If indices zero and one would both match, only index zero is examined. The matched collection sequence receives index zero.

4. **Exhausted search.** If every candidate fails, the last examined candidate supplies neither inherited properties nor selected-path record input.

5. **Signed fallback.** With a valid zero-selector address whose low 16 bits are `0xFFFF`, the final F-893c3f343b70 argument is `−1`, not `65535`.

6. **Fresh record.** With no match and a stable key of `7`, explicit writes place `7` at both `0x18` and `0x1C`. Unwritten padding has no specified value.

7. **Selected-record external call.** The external dependency receives the allocated pointer, dispatch-`0x60` result, and `0x34`. Subsequent explicit writes set `0x18` to the current key and `0x08` to the combined flags. A value of `99` at source offset `0x1C` implies a destination value of `99` only if an independent dependency contract establishes the relevant copying and preservation behavior.

8. **Flags.** Stable flags `0x00000008` yield `0x39530008`. Caller bits are not masked away.

9. **Fresh notification gate.** If an intervening dependency sets resolved-object offset `0x1E0` to zero, notification is skipped even though earlier collection calls used the captured pointer.

10. **Short-circuit gates.** A zero F-3fac671648b8 result prevents the later field and global-byte gate reads. A nonzero low byte of the field at `0xA0` prevents the global-byte read.

11. **Stack-local sequence.** If all gates pass and the captured field is zero, F-ee8733a08210 is skipped. F-ffe0d92c57b7 still runs after F-e57b05520b65. No expected stack-local contents can be specified without dependency contracts.

12. **Return packing.** For resolved-object representation `0x10000000`, with valid address arithmetic, the return value is `0x1000008010000000`.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of selectors, flags, entries, and discriminator values.
- Comparison semantics and operand requirements.
- Allocation failure, initialization, alignment, and ownership guarantees.
- Entry-producing dependency contracts and resource lifetimes.
- Whether the unidentified external dependency copies anything; any transfer extent, overlap rules, or state-preservation guarantees.
- Initialization, size, contents, and consumer validity of the stack-local region.
- Collection ordering, replacement, release, and index semantics.
- Dependency effects on aliases, globals, or object fields.
- Target byte order and platform behavior outside valid memory and address-arithmetic conditions.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent behavioral evidence.

The representation establishes direct access widths and offsets, constants, search termination, explicit record writes, call arguments, visible call ordering, short-circuit gates, and return packing. It does not establish copying by the unidentified external dependency or initialization by F-e57b05520b65.

No runtime tests have been performed.
