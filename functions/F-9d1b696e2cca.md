<!-- Trusted provenance: F-9d1b696e2cca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9d1b696e2cca

## Purpose

F-9d1b696e2cca performs an ordered sequence of unresolved indirect dependency invocations, transforms their results, mutates memory through computed and fixed addresses, and ends with an unresolved indirect transfer.

**Established static evidence:** the supplied representation exposes three 32-bit inputs, no declared scalar result, dependency results represented at 32 and 64 bits, direct memory accesses at 8-, 16-, and 32-bit widths, repeated stores, and four explicit arguments to the final transfer.

**[INFERENCE] Intent:** some stores resemble link construction or state relinking. Several numeric constants have printable character interpretations. Neither observation establishes an object model, ownership protocol, text format, or output device.

This is a static, partially **documented_only** contract. It preserves established dataflow and important access-order requirements without claiming a complete executable specification of unresolved dependencies or machine semantics.

## Interface

### Explicit inputs

| Input | Represented width | Visible use |
|---|---:|---|
| First argument | 32 bits | No explicit body-level use. |
| Second argument | 32 bits | No explicit body-level use. |
| Third argument | Signed 32 bits | Initially shifted left by three; that transformed value supplies later dependency arguments. |

The absence of explicit uses for the first two inputs does not prove their irrelevance at machine level. The first invocation has an unresolved interface and may consume incoming machine state not expressed as arguments.

### Implicit inputs

Execution additionally depends on:

- Incoming control context used to derive the first indirect target.
- Further targets derived from embedded locations using a displacement of 4094 bytes.
- Primary and secondary dependency results.
- Fixed-location values, including samples at addresses 5, 169, and 0x087fbbbc.
- Computed-address memory contents, including nested word and byte indirections.
- Ambient state consumed or changed by unresolved dependencies.

### Outputs

No scalar return value is declared. Observable outputs are dependency interactions, direct memory accesses, resulting state changes, and the final transfer.

The final transfer has four explicit arguments:

1. The 32-bit value 114.
2. A saved 32-bit word loaded at offset 100 from a late-loaded base.
3. A saved 32-bit word loaded earlier at offset 20 from another computed base.
4. A later 32-bit word loaded at offset 116 from the same base used for the second argument.

The precise origins and ordering are specified below. No text conversion or object-handle interpretation is established.

A nominal return follows the transfer in the representation. Whether the transfer returns and whether normal completion is reachable are **documented_only**.

### Preconditions

For execution to progress, each reached target must be callable and every reached memory access must support its width, address, alignment, and permissions.

No null checks, address-range validation, buffer lengths, ownership tokens, or error-recovery interface are visible. Several literal low-address accesses are incompatible with an environment that rejects all such addresses.

## Behavior

### Arithmetic interpretation

The representation distinguishes bytes, halfwords, words, and 64-bit dependency results.

Where numeric consequences are given, they use a **conditional 32-bit bit-pattern interpretation**:

- Left shifts, additions, subtractions, and scaling retain the low 32 bits.
- Unsigned right shifts insert zeros.
- Byte and halfword stores retain the low eight and sixteen bits respectively.
- Widening a byte to a word produces a value from 0 through 255.
- Intermediate 32-bit operations do not silently widen.

This interpretation is not proof of the actual execution semantics of signed overflow, signed left shifts, pointer arithmetic, or excessive shift counts.

The late signed right shift uses a count derived from a low byte, potentially exceeding 31. Treatment of negative operands and out-of-range counts remains **documented_only**. A replacement cannot silently adopt an arbitrary host-language shift rule.

### Dependency interaction phase

The initial invocation obtains a result represented as 64 bits. A subsequent invocation receives its low 32 bits shifted left by eight, its high 32 bits, and the third input shifted left by three.

The remaining dependency phase is visibly straight-line and ordered. Invocations with no explicit arguments alternate with invocations receiving transformed results, saved values, repeated arguments, and zeros. Some invocations expose secondary results whose complete ABI meaning is unresolved.

Visible transformations include:

- Left shifts of 32-bit quantities.
- Unsigned right shifts and low-bit masks.
- High-half extraction from represented 64-bit results.
- Extraction after unsigned 64-bit shifts by 34, 36, and 44.
- Retention of earlier results across intervening invocations.
- Repeated use of one transformed value in multiple argument positions.

