<!-- Trusted provenance: F-dfef7b6c143e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dfef7b6c143e

## Purpose

F-dfef7b6c143e conditionally replaces a pair of 32-bit object fields, calls a notification dependency when replacement occurs, and then processes an embedded state region.

The pair update and embedded-state processing are not transactional. Requested values can be written and notification can occur before the function returns an error. Clearing of embedded state can likewise precede a nonzero dependency result. No rollback is visible.

**Established static behavior:** conditional pair replacement, ordered dependency calls, an owner-like comparison, mode-dependent count handling, and propagation of selected dependency results.

**Inferred intent:** the embedded region resembles an ownership-sensitive synchronization object, with flags, a recursion-like count, and a lower-level release operation. These interpretations are not established API guarantees. In particular, the initial dependency is not proven to acquire anything, and the final dependency is not proven to release memory or synchronization.

## Interface

### Inputs

The supplied representation exposes three 32-bit integer inputs:

| Input | Interpretation within this function |
|---|---|
| Object address | Used to form addresses for direct accesses and dependency arguments. |
| Requested first value | Compared with and potentially stored at object offset `+0x48`. |
| Requested second value | Compared with and potentially stored at object offset `+0x40`. |

Both requested values are handled as complete 32-bit values. No range restriction, normalization, or semantic validation is visible.

The object address is represented as an integer but used as an address. Callers must provide storage suitable for every access reached on their execution path, together with any additional storage required by dependencies. The function does not validate the object before invoking its first dependency.

### Result

The result is represented as a 32-bit integer.

| Reached condition | Returned value |
|---|---|
| Derived embedded-region address is null at the late check | `26` |
| Distinguished-state dependency returns nonzero | That result unchanged |
| Owner-like field differs from the identity dependency’s result | `5` |
| Owner matches, decrement mode applies, and the narrowed decremented count is nonzero | `0` |
| Count and owner are cleared and the final dependency is called | Final dependency result unchanged |

The meanings of `5` and `26` are not established. They must not be assigned symbolic error names from this evidence alone.

A nonzero dependency result need not be positive. For example, a negative result from the distinguished-state dependency is propagated without a visible sign restriction.

### Visible layout

Offsets are relative to the object address.

| Offset | Access width | Role established by this function |
|---|---:|---|
| `+0x04` | 32 bits | Loaded as the first notification argument. |
| `+0x24` | 32 bits | First embedded-state word, compared with a distinguished global value. This address is also passed to two dependencies. |
| `+0x28` | 32 bits | Owner-like field; compared with an identity result and cleared on the final-operation path. |
| `+0x32` | 16 bits | Unsigned flags snapshot. |
| `+0x36` | 16 bits | Signed count representation, conditionally decremented or cleared. |
| `+0x38` | Not established | Address passed to the final dependency. |
| `+0x40` | 32 bits | Stored second requested value. |
| `+0x48` | 32 bits | Stored first requested value. |

For an ordinary contiguous allocation, the direct access at `+0x48` requires storage through offset `+0x4B`. This is not a complete allocation-size requirement: dependencies may access more storage.

Target endianness, alignment requirements, address-arithmetic semantics, and dependency-specific layouts are not established.

## Behavior

### Entry and pair replacement

The first visible action is a call to an entry dependency with the address at object offset `+0x24`. No result from that call is inspected.

This dependency has no supplied mapped opaque identity. This contract uses **F-000000000001** as a document-local opaque label, not as a verified published identity.

All pair comparisons occur after the entry dependency returns. Consequently, they concern the state then present, not necessarily the state at function entry.

The first comparison reads the 32-bit field at `+0x48`.

- If it differs from the requested first value, replacement is selected without reading `+0x40` for comparison.
- If it matches, the function reads `+0x40` and compares it with the requested second value.
- Replacement is skipped only when both comparisons match.

When replacement is selected, the observable direct-access order is:

1. Store the requested first value at `+0x48`.
2. Store the requested second value at `+0x40`.
3. Read the 32-bit notification argument at `+0x04`.
4. Invoke F-1a28d27dcc4e with that argument and integer constant `10`.

Both fields are written even when only one differs. The notification result is ignored.

These stores precede entry into the notification dependency. That ordering does not establish atomic publication, synchronization, or visibility guarantees for concurrent observers. The dependency may itself change the object; the function does not revalidate the pair afterward.

### Late embedded-address check

After the pair phase, the function checks whether the address formed at object offset `+0x24` is null. If so, it returns `26`.

This check does not protect earlier operations. The entry dependency and pair comparisons, and possibly pair writes and notification, have already occurred.

A null object address is not equivalent to a null embedded-region address. Whether the latter can arise through admissible address arithmetic without earlier faults or invalid operations is unresolved.

