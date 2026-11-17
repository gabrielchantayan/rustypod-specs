<!-- Trusted provenance: F-1097b29dbcfe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1097b29dbcfe

## Purpose

F-1097b29dbcfe performs an ordered sequence of indirect invocations and fixed-width transformations, followed by extensive computed-address and global-memory effects and a final indirect invocation.

No domain-level purpose is established. The evidence does not justify identifying the operation as initialization, formatting, serialization, allocation, or collection maintenance. Character-valued constants and self-valued stores do not establish those interpretations.

This contract distinguishes:

- **Established static evidence:** explicit argument flow, transformations, accesses, and ordering in the supplied representation.
- **[INFERENCE]:** conclusions requiring an identified execution or memory model.
- **documented_only:** behavior not resolved by the supplied evidence.

The contract is partial where dependency identities, calling conventions, and execution semantics remain unresolved. It does not replace those gaps with conventional API assumptions.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | Forwarded unchanged as the first explicit operand of the initial indirect invocation. |
| Second argument | 32 bits | No explicit reference in the body. |
| Third argument | 32 bits | No explicit reference in the body. |
| Fourth argument | Signed 32-bit representation | Shifted left seven positions for the initial invocation’s second explicit operand. |

No input has an established pointer, handle, length, ownership, or domain interpretation. No range check precedes the fourth argument’s transformation.

The lack of explicit uses of the second and third arguments does not prove their irrelevance to an unresolved calling convention or dependency.

### Implicit inputs and preconditions

Execution also consumes incoming linkage state, invocation results, additional result channels, global storage, and words and bytes loaded through computed addresses.

Successful completion requires every reached indirect target and memory access to be usable in the execution environment. The body does not validate those requirements. Four-argument compatibility alone is not an established sufficient calling precondition.

### Outputs

There is no declared result. Observable outputs comprise invocation traces, memory accesses and mutations, and the final indirect invocation.

The final invocation receives four explicit operands:

1. `0x72`.
2. A late-loaded 32-bit word obtained through an offset-`0x64` read.
3. A word retained from an earlier offset-`0x14` read in the final memory phase.
4. A word loaded through offset `0x74` of another late-loaded base.

The representation places a return after this invocation. Whether the target returns normally, never returns, or performs an additional control transfer is **documented_only**.

## Behavior

### Execution and arithmetic model

The body has a long straight-line invocation phase followed by a direct-memory phase. No explicit conditional branch, retry, or data-dependent loop is established. A dependency may nevertheless interrupt execution or fail to return.

Values are predominantly 32-bit words; some invocation results are represented as 64 bits. Byte, halfword, and word accesses have widths of eight, sixteen, and thirty-two bits respectively. Widened byte loads are zero-extended.

For a represented 64-bit result, extraction starting at bit 32 selects its high word. Extraction starting at bits 34, 36, or 44 instead selects that high word logically shifted right by two, four, or twelve positions.

**[INFERENCE]** Under fixed-width 32-bit machine arithmetic, scaling and left shifts retain the low thirty-two bits. Intermediate truncation matters: a word shifted left twenty-five positions and subsequently left seven positions becomes zero. It must not be evaluated using an indefinitely wide intermediate.

Signed overflow, signed left-shift interpretation, and address-wrap behavior are not independently resolved at the machine level. Statements using modulo-width arithmetic below are conditional on that model.

### Indirect invocation phase

The initial target is derived from incoming linkage state. Its first explicit operand is the first argument; its second is the fourth argument shifted left seven positions.

Subsequent targets are derived from embedded location values. Their operands repeatedly include:

- Earlier results shifted left, commonly by eight or fifteen positions.
- Duplicated transformed operands.
- Low and high portions of represented 64-bit results, transformed independently.
- Masked values subsequently shifted or logically shifted right.
- Retained results carried across intervening calls.
- Additional result-channel values with unresolved provenance.
- Explicit zeros and transformations of values already set to zero.

Masks select materially different widths, including twenty-four, twenty-two, twenty-one, twenty, nineteen, eighteen, seventeen, sixteen, fifteen, fourteen, thirteen, twelve, eleven, and one bit. They do not establish a single uniform payload format.

Invocation order is observable. A call without explicit operands is not established to be pure; a call with an unused primary result is not established to be dispensable. Neither equal explicit operands nor repeated target calculations establish equal effects or results.

### Roles used to describe direct-memory effects

