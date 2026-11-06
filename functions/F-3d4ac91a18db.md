<!-- Trusted provenance: F-3d4ac91a18db; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3d4ac91a18db

## Purpose

F-3d4ac91a18db populates a caller-supplied output record from a source record and a secondary record referenced by the source. It copies selected fields, widens two unsigned 16-bit values, derives a 32-bit flag field from one source byte, writes fixed values, and computes a rounded block count using 32-bit unsigned arithmetic.

The established behavior is a fixed sequence of memory accesses. No external function calls, allocation, validation, or error reporting are visible.

The apparent intent is to construct a descriptor containing sizes, flags, and several 16-bit attributes. The meanings of those attributes and flags are not established. The contract therefore identifies them by offsets and widths rather than assigning domain-specific names.

## Interface

### Inputs

The function receives two address-like arguments:

- **Source address:** identifies the source record.
- **Output address:** identifies caller-provided writable storage.

The supplied representation uses a 32-bit address-bearing field at source offset `0x2C`. That field identifies a secondary record. This specification describes offsets in bytes and data widths explicitly; it does not infer a portable native interface beyond the supplied representation.

The source fields accessed are:

| Source offset | Width | Use |
|---|---:|---|
| `0x0B` | 8 bits | Copied to the output and used to derive flags |
| `0x0E` | 16 bits | Copied to output offset `0x22` |
| `0x10` | 16 bits | Copied to output offset `0x20` |
| `0x12` | 16 bits | Copied to output offset `0x18` |
| `0x16` | 16 bits | Copied to output offset `0x1E` |
| `0x18` | 16 bits | Copied to output offset `0x1C` |
| `0x1C` | 32 bits | Copied to output offset `0x14`; read again for the final calculation |
| `0x2C` | 32 bits | Secondary-record address; read twice |

Within the secondary record, the function reads:

| Secondary-record offset | Width | Use |
|---|---:|---|
| `0x78` | Unsigned 16 bits | Zero-extended and written to two output fields |
| `0x64` | Unsigned 16 bits | Zero-extended and written to one output field |

The two secondary-record reads do not necessarily use the same address value: the source address field is reread before the second access.

### Output

There is no returned value. The result is conveyed through writes to the output storage.

The function writes every byte from output offset `0x00` through `0x2C`, inclusive. Consequently, the visible output footprint is **45 bytes**, not 44 bytes.

| Output offset | Width | Final value on normal completion |
|---|---:|---|
| `0x00` | 32 bits | Zero-extended initial secondary value from offset `0x78` |
| `0x04` | 32 bits | Zero |
| `0x08` | 32 bits | Derived flag field |
| `0x0C` | 32 bits | One |
| `0x10` | 32 bits | Same cached value written at output offset `0x00` |
| `0x14` | 32 bits | Source value read from offset `0x1C` at this stage |
| `0x18` | 16 bits | Source value read from offset `0x12` |
| `0x1A` | 16 bits | Zero |
| `0x1C` | 16 bits | Source value read from offset `0x18` |
| `0x1E` | 16 bits | Source value read from offset `0x16` |
| `0x20` | 16 bits | Source value read from offset `0x10` |
| `0x22` | 16 bits | Source value read from offset `0x0E` |
| `0x24` | 32 bits | Zero-extended later secondary value from offset `0x64` |
| `0x28` | 32 bits | Wrapped-addition block-count calculation |
| `0x2C` | 8 bits | Cached source byte read from offset `0x0B` |

This table describes logical fields, not a serialized byte order. Endianness is not established by the supplied evidence.

### Preconditions and ownership

For normal completion:

- Every address actually read must designate readable storage for the indicated width.
- The output must support all indicated writes, including the byte at offset `0x2C`.
- Address arithmetic and accesses must be valid under the execution environment’s address and alignment rules.
- Any secondary-record address obtained during execution must be valid for the access that follows it.

The function does not visibly acquire, release, allocate, or retain ownership of any storage. It does not store either argument for later use.

Non-overlap between the input records and output is not checked. Overlap can change values read later and can invalidate a later secondary-record access. A caller wanting a straightforward conversion of stable input values should provide non-overlapping storage and prevent concurrent modification.

## Behavior

### Initial secondary value

The function first reads the secondary-record address from source offset `0x2C`. Through that address, it reads the unsigned 16-bit value at secondary offset `0x78`.

That value is zero-extended to 32 bits and retained for two output writes. In particular, an input value of `0xFFFF` becomes `0x0000FFFF`, not a sign-extended value.

