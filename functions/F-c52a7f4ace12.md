<!-- Trusted provenance: F-c52a7f4ace12; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c52a7f4ace12

## Purpose

F-c52a7f4ace12 transfers rectangular source data into an existing destination surface. Its behavior depends on source and destination discriminators, descriptor flags, and an operation byte. Supported paths expand packed samples, perform table lookups, combine destination bits, copy masked words, or invoke conversion and compositing dependencies.

“Raster transfer,” “coverage,” and “palette” describe inferred intent. They do not establish standard pixel formats or blending mathematics. Observable exceptions include nonadvancing transparent writes, operation-dependent masked-word behavior, mutable foreground scratch, and setup calls on empty transfers.

This contract describes static evidence only. It does not promise clipping, bounds checking, general overlap safety, dependency purity, or successful handling of invalid memory.

## Interface

The function accepts three arguments and has no defined return value:

| Argument | Representation | Use |
|---|---|---|
| Transfer context | 32-bit address | Supplies geometry, descriptors, operations, and prepared transfer state |
| Comparison operand | Unsigned 32-bit value | Used by selected transparency operations |
| Auxiliary context | 32-bit address | Visibly accessed only by ordinary 8-to-32 selector 0x23 |

### Transfer-context layout

Offsets are in bytes. Except for the operation byte, listed fields occupy 32 bits.

| Offset | Role |
|---|---|
| 0x0c, 0x10 | Source top and left |
| 0x14, 0x18 | Source bottom and right, used for overlap handling |
| 0x1c | Source descriptor address |
| 0x20 | Prepared source address |
| 0x24 | Source byte stride |
| 0x2c | Prepared source bit position |
| 0x40 | Color operand |
| 0x44 | 8-bit operation selector |
| 0x48, 0x4c, 0x50, 0x54 | Destination top, left, bottom, and right |
| 0x58 | Destination descriptor address |
| 0x5c | Prepared destination address |
| 0x60 | Destination byte stride |
| 0x64 | Word-span control |
| 0x68 | Prepared destination bit position |
| 0x70, 0x74 | First- and last-word masks |

Each descriptor supplies a data address at offset 0, a signed 32-bit discriminator at offset 8, a 32-bit shift-related field at offset 0x0c, and a 16-bit format/flags field at offset 0x10. A complete descriptor size is not established.

Ordinary rectangular paths derive width from destination right minus left and height from destination bottom minus top. Their source origin uses source coordinates. Prepared-word paths instead use prepared addresses, bit positions, masks, and span control.

### Preconditions and arithmetic

All memory actually accessed must be valid, including tables, indirect call targets, scratch passed to dependencies, neighboring words, and packed-source lookahead bytes. No null, extent, palette-index, or alignment validation is visible.

Arithmetic distinctions are material:

| Paths | Width/count behavior |
|---|---|
| 2-to-16, ordinary 2-to-32 | Signed 32-bit pixel counts; process while positive |
| 4-to-16, ordinary 8-to-16 | Unsigned 32-bit widths and remaining counts; pair processing uses an unsigned comparison against one |
| 4-to-8, 4-to-32, 2-to-32 coverage | Pixel counts decrement until zero, rather than requiring positive counts |
| 2-to-4 | Signed remaining counts, with unconditional initial fragments for some source phases |
| 1-to-16/32, embedded palettes, ordinary 8-to-32, ordinary 16-to-32 | Signed positive-count pixel loops |
| Prepared-word paths | Signed span control; zero selects one word, positive values select first/interior/last processing |

Row-count checks use signed positive-height interpretation. A nonpositive height is not a universal side-effect-free condition.

Packed-source column-to-byte offsets for 1-, 2-, and 4-bit ordinary sources use an arithmetic right shift after interpreting the column as signed 32 bits, by three, two, or one respectively. They must not be replaced with unsigned division for negative columns.

Relative word displacements are unsigned 32-bit differences subsequently tested as signed 32-bit values. A negative signed interpretation causes a 32-bit-position correction and backward source adjustment. Unsigned count and correction arithmetic retains 32-bit wraparound. General signed-overflow and out-of-range-shift behavior remains documented_only; safe callers avoid those cases.

## Behavior

### Dispatch, capture, and access ordering

Supported discriminator pairs are:

