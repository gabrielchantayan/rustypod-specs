<!-- Trusted provenance: F-a3fca5b1d989; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a3fca5b1d989

## Purpose

F-a3fca5b1d989 conditionally performs a drawing-like operation over a rectangular region, optionally emits seven header items, and processes a fixed sequence of 42 cells when an associated object has a nonzero anchor value.

The routine temporarily changes flags and style-related state, calls rendering-like dependencies, and restores selected saved values on normal completion. It does **not** establish a general transaction or restore all state it changes.

The seven-item header and 42-cell sequence suggest a calendar-style presentation. That interpretation is inferred intent, not an established interface requirement. The contract below uses neutral terms such as “header,” “cell,” “anchor,” and “associated object.”

## Interface

### Arguments and result

The routine receives three arguments:

1. A pointer to a mutable object, called the **owner** below.
2. An opaque 32-bit value, forwarded to the per-cell dependency.
3. An opaque 32-bit value, passed to the initial region dependency.

There is no explicit return value. Effects are communicated through memory writes and dependency calls.

The supplied representation uses 32-bit integers, pointers, object fields, and dispatch-table entries. Byte-sized state is also accessed. Arithmetic interpretation must retain these widths rather than silently widening to arbitrary-precision integers.

### Referenced owner layout

Offsets below are byte offsets from the owner:

| Offset | Access and role |
|---|---|
| `0x00` | Dispatch-table pointer. |
| `0x48` | 32-bit entry-gating flags. |
| `0x80`–`0x8C` | Four 32-bit region components, copied at entry after the flag gate. |
| `0xE8` | Selection value, compared against the current cell identifier. |
| `0xEC` | Pointer to the associated object, repeatedly reread. |
| `0xF0` | Header-enable value; any nonzero value enables seven header iterations. |
| `0xF8` | Vertical offset used in header placement. |
| `0x100` | Starting value supplied to the header conversion dependency. |
| `0x104` | Address passed as header style/configuration. |
| `0x108`, `0x10C`, `0x110` | Addresses of owner-resident cell configuration fields. |
| `0x114`, `0x118`, `0x11C` | Values supplied to the cell-style dependency. |
| `0x120`, `0x124` | Values supplied to two associated-object configuration calls. |

Passing an address into the owner is observably different from copying that field’s value. In particular, the per-cell state pointer is assigned an address within the owner.

### Referenced associated-object layout

| Offset | Access and role |
|---|---|
| `0x00` | Dispatch-table pointer. |
| `0x48` | 32-bit flags, including bit `0x40` and selection-style bits. |
| `0x50`–`0x7F` | A 48-byte block copied for selected-cell state handling. |
| `0x80`–`0x8C` | Four saved 32-bit values later supplied to a restoration-like virtual operation. |
| `0xE8` | Nonzero anchor enabling cell processing. |
| `0xF0` | Current cell identifier, written once per iteration. |
| `0xF4` | Pointer to an owner-resident cell configuration field. |
| `0x1B8`, `0x1BC`, `0x274` | Addresses passed to the first configuration dependency call. |
| `0x278`, `0x27C`, `0x334` | Addresses passed to the second configuration dependency call. |

The owner’s virtual operation at dispatch offset `0x5C` returns another object used for flag and byte-state access. It is called repeatedly; the routine does not cache its result for the entire operation or establish that every call returns the same object.

### Preconditions

For a normally completing invocation:

- Every directly accessed object, dispatch table, field, and callback target must be valid when accessed.
- Dependencies must satisfy their own unknown argument and lifetime requirements.
- Stack-backed arguments must remain usable for the duration of the calls receiving them.
- Arithmetic inputs must lie within the executable’s supported behavior. The supplied representation does not independently resolve signed-overflow behavior.
- Callbacks must not invalidate objects or storage that subsequent accesses require.

There are no explicit null checks, allocation-failure checks, or general error-status checks.

## Behavior

### 1. Entry gate and region gate

The first observable access is the owner’s 32-bit flag field at `0x48`.

The routine proceeds only when masking that field with `0x1800` produces exactly `0x0800`. Otherwise, it returns without making the later dependency calls or writes described here.

After that gate:

1. Copy the owner’s four region words at `0x80`–`0x8C` into temporary storage.
2. Preserve the owner pointer and the two opaque arguments in temporary state.
3. Call F-e57b05520b65 with output storage for four 32-bit region components, the third argument, and the copied region.

Call the resulting components **left**, **top**, **right**, and **bottom** for purposes of this contract. These names describe their subsequent arithmetic use, not an independently established coordinate convention.

Further work requires both signed comparisons to hold:

- bottom is greater than top;
- right is greater than left.

Equal or reversed bounds return without subsequent work. The comparison logic accounts for signed subtraction overflow when implementing the comparisons; it is not an unsigned area check.

The initial region dependency has already executed on this rejection path, so rejection is not necessarily free of dependency side effects.

### 2. Establish drawing state

For an accepted region, the following operations occur in order:

1. Call F-de68e1374066 with the owner and the resulting region.
2. Subtract the second component of the copied owner region from its fourth component and call F-870c4fcf7295 with that difference and `7`.
3. Retain the 64-bit result of that call as two 32-bit components, called **Q** and **R** below.
4. Call F-960363c07b32 to populate a temporary pair of 32-bit values.
5. Initialize a temporary context through F-a4c26c2b4842.
6. Pass the owner and that context to F-7fa874cbac7a.
7. Read the current associated-object pointer and save its four words at `0x80`–`0x8C`.

The routine then invokes the owner’s virtual operation at `0x5C` three separate times:

- From the first returned object, save the 32-bit flags at `0x48`.
- From the second returned object, save the byte at `0x101`.
- In the third returned object, write byte value `1` at `0x101`.

Only bit `0x40` of the saved flag word is subsequently restored. The saved byte is subsequently restored as a Boolean, not necessarily as its original byte value.

### 3. Optional seven-item header

If the owner field at `0xF0` is nonzero:

1. Call F-706044cdbb36 with the owner, temporary context, and an opaque static-data argument.
2. Call F-33ca0546dba1 with the context and `0x21`.
3. Execute seven iterations, with an index from zero through six.

Each iteration performs these operations in order:

- Call F-9d0ecc5f6adf.
- Pass its result and `0x80` to F-682adfd5a561.
- Copy three 32-bit words from an opaque static-data block into a temporary 12-byte descriptor.
- Call F-d7be9cfb4000 with the owner’s current field at `0x100` plus the index, and divisor-like argument `7`.
- Replace only the low byte of the descriptor’s third word with a secondary one-byte result from that call.
- Invoke the obtained object’s virtual operation at `0x84`, passing that object, temporary item storage, `0x10`, `3`, and the descriptor.
- Call F-b38e44b3c7fb with the context.
- Read the owner’s current vertical-offset field at `0xF8` and combine it with the first value produced by F-960363c07b32.
- Call F-a0576cc39408 with the context and temporary item.
- Call F-e5e685741bac to position the context.
- Call F-ccb5d9751852 with the context and the address of owner field `0x104`.
- Call F-a8201782fe69 with the context and temporary item.

The horizontal placement combines:

- Q multiplied by the iteration index;
- the second value produced by F-960363c07b32;
- signed Q divided by two;
- subtraction of half the F-a0576cc39408 result;
- addition of signed R divided by two.

The vertical placement subtracts the F-b38e44b3c7fb result from the first saved positioning value plus the current owner vertical offset.

The visible signed divisions by two truncate toward zero. The supplied data does not establish that F-870c4fcf7295 returns a quotient and remainder, despite that being a plausible interpretation.

### 4. Optional 42-cell sequence

After the header branch, the routine reads the associated object’s field at `0xE8`. Zero skips all cell processing.

For a nonzero anchor:

1. Call F-1948d97e66b3 on the anchor and save its result as the cell-range limit.
2. Call F-d06bac1b9d44 on the initially read anchor.
3. Reread the associated-object pointer and its anchor, then call F-0a9a547c6b8c on that value.
4. Establish the initial relative position as one minus the last result.
5. Establish the initial cell identifier as the initially read anchor minus the F-d06bac1b9d44 result, plus the initial relative position.
6. Call F-fbe12ad10a3a using the associated object, addresses at `0x274`, `0x1BC`, and `0x1B8`, and owner value `0x124`.
7. Reread the associated object and call F-fbe12ad10a3a using addresses at `0x334`, `0x27C`, and `0x278`, and owner value `0x120`.

There are then exactly 42 iterations if every dependency returns normally. Both the relative position and cell identifier advance by one after each iteration.

Each iteration follows this order:

1. Call F-5ed372440147 with the owner, relative position, and temporary four-byte output storage.
2. Reread the associated object and write the current cell identifier at `0xF0`.
3. Compare the owner’s current selection at `0xE8` with that identifier.
4. If equal, save associated-object flags and 48 bytes at `0x50`; construct a temporary state descriptor whose flag word is the saved flags masked by `0xFFFC7FFF`, then combined with `0x00020000`; submit it through the associated object’s virtual operation at `0xDC`.
5. Obtain an object through the owner’s virtual operation at `0x5C` and clear bit `0x40` in its current flags at `0x48`.
6. Invoke the associated object’s virtual operation at `0xD8`.
7. Obtain an object again through owner dispatch offset `0x5C`. If its current flags contain bit `0x40`, invoke the associated object’s virtual operation at `0x88`.
8. Determine whether the relative position is strictly positive and no greater than the saved range limit.
9. Select cell style and configuration as described below.
10. Call F-f83eaa5eed8f again with the current associated object and cell identifier.
11. Call F-5318fae1e7d6 with the owner, original second argument, accepted region, the latest classification result, and the in-range Boolean.
12. Reread the owner selection. If it equals the cell identifier, submit a descriptor containing the saved selection flags and saved 48-byte block through dispatch offset `0xDC`.
13. Advance to the next iteration.

Cell style selection is:

| Condition | F-1bc3bedca6a9 argument from owner | Address stored at associated offset `0xF4` |
|---|---|---|
| Outside the saved range | Value at `0x114` | Owner address `0x10C` |
| In range, F-f83eaa5eed8f returns zero | Value at `0x118` | Owner address `0x108` |
| In range, F-f83eaa5eed8f returns nonzero | Value at `0x11C` | Owner address `0x110` |

For in-range cells, classification therefore occurs twice. The later result is not required to match the earlier one. Out-of-range cells still receive the later classification call.

### 5. Normal completion

After any cell processing:

1. Invoke the current associated object’s virtual operation at `0x68`, supplying the saved four-word block and zero.
2. Obtain an object through owner dispatch offset `0x5C`.
3. Restore only bit `0x40` of that object’s current flag word to the saved entry state, preserving its other current bits.
4. Obtain an object again through owner dispatch offset `0x5C`.
5. Write zero to its byte at `0x101` if the saved byte was zero; otherwise write one.
6. Finalize the temporary context through F-282eebd7c413.
7. Return.

## Side effects

Established direct effects include:

- Setting and later Boolean-restoring a byte at offset `0x101`.
- Clearing and later restoring bit `0x40` through repeatedly obtained objects.
- Writing the associated object’s current cell identifier.
- Assigning the associated object a pointer into owner storage.
- Passing temporary selection descriptors to virtual callbacks.
- Invoking all listed dependencies in the stated branch-dependent order.

No explicit allocation or release is visible. This does not establish that dependencies allocate nothing or release nothing.

The final current-cell and configuration-pointer fields are not explicitly restored. The four-word restoration-like callback does not prove restoration of every field or of the original object: the associated-object pointer is reread.

There is no visible cleanup mechanism for nonlocal exits or dependency failures that prevent normal return.

## Dependencies

Mapped opaque dependencies have the roles described in Behavior. Their implementations, ownership rules, global-state effects, and failure behavior are unknown.

Additional dispatch dependencies are identified by table offsets `0x5C`, `0x68`, `0x84`, `0x88`, `0xD8`, and `0xDC`. No opaque function IDs were supplied for those dynamic targets.

Two unmapped external operations perform apparent fixed-size copying and clearing for temporary descriptors: copies use 48 bytes, and descriptor clearing uses 92 bytes. Their exact external contracts are undocumented here.

Static arguments include a header-configuration value and a three-word descriptor template. Their contents are unavailable.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

- **Entry rejection:** If the owner flags masked by `0x1800` are not `0x0800`, no region dependency or later callback is invoked.
- **Empty region:** If the region dependency produces equal horizontal bounds, or equal vertical bounds, no context initialization or header/cell processing occurs.
- **Header only:** If the region passes, header enable is nonzero, and the later anchor read is zero, seven header iterations occur and no cell iterations occur. Normal completion still performs restoration-like calls and context finalization.
- **Cells without header:** If header enable is zero and the anchor is nonzero, no header-item construction occurs, but 42 cell iterations occur.
- **Range boundary:** A relative position of zero is outside the range. A relative position equal to a positive saved limit is inside; one greater than that limit is outside.
- **Nonpositive range limit:** No relative position satisfies both range comparisons when the saved limit is nonpositive.
- **Classification changes:** If an in-range cell’s first classification is zero and its second is nonzero, style selection follows the first result while F-5318fae1e7d6 receives the second.
- **Noncanonical saved byte:** A saved byte of `2` is restored as `1`, not `2`.
- **Flag preservation:** Normal completion restores the saved value of bit `0x40` while retaining other bits present in the final target’s current flag word.
- **Selection mutation:** If a callback changes the owner selection during a cell iteration, the post-processing selection test uses the changed value rather than the earlier comparison result.

## Uncertainties

The following are **documented_only**:

- Calendar semantics, coordinate orientation, text encoding, style meaning, and anchor units.
- Dependency return conventions beyond their visible uses, including the two halves of F-870c4fcf7295 and the secondary byte from F-d7be9cfb4000.
- Pointer retention, allocation, release, and global mutation by dependencies.
- Stability of associated-object pointers and dispatch-returned objects across calls.
- Overflow behavior of signed additions, subtractions, and multiplications outside supported input ranges.
- Exact implicit receiver conventions for virtual calls whose rendered argument lists omit a receiver.
- Runtime safety of reconstructed temporary-storage extents. The 48-byte copies must not be judged solely from individual apparent scratch-array sizes.
- Restoration behavior when selection changes during callbacks. The final selection test is independent of the initial test; it can consume saved state from another iteration or state not established by the current iteration. No stable-selection guarantee is visible.
- Whether restoration-like virtual operations actually restore their apparent target state.

## Evidence

This contract is derived solely from the supplied static routine representation and opaque dependency mapping.

Established evidence covers direct field accesses, masks, constants, branch conditions, visible arithmetic, argument forwarding, iteration counts, and call order. Rendering and calendar terminology are explicitly inferential.

No dependency implementations, runtime observations, or runtime tests were provided or performed.
