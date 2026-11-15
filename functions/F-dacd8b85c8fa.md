<!-- Trusted provenance: F-dacd8b85c8fa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dacd8b85c8fa

## Purpose

F-dacd8b85c8fa transfers packed bits from source storage to destination storage using a relative bit offset, boundary masks, a signed row count, and independent byte strides.

Under the little-endian precondition below, masked boundary bits are replaced with source-derived bits; unselected destination bits are preserved. Interior words are replaced completely. This is masked replacement, not exclusive-OR drawing.

Before transferring data, the function can persistently adjust both addresses in its control record and negate both stored strides. Reversing row traversal to accommodate overlap is an inferred purpose, not an established guarantee of general overlap safety.

All control-record reads used to configure the transfer finish before the first destination-data store. Transfer loops use cached configuration. Destination stores that overlap the record can overwrite its persistent contents, but cannot reconfigure the transfer already in progress.

## Interface

The function receives one 32-bit address identifying a control record. It returns no value and exposes no explicit success or error status.

### Control-record layout

Offsets are byte offsets from the supplied address. Every listed field occupies four bytes.

| Offset | Role |
|---|---|
| `0x0C` | Signed lower row bound for reversal |
| `0x10` | Signed coordinate for reversal |
| `0x14` | Signed upper row bound for reversal |
| `0x18` | Signed coordinate for reversal |
| `0x1C` | Address of a readable 32-bit identity value |
| `0x20` | Mutable source byte address |
| `0x24` | Mutable source row stride, in bytes |
| `0x2C` | Source bit position; only the low five bits participate |
| `0x48` | Signed initial row coordinate |
| `0x4C` | Signed coordinate for reversal |
| `0x50` | Signed terminal row coordinate |
| `0x54` | Signed coordinate for reversal |
| `0x58` | Address of a readable 32-bit identity value |
| `0x5C` | Mutable destination byte address |
| `0x60` | Mutable destination row stride, in bytes |
| `0x64` | Signed transfer-span parameter |
| `0x68` | Destination bit position; only the low five bits participate |
| `0x70` | First-word mask |
| `0x74` | Last-word mask |

Coordinate names describe comparison roles, not a verified geometry API. Identity equality compares the indirectly loaded 32-bit values; the identity addresses need not be equal.

### Widths and byte order

Addresses, masks, data words, and stored control fields are 32 bits. Split stores write 16 bits at a time. Coordinates, row counts, and span-dependent loop bounds use signed 32-bit comparisons.

A **logical word** is the byte reversal of a native 32-bit memory load. The byte reversal is explicit, not conditional on native byte order.

Little-endian execution is a precondition for the common memory-level packed-bit contract:

- Four consecutive bytes are interpreted as a big-endian logical word.
- A full-word result store writes the byte reversal of the logical result.
- A split result store writes two separately constructed native halfwords, lower address first.
- Both forms produce the same four bytes for the same logical result.

For logical result `11223344`, the full-word native value is `44332211`. The split native halfword values are `2211`, then `4433`. Both produce bytes `11 22 33 44` on little-endian execution.

This equivalence concerns result encoding, not access timing or atomicity. On big-endian execution, these values instead produce `44 33 22 11` through the full-word store and `22 11 44 33` through the split stores. The common packed-bit interpretation therefore does not extend to that environment.

### Preconditions and errors

The caller must provide:

- A readable control record, with the four reversal fields writable whenever reversal qualifies.
- Readable identity values through both identity-address fields.
- Readable source storage for every actual word read, including lookahead and reads under zero masks.
- Readable and writable boundary destination words, and writable interior destination words.
- An environment supporting the actual access widths and alignments.
- Valid storage lifetimes throughout execution.

The ordinary numerical contract requires valid, nonwrapping addresses and no signed arithmetic overflow in row-count subtraction, reversal displacement, stride multiplication or negation, span-derived adjustments, or address advancement. Negating the minimum signed 32-bit stride is outside this precondition. Behavior outside these arithmetic restrictions is not established as portable behavior.

