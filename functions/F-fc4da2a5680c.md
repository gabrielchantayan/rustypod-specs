<!-- Trusted provenance: F-fc4da2a5680c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fc4da2a5680c

## Purpose

F-fc4da2a5680c performs an unconditional dependency call on an embedded object location, conditionally processes object state, and finally decides whether to invoke a function obtained from a global function-pointer slot.

Three distinct values govern the final invocation:

- A pointer read from the object.
- The first 32-bit word at that pointer.
- A function pointer read from a separate global slot.

The first two values are guards. Neither supplies the invocation target. The global function-pointer slot is read only after both guards succeed, following all earlier dependency calls and any direct mode-byte clear.

The apparent intent is to coordinate a selected-entry operation and consume a pending mode indication before conditional dispatch. That interpretation is inferred. Application-level meanings for the fields, operations, and global dispatch target are not established.

## Interface

### Input and result

The function receives one object-base address, designated **B**. The supplied representation carries B in a 32-bit integer-shaped parameter and accesses storage at byte offsets from it.

There is no explicit result value. Observable results are dependency invocations, one possible direct byte write, and effects produced by invoked dependencies.

The function also consumes a **global dispatch-target slot** when final dispatch is enabled. This is an implicit global input, separate from the object and its referenced storage.

No null check or object-size check precedes the initial dependency call.

### Data representation

The visible model uses 8-bit bytes, 32-bit integer words, and 32-bit object-pointer slots. Selector-dependent locations have a 24-byte stride. The selected-word comparison is signed and 32-bit.

| Location | Representation | Established use |
|---|---|---|
| B + `0x18` | Embedded location, later read as an object-pointer slot | Its address is passed to F-338d16f9ce72. Its stored pointer is read during final dispatch processing. |
| B + `0xE9` | Byte | First enable condition. |
| B + `0xEA` | Byte | Second enable condition; read only when the first is nonzero. |
| B + `0xEB` | Byte | Mode condition; directly cleared after normal completion of the nonzero-mode path. |
| B + `0xEC` | Byte converted to an integer | Selector for address calculations and operation arguments. |
| B + `0x1C4`, `0x1C8`, `0x1CC`, `0x1D0` | Four 32-bit words | Sequential zero-mode scan. |
| B + `0x88` + 24 × selector | Address | Passed directly to F-e5a6af26f645. |
| B + `0x24` + 24 × selector | Signed 32-bit word | Left operand of a strict comparison. |
| B + `0x28` + 24 × selector | Signed 32-bit word | Right operand of that comparison. |
| First word at the final object pointer | 32-bit word | Nonzero guard for final dispatch; not the dispatch target. |
| Global dispatch-target slot | Function-pointer storage | Read only after both final guards succeed; its value supplies the indirect invocation target. |

The representation does not establish a complete object layout, entry count, selector range, ownership model, or complete function-pointer ABI.

### Preconditions

For defined execution, storage and dependency state must support every operation actually reached:

- F-338d16f9ce72 must accept the address B + `0x18`.
- Reached object fields must be readable, and B + `0xEB` must be writable when the nonzero-mode clear is reached.
- Selector-derived addresses must designate storage suitable for the reached reads or dependency call.
- A nonnull final object pointer must designate a readable 32-bit word.
- When both final guards succeed, the global dispatch-target slot must be readable and contain a target valid for the actual indirect invocation convention.

These conditions are not validated here. In particular, there is no selector bounds check and no explicit null check of the function pointer read from the global slot.

## Behavior

### Initial dependency invocation

The first established action is invocation of F-338d16f9ce72 with the address B + `0x18`.

The argument is the address of the embedded location, not the pointer value stored there. Neither enable byte is consulted beforehand.

All subsequent object reads occur after this dependency returns. Its effects cannot be presumed absent: its separate contract may permit changes to object fields, pointed-to storage, the global dispatch-target slot, or other state.

### Enable gating

After the initial call returns, the function reads the byte at B + `0xE9`.

If that byte is zero, it skips gated processing without reading B + `0xEA`. Otherwise, it reads B + `0xEA` and enters gated processing only if this second byte is also nonzero.

Any nonzero byte satisfies either gate; the value need not be one.

These decisions are not reevaluated after later dependencies. If either gate fails, the mode byte is not read for gated processing and is not directly cleared. Final dispatch processing still follows.

### Zero-mode processing

When both gates pass, the function reads B + `0xEB`.

If the mode byte is zero, it scans the four 32-bit words at B + `0x1C4`, B + `0x1C8`, B + `0x1CC`, and B + `0x1D0`, in increasing address order.

The first nonzero word terminates the scan. At that point, the function reads the selector byte and invokes F-9d39dcd3197c with four explicit arguments:

1. B.
2. The selector converted to an integer.
3. Zero.
4. Decimal 125.

The scanned value and its position are not passed. Only its nonzero status controls the invocation.

