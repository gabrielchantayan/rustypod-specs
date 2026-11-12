<!-- Trusted provenance: F-a8ecb9fc98cf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a8ecb9fc98cf

## Purpose

F-a8ecb9fc98cf processes a caller-supplied value slot according to descriptor flags, a one-byte kind, and optional callbacks. Observable behavior includes collection traversal, recursive processing of subordinate slots, callback invocation, opaque dependency calls, replacement or clearing of a slot, and early returns.

Cleanup or destruction is **inferred intent**, not an established ownership guarantee. The supplied representation does not establish whether any dependency frees memory, decrements references, preserves globals, or assumes ownership.

This contract separates:

- **Established static behavior:** visible accesses, widths, branch conditions, call arguments, ordering, and explicit writes.
- **Inferred intent:** possible lifecycle or resource-management roles.
- **documented_only unknowns:** behavior not established by the supplied representation.

All described completion sequences assume invoked dependencies return normally. No runtime tests have been performed.

## Interface

### Inputs and result

| Input position | Established use |
|---|---|
| 1 | Address of an active 32-bit value slot. Depending on the path, its contents are an opaque value, address, or collection handle. |
| 2 | Address of an outer descriptor containing 32-bit fields. |
| 3 | A 32-bit quantity with no explicit use in the visible body. |
| 4 | A 32-bit quantity used to initialize collection temporary storage and forwarded to F-b591e0f7a774 in one fallback case. |

There is no explicit result value. Outputs are memory effects and calls.

Recursive invocations visibly provide only the first two input positions. Their complete calling convention, including the availability of positions 3 and 4, is **documented_only**. Missing arguments must not be assumed to be zero or inherited.

### Representation

The visible model uses 32-bit addresses and words, an 8-bit kind, and signed 32-bit comparisons for loop bounds, entry counts, selection indexes, and specified dependency results. Byte order and required alignment are not established.

The outer descriptor has these relevant fields:

| Byte offset | Width | Meaning |
|---|---:|---|
| 0 | 32 bits | Flags: mask `0x00000006` selects collection processing; mask `0x00000400` controls finalization for kinds 1 and 2. |
| 16 | 32 bits | Type-record address, or an opaque collection-processing argument. |

The type record has these relevant fields:

| Byte offset | Width | Meaning |
|---|---:|---|
| 0 | 8 bits | Kind discriminator. |
| 4 | 32 bits | Signed fallback selector. |
| 8 | 32 bits | Nested outer-descriptor address for kind 0; entry-array base for kinds 1 and 2. |
| 12 | 32 bits | Signed entry count. |
| 16 | 32 bits | Optional callback-table address. |
| 20 | 32 bits | Replacement value for selector 1 with a nonnull type record. |

Entry-array address calculations use a 20-byte stride. This does not establish the meaning of every field within an entry.

The callback table contains 32-bit callable addresses at offsets 4, 8, 12, and 16. Its offset-0 field is not visibly used.

### Memory preconditions

The outer descriptor is read before any null-slot protection.

Collection processing dereferences the supplied slot without checking its address for null. The slot must support its initial read and eventual write.

Noncollection processing reads the type record’s offset-16 field before checking the slot address. A null slot therefore does not make an invalid outer descriptor or type record safe. A null type-record address is not a supported ordinary-entry precondition.

Any subsequently followed table, descriptor, entry, value header, or returned slot address must support the accesses actually taken. Callable addresses must be valid for their invocation. No explicit address-validation or error-reporting mechanism is visible.

## Behavior

### Collection processing takes precedence

A nonzero intersection between the outer flags and `0x00000006` selects collection processing, regardless of the kind or `0x00000400`.

The function reads and retains the original slot contents as the collection handle. It allocates temporary storage of six 32-bit words and explicitly initializes only its first word, initially from input position 4.

Iteration starts at signed index zero. F-10b760faca36 receives the retained handle before every loop-condition decision, including the terminating decision. An iteration occurs when the current signed index is less than that call’s signed result.