Control-record disjointness is not needed to prevent destination stores from changing cached transfer configuration: the loops do not reread that configuration. Disjoint storage is nevertheless a useful precondition when the caller requires persistent record contents to survive destination writes or requires reversal writes not to alter source data. Without disjointness, the alias effects described below apply.

No null check, bounds check, alignment repair, allocation, or recoverable error return is visible. A fault can occur after partial modification. Concurrent modification is outside the established contract.

## Behavior

### Traversal-reversal decision

The initial access sequence is:

1. Read the identity value through the address at `0x58`.
2. Capture the initial row coordinate at `0x48`.
3. Read the identity value through the address at `0x1C`.

Further reversal conditions are evaluated only as preceding conditions permit. Reversal requires:

- Equal identity values.
- The lower bound at `0x0C` strictly below the captured initial row coordinate.
- That initial coordinate strictly below the upper bound at `0x14`.
- The coordinate at `0x10` strictly below the coordinate at `0x54`.
- The coordinate at `0x4C` strictly below the coordinate at `0x18`.

All comparisons are signed. Equality at either row bound prevents reversal.

When reversal qualifies, the displacement factor is the upper bound minus the lower bound minus one. Four persistent writes occur in this order:

1. Replace the source address with its previous value plus source stride times the displacement factor.
2. Replace the destination address analogously using the destination stride.
3. Negate the stored source stride.
4. Negate the stored destination stride.

These values are not restored. The previously captured initial row coordinate remains the basis for the later row count.

The reversal writes precede transfer setup and all source-data reads. If source storage overlaps these record fields, the writes can change bytes later consumed as source data. This is distinct from destination-store aliasing during the transfer.

### Setup and empty-row behavior

After the reversal decision, setup reads the first mask, destination address, source address, bit positions, and span, followed by the last mask and terminal row coordinate. This ordering does not impose an additional order between operands within a compound expression.

The initial position difference is the source position’s low five bits minus the destination position’s low five bits, mathematically between −31 and 31.

- A negative difference is increased by 32, and the local source address moves backward four bytes.
- Otherwise, the difference itself is the shift distance.

The resulting shift distance is between zero and 31. The local source adjustment is not written back to the record.

The row count is the terminal row coordinate minus the captured initial coordinate. A signed result below one returns without source-data or destination-data accesses.

This is not an early validation return. Reversal and setup have already occurred. Both strides and the applicable store-path quantities are obtained before the selected branch checks the row count. An empty-row operation can therefore modify the four reversal fields.

Before any destination-data store, every control-record read used by the transfer has completed. Addresses, masks, shift, span, row count, strides, and derived advancement quantities are retained locally. No loop iteration rereads those record fields.

### Source-derived candidates

Number source words from the adjusted local source address at four-byte intervals.

For zero shift, candidate position \(j\) is logical source word \(j\).

For nonzero shift distance \(d\), the candidate is:

\[
(\text{source}_j \ll d)
\;|\;
(\text{source}_{j+1} \gg (32-d)).
\]

These are unsigned 32-bit shifts, with left-shift results confined to 32 bits. Both counts are between one and 31. Separate zero-shift paths avoid shifting by 32.

A boundary mask \(M\) produces logical result:

\[
(\text{old destination} \mathbin{\&} \neg M)
\;|\;
(\text{candidate} \mathbin{\&} M),
\]

with the complement confined to 32 bits.

Some branches combine the two masked terms using exclusive OR. Because the masks are disjoint, the numerical result is the same as OR. Existing destination bits are not XORed with selected source bits.

### Zero span

A zero span updates one destination word per processed row. The effective mask is the intersection of the first and last masks, not two successive boundary operations.

For each row:

- Zero shift reads one source word before reading the destination word.
- Nonzero shift reads two consecutive source words before reading the destination word.
- The merged result is then stored.

Successive row starts advance by the cached source and destination byte strides.

An effective mask of zero does not suppress accesses. Required source reads, the destination read, and a destination store still occur. Under the little-endian contract, the stored bytes are unchanged.