The 64-bit extractions after shifts by 34, 36, and 44 contain at most 30, 28, and 20 significant bits respectively. They are not interchangeable with ordinary high-half extraction.

Some chains originate from explicitly assigned zero and remain zero under the conditional arithmetic model. That establishes particular explicit argument values, not the absence of dependency side effects.

Calls must not be discarded merely because their results are unused or their explicit arguments are zero. Their state changes, errors, and possible non-returning behavior are unknown. The complete invocation-by-invocation argument ledger is **documented_only** here; this document does not invent missing dependency contracts.

### Direct-memory terminology

The following names describe saved values, not recovered types or source identifiers:

| Contract term | Meaning |
|---|---|
| Shifted token | A dependency result shifted left by 25. |
| Scaled base | Another dependency result multiplied by \(2^{25}\). |
| Saved shifted value | Another dependency result shifted left by eight. |
| Compact selector | A result shifted left by eight, masked to its low 15 bits, then shifted unsigned right by six. |
| Selector base | Twice the compact selector. |
| Final pair | Low and high 32-bit halves of the last represented 64-bit result before direct stores. |
| Low-global base | The saved, 32-bit-expanded value sampled at address 5. |
| Preserved link | The word sampled at low-global base plus 84 before that location is overwritten. |

Under the conditional interpretation, the compact selector is a multiple of four from 0 through 508. The selector base is a multiple of eight from 0 through 1016.

The original access width of the named value at address 5 is not independently declared in the supplied material. Its expansion and subsequent use as a 32-bit address base are visible; its original width remains **documented_only**.

### Initial stores and first global sampling boundary

The initial mutations preserve these ordering relationships:

- A word store of the shifted token at final-pair high half plus 20 precedes a byte store of that high half’s low byte at shifted token plus 9.
- A word store of the scaled base at the final-pair low half scaled by 256, plus 84, precedes another byte store at shifted token plus 9.
- That second byte store uses the shifted token’s low byte, which is zero under the conditional interpretation.
- A zero word store at scaled base plus 20 precedes a byte read at scaled base plus 5.
- A zero word store at selector base plus 20 precedes a word store of the zero-extended sampled byte at scaled base plus 52.
- A word read at scaled base plus 20 precedes another zero word store at selector base plus 20.
- The sampled word’s low byte supplies two consecutive stores at selector base plus 21.

Aliasing can affect the word read at scaled base plus 20. It cannot generally be replaced by the earlier zero constant.

**The value at address 5 is sampled next, before the byte write at address 1.** Its saved value becomes the low-global base. The byte at address 1 then receives the selector base’s low byte.

Subsequent operations use the saved low-global base, not an implicit resampling of address 5. They include a byte store at offset 21 and, later, a word read at offset 84. Between the sample and that word read, other computed stores occur, including stores through the scaled base and through the earlier offset-20 read result.

The word at low-global base plus 84 is saved as the preserved link; that same word location is then overwritten with the earlier offset-20 read result. Moving the read after the overwrite, or replacing the saved base with a later address-5 sample, changes established behavior.

### Link-like mutations and overlapping layout

The following section uses the preserved link, its multiples by two, four, and eight, the selector base, and additional loaded words as addresses and stored values.

Visible access offsets include:

- Byte offsets 1, 5, 9, 13, 17, 21, and 25.
- Word offsets 4, 20, 36, 52, 68, 84, 100, and 116.

A byte at offset 21 overlaps a word beginning at offset 20. A byte at offset 17 overlaps a word beginning at offset 16. These are not independent fields without additional layout and aliasing assumptions.

Nested accesses include a byte read whose numeric value becomes the base for a word read at offset 116. Another operation uses a word read at selector base plus 116 to select an offset-84 destination and separately reads selector base plus 116 for the stored word. These are distinct reads; equality requires stability not established here. Their relative evaluation order within that operation is **documented_only**.

Several words receive their own address base as a value. Repeated word stores and byte stores remain distinct visible mutations. Fixed-location effects in this portion include a word store at address 84 and a byte store at address 21.

### Numeric writes and second global sampling boundary

For the remaining description:

- **Adjusted base** is four times the preserved link, minus 150.
- **Working base** is four times the adjusted base.
- **Nested value** is a later word obtained through a byte-based offset-116 indirection.
- **Doubled nested base** is twice the nested value.
- **Quadrupled nested base** is four times the nested value.

