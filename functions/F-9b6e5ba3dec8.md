<!-- Trusted provenance: F-9b6e5ba3dec8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9b6e5ba3dec8

## Purpose

F-9b6e5ba3dec8 conditionally processes a flag-bearing request against a receiver and its embedded context. For an enabled receiver, it can normalize flags, update shared mode bytes, invoke preliminary processing, select collection positions, dispatch entry callbacks, perform mode-dependent follow-up, and clear a receiver-relative byte.

These operations and their ordering are established by the supplied static representation. “Request,” “mode,” “collection,” and “entry” describe inferred roles, not established domain semantics.

The receiver’s enable byte gates all substantive processing. A disabled receiver causes no dependency calls or memory writes by this function. Capturing the shared-state pointer still occurs before that gate.

## Interface

### Arguments and result

The function accepts four arguments:

| Argument | Established treatment |
|---|---|
| Receiver base | Address-like value used for receiver-relative accesses and to form an embedded-context address |
| Original flags | Unsigned 32-bit bit field |
| Token | Pointer-like value used as an opaque argument, compared with distinguished values, and conditionally interpreted as a signed displacement |
| Fourth argument | Incoming value has no established observable use; relevant subsequent uses follow assignments that replace it |

There is no explicit return value or recoverable status result.

The original flags and token remain available for the final clearing condition, independently of effective-flag normalization and indirect callback-target selection.

### Receiver-relative layout

| Offset | Access or role |
|---|---|
| `+0x04` | Address passed as the embedded context; this is an address calculation, not a load of a context pointer from that offset |
| `+0xD8` | One-byte enable field |
| `+0xE6` | One-byte field conditionally read during token-`2` normalization |
| `+0xE4 + token` | Conditional one-byte clearing destination |

No complete receiver layout or minimum allocation size is established beyond the storage needed for the accesses actually reached.

### Shared-state layout

A global pointer identifies byte-addressable state:

- Byte `0` supplies the current mode-like value.
- Byte `1` receives a recorded mode-like value.

The function captures the global pointer at entry. The initial new-mode write uses this captured pointer. Other shared-state accesses obtain their address through the global binding again.

In particular, follow-up processing reads byte `0` through the then-current global pointer. Earlier dependencies or callbacks may have changed either the binding or the pointed-to bytes. An earlier callback writing mode `2` is therefore insufficient, by itself, to determine the follow-up branch.

### Widths and preconditions

Flags, masks, and the position accumulator use 32-bit integer operations in the supplied representation. Candidate positions are limited to `0` through `12`, so all position-bit shifts are within that word width.

The final token comparison uses a signed integer interpretation. Under the represented 32-bit integer model, token bit patterns with the sign bit set compare as negative and satisfy the upper-bound check.

Pointer-like values are converted to or represented through integer-sized values. A complete ABI and independent pointer-width confirmation are unavailable. Address arithmetic should not be treated as checked, widened, or portable merely because its intended offset can be described mathematically.

Preconditions for normal execution include valid storage for every reached access, suitable alignment for pointer-like loads, live entry and dispatch objects, and callable targets. The function does not comprehensively validate these conditions.

## Behavior

### Enable gate and normalization

The function first captures the shared-state pointer and reads the receiver byte at `+0xD8`.

If that byte is zero, processing ends. There is no shared-byte dereference, dependency invocation, entry callback, or final clearing write by this function.

For an enabled receiver:

- Effective flags initially equal the original flags.
- Selection restriction initially reflects whether either original bit `0x04` or `0x08` is set.
- The secondary mask starts at zero.
- No mode transition has yet been recorded.

When original bit `0x08` is set and the token is exactly `2`, the function reads receiver byte `+0xE6`. If that byte is zero, it clears effective bit `0x08` and recomputes selection restriction from original bit `0x04` alone.

All other flag bits remain unchanged. A token other than `2` does not cause this normalization read. Neither normalization nor later callback dispatch changes which original flags and token govern the final byte clear.

