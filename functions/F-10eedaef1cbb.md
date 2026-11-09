<!-- Trusted provenance: F-10eedaef1cbb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-10eedaef1cbb

## Purpose

F-10eedaef1cbb transforms a caller-provided 16-byte state in place. The state is treated as four rows of four bytes, with columns processed consecutively.

A 32-bit selector chooses between two table-driven transformations:

- **Zero:** four lookup regions are addressed relative to a shared anchor.
- **Nonzero:** sixteen 256-byte tables are addressed relative to a caller-provided base.

Each completed output byte is the exclusive OR of four lookup values. Before modifying a column, the routine preserves its four input bytes in private temporary storage. In nonzero mode, preservation is interleaved with clearing the corresponding state bytes; the results are then accumulated into those cleared destinations.

The table arrangement suggests a column-oriented byte-mixing operation. No particular cryptographic algorithm, mathematical table construction, or inverse relationship between the modes is established.

## Interface

The represented interface has four arguments and no return value.

| Argument | Represented width | Contract |
|---|---:|---|
| Selector | Signed 32-bit integer | Zero selects shared-anchor mode; every nonzero value selects caller-table mode. |
| Caller-table address | 32-bit integer used as an address | Base of the lookup storage used in nonzero mode. Unused in zero mode. |
| State address | 32-bit integer used as an address | Base of the 16 bytes read and modified in place. |
| Initial scratch value | Unsigned 32-bit integer | Initializes a private four-byte temporary. All four bytes are overwritten before being used in either transformation. |

The scratch argument has no effect on the represented state transformation. That fact does not establish that an external calling convention permits omitting it.

### State layout

Let \(S[r,c]\) denote the byte at state-base offset \(4r+c\), for row and column indices from 0 through 3.

Each row occupies four consecutive bytes. Column \(c\) comprises offsets \(c\), \(4+c\), \(8+c\), and \(12+c\). Columns are processed in order 0, 1, 2, 3; output rows within a column are also processed in ascending order.

### Caller-table layout

In nonzero mode, define:

\[
L[r,j,x]=\operatorname{byte}(T+1024r+256j+x),
\]

where \(T\) is the caller-table base, \(r\) is the output row, \(j\) is the preserved input position, and \(x\) is an unsigned byte.

The layout contains four output-row groups, each containing four consecutive 256-byte tables. Its possible lookup offsets range from 0 through 4095. A readable 4096-byte region supplies every possible lookup, although a particular invocation accesses only the entries selected by its state.

### Shared-anchor layout

For an anchor value \(G\), define four lookup regions:

| Region | Address of entry \(x\) |
|---|---|
| \(A_G[x]\) | \(G+x\) |
| \(B_G[x]\) | \(G-512+x\) |
| \(C_G[x]\) | \(G-256+x\) |
| \(D_G[x]\) | \(G-768+x\) |

For a fixed anchor, these regions occupy the contiguous span from \(G-768\) through \(G+255\). These labels specify addresses, not table contents.

The routine does not use one anchor snapshot for all four regions. Its distinct anchor references are part of the behavior described below.

### Preconditions and ownership

Successful completion requires readable and writable state storage and readable lookup bytes at every address actually accessed. Zero mode additionally requires readable shared-anchor storage.

There are no explicit null, bounds, alignment, or address-overflow checks. For the ordinary valid-memory contract, all address calculations must resolve to the intended accessible storage without relying on wrapping arithmetic.

No allocation, release, ownership transfer, or retention of caller storage is shown. State storage is modified in place. Lookup storage is not explicitly written in its role as lookup storage, but overlap with the state can change its contents through state writes.

## Behavior

### Common input preservation

For each column, the routine copies the four state bytes into temporary byte offsets 0 through 3 in ascending row order. Every temporary byte is replaced before any lookup uses that column’s temporary.

Let \(t_0,t_1,t_2,t_3\) denote those bytes by memory offset. Let \(q\) denote the low eight bits of the temporary when interpreted as a 32-bit value.

The distinction matters only in zero mode: that mode uses \(q\) for one input position and explicit temporary byte offsets for the other three. On a little-endian interpretation, \(q=t_0\). The supplied evidence does not independently establish target endianness, so that equality is conditional.

Under valid-memory conditions, all four columns are processed. There is no early-success exit or returned status.

### Zero mode: complete shared-anchor reference sequence

Each column has five distinct source-level references to the shared anchor. Their sequence and uses are:

| Reference | Position in the column’s processing | Use |
|---|---|---|
| Captured anchor | Before any of that column’s four state-input reads | Supplies the \(A\)-region base for output rows 1, 2, and 3. |
| B anchor | After all four input bytes have been collected | Independently reads the shared anchor and derives the base at anchor minus 512. |
| C anchor | After the B-base derivation | Independently reads the shared anchor and derives the base at anchor minus 256. |
| D anchor | After the C-base derivation | Independently reads the shared anchor and derives the base at anchor minus 768. |
| Row-0 A anchor | During evaluation of the row-0 output expression | References the shared anchor again for that output’s \(A\)-region lookup. |

