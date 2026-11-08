<!-- Trusted provenance: F-95eefe30cf9e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-95eefe30cf9e

## Purpose

F-95eefe30cf9e derives two positions from an input pair, conditionally rejects them, and otherwise modifies a byte-addressed bitmap. Before modifying the bitmap, it conditionally writes two 16-bit state values based on signed byte-position comparisons.

Range marking and accumulated bounds are plausible intent, not unconditional semantics. In particular, the upper comparison reads **32 bits**, extracts the numeric low 16 bits, and interprets those bits as signed; its corresponding update writes **16 bits** at the same starting address. Whether that update changes the value used by a subsequent comparison depends on the memory layout and endianness.

The function does not validate endpoint ordering, bitmap capacity, configuration consistency, or storage ownership. Accepted reversed positions and a zero limit can still cause bitmap writes.

## Interface

### Inputs and result

The function receives four arguments:

1. **State address:** identifies configuration, comparison fields, update destinations, and bitmap address components.
2. **Unused argument:** a 32-bit value not inspected by this function.
3. **Input start:** a signed 32-bit integer.
4. **Input end:** an unsigned 32-bit integer.

There is no returned value or explicit error result. Observable results are memory accesses and writes.

### State layout

All offsets are byte offsets from the state address.

| Offset | Read access | Write access | Interpretation |
|---|---|---|---|
| 0 | 32 bits | None | Numeric low eight bits provide a shift count |
| 4 | 32 bits | None | Quantum used in subtraction, addition, and mask construction |
| 24 | 32 bits, read twice | None | Threshold interpreted as signed 32-bit |
| 60 | 32 bits, conditionally | None | Numeric low 16 bits provide an unsigned limit |
| 64 | 32 bits | None directly | First bitmap address component |
| 152 | 32 bits | None directly | Second bitmap address component |
| 162 | 16 bits | 16 bits, conditionally | Signed lower comparison value and update destination |
| 164 | **32 bits** | **16 bits, conditionally** | Comparison uses the signed interpretation of the read’s numeric low 16 bits; update begins at offset 164 |

“None directly” does not exclude modification through overlapping bitmap storage.

The upper comparison requires all four bytes at offsets **164 through 167** to be readable. This remains true when no upper update occurs. An upper update writes only the two bytes beginning at offset 164.

The limit likewise comes from a full 32-bit read, requiring offsets 60 through 63 to be readable when that path is reached. Numeric low-half extraction does not identify particular addressed bytes without an established endianness.

### Arithmetic and memory preconditions

The supplied representation uses 32-bit arithmetic. Unsigned addition, subtraction, negation, and left shifts retain the low 32 bits. Signed interpretations use two’s-complement values. Signed right shifts are arithmetic in the represented operations.

The shift count is the numeric low eight bits of a 32-bit configuration read. Behavior for counts of 32 through 255 is not established.

Memory requirements are path-dependent:

- Every state field actually read must support its full access width.
- A conditional 16-bit destination must be writable when its update is selected.
- Bitmap endpoint bytes must support both reading and writing.
- Intermediate bitmap bytes must be writable.
- Computed addresses and access alignment must be valid under the execution environment.

There are no explicit checks enforcing these requirements.

## Behavior

### Position derivation and configuration-read order

The function first reads the quantum at offset 4. It calculates a residual by subtracting the input start, then the quantum, from the input end. The operations use 32-bit wrapping arithmetic; the start contributes its 32-bit representation.

A retained upper source initially equals that residual. The function reads the threshold at offset 24 and compares it with the residual, interpreting both as signed 32-bit integers. If the threshold is strictly smaller, the retained source becomes the input end ANDed with the 32-bit negation of the quantum.

The lower-position numerator is the wrapping sum of the quantum and input start, minus one, ANDed with the 32-bit negation of the quantum. The function reads the shift configuration at offset 0, takes its numeric low eight bits, interprets the numerator as signed, and arithmetically shifts it right. The resulting 32-bit pattern is the **derived lower position**.

The derived upper position initially equals the derived lower position.

The threshold is then read again at offset 24. If this second signed comparison finds the threshold strictly smaller than the original residual, the function reads the shift configuration again and arithmetically shifts the retained upper source, interpreted as signed. That result becomes the **derived upper position**.

With stable configuration, the two threshold decisions agree. If they differ, the literal data flow remains important: a false first decision followed by a true second decision shifts the retained residual, not the masked input end. The representation does not provide synchronization or guarantee that concurrent executions expose every listed read separately.

The mask resembles alignment for some positive power-of-two quantum values. No alignment property is enforced; other quantum values retain their literal arithmetic and bitwise effects.

