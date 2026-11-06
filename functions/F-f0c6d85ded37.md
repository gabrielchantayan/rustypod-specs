<!-- Trusted provenance: F-f0c6d85ded37; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f0c6d85ded37

## Purpose

F-f0c6d85ded37 expands position-dependent input bytes into groups of byte lookup tables. Each group contains 16 tables of 256 bytes, occupying 4,096 bytes in total.

For ordinary positive group counts, every table maps an 8-bit index through a shared byte lookup table after XORing the index with a selected input byte. The last group additionally XORs each result with a second input byte associated with that table position.

The source establishes this transformation, its traversal order, and its memory accesses. It does not establish an application-level purpose, such as encryption, key expansion, or encoding. Those interpretations are not part of this contract.

## Interface

### Inputs

The function accepts three integer-represented arguments:

| Argument | Meaning |
|---|---|
| Group count, \(N\) | Controls the number of ordinary groups and identifies the location of the mandatory final group. Comparisons involving this argument are signed. |
| Input address, \(S\) | Base address used for selected-byte and final-mask reads. |
| Output address, \(D\) | Base address used for byte writes. |

The supplied representation uses signed integers for the arguments and address calculations, unsigned integers for the innermost index and XOR intermediate, byte loads, and four-byte spacing between offset entries.

A 32-bit integer/address interpretation is consistent with this representation, but the supplied material does not independently establish the target ABI or its overflow rules. Width-sensitive requirements below distinguish the explicit byte and offset layout from assumptions about address arithmetic.

### Shared data

The function obtains its lookup and offset-table addresses from shared state before entering its loops:

- One evaluation supplies a captured lookup-table base, \(L\).
- A separate evaluation supplies a base from which 64 bytes are subtracted to obtain the captured offset-table base, \(T\).

When the shared value remains stable during these evaluations, \(T=L-64\). The source does not establish a single atomic snapshot covering both evaluations.

The offset table has 16 entries, addressed at:

\[
T+4j,\qquad 0\leq j<16.
\]

Each entry is read as an integer displacement. Negative displacements are not excluded by the source.

The lookup table is accessed by byte at:

\[
L+q,\qquad 0\leq q<256.
\]

Consequently, a stable shared base convention places a 64-byte offset region immediately before a 256-byte lookup region. Entry byte order and exact signed-integer representation depend on the target.

### Outputs and return value

The function has no explicit return value. Its observable result is a sequence of byte writes through the output address.

For positive \(N\), with valid non-overflowing address arithmetic, it writes exactly \(4096N\) bytes spanning:

\[
[D,\ D+4096N-1].
\]

No output buffer length, input length, or status destination is provided.

### Preconditions for a well-defined ordinary invocation

The caller must provide:

- Readable shared base state.
- Readable offset entries wherever execution reaches them.
- Readable selected input bytes and final-mask bytes.
- Readable lookup bytes at the indices actually used.
- Writable destination bytes for every generated table entry.
- Address arithmetic and integer operations that are valid under the execution environment.
- Suitable alignment for integer offset loads, unless the target permits unaligned access.

Positive \(N\) is the natural precondition for treating \(D\) as the beginning of an \(N\)-group output buffer. The function does not enforce that precondition.

There is no explicit requirement in the source that the input, output, lookup table, offset table, or shared base storage be disjoint. Overlap can, however, change later reads and therefore the output.

## Behavior

### Table layout and notation

For positive \(N\), index output groups by \(g=0,\ldots,N-1\), table positions by \(j=0,\ldots,15\), and table entries by \(x=0,\ldots,255\).

The output byte for these coordinates is located at:

\[
D+4096g+256j+x.
\]

Thus:

- Adjacent entries are one byte apart.
- Adjacent tables are 256 bytes apart.
- Adjacent groups are 4,096 bytes apart.

For formulas describing stable memory, let \(o_j\) denote the integer offset stored at \(T+4j\), and let \(B[a]\) denote the byte read at address \(a\).

These formulas are not permission to cache reads. The actual function reloads offsets and input bytes inside the entry loop.

### Initial state capture

Before generating output, the function captures the lookup base and the offset-table base through the two shared-state evaluations described above.

Subsequent table generation uses these captured addresses. The source does not show another read of the shared base value during the loops.

Capturing the addresses does not freeze their contents. Offset entries and lookup bytes remain ordinary memory reads throughout execution.

### Ordinary groups

The function first generates groups before the final group.

