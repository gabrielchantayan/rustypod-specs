<!-- Trusted provenance: F-ee799ab58b1f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ee799ab58b1f

## Purpose

F-ee799ab58b1f converts an input byte sequence into four-byte output groups using a lookup table. It processes three input bytes per complete group and uses the literal byte `0x3D` for padding when the final group contains only one or two input bytes. It appends a zero byte and returns the number of output bytes preceding that terminator.

The grouping and index calculations are consistent with Base64 encoding. However, the supplied evidence does not establish the lookup table’s contents. Calling the result standard Base64 is therefore conditional on the table containing the standard Base64 alphabet in the expected order.

The function performs no visible validation, allocation, capacity checking, or error reporting.

## Interface

The function accepts three arguments:

| Argument | Interpretation |
|---|---|
| Destination address | Address at which output bytes are written. |
| Source address | Address from which unsigned input bytes are read. |
| Input length | Signed 32-bit count controlling input processing. |

The return value is represented as a signed 32-bit integer.

The supplied representation uses:

- Eight-bit input and output bytes.
- Unsigned 32-bit intermediate values for shifts, masks, and table indices.
- A signed 32-bit remaining-input count.
- A signed 32-bit accumulated output count.
- A global value used as the lookup-table base address.

The global table-base value is loaded once on entry. All subsequent table accesses use that captured value.

For a positive input length \(n\), define:

\[
g=\left\lceil \frac{n}{3}\right\rceil
\]

Subject to valid memory and representable arithmetic:

- The source supplies \(n\) input bytes.
- The destination receives \(4g\) encoded bytes.
- A zero terminator is written at destination offset \(4g\).
- The return value is \(4g\).
- The destination therefore needs at least \(4g+1\) writable bytes.

For zero or negative input lengths:

- No source bytes or lookup-table entries are read.
- One zero byte is written at the destination address.
- The return value is zero.
- The global table-base value is nevertheless loaded.

There is no destination-capacity argument. Sufficient writable storage is a caller responsibility.

For an ordinary non-overlapping invocation, the lookup table must provide readable entries at every index actually used. All generated indices are between 0 and 63 inclusive. A readable 64-byte table is sufficient for every possible input.

## Behavior

### Entry and loop control

Before considering the input length, the function captures the global lookup-table base and initializes its output count to zero.

Processing continues while the remaining-input count is strictly positive. Each iteration emits exactly four output bytes. The logical source position advances by three bytes, the destination position advances by four bytes, and the remaining-input count decreases by three.

Consequently, a final one-byte or two-byte group is processed once and then terminates the loop. The remaining count becomes negative rather than being clamped to zero.

After all groups, including when there were no groups, the function writes a zero byte at the current destination position and returns the accumulated output count. The terminator is not included in that count.

### Complete three-byte groups

For three input bytes with unsigned values \(a\), \(b\), and \(c\), the intended four table indices are:

| Output offset within group | Table index |
|---|---|
| 0 | \(\lfloor a/4\rfloor\) |
| 1 | \(16(a\bmod 4)+\lfloor b/16\rfloor\) |
| 2 | \(4(b\bmod 16)+\lfloor c/64\rfloor\) |
| 3 | \(c\bmod 64\) |

Each indexed table entry contributes exactly one byte. No interpretation of table bytes as characters is required by the function.

The supplied expression order exposes an additional detail: the first source byte is read twice for a complete group.

The represented access sequence is:

1. Read the first source byte and retain its contribution to the second output index.
2. Read the second source byte.
3. Read the third source byte.
4. Read the first source byte again to determine the first output index.
5. Read the first selected table entry and write output offset 0.
6. Read the second selected table entry and write output offset 1.
7. Read the third selected table entry and write output offset 2.
8. Read the fourth selected table entry.
9. Write that fourth selected byte at output offset 3.

The first byte’s initial value determines its contribution to output offset 1; its later value determines output offset 0. With stable source memory, these reads agree. The evidence does not justify collapsing them into one observable read in a setting where reads themselves matter.

