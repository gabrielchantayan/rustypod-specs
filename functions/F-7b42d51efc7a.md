<!-- Trusted provenance: F-7b42d51efc7a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7b42d51efc7a

## Purpose

F-7b42d51efc7a conditionally advances or retreats an indexed selection, resolves the proposed selection through a callback, optionally processes the returned record according to its type, and updates destination-associated state.

The source selection and destination slot are separate inputs. The proposed selection is calculated from the source slot, but a successful change is recorded in the destination slot. A no-change decision compares the proposal with the original source selection; it does not inspect the destination’s current selection.

The apparent intent is navigation with optional wrapping and type-specific content preparation. That interpretation is inferred. The established behavior is the sequence of reads, calls, and writes described below.

## Interface

### Inputs

The function has four explicit inputs:

| Input | Representation in the supplied material | Role |
|---|---|---|
| State base | 32-bit address-bearing value | Base of the mutable state layout. |
| Source slot | 32-bit integer | Selects the initial selection value using a four-byte stride. |
| Destination slot | 32-bit integer | Selects destination-associated fields using four-byte and one-byte strides. Also passed to dependencies. |
| Auxiliary value | Opaque 32-bit value | Passed twice to F-44341bd71b97; otherwise not directly interpreted. |

No explicit validation of these inputs is visible.

The supplied material represents addresses and address-bearing fields with 32-bit quantities. Exact target ABI details, pointer validity rules, alignment requirements, and byte order are not independently established.

### Return and completion

There is no explicit result value. Observable results consist of:

- Calls to dependencies and the callback.
- Changes to state memory.
- A one-byte destination status write on ordinary completion.
- An early return when the callback returns a zero address.

The zero-address early return occurs after the preparatory dependency call. It does not write the destination status byte.

### State layout accessed directly

Offsets are byte offsets from the state base. Indexed offsets are arithmetic expressions, not declarations of independently sized arrays.

| Location | Width | Use |
|---|---:|---|
| `0x74 + 4 × source slot` | 32 bits | Initial source selection. |
| `0x92` | 8 bits | Direction or mode discriminator: values 1 and 2 have defined proposal-generation paths. |
| `0xE7` | 8 bits | Zero selects clamping; nonzero selects wrapping. |
| `0xA0` | 32 bits | Selection limit used by particular boundary paths. |
| `0xAC` | Address-sized, represented as 32 bits | Callback address. |
| `0x24 + 4 × destination slot` | 32 bits | Address-bearing destination descriptor field. |
| `0x74 + 4 × destination slot` | 32 bits | Destination selection written after successful resolution. |
| `0x10 + 4 × destination slot` | 32 bits | Receives the result of F-e64c9fe93b53. |
| `0xF8`, `0xF4`, `0xF0` | 32 bits each | Each receives the destination slot, in that order. |
| `0xFC` | 32 bits | Receives the callback-returned record address. |
| `0x8D + destination slot` | 8 bits | Receives 0 for no change or 1 for a completed change. |

The destination descriptor is dereferenced at offset `+4` to read a 32-bit value. The callback-returned record is accessed at:

- Offset `+0x14`: a 16-bit type discriminator.
- Offset `+4`: a 32-bit payload, read only for recognized types.

These offsets do not establish the complete size or ownership of either object.

### Preconditions

For a defined, normally completing execution:

- All accessed addresses must be valid for the corresponding read or write.
- The mode must select a path that establishes a proposal, ordinarily mode 1 or mode 2.
- Any callback address that is invoked must be callable with one 32-bit selection argument.
- A nonzero callback result must permit the record accesses performed on its path.
- Destination descriptor addresses must permit the relevant dereference.
- Arithmetic must have defined target behavior.
- Called functions must return normally if subsequent effects are expected.

No supported slot range is established. The offset expressions can overlap other state fields for some slot values; the function does not prevent such overlap.

## Behavior

### 1. Capture the source selection

The function first reads the 32-bit selection at `0x74 + 4 × source slot`.

This captured value remains the comparison baseline even if a later dependency changes the source selection in memory.

The function then reads the mode byte at `0x92`. If that first read is not 1, the supplied control flow reads the mode byte again to test for 2. The proposal rules below assume a stable mode value during these reads.

### 2. Form a proposed selection

The selection limit is treated as a signed 32-bit integer for the visible boundary comparisons. There is no validation that it is positive or consistent with the current selection.

#### Mode 1: advance

The initial proposal is the captured source selection plus one.

The function reads the limit at `0xA0`, then reads the wrapping byte at `0xE7`.

- If the proposal is strictly less than the limit, it is retained.
- If the proposal is greater than or equal to the limit and wrapping is zero, the proposal becomes the limit minus one.
- If the proposal is greater than or equal to the limit and wrapping is nonzero, the proposal becomes zero.