Thus the post-collection base derivations are ordered **B, then C, then D**, and each uses its own source-level anchor reference. They are not three offsets derived from the earlier captured value.

The row-0 anchor reference belongs to the row-0 XOR expression. Its position relative to the other operand reads within that expression is not established. The table above specifies source-level reference sequencing across the surrounding stages, not a guaranteed number or order of machine load instructions.

After deriving the B, C, and D bases, the routine computes and writes output rows 0, 1, 2, and 3 in that order. There are no preliminary zero writes in this mode.

### Zero mode: output values

Let \(G_0\) be the anchor captured before input collection; let \(G_B,G_C,G_D\) be the independently obtained values used for the B, C, and D base derivations; and let \(G_A\) be the anchor value referenced within the row-0 expression.

For the selected lookup contents, the output equations are:

\[
\begin{aligned}
S'[0,c] &=
 A_{G_A}[q]\oplus B_{G_B}[t_1]\oplus C_{G_C}[t_2]\oplus D_{G_D}[t_3],\\
S'[1,c] &=
 D_{G_D}[q]\oplus A_{G_0}[t_1]\oplus B_{G_B}[t_2]\oplus C_{G_C}[t_3],\\
S'[2,c] &=
 C_{G_C}[q]\oplus D_{G_D}[t_1]\oplus A_{G_0}[t_2]\oplus B_{G_B}[t_3],\\
S'[3,c] &=
 B_{G_B}[q]\oplus C_{G_C}[t_1]\oplus D_{G_D}[t_2]\oplus A_{G_0}[t_3].
\end{aligned}
\]

Here \(\oplus\) is bytewise exclusive OR. Lookup values are read while each output is evaluated, not captured as a complete table snapshot.

If the anchor remains stable, all five anchor values agree and the expressions become cyclic arrangements of four fixed lookup regions. If the anchor changes, the expressions may combine regions based on different anchor values. Snapshot consistency must not be assumed.

### Nonzero mode

The selector’s magnitude and sign have no effect beyond selecting this branch.

Input preservation and clearing are interleaved by row: each original state byte is copied into the corresponding temporary offset, then that same state byte is set to zero before the next row’s input is collected. All four destination bytes in the column have therefore been cleared before accumulation starts.

Output rows are processed from 0 through 3. For each output row, input positions are processed from 0 through 3. Each contribution uses the corresponding temporary byte as an unsigned table index, reads the selected table byte and the current destination byte, XORs them, and writes the resulting byte back to that destination.

Each destination receives four accumulation writes after its initial clear. With stable lookup contents and no interference, its final value is:

\[
S'[r,c]
=
L[r,0,t_0]\oplus L[r,1,t_1]\oplus
L[r,2,t_2]\oplus L[r,3,t_3].
\]

This mode uses temporary bytes by memory offset rather than extracting the numeric low byte of the temporary word. Its input ordering therefore does not depend on that word’s endianness.

The relative evaluation order of the lookup-byte read and destination-byte read within one XOR expression is not established.

### Widths and arithmetic

State bytes, lookup values, and lookup indices are eight-bit quantities. Indices range from 0 through 255; bytes with their high bit set do not become negative indices.

In caller-table addressing:

- The greatest output-row contribution is \(3\times1024=3072\).
- The greatest input-table contribution is \(3\times256=768\).
- Adding the greatest byte index gives \(3072+768+255=4095\).

State offsets range from 0 through 15. Shared-table displacements are 0, −512, −256, and −768.

XOR introduces no carry or saturation. Results are written as bytes. The address expressions use the represented 32-bit domain; portable behavior for overflowing address calculations is not established.

### Aliasing and ordering limits

No disjointness check is present. If lookup storage overlaps the state, a state write can change a later lookup result. In nonzero mode, the four clears can change lookup contents before any accumulation begins.

All destination writes for one column precede input collection for the next column. Consequently, overlap or external mutation can also make later columns depend on earlier writes.

The formulas describe the selected reads and their XOR relationships. They do not authorize replacing ordered execution with an immutable snapshot of all initial lookup storage. Nor do source-level access sequences establish atomicity, synchronization, or externally observable machine-load ordering.

## Side effects

The explicit external effects are state-memory reads and writes, lookup-byte reads, and, in zero mode, shared-anchor references.

For ordinary nonoverlapping storage, the represented access counts for a complete invocation are:

| Mode | State-byte reads | State-byte writes | Lookup-byte reads |
|---|---:|---:|---:|
| Zero | 16 | 16 | 64 |
| Nonzero | 80 | 80 | 64 |

Nonzero mode’s state reads consist of 16 input-preservation reads and 64 accumulation reads. Its writes consist of 16 clears and 64 accumulation writes.

Zero mode additionally contains five source-level shared-anchor references per column, or twenty across four columns. These are representation-level counts, not guarantees about generated instructions, bus transactions, or volatile-memory semantics.