### Rejection

A negative signed derived upper position causes an immediate return. This path does not read the limit and performs no writes.

Otherwise, the function reads 32 bits at offset 60 and extracts the unsigned numeric low 16 bits as the limit, ranging from 0 through 65,535.

It then rejects if either:

- The derived lower position equals the limit; or
- The signed derived lower position is greater than the limit.

Consequently, after the nonnegative-upper check, acceptance requires the signed derived lower position to be strictly less than the limit.

A derived upper position of zero is not independently rejected. All rejection paths precede the bound-comparison accesses and all memory writes.

### Clamping

After acceptance:

- A negative signed derived lower position becomes zero.
- A derived upper position greater than or equal to the limit becomes the limit minus one, using 32-bit subtraction.

These results are the **effective lower position** and **effective upper position**.

There is no later rejection for reversed positions.

For a nonzero limit, both effective positions lie between zero and the limit minus one, but they need not be ordered.

For a zero limit, acceptance remains possible when the derived lower position is negative and the derived upper position is nonnegative. Clamping then produces effective positions zero and `0xffffffff`. The subtraction underflow is retained.

### Masks and byte-position conversion

Bits within a bitmap byte are numbered most-significant first: position remainder zero corresponds to `0x80`, and remainder seven corresponds to `0x01`.

The **lower mask** contains the lower endpoint bit and every less-significant bit in that byte. Its value is the low byte of `0xff` shifted right by the effective lower position modulo eight.

The **upper mask** contains the upper endpoint bit and every more-significant bit in that byte. Its value is the eight-bit complement of `0x7f` shifted right by the effective upper position modulo eight.

Each effective position also undergoes a narrowing conversion: bits 3 through 18 become a signed 16-bit value, then are sign-extended. These values are the **lower byte position** and **upper byte position**.

This is not unrestricted division by eight. For the conversion considered independently:

- Positions 0 through 262,143 produce byte positions 0 through 32,767.
- Positions 262,144 through 524,287 produce byte positions −32,768 through −1.
- Higher position bits do not affect the result.

With a nonzero limit, clamping restricts the actual positions sufficiently that both byte positions are nonnegative and at most 8,191. In the accepted zero-limit case, the byte positions are zero and −1.

### Conditional state updates

First, the function reads a signed 16-bit value at offset 162. If the lower byte position is smaller, it writes the lower byte position as a 16-bit value at offset 162.

Next, it reads a **32-bit value at offset 164**. It extracts the numeric low 16 bits and interprets them as signed. If that signed value is smaller than the upper byte position, it writes the upper byte position as a **16-bit value beginning at offset 164**.

These accesses have asymmetric widths. The upper comparison does not merely read the two bytes that the upper update writes.

The lower operation extends a signed lower accumulator, absent interference. The upper operation can be interpreted as extending a signed upper accumulator only when the memory layout makes the numeric low half of the 32-bit read correspond to the 16-bit storage beginning at offset 164.

For a conventional little-endian layout, those storage locations correspond. For a conventional big-endian layout, the comparison’s numeric low half comes from offsets 166 and 167, while the update changes offsets 164 and 165. In that case, an update need not change the value used by a later upper comparison.

Neither operation initializes the fields or establishes valid ordering between them.

### Bitmap address resolution and modification

After both conditional state updates, the function reads the address components at offsets 64 and 152. Their 32-bit sum, plus the lower byte position through its 32-bit representation, locates the first bitmap byte.

The representation establishes that these address reads follow the state updates; it does not establish an observable ordering between the two component reads.

The signed distance from lower byte position to upper byte position selects the write pattern. Both positions fit signed 16-bit values, so their difference fits signed 32-bit arithmetic.

**Zero or negative distance**

The function reads the first bitmap byte and writes it back ORed with the intersection of the lower and upper masks. It then returns.

No separate upper-end byte is accessed. Even when the byte positions differ, a negative distance uses this single-byte path.

**Positive distance**

The function performs these operations in order:

1. Reads the first byte and writes it back ORed with the lower mask.
2. Writes `0xff` to each intervening byte in increasing address order.
3. Reads the final byte and writes it back ORed with the upper mask.

A distance of one has no intervening byte writes. Endpoint writes preserve bits outside their masks; intervening writes replace the complete byte.

## Side effects

The established write order is:

1. Optional 16-bit write at offset 162.
2. Optional 16-bit write at offset 164.
3. First bitmap-byte write.
4. Any intervening full-byte writes.
5. Final endpoint-byte write on the positive-distance path.

