<!-- Trusted provenance: F-c3b0ba41b00e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c3b0ba41b00e

## Purpose

F-c3b0ba41b00e performs an ordered sequence of indirect calls, transforms their result bit patterns, mutates memory through fixed and calculated addresses, and ends with an unresolved indirect transfer.

The supplied static representation does not establish an application-level purpose. Repeated offsets and address-like stores do not establish an object model. Printable constants do not establish text processing, serialization, or a string format.

This contract distinguishes:

- **Established static evidence:** represented calls, arithmetic, access widths, and ordering.
- **Inferred intent:** interpretations that are not behavioral requirements.
- **documented_only:** unresolved behavior for which no concrete acceptance result is prescribed.

All ordering requirements below describe the supplied representation. They are not runtime observations or independent confirmation of its machine-level fidelity.

## Interface

### Inputs

The represented interface has no explicit parameters. Its implicit inputs include:

- An incoming execution-context value used to form the first indirect target by adding 0xFFE.
- Embedded address-like and symbol-relative target values, likewise adjusted by 0xFFE.
- Primary and secondary call-result channels.
- Memory addressed through call-derived values and subsequently loaded words.
- Fixed-address storage, including 0x01, 0x05, 0x15, 0x21, 0x3A, 0x54, 0x60, 0x67, and 0xA9.
- A separate global word captured as the publication base.
- The unresolved target of the final transfer.

In particular, the value associated with fixed address 0x05 is converted to an unsigned 32-bit quantity. Its declaration is absent: neither its original storage width nor its original signedness is established. The conversion does **not** establish an unsigned byte load or restrict the converted value to 0–255.

### Widths and arithmetic

| Quantity or access | Represented width |
|---|---:|
| Explicit byte access | 8 bits |
| Explicit halfword store | 16 bits |
| Ordinary word access | 32 bits |
| Combined call result | 64 bits |
| Most arithmetic and address-like quantities | 32 bits |
| Original storage at 0x05 | documented_only |

Combined results are numerically separated into low and high 32-bit words. This does not establish memory endianness.

Byte stores retain eight bits; halfword stores retain sixteen. Explicit unsigned byte loads produce values from 0 through 255 when widened.

Examples involving overflow use a **conditional 32-bit bit-pattern model**: each word-sized operation retains its low 32 bits. This is not a promise of portable signed arithmetic. Signed overflow, signed left shifts, address wraparound, alignment, and exceptional shift counts remain documented_only.

### Outputs and completion

No scalar result is declared. Observable behavior includes the call trace, represented arguments, direct memory accesses, any effects of unresolved callees, and the final transfer with four arguments.

A return follows the final transfer in the representation. Whether that transfer returns, and whether normal completion is reachable, remain documented_only.

### Preconditions

Each reached call requires a usable executable target; each reached memory access requires appropriately accessible storage. No explicit checks establish these conditions.

Low-address memory may need to be accessible. Zero-valued calculated addresses are not rejected. No object-size, alignment, non-null, ownership, or non-aliasing guarantee is established.

## Behavior

### Indirect-call prefix

The prefix is a long, straight-line call sequence with no represented conditional branch, retry, or loop. Calls with no represented arguments remain observable calls; they cannot be discarded as harmless.

The first call supplies a combined result. Its next consumer receives four arguments, in order:

- The low word shifted left by 8.
- The high word unchanged.
- The low word shifted left by 15.
- That same low-word shift by 15 again.

Later consumers receive combinations of freshly returned results, retained earlier results, secondary result-channel values, and explicit zeros. Many scalar results are shifted left by 8. Other arguments use shifts, masks, or combinations of both. Some calls consume another call’s scalar result immediately; others reuse a value across many intervening calls.

The distinction between retained and fresh values is contractual. A later combined result does not replace an older high-word field still used by another argument. Incompletely recovered secondary results are unresolved inputs, not implicit zeros.

Visible transformations include left shifts up to 31, unsigned right shifts, masks selecting low-order fields, and extraction from combined results above bit 31. Intermediate widths matter. Under the conditional word model, a left shift by 28 followed by another left shift by 28 yields zero; it is not a single unbounded shift by 56. Masking an already shifted value is not interchangeable with masking the original input.

