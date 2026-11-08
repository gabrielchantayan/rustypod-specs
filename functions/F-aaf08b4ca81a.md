<!-- Trusted provenance: F-aaf08b4ca81a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aaf08b4ca81a

## Purpose

F-aaf08b4ca81a conditionally advances or retreats a signed position stored in a caller-supplied state object, checks that position against a signed extent calculation, and records boundary conditions in a flags word.

The function has two active modes, selected by the lowest two flag bits. Other modes return without changing the object. A separate flag disables processing on entry. Boundary handling depends on the active mode and on a reversal-related flag.

The return value is not a general success, movement, or validity indicator: an invocation that sets the disabling flag still returns `1`. Only an invocation that finds that flag already set on entry returns `0`.

The movement and boundary rules below are established by the supplied static representation. Descriptions such as “position,” “stride,” “extent,” and “reversal” are explanatory labels, not independently established domain meanings.

## Interface

### Input and object layout

The sole input is an address-like value identifying a state object. The function accesses the following fields relative to that address:

| Byte offset | Width | Interpretation |
|---|---:|---|
| `0x164` | 32 bits | Signed position |
| `0x168` | 32 bits | Signed stride |
| `0x16C` | 32 bits | Signed extent |
| `0x170` | 32 bits | Unsigned flags |

The supplied representation uses 32-bit integer fields and four-byte stores. It does not establish byte order, alignment requirements, or the complete object layout.

The flags have these observed roles:

| Mask | Contract label | Observed use |
|---|---|---|
| `0x00000003` | Mode | Values `1` and `2` select active processing |
| `0x00000004` | Reversal selector | Changes direction selection and boundary correction |
| `0x00010000` | Upper/direction flag | Selects retreat in some cases; set after an upper-bound violation |
| `0x00020000` | Lower flag | Set after a negative position; participates in one stopping decision |
| `0x00040000` | Disabled flag | Causes an immediate `0` return when already set |

These labels do not imply that each flag always describes the current position accurately. Existing boundary flags are not comprehensively recomputed or cleared.

### Output

The return value is a 32-bit unsigned integer restricted, in the visible control flow, to:

- `0` when the entry flags contain the disabled flag.
- `1` in every other case that returns normally.

The object may be modified before returning `1`. Depending on the path, the function writes the position, flags, or both, and may invoke F-870c4fcf7295.

### Preconditions

The caller must provide storage permitting every read and write reached by the selected path. A disabled invocation or an inactive-mode invocation accesses only the flags field; active processing can access all four listed fields.

No explicit null check, bounds check, alignment check, or arithmetic-error handling is visible. Validity of the object and defined execution of the signed arithmetic are caller/environment obligations.

## Behavior

### Entry decisions

The function first reads the flags word and derives the mode and reversal selector from that entry snapshot.

1. If the disabled flag is set, it returns `0`. No other field is read, no field is written, and no dependency is called.
2. If the mode is neither `1` nor `2`, it returns `1` without reading the position, stride, or extent and without writing the object.
3. Otherwise, the entry flags determine whether to advance, retreat, or disable further processing.

The movement decision is:

| Mode | Reversal selector | Relevant entry flags | Action |
|---|---|---|---|
| `1` | Clear | Any | Advance by one |
| `1` | Set | Upper/direction clear | Advance by one |
| `1` | Set | Upper/direction set | Retreat by one |
| `2` | Clear | Upper/direction clear | Advance by one |
| `2` | Clear | Upper/direction set | Set disabled flag; return `1` |
| `2` | Set | Upper/direction set | Retreat by one |
| `2` | Set | Upper/direction clear, lower clear | Advance by one |
| `2` | Set | Upper/direction clear, lower set | Set disabled flag; return `1` |

In either immediate-disabling case, the function writes the entry flags with `0x00040000` added and then returns `1`. It does not read or change the position, stride, or extent.

In particular, when mode `2` and the reversal selector are both active, the upper/direction flag takes precedence over the lower flag. If both are set, the function retreats rather than immediately disabling.

### Position update and cached bounds

For a movement path, the observable field-access order is:

1. Read stride.
2. Read extent.
3. Read position.
4. Add the selected signed increment, either `+1` or `−1`.
5. Write the resulting position.

The stride and extent values read here are retained for subsequent calculations. The function does not refresh them after the dependency call described below.

The negative-position check applies to the newly calculated position.

### Negative-position handling

If the new position is nonnegative, this stage performs no additional write.

