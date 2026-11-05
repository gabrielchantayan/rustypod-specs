<!-- Trusted provenance: F-a0bacab96f08; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a0bacab96f08

## Purpose

F-a0bacab96f08 appends a one-, two-, or three-byte representation of an unsigned integer through a caller-provided output cursor.

**Established behavior:** The integer selects the number of byte stores: one for values below `0x80`, two for values from `0x80` through `0x7FF`, and three for all remaining 32-bit values. Each byte store is preceded by an update to the cursor slot. The two- and three-byte paths subsequently invoke an unresolved indirect control-transfer target.

**Inferred intent:** The byte construction resembles UTF-8 encoding for values below `0x10000`. This is not a validated Unicode encoder: surrogate values are accepted, values above `0xFFFF` still use only three bytes, and no four-byte representation is produced.

The contract distinguishes the observable encoding prefix from the unresolved continuation. Completion of the byte stores does not establish the final state or successful return of a multibyte invocation.

## Interface

### Inputs

The function accepts two arguments:

1. **Cursor-slot address.** A pointer to a four-byte memory slot whose contents are interpreted as the current byte-output address.
2. **Input value.** An unsigned 32-bit integer.

The supplied representation uses a four-byte cursor slot and a 32-bit unsigned input. This contract follows that recovered width model; it does not assert compatibility with a wider-pointer ABI.

The initial cursor value is read from the slot before any branch-specific output occurs. The input integer is passed by value. Reassignments to its working representation do not constitute writes to caller-owned integer storage.

### Outputs

There is no explicit return value.

The directly observable results are:

- Writes to the cursor slot.
- Writes to byte-addressed output memory.
- On every path whose input exceeds `0x7F`, an invocation of an unresolved indirect target after the byte stores.

With ordinary, nonoverlapping output storage, and before any effects of that target, the cursor advances by exactly the number of bytes emitted.

### Preconditions for ordinary buffer behavior

The following conditions are needed for the straightforward append behavior described below:

- The cursor-slot address permits the observed four-byte read and subsequent four-byte writes.
- The initial cursor points to writable output storage.
- The output storage has capacity for one, two, or three bytes, as selected by the input.
- Output bytes do not overlap the cursor slot.
- The required pointer increments are meaningful and representable in the target execution model.

No checks establish these conditions. The interface supplies no capacity, end pointer, allocator, error-output parameter, or encoding-mode selector.

For multibyte inputs, returning normally additionally depends on the unresolved target being callable and returning. Its validity and behavior are not established.

## Behavior

### Value classes and byte construction

The comparisons selecting the output length are unsigned comparisons against the full 32-bit input. They are not comparisons against a truncated byte.

For nonoverlapping storage, the emitted bytes are:

| Input range | Byte count | Bytes, in output order |
|---|---:|---|
| `0x00000000`–`0x0000007F` | 1 | The input’s low eight bits |
| `0x00000080`–`0x000007FF` | 2 | `0xC0` combined with input bits 6–10; then `0x80` combined with input bits 0–5 |
| `0x00000800`–`0xFFFFFFFF` | 3 | `0xE0` combined with input bits 12–16; then `0x80` combined with input bits 6–11; then `0x80` combined with input bits 0–5 |

“Combined” in this table means bitwise OR, with the selected input bits placed in the low positions of the byte.

For the two-byte range, the first byte lies between `0xC2` and `0xDF`. The last byte lies between `0x80` and `0xBF`.

For the three-byte range, the first byte lies between `0xE0` and `0xFF`. The other two bytes each lie between `0x80` and `0xBF`.

The final byte is derived from the original input’s low six bits, including on the two-byte path where the working integer is modified before the first byte is stored.

### Single-byte path

For an input no greater than `0x7F`, the function:

1. Reads the initial output address from the cursor slot.
2. Writes the address one byte beyond that output address into the cursor slot.
3. Stores the input’s low byte at the original output address.
4. Returns without invoking the unresolved target.

The cursor update precedes the byte store. This ordering is part of the observed behavior, not an atomic append operation.