There are no explicit shared-anchor writes, function calls, logging operations, synchronization operations, allocations, or releases.

The routine has no error-reporting branch. Inaccessible or otherwise invalid storage can fail according to the execution environment. No rollback is shown: an interrupted invocation can leave completed earlier columns, partially updated current outputs, or cleared but not fully accumulated bytes.

## Dependencies

No function calls appear in the supplied body; there are no callee dependencies requiring additional opaque function IDs.

The behavioral dependencies are:

- The caller’s 16-byte state storage.
- Caller-provided lookup storage in nonzero mode.
- Shared-anchor storage and selected lookup addresses in zero mode.
- A private four-byte temporary.
- The execution environment’s address and byte-order interpretation.

Zero mode does not use the caller-table address. Nonzero mode does not reference the shared anchor.

Initialization, ownership, lifetime, and concurrent mutation rules for the shared anchor and lookup storage are not supplied. No unknown callee is assumed to preserve globals or own or release memory; none is shown.

## Acceptance examples

These examples are conditional consequences of static evidence, not runtime observations.

### Caller-table addressing and unsigned indices

For a nonzero selector, suppose one column’s preserved bytes are \(2,127,128,255\), with valid, stable, nonoverlapping storage.

Output row 2 reads caller-table offsets:

\[
2050,\quad2431,\quad2688,\quad3071.
\]

Its final byte is the XOR of those four values. Indices 128 and 255 remain positive unsigned byte indices.

### Visible accumulation sequence

In nonzero mode, suppose a destination’s four selected table values are hexadecimal 01, 02, 04, and 08.

After all four bytes of that column have been cleared, that destination’s accumulation writes are 01, 03, 07, and 0F, in that order. Its original value contributes only through selection of preserved-input lookup entries.

### Distinct anchor references

Consider a zero-mode column under an environment that permits defined changes to the shared anchor between source-level stages.

Suppose the captured value is \(G_0\), followed by distinct values \(G_B,G_C,G_D\) at the B, C, and D derivations, and \(G_A\) at the row-0 anchor reference. The output lookups must follow the distinct-anchor equations above: rows 1–3 use \(G_0\) for their A lookups, while row 0 uses \(G_A\).

An implementation that derives all bases from \(G_0\), or reads the anchor once after input collection for all three derived bases, does not reproduce this represented behavior. This example does not impose an order among the row-0 expression’s operand reads.

### Stable-anchor cyclic placement

Assume a stable anchor, stable nonoverlapping lookup storage, and little-endian temporary interpretation. Let every A entry equal its index and every B, C, and D entry be zero.

A column containing hexadecimal 11, 22, 33, and 44 from row 0 through row 3 finishes with those same values in the same rows.

### Selector and scratch equivalence

With identical initial memory and stable dependencies, selectors 1, −1, and the greatest positive signed 32-bit value produce the same nonzero-mode behavior.

Changing only the initial scratch value does not change either mode’s represented state transformation, because all four temporary bytes are overwritten before use.

### Aliased lookup entry cleared before use

In nonzero mode, suppose a selected lookup entry occupies a state byte cleared during the current column’s input collection, with no intervening write before that lookup.

That lookup observes the cleared value rather than the byte’s invocation-entry value. Computing from an immutable initial table copy is not an equivalent acceptance criterion.

### Invalid storage

If a required address is inaccessible, no particular returned error, complete output, or atomic failure is promised. Earlier writes may remain visible. Successful recovery is not established.

## Uncertainties

The following remain **documented_only**:

- Target endianness and the relationship between \(q\) and temporary byte offset 0.
- Shared-anchor initialization, lifetime, stability, and concurrency rules.
- Lookup contents, mathematical meaning, and any relationship between modes.
- Whether overlapping state, lookup, or shared-anchor storage is intended usage.
- Machine-level realization of source-level anchor references.
- Operand-read ordering within compound XOR expressions, including the row-0 anchor reference relative to other operands.
- Address-wrap, fault, alignment, volatile-memory, and memory-mapped I/O semantics.
- Atomicity and visibility guarantees for concurrent observers.

These unknowns do not erase the established source-level distinction between the initial capture, ordered independent B/C/D anchor references, and the additional row-0 anchor reference.

## Evidence

**Established static evidence:** the supplied body selects on zero versus nonzero, processes four columns, preserves four input bytes per column, uses the two stated lookup layouts, and performs ordered in-place writes. Zero mode contains the five distinct shared-anchor references described above. Nonzero mode interleaves preservation with clearing and then performs four accumulations per output byte. No calls or explicit error handling appear.

**Inferred intent:** the layouts suggest a table-driven column-mixing transform. This interpretation adds no algorithm-specific guarantees.

**Unknown:** platform semantics and external storage contracts are not supplied beyond the represented widths and accesses.

No runtime tests have been performed. All acceptance examples are derived conditions rather than measured results.
