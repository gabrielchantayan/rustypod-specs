<!-- Trusted provenance: F-efbbf100a972; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-efbbf100a972

## Purpose

F-efbbf100a972 performs an ordered sequence of indirect calls and integer transformations, followed by width-specific memory mutations through fixed addresses, scaled values, and loaded links. It finishes with an unresolved indirect dispatch carrying four represented arguments.

**Established static evidence:** the supplied representation declares no explicit parameters or return value. The enclosing routine has no visible conditional branches, address validation, bounds checks, or rollback path. Successful completion depends on indirect calls returning and all reached memory accesses being supported.

**[INFERENCE] Intent:** repeated offsets and cross-linked stores suggest manipulation of structured state. They do not establish an object model, application protocol, device interface, or text operation. Constants resembling character values do not establish character output.

This contract describes the supplied static representation, not verified execution or memory safety.

## Interface

### Inputs and outputs

There are no represented explicit arguments. Effective inputs include:

- Incoming execution context determining the first indirect target.
- Results of indirect calls, including 32-bit values and represented 64-bit paired results.
- Auxiliary call-associated machine-state values whose production is incompletely specified.
- Memory at fixed and calculated addresses.
- One captured 32-bit publication-global value.
- The unresolved terminal dispatch target.

The actual calling convention and hidden entry requirements are **documented_only**.

There is no represented result value. Outputs consist of dependency activity, direct memory accesses, and the final dispatch. Its represented arguments, in order, are:

| Position | Value |
|---|---|
| First | Integer `0x72` |
| Second | Terminal link, captured from Terminal base plus `0x64` |
| Third | Saved word, captured earlier from Restoration base plus `0x14` |
| Fourth | Word captured from Terminal base plus `0x74` after the terminal object stores |

A return is represented after dispatch. Whether the dispatch target returns is **documented_only**.

### Widths, arithmetic, and layout

Byte, halfword, and word accesses are respectively 8, 16, and 32 bits. Paired results are represented as 64-bit quantities. All offsets are byte offsets.

Unsigned byte-to-word promotion is zero-extension. Byte and halfword stores retain the low eight and sixteen bits respectively. Observed offsets identify access locations, not declared fields or allocation boundaries. A word at offset `0x14` overlaps a byte at `0x15`; byte order is not established.

Arithmetic relationships below describe represented operations. Where stated, a **conditional 32-bit wrapping-bit model** reduces results modulo \(2^{32}\). This model is not a claim that signed-looking overflow, address calculations, or shifts have established portable semantics.

Ordinary completion requires a usable address graph, including supported low fixed addresses, byte-derived addresses, scaled addresses, and potentially unaligned word accesses. The routine does not locally enforce these requirements.

## Behavior

### Dependency phase and captured values

The initial phase is a long straight-line chain of indirect calls interleaved with transformations of earlier results. The first target depends on incoming context; later targets are represented through calculated addresses.

Represented arguments include shifted results, separate components of paired results, duplicated transformed arguments, zeros, masked fields, and saved values reused after intervening calls. A call whose result is unused remains potentially observable.

Most transformations operate on represented 32-bit values. Extraction shifts of 34, 36, and 44 bits operate on 64-bit quantities. Masks applied before shifts remain significant.

Under the conditional wrapping-bit model:

- Left shifts discard bits above bit 31.
- A left shift by eight followed by a logical right shift by eight retains the original low 24 bits.
- Successive left shifts totaling at least 32 bits produce zero.
- Multiplication by `0x02000000` depends only on the operand’s low seven bits.
- A value shifted left by sixteen has a zero low halfword.

The following descriptive labels identify capture roles, not implementation identifiers:

| Label | Meaning |
|---|---|
| Seed | Unscaled 32-bit call result subsequently multiplied by `0x02000000` |
| Anchor | Seed multiplied by `0x02000000` |
| Tag | A separately captured call result shifted left by eight |
| Packed base | Low component of an earlier paired result shifted left by 25 |
| Index | Another call result shifted left by eight, masked with `0x7FFF`, then logically shifted right by six |
| Index base | Twice Index |
| Final low / Final high | Low and high components of the last paired result before direct mutations |
| Displaced | Word captured from an initial byte-selected base plus `0x54` |
| Scaled base | Four times Displaced |
| Eightfold value | Eight times Displaced |

Thus, **Seed shifted left by 26 is calculated from the unscaled call result, not from Anchor**.

Under the conditional model, Index ranges from zero through 508 in multiples of four; Index base ranges from zero through 1016 in multiples of eight. These values are nevertheless used as addresses without visible validation.