The call prefix does not establish callee signatures, preservation of globals, ownership behavior, or successful return.

### Memory-suffix roles

The following descriptive roles identify retained values without assigning object types:

| Role | Meaning |
|---|---|
| **Terminal pair** | Last combined result before direct mutation |
| **Packed base** | Earlier low result word shifted left by 25 |
| **Scaled base** | Earlier scalar result multiplied by 0x02000000 |
| **Small index** | A scalar shifted left by 8, masked to its low 15 bits, then unsigned-shifted right by 6 |
| **Doubled index** | Twice the small index |
| **Retained payload** | Separate earlier scalar result shifted left by 8 |
| **Converted-global base** | Value associated with 0x05 after conversion to unsigned 32 bits |
| **Old word** | Word captured from converted-global base plus 0x54 before replacement |

The small index selects bits 6–14 of the already shifted value, not the original scalar’s low 15 bits before shifting. Under the conditional word model, it is a multiple of four from 0 through 0x1FC.

### Opening mutation and reread boundary

The opening effects preserve this order.

A word store publishes the packed base at terminal high word plus 0x14. A byte at packed base plus 9 receives the terminal high word’s low byte. A word at terminal low word multiplied by 256, plus 0x54, receives the scaled base. The byte at packed base plus 9 is then overwritten with the packed base’s low byte.

Next, a word zero is stored at scaled base plus 0x14. An explicit unsigned byte is captured from scaled base plus 5. A word zero is stored at doubled index plus 0x14, followed by a word store of the captured, zero-extended byte at scaled base plus 0x34.

The word at scaled base plus 0x14 is then reread. **After this reread, another word zero is stored at doubled index plus 0x14, before the two identical byte stores at doubled index plus 0x15.** Both byte stores use the low byte of the captured reread value.

The second zero store is a separate event. Its four-byte range overlaps the subsequent byte destination at offset 0x15. It may also overlap other calculated storage. It does not change the already captured reread value.

Conversely, the reread cannot be replaced with the earlier zero: intervening stores may overlap scaled base plus 0x14.

### Converted global and retention of the old word

After the duplicate byte stores, the value associated with fixed address 0x05 is read and converted to unsigned 32 bits. That converted quantity becomes an address base without an established byte-range restriction.

A byte at fixed address 0x01 receives the doubled index’s low eight bits. The converted-global base’s offset-0x15 byte receives the small index’s low byte.

The scaled base’s offset-0x11 byte receives the retained payload’s low byte. A word at the earlier reread value plus 0x54 receives the retained payload, after which scaled base plus 0x11 is overwritten with zero.

Only then is the **old word** captured from converted-global base plus 0x54. That same word location is subsequently replaced with the earlier reread value. Later arithmetic uses the retained old word, not a fresh load of the replacement.

### Pointer-shaped updates and fixed-address writes

The following region combines the doubled index, the scaled base, the old word, and loaded address-like values. It includes self-relative word stores, byte overwrites, and accesses at offsets 0x14, 0x24, 0x34, 0x44, 0x54, 0x64, and 0x74.

A word loaded from scaled base plus 0x74 becomes another destination base. The doubled index’s offset-0x74 location is also read separately for a nested update: the representation uses two reads from that location for the value and destination of a word store at offset 0x54. No stable relationship or single-read snapshot is established for those two reads.

The fixed-address updates occur at specific boundaries:

- After repeated self-relative stores involving old word plus 0x24 and byte stores at the loaded destination’s offset 9, doubled index plus 0x14 receives the earlier scaling scalar shifted left by 26.
- **Next, a 32-bit store at fixed address 0x54 receives the retained old word shifted left by one.**
- Doubled index plus 0x14 then receives the old word twice, in separate word stores.
- Two byte stores at doubled index plus 0x15 receive the low byte of four times the old word.
- **Next, an 8-bit store at fixed address 0x15 receives the low eight bits of the doubled index.**
- Subsequent computed accesses publish the retained payload at four times the old word plus 0x54, and four times the old word at doubled index plus 0x34.

Neither fixed-address update is merely an implicit input or an eventual final-state property. Both are ordered writes that can overlap calculated destinations or later sources. The write at 0x54 uses the retained old word even though its original location has already been overwritten.

