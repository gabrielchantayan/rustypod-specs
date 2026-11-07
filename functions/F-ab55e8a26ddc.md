<!-- Trusted provenance: F-ab55e8a26ddc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ab55e8a26ddc

## Purpose

F-ab55e8a26ddc selects one of two table descriptions within a caller-supplied object and produces two 16-bit outputs for an unsigned index.

The selected table has a declared count of four-byte records. Each such record contains two explicitly big-endian 16-bit quantities. For indices below the declared count, both outputs come from the indexed record. For indices at or above that count, the second output repeats the first quantity of the last declared record, while the first output comes from a subsequent two-byte sequence.

The function performs endpoint comparisons before reading table data. An unavailable initial record produces two zero outputs. An unavailable element in the subsequent sequence produces a zero first output but retains the second output already obtained from the last declared record.

Every normal return has the same 32-bit value, zero. The return value does not distinguish successful extraction, complete fallback, or partial fallback.

These are static observations. The apparent purpose of the layout is a compact representation of paired values followed by values sharing a common companion quantity; the domain meaning of either quantity is not established.

## Interface

### Inputs

The function accepts five arguments:

| Argument | Role | Width and interpretation |
|---|---|---|
| Object address | Base address of the object containing both table descriptions | Address carried through a 32-bit integer in the supplied representation |
| Selector | Chooses the first or second description | 32-bit integer; zero selects the first, every nonzero value selects the second |
| Index | Requested logical element | Unsigned 32-bit integer |
| First-output address | Destination for the first result | Address of a writable 16-bit object |
| Second-output address | Destination for the second result | Address of a writable 16-bit object |

The selector is not validated as an enumeration. Negative selector values select the same description as positive nonzero values.

The index is unsigned. A bit pattern that would represent a negative signed 32-bit integer is instead a large nonnegative index.

### Selected object layout

All offsets below are byte offsets from the original object address.

| Field | Selector equals zero | Selector is nonzero |
|---|---:|---:|
| Declared record count | `0xFA` | `0x14A` |
| Table start address | `0x2A4` | `0x2AC` |
| Table extent | `0x2A8` | `0x2B0` |

The count is read as an unsigned 16-bit native value and widened without sign extension.

The table start and extent are each read as 32-bit values. The table start is used as an address. The extent participates in address addition to obtain the exclusive table endpoint.

The supplied representation types the extent as a signed integer, but does not explicitly reject negative values. Its intended interpretation as a nonnegative byte extent is inferred from its use, not independently validated.

The count and 32-bit metadata fields use the target’s native load representation. Their byte order is not established by the supplied material. This differs from table quantities, whose byte order is explicit.

### Outputs

The first and second destinations receive 16-bit stores. The function returns a 32-bit zero on every normal path.

Neither output is an allocated object or a returned pointer. No ownership transfer, allocation, release, or persistent state update is explicitly present.

### Preconditions

For a defined, normally completing invocation:

- The selected metadata locations must support their respective native-width reads.
- Both output destinations must support 16-bit writes.
- Any table bytes actually reached by a successful endpoint check must be readable.
- Addresses and calculations must have meaningful behavior under the target’s address and arithmetic model.

There is no explicit null-pointer check, alignment check, object-size check, or verification that the table endpoint describes a real allocation.

For ordinary, independent result values, the outputs should not overlap table bytes that remain to be read. Overlap is not checked and can affect the result because a second-output store precedes later table reads.

## Behavior

### 1. Select and read metadata

The selector controls both the metadata locations and their read order.

When the selector is zero, the function reads:

1. The table start at object offset `0x2A4`.
2. The table extent at object offset `0x2A8`.
3. The unsigned 16-bit count at object offset `0xFA`.

When the selector is nonzero, the function reads:

1. The unsigned 16-bit count at object offset `0x14A`.
2. The table start at object offset `0x2AC`.
3. The table extent at object offset `0x2B0`.

Thus, a zero count does not bypass the table-start or table-extent metadata reads.

The exclusive endpoint is then formed by adding the extent to the table start. The static representation does not establish safe behavior for an overflowing address addition.

### 2. Handle a zero count

If the declared count is zero:

1. Store zero to the first output.
2. Store zero to the second output.
3. Return zero.

No table-data bytes are read on this path. The index has no effect.

### 3. Handle an index below the count

For a nonzero count and an index smaller than the count, the selected record starts at the table start plus four times the index.

The function compares the table endpoint with the address immediately after that four-byte record.

