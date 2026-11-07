<!-- Trusted provenance: F-d2fa2a3e391c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d2fa2a3e391c

## Purpose

F-d2fa2a3e391c checks a selected entry in an object-relative table, requires two particular observations of that entry around a dependency call, explicitly clears the entry, and then performs an embedded-state completion operation.

To reach the explicit entry-clearing stores, the selected entry’s first word must be observed as nonzero before F-c243fa3f5264 and zero after it. **The supplied representation does not establish what causes the change between those observations.** In particular, it does not establish that F-c243fa3f5264 changes the word.

After clearing the entry, the function can:

- Return a nonzero result from a tag-selected dependency.
- Return 5 following an identity mismatch.
- Decrement a 16-bit field and return zero without final completion.
- Clear embedded fields, invoke a final dependency, and return its result.
- Return 26 if a previously formed embedded-state address is null.

**Inferred intent:** The identity field, flags, and decrementing field resemble ownership and recursive-completion state. The table operation resembles relinquishing an occupied entry. These interpretations are not established API semantics.

## Interface

### Explicit inputs

| Input | Representation | Established use |
|---|---|---|
| Object base | 32-bit integer used to form addresses | Locates the selected table entry and embedded state. |
| Entry index | Unsigned 32-bit integer | Selects one of 29 entries, indexed 0 through 28. |
| Auxiliary value A | Opaque 32-bit value | Forwarded to F-621dae46a146 when that dependency is selected. |
| Auxiliary value B | Opaque 32-bit value | Forwarded to F-621dae46a146 when that dependency is selected. |

The representation uses 32-bit address-like arithmetic. It does not independently establish language-level overflow rules, pointer provenance, or the validity of wrapping addresses. A caller must supply addresses valid for every access actually performed.

F-621dae46a146 also receives four 32-bit values represented as ambient machine state rather than explicit inputs to F-d2fa2a3e391c. Their meanings, validity requirements, and relationship to the entry state are unknown. The four explicit inputs therefore do not establish a complete description of the function’s incoming state.

### Object-relative layout

All offsets below are in bytes from the object base.

| Offset | Access width | Established role |
|---|---:|---|
| `0x114` | 32 bits for the direct read | Beginning of embedded state; its first word is compared with a distinguished external value. Its address is also passed to dependencies. |
| `0x118` | 32 bits | Field compared with the result of F-196812b4b122 and conditionally cleared. |
| `0x122` | 16 bits, read unsigned | Flags snapshot used for a later branch. |
| `0x126` | 16 bits | Conditionally decremented field; also explicitly cleared on final completion. |
| `0x128` | Address passed onward | Embedded region passed to F-259d2fe281b0; its extent and contents are unknown. |
| `0x154 + 8 × index` | 32 bits | Selected entry’s first word; checked twice, then explicitly cleared on the continuing path. |
| `0x158 + 8 × index` | 32 bits | Selected entry’s second word; explicitly cleared without a local read. |

The visible table has an eight-byte stride. For index 28, the first word starts at `0x234`, and the second starts at `0x238` and extends through `0x23b`.

These accesses establish a required address range on applicable paths, not a complete object definition. Dependencies may access additional memory.

### Outputs and exceptional exits

The declared result is a 32-bit integer. Ordinary returning paths produce:

- A nonzero result from F-621dae46a146, unchanged.
- A result from F-259d2fe281b0, unchanged.
- Zero.
- The fixed value 5.
- The fixed value 26.

F-a817d8d58c10 and F-feee9156a257 are annotated as non-returning in the supplied representation. Paths reaching those calls have no specified ordinary result. Their termination or control-transfer mechanisms are unknown.

### Preconditions

All memory accessed must be valid for the relevant width, and all dependencies’ applicable preconditions must hold.

The function does not establish these conditions before its initial dependency call. An invalid index does not prevent that call, and the late embedded-address check is not general null-object protection.

The supplied material establishes no allocation size, alignment guarantee, object lifetime rule, concurrency discipline, or resource-ownership convention.

## Behavior

### 1. Initial embedded-state call

The function forms the address at object base plus `0x114` and passes it to an initial dependency whose opaque ID is unavailable.

This call occurs before index validation and before either local read of the selected entry.

The dependency’s effects are unknown. Neither object fields nor global state may be assumed unchanged when it returns. Consequently, later entry validation describes memory observed after this call, not necessarily memory present at function entry.

### 2. Initial entry validation

After the initial dependency returns, the function checks whether the unsigned index exceeds 28.

If it does, the function invokes F-a817d8d58c10. The selected entry is not locally read on this path.

For an index from 0 through 28, the function reads the selected entry’s first 32-bit word:

- If zero, it invokes F-a817d8d58c10.
- If nonzero, it continues to F-c243fa3f5264.

The second entry word does not participate in validation.

Unsigned interpretation matters: an index with bit pattern `0xffffffff` is out of range, not a negative index accepted by this check. For accepted indices, the index-derived displacement is at most 224 bytes; uncertainty about address arithmetic concerns its combination with the supplied base, not a large accepted index.