Before reading the source flag byte, the function writes, in order:

1. The widened secondary value to output offset `0x00`.
2. A 32-bit zero to output offset `0x04`.
3. A 32-bit zero to output offset `0x08`.

These writes matter if the output aliases source storage that will be read later.

### Source-byte capture and derived flags

The function next reads one byte from source offset `0x0B` and retains it. It writes that byte to output offset `0x2C`.

All flag decisions use this retained byte. The source byte is not reread for subsequent conditions.

The function then replaces the zero at output offset `0x08` with `0x00000080`. It applies the following decisions in order:

- If source-byte bit `0x01` is clear, replace the field with `0x00000180`.
- If source-byte bit `0x10` is set, read the current output flag field and write it back with `0x00004000` set.
- If neither source-byte bit `0x08` nor source-byte bit `0x10` is set, read the current output flag field and write it back with `0x00008000` set.

Thus:

- Output bit `0x80` is always set.
- Output bit `0x100` is set exactly when source-byte bit `0x01` is clear.
- Output bit `0x4000` is set exactly when source-byte bit `0x10` is set.
- Output bit `0x8000` is set exactly when both source-byte bits `0x08` and `0x10` are clear.
- No other output flag bits are introduced.
- Source-byte bits outside `0x19` do not affect the derived field, although the entire original byte is copied to output offset `0x2C`.

The initial zero, base value, and any conditional updates are distinct writes in the visible sequence. The routine does not construct the flag field solely in temporary storage and publish it once.

### Remaining field population

After finishing the flag field, the function performs these accesses in order:

1. Write the 32-bit value one to output offset `0x0C`.
2. Write the cached initial secondary value to output offset `0x10`.
3. Read 32 bits from source offset `0x1C` and write them to output offset `0x14`.
4. Read 16 bits from source offset `0x10` and write them to output offset `0x20`.
5. Read 16 bits from source offset `0x0E` and write them to output offset `0x22`.
6. Read 16 bits from source offset `0x12` and write them to output offset `0x18`.
7. Write a 16-bit zero to output offset `0x1A`.
8. Read 16 bits from source offset `0x18` and write them to output offset `0x1C`.
9. Read 16 bits from source offset `0x16` and write them to output offset `0x1E`.

The order of the 16-bit writes is not ascending output-address order. In particular, output offsets `0x20` and `0x22` are written before output offset `0x18`.

The copied 16-bit fields preserve their bit patterns. No signed interpretation or widening is needed for those copies.

### Later secondary read

The function rereads the secondary-record address from source offset `0x2C`. It then reads the unsigned 16-bit value at offset `0x64` of the record addressed by that later value.

The 16-bit result is zero-extended and written as 32 bits at output offset `0x24`.

There is no check that the later secondary-record address equals the initial one. With overlapping storage, the function’s preceding writes can affect this address. With externally mutable storage, an outside writer can also affect it; synchronization behavior is not established.

### Final arithmetic

Finally, the function rereads the 32-bit source field at offset `0x1C`. It does not reuse the value previously copied to output offset `0x14`.

Let the newly read 32-bit bit pattern, interpreted as unsigned, be \(n\). The value written to output offset `0x28` is:

\[
\left((n + 511) \bmod 2^{32}\right) \mathbin{\mathrm{div}} 512
\]

The division is implemented by an unsigned right shift of nine bits after the 32-bit addition. The order and width are significant: addition wraps before the shift.

For \(0 \leq n \leq \texttt{0xFFFFFE00}\), this equals the mathematical ceiling of \(n/512\). For the final 511 values in the unsigned 32-bit range, the addition wraps and the result is zero.

The function returns after this write.

## Side effects

The visible side effects are the output writes described above. There are no visible global-variable accesses, external calls, allocations, releases, logging operations, or explicit status updates.

The output is updated incrementally. The function provides no visible transaction, rollback, or publish-on-success mechanism. If an access fails after earlier writes, the evidence provides no recovery path and no guarantee that the output remains unchanged.

The output is not cleared as a larger buffer. Storage beyond offset `0x2C` is not visibly written.

Aliasing deserves particular attention:

- The first secondary value is read before any output write, then reused from retained state.
- The source byte is read only after three output writes.
- The secondary-record address is read again late in execution.
- The source field at offset `0x1C` is read twice, with many writes between those reads.

Accordingly, output offsets `0x14` and `0x28` need not describe the same source-field value when storage overlaps or is modified during execution.

