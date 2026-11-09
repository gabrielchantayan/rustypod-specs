<!-- Trusted provenance: F-6489b8f5efca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6489b8f5efca

## Purpose

F-6489b8f5efca resolves a name pointer from an input record’s 32-bit selector, combines that pointer with additional record attributes, and invokes three dependencies to process a destination.

Font selection and formatting are inferred intent, supported by embedded names such as `Helvetica`, `Osaka`, and `AppleGothic`. The established behavior is narrower: select or obtain a pointer, pass it and record-derived arguments to F-59c1430f85c2 with temporary storage, pass that storage to F-8e04864cbbaa, and finally pass an interior record address to F-ccb5d9751852.

Some selectors select embedded names; unresolved selectors enter a lookup chain. Three selector-boundary values are unavailable. There are also **three distinct unresolved embedded-name locations**: the location selected by the third unavailable equality value, the location associated with `0x80B9`, and the location associated with `0x80C0`. Their contents, including whether any contents are equal, are unknown.

The function has no explicit return value, status result, or locally implemented failure recovery.

## Interface

### Arguments

The four arguments occur in this order:

| Argument | Established representation | Use |
|---|---|---|
| Context | 32-bit address-like value | Supplies a 32-bit field at byte offset `0x38` only when the deepest fallback is reached. |
| Input record | Pointer to 32-bit unsigned words | Supplies the selector, a size-like word, byte-sized output data, a character-converted word, and an interior address. |
| Destination | Opaque 32-bit value | Initially stored in temporary storage and later passed directly to two dependencies. |
| Byte-output address | Writable byte pointer | Receives one byte from record offset `0x0D`; its pointer value is also initially stored in temporary storage. |

The destination’s representation as a pointer, handle, or other value is not established. The context is not directly dereferenced unless both preceding lookup attempts return null.

The supplied representation uses 32-bit address arithmetic and address-like values. This is not a portable interface specification for targets with different pointer widths.

### Input-record layout

Offsets below are byte offsets from the input-record address.

| Offset | Width or extent | Established use |
|---|---:|---|
| `0x00` | 32 bits | Initial selector; read again if the first fallback lookup returns null. |
| `0x04` | 32 bits | Size-like word, read after name resolution. |
| `0x08` | Extent unknown | Starting address forwarded to F-ccb5d9751852. The enclosing function does not directly read contents at this offset. |
| `0x0C` | 32 bits | Read after the byte-output store and converted to a character-width argument. |
| `0x0D` | 8 bits | Read and copied to the byte-output address. This byte overlaps the word at `0x0C`. |

The direct record reads extend through offset `0x0F`. An accessible 16-byte record covers those reads, but does not establish the extent that F-ccb5d9751852 may access from offset `0x08`.

The record address plus `0x08` is forwarded directly. The function does not read a pointer from that location and forward the pointer’s value.

### Outputs and preconditions

There is no explicit return value. The direct externally visible store is a single byte through the fourth argument. Destination changes, temporary-object construction, and other effects depend on the callees.

For execution without invalid memory accesses:

- The record must remain readable for every reached direct access.
- The byte-output address must permit the one-byte store.
- If the deepest fallback is reached, the context must permit a 32-bit read at `0x38`.
- Addresses and values passed to dependencies must satisfy those dependencies’ contracts, which are unavailable.
- Address calculations must identify valid accessible storage under the execution environment.

There is no explicit null-pointer, bounds, alignment, or address-overflow validation. Target byte order and alignment requirements are not established.

## Behavior

### Entry observations and temporary storage

The first visible read captures a shared 32-bit global value. The next reads the record’s full 32-bit selector.

Temporary storage is then initialized with the destination and the byte-output pointer in adjacent 32-bit slots. These initial values do not establish the complete temporary-object layout, its eventual contents, or its full required extent.

The entry-captured global value is retained for later lookup calls. The first fallback lookup receives a separate, fresh read of that same global. A change between these reads can therefore produce different argument values.

### Embedded selection and precedence

Selection uses full-width equality comparisons and signed 32-bit ordering comparisons. It is not an unsigned range table, and the selector is not truncated before selection.

Three unavailable data values serve as equality selectors and routing boundaries. For description:

- The **Osaka boundary** selects `Osaka` on equality.
- The **AppleGothic boundary** selects `AppleGothic` on equality.
- The **third boundary** selects an embedded name whose contents are unavailable.

These labels do not assign numeric values to the boundaries or establish that their stored values are immutable.

The selection regions have the following precedence:

1. Equality with the Osaka boundary selects `Osaka`.
2. A selector signed-greater than the Osaka boundary enters the high numeric region.
3. Otherwise, equality with the AppleGothic boundary selects `AppleGothic`.
4. A selector signed-greater than the AppleGothic boundary enters the middle numeric region.
5. Otherwise, the remaining lower-region rules apply.

Thus an earlier boundary equality takes precedence over a later numeric association. Missing boundary values prevent unconditional routing claims for individual numeric selectors.

