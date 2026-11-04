<!-- Trusted provenance: F-1fcfebb614a8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1fcfebb614a8

## Purpose

F-1fcfebb614a8 packs selected fields from an eight-byte input region into two 16-bit results, each stored through a 32-bit destination.

The field arrangement is consistent with a packed calendar date and clock time:

- The first destination receives a day field, a month field, and a year offset from 1980.
- The second destination receives a two-second count, a minute field, and an hour field.

That interpretation is inferred intent. The established behavior is masking, shifting, and combining input bits; the function does not validate calendar or clock values.

The function also performs intermediate destination writes. Its contract is therefore not limited to the two final packed values. Overlapping input and output regions, overlapping destinations, or observers of intermediate memory states can make the access sequence significant.

## Interface

The interface has three pointer arguments and no returned value:

| Argument | Role | Required accessed extent |
|---|---|---|
| Input | Supplies five individual bytes and one 16-bit value | Offsets 0–4 and 6–7 must be readable |
| First destination | Receives the packed date-like result through 32-bit accesses | Four bytes must be readable and writable |
| Second destination | Receives the packed time-like result through 32-bit accesses | Four bytes must be readable and writable |

The supplied representation uses 8-bit byte accesses, 16-bit halfword accesses, and 32-bit unsigned word accesses. These widths define the bit-field behavior described below. The platform’s byte order, alignment rules, and pointer-access rules are not independently established.

Although each final result occupies only bits 0–15, neither destination is merely a two-byte output buffer. Each receives an initial 16-bit write, a 32-bit read, and subsequent 32-bit writes.

The input layout is:

| Byte offset | Consumed information |
|---|---|
| 0 | Bits 1–5 become the five-bit two-second field |
| 1 | Bits 0–5 become the six-bit minute field |
| 2 | Bits 0–4 become the five-bit hour field |
| 3 | Bits 0–4 become the five-bit day field |
| 4 | Bits 0–3 become the four-bit month field |
| 5 | Not directly read |
| 6–7 | One native-order 16-bit unsigned year-like value |

There is no input length argument, output capacity argument, success indicator, or error code.

The caller retains ownership of all supplied storage. The function neither allocates nor releases memory.

For ordinary successful use, the caller must provide valid storage for every access and satisfy the execution platform’s alignment and memory-access requirements. In particular, a byte-addressed input is also accessed as a 16-bit value at offset 6.

## Behavior

### Packed result definitions

For nonoverlapping input and destinations, the final low 16 bits of the second destination contain:

| Output bits | Value |
|---|---|
| 0–4 | Input byte 0, restricted to its low six bits, then shifted right by one |
| 5–10 | Low six bits of input byte 1 |
| 11–15 | Low five bits of input byte 2 |

Consequently:

- Input byte 0’s least significant bit is discarded.
- Input byte 0’s top two bits are ignored.
- Input byte 1’s top two bits are ignored.
- Input byte 2’s top three bits are ignored.

The first field represents values from 0 through 31. If interpreted as elapsed seconds divided by two, odd seconds are rounded down and the representable decoded range extends to 62 seconds. The function does not reject the out-of-normal-clock-range values.

The final low 16 bits of the first destination contain:

| Output bits | Value |
|---|---|
| 0–4 | Low five bits of input byte 3 |
| 5–8 | Low four bits of input byte 4 |
| 9–15 | Low seven bits of the 16-bit input value minus 1980 |

The year field is equivalently the year-like input minus 1980, reduced modulo 128. There is no range check before packing.

The fields occupy disjoint bit positions. Combining them introduces no carries between fields.

### Ordered memory operations

The established access sequence is:

1. Write a zero-valued 16-bit halfword at the first destination’s address.
2. Write a zero-valued 16-bit halfword at the second destination’s address.
3. Read a 32-bit word from the second destination and retain it as the snapshot for all three time-building writes.
4. Read input byte 0.
5. Write the second destination as a 32-bit word, replacing bits 0–4 of its saved snapshot with the first time field.
6. Read input byte 1.
7. Write the second destination as a 32-bit word, replacing bits 0–10 of that same saved snapshot with the first two time fields.
8. Read input byte 2.
9. Write the second destination as a 32-bit word, replacing bits 0–15 of that same saved snapshot with all three time fields.
10. Read a 32-bit word from the first destination and retain it as the snapshot for all three date-building writes.
11. Read input byte 3.
12. Write the first destination as a 32-bit word, replacing bits 0–4 of its saved snapshot with the day field.
13. Read input byte 4.
14. Write the first destination as a 32-bit word, replacing bits 0–8 of that same saved snapshot with the day and month fields.
15. Read a 16-bit unsigned value at input offset 6.
16. Write the first destination as a 32-bit word, replacing bits 0–15 of that same saved snapshot with all three date fields.
17. Return without a value.

The snapshots are not refreshed between a destination’s three full-word writes. For example, the second time-building write does not read back the first time-building write.

This sequence describes the supplied static representation. It does not establish atomicity, hardware bus transaction sizes, compiler treatment of concurrent access, or externally observable ordering guarantees for unsynchronized shared memory.

### Initialization and preserved bits

The initial writes clear the halfword at each destination’s address. Whether that halfword corresponds to the numerical low or high 16 bits of a subsequent 32-bit read depends on byte order.

On a conventional little-endian platform with disjoint destinations:

- The initial writes clear numerical bits 0–15.
- The original bits 16–31 survive the snapshot reads.
- Every subsequent full-word write preserves those saved upper bits.
- Each final destination therefore consists of its original upper 16 bits and its newly packed lower 16 bits.

On a conventional big-endian platform with disjoint destinations:

- The initial writes clear numerical bits 16–31.
- The saved upper 16 bits are consequently zero.
- The final full-word writes leave the packed results in numerical bits 0–15 and zeros in bits 16–31.

Thus, preservation of the caller’s original upper halfword is conditional on byte order, not an unconditional property of the function.

### Integer widths and year handling

All packed results fit within 16 bits:

- The time fields use 5, 6, and 5 bits.
- The date fields use 5, 4, and 7 bits.

The 16-bit year-like input ranges from 0 through 65,535. Under the conventional 32-bit integer model, subtracting 1980 produces a range of −1980 through 63,555. Multiplication by 512 then produces a range of −1,013,760 through 32,540,160, which fits within a signed 32-bit integer.

The subsequent mask retains only bits 9–15. Under the conventional two’s-complement interpretation, this yields the modulo-128 year field described above, including for years below 1980. It does not saturate negative offsets or reject years above the intended calendar range.

### Overlapping regions

No overlap check or rejection is present.

If an output overlaps the input, an earlier output write can change a byte that the function reads later. The input is not captured as an eight-byte snapshot at entry. Its individual fields are read at the points listed above.

If the destinations overlap, time-building writes can affect the first destination before its snapshot is read. Later date-building writes can then overwrite some or all of the time result.

If both destination arguments refer to exactly the same 32-bit word, and that word is disjoint from the input, the date-building stage is last. Its final low 16 bits replace the previously constructed time value.

For partial overlap, the final memory contents depend on byte order, pointer offsets, and access legality. The stated sequence, rather than two independently computed output values, governs the result.

## Side effects

For a successful invocation, the supplied representation contains:

- Two 16-bit destination writes.
- Two 32-bit destination reads.
- Six 32-bit destination writes.
- Five individual input-byte reads.
- One 16-bit input read.

Input offset 5 is not directly accessed. Its contents do not affect the packed values when the input and outputs are disjoint.

No explicit global-memory access, allocation, deallocation, logging, external I/O, callback, or synchronization operation appears.

The function performs no pointer, alignment, capacity, or field-range validation. Invalid storage can fail at the first incompatible access. There is no recovery path or rollback: earlier writes may already have occurred before a later access fails.

The precise fault mechanism and treatment of invalid or misaligned accesses are platform-dependent and are not established here.

## Dependencies

No called-function dependencies are visible for F-1fcfebb614a8. There are no additional opaque function IDs to specify.

The function depends on platform behavior for:

- Byte order of the 16-bit year read and destination accesses.
- Validity and alignment of 16-bit and 32-bit memory operations.
- Integer widths and signed representation.
- Legality of overlapping typed accesses.
- Visibility and atomicity of ordinary memory accesses.

No assumption about an unknown callee preserving globals or transferring ownership is needed, because no call is shown.

## Acceptance examples

These are conditional acceptance cases derived from the static operations, not executed test results. Unless otherwise specified, they assume conventional 8/16/32-bit widths, two’s-complement arithmetic, little-endian storage, valid aligned accesses, and mutually disjoint input and destinations.

### Ordinary calendar and clock values

Given:

- Input byte 0: 59.
- Input byte 1: 58.
- Input byte 2: 23.
- Input byte 3: 31.
- Input byte 4: 12.
- The 16-bit value at offset 6: 2026.
- First destination initially: `0xA1B2C3D4`.
- Second destination initially: `0xE5F60718`.

The final first destination is `0xA1B25D9F`.

The final second destination is `0xE5F6BF5D`.

The time field stores 29 two-second units, not the original 59 seconds.

### Ignored high bits and discarded odd-second bit

With all other consumed fields unchanged, input byte 0 values 58, 59, 122, and 123 produce the same first time field: 29.

Changing only the upper two bits of input byte 1, the upper three bits of input bytes 2 or 3, or the upper four bits of input byte 4 does not change the corresponding packed field.

These equivalences require disjoint storage so that changed input bytes cannot also change destination snapshots.

### Year wrapping

Holding day and month at zero:

| Year-like input | Encoded year field | Final date low halfword |
|---|---:|---|
| 1980 | 0 | `0x0000` |
| 2107 | 127 | `0xFE00` |
| 2108 | 0 | `0x0000` |
| 1979 | 127 | `0xFE00` |

These results demonstrate truncation modulo 128 rather than range validation.

### Invalid calendar fields remain representable

If input bytes 0–4 are all 255:

- The time low halfword is `0xFFFF`.
- The day field is 31.
- The month field is 15.

With a year-like input of 1980, the date low halfword is `0x01FF`.

The month value 15 is packed without rejection. The function does not enforce a valid date.

### Identical destinations

Using the ordinary calendar and clock input above, if both destinations point to one disjoint word initially containing `0x12345678`, the final shared word is `0x12345D9F`.

The intermediate time result is overwritten by the later date result.

### Byte-order qualification

On a conventional big-endian platform, with disjoint destinations and the input halfword supplied in native order, the ordinary example’s final destination words are `0x00005D9F` and `0x0000BF5D`.

The original upper halves do not survive because the initial zero-halfword writes clear them.

## Uncertainties

- **documented_only — Runtime platform:** Byte order, alignment requirements, integer ABI, and signed representation are not independently confirmed.
- **documented_only — Semantic naming:** The date/time interpretation and 1980 epoch are inferred from the field layout and arithmetic. No external format contract is supplied.
- **documented_only — Caller guarantees:** No evidence establishes whether callers ensure valid fields, disjoint buffers, initialized destination words, or suitable alignment.
- **documented_only — Overlap legality:** The access sequence is visible, but the language and platform rules governing particular overlapping typed accesses are not supplied.
- **documented_only — Concurrent observation:** No synchronization or volatile-access contract is shown. Intermediate writes must not be treated as a supported concurrent publication protocol.
- **documented_only — Failure behavior:** There is no explicit error handling. Platform-specific faults and the state remaining after an invalid access are not characterized beyond the preceding write sequence.

## Evidence

Established static evidence consists of two initial halfword-zero writes, two full-word snapshot reads, three full-word construction writes per destination, five byte reads, and one halfword read from the input.

The masks establish the output field widths and placement. The year subtraction establishes the 1980 offset. The repeated use of saved destination words establishes that intermediate writes do not refresh preserved bits from memory.

The interpretation as calendar and clock packing is inferred intent. Platform-specific access semantics, caller invariants, and runtime behavior remain unverified.

No runtime tests have been performed.