At most one such invocation occurs. Words after the first nonzero word are not read by the scan. If all four words are zero, the scan completes without invoking F-9d39dcd3197c.

There is no direct mode-byte write on this path. If a dependency modifies the mode byte, this function does not subsequently clear or restore it on account of zero-mode processing.

### Nonzero-mode processing

If the mode byte is nonzero, the function reads the selector and invokes F-e5a6af26f645 with the address B + `0x88` + 24 × selector.

This is an address argument. The function does not first fetch a value from that selected address.

A zero result from F-e5a6af26f645 suppresses both the selected-word comparison and the two later operation calls.

For any nonzero result, the function reads the selector again and derives a new selected base from that reading. It reads the signed 32-bit words at offsets `0x24` and `0x28` from that selected base and tests whether the first is strictly less than the second.

Equality fails the comparison. Negative values participate according to signed 32-bit ordering. The supplied representation does not establish the physical read order between the two comparison operands.

If the comparison succeeds, the function:

- Invokes F-da34ea4872e6 with no explicit arguments.
- After that call returns, reads the selector again.
- Invokes F-9d39dcd3197c with B, the newly read selector converted to an integer, one, and decimal 125.

The initial selected address, comparison pair, and final selector argument need not correspond to the same selector value. Dependencies between those stages may modify the selector. The comparison pair itself uses one selector reading.

After normal completion of this nonzero-mode path, the function stores zero into the byte at B + `0xEB`. This happens whether the first dependency returned zero, the comparison failed, or both later dependencies were invoked and returned.

The clear follows every dependency reached on this path. It therefore overwrites a mode value left by those dependencies. No clear occurs before them.

If a dependency fails to return normally, the visible control flow does not establish execution of the subsequent mode clear or final dispatch processing.

### Final dispatch processing

After gated processing is skipped or completes, the function reads the pointer stored at B + `0x18`.

If this pointer is null, there is no pointee read, no global dispatch-target read, and no indirect invocation.

If the pointer is nonnull, the function reads its first 32-bit word. A zero word suppresses both the global dispatch-target read and the indirect invocation. Any nonzero word satisfies this guard; positivity is not required.

Only after the pointer is nonnull and its first word is nonzero does the function read the separate global dispatch-target slot. The value obtained from this global read supplies the indirect invocation target. The invocation has no explicit arguments in the supplied representation.

There is no intervening explicit validation of that target. The object pointer and its first word are not interpreted as function addresses.

The global read occurs after all preceding dependency calls and after any direct mode clear. The target is therefore not fixed by this function and is not an entry-time snapshot. Earlier dependencies must not be assumed to preserve the global slot. If an earlier dependency changes it, the final invocation uses the value obtained by the later global read, subject to any otherwise unspecified concurrent mutation.

If the indirect dependency returns normally, the function returns without further explicit work.

## Side effects

### Direct memory effects

The sole explicit memory write is the zero byte stored at B + `0xEB` after normally completed nonzero-mode processing.

There are no explicit writes to the enable bytes, selector, scanned words, comparison words, object-pointer slot, or global dispatch-target slot. There are no explicit allocations, releases, or reference-count operations.

These are statements about this function’s direct operations, not guarantees about dependency effects.

### Observable ordering

The established ordering is:

1. Invoke F-338d16f9ce72.
2. Read the first enable byte and conditionally the second.
3. If enabled, read the mode byte and perform the selected mode path.
4. After normal completion of the nonzero-mode path, clear the mode byte.
5. Read the object-pointer slot at B + `0x18`.
6. If that pointer is nonnull, read its first 32-bit word.
7. If that word is nonzero, read the global dispatch-target slot.
8. Invoke the target obtained from that global slot.

Reads within the mode paths have the conditional ordering described above. The comparison operands’ physical read order remains unspecified.

There are no explicit retries, rollback actions, or synchronization operations. Earlier dependency effects are not undone when a later condition fails. Neither object state nor global state is guaranteed to form a coherent snapshot under concurrent mutation.

### Ownership and lifetime

Passing an embedded or selected address does not establish ownership transfer. The evidence supplies no contract for retaining, replacing, releasing, or invalidating the associated storage.

The final object pointer must remain valid for its reached dereference. The function pointer obtained from the global slot must be callable when invoked. Earlier dependencies cannot be assumed to preserve either lifetime or either stored value.

## Dependencies

| Dependency | Invocation condition | Explicit inputs | Result used here |
|---|---|---|---|
| F-338d16f9ce72 | Always first | Address B + `0x18` | No result is used. |
| F-e5a6af26f645 | Both gates pass and mode is nonzero | Address B + `0x88` + 24 × selector | Zero suppresses the comparison; nonzero permits it. |
| F-da34ea4872e6 | F-e5a6af26f645 returns nonzero and the signed comparison succeeds | None shown | No result is used. |
| F-9d39dcd3197c | First nonzero scanned word, or successful nonzero-mode comparison after F-da34ea4872e6 returns | B, converted selector, zero or one, 125 | No result is used. |