### 3. Second observation of the selected word

After successful initial validation, the function invokes F-c243fa3f5264 with no explicit arguments. It then rereads the same selected first word.

If this second observation is nonzero:

1. F-3feab73aed8b is invoked with no explicit arguments.
2. If that call returns, F-feee9156a257 is invoked.

The function’s subsequent explicit entry-clearing stores are not reached on this path.

If the second observation is zero, execution continues to entry clearing.

Thus, continuation requires a nonzero observation before F-c243fa3f5264 and a zero observation after it. This is an observational requirement, not an attribution of responsibility. The dependency, activity it triggers, or an external actor could account for the differing observations; the supplied material does not identify the cause or establish a synchronization protocol.

### 4. Explicit clearing of the selected entry

The function performs two 32-bit stores in this order:

1. Zero to the selected entry’s first word.
2. Zero to the selected entry’s second word.

The first store is explicit even though the preceding read observed zero. The second word is overwritten without a local inspection of its previous value.

These stores precede all completion-path selection, fallback identity checking, and count updates. Later errors or nonzero return values do not undo them. Dependency effects may subsequently change the same memory.

### 5. Late embedded-address condition

After entry clearing, the function tests whether the previously formed address at object base plus `0x114` is null.

If null, it returns 26. It does not perform the tag comparison or fallback completion work.

This condition tests the derived address, not the object base. Under a wrapping 32-bit address interpretation, base bit pattern `0xfffffeec` would yield a null derived address. Whether that arithmetic and the earlier operations are valid is unresolved.

The initial dependency has already received the derived address, and the selected-entry accesses have already occurred. This branch therefore does not establish safe handling of null or otherwise invalid objects.

### 6. Tag-selected completion attempt

For a non-null embedded-state address, the function compares the 32-bit word at offset `0x114` with a distinguished external value.

If the values differ, it skips F-621dae46a146 and proceeds to fallback.

If they match, it calls F-621dae46a146 with eight arguments, in these positions:

1. The embedded-state address at offset `0x114`.
2. Zero.
3. Auxiliary value A.
4. Auxiliary value B.
5. The first ambient 32-bit value.
6. The second ambient 32-bit value.
7. The third ambient 32-bit value.
8. The fourth ambient 32-bit value.

A nonzero result is returned immediately and unchanged. The function then performs no fallback flags read, identity comparison, count update, or F-259d2fe281b0 call.

A zero result selects fallback. No object or global-state preservation may be assumed across this call. Fallback uses the values subsequently observed.

The source does not establish a relative evaluation order between the two operands of the tag comparison.

### 7. Fallback snapshot and identity check

Fallback first reads the unsigned 16-bit flags field at offset `0x122` and retains that snapshot.

It then invokes F-196812b4b122 with no explicit arguments. After the call returns, it reads the 32-bit field at offset `0x118` and compares that field with the dependency’s result.

This ordering creates two distinct observation points:

- Flags are captured before F-196812b4b122.
- The identity-like field is read after F-196812b4b122.

Changes to flags during the call do not replace the saved flags used by the local branch. Changes to the identity-like field can affect the later comparison.

If the comparison fails, the function returns 5. It performs no subsequent explicit count or identity-field stores and does not invoke F-259d2fe281b0. The entry-clearing stores and prior dependency effects have already occurred.

### 8. Matching-identity mode selection

When the comparison succeeds, the saved flags select between two completion behaviors.

The special mode applies exactly when bit 5 is set and bit 4 is clear: the saved flags masked by `0x0030` equal `0x0020`.

Other flag bits do not affect this local decision. In particular, lower bits retained by an intermediate mask do not survive the subsequent shift into the comparison.

#### Special mode

The function reads the 16-bit field at offset `0x126` using a signed interpretation, subtracts one, narrows the result to 16 bits, stores it back, and tests that narrowed result for zero.

If the result is nonzero, the function returns zero without explicitly clearing the identity-like field and without invoking F-259d2fe281b0.

If the result is zero, it proceeds to final completion.

This is not a positive-count check or a saturating decrement. Under low-16-bit narrowing:

| Before decrement | Stored result | Next action |
|---|---|---|
| `0x0001` | `0x0000` | Final completion |
| `0x0002` | `0x0001` | Return zero without final completion |
| `0x0000` | `0xffff` | Return zero without final completion |
| `0x8000` | `0x7fff` | Return zero without final completion |

The signed 16-bit read produces a value from −32768 through 32767. Subtracting one fits within a signed 32-bit intermediate. The `0x8000` case nevertheless requires narrowing −32769 back to 16 bits; its exact target-level semantics are not independently verified.

#### Other modes

For every other saved-flags combination, the function proceeds directly to final completion. It does not first read or decrement the count field.

### 9. Final completion

The function performs these operations in order:

1. Stores 16-bit zero at offset `0x126`.
2. Stores 32-bit zero at offset `0x118`.
3. Invokes F-259d2fe281b0 with the address at offset `0x128`.
4. Returns that dependency’s result unchanged.

In special mode, a decrement reaching zero is therefore followed by another explicit zero store to the same 16-bit field.

