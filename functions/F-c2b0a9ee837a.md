<!-- Trusted provenance: F-c2b0a9ee837a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c2b0a9ee837a

## Purpose

F-c2b0a9ee837a clears one selected bit in a byte-addressed memory region. The caller supplies a base address and an unsigned bit index. The function selects the byte containing that bit, preserves the other seven bits in the byte, and writes the modified byte back to the same address.

Bit numbering is least-significant-bit first within each byte. Index zero identifies the least significant bit of the first byte; index seven identifies its most significant bit; index eight identifies the least significant bit of the next byte.

This is a single-byte read-modify-write operation. It does not initialize a region, allocate storage, check a region’s size, or report whether the selected bit was previously set.

## Interface

**Inputs**

- **Base address:** An integer-valued argument used as the starting address of byte-addressed storage. The supplied representation declares this argument as an `int`, rather than as a typed pointer. Its role as an address is established by its use in a byte dereference.
- **Bit index:** An unsigned integer selecting the bit to clear. There is no length argument or other explicit bound.

**Output**

The function has no return value. Its normal observable result is the byte written at the selected address. It supplies no success flag, previous-bit value, or error code.

**Widths and interpretation**

The representation explicitly performs a byte-sized load and a byte-sized store. The selected bit position is restricted to the range zero through seven, establishing an eight-bit byte interpretation for this operation.

The exact widths of `int`, the unsigned index type, and machine addresses are not independently established by the supplied fragment. Under a 32-bit unsigned-index interpretation, the index ranges from 0 through 4,294,967,295, and the derived byte offset ranges from 0 through 536,870,911. These bounds are conditional on that width; they are not evidence of a permitted allocation size.

**Caller preconditions**

For an ordinary successful memory update:

- The derived address must designate a readable and writable byte.
- Address calculation must be meaningful under the target’s actual integer and address rules.
- The storage must remain accessible through the read and subsequent write.
- Any required synchronization against concurrent access must be supplied externally.

The function does not verify these preconditions.

## Behavior

### Bit selection and layout

The byte offset is the unsigned bit index shifted right by three positions. Equivalently, it is the integer quotient of the index divided by eight.

The bit position within that byte is the index’s low three bits, equivalent to its remainder modulo eight. Consequently:

| Index range | Selected byte offset | Selected bit positions |
|---|---:|---|
| 0–7 | 0 | 0–7 |
| 8–15 | 1 | 0–7 |
| 16–23 | 2 | 0–7 |

The same mapping continues for larger representable indices. There is no special handling at byte boundaries.

Within each byte, position zero has numeric mask 1 and position seven has numeric mask 128. Because the operation accesses one byte rather than a multi-byte integer, machine byte order does not alter this within-byte mapping.

### Read and write

The function derives the selected address by adding the byte offset to the supplied base address.

At the source-visible level, it reads one byte from that address, clears the selected bit in the value read, and writes the resulting byte to that same address. The read supplies the value used by the write. There is no conditional branch that skips the write when the bit is already clear.

For an ordinary byte value, the result is its bitwise intersection with an eight-bit mask whose selected position is zero and whose other seven positions are one. Thus:

- A selected bit that was one becomes zero.
- A selected bit that was zero remains zero.
- Every nonselected bit retains the value obtained by the read.

The bit-position restriction prevents an out-of-range shift when forming the single-bit mask: the shift count is always between zero and seven. Complementing that mask may involve wider intermediate integer operations, but the final byte store retains the byte-sized clearing result.

### Boundaries and address arithmetic

The function contains no check against the capacity of the region. A small bit index can still address invalid storage if the base is invalid; a large bit index can be valid only if the corresponding derived address is valid.

The unsigned right shift used to derive the offset does not round upward and does not sign-extend. There is no negative-index case in the declared interface. Passing a value originating from a negative integer would require conversion before or at the call; the resulting unsigned value, not the original negative interpretation, determines the selected location.

The representation does not establish a portable rule for address overflow or for converting the integer-valued address into a usable byte address. No successful wraparound behavior should be assumed.

