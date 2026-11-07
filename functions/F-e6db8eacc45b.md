<!-- Trusted provenance: F-e6db8eacc45b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e6db8eacc45b

## Purpose

F-e6db8eacc45b conditionally scans indexed entries using an indirect callback, adjusts a stored selection index, clears another state word, and copies one 32-bit array element to caller-supplied storage.

The scan compares the **signed 32-bit numeric representations of addresses**, not the contents at those addresses. The callback supplies the address-like value used in each comparison through a writable temporary slot.

An inferred intent is to select an entry relative to a boundary and optionally move the selection backward. That interpretation is not needed to define the behavior: the established contract is the signed comparison, index manipulation, and final array lookup described below.

## Interface

The function receives four arguments and returns a 32-bit status value.

| Argument | Established role |
|---|---|
| State address | Addresses mutable state, including a dispatch-table pointer, a count-source pointer, a readiness word, a selected index, a reset word, and an array-base address. |
| Comparison address | Supplies a 32-bit address representation used as a signed comparison operand. The function does not directly dereference this argument. |
| Mode | A 32-bit integer. Exactly `1` enables a conditional preliminary decrement. Every other value follows the same alternative path. |
| Output address | Receives one 32-bit array element on successful completion. Its address also initializes the temporary slot passed to the callback. |

The state layout visible to this function is:

| Byte offset | Width | Use |
|---|---:|---|
| `0x000` | 32 bits | Address of a dispatch table. |
| `0x040` | 32 bits | Address of storage whose first 32-bit word is the signed entry count. |
| `0x21C` | 32 bits | Mutable signed selection index. |
| `0x220` | 32 bits | State word set to zero on the completion path. |
| `0x228` | 32 bits | Readiness word; zero prevents processing. |
| `0x22C` | 32 bits | Base address of an array of 32-bit elements. |

The indirect callback address is obtained from byte offset `0x40` in the dispatch table. That offset belongs to the dispatch table, not to the state field at the same numeric offset.

For ordinary non-aliased storage, the state layout extends through byte offset `0x22F`. The function additionally requires accessible count storage, a usable dispatch table and callback, and a readable final array element whenever execution reaches those operations.

There is no supplied array length, output capacity, or explicit validation of the final index.

Return values established by the supplied material are:

- `3`: the entry gate rejects processing.
- `0`: the completion path performs its output store and returns.

These values do not establish named error categories. No additional returned error status is visible.

## Behavior

### Entry gate

The function first examines the count-source address stored at state offset `0x040`.

If that address is nonzero, it reads the readiness word at `0x228`. Otherwise, its provisional readiness value remains zero.

It then checks the count-source field again and returns `3` if either:

- the count-source address is zero at that check; or
- the sampled readiness value is zero.

A nonzero readiness value is sufficient, including one whose signed interpretation is negative.

On this rejection path, the function performs no direct state writes, no output write, and no callback invocation. The count itself is not read.

The repeated access to the count-source field is part of the supplied expression structure. Atomicity, synchronization, and exact machine-load behavior under concurrent modification are not established.

### Temporary callback state

After the gate passes:

- The scan index starts at zero.
- A local, writable 32-bit address slot is initialized with the output address.

This slot persists across callback invocations. It is **not** reinitialized before each invocation.

Each callback receives:

1. the state address;
2. a 32-bit index value;
3. the address of this temporary slot.

The callback may replace the slot’s contents. If it leaves the slot unchanged, the previously held value remains the comparison value. In particular, the first callback sees a slot initially containing the output address.

The callback’s return value is not used.

### Scan

At the start of each scan iteration, the function reads the signed count through the current count-source address.

If the scan index is greater than or equal to that count, scanning ends immediately and execution proceeds to completion processing. This includes an initially zero or negative count: no scan callback is made in that case.

Otherwise, the function invokes the callback with the current scan index.

After the callback returns, it compares:

- the signed 32-bit representation of the comparison address; and
- the signed 32-bit representation currently stored in the temporary slot.

If the comparison address is less than or equal to the temporary value, the function stores the scan index at state offset `0x21C` and ends scanning.

If the comparison address is greater than the temporary value:

- the scan index advances by one;
- the count is read again through the current count-source address;
- the function stores that newly read count minus one at `0x21C`;
- the next iteration begins.

Consequently, a failed comparison does not store the current scan index. It stores a last-entry index derived from a fresh count read.

The count and its source address are not cached across the scan. A callback that changes them can affect both the post-callback selection write and the next loop bound. A callback that makes the count source invalid can make the next dereference invalid.

### Completion processing

Completion processing occurs both after a successful scan comparison and after a loop-bound termination.

First, if the mode is exactly `1` and the current selected index is positive, the function decreases that selected index by one. A zero or negative index is unchanged by this step.

The function then invokes the callback once more, using the selected index present at that point and the same persistent temporary slot.

This final callback is made even when the initial count was zero or negative, provided the entry gate passed. For such a count, the callback can receive a selected index inherited from earlier state rather than one established by a scan comparison.

After the callback returns, the function conditionally performs another decrement. The supplied sign-and-overflow expression has the following effective behavior over 32-bit signed values:

- If the temporary address value is strictly greater than the comparison address, using signed comparison, and the current selected index is positive, decrease the selected index by one.
- Otherwise, leave the selected index unchanged.

Equality does not cause this decrement. A temporary value below the comparison address does not cause it, including when subtracting their representations would overflow signed 32-bit arithmetic.

The selected index used for this decision is read after the final callback. It need not equal the index passed into that callback if the callback modifies state.

The preliminary mode-dependent decrement and the final comparison-dependent decrement are distinct. Both can occur in one invocation.

### Final state and output

After index adjustment, the function:

1. writes zero to the state word at `0x220`;
2. obtains the array base and selected index from state;
3. reads the 32-bit element addressed by the array base plus four times the selected index;
4. writes that element to the original output address;
5. returns `0`.

The output address itself is not replaced by callback changes to the temporary slot.

No final index-range check is present. Negative indices, an index beyond the available array, or an invalid array base can therefore result in an invalid memory access rather than a returned error.

### Widths and address comparisons

The visible state words, indices, element widths, pointer representations, and return value are 32 bits.

The comparison order is signed 32-bit numeric order, not unsigned address order. For example, an address representation with its top bit set compares as negative and therefore below a representation with its top bit clear.

The subtraction-related overflow test in the final adjustment prevents that particular decision from being treated as a naive signed subtraction comparison. It does not establish general overflow safety elsewhere.

Count-minus-one, scan-index advancement, and array-offset arithmetic have no visible overflow guards. Valid callers should keep those operations within the intended representable and accessible ranges. The supplied material alone does not establish a portable language-level contract for overflowing those operations.

## Side effects

Direct effects on the normal completion path are:

- Zero or more scan-related writes to the selected index.
- An optional mode-dependent decrement.
- One final callback invocation.
- An optional post-callback decrement.
- A zero write to the state word at `0x220`.
- A 32-bit write to the output address.

Callback side effects are additional and unspecified. The callback receives the state address and can potentially modify state, dispatch information, count storage, array storage, or other reachable memory. No preservation guarantee is supplied.

No direct allocation, deallocation, ownership transfer, or explicit resource release appears in this function. This does not establish that the callback allocates nothing or releases nothing.

The temporary slot is local to this invocation. Its address is exposed to the callback during calls, but no lasting lifetime or retention permission is established.

Aliasing materially affects the result:

- If output storage overlaps the reset word, the final output store can overwrite the zero just written.
- If the selected array element overlaps the reset word, the array read observes the preceding zero write, absent intervening external modification.
- If output storage overlaps the selected-index field, the final output store replaces that field after the array element has been selected.
- Changes to the temporary slot do not redirect the output store.

No transactional guarantee is present. If an invalid access or callback failure interrupts execution, earlier state writes are not visibly rolled back.

## Dependencies

### F-indirect-01 — unresolved callback

This is an opaque specification identifier for the unresolved indirect dependency, not an identification of its implementation.

Its dispatch address comes from offset `0x40` in the table addressed by the first state word. The same dispatch mechanism is used for scan callbacks and the final callback.

Observed argument roles are:

- state address;
- 32-bit index;
- address of a writable 32-bit temporary slot.

