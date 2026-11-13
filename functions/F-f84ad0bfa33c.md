<!-- Trusted provenance: F-f84ad0bfa33c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f84ad0bfa33c

## Purpose

F-f84ad0bfa33c advances mutable, table-driven cursor state toward a computed target. It can reset the cursor, request table-window refreshes, and apply optional interval restrictions.

Two distinct 64-bit positions participate:

- The **target-facing position** controls whether the computed target has been reached.
- The **traversal position** advances independently and is bounded by a separate limit.

Optional intervals can suppress an item’s contribution to the target-facing position without suppressing its traversal increment, counter changes, or accumulated item value.

These roles are established by the visible operations. Interpreting the routine as time conversion followed by indexed traversal is inferred intent, not an established domain contract.

## Interface

### Arguments and result

Four 32-bit argument words are visible:

1. An address of a caller-supplied context.
2. A requested value.
3. An opaque auxiliary value.
4. Another opaque auxiliary value.

The context has a 32-bit state pointer at byte offset `+0x1B8`. Ownership of the context, state, descriptors, and backing tables is not established.

The requested value is used both as an unchanged 32-bit pattern and, unless a mode selects zero, as a signed 32-bit multiplicand. The auxiliary words are forwarded only to the first arithmetic dependency invocation.

The result is a 64-bit pattern assembled from two 32-bit words:

| Exit condition | Low word | High word |
|---|---:|---:|
| Null state or joint inactive sentinel | `0` | Original requested word |
| Normal stopping condition | `0` | Computed target’s low word |
| Checked dependency returns nonzero | `3` | Computed target’s low word |
| Interval progression limit reached | `3` | Computed target’s low word |

The result does not return the complete computed target or either complete stored position.

### Storage layout

Offsets are in bytes. Fields and pointers occupy 32 bits. Each position pair identifies its low word first and high word second; this does not independently establish byte endianness.

| State offset | Role |
|---|---|
| `+0x0C` | Value-table descriptor pointer |
| `+0x14` | Range-table descriptor pointer |
| `+0x18` | Duration-table descriptor pointer |
| `+0x1C` | Optional interval descriptor pointer |
| `+0x48`, `+0x4C` | Joint inactive sentinel |
| `+0x68` | Persisted next-item index |
| `+0x6C` | Range-table index |
| `+0x70` | Range progression counter |
| `+0x74` | Within-range counter |
| `+0x78` | Accumulated item value |
| `+0x7C` | Signed conversion factor |
| `+0x84` | Within-duration-run counter |
| `+0x88`, `+0x8C` | Target-facing position |
| `+0x90`, `+0x94` | Traversal position |
| `+0x98`, `+0x9C` | Traversal limit |
| `+0xA0` | Duration-table index |
| `+0xA4` | Interval progression counter |
| `+0xA8`, `+0xAC` | Interval start or joint all-ones sentinel |
| `+0xB0`, `+0xB4` | Interval end |
| `+0xB8`, `+0xBC` | Sentinel-interval position adjustment |

The value-table descriptor contains:

| Offset | Meaning |
|---|---|
| `+0x00` | Constant item value; zero selects indexed values |
| `+0x04` | Mode; exactly one selects an effective request of zero |
| `+0x08` | Window base |
| `+0x0C` | Pointer to four-byte item values |

The range-table descriptor contains a count at `+0x00`, a window base at `+0x04`, and a record pointer at `+0x08`. Records occupy 12 bytes. Relative to a selected record:

- The signed word at `+4` is its within-range threshold.
- The signed word at `+12` is the boundary used to consider advancing to the next record. It is also the first word of that next record.

The duration-table descriptor has a record pointer at `+0x08`. Its eight-byte records contain a signed run threshold at `+0` and a signed duration increment at `+4`. Duration records are indexed directly; no duration-window base is visible.

The optional interval descriptor’s first word is a signed progression limit.

### Preconditions

Every accessed field, selected table record, and dependency-required object must be accessible. Except for the initial state-pointer check and optional interval-pointer checks, the routine does not establish pointer validity or sufficient table storage.