## Side effects

The established data-memory effect is a byte write at the selected address, preceded by the byte read used to calculate its value. No adjacent-byte or whole-word data update is expressed in the supplied representation.

An already-clear bit produces an unchanged stored value, but the source still expresses a write. Value-level idempotence therefore does not imply that a repeated call is side-effect-free on memory-mapped devices or other storage where writes themselves have significance.

The function does not express an atomic operation, locking, or a memory barrier. It provides no basis for assuming a concurrent modification will be preserved. For example, an intervening update to another bit in the same byte can be overwritten if the final store uses an earlier value.

There is no allocation, deallocation, ownership transfer, retained reference, explicit global-state access, or dependency call in the supplied body. The base address may nevertheless designate shared or global storage; that cannot be determined here.

The precise machine instruction sequence, bus transactions, and device-access guarantees are not established by this source representation.

## Dependencies

No function dependencies are present in the supplied body. The contract does not rely on any other opaque function ID.

Execution does rely on the surrounding environment providing suitable address semantics and accessible storage. Allocation extent, mapping permissions, storage lifetime, concurrency control, and any device-specific access requirements remain external responsibilities.

Because no callees appear, there is no unknown callee whose memory ownership or global-state preservation must be assumed.

## Acceptance examples

The following are conditional behavioral examples derived from the supplied representation, not results of runtime tests. Each successful example assumes ordinary readable and writable memory, valid address arithmetic, and no concurrent interference.

| Initial selected byte | Bit index | Byte offset | Expected stored byte |
|---|---:|---:|---|
| `0xFF` | 0 | 0 | `0xFE` |
| `0xFF` | 7 | 0 | `0x7F` |
| `0xFF` | 8 | 1 | `0xFE` |
| `0xA5` | 2 | 0 | `0xA1` |
| `0xA5` | 1 | 0 | `0xA5` |
| `0x00` | 15 | 1 | `0x00` |

Additional acceptance conditions:

- **Byte-boundary selection:** Given two accessible bytes initially both `0xFF`, index seven changes the first byte to `0x7F` and leaves the second unchanged. Index eight instead changes the second byte to `0xFE` and leaves the first unchanged.
- **Preservation:** For any initial byte value, all seven nonselected bits in the stored result equal their values in the byte read.
- **Repeated clearing:** With no intervening modification, two calls selecting the same bit leave the same final byte value as one call. Each call still expresses a read and a write.
- **Index-width boundary:** If the index is 32 bits wide, its maximum value selects offset 536,870,911 and bit position seven. This establishes selection arithmetic only, not valid access.
- **Invalid storage:** If the selected byte is inaccessible, no successful-return behavior or particular error outcome is specified here. Acceptance must not require graceful rejection or an error code.

## Uncertainties

**Inferred intent:** The operation is consistent with clearing an entry in a packed bitmap. The actual application meaning of a cleared bit is unknown.

**documented_only — ABI and widths:** Exact argument widths, calling convention, and integer-to-address conversion behavior require target information absent from the fragment.

**documented_only — address validity:** Allocation extent, provenance, lifetime, alignment requirements of the execution environment, and permitted address range are unknown. The byte access itself expresses no multi-byte alignment requirement.

**documented_only — failure behavior:** The fragment contains no explicit error handling. Fault delivery, process termination, recovery, or other consequences of invalid access are environment-dependent and unestablished.

**documented_only — concurrency and devices:** Atomicity, synchronization, volatile-access semantics, and suitability for memory-mapped hardware are not established.

## Evidence

The supplied static representation establishes an unsigned-index-derived byte offset, a low-three-bit bit position, one byte-valued read, a mask that clears the selected bit, and a byte-valued write to the selected address. It also establishes the absence of explicit bounds checks, conditional write suppression, dependency calls, allocation, and a returned result.

The packed-bitmap interpretation is inferred intent, not additional behavior. Width-specific numeric limits are conditional as identified above.

No runtime tests have been performed.