Later accesses multiply loaded words by two, four, or eight. An explicit unsigned byte load from a calculated offset-5 location supplies a small address base for an offset-0x74 word load. Subsequent stores through these loaded values repeatedly replace word and byte fields. The representation does not establish object boundaries or pointer validity.

### Constants and publication-base capture

For this region, **link word** means the word obtained from the later unsigned-byte-derived base at offset 0x74. **Doubled link** and **fourfold link** mean twice and four times that retained word.

Three stores publish 0x3A using different widths and destinations:

- A byte at fourfold link.
- A halfword at eight times the link word.
- A word at fourfold link plus 0x20.

A separate **adjusted base** is four times the old word minus 0x96. The **constant base** is four times this adjusted base. Arithmetic remains subject to the width qualifications above.

At constant base plus 0x20, word-store values occur in this order: 0x20, 0x2F, a zero-extended byte read from constant base plus 4, and 0x20 again. The byte is read between the second and third stores, not at entry.

**After these calculated accesses, the publication global word is read and retained. Only afterward is the 32-bit constant 0x20 stored at fixed address 0x60.**

This capture-before-write order is required. Earlier calculated accesses may overlap the publication global and therefore affect the captured value. The later store at 0x60 must not retroactively change that value, even if its storage overlaps the global.

### Repeated publication

The retained publication base is reused without a represented reload throughout the repeated groups.

Each group contains, in order:

- A word store of the fourfold link at a scaled destination plus 0x54.
- A byte store of the constant base’s low byte at publication base plus 0x19.
- A byte store of the publication base’s low byte at constant base plus 0x11.
- A word store of the scaled destination base at publication base plus 0x34.

The initial group uses eight times the old word as its scaled destination base. Subsequent groups use eight times the adjusted base instead. They repeat without visible conditional termination.

The exact group cardinality remains documented_only in this contract. This does not authorize collapsing the sequence to one group or to final memory contents. Repeated writes and their ordering remain represented effects.

### Final memory suffix and transfer

After publication, a word is captured from fixed address 0xA9, followed by a word from fourfold link plus 0x74. A byte at fixed address 0x3A then receives the low-byte result of eight times the adjusted base plus 0x30.

Further snapshots capture words at doubled link plus 0x14 and constant base plus 4 before subsequent updates.

The final region includes:

- A byte constant 0x6C at constant base plus 8.
- Byte writes at offsets 9, 0x0D, 0x10, and 0x11.
- A word store of the doubled link at its own offset 0x54.
- A byte at fixed address 0x67 receiving the low portion of the captured 0xA9 word.
- A later assignment from the then-current 0x67 value to fixed address 0x21.
- Word copies involving offsets 0x10, 0x40, 0x44, 0x54, and 0x74.
- Restoration of the earlier doubled-link offset-0x14 snapshot into doubled link plus 0x54.

Reads from fixed address 0x67 must retain their positions. Calculated writes may overlap that address between its initial update and later uses.

A byte at doubled link plus 1 receives a right-shift result. Its operand is a loaded signed word; its count is the constant base’s low byte. Negative-operand behavior and counts outside the valid word-shift range remain documented_only.

Near the end, a word loaded from doubled link plus 4 supplies the base for a halfword store of 0x65 at twice that value. Later word loads from its offsets 0x64 and 0x74 are distinct from that halfword write. Another calculated offset-0x54 destination receives decimal 100. The last direct word store uses a destination formed by doubling a loaded word.

The final unresolved transfer receives:

1. 0x72.
2. The word recently loaded from the offset-0x64 location.
3. The earlier snapshot from doubled link plus 0x14.
4. The word recently loaded from the offset-0x74 location.

No externally meaningful success state is established.

## Side effects

### Ordering and aliasing

The suffix is non-atomic. Fixed-address and computed storage are not established as disjoint. Byte and halfword writes may partially overlap words; earlier writes may alter later loaded values or destination addresses.

Trace equivalence therefore cannot be established merely by comparing final memory. The representation does not justify removing overwritten stores, merging repeated writes, widening narrow accesses, reusing earlier reads, or moving global captures across writes.

No synchronization, barrier, transaction, or rollback is visible. Whether individual represented accesses correspond to independently observable physical transactions remains documented_only.