No ownership transfer, allocation obligation, release obligation, concurrency guarantee, or reentrancy guarantee is established.

## Behavior

### Eligibility and retained conversion inputs

The context’s state pointer is read first. A null pointer returns immediately. Otherwise, the inactive high word at `+0x4C` is checked; the low word at `+0x48` is checked only if the high word is all ones. Both must be all ones for the inactive return.

These exits contain no explicit state writes or dependency calls.

For an active state, the state pointer is loaded again. The routine reads and retains the signed conversion factor, then the high and low words of the target-facing position.

It constructs the low 64 bits of that position multiplied by 1000. The low-word multiplication contributes both a low word and carry into the high word; the original high word multiplied by 1000 contributes to the resulting high word.

**Before either arithmetic dependency invocation**, the routine reads the value-table mode and selects the effective requested value:

- Zero when the mode equals one.
- Otherwise, the original requested word.

This selection is retained across the first invocation. The mode is not reread afterward to choose the multiplicand.

The first invocation of F-1e0edc6e0bc9 receives seven visible words:

- The scaled position’s low and high words.
- The retained conversion factor and its sign-extension word.
- The original requested word, not the effective request.
- Both auxiliary words.

Only a 32-bit zero/nonzero result is subsequently used from this invocation.

The retained effective request and retained conversion factor are then interpreted as signed 32-bit values and multiplied to produce a signed 64-bit product. The second invocation receives that product’s two words followed by `1000` and `0`. Its 64-bit result is the target.

Mutations by the first invocation do not change the retained effective request, conversion factor, or original-position snapshot used by these calculations and the reset decision. They can affect later memory accesses.

The dependency’s exact numerical operation, rounding, and exceptional behavior are unknown.

### Conditional reset

The retained original target-facing position and computed target are compared as unsigned 64-bit quantities. Reset is selected when the original position exceeds the target, or when the first arithmetic invocation returned zero.

Reset writes occur in this order:

1. Persisted next-item index becomes one.
2. Range-table index becomes zero.
3. Range progression becomes one.
4. Within-range counter becomes zero.
5. Accumulated item value becomes zero.
6. Duration-table index becomes zero.
7. Interval progression becomes zero.

The context’s state pointer is reloaded for the individual writes.

The routine then checks the current value-table window base. If nonzero, it invokes F-5e3dde636ee5 with the context and zero. A nonzero result returns status three immediately.

Next, it checks the current range-table window base. If nonzero, it invokes F-cef02eaf8f8c with the context and zero. A nonzero result likewise returns status three.

If the current optional interval descriptor pointer is nonnull, F-a71330230cfd is invoked with the context; its result is not checked.

Only after these operations complete normally does the routine clear, in order:

- Target-facing low word, then high word.
- Traversal low word, then high word.
- Within-duration-run counter.

The state pointer is retained within each position-pair clear and reloaded between those pairs. A checked reset failure preserves preceding writes and skips the final clears. There is no rollback.

### Stopping conditions and index persistence

After reset processing, or immediately when reset is unnecessary, the persisted next-item index is loaded into a private working index.

At every traversal boundary, stopping checks occur in this order:

1. Target-facing position is unsigned-greater-than-or-equal to the target.
2. If not, traversal position is unsigned-greater-than-or-equal to its limit.

Either condition produces normal completion. Before that return, the working index is written to `+0x68`.

A target can be overshot. The traversal limit can also stop processing before the target is reached. The limit is checked between items, not by clipping an item’s increment.

### Value-window coverage and selection

Before consuming an item, the routine checks coverage of the working index by the value-table window.

For ordinary non-overflowing values, coverage means that the index is strictly greater than the base and at most the base plus 32768.

At width boundaries, the precise condition matters: coverage requires both of the following signed 32-bit comparisons to hold:

- Working index is at most the 32-bit result of base plus 32768.
- Base is at most the 32-bit result of working index minus one.

These are comparisons of the resulting signed words, not an unbounded-integer interval calculation.

If coverage fails, F-5e3dde636ee5 receives the context and the 32-bit working index minus one. A nonzero result returns status three.