The high region contains these associations:

| Selector | Selected name |
|---:|---|
| `0x80BE` | `Helvetica` |
| `0x80BF` | `Helvetica Neue` |
| `0x80C0` | Unresolved embedded-name location |
| `0x80C1` | `LucidaSans` |
| `0x80C2` | `Gill Sans MT` |
| `0x80C3` | `Podium Sans` |
| `0x80C5` | `Myriad Set Medium` |
| `0x80C6` | `NewsGothic` |
| `0x80C7` | `StoneSansBold` |
| `0x80C8` | `Univers55` |
| `0x80C9` | `Univers65Bold` |

Within this region, `0x80C4` and every other unmatched value enter fallback lookup.

The middle region contains these associations:

| Selector | Selected name |
|---:|---|
| `0x80B8` | `AppleLiGothicMedium` |
| `0x80B9` | Unresolved embedded-name location |
| `0x80BA` | `HiraginoMinchoPro` |
| `0x80BB` | `Espy Sans` |
| `0x80BC` | `Frutiger` |

Every unmatched value in this region enters fallback lookup.

In the lower region:

- Equality with the third boundary selects that boundary’s unresolved embedded-name location.
- If the selector is signed-less than the third boundary, an additional unsigned comparison selects the same location only for `0xFFFFFFFE` and `0xFFFFFFFF`.
- If the selector is signed-greater than the third boundary, `0x80B5` selects `Charcoal` and `0x80B6` selects `Chicago`.
- All remaining cases enter fallback lookup.

The unsigned comparison is strictly greater than `0xFFFFFFFD`. It excludes `0xFFFFFFFD` itself. All three boundary-equality checks compare complete 32-bit values; only the routing comparisons interpret those values as signed.

### Fallback lookup

Fallback lookup proceeds only when embedded selection has not resolved the pointer.

The first attempt calls F-8967e4fb3bba with a fresh read of the shared global. A non-null result becomes the selected name pointer and ends lookup.

If that result is null, F-9d0ecc5f6adf is called with no explicit arguments. After it returns, F-5710a8628cba receives three visible arguments, in this order:

1. The result from F-9d0ecc5f6adf.
2. The entry-captured global value.
3. A new read of the record’s selector.

The selector used here need not equal the selector used for routing if intervening effects changed the record. The original routing decision is not repeated.

If the three-argument lookup returns null, the context’s 32-bit field at `0x38` is read. F-5710a8628cba is then called with two visible arguments: that context field and the entry-captured global value.

The different visible argument counts do not establish the dependency’s true signature or the meaning of an absent third argument. There is no visible null check after the final attempt. Its result, including null, proceeds to completion.

### Completion

After name resolution, the full 32-bit word at record offset `0x04` is read. Exactly zero is replaced with `6`. The resulting value is masked to its low 16 bits for F-59c1430f85c2.

Default substitution occurs before masking:

| Original word | Argument after substitution and masking |
|---:|---:|
| `0x00000000` | `0x0006` |
| `0x00010000` | `0x0000` |
| `0x12345678` | `0x5678` |
| `0xFFFFFFFF` | `0xFFFF` |

The function then reads the byte at record offset `0x0D` and stores it through the byte-output address.

After this store, it reads the complete word at offset `0x0C` and converts that word to a character-width value. The visible conversion is not a direct byte read from a specified record offset. Character signedness and the precise numeric interpretation of out-of-range values remain target-dependent and unresolved.

F-59c1430f85c2 then receives:

1. The temporary-storage address.
2. The selected name pointer, possibly null.
3. The size-like value restricted to 16 bits.
4. The character-converted word.

If it returns normally, F-8e04864cbbaa receives the original destination argument and the same temporary-storage address. If that call returns normally, F-ccb5d9751852 receives the original destination argument and the record address plus `0x08`.

The function then returns without an explicit result.

## Side effects

### Observable ordering

The common completion path establishes this order:

1. Finish name resolution.
2. Read the size-like word and determine its defaulted value.
3. Read the source byte at `0x0D`.
4. Store that byte through the output pointer.
5. Read the word at `0x0C`.
6. Call F-59c1430f85c2.
7. Call F-8e04864cbbaa.
8. Call F-ccb5d9751852.

All lookup calls precede the size-like read and output-byte store. The output-byte store precedes all three completion dependencies.

No rollback is visible. If a later dependency does not return normally, the earlier byte store and any earlier dependency effects have already occurred.

### Aliasing and mutation

No non-aliasing requirement is established.

If the output pointer targets a byte within the word at `0x0C`, the subsequent full-word read observes the intervening store under ordinary sequential execution. The resulting character argument depends on the target byte, byte order, and conversion rules.

If the output pointer targets the size-like word, the already-read size value remains the value used for the current invocation. Later consumers may nevertheless observe the modified memory.

If the output aliases memory later inspected through the address at `0x08`, F-ccb5d9751852 may observe the changed byte. No snapshot of that region is made.