The dependency may alter either cleared field again. The local stores are established; their persistence after the call is not.

## Side effects

The function’s explicit memory writes are limited to:

- Two selected-entry 32-bit zero stores.
- A conditional 16-bit decrement store.
- A conditional 16-bit zero store to the same field.
- A conditional 32-bit zero store to the identity-like field.

Dependencies may have additional effects, including changes to object memory, globals, resources, and control flow. None is established to preserve globals or own, release, or retain memory.

There is no explicit local allocation or deallocation. Overwriting an entry word does not establish disposal of anything it might reference.

The described order is the order of visible operations in the supplied representation. It does not establish atomicity, barriers, safe concurrent access, or a locking protocol.

The operation is not established to be transactional. Return 5, return 26, and a nonzero tag-selected result all occur after explicit entry clearing on their respective paths.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| Unavailable — documented_only | Initial dependency receiving the address at offset `0x114`, before validation. |
| F-a817d8d58c10 | Called for an out-of-range index or an initially observed zero entry word; annotated non-returning. |
| F-c243fa3f5264 | Called between the two selected-word observations; its responsibility for any state change is unknown. |
| F-3feab73aed8b | Called when the second selected-word observation is nonzero. |
| F-feee9156a257 | Called if F-3feab73aed8b returns on that path; annotated non-returning. |
| F-621dae46a146 | Tag-selected attempt; a nonzero result bypasses fallback. |
| F-196812b4b122 | Supplies the value compared with the subsequently read identity-like field. |
| F-259d2fe281b0 | Receives the address at offset `0x128` after explicit field clearing; supplies the final result. |

Names such as “lock,” “unlock,” “thread identifier,” “free,” or “exception” are not established dependency semantics.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless stated otherwise, assume required addresses are valid and preceding dependencies return.

1. **Unsigned rejection:** Index 29 or `0xffffffff` causes F-a817d8d58c10 to be called after the initial dependency, without a local selected-entry read.

2. **Initial zero:** For index 0, a first word observed as zero after the initial dependency causes F-a817d8d58c10 to be called. F-c243fa3f5264 is not reached.

3. **Second observation remains nonzero:** For index 28, nonzero observations on both sides of F-c243fa3f5264 lead to F-3feab73aed8b and, if it returns, F-feee9156a257. No explicit entry-clearing stores follow.

4. **Required observations without assigned cause:** If the first observation is nonzero and the second is zero, the function clears both entry words. This expectation does not require assuming that F-c243fa3f5264 caused the difference.

5. **Tag-selected result:** After entry clearing, a matching tag and result 9 from F-621dae46a146 cause return 9 without fallback.

6. **Identity mismatch:** If fallback’s post-call field value differs from the result of F-196812b4b122, the function returns 5 without a subsequent local count update or final completion call.

7. **Special-mode retention:** With matching identity, saved flags `0x0020`, and count 2, the function stores count 1 and returns zero without explicitly clearing identity or calling F-259d2fe281b0.

8. **Special-mode final completion:** With the same conditions and count 1, it stores zero from the decrement, stores zero again during final completion, clears identity, calls F-259d2fe281b0, and returns its result.

9. **Zero-count boundary:** Under low-16-bit narrowing, special mode with count zero stores `0xffff` and returns zero without final completion.

10. **Flags changed during identity acquisition:** If saved flags select special mode but F-196812b4b122 changes the in-memory flags, the branch still uses the saved value. Conversely, changing the identity-like field during that call can affect the comparison.

11. **Other flag modes:** With matching identity and saved flags `0x0030`, final completion occurs without first reading or decrementing the count.

12. **Late null address:** If execution validly reaches entry clearing with a null derived embedded-state address, the function returns 26 afterward. This does not establish that such execution is reachable under valid caller conditions.

## Uncertainties

The following remain **documented_only**:

- The initial dependency’s opaque ID and behavior.
- The cause of the differing selected-word observations on a successful path.
- Dependency preconditions, memory ownership, global-state effects, synchronization effects, and failure mechanisms.
- The distinguished external value’s meaning and stability.
- The meanings and validity requirements of the four ambient arguments.
- Semantic interpretations of results 5 and 26.
- The control-transfer mechanisms of dependencies annotated non-returning.
- Exact address-overflow, pointer-validity, and out-of-range signed-narrowing semantics.
- Object lifetime, alignment requirements, complete layout, endianness, and concurrency rules.
- Reachability of the late null-address branch under valid conditions.
- Whether the inferred ownership and recursive-completion interpretation matches the intended API.

## Evidence

This contract derives from the supplied static function representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent behavioral evidence.

Established static evidence includes access widths and offsets, unsigned index bounds, the two selected-word checks, dependency-call placement, explicit store order, flags snapshot timing, conditional decrement behavior, argument positions, and return selection. Non-returning behavior rests on supplied annotations rather than observed execution.

The representation establishes required observations around F-c243fa3f5264, not causation by that dependency. Ownership terminology and completion intent remain inferences.

No runtime tests have been performed.