After a successful refresh, the routine reloads the state and descriptor. It uses the descriptor’s first word when nonzero. Otherwise, it reads the four-byte entry at relative index “working index minus window base minus one.”

The selected item word is retained across later dependency calls in this iteration.

### Range-window coverage and interval eligibility

The range index is covered when it is signed-greater-than-or-equal to the range window base and signed-less-than the 32-bit result of base plus 32767.

Failure invokes F-cef02eaf8f8c. This traversal call site visibly supplies only the context. A nonzero result returns status three.

The private working item index is then incremented as a 32-bit value. The persisted index is not updated at this point.

Without an optional interval descriptor, the item is eligible to advance the target-facing position.

With an optional descriptor, a joint all-ones interval start triggers repeated sentinel processing. The start’s high word is checked before its low word. Each repetition:

- Adds the 64-bit adjustment to the target-facing position, writing low word before high word and including low-word carry.
- Increments interval progression.
- Invokes F-a71330230cfd with the context and the currently loaded state pointer.

The sentinel is checked again after the call. No result is checked, no progression-limit test occurs inside this loop, and no iteration cap is visible.

For a non-sentinel interval, eligibility requires unsigned start ≤ traversal position < end. Eligibility is determined before the item’s duration increment and retained for its later target-facing update.

### Counter updates and range transition

Duration-run processing precedes range processing. The within-duration-run counter and current record threshold are compared as signed 32-bit values:

- Below threshold: increment the counter.
- Otherwise: set the counter to one, then increment the duration index.

Thus, a rollover selects the next duration record before this item’s position increments are read.

The within-range counter is then incremented. If the selected range record’s signed threshold is less than the new counter, the routine increments range progression, clears accumulated item value, and sets the within-range counter to one, in that order.

The range index advances by one only when both signed comparisons hold:

- Current index is less than the 32-bit result of descriptor count minus one.
- The next-record boundary is at most range progression.

This is one conditional advancement, not a search through all satisfied boundaries.

After advancement, range-window coverage is checked again. A failed refresh returns status three after the preceding index and counter mutations, but before the position and accumulation updates below.

### Position updates and interval completion

For an eligible item, the routine reads the duration increment from the current duration record and adds its sign-extended value to the target-facing position.

It then independently reloads the duration-table information and reads an increment for the traversal position. This update occurs regardless of eligibility.

Both additions write the low word first, then the high word using sign extension and low-word carry. Results retain their low 64 bits.

Next, the retained item value is added to accumulated item value as a 32-bit update.

The optional interval pointer is checked again afterward. If present, the final end-boundary comparison uses **only the traversal low word, sign-extended to 64 bits**. The stored traversal high word does not participate. That derived signed value is compared with the stored interval end as a signed 64-bit quantity.

When the derived value is at least the interval end:

1. Interval progression is incremented.
2. The new counter is compared, signed, with the current interval descriptor’s first word.
3. A counter at or above that limit returns status three.
4. Otherwise, F-a71330230cfd is invoked with the context, without checking a result.

Traversal then returns to the stopping checks.

## Side effects

State changes are incremental, externally accessible through the context, and nontransactional.

The persisted next-item index has a different commit boundary from other fields. Reset writes one immediately, but traversal advances a private index. Only normal stopping commits that working index. A status-three exit can therefore leave counters, positions, and accumulation changed while the persisted index remains older.

Descriptor and state pointers are repeatedly reloaded, including after dependencies. Different phases can consequently access different objects if dependencies replace pointers. In contrast, retained conversion inputs, the computed target, selected item value, working index, and current eligibility decision survive the relevant calls.

No direct allocation, deallocation, ownership transfer, or external I/O is explicit. Dependency effects in those areas remain unknown. Nothing here establishes that a dependency preserves globals, retains object identity, or keeps previously referenced storage alive.

## Dependencies

