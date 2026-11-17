<!-- Trusted provenance: F-8b6ced299c8f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8b6ced299c8f

## Purpose

F-8b6ced299c8f performs an ordered sequence of indirect interactions and fixed-width transformations, followed by mixed-width memory updates and an unresolved terminal transfer. Its application-level purpose is not established.

The supplied representation does not establish text formatting, collection traversal, allocation, object initialization, or a named protocol. Constants that resemble character values do not establish an encoding.

This contract distinguishes:

- **Established static evidence:** represented accesses, conversions, capture boundaries, and interaction order.
- **[INFERENCE]:** consequences conditional on an explicitly stated arithmetic or execution model.
- **documented_only:** behavior not established, or trace detail expressly left unquantified.

All described effects are conditional on execution reaching them. No runtime tests have been performed.

## Interface

### Explicit inputs

Four positional inputs are represented.

| Position | Represented width | Explicit use |
|---|---:|---|
| First | 32 bits | No explicit body use |
| Second | 32 bits | No explicit body use |
| Third | Signed 32 bits | Passed to an indirect interaction after a four-bit left shift |
| Fourth | Signed 32 bits | Passed after a 25-bit left shift, and elsewhere as two copies of its seven-bit-left-shifted value |

The fourth input’s first 25-bit-shifted use precedes its duplicated seven-bit-shifted use. The third input’s shifted use follows those interactions. Another 25-bit-shifted use of the fourth input occurs later.

No explicit body use of the first two inputs does not establish their irrelevance to execution: incomplete indirect interfaces leave hidden register dependencies unresolved.

### Implicit inputs

Execution depends on incoming return-link state, indirect-call results, secondary result channels, fixed-address storage, and memory reached through calculated addresses.

The first indirect target depends on incoming return-link state plus `0xFFE`. Later targets generally use represented address values plus that displacement. Their implementations are unavailable.

Some results occupy 64-bit containers and are divided numerically into low and high 32-bit portions. This does not establish an application-level 64-bit return convention.

### Outputs and completion

No explicit return value is represented. Outputs consist of dependency interactions, memory mutations, and a terminal transfer with four arguments:

| Position | Value |
|---|---|
| First | `0x72` |
| Second | A captured 32-bit word from terminal base plus `0x64` |
| Third | An earlier captured 32-bit word from paired base plus `0x14` |
| Fourth | A later captured 32-bit word from terminal base plus `0x74` |

The bases and capture boundaries are defined below. These arguments must not be replaced with last-minute rereads.

A no-value return follows the terminal interaction if it returns. Whether that target returns, terminates, or redirects control is **documented_only**.

### Preconditions and ownership

Successful progression requires each reached target to be executable and each reached memory access to be valid for its actual width. Accesses include low fixed addresses, scaled integers treated as addresses, and potentially unaligned locations.

No allocation, release, ownership transfer, or lifetime-management contract is established. Unknown dependencies must not be assumed to preserve globals, retain storage, or release memory.

## Behavior

### Arithmetic and observability

The represented body contains a long indirect-interaction phase followed by direct mutation. No local validation branch, retry, error return, or rollback is visible.

Interactions remain significant when their represented results are unused. Equal visible arguments do not establish equivalent targets or effects.

The representation distinguishes 8-bit bytes, 16-bit halfwords, 32-bit words, and 64-bit result containers. Transformations include left shifts, unsigned right shifts, masks, explicit zeros, duplicated arguments, and high-word extraction. Masks select low-bit ranges including 11 through 24 bits; some expressions extract bits 10–19.

**[INFERENCE]:** Under a 32-bit bit-vector model, each word-sized arithmetic intermediate retains its low 32 bits. A 25-bit left shift followed by an 11-bit left shift therefore produces zero, not a widened 36-bit result. This model is useful for conditional examples; it does not establish defined signed-overflow or signed-left-shift semantics for every represented expression.

Separate accesses must remain separate where aliasing or externally observable storage can distinguish them. Represented access order is not an independently established hardware memory-order guarantee.

### Captured-value vocabulary

These labels describe value provenance, not verified objects or private identifiers.

| Label | Meaning |
|---|---|
| **Shifted base** | Retained dependency-result word shifted left 25 bits |
| **Scaled base** | Later dependency-result word multiplied by `0x02000000` |
| **Publication word** | Another retained dependency-result word shifted left eight bits |
| **Small quantity** | Result word shifted left eight bits, masked to 15 bits, then unsigned-right-shifted six bits |
| **Doubled quantity** | Twice the small quantity |
| **Final low/high words** | Low and high 32-bit portions of the last dependency result before mutation |
| **Fixed-derived base** | Value obtained from the fixed-address `0x05` symbol and explicitly converted to unsigned 32 bits |
| **Captured field** | Word read from fixed-derived base plus `0x54`, before replacement |
| **Fourfold/eightfold field** | Four or eight times the captured field |

