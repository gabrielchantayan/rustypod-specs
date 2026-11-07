<!-- Trusted provenance: F-1bea98cfcfdf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1bea98cfcfdf

## Purpose

F-1bea98cfcfdf searches an owner-associated collection for an entry whose 64-bit identity equals the identity obtained from a supplied object immediately before that entry’s identity is read.

A successful comparison leads to a separate collection access. If that access returns a non-null object, the function performs an ordered sequence of callbacks, temporary-storage operations, counter decrements, notification, and an indexed collection operation. A negative collection count also reaches this second access, without a successful comparison.

**Inferred intent:** removal or detachment by identity. This is not an established end-to-end guarantee. The second access has no explicit index in the supplied representation, and the final indexed operation’s effects are dependency-defined.

The 32-bit return value has path-dependent meaning. It is not established as a Boolean, uniform status code, or consistently usable address.

## Interface

### Inputs

| Argument | Width | Established use |
|---|---:|---|
| Owner address | 32 bits | Forms the collection address at owner plus `0x18`; supplies the signed count at owner plus `0x1c`. |
| Identity-source object address | 32 bits | Supplies virtual-dispatch metadata for the identity operation at table offset `0x0c`. |
| Auxiliary word | 32 bits | Retained initially, then replaced locally by the low 32 bits of each identity-source result obtained during search. |
| Opaque word | 32 bits | Copied into the temporary frame without interpretation. |

The collection address is the owner-relative address itself. The function does not first load a collection pointer from owner offset `0x18`.

The identity-source object is only dereferenced when a search access returns a non-null candidate. Zero or negative counts perform no search identity calls.

### Output

The function returns one 32-bit word:

- **Count zero:** the original owner-address word.
- **Positive count, search exhausted:** zero if the final candidate is null; otherwise the low 32 bits of the final candidate’s identity.
- **Execution reaching frame preparation:** the frame’s first word at return.

A returned identity word with its high bit set retains that bit pattern. Its possible signed interpretation does not establish an error indication.

There is no separate visible error channel.

### Preconditions for normal completion

The function does not visibly validate the following requirements:

- The owner’s count field is readable.
- Owner-relative address formation yields addresses usable by the invoked operations.
- F-0d1bcade3bad accepts both observed invocation forms.
- Objects subjected to virtual dispatch have readable dispatch metadata and callable entries at the required offsets.
- The selected object remains usable through its final virtual call, including after F-6c19e1d86be0.
- F-b9acea4b06ad returns storage with a writable first 32-bit word.
- Accessed global storage, holder addresses, and context fields are valid.
- Dependencies accept the frame addresses and other supplied arguments.

A nonnegative count is consistent with the apparent search intent, but negative counts have distinct established behavior.

Invalid memory, incompatible dispatch metadata, or dependency failures are not converted into documented error returns. No local recovery mechanism is visible.

## Behavior

### Count capture and traversal

The function reads the signed 32-bit count at owner offset `0x1c` once before traversal. That captured value controls the search. Changes to the count field by dependencies do not visibly refresh the bound.

For a positive count, positions are visited in ascending order, beginning at zero and ending no later than count minus one. Each search access invokes F-0d1bcade3bad with:

1. The original owner-derived collection address.
2. The current position.

After each access, the prospective no-match return word is set to zero.

A null candidate causes the search to continue without invoking either identity operation. For a non-null candidate, the order is:

1. Invoke the identity-source object’s virtual operation at table offset `0x0c`, passing that object.
2. Retain its complete 64-bit result for comparison and replace the local auxiliary word with its low 32 bits.
3. Invoke the candidate’s virtual operation at table offset `0x0c`, passing the candidate.
4. Set the prospective no-match return word to the low 32 bits of the candidate’s result.
5. Compare the two complete 64-bit results.

The first full-width equality stops traversal. Equal low halves alone are insufficient.

The identity-source operation is repeated for every non-null candidate; its result is not cached across positions. Each comparison therefore uses the identity-source result obtained immediately before that candidate’s identity operation.

Neither identity operation is established as pure or stable. Dependencies may change objects, collection contents, or global state. The captured traversal bound nevertheless remains unchanged.

### Search termination and count boundaries

When the final search position equals the captured count, the function returns the prospective no-match word immediately. No frame-processing sequence follows.

For count zero, the initial position already equals the count. The function returns the owner-address word without invoking the collection accessor.

For a negative count, the signed traversal condition is false initially, but position zero does not equal the captured count. Execution proceeds to frame preparation and the second collection access. The saved position is zero, and the auxiliary word remains the incoming third argument.

This negative-count behavior is established by the visible control flow; it does not establish that negative counts are intended inputs.

For a positive signed 32-bit count, traversal does not require incrementing beyond the maximum signed 32-bit value. Exhaustion reaches the captured count and exits.

### Temporary frame

On either a matching positive-count path or a negative-count bypass, four contiguous 32-bit words are initialized in this order:

| Offset from frame base | Initial contents |
|---|---|
| `0x00` | Original owner-address word |
| `0x04` | Saved search position |
| `0x08` | Current auxiliary word |
| `0x0c` | Incoming opaque word |