A further dependency is selected through the global dispatch-target slot. No opaque function ID is supplied for its possible targets. The slot is read only after the final object-pointer and pointee-word guards succeed. Its then-read contents, rather than an earlier saved value, determine the invocation.

All dependency side effects, ownership behavior, and complete calling conventions remain unspecified unless supplied by separate contracts.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Except where mutation is specified, assume relevant dependencies return normally and leave the state used by the example unchanged.

1. **First gate disabled.**  
   If B + `0xE9` contains zero after F-338d16f9ce72 returns, the second enable byte and mode byte are not read for gated processing. Final dispatch processing still occurs.

2. **Second gate disabled.**  
   If the first enable byte is nonzero and the second is zero, neither mode path runs and this function does not directly clear the mode byte.

3. **Zero mode without a trigger.**  
   With both gates enabled, zero mode, and four zero scan words, all four words are examined. Gated processing invokes neither F-e5a6af26f645 nor F-9d39dcd3197c.

4. **Zero mode with an early trigger.**  
   If the first scan word is zero and the second is nonzero, only those two words are scanned. F-9d39dcd3197c receives B, the then-current selector, zero, and 125. No direct mode clear follows.

5. **Nonzero mode rejected by the dependency.**  
   If F-e5a6af26f645 returns zero, the comparison words are not read on that path. The mode byte is cleared before the final object-pointer read.

6. **Signed comparison boundaries.**  
   With a nonzero dependency result, comparison values 7 and 7 do not trigger the later operation calls. Values −1 and 0 do. Both normally completed cases clear the mode byte afterward.

7. **Selector changes between stages.**  
   If the selector is 1 for the initial selected-address call and F-e5a6af26f645 changes it to 2, the comparison uses B + `0x54` and B + `0x58`. If that comparison succeeds and F-da34ea4872e6 changes the selector to 3, F-9d39dcd3197c receives selector 3.

8. **Mode changes inside an operation.**  
   If F-9d39dcd3197c writes a nonzero mode byte during nonzero-mode processing and returns, this function subsequently replaces that byte with zero. Zero-mode processing has no corresponding overwrite.

9. **Final guard boundaries.**  
   A null final object pointer causes neither a pointee read nor a global target read. A nonnull pointer to a zero word causes a pointee read but no global target read. A nonnull pointer to any nonzero word permits the global target read and indirect invocation.

10. **Global target mutation.**  
    Suppose the global slot initially contains target A, and a reached dependency replaces it with valid target B. If the final guards succeed and nothing subsequently changes the slot, the invocation uses B. This applies even when both enable gates fail and the only preceding dependency is F-338d16f9ce72.

11. **Distinct guard and target values.**  
    If the final pointee word is a nonzero value such as 1, that value satisfies the guard but does not become an invocation address. The invocation target comes from the global slot. A null value in that slot is not explicitly rejected; this function provides no defined recovery behavior for an invalid target.

## Uncertainties

The following remain **documented_only**:

- **Selector signedness.** The selector is represented as plain character data converted to an integer. Values `0x00`–`0x7F` have the same interpretation under signed and unsigned character models. For `0x80`–`0xFF`, the evidence does not establish whether conversion produces negative values or 128–255.
- **Address arithmetic.** The selector’s one-byte range does not itself threaten a 32-bit multiplication by 24, but adding the object base and offsets may encounter address-range limits. The evidence does not justify portable wraparound guarantees. Reached addresses must be valid.
- **Logical bounds and layout.** No valid selector range, entry count, or complete relationship among the selected regions is established.
- **Global target identity and ABI.** The separate global slot and its late read are established. Its application-level identity, initialization rules, possible targets, function-pointer ABI, and update synchronization are not.
- **Dependency effects.** Object mutation, global mutation, ownership changes, memory lifetime, and nonlocal control flow require separate contracts.
- **Terminal transfer details.** The supplied representation presents the indirect transfer as an invocation without explicit arguments. Its complete machine-level calling and return behavior are not independently established.
- **Failure handling.** No explicit error result, invalid-address recovery, exception handling, rollback, or cleanup path is visible.

## Evidence

Established static evidence consists of the supplied function representation: field offsets and widths, short-circuit gates, sequential scan, explicit dependency arguments, repeated selector reads, signed comparison, mode-byte clear, final object-pointer and pointee-word checks, and subsequent indirect invocation through a separate global slot.

The global target is an implicit input read at the point of dispatch, not a fixed dependency or an entry-time snapshot. This follows from its placement after both final guards and after preceding calls and writes.

Terms such as “enable,” “mode,” “selector,” and “dispatch” describe observed roles, not proven domain semantics. The coordination purpose is inferred intent.

No runtime tests, dependency implementations, independent ABI verification, or memory-layout validation have been performed. The acceptance examples state conditional consequences of the visible control flow, not observed execution results.
