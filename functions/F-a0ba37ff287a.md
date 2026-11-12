<!-- Trusted provenance: F-a0ba37ff287a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a0ba37ff287a

## Purpose

F-a0ba37ff287a checks a stream-backed format, publishes a face count, and optionally initializes a selected face in caller-supplied mutable state.

The visible outcomes are:

- Propagate a nonzero status from an initial, count-related, or selected-face dependency.
- Return `2` for format rejection.
- Return `6` for an out-of-range nonnegative face index.
- Return zero for a successful count-only request.
- Complete selected-face initialization and return a local status value, subject to unresolved dependency effects on that status.

**Inferred intent:** the state represents a font face, with flags, metrics, and optional fixed-size records. This interpretation does not establish the meaning or ownership of individual fields.

**Evidence scope:** static analysis of the supplied function representation only. No runtime tests have been performed.

## Interface

### Arguments and return value

The function has four inputs:

| Input | Established use |
|---|---|
| Stream handle | Passed to dependencies; not directly dereferenced by this function |
| Mutable state pointer | Used for direct reads and writes and passed, in whole or in part, to dependencies |
| Requested face index | A 32-bit value interpreted as signed for selection decisions |
| Additional 32-bit argument | Copied into local storage, then overwritten before any visible behavior uses its incoming value |

The result is a 32-bit status. Zero is the visible success value. Nonzero results are not restricted to `2` and `6`.

### Representation and memory preconditions

All state offsets below are byte offsets from the mutable state pointer. Fields are 32 bits unless specified otherwise. Address-bearing state fields use the visible 32-bit representation.

Applicable state accesses extend through byte offset `0x1A7`. This does not establish the complete object size or the storage requirements of dependencies.

Required memory-safety preconditions include:

- Accessible state storage for every executed read and write.
- When optional allocation executes, an address at `0x68` permitting a 32-bit read at nested offset `0x1C`.
- When source records are processed, sufficient accessible records beginning at the address obtained from `0x184`, with a 36-byte stride.
- When metric scanning executes, sufficient accessible records beginning at the address obtained from `0x1A0`, with a 16-byte stride.
- Valid writable destination storage when the allocation-related status permits record processing.

The function performs no visible null, capacity, alignment, address-overflow, or allocation-size-overflow validation. Counts used for record processing and scanning are full 32-bit values, unlike the published face count.

Packed stores modify two bytes only. Byte order is not independently established; packed fields are therefore specified by byte offset, not as a portable serialized layout.

## Behavior

### 1. Initial dependencies and arithmetic update

The function calls F-e40a3ceddab2 with the stream handle and zero.

- A nonzero result is returned immediately.
- A zero result permits F-de70a5256fd2 to be called with the stream handle and an opaque global datum.
- A nonzero result from that second call is returned immediately.

After both calls return zero, the function updates the 32-bit field at `0xA8` using the then-current fields at `0xA8` and `0xCC`:

- Multiply the value at `0xCC` by `65,536`.
- Add it to the value at `0xA8`.
- Store the low 32 bits at `0xA8`.

Both operations have unsigned 32-bit wraparound. The representation establishes that both field reads precede the store; it does not require an ordering between those two reads.

The update happens before format recognition. No later rejection rolls it back. Values present at function entry are insufficient to predict this update unless the intervening dependency effects are also known.

### 2. Format recognition

Recognition requires all four conditions:

| State field | Required condition |
|---|---|
| `0x84` | Equals an opaque global comparison value |
| `0x88` | Unsigned value less than `5` |
| `0x90` | Unsigned value greater than `57` |
| `0x8C` | Equals `0x00000D0A` |

The visible comparison against `58` does not provide an independent acceptance case. When the value at `0x90` exceeds `57`, the decision instead uses the comparison at `0x8C`. Otherwise, recognition fails.

On failure:

1. Read a signed 32-bit diagnostic-level field at offset `0xE0` within an opaque global object.
2. If it exceeds `3`, call F-4457a9a44a68 with a format-rejection diagnostic.
3. Return `2`.

No face-count operation or selected-face initialization follows. Diagnostic wording is not part of this contract.

### 3. Count discovery and index decision

For a recognized format, call F-e40a3ceddab2 with the stream handle and the current field at `0x98`.

A nonzero result becomes the return status. No count is published on that path.

If the call succeeds, call F-8f49ed59b67e with the stream handle and the address of a local 32-bit status slot, whose value is zero at the call.

- If the status slot is nonzero afterward, return that status without publishing a count.
- Otherwise, retain only the low 16 bits of the dependency’s direct result.

Store the retained count, from `0` through `65,535`, at `0x00`. Only then interpret the original requested index as signed 32-bit:

- A negative index ends the visible work and returns zero.
- A nonnegative index smaller than the count proceeds to selected-face initialization.
- A nonnegative index equal to or greater than the count causes an unconditional call to F-4457a9a44a68, followed by return status `6`.

Thus every index with bit 31 set selects count-only behavior. With count zero, negative indices succeed as count-only requests; index zero is rejected.

### 4. Selected-face dependencies

For an in-range index, the function reads whether `0xCC` is nonzero into local bookkeeping. The resulting boolean is not passed as an explicit argument to any subsequent visible call.

It then calls these dependencies in order:

1. F-5c2239021c53 with the address of state offset `0xF0`, the stream handle, the original requested index, and the current field at `0x98`.
2. Only if the first call returns zero, F-18d6a2021b62 with the address of state offset `0x120`, the stream handle, the current field at `0x11C`, and the current field at `0x118`.

A nonzero result from either call is returned. Earlier state changes, including dependency changes and count publication, remain in effect.

### 5. Selected-face fields and flags

After both selected-face dependencies return zero, direct writes occur in this order:

1. Store the requested index at `0x04`.
2. Copy `0x198` to `0x10`.
3. Set flags at `0x08` to `1`.
4. Replace flags with `5` if bit `0x04` of `0x144` is clear.
5. OR flags with `0x10` if bit `0x01` of `0x144` is clear; otherwise OR with `0x20`.
6. OR flags with `0x02` if `0x17C` is nonzero.
7. OR flags with `0x40` if `0x1A4` is nonzero.
8. Copy `0x174` to `0x14`. Test `0x174` again; if zero, copy `0x170` to `0x14`.
9. Copy `0x178` to `0x18`.
10. Clear `0x1C`, then `0x20`.
11. Copy `0x134`, `0x138`, `0x13C`, and `0x140` to `0x34`, `0x38`, `0x3C`, and `0x40`, respectively.

These operations establish value copies. Their ownership consequences are unknown, including for any address-valued fields.

### 6. Initial packed metrics

The next direct stores are:

| Destination | Width | Value |
|---|---:|---|
| `0x44` | 16 bits | Low half of `0x12C` |
| `0x46` | 16 bits | Low half of `0x140` |
| `0x48` | 16 bits | Low half of `0x138` |

The values read from `0x140` and `0x138` for these stores are retained for the subsequent difference calculation.

F-d7be9cfb4000 is then called with:

- Twelve times the zero-extended low 16 bits of a read from `0x12C`.
- `10`.

The first argument is at most `786,420`; this multiplication cannot overflow 32 bits.

The dependency result is narrowed to signed 16 bits. Independently, the function subtracts the signed 16-bit interpretation of the saved `0x138` value from that of the saved `0x140` value. This full-width signed difference ranges from `−65,535` through `65,535`.

The narrowed dependency result is stored at `0x4A`. If that signed 16-bit result is less than the full difference, the low 16 bits of the difference replace it.

This is not saturating arithmetic. A positive difference exceeding `32,767` can become negative when the replacement field is interpreted as signed 16-bit.

### 7. Optional destination records

