<!-- Trusted provenance: F-de2b627bb9b5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-de2b627bb9b5

## Purpose

F-de2b627bb9b5 conditionally processes a pending object through a generation-associated phase and a publication-associated phase. It returns no value. Observable outcomes consist of dependency calls, direct memory changes, and early returns; there is no uniform success indication or visible rollback.

“Generation,” “publication,” “schema,” “name,” and “query” are inferred semantic labels, not guarantees about dependency implementations. The established behavior is the access, call, comparison, and mutation sequence described below.

Mode is observed repeatedly. Consequently, generation and publication are not mutually exclusive phases: dependency effects can allow both during one invocation.

## Interface

### Arguments

The function receives four arguments:

1. **Context:** a structure pointer.
2. **Ending descriptor:** a pointer whose first 32-bit word is inspected only during final bookkeeping.
3. **Alternate descriptor:** a nullable pointer used both as an entry gate and as a possible final fallback.
4. **Query input:** an opaque 32-bit value. Zero selects the no-query generation branch.

The supplied representation uses 32-bit pointers and words. Offsets are byte offsets. Unless otherwise specified, listed accesses are 32 bits.

| Structure | Offset | Observed role |
|---|---:|---|
| Context | `0x000` | Connection pointer, loaded on entry |
| Context | `0x040` | Status word |
| Context | `0x044` | Query-path state, written with `2` |
| Context | `0x144`, `0x148` | Metadata copied into temporary storage |
| Context | `0x148` | Also an explicit dependency argument |
| Context | `0x168` | Base value for the final calculation |
| Context | `0x180` | Pending-object pointer |
| Connection | `0x008` | Descriptor-array pointer |
| Connection | `0x00c` | Flags word |
| Connection | `0x01e` | Failure byte |
| Connection | `0x028` | Counter word |
| Connection | `0x07c` | Metadata copied into the object |
| Connection | `0x080` | Mode byte |
| Pending object | `0x000` | Name/key word |
| Pending object | `0x004`, `0x008` | Query-derived words |
| Pending object | `0x014` | Destination for connection metadata |
| Pending object | `0x018` | Zero/nonzero discriminator |
| Pending object | `0x024` | Associated-entry list head |
| Pending object | `0x02c` | Optional auxiliary input |
| Pending object | `0x030` | Final adjusted-result destination |
| Pending object | `0x038` | Conditional-work byte |
| Pending object | `0x04c` | Schema-associated pointer |
| Associated entry | `0x004` | Next pointer |
| Associated entry | `0x008` | Name/key word |
| Associated entry | `0x00c` | Dependency-result destination |

These are partial layouts, not complete type definitions.

### Preconditions and result

The context must support its initial connection-pointer load even when both optional inputs are absent.

Every address reached on the selected path must support the indicated access. No general pointer validation, descriptor-index bounds check, list-cycle detection, or span validation is visible.

A nonzero query input permits entry with a null alternate descriptor. It does not make the final fallback safe: if the ending descriptor’s first word is zero, the alternate descriptor is subsequently dereferenced.

No scalar result is returned. An early return may occur after mutations or dependency effects.

## Behavior

### Entry gates

The connection pointer is loaded from the context and retained.

If both the alternate descriptor and query input are zero, the function returns without inspecting the pending-object field.

Otherwise, the context status word is read. A nonzero value returns immediately, without reading the connection failure byte. If status is zero, that byte is read; nonzero also causes return.

The pending-object pointer is then loaded. Null causes return. The non-null pointer is retained rather than repeatedly fetched from the context.

F-49f750c2c596 receives the retained connection and the object’s schema-associated word. Its 32-bit result is retained as a descriptor index. The source does not establish a valid range for this result.

### Auxiliary processing and temporary layout

After index resolution, the object’s auxiliary word is tested. Zero skips this stage.

For nonzero auxiliary input, an unidentified external dependency is called twice: first with a temporary region of `0x24` bytes, then with another region of `0x38` bytes. No opaque ID or implementation is supplied for this dependency. Its calls do not establish that either region is cleared.

The subsequent explicit writes occur in this order:

1. A 16-bit `1` at second-region offset `0x00`.
2. The object’s then-current name word at second-region offset `0x0c`.
3. The 32-bit pattern `0xffffffff` at second-region offset `0x20`.
4. The second-region address at first-region offset `0x04`.
5. A one-byte `1` at first-region offset `0x16`.
6. The context address at first-region offset `0x00`.
7. The retained object address at second-region offset `0x14`.

F-efc2c252d6ce receives the first-region address and a fresh read of the object’s auxiliary word. That argument need not equal the value tested before the two external calls. A nonzero result causes immediate return.

The temporary layout matters beyond this stage:

- The 16-byte query workspace occupies second-region offsets `0x24` through `0x33`.
- A later common-continuation word occupies second-region offset `0x34`.
- Several later working words occupy the first region.

If auxiliary processing runs, these addresses have therefore already been exposed to a dependency. Whether any dependency retains or later uses them is unresolved.

### Mode selection

The retained connection’s mode byte is read.

- Zero enters generation.
- Nonzero first causes connection metadata at `0x07c` to be copied into object offset `0x014`; mode is then read again. Zero enters generation, while nonzero skips it.

The second mode observation is not replaced by the first. There is no intervening status or failure-byte recheck.

### Generation setup

F-9b3ea8470521 receives the context. A zero result returns immediately. A nonzero result is retained as a handle.

F-211638bb75da receives that handle, `0x20`, and zero.

The object discriminator is read to select one of two address-selected static references. The zero case selects a table-associated reference. The complete contents of the other reference are unavailable. The selected reference is used later without rereading the discriminator for that selection.

### Generation without query input

F-1983453691c7 receives the connection and an address-selected CREATE-associated format reference. Its result supplies a text-like working value. No explicit zero-result check follows.

The visible call representation does not establish additional formatting operands or the complete generated text.

### Generation with query input

The following sequence is established:

1. F-2ea344827b50 receives the handle, `9`, `1`, context metadata at `0x148`, and the retained index.
2. F-d2aa1dc1574b receives the handle and `1`.
3. Context offset `0x044` is written with `2`.
4. F-1e7ea6aa642e receives the 16-byte workspace, `8`, and `1`.
5. First-region words at offsets `0x00` and `0x04` are zeroed, in that order.
6. F-7a1ceda22360 receives the context, query input, workspace, zero, and zero.
7. F-211638bb75da receives the handle, `0x20`, and `1`.
8. The context status word is reread.

If status is zero, F-892de4fc8469 receives the context, zero, and query input. Zero result causes immediate return.

For a nonzero returned address, its word at `0x004` is read and copied into object offset `0x004`; then its word at `0x008` is read and copied into object offset `0x008`. The returned object’s words at `0x004` and `0x008` are subsequently cleared, in that order. F-da1a289a3277 is then called without explicit arguments shown.

These operations establish copying and clearing, not an allocation type or a complete ownership transfer. No visible check excludes aliasing.

If the status reread is nonzero, extraction is skipped but execution continues. In either continuing case, F-bae1c1d99f99 receives the connection and retained object; its result supplies the text-like working value.

### Common generation continuation

The following temporary writes occur before the first common F-66c61ffd0a02 call:

- The current object name is copied to first-region offset `0x00`.
- Context metadata at `0x148` is copied to first-region offset `0x08`.
- Context metadata at `0x144` is copied to first-region offset `0x10`.
- After catalog selection, the copied name is duplicated at first-region offset `0x04`.
- The text-like working value is copied to second-region offset `0x34`.

Index `1` selects a temporary-catalog static reference; every other index selects an ordinary-catalog static reference.

F-66c61ffd0a02 then receives:

1. The context.
2. **A global operand evaluated at this call site.**
3. The first word of the indexed descriptor.
4. The selected catalog reference.
5. The earlier discriminator-selected reference.

The second operand is not established to be an address-selected static format reference. The representation supplies a global operand directly. Its type, contents, and immutability remain unresolved; earlier dependency effects cannot be assumed irrelevant to its value.

The descriptor word is obtained using the connection’s then-current array pointer and a `0x18`-byte stride with the retained index. No index validation is visible. The representation does not establish a total evaluation order among this call’s independent argument expressions.

**After F-66c61ffd0a02 returns, second-region offset `0x34` is read again. That post-call value is supplied to F-94cb53877c15.** Equality with the previously produced text-like value requires that the intervening call leave the temporary word unchanged.