- If the endpoint is strictly below that address, the record is rejected. The first output is set to zero, then the second output is set to zero, and the function returns zero.
- If the endpoint is equal to or above that address, extraction proceeds.

The four bytes at the selected record have this meaning:

| Position within record | Meaning |
|---|---|
| Byte 0 | High byte of the second output |
| Byte 1 | Low byte of the second output |
| Byte 2 | High byte of the first output |
| Byte 3 | Low byte of the first output |

The observable sequence is:

1. Read the first two bytes needed for the second output.
2. Store their big-endian decoded value to the second output.
3. Read the next two bytes needed for the first output.
4. Store their big-endian decoded value to the first output.
5. Return zero.

The supplied expression structure does not independently establish which of the two byte operands within a single decoding expression is read first. It does establish the second-output store before the subsequent pair is decoded.

The endpoint check is local to the requested record. This path does not require the whole declared collection of four-byte records to fit within the extent.

### 4. Handle an index at or above the count

For a nonzero count and an index greater than or equal to it, the function first identifies:

- The boundary immediately after all declared four-byte records.
- The beginning of the last declared four-byte record, four bytes before that boundary.
- The excess index, equal to the requested index minus the declared count.

The function first checks whether the table endpoint reaches the boundary after the declared records.

If it does not:

1. Store zero to the first output.
2. Store zero to the second output.
3. Return zero.

If it does:

1. Read the first two bytes of the last declared record.
2. Decode them as a big-endian 16-bit quantity.
3. Store that quantity to the second output.

The candidate first-output quantity is located in the two-byte sequence immediately after the declared four-byte records. Its position within that sequence is the excess index:

- Index equal to the count selects the first two bytes after the declared records.
- Index equal to count plus one selects the next two bytes.
- Each further index advances by two bytes.

A second endpoint comparison checks whether the endpoint reaches immediately past the candidate two-byte quantity.

If that comparison fails:

1. Store zero to the first output.
2. Return zero without another explicit second-output store.

If that comparison succeeds:

1. Read the candidate two bytes.
2. Decode them as a big-endian 16-bit quantity.
3. Store the quantity to the first output.
4. Return zero.

An endpoint exactly at the required boundary is accepted in both checks.

### Integer and address boundaries

The count ranges from zero through 65,535. Consequently:

- Four times the count is at most 262,140.
- On the below-count path, four times the index is at most 262,136.
- On that path, the required endpoint displacement is at most 262,140.

These displacement calculations fit in 32 bits. Adding them to a table address may still overflow the address range.

On the at-or-above-count path, the excess index can be much larger. With a nonzero count, its maximum mathematical value is 4,294,967,294. Doubling that value need not fit in 32 bits.

The supplied representation also places this difference in a signed integer temporary before using it in address calculations. It does not establish a portable signed-overflow policy or enough machine-level detail to define all extreme-index outcomes.

Accordingly, the linear layout description above is established for nonoverflowing address calculations. Behavior involving signed-range crossings, multiplication overflow, or address wraparound is documented_only rather than assigned a guessed wraparound or rejection rule.

## Side effects

### Writes and their order

There are three normal output patterns:

| Outcome | Explicit store order |
|---|---|
| Zero count or failed initial endpoint check | First output becomes zero, then second output becomes zero |
| Complete extraction | Second output receives its decoded quantity, then first output receives its decoded quantity |
| Subsequent-sequence endpoint failure | Second output receives the last record’s decoded quantity, then first output becomes zero |

No output write precedes the metadata reads.

No explicit write targets the object or table. However, caller-provided output addresses can overlap them, so output stores can modify those regions.

### Aliasing

Output aliasing is not rejected.

If both destinations identify the same 16-bit storage:

- Complete extraction leaves the first-output quantity there because its store occurs last.
- A subsequent-sequence failure leaves zero there, despite the earlier second-output store.
- Complete fallback leaves zero there.

If the second destination overlaps bytes still needed for the first output, its store can change the bytes subsequently decoded. This applies both to an ordinary four-byte record and to the subsequent two-byte sequence.

Partial overlap between the output destinations can make final bytes depend on native store byte order and the overlap geometry. The function does not provide an independent-copy guarantee.

### Failure behavior

Insufficient table extent is represented through output values, not through a distinct return code.

Zero output values are ambiguous: they can be decoded data or fallback values. A nonzero second output paired with a zero first output can likewise be either valid data or a subsequent-sequence bounds failure.