| Source | Destination |
|---:|---|
| 1 | 2, 4, 8, 16, 32 |
| 2 | 4, 16, 32 |
| 4 | 8, 16, 32 |
| 8 | 16, 32 |
| 16 | 16, 32 |

Other combinations return without visible pixel transfer, but entry reads still occur.

Before reading the source descriptor, the function reads seven global values, in this order:

1. 2-to-4 expansion-table address.
2. 1-to-8 expansion-table address.
3. 1-to-4 expansion-table address.
4. 1-to-2 expansion-table address.
5. 4-to-32 table address.
6. Default indirect conversion target.
7. 2-to-32 table address.

These are captured values, not an invocation-wide snapshot of global state. Some paths use them; others reload corresponding globals. Capturing a table address does not capture its contents.

Ordinary traversal is left-to-right, row-by-row. Source row starts normally advance by source stride. Destination row correction normally adds destination stride minus the nominal transferred byte width to the advanced destination pointer. The anomalies below can therefore affect subsequent rows.

Reads, dependency calls, and writes occur progressively; there is no complete-image staging. Overlapping storage can influence later reads. Packed processing can read another byte immediately after consuming the final contributing sample.

### Table expansion into 16 bits

For 2-to-16, 4-to-16, and ordinary 8-to-16, source samples select 16-bit entries.

| Selector | Behavior |
|---:|---|
| 0 | Replace |
| 1 | OR entry with destination |
| 2 | AND entry with destination |
| 3, 7 | XOR entry with destination |
| 0x20 | Replace samples unequal to the comparison key |
| 0x21 | Dependency-assisted coverage processing |
| 0x23 | Same visible helper route as 0x21, ordinary 8-to-16 only |

Other selectors return, subject to setup effects.

The 2- and 4-bit tables are selected using destination flag bit 0x2000 and whether any other format-field bit is set. Comparison keys remain full-width for 2-to-16, are masked to four bits for 4-to-16, and to eight bits for 8-to-16.

The 4- and 8-bit paths may process an initial alignment halfword, then 32-bit pairs, then a trailing halfword. In pair stores, the first sample occupies the low half and the second the high half.

For selector 0x20 in those two paths, a skipped initial or final halfword does not advance the destination pointer. Pair processing nevertheless reads and writes a reconstructed word and advances four bytes, even when both samples match the key. Nominal row correction is unchanged. Skipped edges can thus shift both current-row output and later row starts.

Unsigned remaining counts make zero width unsafe to treat uniformly: an initial alignment halfword can still be processed, followed by decrement wraparound.

#### Coverage setup and helper timing

For 2-to-16 and 4-to-16 selector 0x21, the selected indirect converter is called on the low 16 color bits before the height check. Its result becomes foreground scratch.

Each subsequent F-c16187b321ab call obtains its branch-specific global helper/table operand afresh, including calls after the setup converter and previous helpers. The selected converter target itself is retained. Destination/output associations partly depend on adjacent scratch rather than explicit recovered arguments; exact helper update semantics are documented_only.

Ordinary 8-to-16 first checks shared conversion state, before operation support and height checks. It compares a cached unsigned 16-bit value against the full 32-bit color operand; the validity byte is consulted when that comparison matches. A mismatch or invalid state invokes F-e6020fb0c578 with the color scratch address and a flag-selected conversion target.

After that call, the transfer-table global is loaded, context descriptors and geometry are read again, and a coverage converter is selected from then-current destination flags. For selectors 0x21 and 0x23, that converter is called on the then-current low 16 color bits before checking height. Its visible return is discarded. Pixel processing uses F-2314074e0839; its scratch contract is not fully established.

### Packed direct expansion

Packed samples are most-significant-first: 1-bit samples use bits 7 through 0; 2-bit samples use shifts 6, 4, 2, 0; 4-bit samples use high then low nibble.

**4-to-8:** only selector 1 is supported. Each nibble is replicated into both halves of a byte and ORed into the destination.

**2-to-4:** selectors 0 and 1 replace or OR bytes constructed using the entry-captured four-entry byte table. Complete four-sample groups produce two bytes, placing the first sample’s contribution in the high nibble.