The following descriptive roles identify retained values, not object types:

| Role | Derivation |
|---|---|
| Publication seed | A preceding invocation result shifted left twenty-five positions. |
| Shifted word | Another retained invocation result shifted left eight positions. |
| Unscaled word | A retained invocation result used for later scaling. |
| Initial cleared base | The unscaled word multiplied by `0x02000000`. |
| Narrow index | A later result shifted left eight positions, masked with `0x7fff`, then logically shifted right six positions. |
| Doubled-index base | Twice the narrow index. |
| Byte-derived base | The zero-extended byte read from absolute `0x05`. |
| Anchor | A word read at offset `0x54` of the byte-derived base, before that location is overwritten. |
| Quadrupled anchor | Four times the retained anchor. |
| Adjusted anchor | The quadrupled anchor minus `0x96`. |
| Publication base | Four times the adjusted anchor. |
| Global-derived base | A word read from a separate global slot shortly before the repeated publication sequence. |

These roles may alias one another or absolute storage.

### Entry into direct-memory effects

The final represented 64-bit result of the invocation phase supplies a low word and a high word. The first direct stores publish the publication seed at high-word base plus `0x14`, write the high word’s low byte at publication-seed base plus nine, and write the initial cleared base at low-word-times-256 base plus `0x54`. The byte at publication-seed base plus nine is then overwritten with the publication seed’s own low byte.

The initial cleared base’s offset-`0x14` word is cleared before its offset-five byte is read. A zero word is then stored at doubled-index base plus `0x14`, followed by a zero-extended word publication of the loaded byte at initial cleared base plus `0x34`.

The offset-`0x14` word of the initial cleared base is then read back. Call this the **cleared-word read**.

**[INFERENCE]** With ordinary stable memory, conventional byte addressing, and no external mutation, this read yields zero. The intervening potentially overlapping word store also writes zero. The widened-byte store at offset `0x34` is disjoint from the same base’s offset-`0x14` word. Aliasing alone does not establish a nonzero result here.

The read remains an explicit access. Removing it requires assumptions about ordinary memory, mapping, and external observability that the evidence does not supply.

### Early global accesses and surrounding mutations

After another zero word is written at doubled-index base plus `0x14`, two byte stores at that base plus `0x15` each use the cleared-word read’s low byte.

Next, the byte at absolute `0x05` is read and retained as the byte-derived base. Only after that read is absolute `0x01` written with the low eight bits of the doubled-index base.

The following computed-memory effects depend on those retained values:

- Byte-derived base plus `0x15` receives the narrow index’s low byte.
- Initial cleared base plus `0x11` receives the shifted word’s low byte.
- Cleared-word-read base plus `0x54` receives the full shifted word.
- Initial cleared base plus `0x11` is overwritten with zero.

Under the ordinary-memory conditions establishing a zero cleared-word read, the third effect is a word store at absolute `0x54`. Its value is the shifted word, not zero.

The anchor is then read at byte-derived base plus `0x54`; that location is subsequently overwritten with the cleared-word read. Doubled-index base plus `0x34` receives the narrow index.

Further reads through offset `0x74` select destinations for word writes. One such expression contains two reads of the same offset-`0x74` location, using it both to select a destination and to supply stored data. No ordering between those two operand reads beyond the surrounding access boundaries is established here.

Self-valued stores at anchor plus `0x24` and low-byte publications through a selected base plus nine are repeated.

Later, doubled-index base plus `0x14` receives the unscaled word shifted left twenty-six positions. Absolute `0x54` then receives the full 32-bit anchor shifted left one position. This global write occurs before two further word stores of the anchor at doubled-index base plus `0x14`.

Two byte stores at doubled-index base plus `0x15` subsequently receive the low byte of four times the anchor. Absolute `0x15` then receives the low byte of the doubled-index base. This absolute byte publication precedes the next word store at quadrupled-anchor base plus `0x54` and the publication of the quadrupled anchor at doubled-index base plus `0x34`.

The two absolute-`0x54` word writes are therefore distinct: the earlier ordinary-memory case stores the shifted word; the later explicit global store publishes twice the retained anchor.

### Computed relationships and constant publications

The intervening mutation phase follows word and byte loads through offsets four, five, `0x14`, `0x44`, `0x54`, `0x64`, and `0x74`. Loaded words become bases; loaded bytes can also become bases for subsequent word reads.

