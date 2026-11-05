<!-- Trusted provenance: F-288efaeb11be; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-288efaeb11be

## Purpose

F-288efaeb11be partially initializes caller-supplied destination storage using a source selector and two captured four-byte values.

The established operation has three parts:

- Write a fixed marker and copy the source selector into the destination header.
- Initialize a destination area differently for selectors `0x00`, `0x02`, and `0x03`, leaving that area untouched for other selectors.
- Duplicate each captured value into two destination fields and clear four other fields.

This is not a complete destination initialization. Several gaps, and some selector-dependent bytes, retain their previous contents.

Describing the destination as a structured object is inferred intent. The application meanings of its fields, the repeated values, and the conditional constants are not established.

## Interface

The function accepts three arguments and returns no value.

| Argument | Established role |
|---|---|
| First | A four-byte value unused by the supplied body. |
| Second | Source base address, carried by an integer-typed argument in the supplied representation. |
| Third | Destination base address, used for byte-addressed writes. |

All offsets below are byte offsets from the relevant base. Widths describe accessed storage, not application-level types.

### Source storage

| Offset | Width | Role |
|---|---:|---|
| `0x08` | 1 byte | Selector; read twice after the destination marker is written. |
| `0x0C` | 4 bytes | First captured value. |
| `0x10` | 4 bytes | Second captured value. |

The highest explicitly accessed source byte is at offset `0x13`. The four-byte values are opaque: the function does not compare them, perform arithmetic on them, or narrow them before writing their copies.

### Destination storage

| Offset | Width | Value written |
|---|---:|---|
| `0x00` | 1 byte | Value from the first selector read. |
| `0x01` | 1 byte | Fixed marker `0x03`. |
| `0x04` | 4 bytes | Zero. |
| `0x08` | 4 bytes | Zero. |
| `0x0C` | 4 bytes | Second captured value. |
| `0x10` | 4 bytes | First captured value. |
| `0x14` | 4 bytes | Second captured value. |
| `0x18` | 4 bytes | First captured value. |
| `0x24` | 4 bytes | Zero. |
| `0x28` | 4 bytes | Zero. |
| `0x30` | Conditional | Selector-dependent initialization described below. |

There is no length argument or explicit capacity check. The caller must provide accessible storage for every access made by the selected execution, with any alignment required by the target.

For valid, nonwrapping address calculations, a destination region covering offsets `0x00` through `0x33` accommodates all explicit destination accesses for every selector. The highest unconditional destination write ends at `0x2B`. Selector `0x00` extends the highest written byte to `0x32`; selector `0x02` to `0x31`; selector `0x03` to `0x33`.

These extents do not imply that intervening gaps are written.

## Behavior

### Capture and header ordering

Both four-byte source values are captured before any destination write. The value at source offset `0x0C` is captured first, followed by the value at source offset `0x10`.

The destination header is then updated in this order:

1. Destination offset `0x01` receives the one-byte marker `0x03`.
2. Source offset `0x08` is read, and that byte is written to destination offset `0x00`.
3. Source offset `0x08` is read again to select the conditional initialization.

Thus the header copy and the branch decision use two distinct source reads. Neither selector read precedes the marker write.

The two captured values have snapshot behavior relative to this function’s later writes. If those writes overwrite either original source field through aliasing, the destination copies still use the values captured at entry. The selector is not captured at entry: its first observation follows the marker write.

### Conditional initialization

The second selector read determines which writes occur at destination offset `0x30`.

| Selector | Conditional writes, in order | Bytes left untouched within `0x30`–`0x33` |
|---|---|---|
| `0x00` | One byte `0x10` at `0x30`; one byte zero at `0x31`; one byte `0xFF` at `0x32`. | `0x33` |
| `0x02` | Two-byte integer value `0x001F` at `0x30`. | `0x32`–`0x33` |
| `0x03` | Four-byte zero at `0x30`. | None |
| Any other byte | No conditional write. | `0x30`–`0x33` |

The selector-zero case consists of three ordered one-byte writes, not one four-byte replacement.

For selector `0x02`, the established result is a two-byte value of `0x001F`. Its byte representation depends on target byte order: `1F 00` on a little-endian target and `00 1F` on a big-endian target. Neither byte order is established by the supplied body.