This distinction is material when auxiliary processing has exposed the `0x38`-byte region. An escaped alias could permit the intervening dependency to change offset `0x34`. Neither address-retention nor preservation guarantees are supplied. Thus F-94cb53877c15 is not unconditionally described as receiving the original producer result.

F-2e31112ab290 next receives the context and retained index.

The object’s conditional-work byte is then read. If nonzero, the current descriptor-array pointer is loaded again, the indexed descriptor’s pointer at `0x014` is followed, and target offset `0x054` is tested. Zero causes an additional F-66c61ffd0a02 call with the context and an address-selected sequence-table-associated static reference.

F-1983453691c7 then receives the connection, a name-predicate static reference, and the current object name. Its result is written to first-region offset `0x00`, followed by `0xffffffff` at offset `0x04`.

F-0b0cd8d831b7 receives the handle, `0x1e`, the index, zero, and zero.

The connection mode byte is reread. Zero returns immediately; nonzero permits publication-associated processing.

### Publication-associated processing

This stage proceeds only if the current context status word is zero. The connection failure byte is not rechecked.

The object’s current schema-associated pointer is loaded and retained for this stage. It need not equal the earlier value passed to F-49f750c2c596.

F-1cb585febd84 receives the current object name. F-30162766ee7c subsequently receives:

- The retained schema pointer plus `0x004`.
- A fresh read of the object name.
- The preceding 32-bit result plus one.
- The retained object pointer.

The schema pointer is retained across the first call; the name is reread. Dependency mutation can therefore make the two name arguments differ.

A nonzero F-30162766ee7c result causes the connection failure byte to be written with `1`. This branch performs no associated-entry traversal or final bookkeeping. In particular, this function does not subsequently clear the context pending-object field.

A zero result selects traversal; it does not independently establish insertion success.

### Associated entries

The list head is loaded from the retained object after the primary F-30162766ee7c call.

For each non-null retained entry:

1. Its key at `0x008` is read for F-1cb585febd84.
2. F-a09bb08bf954 receives the retained schema pointer plus `0x040`, a fresh key read, and the preceding result plus one.
3. That dependency’s result is stored at entry offset `0x00c`.
4. F-30162766ee7c receives the same schema-relative address, another fresh key read, the same earlier result plus one, and the entry pointer.
5. If its result equals the entry address, the connection failure byte is written with `1`.
6. The next pointer is read from entry offset `0x004`.

The three key reads can differ. The length-like result is not recomputed after F-a09bb08bf954. Next-link mutation by dependencies can affect traversal.

The equality-triggered failure write does not stop the loop or suppress bookkeeping.

### Final bookkeeping

After traversal:

1. Context offset `0x180` is cleared.
2. The connection counter is read and incremented.
3. Connection flags are read and written back with bit `0x10` set.

Clearing the context field does not establish that it still contained the retained object pointer.

The object discriminator is reread. Nonzero finishes the function.

For zero, the ending descriptor’s first word is tested. If zero, the alternate descriptor is selected. The selected descriptor’s first word is then read for the final calculation, including a reread when the ending descriptor remains selected.

F-b4209d504f85 receives the context base at `0x168` and the selected word minus that base. Its result plus `0x0d` is stored at object offset `0x030`.

No null, span-order, or arithmetic-range check protects this calculation.

## Side effects

Direct effects include temporary-region writes, conditional metadata copying, query-state assignment, query-derived copying and clearing, entry-result stores, failure-byte writes, pending-field clearing, counter increment, flags update, and final-result storage.

Dependencies may have additional effects. No unknown dependency is assumed to preserve globals, reachable fields, or exposed temporary memory. Such mutation remains possible, not asserted.

The connection and pending-object pointers are retained despite possible changes to their context fields. The descriptor index is retained. The publication-stage schema pointer is retained, while names, keys, and links are reloaded as described.

The function shows no general object release, rollback, or comprehensive ownership transfer. Passing temporary addresses does not establish permission or a valid lifetime for retaining them after return.