Its group-control value starts at 1 and advances while it is less than signed \(N\). For each such value, the corresponding output group is one less than that control value. For positive \(N\), these are exactly groups \(0\) through \(N-2\).

Within each ordinary group:

1. Table positions are processed in increasing order from 0 through 15.
2. Within each table, entries are processed in increasing order from 0 through 255.
3. Each entry reads the offset for its table position.
4. That offset selects an input byte from the current 16-byte-stride input group.
5. The selected byte is XORed with the entry index.
6. The resulting index selects a lookup byte, which is written to the current output entry.

With stable memory and valid arithmetic, the ordinary-group result is:

\[
B_{\mathrm{out}}[D+4096g+256j+x]
=
B[L+(B[S+16g+o_j]\mathbin{\oplus}x)]
\]

for:

\[
0\leq g<N-1,\quad 0\leq j<16,\quad 0\leq x<256.
\]

The selected input byte need not come from within the apparent 16-byte input group. Its address depends on the unrestricted integer displacement \(o_j\).

### Mandatory final group

After the ordinary-group loop, the function advances its working input address by \(16N\). It then always generates one more group, independently of whether the ordinary-group loop executed.

For positive \(N\), this writes group \(N-1\). Its selected input bytes use the same addressing pattern as an ordinary group at that position, but its additional masks come from the following 16-byte-stride input region.

With stable memory and valid arithmetic:

\[
B_{\mathrm{out}}[D+4096(N-1)+256j+x]
=
B[L+(B[S+16(N-1)+o_j]\mathbin{\oplus}x)]
\mathbin{\oplus}
B[S+16N+j].
\]

For each final-group table, the mask address is independent of \(x\). Nevertheless, the source reads that mask again for every entry; it does not capture one mask value for the entire table.

The final group follows the same increasing table-position and entry order as ordinary groups.

### Byte widths and XOR behavior

Both the selected input byte and entry index lie in the range 0 through 255. Their XOR therefore also lies in that range. The lookup index cannot exceed 255 through this operation.

The final-group mask is also a byte. XORing it with the lookup byte produces another value in the range 0 through 255, which is stored as one byte.

There is no addition of byte values in the transformation, no carry between entries, and no dependence on a previously generated output byte except when memory overlap makes that byte a later input.

The entry loop executes exactly 256 iterations. Its unsigned counter advances from 0 to 256; the signed comparison shown for loop continuation does not encounter a value outside the positive range during these iterations.

### Zero and negative counts

A nonpositive \(N\) skips the ordinary-group loop but does not suppress the final group.

For \(N=0\), assuming valid arithmetic and backing memory:

- Selected input reads use \(S-16+o_j\).
- Final-mask reads use \(S+j\).
- Output writes span \(D-4096\) through \(D-1\).
- Exactly 4,096 byte writes occur.

For negative \(N\), the final group is similarly located at the displacement \(4096(N-1)\) relative to \(D\), provided all relevant operations remain defined.

These are consequences of the supplied control flow, not evidence that nonpositive counts are intended or safe inputs. No empty-output case is implemented.

## Side effects

### Observable access order

Across entries, output writes occur in increasing entry order, then increasing table-position order, then increasing group order for positive \(N\).

For each ordinary entry, the data dependencies require:

- Offset-entry read before the selected input-byte read.
- Selected input-byte read before the indexed lookup-byte read.
- Lookup-byte read before the output-byte write.

For each final entry, the lookup-selection chain has the same dependencies. The additional mask-byte read occurs before the output write.

The supplied expression does not establish a portable ordering between the final mask-byte read and the lookup-byte read. Neither ordering should be required by a consumer contract.

No bulk-copy or snapshot behavior is present. An earlier output write that overlaps an offset entry, lookup byte, selected input byte, or final-mask byte can affect a later iteration.

For example, the fact that one table nominally uses one selected byte does not guarantee that all 256 entries see the same byte: that address is reread for each entry.

### Access counts

For positive \(N\), normal completion performs:

| Access | Count |
|---|---:|
| Output-byte writes | \(4096N\) |
| Offset-entry reads | \(4096N\) |
| Selected input-byte reads | \(4096N\) |
| Lookup-byte reads | \(4096N\) |
| Additional final-mask reads | \(4096\) |

For nonpositive \(N\), if execution remains valid and completes, each of these counts is 4,096 because only the mandatory final group executes.

These are logical accesses in the supplied representation, not a guarantee about machine instruction counts or memory transactions.

### Ownership and state

The function contains no explicit allocation, release, ownership transfer, or dependency call. The caller supplies the memory it accesses.