A candidate-mask seed is then computed: `1` if effective flags intersect `0x188`, otherwise `0`. This seed is computed before the optional mode transition and preliminary dependency call. It is not recomputed from shared or receiver memory afterward.

### Optional mode transition

Mode processing occurs only when the entire effective-flags word equals `1`. Having bit `0x01` set alongside another bit does not qualify.

The function reads the current mode through the current global pointer and determines a proposed byte:

| Token identity | Proposed mode |
|---|---:|
| Exactly `5` | `3` |
| Exactly `6` | `4` |
| Exactly `7` | `1` |
| A fourth distinguished symbolic token | `2` |
| Any other token | The mode just read |

The numeric identity of the fourth distinguished token is not established.

If the proposed and read values are equal, this phase performs no shared-byte write and records no transition.

If they differ, the function performs these actions in order:

1. Writes the old mode to byte `1` through the current global binding.
2. Writes the proposed mode to byte `0` through the pointer captured at function entry.
3. Retains an invocation-local indication that a transition occurred.

No opaque call occurs between the entry-time pointer capture and these initial mode writes. Without concurrent rebinding, the accesses therefore concern the same shared-state object. The representation establishes no synchronization guarantee for concurrent changes.

### Preliminary dependency

F-f726434d18e4 is called with the receiver base, effective flags, and original token if either selection restriction is active or this invocation recorded a mode transition.

Otherwise, the call is skipped.

Its side effects are unknown. Subsequent behavior uses the effective flags, restriction indicator, candidate seed, and transition indicator already computed; it does not recompute them from receiver or shared-state memory.

### Initial candidate-mask accumulation

For every enabled invocation that reaches this phase, F-62d1a359629f is called with the embedded-context address and each selector in this exact order:

1. `0`
2. `1`
3. `2`

Each returned 32-bit word is ORed into the candidate mask, starting from the seed described above. All three calls occur, even if the seed or an earlier result already makes the mask nonzero.

If the resulting mask is zero, secondary-mask selection and entry dispatch are skipped. Mode-transition follow-up and the final clearing condition remain reachable.

### Secondary-mask selection

For a nonzero accumulated candidate mask, effective flags determine one of three paths:

- **Bit `0x04` set:** F-62d1a359629f receives the embedded context and original token. Its result becomes the secondary mask. The candidate mask remains unchanged.
- **Bit `0x04` clear, bit `0x08` set:** F-9d8c91d0e283 receives the embedded context and original token. Its result is ORed with `1`; that value replaces the candidate mask and also becomes the secondary mask.
- **Neither bit set:** No additional mask-producing call occurs. The secondary mask remains zero.

Bit `0x04` takes precedence when both bits are present. The `0x08`-only path replaces the accumulated mask rather than adding to it.

### Ordered entry dispatch

Positions `0` through `12` are considered in ascending order. Candidate bits above position `12` never produce entry processing.

For each position whose candidate bit is set:

1. F-7c8a9c6c9d7f is called with the embedded context.
2. Its returned word must intersect the effective flags.
3. If selection restriction is active, the position’s secondary-mask bit must also be set.
4. Only if both applicable conditions pass is F-9266831caef7 called with the embedded context and position.
5. A nonnull returned entry supplies a dispatch-structure address from its first pointer-like word.
6. The callable target is read at offset `0x0C` within that dispatch structure.
7. The target is invoked with the entry, effective flags, and original token.

The lookup position passes through a one-byte character conversion. Every possible position here, `0` through `12`, survives that conversion unchanged.

F-7c8a9c6c9d7f is called even when the secondary mask will reject the position. Its result is obtained separately for every candidate; it is not cached across positions.

Earlier calls may change receiver state, entries, dispatch targets, shared bytes, or the global shared-state binding. Later eligibility queries, lookups, and callbacks consequently operate after those effects. The function does not restart selection or recompute its retained masks in response.