It writes scaled anchor values, self-valued words, and narrowed bytes at offsets including nine, `0x0d`, `0x14`, `0x15`, `0x34`, `0x44`, and `0x54`. Repeated reads, including byte reads at offset five, occur after potentially overlapping writes and must not be replaced by entry-time snapshots.

A later word obtained through a byte-derived offset-`0x74` read determines three constant destinations. The operation writes `0x3a` as:

- A byte at four times that word.
- A halfword at eight times that word.
- A word at four times that word plus `0x20`.

Twice this loaded word is retained as a **late working base**. Four times it is retained as a **scaled publication value**.

At publication base plus `0x20`, the ordered word publications are `0x20`, `0x2f`, the zero-extended byte read from publication base plus four, and `0x20` again.

Only after these accesses is the separate global slot read to obtain the global-derived base. Absolute `0x60` is then written with the 32-bit word `0x20`, before the repeated publication sequence starts. This slot read must not be moved past the absolute store: computed aliasing and global overlap are unresolved.

Each publication group has four ordered writes:

1. A word at a scaled destination plus `0x54`, carrying the scaled publication value.
2. A byte at global-derived base plus `0x19`, carrying the publication base’s low byte.
3. A byte at publication base plus `0x11`, carrying the global-derived base’s low byte.
4. A word at global-derived base plus `0x34`, carrying the scaled destination base.

The first group uses eight times the anchor as its scaled destination base. Subsequent groups use eight times the adjusted anchor. Their byte operands and stored publication value remain retained values; the global base slot is not visibly reread for each group.

The representation contains a finite repeated sequence, not a demonstrated runtime loop. Its exact group count is **documented_only** in this contract; no count-dependent replacement criterion is asserted.

### Late global reads and alias-sensitive byte publication

After the repeated groups, the word at absolute `0xa9` is read and retained. This read precedes both the next offset-`0x74` read from the scaled-publication-value base and the byte write at absolute `0x3a`.

The byte written at `0x3a` is the low eight bits of eight times the adjusted anchor plus `0x30`. The represented calculation first narrows the adjusted anchor to a byte-sized quantity, scales it by eight, adds `0x30`, and stores a byte. Under modulo-256 narrowing, retaining only the final low byte gives the same result. This is not a constant `0x3a` publication.

The operation then retains words from late working base plus `0x14` and publication base plus four. It writes:

- The publication base’s low byte at late working base plus `0x0d`.
- Byte `0x6c` at publication base plus eight.
- The global-derived base’s low byte at publication base plus `0x0d` and plus nine.
- The late working base itself at that base plus `0x54`.

Only then is absolute `0x67` written with the low byte of the retained `0xa9` word.

A word read at late working base plus `0x44` occurs next. After that read, the then-current byte at `0x67` is read and copied to absolute `0x21`. This is a fresh byte read, not a direct copy of the retained `0xa9` word.

The newly read word supplies a byte at publication base plus `0x10`, a word at global-derived base plus `0x54`, and a dynamically shifted byte at late working base plus one. The shift count is the low byte of the publication base. The shifted source has a signed 32-bit representation; exact signed-shift behavior and counts of at least 32 are **documented_only**.

Global-derived base plus `0x54` is then overwritten with the publication base. A word read from publication base plus `0x40` is later written at publication base plus `0x10`, replacing the earlier byte-level publication in that region. Other intervening stores publish the previously retained offset-four word and the late working base through global-derived base plus `0x74` and plus `0x44`.

The byte at `0x67` is subsequently read again for a store at publication base plus nine. Between this store and a second such store, the operation writes the publication base’s low byte at late working base plus nine, writes the retained `0xa9` word at publication base plus `0x44`, and reads a word from publication base plus `0x54`.

The second store at publication base plus nine reads `0x67` again. Neither later read is guaranteed to equal the originally published byte. Intervening computed stores can alias absolute `0x67`; that possibility applies even without concurrent activity.

### Final mutations and terminal operands

The offset-`0x74` word retained immediately after the `0xa9` read supplies narrowed bytes written through the newly selected base plus `0x0d` and late working base plus `0x11`.

The word retained from late working base plus `0x14` is restored at that base plus `0x54` and retained for the final invocation’s third operand.

Further effects include a word read at publication base plus `0x14`, an overwrite of publication base plus nine with a selected base’s low byte, and publication of that selected base at publication base plus `0x44`.