### Opening mutations

The initial stores publish Packed base as a word at Final high plus `0x14`, write Final high’s low byte at Packed base plus `9`, and write Anchor at 256 times Final low plus `0x54`. Packed base plus `9` then receives Packed base’s low byte. Both byte writes are represented effects.

Anchor plus `0x14` is cleared before a byte is captured from Anchor plus `5`. Index base plus `0x14` is then cleared, followed by a word store of the zero-extended captured byte at Anchor plus `0x34`.

The word at Anchor plus `0x14` is captured as **Captured anchor word**. Index base plus `0x14` is cleared again, and Captured anchor word’s low byte is written twice at Index base plus `0x15`. Captured anchor word is not unconditionally zero: intervening destinations can alias its origin.

A byte read at fixed address `0x05` supplies **Initial byte-selected base**, zero-extended for address use. After that capture, fixed byte address `0x01` receives Index base’s low byte, and Initial byte-selected base plus `0x15` receives Index’s low byte.

Anchor plus `0x11` receives Tag’s low byte. Tag is then stored as a word at Captured anchor word plus `0x54`, and Anchor plus `0x11` is cleared. Displaced is captured from Initial byte-selected base plus `0x54` before that location receives Captured anchor word. Index base plus `0x34` subsequently receives Index.

### Initial linked and scaled changes

A word captured from Anchor plus `0x74` becomes **Anchor link**. Anchor link plus `0x54` receives Seed shifted left by 26.

A separate linked self-value store uses two represented word reads at Index base plus `0x74`: one supplies its destination base and one supplies its stored value. Both reads precede the resulting word store at the destination base plus `0x54`; their relative evaluation order is **documented_only**.

Displaced is then stored at Displaced plus `0x24`, followed by its low byte at Anchor link plus `9`. Anchor link plus `0x54` receives Tag. The Displaced self-value store and Anchor-link byte store then occur again, in that order.

Index base plus `0x14` receives Seed shifted left by 26. Fixed word address `0x54` then receives twice Displaced, after which Index base plus `0x14` receives Displaced twice. Two byte stores place Scaled base’s low byte at Index base plus `0x15`; fixed byte address `0x15` subsequently receives Index base’s low byte.

Scaled base plus `0x54` receives Tag, and Index base plus `0x34` receives Scaled base. A word read at Index base plus `0x74`, doubled, supplies **Doubled link**. Scaled base plus `0x54` then receives Scaled base.

Scaled base plus `0x15` receives Eightfold value’s low byte before Displaced plus `0x44` receives Doubled link. A byte read at Scaled base plus `5` selects a word read at the byte-derived base plus `0x74`; that word becomes **Earlier selected link**. Its offset `0x44` receives Doubled link.

Scaled base plus `0x15` next receives Scaled base’s low byte. Earlier selected link’s low byte is written at Scaled base plus `0x0D`; Scaled base receives its own value at `0x14` and again at `0x54`.

A word captured from Scaled base plus `0x64` supplies another link. That link receives Earlier selected link’s low byte at `0x0D` and its own base value at `0x14`.

### Distinct-base reload and Selected-link boundary

A word read at Scaled base plus `4` captures **Distinct base**. Before the first byte read at Distinct base plus `5`, Doubled link receives Earlier selected link’s low byte at offset `9`, followed by Eightfold value as a word at offset `0x44`.

The first byte capture at Distinct base plus `5` is followed by these ordered mutations:

- Scaled base plus `9` receives Doubled link’s low byte.
- Earlier selected link plus `0x44` receives the first captured byte as a zero-extended word.
- Scaled base plus `0x14` receives Eightfold value.
- Doubled link plus `0x15` receives Distinct base’s low byte.
- Doubled link plus `0x44` receives Scaled base.

Only afterward is the byte at Distinct base plus `5` read again. The second read is a reload; intervening stores can change its result through aliasing.

**After that reload, Scaled base plus `0x15` receives Eightfold value’s low byte before the selected word is read.** The reloaded byte, zero-extended, supplies the base for the word read at offset `0x74`. That word becomes **Selected link**. Twice Selected link becomes **Restoration base**.

Following the Selected-link capture, the ordered stores are:

- Distinct base’s low byte at Scaled base plus `0x15`.
- Scaled base as a word at Scaled base plus `0x54`.
- Restoration base’s low byte at Scaled base plus `9`.
- Eightfold value as a word at Scaled base plus `0x14`.

The intervening Eightfold-byte store is therefore on the earlier side of the Selected-link capture; these four stores are on its later side.