The function rereads `0x17C` and captures it as an unsigned 32-bit count. Zero skips this stage.

For a nonzero count, local bookkeeping is prepared with a null pointer and the address of the local return-status slot. F-6dd003795db7 receives four explicit arguments:

1. The 32-bit value at nested offset `0x1C` from the object addressed by `0x68`.
2. Element size `16`.
3. Zero.
4. The captured count.

The status address is not among those explicit arguments. Any implicit allocation/status convention remains **documented_only**.

The dependency’s returned address is stored at `0x20` before the local status is checked. A nonzero status is returned immediately. There is no visible null check when status is zero.

If processing continues, obtain the source address from `0x184`. For each of the captured number of records, perform these operations in order:

| Destination offset within record | Width | Source operation |
|---|---:|---|
| `0x00` | 16 bits | Read source word at `0x04`; retain low 16 bits |
| `0x02` | 16 bits | Read source word at `0x00`; retain low 16 bits |
| `0x04` | 32 bits | Read source word at `0x04`; shift left by six |
| `0x08` | 32 bits | Read source word at `0x00`; shift left by six |
| `0x0C` | 32 bits | Read source word at `0x04` again; shift left by six |

Advance source records by 36 bytes and destination records by 16 bytes. After all records are written, store the captured count at `0x1C`.

The source words are not visibly snapshotted once per record. The repeated reads matter if source and destination storage overlap. Signed left-shift behavior for negative values or overflowing results is unresolved; unrestricted mathematical multiplication by 64 is not established.

### 8. Remaining metrics and final dispatch

Reread `0x144`.

- If bit `0x04` is clear, store the low 16 bits of `0x148` at `0x4C`.
- Otherwise, obtain the scan address from `0x1A0` and the unsigned count from `0x198`. Traverse 16-byte records, comparing each signed 32-bit field at record offset `0x04` with a running maximum initialized to zero. Store the low 16 bits of the final maximum at `0x4C`.

An empty scan and a scan containing only negative values both produce zero. A positive maximum is narrowed, not saturated.

Copy the two bytes at `0x4A` to `0x4E`.

Read the 32-bit field at `0x44` once and retain its low half interpreted as signed 16-bit. Use that same captured value for:

1. F-870c4fcf7295 with second argument `0xFFFFFFF6`; store the low 16 bits of its result at `0x50`.
2. F-d7be9cfb4000 with second argument `30`; store the low 16 bits of its result at `0x52`.

Prepare local bookkeeping containing the state pointer, an opaque global datum, and `0x00010003`. Call F-e504f72ffcc9 with another opaque global datum, zero, the address of the local slot containing the state pointer, and zero.

Its direct result is ignored. No complete interpretation of the addressed bookkeeping is established.

Afterward, reread `0x1A4`. If nonzero, OR `0x40` into the then-current flags at `0x08`. If zero, perform no direct flag update.

Return the local status slot. Unknown indirect effects must not be replaced with an assumption that this status necessarily remains zero.

## Side effects

Established direct effects include:

- Incremental state mutation without rollback.
- An update to `0xA8` before format validation.
- Count publication before index validation.
- Clearing `0x1C` and `0x20` during selected-face initialization.
- Publication of the allocation-related address before checking its status.
- Packed metric writes, flag updates, and optional destination-record writes.
- Conditional format diagnostics and unconditional invalid-index diagnostics.

No explicit release operation appears. Overwriting an address does not establish that its previous storage was released, nor does copying an address establish an ownership transfer or its absence. Ownership consequences, lifetimes, and release duties are **documented_only**.

Dependencies may change stream state, globals, and reachable memory. Successful return alone does not establish preservation. In particular, reads after a dependency must use the values present then, not assumed entry values.

## Dependencies