### Nonzero span

For positive span \(n\), each row writes \(n+1\) destination words:

- Position zero uses the first mask.
- Positions one through \(n-1\) are complete replacements, without old-destination reads.
- Position \(n\) uses the last mask.

Span one writes two distinct boundary words and no interior words. Its masks are not intersected.

Negative nonzero spans are not rejected. They also write two boundary words with no interior words.

For a unified description, let \(m=\max(1,n)\), using signed span \(n\). Each row writes positions zero through \(m\). Successive row starts advance by:

\[
\text{source stride}+4(m-n)
\quad\text{and}\quad
\text{destination stride}+4(m-n).
\]

Positive spans therefore advance by the stored strides. Negative spans introduce additional forward displacement, subject to the arithmetic preconditions. This describes established behavior, not intended input validity.

### Access order within multiword rows

Words within a row are processed toward increasing addresses, even after row traversal reverses.

For nonzero shift, both store paths:

1. Read source words zero and one.
2. Read destination word zero and store the first-mask result.
3. Read the next source word before each interior result is stored.
4. Read the additional source word needed for the final candidate.
5. Read the final destination word and store the last-mask result.

For final position \(m\), source words zero through \(m+1\) must be readable.

For zero shift, access timing differs:

- **Full-word stores:** Read the first destination word before source word zero, then store the first result. Read each interior source word immediately before storing it. At the final boundary, read the destination before the final source word.
- **Split stores:** Read source words zero and one before reading or writing the first destination word. Before each interior store, prefetch the following source word. The final boundary uses the cached final source word, then reads and updates the destination.

Both zero-shift paths read source words zero through \(m\). Different read timing can produce different results under overlap. Boundary masks of zero do not suppress the associated reads or stores.

### Store-path selection

For zero span, the alignment test uses the local source address, local destination address, and both strides.

For nonzero span \(n\), it uses:

- The local source address.
- The local destination address.
- Source stride minus \(4n+4\).
- Destination stride minus \(4n\).

If every tested quantity has its low two bits clear, result stores are native 32-bit stores. Otherwise, each result uses two native 16-bit stores: first at the destination address, then two bytes later.

For zero-shift split-path interiors, the cached native source word’s lower halfword is written before its upper halfword. On little-endian execution this preserves the four source bytes.

Split-store selection does not remove 32-bit loads. Unaligned sources and boundary destinations can still require unaligned word reads; odd destination addresses require unaligned halfword stores. This path does not establish arbitrary-alignment support.

## Side effects

Explicit caller-visible writes are the four reversal fields, when reversal qualifies, and destination storage.

Ordinary row advancement does not publish final cursors, a processed-row count, or completion status. Stored addresses change explicitly only during reversal, although destination stores may overwrite them if storage aliases the record.

Aliasing has three distinct effects:

- **Reversal writes overlapping source storage:** These occur before setup and can alter source bytes before any source-data read.
- **Destination stores overlapping the record:** These can overwrite persistent record contents, including addresses, masks, coordinates, or identity-address fields. They do not alter the already cached transfer configuration, row count, or store-path selection.
- **Destination stores overlapping future source reads:** These can change subsequently consumed data, including source data located within the record. Previously cached source words retain their captured values.

Thus, even if a destination store overwrites a stored stride or terminal coordinate, later rows continue using the captured configuration. Record corruption remains externally visible after return; it is not a mechanism for reconfiguring the current loop.

Source storage is not snapshotted. Neither reversal nor prefetching establishes general memory-move semantics.

No allocation, release, synchronization, callback, or explicit external-resource operation is visible. Ownership is not transferred; the caller remains responsible for storage lifetime.

Faults can leave partial reversal or destination updates. No rollback is visible. Split stores expose an intermediate halfword-updated state. No atomicity guarantee is established.

## Dependencies

No function calls are visible in F-dacd8b85c8fa. There are no callable dependency IDs to list.