Partial groups do not generally preserve unused destination nibbles. A trailing single sample supplies only a high-nibble expression; three trailing samples produce one complete byte and a second high-nibble expression. Replacement can clear neighboring nibble bits.

Initial source phases 4, 2, and 0 trigger fixed fragments consuming three, two, and one samples, regardless of requested width. For phases 4 and 0, the first fragment byte uses an unshifted table contribution. These fragments also read the next source byte. Destination-column parity determines the starting byte address but does not independently select an output-nibble phase.

### One-bit sources to 16 or 32 bits

Foreground is the low 16 color bits for 16-bit output and the full color operand for 32-bit output. Background is a destination-flag-selected 16-bit constant or 0xffffffff respectively.

| Selector | Set source bit | Clear source bit |
|---:|---|---|
| 0 | Write foreground | Write background |
| 1 | Preserve | Write background |
| 2, 0x21 | Write foreground | Preserve |
| 3, 7 | XOR background with destination | Preserve |

Thus 32-bit selectors 3 and 7 complement destination words at set bits.

Only 1-to-16 supports 0x20. Comparison zero makes set bits write zero and clear bits preserve. Any nonzero comparison makes clear bits write background and set bits preserve. Foreground is not written by this selector.

### Word-oriented one-bit expansion

The 1-to-2/4/8 paths use lookup-expanded 32-bit words and prepared state. Source bit positions are reduced modulo 32. Negative relative displacement moves source selection backward by two bytes, one byte, or one nibble respectively.

For 1-to-4, reduced positions 8–15, 16–23, and 24–31 select source bytes one, two, and three bytes ahead.

Destination words are byte-reversed into a normalized domain before combination and reversed again for storage. Let D denote the normalized old destination, E the aligned expansion, C the color operand, and M the active mask.

For destination discriminators 4 and 8:

| Selector | Normalized result |
|---:|---|
| 0 | (D AND NOT M) OR (E AND C AND M) |
| 1 | (E AND C AND M) OR (D AND NOT(E AND M)) |
| 2 | D AND ((E AND M) OR NOT M) |
| 3 | (D AND NOT M) XOR (M AND NOT E) |

Masked selector 3 replaces active bits with complemented expansion bits; it is not destination XOR inside that mask. Whole-word selector 3 instead uses D XOR E.

Zero span control processes one word under the intersection of edge masks. Positive control processes a first masked word, intervening whole words, and a final masked word; control one means two words.

Shifted 1-to-4 selector-0 interior words apply C only to the current-byte contribution, not the right-shifted lookahead contribution.

The 1-to-2 path does not dispatch on operation. It performs masked replacement, combining two 16-bit expansion entries with the first source byte in the high half. Depending on addresses and row corrections, stores use either one 32-bit write or two ordered 16-bit writes, lower address first. The latter does not eliminate visible 32-bit destination reads.

#### Expansion-table capture versus reload

The visible distinctions are:

| Path | Table-address use |
|---|---|
| 1-to-8, aligned single word | Entry capture for all supported selectors |
| 1-to-8, aligned multiword | Entry capture for 0, 2, 3; global reloads for 1 |
| 1-to-8, shifted | Global reloads at expansion expressions |
| 1-to-4, aligned single word | Global reloads for 0, 1; entry capture for 2, 3 |
| 1-to-4, aligned multiword | Entry capture for 0, 3; reloads for 1; selector 2 reloads once per row and retains that value through the row |
| 1-to-4, shifted | Global reloads; selector-0 multiword interiors use a value reloaded after the first store, while edges read globals separately |
| 1-to-2, aligned | Entry capture |
| 1-to-2, shifted | Global reloads at edges; multiword interiors use a value reloaded after the first store |

No atomic snapshot or synchronization is established.

### Two- and four-bit sources to 32 bits

Ordinary 2-to-32 uses the entry-captured table: selectors 0, 1, 2, and 3/7 replace, OR, AND, and XOR respectively. Selector 0x20 is unsupported.

For 4-to-32, selectors 0 and 2 both replace with an indirectly converted entry from the entry-captured table.

Both support 0x21 coverage. Before checking height, they decode the foreground using F-1e2f295ee27c. Their output converter is the entry-captured default unless destination bit 0x2000 selects an alternate target during setup. The 2-bit opacity constant is read after foreground decoding, so dependency-induced global changes can affect it.

