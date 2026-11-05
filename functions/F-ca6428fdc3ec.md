<!-- Trusted provenance: F-ca6428fdc3ec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ca6428fdc3ec

## Purpose

F-ca6428fdc3ec fills caller-supplied storage with blocks of byte-valued lookup tables. Each block contains 16 rows of 256 bytes, totaling 4,096 bytes.

The transformation uses an input address, a signed count, and two separately captured values from one global address-holding location. The first capture selects the lookup-byte base. The second capture selects the base from which an offset-table address is derived. These captures are ordered but are not established to be equal.

Blocks above block zero are constructed in descending block order. Block zero is constructed last and applies an additional input-byte XOR. Block zero is constructed even when the count is zero or negative.

The established behavior is a memory-to-memory table transformation. Any cryptographic or other application-level purpose is inferred intent, not established behavior.

## Interface

### Arguments and representation

The recovered interface has three integer-valued arguments and no return value:

| Argument | Role |
|---|---|
| Count | Signed value controlling how many higher blocks are constructed. |
| Input address | Base address used for input-byte accesses. |
| Output address | Base address used for destination-byte stores. |

There are no length arguments, status outputs, or explicit address-validation operations.

The recovered representation distinguishes signed integers, unsigned integers, and bytes. This contract uses the apparent 32-bit integer model: signed 32-bit count and offset values, unsigned 32-bit column calculations, and integer-valued addresses of that width. The exact machine ABI and original source-language arithmetic rules are not independently established.

Bytes are used as values from 0 through 255. A row offset occupies four bytes in this interpretation. Address calculations mix signed and unsigned operations; the mathematical formulas below describe valid-access results, not a promise that every intermediate calculation has mathematical, unbounded-integer semantics.

### Global inputs: two captures

At entry, the function performs two ordered reads of the same global address-holding location:

1. The first read captures **L**, used as the lookup-byte base.
2. The second read captures **T**, from which the offset-region base **T − 384** is calculated.

Neither captured value is refreshed during construction. The offset-region base is derived from T, not from L.

For row r, where 0 ≤ r < 16, define **O[r]** as the signed integer read at:

**T − 384 + 4r**

This notation identifies an address, not a cached value. The integer is read again for each output byte using that row.

Lookup bytes are read at **L + k**, where k is the generated lookup index.

Under the apparent four-byte integer model, the offset region occupies 64 bytes. It lies 384 bytes before the lookup base only if the two global reads return the same value, so that L = T. No supplied synchronization or stability guarantee establishes that equality. If L and T differ, the two regions have no established relative placement.

### Output layout and capacity

For block b, row r, and column x, the destination byte is at:

**output address + 4,096b + 256r + x**

The row and column ranges are:

- 0 ≤ r < 16.
- 0 ≤ x < 256.

On normal completion, the number of blocks written is count when count is at least 2, and one otherwise. The mathematical destination extent is therefore:

**4,096 × max(1, count) bytes**

This extent is not a validated capacity. It assumes all required arithmetic and accesses are valid.

### Ownership and return

The function does not allocate or release memory, transfer ownership, or expose a status result. The caller must supply usable storage for every executed access.

No function calls appear in the supplied body. There is no callee whose memory ownership or global-preservation behavior must be assumed.

## Behavior

### Entry and retained state

The lookup-base capture occurs before the second global read used to derive the offset-region base. No output stores occur before those captures.

After entry, the construction retains the first lookup base and the separately derived offset-region base. Later changes to the global address-holding location do not redirect either category of access.

This retention does not snapshot memory contents. Offset integers, selected input bytes, and lookup bytes remain read-dependent throughout construction.

### Higher blocks

When count is at least 2, the function processes stage values in descending order:

**count, count − 1, …, 2**

Stage s constructs block s − 1. Within each block, rows proceed from 0 through 15, and columns within each row proceed from 0 through 255.

For each destination byte, the function:

- Reads the row’s signed offset at T − 384 + 4r.
- Uses that offset to read the byte at input address + 16s + O[r].
- XORs the selected byte with column x.
- Reads the lookup byte at L plus that XOR result.
- Stores that lookup byte in block s − 1, row r, column x.

For stable source contents and valid, noninterfering storage, the resulting relation is:

**output[s − 1, r, x] = lookup at L[input[16s + O[r]] XOR x]**

Both XOR operands are between 0 and 255, so the lookup index is also between 0 and 255. No higher-block lookup index requires a byte beyond the first 256 bytes starting at L.

The offset and selected input byte are read inside the column loop. They are not captured once per row. Each normally completed higher block performs 4,096 destination-byte stores and, for each store, reads an offset integer, a selected input byte, and a lookup byte.