### Distinguished-state gate

For a nonnull embedded-region address, the function compares its first 32-bit word with a distinguished global value.

If they differ, F-621dae46a146 is skipped and processing continues.

If they match, F-621dae46a146 is called with the embedded-region address and integer zero.

- A nonzero result is returned immediately.
- A zero result permits subsequent processing.

A normal return from this dependency is therefore insufficient to establish that later count processing occurs: its returned value must be zero.

On the early nonzero-result path, the function does not subsequently read flags, call the identity dependency, compare the owner, modify the count, clear the owner, or call the final dependency.

No preservation contract is supplied for F-621dae46a146. After a zero result, subsequent accesses observe whatever state exists after that call.

### Flags snapshot and owner gate

After passing the distinguished-state gate, the function reads the unsigned 16-bit flags at `+0x32`. It retains this snapshot, then calls F-196812b4b122 with no visible arguments.

Only after that dependency returns does it read the 32-bit owner-like field at `+0x28` and compare it with the dependency’s result.

This ordering has two distinct consequences:

- Changes to flags during F-196812b4b122 do not replace the saved flags used for this invocation.
- Changes to the owner-like field during that dependency affect the subsequent comparison.

If the owner-like field does not equal the returned identity value, the function returns `5`. It performs no direct count write, owner clearing, or final dependency call on that branch. Earlier dependencies and pair updates may already have had effects.

Owner equality is a separate prerequisite for count processing. Neither normal dependency returns nor absence of dependency mutations establishes it.

### Mode and count handling

With a matching owner, saved flags select decrement mode exactly when bits 5 and 4 are binary `10`: bit 5 set and bit 4 clear. Equivalently, the saved flags masked by `0x0030` equal `0x0020`.

All other flag bits are irrelevant to this selection.

In decrement mode, the function reads the signed 16-bit count at `+0x36`, subtracts one, and writes the result back as 16 bits. It then tests that narrowed result for zero.

- A nonzero narrowed result causes return `0`, without direct owner clearing or the final dependency call.
- A zero narrowed result selects the clearing path.

There is no visible requirement that the initial count be positive. In particular, zero becomes negative one, which is nonzero, so this case returns `0` without clearing the owner.

For a signed 16-bit input, the mathematical subtraction ranges from `−32769` through `32766`, which fits a signed 32-bit intermediate. The exceptional conversion boundary is the narrowing of `−32769` back to a signed 16-bit representation. Exact behavior at that boundary remains documented_only.

When decrement mode is not selected, the function proceeds directly to clearing. This branch does not require a direct read of the previous count.

### Clearing and final operation

The clearing path performs these actions in order:

1. Store 16-bit zero at `+0x36`.
2. Store 32-bit zero at `+0x28`.
3. Call F-259d2fe281b0 with the address at object offset `+0x38`.
4. Return its result unchanged.

If decrement mode began with count one, the count receives two zero writes: the decrement write and the explicit clearing write.

A nonzero final result does not undo either clearing write. However, the fields’ contents after the dependency returns remain dependent on its unknown effects; the direct pre-call stores do not guarantee final zeros.

## Side effects

Direct writes are confined to:

- The 32-bit pair at `+0x48` and `+0x40`, when replacement is selected.
- The 16-bit count at `+0x36`, when decrementing or clearing.
- The 32-bit owner-like field at `+0x28`, when clearing.

The function reads a distinguished global value but shows no direct write to it. Dependencies may nevertheless change globals or object storage; preservation must not be assumed.

No direct allocation, deallocation, address retention, or transfer of storage ownership is visible. Supplying an address to a dependency does not establish whether that dependency retains it, owns it, or releases associated storage.

No explicit atomic access or memory-order guarantee is established. Any synchronization depends on missing dependency contracts and caller requirements.

Errors are not side-effect-free. A nonzero return may follow pair replacement and notification, or count and owner clearing. Dependencies can introduce additional effects on any reached path.

## Dependencies

| Opaque ID | Invocation and visible result handling | Unestablished properties |
|---|---|---|
| F-000000000001 | Called first with address `+0x24`; result not inspected. Document-local identity only. | Published identity, synchronization, memory effects, failure behavior. |
| F-1a28d27dcc4e | Called after pair replacement with the 32-bit value read at `+0x04` and constant `10`; result ignored. | Meaning of arguments, notification semantics, callbacks, reentrancy, memory effects. |
| F-621dae46a146 | Called only on a distinguished-word match, with address `+0x24` and zero; nonzero result propagated. | State initialization or repair, global preservation, memory effects, error meanings. |
| F-196812b4b122 | Called after the flags snapshot; result compared with a subsequent owner-field read. | Identity domain, uniqueness, stability, result interpretation, memory effects. |
| F-259d2fe281b0 | Called after clearing, with address `+0x38`; result propagated. | Release semantics, ownership effects, storage lifetime, memory effects, error meanings. |