Only the three listed selector values receive special handling. Signed interpretation of the intermediate selector does not alter these matches. All other byte patterns take the no-conditional-write case; there is no explicit rejection or error result for them.

### Final field updates

After the conditional initialization, the remaining writes occur in the following order:

| Order | Destination offset | Width | Value |
|---:|---|---:|---|
| 1 | `0x10` | 4 bytes | First captured value. |
| 2 | `0x0C` | 4 bytes | Second captured value. |
| 3 | `0x18` | 4 bytes | First captured value. |
| 4 | `0x14` | 4 bytes | Second captured value. |
| 5 | `0x28` | 4 bytes | Zero. |
| 6 | `0x24` | 4 bytes | Zero. |
| 7 | `0x04` | 4 bytes | Zero. |
| 8 | `0x08` | 4 bytes | Zero. |

The function then returns without a result value.

This ordering is the memory-operation order expressed by the supplied body. It is not an assertion of machine-level atomicity, compiler emission order under all conditions, or visibility guarantees to concurrent observers.

### Untouched destination storage

The function performs no explicit writes to these destination ranges:

- `0x02`–`0x03`.
- `0x1C`–`0x23`.
- `0x2C`–`0x2F`.
- The selector-dependent untouched bytes listed above.
- Bytes beyond the applicable write extent.

Under ordinary valid memory access and without external mutation, these bytes retain their previous contents. Source/destination overlap does not create additional writes beyond the specified destination accesses.

A consumer must not treat the entire destination region as initialized merely because the function returned.

### Widths and arithmetic

All address displacements are fixed byte offsets. There is no size-derived iteration, multiplication, length calculation, saturation, or arithmetic transformation of the captured values.

The explicit write widths are one, two, or four bytes:

- The marker, copied selector, and selector-zero conditional values use one-byte writes.
- `0x001F` uses a two-byte write.
- Captured-value copies and unconditional clearing fields use four-byte writes.
- Selector `0x03` also uses a four-byte clearing write.

The constants fit their respective widths. No bounds check or address-overflow handling is present. This contract does not assign defined wrapping behavior to invalid address calculations.

## Side effects

All explicit writes are relative to the destination base. When destination and source overlap, those writes can also modify source storage.

The supplied body contains no allocation, deallocation, ownership transfer, callback, I/O, synchronization operation, or explicit named-global access. It does not retain either address in a separately identified location. Storage remains caller-provided; the body does not release it.

The first argument has no effect on the established reads, writes, or branch choice.

### Aliasing and the selector reads

The marker write can affect both selector reads. Specifically, if destination offset `0x01` and source offset `0x08` identify the same byte, the marker replaces the source selector with `0x03` before either selector read. In the absence of external mutation, both reads then observe `0x03`.

The selector-copy write alone cannot make the two selector reads differ:

- Both the selector-copy destination access and each selector source access are one byte wide.
- They overlap only when destination offset `0x00` identifies exactly the same byte as source offset `0x08`.
- Under that arrangement, the copy writes the value just read back to the same byte.
- If those addresses differ, the copy does not write the selector’s byte.

Consequently, for ordinary valid storage with no external mutation, both selector reads agree even when source and destination overlap. Their common value need not equal the selector present at entry, because the earlier marker write may have changed it.

External activity between the reads is a separate possibility. If such activity changes the selector, the copied destination selector and the selector used for the branch may differ. The body establishes no synchronization or concurrency contract; this observation does not make unsynchronized concurrent modification supported or well-defined.

### Errors and partial completion

There is no explicit validation, status result, fallback, or rollback.

Invalid addresses, insufficient storage, unsupported alignment, or other invalid memory accesses are not translated into a defined function-level error. Their consequences depend on the target and execution environment.

No transactional guarantee is established. Earlier writes may already have occurred when a later access fails, but the supplied body alone does not define a portable post-failure memory state or recovery mechanism.

## Dependencies

No function calls appear in the supplied body. There are no opaque function dependencies to enumerate.

Execution relies on ordinary memory access and the target representation of multi-byte values. No claims about an unknown callee preserving globals, retaining addresses, or releasing storage are necessary because no callee is present.