All scaling is conditional on the arithmetic interpretation above.

The numeric value 58 is stored as a byte at quadrupled nested base, as a halfword at eight times the nested value, and as a word at quadrupled nested base plus 32. No character stream is established.

Working base plus 32 subsequently receives four word stores, in order:

1. 32.
2. 47.
3. A zero-extended byte sampled at working base plus 4.
4. 32.

**The separate 32-bit global at address 0x087fbbbc is sampled after this four-store sequence and before the word store of 32 at address 96.** Its saved value is the **repeated-section base**.

The repeated mutation section uses that saved base throughout. It does not repeatedly reload the global. Consequently:

- Earlier computed stores may affect the global before sampling.
- The address-96 store occurs too late to change the saved sample.
- Later stores that alias the global do not retroactively change the repeated-section base.

The repeated groups contain an ordered word store at a scaled destination plus 84, a byte store at repeated-section base plus 25, a byte store at working base plus 17, and a word store at repeated-section base plus 52.

The initial group uses eight times the preserved link as its scaled destination base and final stored link value. Subsequent groups use eight times the adjusted base. Their offset-84 word value is the quadrupled nested base. The two byte values are the low bytes of the working base and repeated-section base respectively.

The repetition is explicit and finite, not an established data-dependent traversal. Its exact occurrence count is **documented_only** in this contract. Collapsing it to a single final-state update is unsupported.

### Final memory phase: saved word versus byte resampling

The following boundaries are established:

- A **32-bit word at address 169 is sampled and saved**.
- A word at quadrupled nested base plus 116 is then sampled.
- A byte is then written at address 58, using the low-byte arithmetic result of eight times the adjusted base plus 48.
- Words at doubled nested base plus 20 and working base plus 4 are subsequently saved.

The address-169 word is not a byte sample from address 103. It remains a saved 32-bit value used later in a word store at working base plus 68.

After byte stores at doubled nested base plus 13 and working-base offsets 8, 13, and 9, and a self-valued word store at doubled nested base plus 84, **address 103 receives the low byte of the saved address-169 word**. Working base plus 8 receives 108.

Address 103 is subsequently sampled three separate times:

| Sampling point | Consumer | Required intervening boundary |
|---|---|---|
| First byte read | Byte store at address 33 | After a word read at doubled nested base plus 68. |
| Second byte read | First later store at working base plus 9 | After intervening byte/word mutations, including a word store at repeated-section base plus 68. |
| Third byte read | Second later store at working base plus 9 | After a byte store at doubled nested base plus 9, the word store of the saved address-169 value at working base plus 68, and a word read at working base plus 84. |

These are fresh memory reads, not guaranteed copies of the original low byte. Intervening computed stores may alias address 103. The two working-base-plus-9 stores therefore need not match one another or the value written to address 33.

Other established transitions in this interval include:

- Saving the word at doubled nested base plus 68 before using it in later stores and the signed shift.
- Writing a word at repeated-section base plus 84 twice, first from that saved word and then from the working base.
- Reading working base plus 64 before writing that sampled word at working base plus 16.
- Writing the earlier saved working-base-plus-4 word at repeated-section base plus 116.
- Preserving the earlier doubled-nested-base-plus-20 word across later mutations.

### Final linking and transfer

The word sampled at working base plus 84 becomes a **late link base**. After the third address-103 byte read:

- The saved quadrupled-nested-base-plus-116 word supplies low-byte stores at late link base plus 13 and doubled nested base plus 17.
- Doubled nested base plus 84 receives the saved doubled-nested-base-plus-20 word.
- A fresh word at working base plus 20 supplies a later low-byte store at late link base plus 17.
- Working base plus 9 receives the late link base’s low byte, and working base plus 68 receives that base as a word.

A word read at doubled nested base plus 4 then establishes the **final lookup base**. A halfword of 101 is stored at twice that base before the word read at final lookup base plus 100.

The word at offset 100 is saved as the second final-transfer argument. After a byte store at late link base plus 17 and a word store of 100 at late link base plus 84, a fresh word at final lookup base plus 116 becomes the fourth argument.

A further word read at doubled nested base plus 100 supplies another destination base. Subsequent effects include a byte store at doubled nested base plus 9, a word store through that destination plus 84, and a word store of the destination base at twice the second final-transfer argument.