Under the bit-vector model, the small quantity ranges from `0` through `508` in multiples of four; the doubled quantity ranges from `0` through `1016` in multiples of eight.

**The fixed-derived base is not established as a zero-extended byte.** The declaration of the fixed-address `0x05` symbol is absent. Its original access width and signedness are **documented_only**. The explicit conversion establishes an unsigned 32-bit resulting value, not the source width, zero extension, or a range limited to `0`–`255`.

### Initial publications and clearing

The initial effects publish the shifted base as a word at final high word plus `0x14`, then its high-word provenance’s low byte at shifted base plus `0x09`. A word store publishes the scaled base at 256 times the final low word plus `0x54`. The shifted-base byte location is then overwritten with the shifted base’s low byte.

The clearing region clears scaled base plus `0x14`, reads an explicitly unsigned byte at scaled base plus `0x05`, clears doubled quantity plus `0x14`, and publishes that byte as a zero-extended word at scaled base plus `0x34`.

The unsigned-byte read through the scaled base is distinct from the later fixed-symbol read at address `0x05`.

A word is subsequently captured from scaled base plus `0x14`. Doubled quantity plus `0x14` is cleared again, and the captured word’s low byte is written twice at doubled quantity plus `0x15`. Aliasing prevents universally replacing the captured read with zero.

The fixed-address `0x05` symbol is then read and converted to unsigned 32 bits, establishing the fixed-derived base. **This capture precedes the byte write at fixed address `0x01`.** The `0x01` write contains the doubled quantity’s low byte. The small quantity’s low byte is then stored at fixed-derived base plus `0x15`.

Further publications place the publication word’s low byte at scaled base plus `0x11`, place the full publication word at the earlier cleared-field capture plus `0x54`, and clear the scaled-base offset-`0x11` byte.

The captured field is read from fixed-derived base plus `0x54`. That location is replaced with the earlier scaled-base offset-`0x14` capture. A word publication also places the small quantity at doubled quantity plus `0x34`.

### Derived updates and fixed publications

The next region follows offset-`0x74` words as address bases and updates offsets including `0x09`, `0x14`, `0x24`, and `0x54`.

One self-relative publication contains two distinct reads of doubled quantity plus `0x74`: one supplies a destination base and the other supplies the published value. Their mutual evaluation order is not established. They are not a single guaranteed capture.

Repeated offset-`0x24` word publications and offset-`0x09` byte publications are followed by:

- A word publication at doubled quantity plus `0x14` containing the retained scaled-base result shifted left 26 bits.
- A 32-bit store at fixed address `0x54` containing the captured field shifted left once.
- Two word publications of the captured field at doubled quantity plus `0x14`.
- Two byte publications of the fourfold field’s low byte at doubled quantity plus `0x15`.
- A byte publication of the doubled quantity’s low byte at fixed address `0x15`.

These fixed-address stores are direct effects, not attributed to unknown dependencies.

Further updates use fourfold and eightfold field values, doubled offset-`0x74` contents, and unsigned-byte-selected bases. They publish words at offsets `0x14`, `0x34`, `0x44`, and `0x54`, and bytes at offsets `0x09`, `0x0D`, and `0x15`.

A word captured at fourfold field plus `0x64` is followed as a base and updated at offsets `0x0D` and `0x14`. Another word, captured at fourfold field plus `0x04`, supplies a separate base. Its offset-`0x05` unsigned byte is read repeatedly with intervening stores. Those reads are not interchangeable.

### Mixed-width constants and repeated publication

A later offset-`0x74` read through a byte-selected base yields the **selected word**. Twice that word is the **paired base**.

The constant `0x3A` is published at three widths:

| Width | Destination |
|---|---|
| Byte | Four times the selected word |
| Halfword | Eight times the selected word |
| Word | Four times the selected word plus `0x20` |

The **adjusted field** is the fourfold field minus `0x96`. The **publication base** is four times the adjusted field.

Publication base plus `0x20` receives four ordered word writes: `0x20`, `0x2F`, a zero-extended unsigned byte freshly read from publication base plus `0x04`, and `0x20` again.

A **late publication base** is captured from a separate fixed word location before the word store of `0x20` at fixed address `0x60`. No ownership or object meaning is established for this pointer.

Before the repeated run, four times the selected word is published at eightfold field plus `0x54`.

The repeated pattern uses unchanged address expressions. Each complete repetition publishes, in order:

- The publication base’s low byte at late publication base plus `0x19`.
- The late publication base’s low byte at publication base plus `0x11`.
- A word at late publication base plus `0x34`.
- Four times the selected word at eight times the adjusted field plus `0x54`.

The first offset-`0x34` publication contains the eightfold field; subsequent ones contain eight times the adjusted field. The final partial repetition ends after its offset-`0x34` publication, before the next fixed-address read.

No advancing pointer or runtime progress condition is visible. Record traversal is not established. Exact repetition multiplicity remains **documented_only** as an unquantified static-trace detail, not an unknown runtime loop bound.

### Late fixed-address dependencies

A 32-bit **fixed word** is captured from address `0xA9`. This precedes the derived word capture at four times the selected word plus `0x74`; that capture precedes the byte write at fixed address `0x3A`.

The `0x3A` byte contains the low eight bits of eight times the adjusted field plus `0x30`. It is computed, unlike the earlier constant-`0x3A` publications.

After that write, the operation captures the **retained terminal word** from paired base plus `0x14`, then an **early field capture** from publication base plus `0x04`.

Derived byte publications place the publication base’s low byte at paired base plus `0x0D`, `0x6C` at publication base plus `0x08`, and the late publication base’s low byte at publication base plus `0x0D` and `0x09`. Paired base plus `0x54` receives the paired base itself.

Fixed address `0x67` receives the fixed word’s low byte. A word read from paired base plus `0x44` intervenes before a fresh read of `0x67` supplies the byte assigned to fixed address `0x21`.

That derived word also supplies a byte at publication base plus `0x10`, a word at late publication base plus `0x54`, and a signed-right-shift result narrowed to a byte at paired base plus `0x01`. The shift count is the publication base’s low eight bits. Negative-value and large-count behavior is unresolved.

The next publication boundary has this represented order:

1. Late publication base plus `0x54` is overwritten with the publication base.
2. A word is freshly captured from publication base plus `0x40`.
3. The early field capture is stored at late publication base plus `0x74`.
4. The newly captured offset-`0x40` word is stored at publication base plus `0x10`.

**The offset-`0x74` store precedes the offset-`0x10` word store.** The latter must not be moved before it; overlap or externally observable storage can distinguish that change.

Late publication base plus `0x44` then receives the paired base. Subsequent byte publications at publication base plus `0x09` freshly read fixed address `0x67`. Intervening derived stores and a word read from publication base plus `0x54` mean these reads need not reproduce the original fixed-word byte.

The fixed word is separately published at full word width to publication base plus `0x44`.

### Terminal captures and final effects

The **late destination base** is the word captured from publication base plus `0x54`.

The earlier derived offset-`0x74` capture’s low byte is published at late destination base plus `0x0D`. The retained terminal word is republished to paired base plus `0x54`. A fresh word is captured from publication base plus `0x14`.

After byte publications at paired base plus `0x11` and publication base plus `0x09`, publication base plus `0x44` receives the late destination base.

The **terminal base** is captured from paired base plus `0x04`. A halfword `0x65` is stored at twice the terminal base before the terminal second argument is captured from terminal base plus `0x64`.

After that capture, the publication-base offset-`0x14` capture’s low byte is stored at late destination base plus `0x11`, and word value `100` is stored at late destination base plus `0x54`.

The terminal fourth argument is then captured from terminal base plus `0x74`. Only afterward is a word freshly read from paired base plus `0x64`, establishing the **final destination value**. This is not the terminal second argument.

The final three stores publish:

- The terminal base’s low byte at paired base plus `0x09`.
- The captured second argument at final destination value plus `0x54`.
- The final destination value as a word at twice the captured second argument.

The unresolved transfer then receives `0x72`, the captured second argument, the retained terminal word, and the captured fourth argument.

Aliasing can distinguish the two offset-`0x64` reads. Subsequent stores do not refresh the terminal argument captures.

## Side effects

Observed offsets constitute an access footprint, not a verified structure layout.

| Width | Representative locations |
|---|---|
| 8 bits | Offsets `0x01`, `0x05`, `0x08`, `0x09`, `0x0D`, `0x10`, `0x11`, `0x15`, `0x19`; explicit fixed-byte stores |
| 16 bits | Scaled destinations receiving `0x3A` or `0x65` |
| 32 bits | Offsets `0x04`, `0x10`, `0x14`, `0x20`, `0x24`, `0x34`, `0x40`, `0x44`, `0x54`, `0x64`, `0x74` |

The fixed-symbol read at address `0x05` is excluded from the established byte-width footprint: its width and original signedness are unresolved.