| Opaque ID | Visible use and checked behavior |
|---|---|
| F-1e0edc6e0bc9 | Two arithmetic-related invocations: seven visible argument words for the first, four for the second. First contributes a 32-bit reset decision; second supplies the 64-bit target. Numerical semantics and side effects are unknown. |
| F-5e3dde636ee5 | Value-window refresh. Receives context plus zero during reset, or working index minus one during traversal. Any nonzero result returns status three. |
| F-cef02eaf8f8c | Range-window refresh. Reset visibly supplies context and zero; traversal sites visibly supply only context. Any nonzero result returns status three. |
| F-a71330230cfd | Optional interval refresh. Reset and end-boundary sites visibly supply context; sentinel processing also supplies a current state pointer. No result is checked. |

Visible argument counts do not establish complete dependency signatures or calling conventions.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless stated otherwise, storage is valid and dependencies make no unmentioned mutations.

1. **Missing state.** With a null state pointer and requested word `0x12345678`, the result is `0x1234567800000000`. No dependency is invoked.

2. **Joint inactive sentinel.** Both inactive words being `0xFFFFFFFF` produces the inactive return. Either word alone is insufficient. The low sentinel word is not checked when the high word differs.

3. **Mode mutation does not change retained selection.** Mode is one and conversion factor is seven when initially read. The first arithmetic invocation changes mode to zero and conversion factor to nine. The subsequent product remains zero, and the second invocation receives product words zero and zero followed by `1000` and `0`.

4. **Nonzero request remains selected across mutation.** Mode initially differs from one, request is three, and conversion factor is seven. If the first invocation changes mode to one and factor to nine, the subsequent product remains 21.

5. **Already at target.** The first arithmetic result is nonzero and the retained original position equals the target. With no intervening position mutation, no item is consumed; the loaded next-item index is written back unchanged.

6. **Reset failure preserves partial state.** A backward target selects reset. A required value-window refresh returns nonzero. The initial seven reset writes remain visible, but the final position and duration-run clears do not occur. The result has status three.

7. **Exclusive interval end.** Traversal position equals interval end before item processing. The item is ineligible for the target-facing duration update, but traversal, counters, and accumulated item value still update.

8. **Duration rollover affects this item.** Duration index is zero, its threshold and current run counter are both two, and record one has increment seven. Processing selects record one before duration addition; traversal advances by seven.

9. **Final boundary ignores traversal high word.** After an item, traversal position is `0x0000000080000000` and interval end is positive one. The final comparison uses negative 2147483648, so it does not trigger interval progression.

10. **Persistent sentinel.** If the interval dependency indefinitely leaves both start words all ones, sentinel processing does not terminate visibly. Each repetition still adds the adjustment and increments progression.

## Uncertainties

- **documented_only — arithmetic dependency:** Exact operation, signedness conventions, rounding, exceptional cases, and meaning of the first invocation’s extra arguments are unresolved.
- **documented_only — ownership and lifetime:** Context and table ownership, dependency allocation or release, and storage-lifetime requirements beyond accessible reads and writes are unknown.
- **documented_only — dependency ABI and mutations:** Complete signatures and mutation guarantees are unavailable.
- **documented_only — extreme arithmetic:** Width-limited calculations and signed comparisons are visible, but valid index ranges, address-overflow rules, and a safe domain for every signed increment are not supplied.
- **documented_only — termination:** Zero or negative increments, wraparound, changing dependencies, and persistent sentinels prevent a general termination guarantee.
- **documented_only — domain interpretation:** Units, item meanings, and reasons for the interval-end low-word asymmetry are unknown.
- **documented_only — error distinction:** Status three merges checked refresh failures and interval-count exhaustion. No separate cause is returned.

## Evidence

Established static evidence consists of the supplied representation of F-f84ad0bfa33c and its opaque dependency mapping. It supports the argument widths, offsets, retained inputs, access ordering, branch conditions, dependency invocations, state transitions, and result-word construction described above.

In particular, mode selection precedes the first arithmetic invocation and remains retained afterward. The context is caller-supplied; ownership is not established.

Domain interpretation is explicitly inferred. Dependency internals and unresolved execution conditions remain documented unknowns.

No runtime tests have been performed.
