<!-- Trusted provenance: F-e21ff2d11b9b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e21ff2d11b9b

## Purpose

F-e21ff2d11b9b makes one bounded transfer attempt from a mutable state object to a caller-supplied destination. If its initial availability calculation is not signed-positive, it first calls F-d1a8963fd3c6. Every result from that dependency except exactly −1 contributes one destination byte. A subsequent transfer call handles the selected additional length.

The function does not repeatedly obtain data until the request is satisfied. It does not reject zero or negative requests, and a zero request does not necessarily prevent destination writes.

“Availability,” “obtain one byte,” and “transfer” describe inferred roles. Established behavior consists of field reads, arithmetic, dependency calls, destination writes, state updates, and return construction. The transfer dependency’s implementation is not supplied.

## Interface

The function accepts four arguments and returns a signed 32-bit integer.

| Argument | Representation | Observable role |
|---|---|---|
| Destination | Byte-addressed pointer | Receives a dependency-produced byte when applicable; otherwise, or subsequently, is supplied to the transfer dependency. |
| Requested length | Signed 32-bit integer | Bounds the additional transfer after subtracting the zero or one byte already contributed. |
| State | Pointer to a mutable, 32-bit-word-addressed object | Provides flags, an availability value, and address-related fields; receives final bookkeeping writes. |
| Auxiliary value | Opaque 32-bit value | Passed unchanged as both the fourth and fifth arguments to F-d1a8963fd3c6. |

The supplied representation uses 32-bit integer fields and stores address-bearing values in 32-bit words. Its address arithmetic requires a compatible target model.

### Directly accessed state layout

| Byte offset | Width | Use |
|---|---:|---|
| 0 | 32 bits | Availability in count mode; conditionally overwritten after the transfer call. |
| 4 | 32 bits | Transfer-source address; arithmetic operand in alternate mode; advanced after the transfer call. |
| 12 | 32 bits | Flags controlling availability calculation and the final count write. |
| 44 | 32 bits | Alternate-mode boundary operand; sometimes supplied as the third argument to F-d1a8963fd3c6. |

Direct accesses require an accessible extent through byte 47. This is not evidence that 48 bytes satisfy either dependency’s complete state requirements.

Two external 32-bit values act as a mask and a comparison value. Their values, initialization, and broader meanings are unknown.

### Preconditions and ownership

Ordinary bounded-transfer use requires valid state storage, sufficient destination capacity, a suitable source range, and compliance with both dependencies’ contracts. Alignment and overlap requirements are not supplied.

A positive requested length is necessary for an ordinary request-bounded interpretation when the one-byte path can occur. The function does not enforce that condition.

No direct allocation, release, or ownership transfer is visible. Neither dependency may be assumed to preserve ownership or avoid memory-management effects.

## Behavior

### External reads and initial mode selection

The function first captures the external mask, then captures the external comparison value. It next reads the state flags at offset 12.

Those entry captures are **not** used for the initial alternate-mode comparison:

- If flag `0x01000000` is clear, the function reads offset 0 as raw availability. No additional external reads are required by this branch.
- If that flag is set, the initial condition reads the external mask and external comparison value again and compares the masked state flags with that reread comparison value.
- The supplied expression does not establish an evaluation order between those two external rereads. Neither “mask reread first” nor “comparison reread first” is part of this contract.
- If the initial comparison fails, raw availability is zero.
- If it matches, the function performs the alternate-mode calculation described below.

The state flags used for this initial comparison come from the preceding flag read; the comparison does not itself require another flag read.

Equality between the entry captures and the values used in the initial comparison requires a stability assumption. Without that assumption, the initial comparison may select a different mode from a later comparison using the captures.

### Alternate-mode availability and prepared argument

The alternate calculation uses the address-bearing field at offset 4 and the boundary field at offset 44.

With stable fields and a compatible 32-bit address-ordering model, its resulting raw availability is the address at offset 4 minus the greater of the two field values, modulo \(2^{32}\). Consequently:

- An address equal to or above the boundary produces zero.
- An address below the boundary produces the modular representation of a negative difference, rather than a conventional positive distance to the boundary.

Flag `0x00001000` introduces an additional comparison path. During the **initial** calculation, this path reads the two fields as pointer values and prepares the offset-44 value as the third argument to F-d1a8963fd3c6. If the selected greater pointer is no greater than the original offset-4 pointer, the calculation produces zero without entering the fallback arithmetic.

Otherwise, the calculation falls through and rereads fields for the unsigned arithmetic. With stable fields and consistent pointer and integer ordering, this produces the same availability described above.

When this special flag path is not entered, the prepared third argument remains the state pointer. In particular, setting `0x00001000` alone does not change the argument: the initial alternate-mode comparison must also match.

The simplified arithmetic assumes stable field values during the calculation. Some paths read offset 44 more than once, and the special path can reread both operands through its fallback. No coherent snapshot or concurrent-access protocol is established.