Each iteration obtains an item from F-6bbddce1b930 using the retained handle and index. Its 32-bit result replaces the temporary area’s first word. F-50507157c742 then receives the temporary-area address, a freshly read outer-descriptor field at offset 16, and zero. The index advances by one.

After iteration stops, F-9aef742d6dcd receives the retained handle. The original slot is then written with zero.

The bound is not captured once. Changes visible to F-10b760faca36 can alter termination. Changes to the original slot do not replace the retained handle. The trailing five temporary words are not explicitly initialized by this body, and no retention rights for the temporary address are established.

### Noncollection setup

When collection processing is not selected, the function rereads the outer flags to capture the `0x00000400` setting. It obtains the type-record address and reads that record’s callback-table address.

Only then does a null slot address cause a return.

The kind byte is read next. For every nonzero kind, a zero slot value causes an immediate return. Kind 0 does not perform this initial empty-value check.

For continuing execution, the function reads callback-table offset 16 when the table address is nonzero. That callable address, if nonzero, is captured for possible kind-1 or kind-2 use. This lookup occurs even for kinds that do not subsequently invoke that callback.

### Kind 0: descriptor redirection

The type-record field at offset 8 is interpreted as another outer-descriptor address.

A nonzero address restarts processing at the outer flag check with the same slot and the replacement descriptor. This can select collection processing. There is no visible cycle detection or redirection limit.

A zero address enters fallback processing using the current type record. Redirection itself performs no slot write or release-like call.

### Kind 1: reverse subordinate traversal

F-fa1cf018d20d first receives the slot address, `0xFFFFFFFF`, and the type-record address. A strictly positive signed result returns immediately.

Otherwise, the captured offset-16 callback, if present, receives event 2, the slot address, and the type-record address. A result exactly equal to 2 returns immediately. Other results continue.

F-4da75e9c77cb then receives the slot and type-record addresses. After it returns, the entry base and signed count are read to form an initial endpoint at base plus count multiplied by 20.

Traversal starts with iteration number zero. Before each condition decision, including the terminating decision, the current entry address is reduced by 20. The iteration number is compared against a newly read signed count at type offset 12.

For each admitted iteration:

- F-23eebec8340f receives the slot address, current entry address, and zero.
- A zero result skips subordinate processing for that entry.
- A nonzero result is passed with the slot address to F-50a87c4da87c.
- The slot address returned by F-50a87c4da87c is recursively processed by F-a8ecb9fc98cf, using that same nonzero resolver result as its outer descriptor.

With a stable positive count, entries are visited from index count minus one down to zero. The initial endpoint remains captured, while the count is reread for each decision. Count mutation can therefore change termination without adjusting that endpoint.

For a nonpositive count, no entry-processing dependency is called, but initial address formation and the first subtraction still occur. Normal traversal completion proceeds to shared finalization.

### Kind 2: selected subordinate traversal

The captured offset-16 callback, if present, first receives event 2, the slot address, and the type-record address. A result exactly equal to 2 returns immediately.

F-7eb30049adbd then receives the slot and type-record addresses. Its signed result is retained as the selection index.

If the callback exists, it is invoked again with event 2 and the slot address, without an explicit type-record argument. Its return value does not determine selection.

The subsequent descriptor reads are conditional:

- A negative retained index skips the type-offset-12 count read entirely and proceeds to shared finalization.
- A nonnegative index causes a signed count read after the optional second callback.
- An index equal to or greater than that count selects nothing.
- Only an index strictly below the signed count causes the entry base at type offset 8 to be read.

For a selected entry, its address is base plus index multiplied by 20. F-50a87c4da87c receives the slot and entry addresses. Its returned slot is recursively processed using that entry address as the outer descriptor.

Shared finalization follows whether or not an entry was selected.

### Shared finalization for kinds 1 and 2

The captured offset-16 callback, if present, receives event 3 and the slot address. Its result is ignored.

If the captured `0x00000400` setting is clear, the function reads the slot’s current value, passes it to F-5990a26eaaa3, and subsequently writes zero to the slot. The value passed is read after earlier callbacks and recursive processing, not retained from entry.