### Block zero

Block zero is constructed after every higher block, or immediately when count is at most 1.

Rows again proceed from 0 through 15, with columns from 0 through 255. The selected lookup-index byte for row r comes from:

**input address + 16 + O[r]**

The selected byte is XORed with column x, and the resulting index selects a lookup byte relative to L. That lookup byte is then XORed with the byte at input address + r.

For stable source contents and valid, noninterfering storage:

**output[0, r, x] = lookup at L[input[16 + O[r]] XOR x] XOR input[r]**

The additional XOR applies only to block zero. It is applied to the lookup result, not incorporated into the lookup index.

The byte at input address + r is read for each destination byte. It is not explicitly retained for the entire row. Block zero performs 4,096 destination-byte stores and, for each store, reads an offset integer, a selected input byte, a lookup byte, and the additional input byte.

### Observable ordering

The established destination-store order is:

1. Blocks count − 1 down through 1, if count is at least 2.
2. Block zero.
3. Within each block, increasing row.
4. Within each row, increasing column.

For count 3, all stores to block 2 precede all stores to block 1, which precede all stores to block zero. Destination addresses increase within a block and move backward between successive blocks.

For each higher-block byte, the offset read supplies the selected input address; the selected input byte supplies the lookup index; the lookup read supplies the stored value. Those reads precede their corresponding store.

Block zero has the same offset-to-input-to-lookup dependency chain. Its additional input-byte read must also complete before the store. The supplied expression does not establish whether that additional read occurs before, after, or between reads on the lookup-dependent branch. No stricter ordering is specified here.

The stable-memory formulas do not authorize replacing the construction with a snapshot-based transformation when doing so would change observable results. In particular, caching offsets or input bytes across columns, snapshotting the lookup region, or changing store order can change behavior under overlap.

### Count boundaries

- **Count = 1:** Only block zero is constructed.
- **Count = 0:** Only block zero is constructed.
- **Count < 0:** Only block zero is constructed.
- **Count = 2:** Block 1 is constructed, followed by block zero.
- **Count > 2:** Higher blocks are constructed in descending order, followed by block zero.

A nonpositive count is not a no-op. It still causes the block-zero reads and 4,096 byte stores on normal completion.

The stage value is decremented only while it is greater than 1. Negative counts do not enter that loop, and positive execution stops the higher-block loop at 1. No explicit upper count limit is checked.

### Arithmetic and access requirements

Offsets are signed displacements. They are not restricted to 0 through 15, checked against an input length, or clamped. Negative offsets and offsets outside a nominal 16-byte group are used directly.

Higher-block destination construction includes multiplication of the stage by 4,096 before the final subtraction of 4,096. A mathematically valid final block address does not establish that every recovered intermediate is valid.

Under the apparent signed 32-bit interpretation, stage × 4,096 exceeds the positive signed range for stages greater than 524,287. Address additions can exceed the applicable range at smaller stages, depending on the supplied values. The multiplication by 16 used for selected input addressing also has finite-width constraints.

Column-dependent address expressions include unsigned arithmetic. Consequently, neither universal signed arithmetic nor universal unsigned wrapping is an established model for the entire function. The supplied material does not resolve all out-of-range conversions, overflowing signed calculations, or address-validity consequences.

Normal completion requires:

- Two readable accesses to the global address-holding location.
- Valid derivation of the offset-region base from T.
- Readable offset integers at every executed offset access.
- Readable selected input bytes.
- Readable lookup bytes for every generated index.
- Readable additional input bytes for block zero.
- Writable destination bytes.
- Valid arithmetic and any required alignment for integer accesses.

Integer byte order and alignment requirements are not independently established.

## Side effects

The explicit writes are single-byte destination stores. With valid, stable, disjoint source and destination regions, these stores are the only visible memory modifications in the supplied body.

Overlap is not checked or prevented. Earlier stores can affect later reads when destination storage overlaps:

- Input bytes.
- Offset integers relative to T − 384.
- Lookup bytes relative to L.
- The global address-holding location.

For example, a higher-block write can alter input later used by a lower block. A write during a row can alter the offset or lookup contents used by subsequent columns. If an offset integer is partly overwritten, later reads use the resulting integer representation, subject to the environment’s access rules.

Overwriting the global address-holding location during construction does not refresh either retained base. Lookup accesses continue relative to the first captured value L, and offset accesses continue relative to the base derived from the second captured value T. This remains true whether or not L and T were equal at entry.

There is no explicit transaction, rollback, recovery, or error-reporting path. If execution cannot complete an access, earlier stores may already have taken effect. The precise failure mechanism and recoverability are environmental unknowns.