### Initial signed test

The selected 32-bit availability is interpreted as signed for a strict greater-than-zero test.

Only a signed-positive value skips F-d1a8963fd3c6. Zero and values represented as negative take the dependency path. Under the represented 32-bit signed interpretation, raw availability `0x80000000` is therefore not a directly usable positive count.

There is no request-length check before this decision.

### Obtain-one-byte dependency

When initial availability is not signed-positive, F-d1a8963fd3c6 is invoked once with:

1. The state pointer.
2. Zero.
3. The prepared state pointer or offset-44 address.
4. The auxiliary value.
5. The same auxiliary value again.

The result is compared with exactly −1.

If it equals −1, F-e21ff2d11b9b immediately returns zero. It performs no direct destination write, transfer invocation, or final state bookkeeping on that path. Dependency effects already performed remain in effect.

Every other result is treated as a one-byte contribution. Other negative results are not rejected.

### Availability recomputation after success

Before writing the contributed byte, the function rereads state flags and recomputes availability.

This recomputation differs from the initial calculation in its treatment of the external values:

- Count mode again obtains availability from offset 0.
- Alternate mode compares the newly read flags using the **entry-captured** mask and comparison value.
- It does not reread the external values for this comparison.

Consequently, F-d1a8963fd3c6 may change the external values without changing the captured operands used here. Its state changes can still affect the recomputation.

The special `0x00001000` path in this recomputation compares unsigned 32-bit field values rather than the pointer-typed values used in the corresponding initial path. Under stable fields and a compatible address model, the same simplified availability formula applies.

The recomputed availability is not subjected to another signed-positive test.

After recomputation, the function writes the low-byte contribution of the dependency result at the original destination, advances the destination used for the transfer call by one byte, and records one byte already contributed.

Availability calculation therefore precedes this destination write. If destination aliases state, the write can affect subsequent source-address reads and final bookkeeping.

### Additional transfer length

Without a successful one-byte contribution, the number already contributed is zero.

The additional transfer length is the unsigned 32-bit minimum of:

- The selected or recomputed raw availability.
- The requested length minus the number already contributed, converted to unsigned 32 bits.

For nonnegative requests with a representable subtraction, this bounds the additional transfer by the remaining request. It does not undo an earlier one-byte write.

Negative requests are not rejected. Their converted remaining length can be a large unsigned value.

Subtracting one from the minimum signed 32-bit requested length exceeds the signed range of the supplied expression. The evidence does not establish portable behavior for that case; modular subtraction must not be assumed for it.

### Transfer call

Except for the exact −1 return path, the function invokes the unidentified transfer dependency with:

1. The original destination or its one-byte-advanced address.
2. The address read from state offset 4 at this stage.
3. The selected unsigned 32-bit transfer length.

The call occurs even when the selected length is zero.

A byte-copy role is inferred from the arguments and bookkeeping. Exact copying behavior, permitted overlap, zero-length pointer requirements, failure behavior, and additional side effects remain unknown.

No dependency result is inspected. Subsequent bookkeeping assumes only that control returns normally, not that a transfer succeeded.

### Final state and result

After the transfer call returns normally:

1. Offset 4 is read again.
2. The selected transfer length is added to that current value, with a 32-bit result stored back at offset 4.
3. Flags at offset 12 are read again.
4. If `0x01000000` is clear, offset 0 is overwritten with the previously calculated raw availability minus the selected transfer length, modulo \(2^{32}\).
5. Otherwise, offset 0 is not directly written.

The address increment uses the field value present **after** the dependency returns, not necessarily the address supplied to it.

The count subtraction uses the earlier availability calculation, not a fresh offset-0 read. A dependency’s count mutation may therefore be overwritten. The final flag read determines whether that overwrite occurs.

The result is the zero or one byte already contributed plus the unsigned transfer length, converted to the signed return type. For ordinary positive requests it stays within the requested bound. Outside that domain, an out-of-range unsigned-to-signed conversion is target-dependent in the supplied representation.

## Side effects

The established sequencing is:

1. Capture external mask.
2. Capture external comparison value.
3. Read initial state flags.
4. If initial alternate mode is selected, reread both external values for its comparison, without a specified order between those rereads.
5. Complete initial availability calculation.
6. If availability is not signed-positive, invoke F-d1a8963fd3c6.
7. On exactly −1, return zero without further direct effects.
8. Otherwise, on that path, recompute availability from state using the captured external operands, then write one destination byte.
9. Select the additional length and invoke the transfer dependency, including for zero length.
10. Read, advance, and write the current offset-4 address.
11. Reread flags and conditionally overwrite offset 0.
12. Return the constructed count.

This describes ordering established by the supplied representation, not a hardware access trace or an atomic transaction. It does not impose an order on operands where the representation supplies none.

Dependency effects may be interleaved at their calls. Destination aliasing can modify state before the transfer-source read; transfer effects can modify state before final address and flag reads.