If that setting is set, neither this dependency call nor this final zero write occurs. Earlier dependency effects, recursive processing, and callbacks are not suppressed.

### Kinds 3 and 4

Kind 3 uses the previously obtained callback-table address to read offset 4. If both table and callable address are nonzero, the callback receives the current slot value.

Kind 4 instead reads table offset 8. A present callback receives the slot address and type-record address.

Either kind returns afterward without shared finalization or an explicit slot write. A missing table or zero callable address simply returns.

### Kind 5, unknown kinds, and fallback dispatch

Kind 5 enters fallback processing, subject to the earlier nonzero-kind empty-value check. Unrecognized nonzero kinds return without fallback or an explicit slot write.

With a nonnull type record, fallback rereads its callback-table field and, when present, table offset 12. A nonzero callback receives the slot address; the function then returns without further explicit writes.

Without that callback, a newly read kind of 5 forces selector `−1`. Otherwise, the signed selector is read from type offset 4.

**For a nonnull type record only**, selector 1 bypasses fallback’s ordinary empty-value check. It reads type offset 20, writes that 32-bit replacement to the active slot, and returns without a release-like call. This can replace an initially zero slot when reached through kind 0.

For other selectors, the active slot is read. A zero value returns. A nonzero value produces the following behavior:

| Selector | Observable sequence |
|---|---|
| `−4` | F-b591e0f7a774 receives the active slot address, zero, the previously read nonzero value, and input position 4. Afterward, the slot is reread and passed to F-5990a26eaaa3, then written with zero. |
| `5` | The slot is written with zero, without an intervening explicit dependency call. |
| `6` | F-d82460213e2d is called with no explicit arguments, then the slot is written with zero. |
| Any other value | The current slot value is passed to F-6c81b0b12feb. The slot then receives two consecutive explicit zero writes. |

The two default-case writes have no intervening explicit call. Their physical observability is not established by the supplied representation.

### Null-type-record fallback fragment

A distinct fallback fragment exists for a null type-record address. Its safe reachability through the ordinary entry is **documented_only**, because ordinary noncollection setup already dereferences the type record.

Within this fragment, the current slot value is treated as the address of a two-word region:

- The first 32-bit word supplies the signed selector.
- The active slot is redirected to the immediately following 32-bit payload word.
- That payload is read, and a zero payload causes an immediate return for **every selector, including selector 1**.

For a nonzero payload, selector 1 writes `0xFFFFFFFF` to the payload and returns. The remaining selectors follow the fallback actions above, targeting the payload.

The fragment does not explicitly clear the original slot holding the header address or overwrite the selector header. Aliasing could make a payload write affect other identified storage; no disjointness guarantee is established.

## Side effects

Explicit writes target collection temporary storage or the active slot. Recursive calls can target subordinate slots supplied by F-50a87c4da87c. In the null-type-record fragment, the active slot is the redirected payload, not necessarily the caller’s original slot.

No allocator pairing, ownership transfer, reference-counting rule, or release guarantee is established. Selector 1 overwrites without an explicit release-like call; selector 5 clears without one.

Dependencies and callbacks have unknown effects on reachable memory and globals. Relevant captured-versus-reread distinctions are:

- Collection processing retains its original handle but rereads the outer offset-16 argument each iteration.
- Noncollection processing captures its finalization flag and offset-16 callback.
- Kind 1 captures an initial endpoint but rereads the count for each traversal decision.
- Kind 2 retains its selection index. It reads count only for a nonnegative index, after the optional second callback, and reads base only for an in-range index.
- Shared finalization and selector `−4` read the slot after earlier calls.

There is no explicit rollback, exception handling, error status, or recovery from invalid memory access. Descriptor cycles, changing bounds, or recursive structures may prevent termination.

## Dependencies