### Ownership and lifetime

No direct allocation or release operation is identified. Unresolved callees may nevertheless allocate, release, retain, or invalidate memory.

Address-like values and self-relative stores do not establish ownership transfer. No lifetime or global-preservation guarantee is assigned to an unknown callee.

### Errors

No explicit error result, bounds check, null check, status-dependent recovery branch, or cleanup path is represented.

A faulting access or failing transfer may follow partial mutation. Neither continued execution nor restoration of previous state is promised.

## Dependencies

The supplied map identifies only F-c3b0ba41b00e. It supplies no opaque IDs for the indirect targets; this contract does not invent identities for them.

Unresolved indirect dependencies include the execution-context-relative entry target, embedded-relative targets, result producers and consumers, and the final transfer target.

Their signatures, calling conventions, result-channel meanings, memory effects, ownership behavior, global preservation, and normal-return behavior remain documented_only.

## Acceptance examples

These examples are conditional static consequences, not executed tests. Memory examples require all preceding operations to complete and the stated arithmetic model to apply.

### Combined result arguments

If the first combined result has low word 0x00000001 and high word 0x12345678, its next consumer receives 0x00000100, 0x12345678, 0x00008000, and 0x00008000, in that order.

### Intermediate-width truncation

A low word of 0x01000000 shifted left by 8 becomes zero under the conditional word model. Retaining a bit above bit 31 would not match that model.

### Small-index boundary and intervening zero

For an index-producing scalar of 0xFFFFFFFF, the small index is 0x1FC and the doubled index is 0x3F8. The relevant word destination is 0x40C and byte destination is 0x40D.

After the scaled-base reread, the trace includes a word zero at 0x40C before both byte stores at 0x40D. Those byte stores use the captured reread value, even if the intervening zero overlaps its original source.

The later fixed-address byte write at 0x15 receives 0xF8, the doubled index’s low byte.

### Retained old word

If the captured old word is 0x00123456, the fixed-address word write at 0x54 receives 0x002468AC under the conditional model. Replacing the original source location before that write does not change the retained operand.

### Unresolved width at 0x05

If the converted value associated with 0x05 is 0x00000120, the represented base is 0x120, not 0x20. Whether that converted value can arise from the actual undeclared storage is documented_only.

### Publication capture before 0x60

If the publication global yields base 0x2000 at its capture, later publication uses 0x2000 even if the subsequent word store at 0x60 overlaps and changes that global’s storage. Changes caused by preceding calculated writes, however, are visible to the capture.

### Explicit unsigned byte widening

An explicit unsigned byte load yielding 0xFE produces word value 0x000000FE when widened, not 0xFFFFFFFE. This rule does not establish the load width at 0x05.

### Failure after mutation

If an access faults after earlier stores complete, acceptance cannot require rollback. No rollback mechanism is established.

## Uncertainties

The following remain **documented_only**:

- Application purpose and fidelity of the represented boundary and control flow.
- Indirect-target identities, signatures, and result conventions.
- Provenance of incompletely recovered secondary results.
- Original width and signedness of storage at 0x05.
- Endianness, alignment, address wraparound, and signed-overflow behavior.
- Variable right-shift semantics for negative operands or large counts.
- Validity of low-address accesses and calculated pointers.
- Object layouts, alias restrictions, and physical access observability.
- Exact repeated-publication cardinality.
- Callee allocation, release, retention, and global mutation.
- Final-target identity, return behavior, and meaningful completion state.

**Inferred intent:** recurring offsets and self-relative stores may indicate structured-state manipulation. Printable constants may indicate text-related behavior. Neither inference imposes a data model or format.

## Evidence

The sole behavioral evidence is the supplied static representation. It establishes represented access widths, result transformations, ordered captures and overwrites, fixed-address updates, repeated publication, and the final four-argument transfer.

In particular, the represented trace places the second doubled-index zero store after the scaled-base reread, retains the old word before replacing its source, writes the fixed addresses 0x54 and 0x15 at the boundaries described above, and captures the publication global before writing 0x60.

Comments, warnings, prior prose, and reviewer feedback are not independent evidence. Missing declarations and unresolved interfaces remain unresolved.

No implementation access, executable inspection, runtime observation, or runtime tests have been performed.