Execution depends on native memory access, byte order, integer behavior, alignment support, and environmental fault handling. Indirect identity reads are data dependencies, not calls.

No assumptions about unknown callees preserving globals or retaining, owning, or releasing memory are needed.

## Acceptance examples

These are conditional static criteria, not executed tests. Unless stated otherwise, assume little-endian execution, valid storage and arithmetic, and no overlap affecting source reads. Hexadecimal word values denote logical words.

1. **Single-word intersection.** One row, zero span, zero shift, masks `FFFF0000` and `00FFFF00`, source `12345678`, and destination `AABBCCDD` produce `AA34CCDD`.

2. **Distinct boundaries.** Span one, zero shift, masks `FFFF0000` and `0000FFFF`, sources `11223344`, `55667788`, and destinations `AABBCCDD`, `EEFF0011` produce `1122CCDD`, `EEFF7788`.

3. **Shifted candidate.** Shift eight with source words `11223344`, `55667788` produces candidate `22334455`. A full mask selects that value.

4. **Negative position difference.** Source position three and destination position eleven yield shift 24 and a local source address four bytes before the stored source address.

5. **Zero mask.** A zero effective mask preserves destination bytes but still requires source reads, a destination read, and a store.

6. **Empty transfer after reversal.** A qualifying reversal with zero calculated rows displaces both stored addresses and negates both strides, without data-word accesses.

7. **Strict bounds.** An initial coordinate equal to either row bound prevents reversal.

8. **Negative span.** Span minus one writes two boundary words per row. Successive row starts advance by the corresponding stride plus eight bytes.

9. **Store encoding.** Logical result `11223344` produces bytes `11 22 33 44` through either little-endian store form. Split stores update the lower-address halfword first.

10. **Big-endian exclusion.** The same boundary result produces different byte sequences through the two store forms on big-endian execution; common store equivalence is not required there.

11. **Prefetch-sensitive overlap.** With zero shift and nonzero span, the split path captures source word one before the first destination store. The full-word path does not. If that store overwrites source word one, identical results are not required.

12. **Record overwritten during transfer.** If an otherwise valid destination store overwrites the record’s stride fields while more rows remain, later row advancement still uses the captured stride-derived quantities. The overwritten field bytes remain changed after return.

13. **Reversal changes later source input.** If a reversal field also occupies source storage read afterward, the transfer consumes the bytes present after reversal, unless an intervening destination write changes them again. It does not recover the field’s entry-time bytes.

## Uncertainties

- **Inferred intent:** Identity comparison and traversal reversal resemble overlap handling; universal overlap safety is not established.
- **documented_only:** External coordinate meanings, permitted span ranges, and intended validity of negative spans are unknown.
- **documented_only:** Deployment endianness is not independently established. Little-endian execution is a contract precondition.
- **documented_only:** No common big-endian packed-bit-transfer contract is established.
- **documented_only:** Portable behavior for unsupported alignment, signed overflow, or address wraparound is unresolved.
- **documented_only:** Concurrency, volatile memory, device memory, and atomic visibility are unspecified.
- **documented_only:** Fault delivery and recovery depend on the environment.
- **documented_only:** External calling-convention details and instruction-level correspondence have not been independently verified.

## Evidence

Established static evidence includes control-record offsets, indirect identity reads, strict signed reversal comparisons, ordered persistent reversal writes, explicit byte reversal, masked merges, loop boundaries, source lookahead, and alternative store widths.

Every control-record read used by the transfer precedes its first destination-data store. Transfer loops contain data accesses and local-state advancement, not control-record reloads. This establishes cached-configuration stability under destination-record aliasing, while leaving persistent record contents and future source-data reads exposed to overlapping writes.

Reversal writes precede transfer setup and source-data reads, establishing the separate possibility of source mutation before transfer begins.

The endian restriction follows from the numerical full-word and halfword store encodings. Geometry and overlap-handling purpose remain interpretations rather than established API semantics.

No runtime tests, target execution, caller inspection, or independent implementation verification have been performed.