This path does not independently enforce a nonnegative proposal. With unusual initial selections or limits, its result need not be a conventional valid index.

#### Mode 2: retreat

The initial proposal is the captured source selection minus one.

The function reads the wrapping byte at `0xE7`.

- If the proposal is nonnegative, it is retained, regardless of the wrapping byte.
- If the proposal is negative and wrapping is zero, the proposal becomes zero.
- If the proposal is negative and wrapping is nonzero, the function reads the limit at `0xA0` and sets the proposal to the limit minus one.

Unlike the advance path, the retreat path does not always read the limit. It reads it only for a negative proposal with wrapping enabled.

This path does not independently enforce an upper bound on an already nonnegative proposal.

#### Other modes

For a stable mode other than 1 or 2, the supplied material reaches the comparison without establishing the proposal within this function.

No deterministic selection behavior is specified for that case. It must not be treated as a documented no-op, a default direction, or an error return. This case is documented_only.

### 3. Compare with the captured source selection

The function compares the proposal with the captured source selection.

If they are equal:

1. No dependency or callback is invoked by the visible path.
2. A zero byte is written at `0x8D + destination slot`.
3. The function returns.

It does not copy the source selection into the destination selection field. Thus, even if source and destination slots differ, equality between the proposal and the source selection suppresses the change path.

If the values differ, processing continues.

### 4. Perform preparation and resolve the proposal

The first call on the change path is F-44341bd71b97. It receives six arguments in this order:

1. State base.
2. Destination slot.
3. Destination slot.
4. Auxiliary value.
5. Destination slot.
6. Auxiliary value.

No result from this call is used directly.

After that call returns, the function obtains the callback address from state offset `0xAC` and calls it with the proposed selection as its sole visible argument. Consequently, the callback address used can reflect changes made by F-44341bd71b97.

If the callback returns zero, the function returns immediately. On this path, it performs none of the subsequent direct descriptor reads, type dispatch, selection commits, tracking-field writes, or status-byte writes.

This early return does not undo any effects of F-44341bd71b97 or the callback.

### 5. Inspect and optionally process the resolved record

For a nonzero callback result, the function calculates the destination’s four-byte-stride state address.

It first:

1. Reads the destination descriptor address from `0x24 + 4 × destination slot`.
2. Reads a 32-bit value at offset `+4` from that descriptor.

This descriptor-derived value is read even if the record type is unrecognized.

The function then reads the record’s 16-bit type at offset `+0x14`.

- For type `0x0565`, it calls F-a7562a402053.
- Otherwise, it reads the 16-bit type again and compares it with `0x1888`.
- For type `0x1888`, it calls F-08741ff22c05.
- For any other observed type, it makes neither type-specific call and continues to commit the change.

Each recognized-type dependency receives:

1. State base.
2. Destination slot.
3. The record’s 32-bit payload at offset `+4`.
4. The descriptor-derived 32-bit value already read.
5. The integer constant 6.
6. The integer constant `0x50`.

Neither type-specific call has a directly consumed result. An unrecognized type is not a rejection condition in this function.

The second type read matters if the record can change between reads: the second branch is based on that later observation, not necessarily the first. No concurrency or synchronization guarantees are established.

### 6. Commit the destination state

After type processing, or immediately after an unrecognized type, the function performs these actions in order:

1. Writes the proposal as a 32-bit value to `0x74 + 4 × destination slot`.
2. Reads the descriptor address again from `0x24 + 4 × destination slot`.
3. Calls F-e64c9fe93b53 with the state base, that newly read descriptor address, and zero.
4. Writes the dependency’s 32-bit result to `0x10 + 4 × destination slot`.
5. Writes the destination slot to `0xF8`.
6. Writes the destination slot to `0xF4`.
7. Writes the destination slot to `0xF0`.
8. Writes the callback-returned record address to `0xFC`.
9. Writes the byte value 1 to `0x8D + destination slot`.
10. Returns.

The descriptor used by F-e64c9fe93b53 is freshly read. It is not necessarily the descriptor used earlier to obtain the value supplied to type-specific processing.

The record address stored at `0xFC` is the value originally returned by the callback. The function does not resolve it again after subsequent dependency calls.

## Side effects

The visible function directly writes state only on the no-change and completed-change paths described above. Dependencies may introduce additional effects that are not constrained by this material.

The sequence is not transactional:

- Preparation precedes callback resolution.
- A zero callback result does not roll back preparation.
- The destination selection is written before F-e64c9fe93b53 is called.
- The tracking fields and final status byte are written only after that call returns.
- No rollback or exception handling is visible.

A dependency failure, nonlocal exit, invalid memory access, or failure to return may therefore leave partial effects. The source does not define an error-reporting protocol for such cases.