If entry lookup returns null, F-a817d8d58c10 is invoked without arguments. The supplied representation classifies that call as nonreturning. There is no recoverable lookup-error result or skip path. If it unexpectedly returns, the subsequent entry access has no valid null-handling branch.

### Mode-transition follow-up

This phase runs only if this invocation recorded an actual initial mode transition. It occurs after normal completion of entry dispatch, including cases where no entries were dispatched.

The function:

1. Reads mode byte `0` through the then-current global pointer.
2. Writes that freshly read byte to byte `1` through the global binding.
3. Uses the freshly read value to decide whether to perform the temporary-region calls.

The decisive value is this follow-up read, not the initially installed mode or any earlier callback’s write. Intervening eligibility queries, entry lookups, callbacks, or other dependencies may change the value or rebind the pointer before the read.

If the freshly read byte is `1` or `2`, these calls occur in order:

1. F-bd3e4559b3e7 receives a 12-byte temporary region.
2. F-799bc49cafe5 receives a separate 8-byte temporary region and integer `100`.
3. F-98a2c6875ccd receives both region addresses, with the 12-byte region first.
4. F-3e5f2a992ae5 receives the 12-byte region.

Once this condition is satisfied, changes made by those calls do not cause the function to recheck the mode between them.

F-66941fb7ed7c then receives the embedded context, regardless of whether the four temporary-region calls were selected.

The temporary regions have automatic storage duration. No explicit initialization by this function precedes their first dependency calls. Their formats, initialization obligations, ownership semantics, and permissible retention are unknown.

### Final byte clear

After normal completion of preceding phases, the function checks original bit `0x08`.

If that bit is set and the original token’s signed integer interpretation is less than `3`, one zero byte is written at receiver base plus `0xE4` plus the token displacement.

For nonnegative tokens:

| Token | Clearing offset |
|---|---|
| `0` | `+0xE4` |
| `1` | `+0xE5` |
| `2` | `+0xE6` |

There is no lower-bound check. For example, signed token `−1` satisfies the comparison and denotes offset `+0xE3` when the address calculation is valid.

This condition uses original flags even if normalization removed effective bit `0x08`. It is not reached for a disabled receiver or after a dependency that does not return.

Overflowing address arithmetic and invalid destinations have no established safe behavior.

## Side effects

Directly established effects are:

- Receiver-byte reads and the conditional one-byte clear.
- Shared-mode reads and ordered byte writes.
- Ordered calls to opaque dependencies.
- Zero or more indirect entry callbacks.
- Exposure of temporary-region addresses to dependencies.

No direct heap allocation, explicit release, or ownership transfer appears in this function. That observation provides no allocation, release, retention, or ownership guarantee for its dependencies or callbacks.

The function has no visible rollback mechanism. Initial mode writes and earlier callbacks can remain effective if a later operation fails or does not return.

No synchronization, reentrancy guard, comprehensive pointer validation, or exception-recovery protocol is established.

## Dependencies

| Opaque ID | Established interface and use |
|---|---|
| F-f726434d18e4 | Receiver, effective flags, token; preliminary processing |
| F-62d1a359629f | Embedded context and selector or token; returns mask words |
| F-9d8c91d0e283 | Embedded context and token; returns the `0x08`-path replacement-mask basis |
| F-7c8a9c6c9d7f | Embedded context; returns a word intersected with effective flags |
| F-9266831caef7 | Embedded context and position; returns an entry pointer |
| F-a817d8d58c10 | No arguments; null-entry failure path represented as nonreturning |
| F-bd3e4559b3e7 | Address of the 12-byte temporary region |
| F-799bc49cafe5 | Address of the 8-byte temporary region and integer `100` |
| F-98a2c6875ccd | Addresses of the 12-byte and 8-byte regions, in that order |
| F-3e5f2a992ae5 | Address of the 12-byte region |
| F-66941fb7ed7c | Embedded context; final mode-transition follow-up |

Indirect callbacks have no supplied opaque identity. Their visible arguments are the selected entry, effective flags, and original token.