Coverage zero preserves without a destination-pixel read. Maximum coverage writes the converted original color operand. Intermediate coverage uses constant-provided opacity bytes for 2-bit samples or nibble replication for 4-bit samples.

Partial processing converts and decodes the old destination, calls F-86dd6fb1ad71 with final operand 255, then F-35c2e4297234, exchanges the low and third component bytes, converts, and stores. Foreground scratch passed by address is reused; its preservation by dependencies is not established.

### Embedded palettes

Source discriminator 8 with format field 100 supports embedded palettes when targeting 32 bits. Source discriminator 16 with format field 101 supports them when targeting 16 or 32 bits. These paths ignore the operation selector.

Data begins with a 32-bit palette count followed by 32-bit entries. Samples start after that header plus source row and column offsets. Indices are unsigned 8- or 16-bit values; zero selects the first entry. No count/index check is visible.

Each entry is passed through the entry-captured default converter. The global format operand for F-1e2f295ee27c is read after that conversion, for each sample. Output converters are separately loaded during branch setup from destination flags, including a fresh load for the default 32-bit output target.

Decoded high byte zero preserves without reading the destination pixel. High byte 255 produces an opaque output. Intermediate values read and decode destination, combine, finalize, convert, and store.

For 16-bit output, F-82e3ffb61877 receives the three low component bytes before output conversion. Communication between these calls is documented_only.

### Ordinary 8-to-32 coverage

Before initial foreground decoding, this path captures three multiplier words, source and destination addresses, signed width and height, and both row corrections. It then calls F-1e2f295ee27c using the color operand and destination format.

Only afterward is the operation byte read. If it is not 0x21, it is read again to check for 0x23. Changes made by the decoder can therefore affect dispatch, but do not retroactively recompute captured geometry.

For 0x21:

- Coverage zero performs no destination-pixel read or write.
- Coverage 255 clears the foreground record’s high byte and stores its current low three bytes, rearranged, with output high byte 255.
- Intermediate coverage replaces the record’s high byte, reads and rearranges destination, passes the foreground record’s address to F-86dd6fb1ad71, finalizes through F-35c2e4297234, and stores the rearranged result.

The same foreground record survives across pixels and rows. Its low three bytes are not restored after dependencies. Later full-coverage pixels therefore need not use the original decoded color. That stronger guarantee would require an unavailable dependency-preservation contract.

Selector 0x23 additionally reloads the auxiliary signed field at offset 0x0c and context destination-left field for each pixel. It forms a 32-bit horizontal difference using captured width and remaining count. Adjustment occurs only when the unsigned difference is below six. Adjusted coverage is bits 8–15 of the 32-bit multiplier/source-byte product.

Only three multiplier words are visibly initialized. Six-position indexing reaches other scratch storage; a complete ramp is documented_only.

Unsupported selectors, and 0x21 with nonpositive height, reach an unresolved indirect control transfer. Selector 0x23 with nonpositive height returns after decoding.

### Ordinary 16-bit sources

**16-to-16:** operation is ignored; prepared masked words are copied. Equal source and destination data bases trigger overlap analysis. Under ordinary nonoverflowing signed coordinates, row reversal occurs when source top < destination top < source bottom and horizontal intervals overlap strictly.

Reversal updates prepared source address, prepared destination address, source stride, and destination stride, in that order. Address adjustments use their respective strides times source height minus one; both strides are then negated. These writes persist even if the subsequent height check prevents copying.

Relative alignment can move source back one word. Shifted copying assembles neighboring byte-reversed words. Edges preserve excluded destination bits; aligned interiors copy directly. Some paths use two ordered halfword stores while retaining word reads. No horizontal reversal is visible.

**16-to-32:** source and destination conversion targets are selected during setup from their respective flags. Each halfword is converted, decoded, rearranged, and converted for output.

Selectors 0, 1, 2, and 3/7 replace, OR, AND, and XOR. Selector 0x20 compares the rearranged decoded value with the full comparison operand before output conversion. Matching samples do not advance destination; nonmatches write and advance. Nominal row correction can displace later rows. The nonmatching output converter’s input association is incomplete.

Selector 0x22 composites only for source format 0x1444. Zero alpha preserves; 255 replaces; partial alpha composites and then explicitly sets alpha to the maximum of saved source and destination alpha. Other source formats use replacement. Other selectors return.