No synchronization, rollback, fault recovery, or concurrency guarantee is visible. External changes during the initial comparison need not yield a coherent mask/comparison pair.

## Dependencies

### F-d1a8963fd3c6

Established call-site contract:

- Called at most once, only for initial signed-nonpositive availability.
- Receives the five arguments specified above.
- Exactly −1 causes an immediate zero return.
- Every other return supplies a one-byte contribution.
- State is reread after success.
- Recomputed alternate-mode comparison uses the entry-captured external values, not fresh external reads.

Its broader argument requirements, blocking behavior, errors, state and global mutations, ownership effects, and meanings of non-byte return values are documented_only.

### Unidentified transfer dependency

No opaque function ID is supplied for this dependency; its identity is documented_only.

Established behavior is a three-argument call with destination, current source address, and unsigned length. The caller ignores its result and performs bookkeeping if it returns normally.

Copy semantics, overlap support, pointer validation, zero-length behavior, ownership effects, and failure signaling are documented_only.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed results. Unless otherwise specified, assume valid memory, stable fields and external values during each calculation, a normally returning byte-copy transfer dependency, and no dependency mutation of state.

### Buffered request

With count mode active, availability five, and requested length three:

- F-d1a8963fd3c6 is skipped.
- The transfer length is three.
- The source address advances by three.
- Availability becomes two.
- The result is three.

With availability two and request five, only two bytes are transferred, availability becomes zero, and the result is two. No further obtain-one-byte call occurs.

### Exact failure sentinel

With initial availability zero and F-d1a8963fd3c6 returning −1, the result is zero. No direct destination write, transfer call, or final bookkeeping occurs. Any dependency effects remain.

A different negative return does not take this failure path.

### Successful one-byte contribution

With initial availability zero, request four, and a dependency result whose low byte is `0x41`, suppose recomputation finds count-mode availability two.

The first destination byte is `0x41`; two additional bytes are transferred after it. The source address advances by two, availability becomes zero, and the result is three.

### Zero request

With positive initial availability and request zero, the one-byte dependency is skipped, but the transfer dependency is called with length zero. Under the no-mutation assumption, state values remain unchanged and the result is zero.

With initial availability zero and a successful dependency result, one byte is written despite request zero. The remaining request converts from −1 to `0xffffffff`, so the additional transfer length equals the recomputed raw availability. This is not a safe no-op.

### Initial rereads versus entry captures

Suppose the entry captures are mask `0x00000001` and comparison zero. Initial flags are `0x01000001`. Before the initial alternate-mode comparison reads the external values, suppose the comparison value becomes one and remains stable through those rereads.

The initial comparison matches using the reread values. If the offset-4 address equals the boundary, availability is zero and F-d1a8963fd3c6 is called.

If that dependency succeeds and leaves the flags unchanged, recomputation compares against the captured comparison zero. That comparison fails and recomputed availability is zero.

This distinguishes the initial rereads from the later captured operands. It does not prescribe the order of the two initial rereads.

### Alternate arithmetic

In matching alternate mode, address `0x1000` and boundary `0x1004` produce raw availability `0xfffffffc` under stable operands.

At entry, its signed interpretation triggers F-d1a8963fd3c6. If recomputed after success, it participates as a large unsigned transfer bound without another positivity check.

### Dependency-controlled bookkeeping

If the transfer dependency changes offset 4, the selected length is added to that changed value.

If it changes final flags to count mode, offset 0 is overwritten using the earlier calculated availability minus transfer length. If it changes final flags to alternate mode, that count write is skipped.

## Uncertainties

The following remain documented_only:

- External-value meanings, initialization, stability, and synchronization.
- Full state layout and invariants.
- Complete dependency contracts and transfer-dependency identity.
- Ownership, allocation, release, and error-reporting effects inside dependencies.
- Valid pointer ranges, alignment, and permitted aliasing.
- Supported target pointer ordering and pointer-to-integer conversion behavior.
- Behavior under unsupported concurrent mutation.
- Caller restrictions on zero and negative requests.
- Exceptional signed arithmetic and out-of-range signed return conversion.
- Whether the alternate subtraction expresses intended semantics.

These unknowns do not justify replacing the visible arithmetic with a conventional positive distance or treating initial external rereads as entry snapshots.

## Evidence

This contract derives from the supplied static representation of F-e21ff2d11b9b and the supplied mapping for F-d1a8963fd3c6.

Established evidence includes field offsets, 32-bit arithmetic roles, entry external captures, conditional initial external rereads, snapshot-based post-dependency comparison, dependency arguments, the exact −1 check, destination-write sequencing, final state updates, and return construction.

Transfer terminology and the interpretation of the state as buffered data are inferred intent, not independently established dependency behavior.

No implementation access, dependency-body inspection, runtime tests, or empirical validation has been performed.