For every dependency and callback, mutation, global preservation, allocation, release, ownership, and retention behavior remain unspecified unless directly constrained by the calling sequence above.

## Acceptance examples

These are conditional consequences of static control flow, not executed tests. Unless stated otherwise, reached storage is valid and invoked operations return normally.

1. **Disabled receiver.** Enable byte zero prevents all dependency calls and byte writes by this function, regardless of flags or token. Entry-time capture of the global pointer does not dereference shared bytes.

2. **Normalized token `2`.** Original flags `0x08`, token `2`, and receiver byte `+0xE6` equal to zero produce effective flags zero and inactive selection restriction. Preliminary processing is skipped. The three initial mask queries still occur. No entry passes the effective-flags intersection. The final clear still writes zero to `+0xE6`.

3. **Mode change without intervening shared-state mutation.** Effective flags exactly `1`, token `5`, and initial mode `4` cause old-mode byte `1` to receive `4`, then current-mode byte `0` to receive `3`, before preliminary processing. If later operations preserve the relevant value and binding, follow-up reads and records `3`, skips the four temporary-region calls, and invokes F-66941fb7ed7c.

4. **Unchanged mode.** Effective flags exactly `1`, token `5`, and current mode `3` cause no initial mode write, no preliminary call driven by a transition, and no transition follow-up. The three initial mask queries still occur.

5. **Full-word mode condition.** Effective flags `0x05` do not enter mode processing, despite containing bit `0x01`.

6. **Mask precedence.** Effective flags `0x0C` and a nonzero accumulated candidate mask select the token-based F-62d1a359629f call. F-9d8c91d0e283 is not called, and the candidate mask is retained.

7. **Candidate replacement.** With effective flags `0x08`, a token unaffected by normalization, and a F-9d8c91d0e283 result of `0x20`, the replacement candidate and secondary masks are `0x21`. Only positions `0` and `5` are candidates, each subject to a fresh eligibility query.

8. **Fresh follow-up value is decisive.** Suppose this invocation recorded a transition and a callback wrote mode `2`. The four temporary-region calls are guaranteed by that scenario only if the fresh follow-up read through the then-current global pointer yields `2`. A sufficient additional assumption is that no intervening operation changes that value or binding. Under that condition, follow-up records `2`, performs the four calls in order, and invokes F-66941fb7ed7c. Without that condition, the earlier callback write alone does not determine the branch.

9. **High candidate bits only.** If the seed is zero, initial mask results combine to `0x80000000`, and neither secondary-mask path applies, no position is processed and no per-position eligibility query occurs.

10. **Negative displacement.** Original bit `0x08` and signed token `−1` satisfy the final comparison. If execution reaches the tail and address arithmetic and storage are valid, byte `+0xE3` is cleared.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of flags, modes, selectors, tokens, and collection positions.
- Numeric identity of the fourth distinguished mode token.
- Complete ABI, independently confirmed pointer width, and overflow behavior.
- Dependency and callback side effects, global rebinding, ownership, release, and retention rules.
- Entry and dispatch-object lifetime guarantees.
- Temporary-region formats and the meaning or units of `100`.
- Exact failure mechanism and independently confirmed nonreturn behavior of F-a817d8d58c10.
- Concurrent access, synchronization, and reentrant execution semantics.

Inferred intent is limited to request processing, mode management, and collection notification. The temporary-region sequence does not independently establish a delay, resource lifecycle, or ownership transfer.

## Evidence

This contract derives from the supplied static function representation and opaque dependency mapping. Established evidence covers branch conditions, integer masks, access widths and offsets, call order and arguments, dispatch layout, and distinctions between original and effective flags.

The follow-up condition is supported by a fresh shared-byte read after entry processing. No preservation guarantee connects an earlier callback’s mode write to that read.

Dependency implementations, an independently verified ABI, and executable observations were not supplied. No runtime tests have been performed. Acceptance examples are static expectations under explicit conditions, not runtime observations.