| Opaque ID | Visible role |
|---|---|
| F-10b760faca36 | Supplies each signed collection bound from the retained handle. |
| F-6bbddce1b930 | Supplies a 32-bit item from a handle and index. |
| F-50507157c742 | Receives temporary item storage, outer offset-16 contents, and zero. |
| F-9aef742d6dcd | Runs after collection iteration and before slot clearing. |
| F-fa1cf018d20d | Kind-1 preliminary operation; a positive signed result stops processing. |
| F-4da75e9c77cb | Runs before kind-1 traversal setup. |
| F-23eebec8340f | Resolves a kind-1 entry; zero suppresses its recursion. |
| F-50a87c4da87c | Supplies a subordinate slot address. |
| F-7eb30049adbd | Supplies the signed kind-2 selection index. |
| F-5990a26eaaa3 | Release-like operation in shared finalization and selector `−4`. |
| F-b591e0f7a774 | Delegated selector-`−4` operation with four visible arguments. |
| F-6c81b0b12feb | Default fallback operation on the active value. |
| F-d82460213e2d | Selector-6 operation with no explicit arguments. |
| F-a8ecb9fc98cf | Recursive descriptor-directed processing. |

Runtime callbacks are identified by table offsets and invocation shapes, not by established implementation identities. Event values 2 and 3 have no proven semantic names.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Collection precedence:** With mask `0x6` set, bounds consistently equal to 2, and normally returning dependencies, indexes 0 and 1 are processed. The bound dependency is called three times. Collection finalization precedes clearing, even with `0x400` set.

2. **Null slot:** With collection flags clear and valid descriptor/type storage, a null slot returns after the type’s callback-table field is read, before kind dispatch.

3. **Initial empty value:** Kind 5 with a zero slot returns before fallback callback dispatch. Kind 0 is exempt from this initial check.

4. **Reverse traversal:** Kind 1 with stable count 3 and no early exit presents entry addresses base plus 40, base plus 20, and base to F-23eebec8340f. A zero resolver result skips only that entry’s subordinate processing.

5. **Negative selection:** Kind 2 with retained index `−1` still performs the optional second event-2 callback, but does not read the count or entry base for selection. Shared finalization follows.

6. **Selection boundary:** With count 3, index 2 reads the base and processes the third entry. Index 3 reads the count but not the base, processes no entry, and proceeds to finalization.

7. **Preservation flag:** Kinds 1 and 2 with captured `0x400` set omit the final F-5990a26eaaa3 call and clearing. This does not guarantee preservation of the entry value.

8. **Nonnull-type replacement:** Kind 0 with no nested descriptor, no fallback callback, selector 1, and an empty slot writes the type’s offset-20 replacement.

9. **Null-type fragment, empty payload:** Conditional on reaching that fragment safely, header selector 1 and payload zero cause a return without replacing the payload with `0xFFFFFFFF`.

10. **Null-type fragment, nonempty payload:** Under the same reachability condition, selector 1 with a nonzero payload replaces that payload with `0xFFFFFFFF`, without explicitly changing the header or original header-address slot.

11. **Default fallback:** A nonzero active value and an otherwise unmatched selector invoke F-6c81b0b12feb, followed by two explicit zero writes.

## Uncertainties

- **documented_only:** Ownership, allocation, release, global preservation, and retention behavior of every opaque dependency and callback.
- **documented_only:** Complete callback and recursive-call conventions, especially differing callback argument counts and missing recursive input positions.
- **documented_only:** Safe external reachability of the null-type-record fallback fragment. Its internal empty-payload behavior is nevertheless established.
- **documented_only:** Overflow semantics for signed increments, count multiplication, and 32-bit address arithmetic. Ordinary examples require representable calculations and accessible addresses; no wrapping or overflow-error contract is asserted.
- **documented_only:** Alignment, byte order, concurrency, aliasing restrictions, and physical observability of repeated stores.
- **Inferred intent:** Traversal and event callbacks resemble lifecycle processing, but their resource-management meanings are unproven.

## Evidence

This contract uses the supplied static function representation and opaque dependency mapping. Established claims concern visible widths, offsets, conditions, argument relationships, access ordering, and explicit writes.

The conditional count/base accesses in kind 2 and the null-type fragment’s payload check follow their respective branch conditions, not assumptions about intended cleanup behavior. Comments, prior prose, and ownership interpretations are not treated as behavioral authority.

No runtime tests have been performed.
