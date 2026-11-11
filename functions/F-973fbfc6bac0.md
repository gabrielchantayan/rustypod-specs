<!-- Trusted provenance: F-973fbfc6bac0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-973fbfc6bac0

## Purpose

F-973fbfc6bac0 scans a caller-supplied byte range using a mutable parser object. It recognizes selected prefixes, dispatches slash-prefixed names through a descriptor table, and invokes a binary-processing dependency when an introducer follows a classifier-positive token.

Special handling searches beyond `FontDirectory` for `known`. Other paths stop on qualifying `eexec` or `closefile` prefixes.

A font-program parsing role and numeric-token interpretation are inferred intent, not established grammar. This contract specifies visible operations and their ordering without assuming that dependencies preserve parser fields, input bytes, descriptor contents, global state, or memory ownership.

## Interface

### Arguments and result

The five arguments are:

1. An opaque 32-bit context value, retained for descriptor dispatch.
2. A pointer to mutable parser storage.
3. A 32-bit input-start address.
4. A 32-bit extent, added to the start to form the retained end address.
5. A pointer to recognition flags, one byte per descriptor position.

The result is a 32-bit integer. Ordinary exits read and return the parser status field. One binary-processing failure path instead returns the literal `3` without directly writing `3` to that field.

The visible representation uses 32-bit addresses, arithmetic values, parser slots, and callable addresses. Wider arithmetic must not silently change the observed range or length conditions. The name-length eligibility expression specifically uses unsigned 32-bit subtraction.

Defined use requires accessible parser storage, input, descriptors, flags, and callable targets. The evidence does not establish safe behavior for null pointers, negative extents, wrapping address arithmetic, out-of-range cursors, or insufficient storage.

### Parser layout

| Byte offset | Width | Directly used role |
|---|---:|---|
| 0 | 32 bits | Current input address |
| 8 | 32 bits | Stored end address |
| 12 | 32 bits | Status |
| 28 | 32 bits | Pre-inspection callback address |
| 32 | 32 bits | Advance callback address |
| 56 | 32 bits | Special callback used after finding `known` |

The callback names describe placement, not guaranteed lexical semantics.

### Descriptor table and flags

The descriptor-table root is a **mutable global pointer**, not a function argument or an immutable table address.

Each record occupies seven 32-bit words, a 28-byte stride. Its first word is a name pointer. A null name pointer ends traversal. Other record fields are opaque here; dispatch receives the matched record’s address.

The flag pointer begins at the supplied flag-array address for each lookup. It advances by one byte for each unmatched record. The first matching record ends traversal, including when its flag suppresses dispatch.

No ownership transfer is directly established for input storage, parser storage, flags, descriptors, or names.

## Behavior

### Initialization and outer iteration

The function performs these parser writes in order:

1. Current address becomes the supplied start.
2. Status becomes zero.
3. Stored end becomes start plus extent.

The computed end is retained locally. Subsequent range decisions use that retained value, not later contents of the stored-end field.

The function then caches the callback address at offset 28, clears its pending-token state, and initializes a saved-token address to null.

Every outer iteration first invokes the cached pre-inspection callback. Only afterward does it read the parser’s current address into a retained local input address and compare it with the retained end. An address at or beyond the end causes an ordinary return.

Consequently, even an initially empty range invokes the callback. A status set by that callback does not itself cause an immediate exit.

On a continuing iteration, the callback address at offset 28 is reloaded at the bottom for use by the next outer iteration. Internal calls in the `FontDirectory` branch load callback addresses at their individual call sites. Advance calls likewise obtain the callback from offset 32 when called.

### Input-byte observations

The input byte is not captured once for the entire iteration:

- The function first reads the byte at the retained input address to select the initial `F` branch.
- If a qualifying `FontDirectory` comparison returns nonzero, the stop-prefix path rereads the byte at that same retained address.
- The `e` and subsequent `c` conditions inspect input memory separately. A failed `eexec` comparison can therefore be followed by a `c` decision using bytes changed by that comparison dependency.
- Classification reads the byte at the retained address for its argument.
- If classification returns zero, the function reloads that byte before choosing `RD`, `-|`, slash handling, or ordinary advancement.