## Side effects

Direct effects comprise progressive destination writes, scratch changes, and the four persistent overlap-reversal context writes.

Entry global reads, cache preparation, foreground conversion, and decoding can occur without pixel transfer. They must not be removed merely because height is nonpositive.

No direct allocation, release, ownership transfer, error report, rollback, or synchronization is visible. Dependencies receive mutable memory addresses. Their global effects, pointer retention, ownership behavior, and preservation of scratch are unknown. Captured call targets remain distinct from globals reread after calls; neither table contents nor all context fields are fixed for the invocation.

## Dependencies

| Opaque ID | Established visible role |
|---|---|
| F-1e2f295ee27c | Decode a value and format operand into component scratch |
| F-86dd6fb1ad71 | Combine component records with final operand 255 |
| F-35c2e4297234 | Finalize or transform component results |
| F-c16187b321ab | Packed coverage processing for 16-bit output |
| F-2314074e0839 | Byte coverage processing for 16-bit output |
| F-e6020fb0c578 | Prepare shared ordinary 8-to-16 conversion state |
| F-82e3ffb61877 | Consume three component bytes before 16-bit output conversion |

Indirect converters and the unresolved indirect control target have no supplied opaque IDs. Their complete contracts remain documented_only.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Nibble replication:** 4-to-8 selector 1, sample 0x0a, destination 0x05 produces 0xaf.
2. **Comparison widths:** 2-to-16 key 4 matches no possible sample. For 4-to-16, keys 0x12 and 2 are equivalent.
3. **Transparent edge:** A matching initial alignment sample in 4-to-16 selector 0x20 leaves destination advancement at zero; later pair processing uses that unadvanced address.
4. **One-bit special transparency:** 1-to-16 selector 0x20 with comparison zero writes zero for set bits regardless of foreground.
5. **Masked selector 3:** A single-word 1-to-8 transfer preserves bits outside the intersected mask and writes complemented E inside it.
6. **Empty coverage setup:** Nonpositive height does not suppress the selected converter in 2-/4-to-16 coverage or ordinary 8-to-16 selectors 0x21/0x23, nor foreground decoding in 2-/4-to-32 coverage.
7. **Post-call global reload:** If the packed-coverage setup converter changes the global helper/table operand, the next F-c16187b321ab call receives the newly read operand, not an entry snapshot.
8. **Post-decode dispatch:** If ordinary 8-to-32 decoding changes the operation byte to 0x23, dispatch observes that value while retaining geometry captured before decoding.
9. **Mutable foreground:** If partial compositing changes the foreground record’s low bytes, a later coverage-255 store uses those current bytes and clears the record’s high byte.
10. **Palette transparency:** A decoded palette high byte of zero avoids destination-pixel reads and writes on the visible path.
11. **Alpha maximum:** For format 0x1444 and saved source/destination alpha 80/120, partial 16-to-32 selector 0x22 supplies alpha 120 to final conversion.
12. **Persistent reversal:** Qualifying 16-to-16 overlap leaves adjusted prepared addresses and negated strides in the context even when no rows are copied.

## Uncertainties

The following remain documented_only:

- Table contents, background constants, indirect conversion semantics, and color-space meanings.
- Exact blending mathematics and dependency mutation of passed records.
- Complete packed-coverage scratch/output associations and some implicit converter inputs.
- The full six-position multiplier behavior.
- The unresolved indirect control transfer.
- General signed-overflow, unusual shift, invalid-dimension, unsuitable-alignment, and inaccessible-memory behavior.
- General overlap safety beyond the explicit row-reversal mechanism.
- Dependency effects on globals, memory ownership, and pointer retention.

These unknowns do not erase established read timing, setup calls, widths, pointer advancement, or persistent writes.

## Evidence

Established evidence is limited to the supplied static function representation: dispatch, field offsets and widths, arithmetic interpretations, global capture and reload sites, call ordering, scratch transitions, progressive memory accesses, and context writes.

Intent labels are interpretations of that structure, not additional guarantees. No dependency bodies, table contents, implementation access, runtime observations, or runtime tests were available. Acceptance examples are static deductions under their stated conditions. No snapshot identifier is asserted.