The second and third source-byte values are captured before any output byte for that group is written.

### Final two-byte group

For final input bytes \(a\) and \(b\), the output consists of:

1. Table entry \(\lfloor a/4\rfloor\).
2. Table entry \(16(a\bmod 4)+\lfloor b/16\rfloor\).
3. Table entry \(4(b\bmod 16)\).
4. Literal byte `0x3D`.

Only two source bytes are read. The function does not read an absent third byte.

Both source-byte values are captured before the first output write for this group. The three table reads and their corresponding output writes occur in output order. The fourth output byte is padding, not a lookup-table entry.

The third index depends on unsigned 32-bit shift behavior. A left shift discards bits beyond the 32-bit width, and the following logical right shift yields \(4(b\bmod16)\). Treating that intermediate as an unbounded integer would give an incorrect result.

### Final one-byte group

For a final input byte \(a\), the output consists of:

1. Table entry \(\lfloor a/4\rfloor\).
2. Table entry \(16(a\bmod 4)\).
3. Literal byte `0x3D`.
4. Literal byte `0x3D`.

Only the one available source byte is read. No second or third source byte is accessed.

The source byte is captured before any output write for the group. The first table entry is read and written before the second table entry is read and written. Padding bytes then occupy output offsets 2 and 3.

### Termination and returned size

There is one final zero-byte write after the group loop. No newline, separator, or additional length field is emitted.

For stable, non-overlapping memory, the function does not inspect source bytes beyond the stated positive input length. It still advances its internal source position by three after a partial final group. That advanced position is not subsequently dereferenced.

A zero byte appearing in the lookup table does not stop encoding. Such a byte is written as ordinary output, and processing continues. The returned count therefore describes emitted bytes, not necessarily the length observed by a consumer that stops at the first zero byte.

## Side effects

### Memory reads and writes

The visible reads are:

- One read of the global lookup-table base at entry.
- Source-byte reads required by each group, including the additional first-byte read for every complete group.
- Four table-entry reads for a complete group.
- Three table-entry reads for a two-byte final group.
- Two table-entry reads for a one-byte final group.

The visible writes are:

- Four destination-byte writes per processed group.
- One final zero-byte write.

The function contains no explicit assignment to a separate global state object. Nevertheless, destination aliasing can change global storage, source storage, or table storage indirectly.

### Aliasing and access order

No overlap check appears in the supplied evidence.

If destination writes overlap input needed by a later group, later source reads can observe the modified bytes. The function therefore does not establish general in-place encoding support.

Within one group, the source values needed for that group are read before its first output write. This limits same-group interference with source reads, but it does not protect subsequent groups.

Table entries are not copied into a private table. Each output lookup reads through the captured table base when that output is computed. If earlier destination writes overwrite table entries needed later, later lookups can observe the changed table bytes, including within the same group.

If destination writes overwrite the storage holding the global table-base value, this invocation continues using the base captured at entry. Such writes can still alter that global storage for other uses or later invocations.

These aliasing cases are not rejected, but their results need not match an encoding calculated from immutable input and table snapshots. Non-overlapping source, destination, and lookup-table storage provide the straightforward contract.

### Ownership and failures

No allocation or release operation is visible. The function uses caller-supplied addresses and does not visibly transfer ownership.

There are no explicit failure returns. Invalid addresses, insufficient destination capacity, unavailable table entries, or problematic arithmetic are not converted into a documented error code.

The function does not provide rollback. If an access fails after earlier output writes, the evidence supplies no guarantee that the destination remains unchanged or contains a complete terminated result.

## Dependencies

No function calls are visible in F-ee799ab58b1f. There are therefore no established callable dependencies requiring additional opaque function IDs.

Its external data dependency is the global lookup-table base:

- The base is captured once per invocation.
- Entries are read as single bytes.
- Every computed entry index is within 0 through 63.
- Alphabet contents, initialization, accessibility, and stability are not established by the supplied function.
- The literal padding byte `0x3D` does not come from the table.
- The final zero byte does not come from the table.