## Acceptance examples

These examples are conditional predictions from static evidence. None has been executed. Unless stated otherwise, they assume valid, nonwrapping accesses, adequate storage and alignment, and no external mutation.

### Selector `0x00`: three-byte initialization

With nonoverlapping source and destination and source selector `0x00`:

- Destination offsets `0x00` and `0x01` become `0x00` and `0x03`.
- Destination offsets `0x30`–`0x32` become `10 00 FF`.
- Destination offset `0x33` retains its prior value.
- Both captured values are duplicated into their specified fields.
- The four unconditional clearing fields become zero.

A destination byte initially containing `0xA5` at offset `0x33` must therefore remain `0xA5`, rather than being cleared with the preceding three bytes.

### Selector `0x02`: two-byte initialization

With nonoverlapping storage and selector `0x02`, destination offset `0x00` becomes `0x02`, and the two-byte value at destination offset `0x30` becomes `0x001F`.

Destination offsets `0x32` and `0x33` retain their prior contents. The expected conditional bytes are `1F 00` only if the target is little-endian; `00 1F` is the corresponding big-endian expectation.

### Selector `0x03`: four-byte clearing

With selector `0x03`, all four bytes at destination offsets `0x30`–`0x33` become zero. The captured-value copies and unconditional clearing writes still occur afterward.

### Unrecognized selector

With nonoverlapping storage and selector `0x7F`, destination offset `0x00` becomes `0x7F`. Destination offsets `0x30`–`0x33` remain unchanged.

The marker, captured-value copies, and unconditional clearing fields are still written. No error status is returned.

### Distinct captured values

If source offsets `0x0C` and `0x10` contain distinct four-byte values, the first must appear at destination offsets `0x10` and `0x18`, while the second must appear at offsets `0x0C` and `0x14`.

This distinguishes the required placement from a same-offset source-to-destination copy.

### Identical source and destination bases

When the bases are identical:

- Both four-byte values are captured before any destination write.
- The marker at offset `0x01` does not overlap the selector at offset `0x08`.
- Both selector reads precede the final four-byte clearing write at offset `0x08`.
- That final write clears the shared selector storage, without changing the branch already selected.
- The copies use the original captured values despite intervening overwrites of the shared source fields.

### Marker aliases the source selector

If destination offset `0x01` equals source offset `0x08`, the marker write changes the selector to `0x03` before both reads.

The destination selector becomes `0x03`, and the four-byte-zero conditional branch is taken, regardless of the selector’s entry value.

### Selector copy aliases its source

If destination offset `0x00` equals source offset `0x08`, and that byte initially contains `0x02`:

- The marker write targets the following byte, not the selector.
- The first selector read obtains `0x02`.
- The selector copy writes `0x02` back to that same byte.
- The second selector read also obtains `0x02`.
- The two-byte conditional initialization is selected.

The copy does not itself create a mismatch between the copied selector and the branch selector.

## Uncertainties

The following remain **documented_only**:

- The application-level meanings of the fields and constants.
- Why each captured value is duplicated.
- Whether callers intentionally permit overlapping storage.
- The machine pointer width and the source-address argument’s exact width and conversion rules.
- Target byte order, alignment requirements, address-overflow semantics, and memory-fault behavior.
- Whether surrounding code initializes the untouched destination bytes.
- Whether an external interface restricts the selector domain.
- Machine-level atomicity, compiler transformations, concurrent visibility, and the legality of external mutation.

The interpretation as a partial structured-object initializer is inferred intent. The accesses, widths, branch conditions, and source-level ordering described above are established static evidence.

## Evidence

This contract is based on the supplied body for F-288efaeb11be, not on execution observations.

The body establishes two initial four-byte captures, a one-byte marker write, two selector reads separated by a one-byte selector-copy write, three conditional initialization forms, four captured-value writes, and four unconditional four-byte clearing writes. It contains no function calls, explicit validation, or returned status.

The aliasing conclusion follows from the one-byte access widths and their order: the selector copy either targets a different byte or writes the value just read back to the same byte. The preceding marker write, unlike that self-copy, can replace the selector before both reads.

No runtime tests have been performed. All acceptance examples remain conditional static predictions.