A word loaded from late working base plus four selects another base. Halfword `0x65` is written at twice that loaded value. The operation reads its offset-`0x64` word, writes a narrowed byte and word `100` through another selected base, and reads its offset-`0x74` word for the final invocation’s fourth operand.

Finally, an offset-`0x64` read from the late working base selects a destination. The late working base’s offset-nine byte is updated, the retained offset-`0x64` word is stored at destination plus `0x54`, and that destination value is stored at twice the retained word.

The terminal invocation then receives `0x72`, the retained offset-`0x64` word, the earlier offset-`0x14` word, and the terminal offset-`0x74` word, in that order.

## Side effects

Access offsets do not establish validated record layouts, object bounds, or allocation sizes. Address alignment, byte order, memory mapping, and distinctness of computed regions remain unresolved.

Byte and word accesses overlap structurally. Offset `0x15` lies within a word beginning at `0x14`; offset `0x11` lies within a word beginning at `0x10`. Logical field separation is therefore insufficient to preserve behavior.

Low absolute addresses are meaningful parts of the represented access trace, not automatically invalid accesses to discard. A compatible environment is required if they are literal executable accesses.

No allocation, release, reference counting, or ownership transfer is established. Unknown dependencies may have such effects, but none may be assumed.

No pointer checks, status protocol, cleanup, rollback, synchronization, or atomicity guarantee is visible. If execution stops partway through, earlier effects are not visibly compensated.

## Dependencies

F-1097b29dbcfe is the only supplied opaque function ID. No opaque dependency IDs are supplied for the indirect targets; this contract assigns none.

Dependency boundaries comprise the linkage-derived initial target, the ordered internal indirect sites, their consumed result channels, and the terminal target.

For each unresolved dependency, identity, calling convention, memory effects, global preservation, ownership effects, termination, and error behavior are **documented_only**. No target is established as a getter, allocator, formatter, destructor, or no-op.

## Acceptance examples

These are conditional static criteria, not executed tests.

- **Initial shift:** Under 32-bit truncating shifts, a fourth argument of one produces initial second operand `0x80`; `0x02000000` produces zero. Saturation is not equivalent.
- **Narrowing:** A retained word `0x123456ab` supplies byte `0xab`. A byte load of `0x80` widened to a word supplies `0x00000080`.
- **Cleared-word read:** With ordinary stable memory and no external mutation, the initial offset-`0x14` reread yields zero. The specified intervening stores do not support a nonzero aliasing example.
- **Early globals:** The `0x05` byte read precedes publication to `0x01`, even if some computed destination aliases either location.
- **Distinct global word values:** Under the ordinary zero-read conditions, the early absolute-`0x54` store carries the shifted word; the later explicit store carries twice the anchor.
- **Calculated byte:** If the adjusted anchor’s low byte is two, the calculated absolute-`0x3a` byte is `0x40`, under modulo-256 arithmetic.
- **Publication setup:** The global-base slot read precedes the word `0x20` store at `0x60`, which precedes the first publication group.
- **Fresh global rereads:** If `0x67` initially receives `0xab` but an intervening aliasing word store changes that byte to `0xcd`, a later specified read publishes `0xcd`, not a cached `0xab`.
- **Interruption:** If an indirect dependency does not return, subsequent invocations and direct writes are not reached. No cleanup or success result follows by implication.

## Uncertainties

The following remain **documented_only**:

- Domain purpose and executable fidelity of the supplied representation.
- Exact dependency identities, calling conventions, and additional result-channel provenance.
- Complete invocation-site enumeration and exact publication-group count in this contract.
- Valid address-space configuration, alignment, byte order, and external-memory behavior.
- Machine-level signed overflow, address wrapping, and dynamic-shift semantics.
- Ownership, failure propagation, concurrency, and reentrancy guarantees.
- Whether the terminal target returns.

These gaps prevent treating this document as a complete executable replacement specification.

## Evidence

The evidence is limited to static inspection of the supplied representation. Prior prose, reviewer feedback, comments, and warnings are not independent behavioral authority.

Established claims rest on explicit operand flow, access widths, masks, result extraction, retained values, constants, and represented access order. Ordinary-memory and fixed-width conclusions are identified as conditional inferences.

No tools, implementation access, runtime execution, smoke runs, or tests have been used. Acceptance examples are proposed observations for future verification, not reported results.