Dependency calls are not proven to return normally. Blocking, faults, exceptions, and nonlocal transfer are outside the established contract.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Each assumes valid storage for reached accesses, normal return from reached dependencies, and no dependency mutations except those explicitly described. Normal return alone does **not** imply a zero result.

1. **Pair already present.**  
   Both stored values match their requested values after F-000000000001 returns. The function performs no pair writes and no notification. Embedded-state processing still occurs, subject to its gates; pair equality does not determine the result.

2. **First value differs.**  
   After the entry call, the first stored value differs. The function does not read the second field for comparison. It writes both requested values in first-then-second order, reads `+0x04`, and invokes F-1a28d27dcc4e with that value and `10`. Later processing may still return nonzero.

3. **Only the second value differs.**  
   The first comparison matches and the second does not. Both fields are read for comparison, then both are written, including the already-matching first field. Notification follows.

4. **Distinguished-state rejection.**  
   The embedded address is nonnull, its first word matches the global value, and F-621dae46a146 returns `−7`. The function returns `−7` without the later flags read, identity call, or count processing. Earlier pair updates remain possible.

5. **Owner mismatch after passing the gate.**  
   The embedded address is nonnull. Its first word either differs from the global value or matches and F-621dae46a146 returns zero. The owner read after F-196812b4b122 differs from that dependency’s result. The function returns `5` without directly writing the count or owner.

6. **Count two decrements without clearing.**  
   The embedded address is nonnull; distinguished-state handling either skips F-621dae46a146 or receives zero from it. The owner read after F-196812b4b122 equals its result. Saved flags select decrement mode, and the count read is two. The count becomes one, the function returns `0`, and no final dependency call occurs.

7. **Count one reaches clearing.**  
   The embedded address is nonnull; the distinguished word differs from the global value or F-621dae46a146 returns zero. The post-identity-call owner read equals the identity result. Saved flags select decrement mode, and the count read is one. The count is written as zero twice, the owner is cleared, and F-259d2fe281b0 is called. Its result is returned.

8. **Non-decrement mode clears directly.**  
   The embedded address is nonnull, distinguished-state handling does not return early, and the post-identity-call owner read matches the identity result. Saved flags do not select decrement mode. The function clears the count without directly reading its prior value, clears the owner, calls F-259d2fe281b0, and returns its result.

9. **Final operation reports an error.**  
   All gates permit the clearing path, and F-259d2fe281b0 returns `9`. The function returns `9`. The count and owner clearing writes occurred before the call and are not rolled back by this function.

10. **Zero count in decrement mode.**  
    The embedded address is nonnull; the distinguished word differs from the global value or F-621dae46a146 returns zero. The owner read after F-196812b4b122 equals its result. Saved flags select decrement mode, and the count read is zero. The count becomes negative one; the function returns `0` without clearing the owner or calling F-259d2fe281b0.

11. **Identity call changes flags.**  
    The embedded address is nonnull and distinguished-state handling permits continuation. Saved flags select decrement mode. F-196812b4b122 changes the flags to another mode, but the subsequent owner read still equals its result. The function uses the earlier snapshot and performs decrement-mode processing.

## Uncertainties

The following remain **documented_only**:

- Exact ABI, endianness, alignment, pointer conversion, address overflow, and address-wrap behavior.
- Published identity and contract of the entry dependency.
- Distinguished global value, meaning, initialization, and permissible concurrent changes.
- Dependency preservation of object fields, globals, addresses, or storage lifetime.
- Whether the owner-like value denotes a thread, task, or another identity.
- Whether callers must maintain a positive count in decrement mode.
- Signed narrowing at the minimum count boundary. Under low-16-bit truncation, decrementing representation `0x8000` produces `0x7FFF`, a nonzero result; that conversion rule is not independently established.
- Reachability of the late null-derived-address branch without earlier invalid operations.
- Final object contents after mutating dependencies.
- Synchronization, ownership-transfer, and allocation or release semantics.

The ownership and synchronization interpretation remains inferred intent, not established behavior.

## Evidence

This contract derives from the supplied static function representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent proof of behavior.

Static evidence establishes access widths and offsets, short-circuit pair comparisons, direct write order, dependency argument formation, the distinguished-state early return, flags-snapshot timing, the post-call owner read, count-mode selection, clearing order, and visible result propagation.

The count examples require both continuation past distinguished-state handling and a matching owner. These are independent gates in the supplied control flow; normal dependency return does not establish either condition.

No dependency implementations, complete object definition, caller invariants, or target-level arithmetic confirmation were supplied. No runtime tests have been performed.