The value zero produces a single zero byte. It does not cause a terminator to be added after that byte.

### Two-byte path

For inputs from `0x80` through `0x7FF`, the function:

1. Reads the initial output address from the cursor slot.
2. Advances the slot by one byte.
3. Stores the first encoded byte at the initial output address.
4. Reads the current output address back from the cursor slot.
5. Advances the slot by one byte relative to that reloaded address.
6. Stores the final encoded byte at the reloaded address.
7. Invokes the unresolved indirect target.

If that target returns normally, the displayed function then returns.

The second output address comes from an actual reread of the cursor slot. It is not necessarily the initial address plus one when the output overlaps the slot or another agent changes the slot.

### Three-byte path

For every input at least `0x800`, the function:

1. Reads the initial output address from the cursor slot.
2. Advances the slot by one byte.
3. Stores the first encoded byte at the initial output address.
4. Rereads the output address from the slot.
5. Advances the slot by one byte relative to that address.
6. Stores the middle encoded byte at that address.
7. Rereads the output address from the slot again.
8. Advances the slot by one byte relative to that address.
9. Stores the final encoded byte at that address.
10. Invokes the unresolved indirect target.

If that target returns normally, the displayed function then returns.

Both later output addresses depend on cursor-slot rereads. With nonoverlapping, exclusively accessed storage, these addresses are consecutive. Without those conditions, a later store may be redirected.

### Width-sensitive behavior

The middle byte on the three-byte path selects input bits 6–11. The displayed shift construction performs a 32-bit left shift followed by a logical right shift; bits shifted beyond the 32-bit width are discarded. It must not be interpreted as an unlimited-width arithmetic expression.

For the first byte on that path, the right-shifted input is truncated to eight bits before being combined with `0xE0`. Since `0xE0` forces the high three output bits to one, only input bits 12–16 can affect that byte.

Consequently, within the three-byte path, all three emitted bytes depend only on the input’s low 17 bits. Higher input bits affect neither byte construction nor the already-selected three-byte length.

This does not imply that reducing every input modulo `0x20000` preserves the complete behavior. Such reduction can change the length-selection branch.

### Validation and errors

There is no visible rejection of:

- Unicode surrogate values.
- Values greater than `0x10FFFF`.
- Any other 32-bit unsigned value.
- Null or invalid pointers.
- Insufficient output capacity.
- Overlap between output storage and the cursor slot.

No error code or explicit rollback mechanism is present.

If a byte write fails after its preceding cursor update, the cursor may already have advanced. If a later access or the indirect target fails, earlier writes have already occurred. The operation is not transactional.

## Side effects

### Memory access order

Each direct output byte store is preceded by a cursor-slot write. On multibyte paths, a cursor-slot read separates successive output-byte stores.

This matters in two cases:

- **Aliasing:** An output byte can overwrite part of the cursor slot. A subsequent reread then uses the modified slot contents. Even on the single-byte path, the output store can overwrite part of the cursor update.
- **Concurrent or externally changing storage:** Later stores follow the reloaded addresses, not a cursor value cached for the entire call. No synchronization or concurrency guarantee is visible.

The precise consequences of overlapping storage depend on address layout and target byte order. The ordinary “advance by byte count” postcondition therefore requires nonoverlap.

### Ownership and lifetime

The visible body neither allocates nor explicitly releases memory. It directly uses caller-supplied storage.

That observation does not establish ownership behavior for the entire multibyte invocation. The unresolved target might access, retain, modify, or release memory; its contract is unavailable.

### Other state

No explicit named global read or write appears in the displayed encoding body. This does not establish that the unresolved target preserves globals, the cursor slot, output bytes, or any other process state.

For a multibyte input, the emitted-byte and cursor facts describe the state immediately before the unresolved invocation. They are final postconditions only if that target leaves the relevant state unchanged.

## Dependencies

### F-unknown-01 — unresolved indirect target

This is an opaque documentation identifier for the unresolved call site, not a recovered identity or evidence that a single fixed function is always selected.

Established properties:

- It is reached for every input greater than `0x7F`.
- It is not reached for inputs at most `0x7F`.
- It is invoked after all two or three output bytes have been stored.
- No explicit arguments or consumed return value are shown.
- The displayed function returns if the target returns normally.

Unknown properties, classified as **documented_only**:

- Target identity and selection mechanism.
- Actual calling convention and argument requirements.
- Access to registers, globals, cursor storage, or output memory.
- Ownership and lifetime effects.
- Whether it returns, fails, transfers control elsewhere, or terminates execution.

The visible empty argument list is insufficient evidence for a complete no-argument ABI contract. The unresolved transfer must not be omitted when describing multibyte behavior.

## Acceptance examples

These are conditional acceptance examples derived from static evidence. They are not runtime test results.

Assume a valid four-byte cursor slot, sufficient writable output storage, no overlap, no concurrent mutation, and representable cursor increments. For multibyte inputs, the listed state is the state immediately before F-unknown-01 is invoked.

| Input | Output bytes, hexadecimal | Cursor advance | F-unknown-01 invoked |
|---|---|---:|---|
| `0x00000000` | `00` | 1 | No |
| `0x0000007F` | `7F` | 1 | No |
| `0x00000080` | `C2 80` | 2 | Yes |
| `0x000007FF` | `DF BF` | 2 | Yes |
| `0x00000800` | `E0 A0 80` | 3 | Yes |
| `0x0000D800` | `ED A0 80` | 3 | Yes |
| `0x0000FFFF` | `EF BF BF` | 3 | Yes |
| `0x00010000` | `F0 80 80` | 3 | Yes |
| `0x00020000` | `E0 80 80` | 3 | Yes |
| `0xFFFFFFFF` | `FF BF BF` | 3 | Yes |

Additional conditional acceptance criteria:

- For input `0x7F`, the cursor-slot write occurs before the `7F` byte store, and no unresolved invocation occurs.
- For input `0x80`, both byte stores complete before the unresolved invocation. If that target changes the cursor afterward, the final cursor need not equal the initial cursor plus two.
- Input `0xD800` is not rejected, despite being a Unicode surrogate value.
- Input `0x10000` emits three bytes, not a standard four-byte UTF-8 sequence.
- Input `0x20000` does not reduce to the single-byte behavior of zero. Its full-width value selects the three-byte path.
- If an output byte overlaps the cursor slot, contiguous output and a final advance equal to the byte count are not acceptance requirements. The specified store order and cursor rereads remain the relevant behavior.
- Insufficient capacity has no specified graceful-failure result. Earlier cursor and byte writes are not rolled back by any visible mechanism.

## Uncertainties

The following remain **documented_only**, rather than being filled in by assumptions:

- The identity and complete semantics of F-unknown-01.
- Whether the recovered indirect invocation accurately represents the original control-transfer mechanism.
- Target byte order, alignment requirements, and invalid-access behavior.
- Pointer-overflow behavior at address-space boundaries.
- Whether callers guarantee valid Unicode input, sufficient capacity, or nonoverlap.
- Whether the routine belongs to a larger validated encoding workflow.
- Any final-state guarantee after the unresolved target executes.

The UTF-8 interpretation is inferred intent only. For values through `0xFFFF`, excluding surrogate values, the bytes match the corresponding one-, two-, or three-byte UTF-8 construction. This resemblance does not establish validation, full Unicode support, or a standard encoding API.

## Evidence

This contract is based solely on the supplied static function representation.

Directly established observations are:

- A four-byte cursor-slot interface and an unsigned input interpreted in the recovered 32-bit model.
- Unsigned length-selection thresholds at `0x80` and `0x800`.
- Byte truncation, masks, shifts, and fixed leading-bit patterns.
- Cursor-slot updates before each output-byte store.
- Cursor-slot rereads before later multibyte stores.
- A direct return on the single-byte path.
- An unresolved indirect invocation after both multibyte paths.
- No visible capacity check, scalar-value validation, allocation, explicit release, or error result.

The byte table and acceptance examples are arithmetic consequences of those observations. They do not establish the unresolved target’s effects or actual execution outcomes.

No runtime tests have been performed.