Every accepted path attempts a bitmap-byte write, including cases where ORing the selected mask leaves its value unchanged.

The function contains no explicit calls, allocations, releases, logging, synchronization, or error reporting. It neither explicitly acquires nor releases ownership of the state or bitmap.

There is no rollback. An invalid access can occur after state updates or earlier bitmap writes have already taken place; recovery and fault delivery are environmental matters.

No non-aliasing guarantee is established. Bitmap writes may modify state storage if the computed addresses overlap it. Address components are read only after the conditional state updates.

The endpoint read-modify-write operations are not established as atomic. Concurrent access can lose bitmap updates or change configuration observations.

## Dependencies

No external function dependency is visible in F-95eefe30cf9e.

Environmental dependencies are:

- The represented 32-bit arithmetic and address model.
- Arithmetic signed right shifts for supported counts.
- Memory endianness and integer storage layout.
- Valid, suitably accessible state and bitmap storage.
- Platform handling of alignment, invalid addresses, and unsupported shifts.

There are no visible unknown callees whose effects on globals or memory ownership need to be assumed.

## Acceptance examples

These are conditional static examples, not executed tests. Unless specified otherwise, assume stable configuration, valid non-overlapping storage, shift count zero, quantum one, and limit at least 64.

### Three bits in one byte

For input start 2, input end 4, and threshold 0, the residual is 1. Both threshold decisions select the alternate calculation. Effective positions are 2 and 4, and both byte positions are zero.

An initially zero first bitmap byte becomes `0x38`. Existing bits outside that mask are preserved.

### Adjacent endpoint bytes

For input start 6, input end 9, and threshold 0, effective positions are 6 and 9.

The first byte is ORed with `0x03`, then the next byte is ORed with `0xc0`. No full intermediate byte is written.

### One full intervening byte

For input start 6, input end 17, and threshold 0, effective positions are 6 and 17.

The first byte is ORed with `0x03`; the next byte is assigned `0xff`; the following byte is ORed with `0xc0`, in that order.

### Threshold equality

For input start 2, input end 4, and threshold 1, the residual equals the threshold. Neither strict comparison selects the alternate calculation.

Both derived positions are 2. The first byte is ORed with `0x20`.

### Rejection and clipping

With limit 8 and derived lower position 8, execution returns without state or bitmap writes if it reaches the limit comparison.

With limit 10, input start 6, input end 20, and threshold 0, the upper position is clipped to 9. The first two bitmap bytes receive endpoint masks `0x03` and `0xc0`.

### Zero limit still permits a write

With limit zero, input start −1, input end zero, and threshold −1, the residual is zero. The derived positions are −1 and zero.

Acceptance is followed by clamping to zero and `0xffffffff`, producing byte positions zero and −1. The nonpositive-distance path ORs the first bitmap byte with `0xff`.

### Upper access width and conditional accumulation

Assume an accepted operation has upper byte position 5. If the 32-bit read at offset 164 has numeric low half 3, the function selects a 16-bit write of 5 at offset 164.

All four bytes at offsets 164 through 167 must be readable even though only two bytes are written.

Under conventional little-endian storage, a later unchanged read sees numeric low half 5. Under conventional big-endian storage, the write changes the other half of the 32-bit field, so its numeric low half remains 3 unless modified separately. Only the former arrangement supports the ordinary upper-accumulator interpretation.

## Uncertainties

- **documented_only — oversized shifts:** Counts from 32 through 255 have no established behavior here.
- **documented_only — actual endianness:** The representation does not independently establish the target layout. Upper accumulation remains conditional on read/write storage correspondence.
- **documented_only — intended domain:** No validated ranges or semantic units are established for inputs, quantum, or threshold.
- **documented_only — storage:** Allocation sizes, initial field values, alignment guarantees, address validity, and permitted aliasing are unknown.
- **documented_only — concurrency:** No synchronization or stable-configuration guarantee is supplied. The listed read order is not a concurrent-memory protocol.
- **documented_only — faults and address realization:** Exact fault behavior and the execution environment’s realization of the represented addresses are not established.

## Evidence

Established static evidence includes access offsets and widths, arithmetic narrowing, signed comparisons, the two threshold decisions, conditional clamping, mask construction, signed byte-position conversion, state-update order, bitmap-write order, and the absence of external calls.

The upper comparison’s full 32-bit read and separate 16-bit write are distinct established operations. Interpreting them as one persistent signed upper accumulator requires a compatible memory layout.

Bitmap range marking and accumulated bounds are inferred intent only. Environmental guarantees remain explicitly unresolved.

No runtime tests have been performed.