Invalid addresses, inaccessible memory, unsupported alignment, and invalid arithmetic have no explicit recovery path in the supplied material. No normal return or completed output state is established for such cases.

## Dependencies

No external function invocation is visible in F-ab55e8a26ddc. No additional opaque function IDs are therefore required for its described execution.

The behavior depends on:

- Native 16-bit and 32-bit metadata reads.
- Individual table-byte reads.
- Native 16-bit output stores.
- Target address addition and pointer-order comparisons.

These are execution-model dependencies, not evidence of validation, memory ownership, or fault handling by another function.

## Acceptance examples

The following are conditional acceptance cases derived from static evidence, not executed tests. Unless stated otherwise, assume valid accessible memory, nonoverlapping outputs, stable metadata and table contents, and nonoverflowing address arithmetic.

### 1. Zero count ignores table contents

With selected count zero and any index, both outputs become zero and the return value is zero.

The selected table-start and extent fields are still read. No table-data read is required.

### 2. An ordinary record uses explicit big-endian decoding

With count two, index zero, and a selected record containing bytes `12 34 AB CD`, an extent reaching at least the end of that record produces:

- First output: `0xABCD`.
- Second output: `0x1234`.
- Return value: zero.

The second output is stored before the bytes for the first output are decoded.

### 3. A truncated ordinary record produces complete fallback

With count two and index one, the selected record requires the endpoint to reach eight bytes past the table start.

An extent of seven bytes produces two zero outputs. An extent of eight bytes admits the record.

Neither case requires access to bytes beyond that selected record.

### 4. The first subsequent element reuses the last record’s first quantity

With count two and index two:

- The last declared record starts four bytes past the table start.
- The first subsequent quantity starts eight bytes past the table start.

If the last record begins with bytes `01 02`, the subsequent quantity contains bytes `CA FE`, and the extent is ten bytes, the outputs are:

- First output: `0xCAFE`.
- Second output: `0x0102`.

### 5. Missing subsequent data retains an already stored companion quantity

Using count two and index two again, let the extent be eight bytes and let the last declared record begin with `01 02`.

The declared-record boundary check succeeds. The subsequent-quantity check fails.

The second output is first set to `0x0102`; the first output is then set to zero. The return remains zero.

An extent of nine bytes has the same outcome: a single available byte is insufficient for the two-byte quantity.

### 6. An incomplete declared-record region produces complete fallback

With count two, any index at or above two, and extent seven, the initial check for the declared-record boundary fails.

Both outputs become zero. The function does not read the last record’s first quantity on this path.

### 7. Selector values are classified only as zero or nonzero

For identical arguments other than the selector, values one and negative one select the same metadata locations. Zero selects the alternate locations.

No out-of-range selector error is produced.

### 8. Identical output destinations expose the final store

For the successful ordinary-record example above, if both output arguments address the same 16-bit object, that object ends as `0xABCD`, not `0x1234`.

For the subsequent-data failure example, identical destinations end as zero.

## Uncertainties

The following remain documented_only:

- **Semantic meaning:** The domain meanings of the two quantities and the reason for the compact layout are unknown.
- **Native representation:** Metadata byte order, output-store byte order, and alignment requirements are not specified.
- **Extreme arithmetic:** Overflow, signed-range crossings, address wraparound, and pointer comparisons outside an ordinary valid region are not fully established.
- **Allocation validity:** An endpoint comparison does not establish that the intervening addresses belong to a readable allocation.
- **Concurrency and special memory:** No synchronization, atomicity, volatile-access contract, or coherent-snapshot guarantee is shown.
- **Exceptional completion:** Fault behavior and partially completed stores after invalid memory access are not specified.
- **Caller aliasing rules:** The implementation does not check overlap, but no broader caller contract establishing whether overlap is permitted was supplied.

These unknowns do not alter the established normal-path store order, field widths, explicit table-byte decoding, or zero return value.

## Evidence

This contract is based solely on static inspection of the supplied representation of F-ab55e8a26ddc.

Established evidence includes the selector-dependent metadata offsets and read order, unsigned 16-bit count, unsigned index comparison, four-byte record spacing, two-byte subsequent spacing, explicit high-byte/low-byte decoding, endpoint comparisons, output-store order, absence of visible external calls, and constant normal return value.

The interpretation as paired records followed by a shared-companion sequence is inferred intent, not an independently documented format definition.

No runtime tests have been performed. No target ABI documentation, machine-level overflow analysis, caller contract, or memory-allocation evidence was supplied.