These reads do not obtain a new address from the parser. Dependencies may change the parser cursor, but the current iteration’s prefix and classification decisions continue to use the retained input address unless a branch explicitly selects another address.

A short initial `F` case bypasses stop-prefix checks and proceeds to classification. Likewise, an `e` case with insufficient room for its stop comparison proceeds directly to classification.

### `FontDirectory` search

When the initial byte is `F`, comparison against `FontDirectory` occurs only if the retained end is strictly beyond input-address-plus-25. Thus at least 26 bytes must remain, although the comparison length is 13.

A nonzero comparison result enters the stop-prefix path, including its fresh byte read. A zero result performs:

1. An advance callback.
2. A pre-inspection callback.
3. A parser-cursor read saved as the fallback address.
4. A forward search for `known`.

There is no status check between the first two calls or immediately after them.

At each search position, `known` is compared only if the byte is `k` and the retained end is strictly beyond position-plus-five. The comparison length is five; no boundary after the prefix is checked.

For an unsuccessful search position, the function calls advance and then checks status. Nonzero status exits immediately, without fallback restoration. Otherwise it calls pre-inspection, reloads the parser cursor as the next search position, and repeats.

If a qualifying `known` comparison succeeds before the end, the function calls advance and then the callback at offset 56, passing the parser and a scratch-output address. It does not check status between or immediately after these calls.

A scratch byte is then tested against `3`. If equal, the parser’s current address replaces the fallback address. The function writes the selected fallback address to the parser cursor and clears pending-token state.

If the search reaches or passes the end, it restores the original fallback address and clears pending-token state.

Only the cursor is restored. Status, input mutations, other parser fields, and dependency effects are not rolled back.

The tested scratch byte is not in the same scratch region whose address is passed to the offset-56 callback. No visible initialization establishes its value for this branch. The output extent and relationship between these regions remain unresolved.

### Stop prefixes

Outside a successful `FontDirectory` branch:

- `eexec` stops scanning when the inspected byte is `e`, the retained end is strictly beyond address-plus-five, and the five-byte comparison returns zero.
- `closefile` stops scanning when the inspected byte is `c`, the retained end is strictly beyond address-plus-nine, and the nine-byte comparison returns zero.

Both are prefix checks, not whole-token checks. They require at least six and ten remaining bytes respectively. Exact five-byte `eexec` and nine-byte `closefile` suffixes do not satisfy these guards.

A recognized stop prefix causes an ordinary return without an advance call or direct cursor write for that prefix. Dependency changes already made remain observable.

### Classification and pending-token state

F-cfad0582408b receives a byte read from the retained input address.

For a nonzero result, the function calls advance and checks status. Nonzero status ends scanning. Otherwise, pending-token state becomes true and the retained input address becomes the saved-token address.

A later classifier-positive iteration replaces the saved address. What advance consumed, and whether the classifier recognizes digits or another class, are not established.

For a zero result, the function rereads the byte at the retained address and selects the following handling.

### Binary introducers

`RD` and `-|` qualify only when the retained end is strictly beyond address-plus-six: at least seven bytes must remain. The first byte comes from the post-classifier reload; the second byte is read when the corresponding guard permits it. Only those two introducer bytes are checked.

With pending-token state true:

1. The parser cursor is written with the saved-token address.
2. F-c6d4679527d4 receives the parser and two scratch-output addresses.
3. A zero result causes an immediate literal return of `3`.
4. A nonzero result clears pending-token state and continues at the next outer iteration.

There is no status check after a nonzero helper result. There is no additional advance call on that path.

Without a pending token, a qualifying introducer falls through to ordinary advancement. Insufficient remaining length likewise prevents special binary handling.

### Slash-name consumption and table capture

Slash handling requires the post-classifier byte to be `/` and at least three remaining bytes.

The function advances the retained input address by one, writes that address to the parser cursor, and invokes advance. Nonzero status causes an ordinary exit.

After successful advancement, operations occur in this order:

1. Read the parser cursor and compute the consumed name length relative to the address immediately after the slash.
2. Initialize the local flag position to the supplied flag-array address.
3. Load the mutable global descriptor-table root.
4. Evaluate name-length eligibility.
5. If eligible, reread the parser cursor and require it to be strictly before the retained end.

The global-root load therefore occurs even when length or end-position eligibility later excludes traversal.

Length eligibility is the unsigned 32-bit condition that length-minus-one is less than 21. Under the stated valid-range preconditions, admitted lengths are 1 through 21. A name ending exactly at the retained end is not dispatched.

Traversal uses the captured table root. Later dependency writes to the global root do not redirect that traversal. They may affect a later lookup, which loads the root again.

### Record matching and ordered dispatch decision

For each visited record, the name pointer is loaded once. Null ends traversal. A nonnull pointer is retained through that record’s first-byte check, length call, byte-comparison call, and any special-name calls.

Matching proceeds by short-circuit evaluation:

1. Compare the input name’s first byte with the descriptor name’s first byte.
2. If equal, call F-1cb585febd84 with the captured descriptor-name pointer and require its result to equal the consumed length.
3. If lengths agree, call F-7f14f1ab6851 with the input name, captured descriptor-name pointer, and consumed length; zero selects a match.

Name bytes are not snapshotted. A retained pointer can still refer to bytes mutated by a dependency. Input first-byte checks are performed again for later records.

After a match, dispatch selection is ordered:

1. Read the corresponding flag byte.
2. If zero, permit dispatch without either special-name comparison.
3. Otherwise, compare the captured descriptor name with `Subrs`.
4. If that comparison returns zero, permit dispatch without comparing `CharStrings`.
5. Otherwise, compare the captured descriptor name with `CharStrings`; zero permits dispatch.

The special-name comparison dependency has no supplied opaque identity. The flag and descriptor-name pointer are not reread after these calls. Mutating the record’s name-pointer field does not replace the pointer already selected for them.

For permitted dispatch, F-b6cdf3faef95 receives the retained context value, parser, and matched record address. Its 32-bit result is written to status before result handling:

- Zero: write exactly `1` to the selected flag byte.
- Nonzero with low eight bits `0xA2`: clear status to zero; make no direct flag write.
- Any other nonzero result: exit through the status-return path.

The low-byte exception includes values such as `0x000001A2`, not just `0x000000A2`.

The first match ends traversal even if dispatch is suppressed. An unmatched record advances both the record position by 28 bytes and the flag position by one byte.

Continuing slash paths clear pending-token state, including ineligible names, no match, suppressed dispatch, successful dispatch, and the low-byte exception.

### Ordinary advancement and progress

Other classifier-zero cases invoke advance. A nonzero status ends scanning; zero clears pending-token state and continues.

No progress check is visible. Dependencies that leave the cursor unchanged on a continuing path can cause nontermination.

## Side effects

Direct writes comprise:

- Initial cursor, status, and stored-end initialization.
- Cursor movement past a slash.
- Cursor rewind before binary processing.
- Cursor restoration after the `FontDirectory` search.
- Dispatch-result assignment to status.
- Status clearing for the low-byte exception.
- Flag assignment to `1` after zero-result dispatch.

The global descriptor-table root is read, not directly written. Reads of it, input bytes, descriptor-name pointers, flags, and parser fields occur at the points specified above; they are not interchangeable with cached snapshots.

Dependencies may mutate any accessible state, including the global root or contents reached through a captured pointer. No purity, allocation, release, ownership, or preservation guarantees are supplied. There is no general rollback.

Scratch addresses refer to function-local storage. No durable ownership transfer or permission to retain those addresses after return is established.

## Dependencies