### Constants and publication

Selected link determines three stores of `0x3A`, in order: a byte at four times Selected link, a halfword at eight times Selected link, and a word at four times Selected link plus `0x20`.

Define **Reduced base** as Scaled base minus `0x96`, **Working base** as four times Reduced base, and **Publication value** as four times Selected link.

Working base plus `0x20` receives four word writes: `0x20`, then `0x2F`, then a zero-extended byte read from Working base plus `4`, then `0x20` again. The byte read occurs after the `0x2F` store and before its byte-derived word store.

**Publication base is captured by one 32-bit global read after these updates and before the word store of `0x20` at fixed address `0x60`.** Publication begins after that fixed-address store.

Each publication unit preserves this access order:

- Word store of Publication value at the unit’s scale value plus `0x54`.
- Byte store of Working base’s low byte at Publication base plus `0x19`.
- Byte store of Publication base’s low byte at Working base plus `0x11`.
- Word store of the unit’s scale value at Publication base plus `0x34`.

The first unit’s scale value is eight times Displaced; subsequent units use eight times Reduced base. Publication base and Publication value remain captured values throughout. Aliasing does not introduce global reloads.

Exact publication multiplicity is **documented_only** in this contract. The supplied representation contains explicit repetitions; this limitation does not permit collapsing them into one unit.

### Post-publication captures and overlaps

The next capture boundary is:

**Word read at fixed address `0xA9` → word read at Publication value plus `0x74` → byte write at fixed address `0x3A` → word read at Restoration base plus `0x14` → word read at Working base plus `4`.**

These captures become **Fixed word**, **Captured link word**, **Saved word**, and **Saved working link**, respectively. The byte written at `0x3A` is the low-eight-bit result of eight times Reduced base plus `0x30`. Its write precedes both Saved-word and Saved-working-link captures.

The following ordered mutations place Working base’s low byte at Restoration base plus `0x0D`, literal byte `0x6C` at Working base plus `8`, and Publication base’s low byte at Working base plus `0x0D` and then plus `9`. Restoration base plus `0x54` then receives Restoration base as a word.

Fixed byte address `0x67` receives Fixed word’s low byte. A word captured from Restoration base plus `0x44` becomes **Shift source**. A subsequent byte read at `0x67` supplies the byte write at fixed address `0x21`.

Working base plus `0x10` receives Shift source’s low byte; Publication base plus `0x54` receives Shift source as a word. Restoration base plus `1` receives a byte derived by right-shifting Shift source using Working base’s low byte as the shift count. Shift signedness and counts of at least 32 remain unresolved. Publication base plus `0x54` is then overwritten with Working base.

A word is captured from Working base plus `0x40`. Publication base plus `0x74` receives Saved working link, Working base plus `0x10` receives the newly captured word, and Publication base plus `0x44` receives Restoration base. The word at Working base plus `0x10` overlaps its earlier byte write.

A fresh byte read at `0x67` supplies Working base plus `9`. Restoration base plus `9` receives Working base’s low byte, and Working base plus `0x44` receives Fixed word.

A word captured from Working base plus `0x54` becomes **Terminal object**. Another fresh byte read at `0x67` supplies another Working-base-offset-`9` write. Terminal object plus `0x0D` receives Captured link word’s low byte.

Restoration base plus `0x54` receives Saved word. This is not established to restore that destination’s earlier contents: Saved word originated at offset `0x14`.

A word captured from Working base plus `0x14` becomes **Terminal byte source**. Restoration base plus `0x11` receives Captured link word’s low byte. Working base plus `9` and plus `0x44` then receive Terminal object’s low byte and full word, respectively.

### Terminal mutation and dispatch

A word read from Restoration base plus `4` captures **Terminal base**. A halfword `0x65` is written at twice Terminal base before a word read at Terminal base plus `0x64` captures **Terminal link**.

Terminal object plus `0x11` then receives Terminal byte source’s low byte, and Terminal object plus `0x54` receives the 32-bit value decimal 100.

**Only after those stores** is the word at Terminal base plus `0x74` captured for the fourth dispatch argument. A later word read at Restoration base plus `0x64` captures **Other terminal link**.

The remaining stores place Terminal base’s low byte at Restoration base plus `9`, Terminal link as a word at Other terminal link plus `0x54`, and Other terminal link as a word at twice Terminal link.

Final dispatch follows all these stores, using `0x72`, Terminal link, Saved word, and the terminally captured offset-`0x74` word. Earlier contents of those locations are not substitutes.

## Side effects