Byte and halfword stores retain low eight and sixteen bits respectively. Explicit unsigned-byte reads used in word publications are zero-extended. That rule must not be generalized to the undeclared fixed-address `0x05` symbol.

Different base labels do not establish disjoint storage. Wider stores may overlap fixed bytes, pointer sources, or previous publications. Captured values remain distinct from fresh reads.

No error result, cleanup path, or transactional boundary is visible. Earlier effects are not represented as undone after a later failure. Trapping, blocking, exception behavior, thread safety, and reentrancy are **documented_only**.

## Dependencies

F-8b6ced299c8f is the only supplied mapped opaque function ID. No dependency IDs are supplied for the indirect targets; their identities remain **documented_only** rather than being assigned guessed function names.

Their represented interfaces contain zero through four explicit arguments. Results are consumed as words, wide containers, and secondary channels. The terminal target receives the four captured values specified above.

No dependency is assumed pure, deterministic, normally returning, globally preserving, or responsible for storage ownership.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Input transformations

Under 32-bit modular shifting, third input `0x12345678` supplies `0x23456780` at its four-bit-shifted use. Fourth input `1` supplies `0x02000000` at its 25-bit-shifted uses and two copies of `0x00000080` at its duplicated seven-bit-shifted use.

### Fixed-source conversion

If the undeclared fixed-address `0x05` symbol is an unsigned eight-bit value `0x80`, its conversion yields `0x00000080`.

If instead it is a signed eight-bit value representing `−128`, conversion to unsigned 32 bits yields `0xFFFFFF80`. The supplied representation does not choose between those declarations.

In either case, the read and conversion precede the byte publication at `0x01`. Consumers must not impose an unconditional `0`–`255` range on the fixed-derived base.

### Early fixed effects

If the small quantity is `0x14`, the fixed-byte publications at `0x01` and later `0x15` each contain `0x28`.

If the captured field is `0x12345678`, the fixed word publication at `0x54` contains `0x2468ACF0` under modular arithmetic.

### Ordered publications

For externally observable storage, the four publication-base offset-`0x20` word writes remain ordered as `0x20`, `0x2F`, the freshly loaded byte, and `0x20`. One final write is not trace-equivalent.

At the late publication boundary, the offset-`0x40` word is captured before the offset-`0x74` store, and that store precedes the offset-`0x10` word store.

If those two store destinations coincide and their captured values differ, the immediate value after the second store is the offset-`0x40` capture, not the early offset-`0x04` capture.

### Fresh fixed-byte reads

If `0x67` receives `0xA5` but permitted intervening effects change it to `0x3C` before its read for the `0x21` assignment, that assignment consumes `0x3C`. Later publications consume their own fresh reads.

### Terminal provenance

Suppose the retained terminal word is `0x11223344`, terminal base plus `0x64` yields `0x2000`, terminal base plus `0x74` yields `0x55667788`, and the subsequent paired-base offset-`0x64` read yields `0x3000`.

If execution reaches the transfer, its arguments are `0x72`, `0x2000`, `0x11223344`, and `0x55667788`. The final word publications place `0x2000` at `0x3054` and `0x3000` at `0x4000`, conditional on valid accesses and modular address arithmetic.

## Uncertainties

The following remain **documented_only**:

- Application intent, valid callers, and complete calling convention.
- Dependency identities, hidden inputs, side effects, ownership, and return guarantees.
- Whether wide containers represent genuine 64-bit results or paired machine channels.
- The fixed-address `0x05` symbol’s access width, original signedness, and resulting extension behavior.
- Address validity, allocation boundaries, lifetimes, concurrency, and device-memory behavior.
- Endianness, alignment, atomicity, and hardware memory ordering.
- Signed overflow, signed shifts, and shift counts of 32 or greater.
- Exact repeated-publication multiplicity, unquantified here.
- Meaning of the late fixed pointer and identity of the terminal target.
- Fidelity of the unusual represented interactions to executable behavior.

These uncertainties do not justify deleting explicit accesses, collapsing capture boundaries, or assigning guessed object or protocol semantics.

## Evidence

The supplied static representation is the sole behavioral evidence. Embedded comments, the prior document, and reviewer feedback are not independent proof.

The late publication boundary explicitly places the offset-`0x40` capture before the offset-`0x74` publication and the offset-`0x10` publication afterward. The fixed-address `0x05` expression explicitly establishes conversion to unsigned 32 bits, but supplies no declaration establishing an unsigned-byte source.

Static evidence supports the interface, width-sensitive transformations, ordered mutations, fresh-read distinctions, and terminal argument provenance described above. Application intent and runtime safety remain unestablished.

No tools were used, no implementation beyond the supplied representation was accessed, and no runtime tests were performed.