If the new position is negative and the entry reversal selector is clear:

- The function calls F-870c4fcf7295.
- Its first argument is the saved extent minus the saved stride.
- Its second argument is the saved stride.
- The dependency’s 32-bit result is written into the position field.
- The function then reads the current flags and writes them back with the lower flag set.

No local contract establishes what value the dependency returns or whether that value is nonnegative.

If the new position is negative and the entry reversal selector is set:

- In mode `1`, the position is replaced with signed `1`.
- In mode `2`, the position is replaced with signed `0`.
- The function writes flags equal to the **entry snapshot** with the upper/direction flag cleared.
- It then reads the current flags and writes them with the lower flag set.

Thus, on this path, the initial flags replacement is not a merge against the flags currently in memory. Under ordinary nonconcurrent execution without external mutation, the combined effect is to clear upper/direction, set lower, and preserve the other entry bits.

The function does not return immediately after negative-position correction. It continues to the upper-bound check.

### Upper-bound check

After any negative-position handling, the function reads the position field again. Using that read and the previously saved stride and extent, it checks whether:

**stride × position + stride ≤ extent**

The comparison is signed in the supplied representation.

Equality is accepted. If the expression is less than or equal to the extent, the function returns `1` without further writes.

The evidence does not establish that stride is positive or nonzero. Consequently, this expression must not be replaced by an index-range test involving division, nor treated as an ordinary monotonic upper bound for all possible inputs.

If the expression exceeds the extent, the correction depends on the entry mode and reversal selector:

| Entry reversal selector | Entry mode | Position correction |
|---|---|---|
| Clear | `1` | Write `0` |
| Clear | `2` | Write the most recently read position minus `1` |
| Set | Either active mode | Write the most recently read position minus `2` |

For the reversal-selected correction, the function first reads the current flags and writes them with the lower flag cleared, and only then writes the corrected position.

For all three upper-bound corrections, after the position write the function reads the current flags, writes them with the upper/direction flag set, and returns `1`.

No second upper-bound check or negative-position check follows the correction. A corrected position can therefore remain outside an intuitive range.

### Arithmetic constraints

The observed data widths are 32 bits. Position updates, the dependency’s first argument, the upper-bound multiplication and addition, and corrective subtraction are expressed using signed values.

The supplied representation does not resolve signed-overflow semantics. This contract establishes the numerical results above when all relevant signed intermediate results are representable. It does not assume saturation, checked arithmetic, or modulo-\(2^{32}\) wrapping outside that range.

The dependency result is stored as a 32-bit value and subsequently read as a signed position. Its interpretation as a useful index is not established.

## Side effects

### Object writes

Only the position and flags fields are explicitly written. Stride and extent are only read by this function.

Observable write sequences include:

- **Immediate disabling:** one flags write.
- **Movement without boundary handling:** one position write.
- **Negative position without reversal:** initial position write, dependency call, replacement position write, flags read/write setting lower.
- **Negative position with reversal:** initial position write, replacement position write, flags write clearing upper/direction from the entry snapshot, flags read/write setting lower.
- **Upper-bound correction without reversal:** corrected position write, flags read/write setting upper/direction.
- **Upper-bound correction with reversal:** flags read/write clearing lower, corrected position write, flags read/write setting upper/direction.

Negative-position handling and upper-bound correction can both occur during one invocation, so their sequences can compose.

Existing flags are not generally normalized. For example, an accepted position can retain an old upper/direction flag, and an upper-bound correction without reversal does not clear the lower flag.

### Ownership and external state

The function performs no explicit allocation, deallocation, ownership transfer, or retention of the input object.

This observation does not constrain F-870c4fcf7295. Its effects on global state, memory, ownership, callbacks, or execution flow are unknown. Although it receives two integers rather than the object address, it cannot be assumed incapable of reaching the object through other state.

The visible implementation provides no synchronization or atomic multi-field update. Intermediate position and flags states are written separately.

### Errors and abnormal completion

No explicit error result exists for invalid memory, arithmetic overflow, or dependency failure. The `0` return is a disabled-state observation, not an established error code.

If the dependency does not return normally, the later replacement-position and flags writes are not reached. The initial movement write has already occurred before the call.

## Dependencies

### F-870c4fcf7295

This is the sole visible external dependency.

**Invocation condition:** active movement produces a negative position while the entry reversal selector is clear.

**Arguments, in order:**