No synchronization or atomicity contract is shown.

## Dependencies

No calls to other functions are visible. F-ca6428fdc3ec therefore has no identified function dependencies in the supplied body.

Its external dependencies are memory and execution conditions:

- The global location read twice at entry.
- Lookup storage selected by the first capture.
- Offset storage selected by the second capture.
- Caller-provided input and destination storage.
- The execution environment’s arithmetic and memory-access rules.

No preservation, allocation, or release behavior is attributed to an unknown callee.

## Acceptance examples

These are conditional acceptance examples, not executed tests. Except where stated otherwise, assume valid arithmetic, sufficient accessible storage, stable source contents, and no source/destination overlap.

### Different values from the two global reads

Suppose the first global read yields address A and the second yields a distinct address C.

Acceptance requires lookup bytes to be selected relative to A and offset integers to be read at C − 384 + 4r. Using A − 384 for the offsets, or C for lookup bytes, would not match the supplied behavior.

If the global location later changes again, neither retained base changes. This example does not assert a mechanism or synchronization guarantee for producing the two entry values.

### Equal captures and identity lookup

Suppose both global reads return the same value, each lookup entry contains its index, and every offset entry is zero.

For count 2:

- Block 1, row r, column x contains input[32] XOR x.
- Block zero, row r, column x contains input[16] XOR x XOR input[r].
- All stores to block 1 precede all stores to block zero.

The equality assumption permits describing the offset region as 384 bytes before the lookup base in this example only.

### Zero lookup

Suppose every lookup byte selected relative to L is zero.

For any valid count:

- Every constructed higher-block byte is zero.
- Every byte in row r of block zero equals input[r].

For counts at most 1, there are no higher-block stores. The additional block-zero XOR is applied after lookup selection.

### Zero and negative counts

With identical stable sources and the same captured bases, counts 1, 0, and −7 produce the same 4,096 bytes in block zero.

Each execution still reads offsets, selected bytes relative to input + 16, lookup bytes relative to L, and additional bytes input[0] through input[15]. No destination block above zero is written.

### Nonzero and negative offsets

Suppose the row-5 offset read from T − 384 + 20 is 9.

- At stage 3, row 5 selects the input byte at input + 57.
- In block zero, row 5 selects the lookup-index byte at input + 25 and the additional XOR byte at input + 5.

Separately, if row 0 has offset −16 and the resulting accesses are valid:

- Block zero selects its lookup-index byte at input + 0.
- Stage s selects its lookup-index byte at input + 16s − 16.

Neither offset is rejected or normalized.

### Overlapping storage

If a destination store changes a lookup entry that is subsequently read, the subsequent lookup uses the changed contents. An immutable copy of the original lookup region is not an equivalent substitute.

If a store changes the global address-holding location, subsequent accesses nevertheless retain the original L and the offset base derived from the original T.

Cases whose outcome depends on the unspecified relative order of block zero’s two read branches remain documented_only rather than receiving a guessed output.

## Uncertainties

- **documented_only — Application intent:** The broader purpose and meaning of the rows are unknown.
- **documented_only — Entry stability:** No guarantee makes the two ordered global reads return equal values.
- **documented_only — ABI and widths:** The 32-bit interpretation is consistent with the recovered representation but is not independently verified.
- **documented_only — Offset representation:** Integer byte order, required alignment, and intended offset range are unknown.
- **documented_only — Arithmetic failures:** Overflow, out-of-range conversions, and invalid address calculations lack a fully established execution model.
- **documented_only — Read ordering:** Block zero’s additional input read has no established order relative to the lookup-dependent read chain.
- **documented_only — Concurrency:** No synchronization, atomicity, or supported concurrent-mutation policy is supplied.
- **documented_only — Access failures:** Fault delivery, termination, and recoverability are environmental.
- **documented_only — Supported overlap:** The body does not reject overlap, but no external supported-use policy is established.

## Evidence

The supplied static body establishes two ordered reads of the same global location: the first provides the lookup base, and the second supplies the value from which 384 is subtracted for the offset-region base. Neither base is refreshed in the loops. Equality of the captures is not established.

The static body also establishes the descending higher-block sequence, unconditional block-zero construction, increasing row and column order, repeated source accesses, byte stores, XOR transformations, and absence of visible calls or explicit validation.

The block layout, output extent, and stable-memory formulas follow from the address calculations and loop bounds. Overlap sensitivity follows from interleaving repeated reads with destination stores. The finite-width cautions follow from the recovered signed and unsigned operations; they do not establish an original ABI or universal overflow behavior.

No runtime tests have been performed.