The transfer then receives 114, the saved offset-100 word, the preserved doubled-nested-base-plus-20 word, and the later offset-116 word.

## Side effects

Direct memory mutations are nontransactional in the visible representation. No locks, atomic operations, rollback, or cleanup path are established.

Saved scalar values remain distinct from later memory samples. This distinction applies especially to the address-5 base, the 0x087fbbbc base, the address-169 word, and the three later address-103 byte reads.

Endianness is unknown. Access widths and offsets are established, but overlapping word and byte effects depend on byte order. Aliasing, concurrent modification, memory-mapped behavior, and faults can make intermediate accesses observable.

No allocation, destruction, retention, release, or ownership transfer is identified. Dependencies may change globals or object lifetimes; preservation cannot be assumed.

No visible error result, bounds check, recovery branch, or diagnostic exists. A failing dependency or unsupported access may prevent later operations after partial side effects. No rollback is promised.

## Dependencies

Only the subject’s opaque ID, **F-9d1b696e2cca**, is supplied. No mapped opaque IDs identify the indirect targets, so none are fabricated.

| Unresolved dependency class | Established interaction | Unknowns |
|---|---|---|
| Incoming-context target | Initial invocation; represented 64-bit result | Identity, full arguments, ABI, state effects |
| Embedded-location-derived targets | Ordered invocations with zero through four explicit arguments | Target equivalence, secondary results, errors, ownership and global effects |
| Final target | Four explicit 32-bit arguments | Identity, transfer semantics, return behavior, external result |

An invocation with no explicit arguments is not proof of an argument-free machine interface. No dependency is assumed pure, memory-preserving, global-preserving, or ownership-neutral.

## Acceptance examples

These are conditional review examples, not executed tests.

- **Input transformation:** under the conditional arithmetic model, a third input of 1 initially becomes 8. The early invocation receiving two copies shifted a further ten bits receives 8192 in both positions.

- **Selector boundary:** a selector-producing result with its low seven bits set gives compact selector 508 and selector base 1016. Selector base plus 20 is address 1036; accessibility is not implied.

- **First saved global:** if address 5 supplies base 4096 at its sampling point, later mutation of that global does not change the saved base. The offset-84 read uses address 4180 and precedes the overwrite there.

- **Second saved global:** if 0x087fbbbc supplies base 8192 after the working-base-plus-32 sequence, the repeated section uses 8192 even if a later computed store changes the global. Sampling before that sequence or after the address-96 store violates the specified boundary.

- **Aliasing-sensitive byte reads:** if a computed store changes address 103 from 0x12 to 0x34 between the first and second byte samples, address 33 may receive 0x12 while the first later working-base-plus-9 store receives 0x34. Reusing the original saved low byte fails this requirement.

- **Saved word distinction:** the word at address 169 remains the source for its later full-width store even when address 103 changes. Conversely, the address-103 consumers use fresh bytes, not that saved word.

- **Observable intermediate stores:** an observer of working base plus 32 can distinguish stores of 32, 47, the byte-derived word, and 32. Matching only final contents is insufficient.

- **Unsupported access:** an inaccessible computed destination does not require an invented error code, fallback, or undo operation.

## Uncertainties

The following remain **documented_only**:

- Actual function boundary, complete calling convention, and incoming live arguments.
- Indirect dependency identities, secondary-result semantics, and effects.
- Signed arithmetic, address wrapping, and excessive-shift-count behavior.
- Original width of the address-5 named value.
- Endianness, alignment, address validity, and memory-map interpretation.
- Relative evaluation order of separate reads within the nested offset-116 assignment.
- Complete dependency argument ledger and exact repeated-group count.
- Data structure types, ownership, lifetime, concurrency, and reentrancy.
- Application purpose, external output, normal termination, and final-target return.

These limits do not make the established global sampling boundaries or address-103 resampling optional.

## Evidence

This contract is based solely on static inspection of the supplied representation of F-9d1b696e2cca. Prior prose, comments, and feedback are not independent behavioral evidence.

Established observations include represented widths, explicit transformations, memory-access offsets, saved-value dataflow, fixed-location sampling boundaries, repeated stores, and final-transfer arguments. Link-like intent is inference; unresolved dependency and machine behavior remain explicitly qualified.

**No runtime tests have been performed.** Successful execution, memory safety, dependency behavior, concrete external output, and a reachable normal return have not been observed.