1. Saved signed extent minus saved signed stride.
2. Saved signed stride.

**Result use:** the returned 32-bit value is written directly to the position field. The function subsequently rereads that field and applies the signed upper-bound calculation.

**Not established:**

- The mathematical operation performed.
- Handling of zero or negative arguments.
- Whether the result is signed in the dependency’s own interface.
- Whether the result is always a valid position.
- Memory ownership or global-state behavior.
- Whether the dependency can fail or terminate execution abnormally.

An interpretation such as selecting a replacement position is inferred intent only. No particular selection rule is part of this contract.

## Acceptance examples

These examples assume valid accessible storage, no concurrent mutation, and representable signed arithmetic. Unmentioned flags are clear.

### Disabled on entry

Given flags `0x00040001`, the function returns `0`.

Position, stride, and extent are not read. No writes or dependency calls occur.

### Inactive mode

Given flags `0x00000003`, the function returns `1` without modifying the object or reading the other listed fields.

### Ordinary advancement at equality

Given mode `1`, reversal clear, position `1`, stride `4`, and extent `12`:

- Position becomes `2`.
- The bound expression is `4 × 2 + 4 = 12`.
- Equality is accepted.
- Flags remain unchanged and the function returns `1`.

### Upper-bound reset in mode 1

Given mode `1`, reversal clear, position `2`, stride `4`, and extent `12`:

- The initial position write stores `3`.
- The bound expression is `16`, exceeding `12`.
- Position is then reset to `0`.
- Flags become `0x00010001`.
- The return value is `1`.

The intermediate position write is part of the observable sequence.

### Mode 2 disables without movement

Given flags `0x00010002`, the function writes `0x00050002` and returns `1`, without reading the position or bounds.

A subsequent invocation with those unchanged flags returns `0`.

### Reversal retreat takes precedence over lower

Given flags `0x00030006`, position `2`, stride `4`, and extent `12`:

- Both boundary flags are initially set.
- The function retreats to position `1`, rather than disabling.
- The bound expression is `8`, so no correction occurs.
- Flags remain `0x00030006`.
- The return value is `1`.

### Negative retreat in reversal mode 2

Given flags `0x00010006`, position `0`, stride `4`, and extent `12`:

- The initial position write stores `−1`.
- Negative-position handling replaces it with `0`.
- Upper/direction is cleared and lower is set.
- The bound expression is `4`, which is accepted.
- Final flags are `0x00020006`; the return value is `1`.

With those flags unchanged, the next invocation sets disabled without moving.

### Conditional dependency result

Given mode `1`, reversal clear, position `−2`, stride `4`, and extent `20`:

- The initial position write stores `−1`.
- F-870c4fcf7295 is called with arguments `16` and `4`.

If the dependency returns `2` and does not otherwise alter the object, position becomes `2`, lower is set, and the bound expression is `12`, which is accepted. Final flags are `0x00020001`, and the function returns `1`.

This example is conditional; a result of `2` is not an asserted dependency behavior.

### Zero stride does not prevent a negative correction

Given mode `1`, reversal set, position `0`, stride `0`, and extent `−1`:

- Movement first produces position `1`.
- The bound expression is `0`, which exceeds `−1`.
- Reversal correction subtracts `2`, producing position `−1`.
- Upper/direction is set and lower is cleared.
- The function returns `1` without rechecking the negative result.

## Uncertainties

The following are **documented_only**:

- Domain meaning of the object, position, stride, extent, modes, and flags.
- Full object size, byte order, alignment, and pointer validity requirements.
- Signed-overflow behavior and address-calculation overflow behavior.
- The implementation and side effects of F-870c4fcf7295.
- Whether callers constrain stride, extent, or position to a narrower domain.
- Whether external synchronization prevents concurrent modification.
- Application-level meaning of the return value beyond the observed `0`/`1` distinction.

Inferred intent is a position-stepping mechanism with optional reversal and terminal stopping. That interpretation does not replace the explicit transition rules, especially for zero or negative strides, unusual extents, stale boundary flags, or dependency-produced positions.

## Evidence

This contract is based solely on static inspection of the supplied representation of F-aaf08b4ca81a and the supplied opaque dependency mapping.

Established observations include the field offsets and widths, entry-flag decisions, read/write ordering, dependency invocation arguments, signed comparisons, boundary corrections, and explicit return values.

No dependency implementation, caller implementation, runtime trace, or execution result was supplied. No runtime tests have been performed.