The function does not explicitly store into the shared global or record through their primary access expressions. Aliasing and dependency effects can still mutate either. No dependency is assumed to preserve global state, input memory, destination contents, or temporary contents.

### Ownership and lifetime

There is no explicit allocation, release, or destruction operation in this function. Allocation or ownership effects inside dependencies remain unknown.

The temporary storage is local to this invocation. Its address is passed to two dependencies, but the visible function does not establish whether retention is allowed or whether destruction is required. It performs no explicit cleanup afterward.

The function does not visibly copy, free, or transfer ownership of the selected name’s contents. Forwarding a pointer alone does not establish any ownership rule.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-8967e4fb3bba | First fallback lookup; receives a fresh global value. Its result is tested for null. |
| F-9d0ecc5f6adf | Called after the first lookup returns null. Its result is forwarded to the next lookup. |
| F-5710a8628cba | Receives three visible arguments on the first attempt and two on the optional final attempt. Each result is used as a candidate name pointer. |
| F-59c1430f85c2 | Receives temporary storage, the name pointer, the masked size-like value, and the character-converted word. No explicit result is used. |
| F-8e04864cbbaa | Receives the destination and temporary storage after F-59c1430f85c2 returns. |
| F-ccb5d9751852 | Final call; receives the destination and record address plus `0x08`. |

Null acceptance, memory access extents, ownership, global preservation, errors, exceptions, and normal return are not established for these dependencies.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless an example states otherwise, reached accesses are valid and dependencies return normally.

1. **Embedded high-region selection.** If `0x80BE` reaches the high region, the selected pointer names `Helvetica`. No fallback dependency is invoked.

2. **High-region gap.** If `0x80C4` reaches the high region, it enters fallback lookup. It does not select an embedded name through that region.

3. **Boundary precedence.** If the Osaka boundary equals a numeric selector otherwise listed in a later region, equality selects `Osaka` before that later association is considered.

4. **Three unresolved locations.** A qualifying third-boundary equality, middle-region `0x80B9`, and high-region `0x80C0` select their respective unresolved embedded-name locations. No particular name content or equality between those contents is required by the available evidence.

5. **First fallback success.** If F-8967e4fb3bba returns non-null, F-9d0ecc5f6adf and F-5710a8628cba are not called. The returned pointer is forwarded to F-59c1430f85c2.

6. **Second fallback success.** If the first lookup returns null and the three-argument lookup returns non-null, this function does not directly read the context field at `0x38`.

7. **Lookup exhaustion.** If all lookup attempts return null, F-59c1430f85c2 is still called with null as its name argument. Successful completion cannot be assumed.

8. **Global mutation.** If a fallback dependency changes the shared global, subsequent lookup arguments still use the entry-captured value where specified. They do not refresh that snapshot.

9. **Selector mutation.** If F-9d0ecc5f6adf changes the record selector, the following three-argument lookup receives the changed selector. Selection routing is not restarted.

10. **Default before truncation.** A size-like word of zero supplies `6`; `0x00010000` supplies zero. Applying the default after truncation would not match this contract.

11. **Aliasing output.** If the output pointer targets a byte of the word at `0x0C`, the full-word read follows that store. An implementation that snapshots the word before the store can differ observably.

12. **Unsigned lower boundary.** In the qualifying signed-lower region, `0xFFFFFFFE` and `0xFFFFFFFF` select the third unresolved name location. `0xFFFFFFFD` does not qualify through the unsigned comparison, although an earlier boundary equality could independently resolve it.

## Uncertainties

The following remain **documented_only**:

- The three unavailable selector-boundary values and unconditional reachability of numeric associations.
- The contents of all three unresolved embedded-name locations: the third-boundary location, the `0x80B9` location, and the `0x80C0` location.
- Whether any unresolved name contents are identical.
- Shared-global meaning, mutability, synchronization, and initialization.
- Semantic meanings of record fields beyond their observed uses.
- Target byte order, alignment, character signedness, and character-conversion interpretation.
- Complete dependency signatures, especially F-5710a8628cba’s differing visible argument counts.
- Destination representation and the complete temporary-storage layout.
- Dependency memory extents, ownership, retention, release, and cleanup requirements.
- Behavior on invalid addresses, null names, dependency failure, exceptions, or non-returning calls.
- Guarantees for concurrent mutation or changes to boundary data between reads.

No local error handling resolves these uncertainties. Font-related intent does not establish undocumented callee semantics.

## Evidence

Established behavior is derived solely from the supplied static function representation: access widths and offsets, comparison signedness, embedded-name labels, argument expressions, null checks, and call order.

The static evidence distinguishes three unresolved embedded-name locations. It also establishes full-width selector comparisons, signed routing boundaries, the unsigned threshold `0xFFFFFFFD`, zero substitution before low-16-bit masking, and the byte-output store before the later word read and completion calls.

Font selection and formatting are inferred intent. Dependency internals and target-specific details listed above remain unknown.

No runtime tests have been performed. The acceptance examples describe conditional static behavior, not observed execution results.