| Identity or interface | Established use | Unresolved contract |
|---|---|---|
| F-fca51b715561 | Fixed-length prefix comparison; zero selects a match | Full comparison semantics and side effects |
| F-cfad0582408b | Byte classification; nonzero selects pending-token handling | Accepted class and high-bit-byte interpretation |
| F-c6d4679527d4 | Processing after cursor rewind; zero selects literal return `3` | Format, output layouts, cursor changes, ownership |
| F-1cb585febd84 | Descriptor-name length result | String representation and mutations |
| F-7f14f1ab6851 | Length-limited name comparison; zero selects equality | Full comparison and mutation behavior |
| F-b6cdf3faef95 | Matched-record dispatch | Record schema, parsing, allocations, state changes |
| Unmapped comparison dependency | Ordered `Subrs` and `CharStrings` comparisons; zero selects equality | Opaque identity and implementation |
| Callback at offset 28 | Pre-inspection calls | Lexical behavior, status changes, progress |
| Callback at offset 32 | Advance calls | Consumption, errors, bounds, progress |
| Callback at offset 56 | Special operation after `known` | Output extent and connection to tested scratch byte |

No identities are invented for dependencies absent from the supplied mapping.

## Acceptance examples

These are conditional static consequences, not executed tests.

1. **Empty range:** If pre-inspection leaves cursor at the supplied end and status zero, it runs once and the function returns zero.

2. **Failed prefix comparison mutates input:** If a qualifying `FontDirectory` comparison returns nonzero after changing the retained input bytes to `eexec` plus another byte, the subsequent reread can select the `eexec` stop path.

3. **Classifier mutates input:** If the classifier returns zero after changing the retained address to a slash, slash handling uses that newly read byte, provided its length guard holds.

4. **Exact-end name:** Successful slash advancement ending exactly at the retained end still causes a global-root load, but no descriptor traversal.

5. **Ineligible name length:** A consumed length of 22 excludes traversal, but does not eliminate the preceding global-root read.

6. **Root changed during matching:** If a name-length dependency replaces the global root, the current traversal continues from its captured root and record positions. A subsequent lookup can observe the replacement.

7. **Captured name pointer:** If a dependency replaces the visited record’s name-pointer field, later comparisons for that record still receive the previously captured pointer. Mutations to bytes at that pointer remain observable.

8. **Zero flag:** A matched record with a zero flag dispatches without either special-name call.

9. **Nonzero flag:** `Subrs` is checked first. Equality bypasses `CharStrings`; inequality permits that second call. A flag mutation during either call is not reread to select dispatch.

10. **Successful dispatch:** A zero dispatch result is stored in status and then causes the selected flag byte to become exactly `1`.

11. **Low-byte exception:** Result `0x000001A2` is first stored in status, then status is cleared. This function makes no flag write on that path.

12. **Binary rejection:** A pending token followed by qualifying `RD` rewinds the cursor before invoking F-c6d4679527d4. Its zero result returns `3`, without a direct status assignment of `3`.

13. **Search exhaustion:** A `FontDirectory` search reaching the end without a match restores the fallback captured after its initial advance and pre-inspection calls. An earlier checked advance error instead exits without that restoration.

## Uncertainties

The following remain **documented_only**:

- Complete grammar, token boundaries, whitespace rules, and binary format.
- Valid wrapping-address behavior and the intended address-ordering model outside ordinary nonwrapping ranges.
- Signed versus unsigned promotion of high-bit input bytes into classification.
- Descriptor fields beyond the name pointer, table extent, and required flag-array extent.
- Global-root initialization and external mutation discipline.
- Offset-56 output extent and how, if at all, it establishes the separately tested scratch byte.
- The tested byte’s value when not established by a visible prior write. A binary-helper call in an earlier iteration receives that scratch region, but does not establish a general initialization guarantee.
- Semantic meanings of `3`, low-byte-`0xA2` results, and other statuses.
- Dependency ownership, allocation, release, parser preservation, and global effects.
- Bounds safety and progress guarantees.

These unknowns are not resolved by the inferred font-program or numeric-token interpretation.

## Evidence

The supplied static body establishes field offsets, 32-bit representations, initialization order, retained addresses, callback placement, range guards, input-byte rereads, global-root capture, descriptor stride, retained per-record name pointers, short-circuit dispatch conditions, writes, and return selection.

Dependency implementations and external format definitions were not supplied. Their names and apparent roles do not establish additional behavior.

No runtime tests have been performed. Acceptance examples describe conditional consequences of static control flow, not observed executions.