The frame occupies 16 bytes. On an ordinary matching path, its auxiliary word is the low half of the identity-source result used in the matching comparison.

The frame is mutable. Initial field meanings do not establish later contents or the meaning of the returned first word.

The original owner and saved search position remain independently available. Mutation of their frame copies does not visibly redirect the final indexed collection operation.

### Second collection access

After frame initialization, F-0d1bcade3bad is invoked again with the original owner-derived collection address. No position argument is explicit in this invocation’s supplied representation.

If the result is null, selected-object processing is skipped and the frame’s first word is returned. That word was initialized to the owner-address word; no subsequent direct frame write or explicit exposure of the frame to a dependency occurs on this branch.

If the result is non-null, it becomes the selected object. The supplied representation does not establish that this object is the matching candidate or that this access selects the saved search position.

### Ordered selected-object processing

For a non-null second result, the visible sequence is:

1. Invoke F-bfade2068003 with `0x1b` and zero.
2. Invoke the selected object’s virtual operation at table offset `0x10`, passing the frame base first and the selected object second.
3. Invoke F-b9acea4b06ad with the address of frame offset `0x08`, followed by the frame base.
4. Read the then-current 32-bit global value and write it into the first word of the storage returned by F-b9acea4b06ad.
5. Invoke F-f13a263e790a with that returned address and zero.
6. Invoke F-289ec4311c64 with the address of frame offset `0x08`.
7. Invoke F-289ec4311c64 with the frame base.
8. Invoke F-6c19e1d86be0 with the selected object.
9. Read the outer global holder storage and retain the resulting holder address for notification.
10. Read that outer global storage again, then dereference this second holder address to obtain the counter-context address.
11. Read the 32-bit field at counter-context offset `0x428`, decrement it, and write the result back.
12. Read the 32-bit field at the same counter-context offset `0x424`, decrement it, and write the result back.
13. Invoke the selected object’s virtual operation at table offset `0xac`, passing that object and obtaining a 64-bit result.
14. Dereference the holder retained in step 9, then read the 32-bit routing word at offset `0x30` from the context it now references.
15. Invoke F-957fde6b6d36 with the routing word, the high 32 bits of the virtual result, its low 32 bits, and zero, in that order.
16. Invoke F-494c11edf385 with the original owner-derived collection address and the saved search position.
17. Invoke F-85722f62e20a with `0x1b` and `1000`.
18. Return the frame’s first word as it then exists.

The global value used in step 4 is read after F-b9acea4b06ad returns. Earlier dependencies are not assumed to preserve it.

Likewise, the counter decrements use values read at steps 11 and 12, not entry-time values. Earlier dependencies, including F-6c19e1d86be0, may change those fields before the direct reads.

The two outer-global reads are distinct in the supplied representation. Their results are not assumed equal. Both decrements use the single counter-context address obtained through the second read. Notification later dereferences the holder retained from the first read; it does not reevaluate the outer global.

No dependency return status visibly gates the remaining sequence. A fault, exception, or other nonlocal exit may interrupt execution; no rollback is visible.

## Side effects

### Objects and collection

Search identity operations occur only for non-null candidates, with the identity-source operation immediately preceding the candidate’s identity operation.

The selected object receives the frame-processing callback and is passed to F-6c19e1d86be0. It is subsequently dereferenced for dispatch at offset `0xac`. This later use does not prove that F-6c19e1d86be0 preserves its lifetime or ownership.

F-494c11edf385 runs after notification and receives the independently saved search position. Its exact mutation, removal, and ownership effects are unknown.

### Frame and returned storage

The frame has local storage duration. Dependencies receive its base and its upper eight-byte region and may modify accessible contents.

F-b9acea4b06ad returns an address whose first word is overwritten directly. There is no visible null check. Whether the returned storage aliases the frame is unresolved.

The calls to F-f13a263e790a and F-289ec4311c64 do not independently establish construction, destruction, reference counting, or deallocation semantics. They are not assumed to preserve the first frame word.

No lasting ownership transfer of the frame is established. Retaining its address for later use would require a lifetime arrangement not supplied here.

### Global state and arithmetic

The global value read for the direct storage overwrite is a late read, not an entry-time snapshot.

The two holder addresses may differ and may reference either the same context or different contexts. Both counter updates use the context loaded through the second holder.

Counter update order is `0x428`, then `0x424`. Each update reads that field’s current 32-bit value and writes its decremented value. No visible zero check, saturation, or lower-bound enforcement occurs. A value of zero at a decrement’s read produces negative one under ordinary signed arithmetic.

The representation uses signed 32-bit arithmetic for these decrements. Behavior at signed minimum is unresolved; wrapping or saturation is not silently imposed.

After the final selected-object virtual call, notification dereferences the retained holder afresh. That holder may now reference a different context. Replacing the outer global alone does not change which holder has already been retained.

The selector calls do not establish locking, transaction boundaries, timers, or restoration guarantees. No atomicity or synchronization guarantee is supplied.

## Dependencies