| Opaque ID | Established role in this function |
|---|---|
| F-e40a3ceddab2 | Initial stream operation and later count-positioning operation; nonzero status propagates |
| F-de70a5256fd2 | Second initial stream operation with opaque datum; nonzero status propagates |
| F-4457a9a44a68 | Diagnostic call |
| F-8f49ed59b67e | Count-like result with explicit local status address |
| F-5c2239021c53 | First selected-face operation |
| F-18d6a2021b62 | Second selected-face operation |
| F-d7be9cfb4000 | Numeric transformation used for two packed metrics |
| F-6dd003795db7 | Returns an address subsequently used for 16-byte destination records |
| F-870c4fcf7295 | Numeric transformation used for one packed metric |
| F-e504f72ffcc9 | Final dispatch receiving an address into local bookkeeping |

These roles describe visible use, not complete callee contracts.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Initial failure.** If the first F-e40a3ceddab2 call returns `9`, return `9`. No later dependency or direct state write by this function executes. The failed dependency’s own effects remain unspecified.

2. **Wrapped arithmetic before rejection.** Immediately after both initial dependencies return successfully, and before the arithmetic update, suppose `0xA8` contains `0xFFFF0000` and `0xCC` contains `2`. The update stores `0x00010000`. If recognition then fails, return `2` without rolling back that update. Entry-state values alone do not establish this result.

3. **Recognition boundaries.** At recognition time, with the other conditions satisfied, `0x90 = 57` is rejected. Values `58` and `59` both require `0x8C = 0x00000D0A`. A value of `5` at `0x88` is rejected.

4. **Count truncation.** If F-8f49ed59b67e returns `0x12340003` and leaves its status slot zero, publish `3`. Requested index `0xFFFFFFFF` is negative and returns zero without selected-face initialization.

5. **Index boundary.** With published count `3`, index `3` causes a diagnostic and status `6`; index `2` reaches the selected-face dependencies. Count zero rejects index zero.

6. **Metric narrowing.** If the saved signed values from `0x140` and `0x138` are `32,767` and `−32,768`, the difference is `65,535`. It exceeds every possible signed 16-bit initial result, so replacement stores `0xFFFF`, which represents `−1` as signed 16-bit.

7. **Record conversion.** For nonoverlapping valid storage and a source record whose first two words remain `2` and `3` during processing, the destination receives 16-bit values `3` and `2`, then 32-bit values `192`, `128`, and `192`.

8. **Scan behavior.** When scanning is selected, an empty array or exclusively negative scanned fields produces zero at `0x4C`. A maximum of `65,536` also stores zero after 16-bit narrowing.

9. **Late flag decision.** If `0x1A4` is nonzero after final dispatch, the subsequent direct operation sets flag `0x40`. If it is zero, this function performs no late clearing or setting of that bit; dispatch may already have changed the flags.

## Uncertainties

The following remain **documented_only**:

- Exact format identity and opaque global values.
- Complete dependency contracts, including reachable-state and global mutations.
- Ownership consequences of field assignments, allocation ownership, and cleanup responsibilities.
- How F-6dd003795db7 could communicate status beyond its explicit arguments.
- Behavior when an address is null but the checked status remains zero.
- The final dispatcher’s expected bookkeeping layout and indirect status effects.
- Mathematical semantics of the numeric dependencies.
- Signed left-shift semantics for negative or overflowing source values.
- Complete object layout, byte order, and public meanings of fields and flag bits.

No unknown is resolved by assuming that a dependency preserves globals, maintains earlier field values, or owns or releases memory.

## Evidence

Established behavior derives from the supplied static representation: branch conditions, argument expressions, access widths, state offsets, store order, explicit status checks, and return paths.

The recognition predicate accounts for the overwritten intermediate comparison. Arithmetic distinguishes unsigned 32-bit wraparound, signed index comparisons, signed 16-bit interpretation, full-width differences, and narrowing stores.

Font terminology is inferred intent. Ownership, implicit conventions, dependency internals, and unobserved runtime behavior remain unresolved. No runtime execution or test result is asserted.