No non-aliasing relationship is established among fixed addresses, scaled destinations, loaded links, or globals. Repeated stores, overlapping widths, and fresh reads are part of the represented access trace, not merely a description of final memory.

Capture-time values remain saved even if subsequent writes modify their originating locations. Conversely, a specified reload cannot be replaced by an earlier capture.

No direct allocation, release, reference counting, or ownership transfer is identified. Address use alone establishes no ownership or lifetime guarantee. Unknown callees must not be assumed to preserve globals or acquire or release memory.

There is no represented error return, cleanup path, or rollback. A non-returning dependency or unsupported access prevents any guarantee that later operations occur; earlier effects are not shown to be undone. Atomicity, synchronization, fault handling, thread safety, and reentrancy are **documented_only**.

## Dependencies

F-efbbf100a972 is the only supplied resolved opaque function identity. No opaque identities are supplied for its indirect dependencies; none are invented or attributed to this function.

Unresolved dependencies comprise the entry-context-derived target, subsequent computed targets, producers of ordinary and paired results, auxiliary machine-state producers, and final dispatch.

Their identities, calling conventions, memory effects, ownership effects, failure behavior, and return guarantees are **documented_only**. Different call sites are not established to reach the same function.

## Acceptance examples

These are conditional criteria, not executed tests.

### Unscaled Seed arithmetic

Under 32-bit wrapping-bit arithmetic, unscaled Seed values `0x00000001` and `0x00000081` both produce Anchor `0x02000000`. Seed shifted left by 26 produces `0x04000000` in both cases. Shifting Anchor instead would produce zero and fails this criterion.

### Distinct-base reload

If intervening stores alter Distinct base plus `5`, the second byte capture observes the altered byte. Earlier selected link plus `0x44` still receives the zero-extended first capture. Reusing that first capture to select Selected link fails the criterion.

### Selected-link interleaving

If the word selected by the second byte capture overlaps Scaled base plus `0x15`, it is read after Eightfold value’s low-byte store. It is read before the subsequent Distinct-base-byte store and the three following stores. Moving either byte store across this capture fails the criterion.

### Publication capture

Acceptance requires one publication-global read before the fixed word store at `0x60`, followed by publication units using that captured base. A post-store read or per-unit reload is not equivalent.

### Fixed-address capture boundary

If the byte write at `0x3A` aliases a later capture location, Saved word and Saved working link observe memory after that write. Fixed word and Captured link word are captured before it.

### Terminal alias sensitivity

If Terminal object plus `0x54` overlaps Terminal base plus `0x74`, the fourth dispatch argument reflects the word read after the decimal-100 store. Preloading terminal links fails this criterion.

### Provenance, widths, and overwrites

A Saved word of `0x12345678` is later stored at Restoration base plus `0x54` and passed as the third dispatch argument, irrespective of that destination’s former contents.

A captured unsigned byte `0xE7` is promoted to word `0x000000E7`. The three Selected-link-derived `0x3A` stores retain their distinct byte, halfword, and word widths. Both opening Packed-base-offset-`9` writes and all four Working-base-offset-`0x20` writes remain represented effects.

## Uncertainties

The following remain **documented_only**:

- Actual calling convention, hidden inputs, and executable reachability.
- Unknown dependency identities, effects, and auxiliary-state production.
- Fidelity of represented widths and paired-result conventions.
- Signed arithmetic, address overflow, and signed-looking shift semantics.
- Variable right shifts with counts of at least 32.
- Byte order, alignment restrictions, and special memory semantics.
- Relative order of the two reads within the linked self-value store.
- Validity and application meaning of low fixed addresses.
- Object layouts, ownership, and lifetime boundaries.
- Exact publication repetition count as enumerated by this contract.
- Final dispatch identity, return behavior, and application-level purpose.

These limits do not make the concrete Distinct-base mutations or Selected-link capture boundary unresolved. Nor do they authorize guessed semantics, omitted repetitions, assumed non-aliasing, or reordered captures.

## Evidence

Evidence consists solely of the supplied static representation. Prior prose, comments, and reviewer assertions are not independent behavioral authority.

The representation supports the distinction between unscaled Seed and multiplied Anchor; the concrete stores surrounding both Distinct-base byte reads; the intervening Eightfold-byte store before Selected-link capture; width-specific mutations; publication-global capture timing; and terminal dispatch argument provenance.

No runtime tests, executable inspection, smoke runs, or memory traces have been performed. This contract records conditional obligations from static evidence, not proof of successful execution, memory safety, dependency purity, or application intent.