| Opaque function ID | Observed interface and unresolved boundary |
|---|---|
| F-0d1bcade3bad | Collection access. Search calls explicitly supply an index; the later call does not. Complete ABI, selection, and ownership are unresolved. |
| F-bfade2068003 | Receives `0x1b` and zero before selected-object processing. |
| F-b9acea4b06ad | Receives upper-frame and frame-base addresses; returns storage whose first word is directly overwritten. |
| F-f13a263e790a | Receives that returned storage and zero after the overwrite. |
| F-289ec4311c64 | Receives the upper-frame address, then the frame base, in separate calls. |
| F-6c19e1d86be0 | Receives the selected object before the holder reads, counter updates, and later virtual dispatch. |
| F-957fde6b6d36 | Receives a routing word, high and low halves of a 64-bit result, and zero. |
| F-494c11edf385 | Receives the original collection address and saved search position. |
| F-85722f62e20a | Receives `0x1b` and `1000` after the indexed collection operation. |

Dynamic virtual dependencies occur at table offsets `0x0c`, `0x10`, and `0xac`. No independent opaque IDs are supplied for their implementations. Their internal behavior and ownership effects remain unknown.

## Acceptance examples

These are conditional static acceptance criteria, not runtime observations. Processing examples require valid accessed storage and normally returning dependencies. Normal return alone does not imply preservation of global state, object state, frame contents, or counters.

1. **Empty collection:** With count zero, return the original owner-address word. Do not invoke the collection accessor or identity operations.

2. **All-null search:** With count three and null candidates at all three positions, return zero. Do not invoke the identity-source operation.

3. **Final mismatch:** With a positive count, no match, and a final non-null candidate whose identity has low word `0x12345678`, return that word.

4. **Final null:** A final null candidate sets the no-match return word to zero, overriding a previous mismatching candidate’s low identity word.

5. **Full-width equality:** Identity results `0x0000000100000007` and `0x0000000200000007` do not match, despite equal low halves.

6. **Repeated source identity:** Two non-null candidates, with the first mismatching, require two identity-source invocations. Reusing the first identity-source result would not preserve the established behavior.

7. **Null second access:** After a match, a null second access skips selector calls, frame callbacks, direct counter updates, notification, and the final indexed operation. The frame’s first word remains the initialized owner-address word absent an unestablished out-of-band modification.

8. **Negative count:** A negative count performs no search accesses or identity calls but still attempts the second access. The initialized frame contains position zero and the incoming auxiliary word.

9. **Late global-value read:** If F-b9acea4b06ad changes the global value from `0x11111111` to `0x22222222`, the direct overwrite uses `0x22222222`, provided that is still the value when the read occurs.

10. **Different holder reads:** If the first outer-global read yields holder A and the second yields holder B, with B referencing context C, both direct decrements address fields in C. Notification later dereferences A, not B.

11. **Changed notification context:** If the retained holder initially references context A but the final virtual operation changes it to reference context B, notification reads routing offset `0x30` from B. The earlier counter updates used the already loaded counter context.

12. **Outer-global replacement:** If the final virtual operation replaces the outer global without changing the retained holder’s contents, notification still uses the retained holder.

13. **Counter values at their reads:** If the field at offset `0x428` contains five when its decrement reads it, that update writes four. If the field at offset `0x424` contains nine when its decrement reads it, that update writes eight. Both writes precede the final virtual operation and notification. Values of five and nine at function entry alone do not establish these results, because earlier dependencies may change the fields.

14. **Frame mutation:** If a callback changes the frame’s owner and position words, the final indexed operation still receives the original owner-derived collection address and independently saved position. The returned first word may reflect frame mutation.

15. **Notification argument halves:** If the final virtual result is `0x1122334455667788`, notification receives `0x11223344` before `0x55667788`, following the routing word and preceding zero.

## Uncertainties

The following remain **documented_only**:

- The complete ABI and any implicit argument requirements of the second collection access.
- Whether the second access returns the matching candidate.
- The meanings of identities, selectors, counters, routing words, and frame fields.
- Allocation, release, retention, and ownership effects of dependencies.
- Whether returned temporary storage aliases the frame.
- The intended interpretation of the returned frame word.
- External mechanisms permitting changes between distinct global reads.
- Signed-minimum decrement behavior and complete address-overflow semantics.
- Concurrency, reentrancy, atomicity, synchronization, and nonlocal-exit behavior.
- Complete object layouts and target byte order beyond the accessed widths and offsets.

No dependency is assumed to preserve globals or frame contents, or to keep an object alive, merely because later execution relies on those properties.

## Evidence

This contract uses the supplied static function representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent behavioral evidence.

Established static observations include the captured signed count, ascending indexed search, repeated 64-bit identity evaluation, full-width comparison, negative-count bypass, four-word mutable frame, separate second collection access, late global-value read, distinct outer-holder reads, ordered counter read-modify-writes, retained-holder notification routing, and branch-specific return words.

The counter writes depend on field values at their respective decrement reads. Entry-time counter values are insufficient because intervening dependencies are not established to preserve them.

Removal by identity and cleanup-like interpretations remain inferred intent. Dependency bodies, complete ABI information, and runtime behavior were not supplied. No runtime tests have been performed.