No allocation, release, reference-count adjustment, or explicit ownership transfer is visible. Storing the callback-returned address does not establish whether it is borrowed, retained, or transferred. Likewise, passing descriptor and payload values to dependencies does not establish their ownership behavior.

No locking, atomic grouping, or memory-ordering mechanism is visible. The widths of accesses do not by themselves establish atomicity.

## Dependencies

| Opaque ID | Visible role and contract boundary |
|---|---|
| F-44341bd71b97 | Called before callback lookup whenever the proposal differs from the captured source selection. Receives repeated destination and auxiliary arguments. Effects and ownership behavior are unknown. |
| F-a7562a402053 | Called for observed record type `0x0565`, with payload, descriptor-derived value, and constants 6 and `0x50`. Effects are unknown. |
| F-08741ff22c05 | Called for the second type comparison matching `0x1888`, with the same argument layout as the other type-specific dependency. Effects are unknown. |
| F-e64c9fe93b53 | Called after the destination selection write. Its 32-bit result is stored in the destination-associated result field. Result meaning and ownership are unknown. |

The indirect callback has no supplied opaque dependency ID. Its identity is obtained from state offset `0xAC` after preparation. Its visible interface is one proposed-selection argument and an address-like return value, with zero causing early return.

No dependency is assumed to preserve state fields, external objects, or global state. In particular, later reads must not be replaced conceptually with earlier values merely because a dependency’s purpose appears unrelated.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless stated otherwise, they assume valid nonoverlapping storage, defined arithmetic, stable relevant fields, normally returning dependencies, and no dependency mutations that alter the described state.

1. **Advance within the limit.**  
   With source selection 2, mode 1, and limit 5, the proposal is 3 for either wrapping setting. Preparation and callback resolution occur. If resolution returns a valid nonzero record, the destination selection is eventually written as 3 and the final status byte as 1.

2. **Clamp at the upper boundary.**  
   With source selection 4, mode 1, limit 5, and wrapping zero, the proposal becomes 4. The function writes status 0 and invokes no dependency or callback.

3. **Wrap at the upper boundary.**  
   With source selection 4, mode 1, limit 5, and wrapping nonzero, the proposal becomes 0. A nonzero callback result permits the ordinary commit path.

4. **Clamp at the lower boundary.**  
   With source selection 0, mode 2, and wrapping zero, the initial proposal is −1 and the final proposal is 0. The function writes status 0. This path does not read the limit.

5. **Wrap at the lower boundary.**  
   With source selection 0, mode 2, wrapping nonzero, and limit 5, the proposal becomes 4. The limit is read on this boundary path.

6. **Different source and destination slots.**  
   If the source is clamped to its existing value, a different destination selection is not corrected or synchronized. Only the destination status byte is directly written, as zero.

7. **Resolution returns zero.**  
   If the proposal differs but the callback returns zero, preparation and callback invocation have occurred. The function then returns without directly updating the destination selection, result field, tracking fields, record field, or status byte.

8. **Unrecognized record type.**  
   A valid nonzero record with a stable type other than `0x0565` and `0x1888` receives no type-specific call. The descriptor’s `+4` value is nevertheless read, and the normal commit sequence still occurs.

9. **Zero limit is not rejected.**  
   With source selection 0, mode 1, limit 0, and wrapping zero, the proposal becomes −1 and follows the change path. With the same values and wrapping nonzero, the proposal becomes 0 and follows the no-change path.

10. **Descriptor mutation during type processing.**  
    If a type-specific dependency changes the destination descriptor field, F-e64c9fe93b53 receives the subsequently reread descriptor address. The earlier type-specific call still received the value obtained from the earlier descriptor.

## Uncertainties

The following are documented_only:

- Exact target ABI, byte order, alignment requirements, and address-space rules.
- Overflow behavior for selection increment, decrement, limit-minus-one, and indexed address arithmetic. The visible comparisons are signed, but the supplied material is insufficient to choose portable signed-overflow semantics or guaranteed modular wrapping.
- Valid ranges for source slots, destination slots, selections, and limits.
- Intended behavior for modes other than 1 and 2.
- Behavior under concurrent mutation, including differing repeated mode or type reads.
- Callback identity, record lifetime, descriptor lifetime, and all ownership obligations.
- Meanings of the auxiliary value, descriptor-derived value, dependency result, and tracking fields.
- Failure behavior inside dependencies or the callback.

These unknowns do not justify adding validation, fallback behavior, retries, rollback, or ownership operations to the observed contract.

## Evidence

Established static evidence consists solely of the supplied function body and the supplied opaque dependency mapping. It supports the access widths, offsets, branch conditions, dependency arguments, early return, and ordered direct effects stated above.

Navigation, wrapping, content processing, and status interpretations are inferred intent, not independently established domain semantics.

No runtime tests have been performed. No dependency implementation, caller contract, complete object layout, or external documentation was available for this specification.