Its explicit stores target output-byte addresses only. Because disjointness is not enforced, those stores may also modify memory serving another role.

There is no rollback, synchronization, or error-reporting path. Invalid memory or arithmetic is not converted into a defined status result.

## Dependencies

No function-call dependencies are visible in F-f0c6d85ded37.

Its visible data dependencies are:

- The shared base value evaluated during initialization.
- Sixteen integer offset entries.
- Byte-addressable lookup data.
- Caller-supplied input memory.
- Caller-supplied writable output memory.

No behavior of an unknown callee is needed to explain the supplied body. Runtime facilities, fault handling, and concurrent modification behavior are not specified by the evidence.

## Acceptance examples

These examples are conditional expectations derived from static evidence. They are not executed test results.

### One group contains only final-group tables

Given \(N=1\), stable disjoint memory, and valid arithmetic:

- The ordinary-group loop performs no writes.
- The function writes 4,096 bytes starting at \(D\).
- Table position \(j\) selects its input byte at \(S+o_j\).
- Its mask comes from \(S+16+j\).

Every entry must satisfy:

\[
B_{\mathrm{out}}[D+256j+x]
=
B[L+(B[S+o_j]\oplus x)]\oplus B[S+16+j].
\]

### Two groups distinguish ordinary and final behavior

Given \(N=2\) under the same conditions:

- The first 4,096-byte group uses selected bytes at \(S+o_j\), without a final mask.
- The second group uses selected bytes at \(S+16+o_j\), with masks at \(S+32+j\).
- Exactly 8,192 output bytes are written.

Applying final masks to both groups would not satisfy this contract.

### Identity lookup exposes the byte transformation

Suppose the lookup byte at index \(x\) equals \(x\), and a particular ordinary table consistently selects input byte \(0xA5\).

Its entries must then equal \(x\oplus 0xA5\). In particular:

- Entry 0 is \(0xA5\).
- Entry 1 is \(0xA4\).
- Entry 255 is \(0x5A\).

If a final-group table has the same selected byte and a stable mask of \(0x3C\), these entries become \(0x99\), \(0x98\), and \(0x66\), respectively.

### Repeated offsets need not imply identical final tables

If two positions have equal offsets and all relevant memory is stable, their ordinary-group tables are identical within the same group.

Their final-group tables can differ because their mask addresses are position-specific. Equal offsets alone do not imply equal final outputs.

### Zero count is not a no-op

Given \(N=0\), valid backing memory before both base addresses where needed, and valid arithmetic, the function writes a full final group immediately before \(D\).

An implementation that simply returns for zero would differ from the supplied behavior.

### Overlap requires live-read behavior

If an earlier output write changes a byte subsequently used as a selected input or mask, later entries must reflect the subsequently read value.

Acceptance based on a precomputed snapshot is valid only when overlap and external mutation cannot alter the relevant reads.

## Uncertainties

### Established static evidence

The supplied body establishes:

- Sixteen tables per group and 256 bytes per table.
- Four-byte spacing between offset entries.
- Byte lookup and output operations.
- Increasing loop traversal.
- Per-entry offset, input, and lookup reads.
- A mandatory final group with an additional byte XOR.
- No explicit validation, allocation, dependency call, or error result.

### Inferred intent

A positive count and a stable shared base appear to be the intended operating conditions. Under those conditions, the function is naturally described as producing \(N\) contiguous groups from 16-byte-stride input regions.

This interpretation does not establish that offsets form a permutation, remain within a 16-byte region, or have any particular values.

### documented_only unknowns

The supplied material does not establish:

- The actual shared base, offsets, lookup contents, or input bytes.
- The precise target ABI, byte order, alignment rules, or address-wrap behavior.
- Whether signed overflow is defined by the execution environment.
- Caller-enforced limits on \(N\).
- Whether aliasing or concurrent modification is permitted in intended use.
- Whether the two initial shared-state evaluations can observe different values.
- Platform behavior after an invalid access.
- Any mathematical property of the lookup table, including invertibility.

Large positive counts require particular care: the signed products used for input and output addressing can exceed a 32-bit signed range even before considering address-base addition. A contract must not silently replace these operations with wider arithmetic or assume wraparound without target evidence.

## Evidence

This specification is based solely on the supplied static representation of F-f0c6d85ded37. The formulas describe stable-memory outcomes; the access-order requirements preserve the repeated reads and writes visible in that representation.

No implementation was executed, no runtime tests were performed, and no external dependency behavior was inspected. The acceptance examples are derived expectations, not observed results.