No general atomicity, thread-safety, memory-ordering, or memory-mapped-I/O guarantee is established. The ordered accesses here describe the supplied static representation, not a verified machine-level concurrency contract.

## Dependencies

No function dependencies are visible. F-3d4ac91a18db contains no visible call to another opaque function ID.

Its data dependencies are:

- The source record.
- The secondary record selected by the first address read.
- The secondary record selected by the later address read.
- Caller-provided output storage.
- The environment’s memory-access, alignment, address-representation, and byte-order rules.

No assumption about an unknown callee preserving globals or managing memory is needed, because no such callee is visible.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless noted otherwise, they assume valid, stable, non-overlapping records and sufficient output storage.

### Widening and repeated output value

Given a secondary value of `0xFEDC` at offset `0x78`:

- Output offset `0x00` contains the 32-bit value `0x0000FEDC`.
- Output offset `0x10` contains the same 32-bit value.
- Neither field contains a sign-extended result.

Given a secondary value of `0xFFFF` at offset `0x64`, output offset `0x24` contains `0x0000FFFF`.

### Flag combinations

| Source byte | Expected 32-bit output at `0x08` | Expected byte at `0x2C` |
|---|---|---|
| `0x00` | `0x00008180` | `0x00` |
| `0x01` | `0x00008080` | `0x01` |
| `0x08` | `0x00000180` | `0x08` |
| `0x09` | `0x00000080` | `0x09` |
| `0x10` | `0x00004180` | `0x10` |
| `0x11` | `0x00004080` | `0x11` |
| `0x18` | `0x00004180` | `0x18` |
| `0xFF` | `0x00004080` | `0xFF` |

The last two rows demonstrate that bit `0x08` does not cancel `0x4000`, and unrelated source bits remain present in the copied byte without appearing in the derived field.

### Field placement

Given distinct 16-bit source values at offsets `0x0E`, `0x10`, `0x12`, `0x16`, and `0x18`, those values must appear respectively at output offsets `0x22`, `0x20`, `0x18`, `0x1E`, and `0x1C`.

Output offset `0x1A` must contain a 16-bit zero regardless of the prior output contents.

### Arithmetic boundaries

For a stable source value at offset `0x1C`:

| Source value | Expected output at `0x28` |
|---|---:|
| `0` | `0` |
| `1` | `1` |
| `511` | `1` |
| `512` | `1` |
| `513` | `2` |
| `0xFFFFFE00` | `0x007FFFFF` |
| `0xFFFFFE01` | `0` |
| `0xFFFFFFFF` | `0` |

The last two cases reject an implementation that uses wider non-wrapping addition to calculate an unrestricted mathematical ceiling.

### Write extent

Given valid output storage with observable guard bytes immediately after offset `0x2C`, normal completion must leave those guard bytes unchanged. The byte at offset `0x2C` is part of the result and must not be treated as a guard byte.

### Aliasing-sensitive behavior

If a permitted overlap changes source offset `0x1C` between its two reads, the copied value at output offset `0x14` reflects the earlier read, while the calculation at output offset `0x28` reflects the later read.

If overlap changes the secondary-record address before its second read, the later access uses the changed address. Acceptance of a concrete overlapping arrangement also requires every resulting access to remain valid.

## Uncertainties

The following remain **documented_only**:

- Domain meanings, units, and names of the copied fields and derived flag bits.
- The intended public record definitions and required alignment.
- Target endianness and the portable calling convention.
- Whether callers are required by an external contract to avoid overlap.
- Whether surrounding code guarantees immutability during execution.
- Behavior on null, invalid, misaligned, or inaccessible addresses, including fault delivery.
- Whether source values near the 32-bit arithmetic wrap boundary are valid application inputs.
- Whether the wrap behavior is deliberate or an incidental consequence of the arithmetic width.

Descriptor construction and 512-unit upward rounding are inferred intent. The fixed field transfers, bit decisions, repeated reads, and wrapped calculation are established static behavior.

## Evidence

This contract is based solely on the supplied static representation of F-3d4ac91a18db.

That representation establishes the access widths and offsets, output-write order, retained initial secondary value, retained source byte, flag conditions, repeated source reads, absence of visible calls, and unsigned 32-bit addition followed by a nine-bit right shift.

No runtime tests have been performed. The acceptance examples are derived criteria, not observations of execution. No independent evidence establishes the application-level field meanings, target memory model, or caller-side validity guarantees.