No assumptions about unknown callees preserving globals or managing memory are needed: no callees are shown.

## Acceptance examples

The following are conditional acceptance cases derived from static evidence, not observed runtime results. Unless stated otherwise, they assume stable, readable input and table storage; sufficient writable destination storage; non-overlapping regions; and valid address arithmetic.

### Nonpositive lengths

For input length zero:

- No source byte or table entry is read.
- The destination’s first byte becomes zero.
- The return value is zero.
- The global table-base value is still read.

The same behavior applies to a negative length, including the minimum signed 32-bit value. No decrement is performed because the loop is not entered.

### One-byte input

For input byte `0xFF`:

- The first two table indices are 63 and 48.
- Output offsets 2 and 3 are both `0x3D`.
- Output offset 4 is zero.
- The return value is 4.

The actual first two output-byte values depend on the table.

### Two-byte input

For input bytes `0xFF`, `0xFF`:

- The table indices are 63, 63, and 60.
- Output offset 3 is `0x3D`.
- Output offset 4 is zero.
- The return value is 4.

This case distinguishes the required 32-bit truncating shift calculation from an incorrect unbounded-integer calculation.

### Complete group

For three input bytes all equal to `0xFF`:

- All four table indices are 63.
- No padding byte is inserted by the function.
- A zero byte follows the four selected table bytes.
- The return value is 4.

A selected table entry could itself equal `0x3D`; that would be a table result, not inserted padding.

### Standard-alphabet condition

If the lookup table is exactly the standard Base64 alphabet, in standard index order:

| Input bytes interpreted as ASCII | Output bytes before terminator | Return value |
|---|---|---|
| `M` | `TQ==` | 4 |
| `Ma` | `TWE=` | 4 |
| `Man` | `TWFu` | 4 |
| `ManM` | `TWFuTQ==` | 8 |

Each output is followed by one zero byte. Without the stated table condition, these textual outputs are not established.

### Embedded output zero

If every referenced table entry contains zero and the input is one byte long:

- The first two output bytes are zero.
- The next two output bytes are `0x3D`.
- The final terminator is also zero.
- The return value remains 4.

An early zero does not suppress later writes.

## Uncertainties

### Established static evidence

The supplied representation establishes the grouping, index calculations, padding bytes, terminator, loop condition, visible access sequence, and absence of explicit bounds checks or function calls.

It also establishes a signed 32-bit output accumulator. A nonnegative mathematical output count must fit that representation for the returned-length contract to hold without overflow.

The largest positive input length satisfying this requirement is:

\[
n=1{,}610{,}612{,}733
\]

It produces 536,870,911 groups and a returned count of 2,147,483,644. Larger positive lengths require an output count exceeding the signed 32-bit maximum.

This numerical limit does not establish that such large buffers or address ranges are available.

### Inferred intent

Base64-style encoding is the inferred intent. Standard Base64 compatibility additionally requires the expected alphabet.

The final zero byte suggests intended use with zero-terminated byte strings. That interpretation is conditional because table entries may themselves contain zero.

### documented_only

The following are not resolved by the supplied evidence:

- The actual lookup-table contents.
- How the global table base is initialized or synchronized.
- The runtime consequence of signed output-count overflow.
- Memory-fault behavior and recovery.
- The validity of extreme address calculations or source-position advances.
- Any promised behavior under concurrent source or table modification.
- Any caller guarantee prohibiting or permitting overlap.
- Whether generated machine accesses preserve every represented read as a distinct physical operation.

No overflow wrapping, fault handling, concurrency safety, or arbitrary-overlap guarantee is asserted.

## Evidence

This contract is based solely on the supplied static representation of F-ee799ab58b1f.

The relevant evidence consists of the entry-time global-base load, signed positive-length loop, unsigned byte reads and 32-bit bit operations, indexed byte lookups, four-byte group writes, literal padding bytes, final zero-byte write, and signed output-count return.

The representation contains no visible calls, allocation, release, capacity check, validation branch, or explicit error result.

No runtime tests have been performed. The acceptance examples are conditional deductions, not execution reports.