The caller consumes the temporary slot and state after return. No useful callback return value is established.

The following remain unspecified:

- the meaning of the produced address-like value;
- whether every invocation writes the slot;
- whether repeated calls for one index produce identical values;
- which state fields or globals are preserved;
- allocation, ownership, and release behavior;
- exception, fault, or non-returning behavior;
- whether the callback retains any supplied address.

The dispatch-table pointer is obtained for the individual invocations shown by the supplied material. No stable callback identity across state mutation should be assumed.

## Acceptance examples

These are conditional static examples, not runtime test results. Unless stated otherwise, assume valid accessible storage, nonzero readiness, stable counts, no relevant aliasing, and callbacks that do not modify state except for the temporary slot.

### Rejected entry

If the count-source address is zero, the result is `3`.

There is no callback invocation, selected-index update, reset-word write, or output write.

The same direct effects apply when the count-source address is nonzero but readiness is zero.

### Exact boundary match

Suppose the count is three, mode is not `1`, and the comparison address has signed numeric value `20`.

Suppose scan callbacks produce `10` for index zero and `20` for index one. The scan selects index one. If the final callback also produces `20`, the final adjustment does not decrement.

The reset word becomes zero, the output receives array element one, and the function returns `0`.

### Strictly greater boundary moves backward

Suppose the count is three, mode is not `1`, and the comparison value is `15`.

If scan callbacks produce `10` and then `20`, scanning selects index one. If the final callback produces `20` again, the selected index decreases to zero because it is positive and the final temporary value exceeds the comparison value.

The output receives array element zero.

### No scan match

Suppose the count is three, mode is not `1`, the comparison value is `40`, and the scan callbacks produce `10`, `20`, and `30`.

Each scan comparison fails. Each failure stores count minus one, so the selected index is two when the loop terminates.

If the final callback produces `30`, no final decrement occurs. The output receives array element two.

### Nonpositive count

Suppose the count is zero and the preexisting selected index is two.

No scan callback occurs. With mode other than `1`, the final callback receives index two. If it produces a value no greater than the comparison value and leaves state unchanged, the output receives array element two.

This is not an empty-array success guarantee: element two must still be readable.

### Two decrements

Suppose scanning selects index three and mode is `1`.

The preliminary adjustment changes the index to two. If the final callback leaves that index unchanged but produces a temporary value strictly greater than the comparison value, the final adjustment changes it to one.

The output receives array element one.

### Signed address ordering

Suppose the comparison representation is `0x80000000` and the callback supplies `0x00000000`.

The comparison value is negative in signed 32-bit interpretation, so it is less than the callback value. The scan comparison succeeds; treating these representations as unsigned addresses would give the wrong result.

## Uncertainties

**documented_only — Semantic names.** The meanings of readiness, selected entries, the reset word, and the array elements are unknown. Their names here describe observed use, not domain semantics.

**documented_only — Callback contract.** No implementation or independent contract for F-indirect-01 is supplied. State preservation, ownership, and stable callback output cannot be assumed.

**documented_only — Valid index invariant.** Nothing establishes that the stored selection index is valid before entry, particularly when the count is nonpositive.

**documented_only — Overflow and invalid access.** No explicit handling is supplied for arithmetic overflow, null output storage, inaccessible count storage, invalid dispatch data, or an invalid final array address.

**documented_only — Concurrency.** No synchronization, atomic-access guarantee, or concurrent-mutation contract is visible.

**Inferred intent.** The scan resembles boundary-based selection with backward adjustment. Sorting, monotonic callback results, and a predecessor-selection guarantee are not established.

## Evidence

This contract is derived solely from the supplied static representation of F-e6db8eacc45b.

Established evidence includes the field offsets and 32-bit accesses, entry status values, signed address comparisons, persistent temporary slot, indirect callback arguments, index writes, reset-before-output ordering, and final array-element copy.

The simplified final-decrement rule follows from analyzing the supplied equality, sign, and signed-subtraction-overflow conditions together. It does not depend on assuming ordinary small address values.

No runtime tests have been performed. The acceptance examples are conditional consequences of the static behavior, not observed executions.