Index scaling, schema-relative addressing, result-plus-one arguments, counter increment, span subtraction, and final addition use 32-bit operands in the supplied representation. No wider intermediate or overflow guard is established. Exact signed-overflow and target-address consequences remain documented_only.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-49f750c2c596 | Produces the retained descriptor index |
| F-efc2c252d6ce | Receives auxiliary state; nonzero returns early |
| F-9b3ea8470521 | Produces a handle; zero returns early |
| F-211638bb75da | Receives handle, `0x20`, and zero or one |
| F-1983453691c7 | Produces values from static-reference-associated calls |
| F-2ea344827b50 | Receives handle, constants, metadata, and index |
| F-d2aa1dc1574b | Receives handle and one |
| F-1e7ea6aa642e | Receives workspace, `8`, and one |
| F-7a1ceda22360 | Receives context, query input, workspace, and zeros |
| F-892de4fc8469 | Produces an extraction address or zero |
| F-da1a289a3277 | Called after extraction and clearing |
| F-bae1c1d99f99 | Produces the query-path text-like value |
| F-66c61ffd0a02 | Receives the global operand in its common call; a static reference in its conditional call |
| F-94cb53877c15 | Receives the post-call read of second-region offset `0x34` |
| F-2e31112ab290 | Receives context and index |
| F-0b0cd8d831b7 | Receives handle, `0x1e`, index, and zeros |
| F-1cb585febd84 | Produces a value later increased by one |
| F-a09bb08bf954 | Produces the word stored in an entry |
| F-30162766ee7c | Produces results compared with zero or an entry address |
| F-b4209d504f85 | Produces the final value before addition of `0x0d` |

The external temporary-region dependency has no supplied opaque ID. String measurement, lookup, insertion, formatting, and release are inferred roles rather than established dependency contracts.

## Acceptance examples

These are conditional static expectations, not executed tests.

- **Absent optional inputs:** The connection pointer is loaded, then return occurs without pending-object inspection.
- **Initial status failure:** Nonzero status prevents even the initial failure-byte read.
- **Auxiliary reload:** If either external call changes auxiliary input to zero, F-efc2c252d6ce receives zero; the already-entered stage is not retroactively skipped.
- **Stable nonzero mode:** Metadata is copied, generation is skipped, and current zero status permits publication.
- **Stable zero mode:** Given a nonzero handle and no earlier return, generation runs; zero post-generation mode prevents publication.
- **Temporary preservation:** If second-region offset `0x34` remains unchanged during the common F-66c61ffd0a02 call, F-94cb53877c15 receives the copied producer result.
- **Escaped temporary alias:** If auxiliary processing exposed that region and the common call changes offset `0x34` through a retained alias, F-94cb53877c15 receives the changed word.
- **Global mutation:** If an earlier dependency changes the global operand, the common call uses its call-site evaluation, not an assumed immutable format reference.
- **Primary name mutation:** A changed name is supplied to F-30162766ee7c with the earlier result plus one and retained schema-relative address.
- **Primary nonzero result:** The failure byte becomes `1`; this function performs no later pending-field clear on that branch.
- **Entry-address match:** The failure byte is set, but traversal and final bookkeeping continue.
- **Unsafe fallback:** A zero ending word selects the alternate descriptor even if it is null.
- **Final arithmetic:** With stable base `100` and selected word `140`, F-b4209d504f85 receives `100` and `40`. A result of `7` produces a stored value of `20`.

## Uncertainties

The following remain **documented_only**:

- Dependency implementations, preservation guarantees, allocation, ownership, and unobserved error conventions.
- The global operand’s type, contents, and immutability.
- Complete static text and any formatting operands absent from the visible call representation.
- External temporary-region helper semantics and address retention.
- Whether names and keys are strings, their encoding, and result units.
- Semantic meanings of operation constants, discriminator, counter, and final adjustment.
- Aliasing restrictions, concurrency rules, invalid-memory behavior, and exact overflow consequences.
- Whether F-94cb53877c15 releases anything, and whether its post-call argument equals the earlier producer result.

## Evidence

Established evidence consists of the supplied static representation: access widths and offsets, overlapping temporary layout, branch conditions, explicit arguments, global-operand use, result comparisons, and direct-operation order.

In particular, the text-like value is copied into second-region offset `0x34` before the common F-66c61ffd0a02 call and read afterward for F-94cb53877c15. That call’s second operand is supplied as a global operand, unlike the explicitly address-selected static references elsewhere.

Semantic labels and likely dependency roles remain inferences. No external implementation was inspected. No runtime tests have been performed.
