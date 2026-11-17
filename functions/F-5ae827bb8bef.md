<!-- Trusted provenance: F-5ae827bb8bef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5ae827bb8bef

## Purpose

F-5ae827bb8bef performs an ordered sequence of unresolved indirect invocations, transforms values obtained from the execution context and dependency results, then mutates memory through computed and fixed addresses. It ends with an unresolved indirect invocation carrying four visible arguments.

The supplied static representation does not establish an application-level purpose. Repeated offsets, self-valued stores, and character-valued constants do not establish object initialization, collection management, string formatting, or a protocol.

This contract distinguishes:

- **Established static evidence:** represented accesses, value dependencies, widths, and ordering.
- **Inferred intent:** possible interpretations that are not acceptance requirements.
- **documented_only:** behavior not sufficiently resolved for an executable guarantee.

No runtime tests have been performed.

## Interface

### Invocation and inputs

The nominal interface declares no arguments and no result. Its effective inputs nevertheless include:

- An incoming 32-bit execution-context value used to construct the first arguments.
- Incoming control context used to derive the first indirect target.
- Primary and secondary dependency-result channels.
- Existing contents of all subsequently read computed addresses.
- A fixed-address input at `0x05`, whose value is widened and used directly as an address base.
- A fixed 32-bit word at `0xa9`.
- The byte at `0x67`, read after its assignment and again at later points.
- A word-sized global loaded late in the direct-memory phase and retained as an address base.

The global-base input is read at its represented late position, not at entry. Dependencies and preceding direct writes may affect its contents before that read.

A parameterless invocation alone therefore does not specify the function’s complete input state.

### Outputs and completion

Observable outputs include the indirect-call trace, dependency side effects, direct writes, intermediate overwritten values, and the final four-argument invocation.

No ordinary return value or explicit error result is established. A return is represented after the final invocation, but the material does not establish whether that dependency returns or whether this boundary is a conventional callable interface.

### Widths and arithmetic

| Value or access | Represented width | Required distinction |
|---|---:|---|
| Byte | 8 bits | Stores retain only the low eight bits |
| Halfword | 16 bits | Stores retain only the low sixteen bits |
| Word | 32 bits | Used for values and address-like quantities |
| Combined dependency result | 64 bits | Low and high 32-bit portions are consumed separately |

Explicit unsigned byte reads are zero-extended when used as words or bases. Numeric low-byte extraction is distinct from selecting the byte at the lowest memory address of a stored word; the latter depends on endianness.

Most address calculations, shifts, and multiplications are represented at 32-bit width. Signed-looking overflow, signed left shifts, address wraparound, and out-of-range variable shifts are not independently resolved. Modulo-\(2^{32}\) examples below are conditional models, not universal execution claims.

Endianness, alignment, and unaligned-access behavior remain documented_only.

### Preconditions

Completion requires callable indirect targets, returning dependencies, valid result-channel conventions, and readable or writable mappings for every reached access. These include low fixed addresses and addresses derived from byte values.

No visible validation enforces these requirements. No null check, bounds check, or recovery branch protects the direct-memory phase.

## Behavior

### Ordered dependency phase

Before the first explicit direct data-memory store, the function follows a long straight-line sequence of indirect invocations. Dependencies may themselves access memory; their effects are unknown.

F-DEP-001 receives two equal visible arguments: the incoming 32-bit value shifted left seven positions. Subsequent F-DEP-002 interactions consume fresh results, retained older results, secondary channels, and selected halves of combined results.

Trace compatibility requires preserving these distinctions:

- Calls without visible arguments still occur.
- Calls with discarded primary results still occur.
- Equal arguments occupy separate visible argument positions.
- Older retained results are not replaced by newer results.
- Secondary-channel consumption is not eliminated merely because no primary-result assignment accompanies it.
- Similar target-address construction does not establish identical dependency behavior.

Represented transformations include left shifts of one through thirty-one positions, unsigned right shifts, masking, duplicated arguments, and explicit zeros. Examples include retaining the low 24 or 22 bits before scaling, and retaining the low twenty bits before shifting right ten positions.

Intermediate width matters. Under 32-bit wrapping arithmetic, successive left shifts by twenty-five and seven discard all original bits. A 64-bit result shifted right thirty-six selects its high word shifted right four, not a portion of its low word. Explicitly zeroed values remain zero through their subsequent represented shifts, but calls carrying those zeros remain observable interactions.

The application meaning and complete contracts of these dependencies are documented_only.

### Initial direct-memory cluster

The following descriptive terms identify retained quantities without asserting object layouts:

- **High-scale base:** an earlier primary result multiplied by \(2^{25}\).
- **Shifted base:** a different retained result shifted left twenty-five positions.
- **Masked quantity:** an eight-position-shifted result masked with `0x7fff`, then unsigned-shifted right six.
- **Small base:** twice the masked quantity.
- **Latest result pair:** the low and high words of the last combined result before direct mutation.
- **Saved shifted value:** an earlier primary result shifted left eight positions.

Under the displayed 32-bit bit-vector interpretation, the masked quantity ranges from 0 through 508 in multiples of four; the small base ranges from 0 through 1016 in multiples of eight. Neither range establishes allocated storage.

The cluster begins with a word write at latest-high-word offset `0x14`, carrying the shifted base. This precedes a byte write at shifted-base offset `0x09`. A destination formed from 256 times the latest low word, plus `0x54`, receives the high-scale base. The shifted-base byte at `0x09` is then overwritten with the low byte of the shifted base itself.

High-scale-base offset `0x14` is cleared before an unsigned byte is read at its offset `0x05`. Small-base offset `0x14` is cleared before the widened byte is stored at high-scale-base offset `0x34`. The function then reads the word at high-scale-base offset `0x14`, clears small-base offset `0x14` again, and performs two byte writes at small-base offset `0x15`, both using the low byte of that loaded word.

These accesses cannot be reduced to final values: overlapping addresses can change later reads, and repeated writes may be individually observable.

### Fixed-address input and early linking effects

Immediately after those two small-base byte writes, the function reads the fixed-address input at `0x05` and retains its widened value as an address base. It then writes the low byte of the small base to fixed address `0x01`.

Only after that fixed write does it write the low byte of the masked quantity at offset `0x15` from the newly loaded fixed-input base. This is a computed destination, not fixed address `0x15`.

The saved shifted value is written as a byte at high-scale-base offset `0x11` and as a word at offset `0x54` through the previously loaded high-scale `0x14` word. The high-scale `0x11` byte is then cleared. Next, the word at fixed-input-base offset `0x54` is saved before that location is overwritten.

That saved old word becomes the **pivot** for later address calculations. The fixed-address `0x05` read is therefore a substantive input to the traversal, not an incidental read that may be omitted or delayed.

Further early mutations include:

- A word from high-scale-base offset `0x74` becomes another write base.
- A nested update uses small-base offset `0x74` separately for its destination base and stored word. A single cached read is not established; intra-expression evaluation order is documented_only.
- Pivot offset `0x24` receives the pivot twice.
- Another base’s `0x09` byte repeatedly receives the low byte of the pivot.
- Small-base offset `0x14` receives a shifted earlier value, then the pivot twice.
- Fixed word address `0x54` receives the pivot shifted left one position.

After the two pivot assignments to small-base offset `0x14`, small-base offset `0x15` receives two further byte writes carrying the low byte of four times the pivot. The function then writes the low byte of the small base to **fixed address `0x15`**. This fixed write precedes the following fourfold-pivot word updates; it is distinct from all computed `0x15` accesses.

### Computed layouts and alias-sensitive traversal

The representation uses these offsets across multiple bases:

| Offset | Represented use |
|---|---|
| `0x01` | Byte store derived from a variable right shift |
| `0x04` | Byte and word reads |
| `0x05` | Unsigned byte reads |
| `0x08`, `0x09`, `0x0d`, `0x11`, `0x15`, `0x19` | Byte stores |
| `0x10` | Byte and word stores |
| `0x14` | Word reads and writes |
| `0x20`, `0x24`, `0x34`, `0x40`, `0x44`, `0x54`, `0x64`, `0x74` | Word accesses |

This is an access map, not a recovered structure definition. Different bases may denote unrelated storage, overlapping storage, or device registers.

The next traversal scales the pivot by two, four, and eight; writes self-valued words; copies truncated addresses into byte locations; and updates word links at `0x44` and `0x54`. Reads at offset `0x64`, equivalently decimal 100, supply additional bases.

More than once, a byte read at a computed offset `0x05` is zero-extended and used directly as the base of a word read at offset `0x74`. No table-index interpretation is established. Repeated byte reads remain fresh reads because intervening writes may alias them.

A later such word supplies a **leaf value**. Twice that value becomes the retained **double-leaf base** used in the final phase; four times it is used in the constant-write and publication phases.

### Constants, retained global load, and publication

The constant `0x3a` is stored through three distinct accesses: a byte at four times the leaf value, a halfword at eight times it, and a word at four-times-leaf offset `0x20`. These are not established as adjacent characters.

The **adjusted scalar** is four times the pivot minus `0x96`; the **working base** is four times that adjusted scalar. Working-base offset `0x20` receives four word writes, in order: `0x20`, `0x2f`, a widened unsigned byte freshly read from working-base offset `0x04`, and `0x20` again. The byte read follows the first two writes.

After the final `0x20` write at that computed destination, the function loads the word-sized global base. **That load precedes the fixed word write of `0x20` at address `0x60`.** The loaded base is retained across the fixed write and the entire repeated publication cluster. Moving the load after the write is incompatible without a non-aliasing guarantee.

Each full publication group comprises, in order:

- A scaled destination’s `0x54` word receives four times the leaf value.
- Retained-global-base offset `0x19` receives the working base’s low byte.
- Working-base offset `0x11` receives the retained global base’s low byte.
- Retained-global-base offset `0x34` receives the scale source multiplied by eight.

The first scale source is the pivot; subsequent groups use the adjusted scalar. The addresses do not advance as an established array traversal. The last displayed group also includes its leading destination write and ends with its global-base `0x34` write.

The exact numerical group count is not certified here and remains documented_only. Replacing the repeated cluster with one final-state update is not justified.

### Final reads, mutations, and handoff

The opening of the final phase has an established interleaving:

1. Read the fixed 32-bit word at `0xa9`.
2. Read the word at four-times-leaf offset `0x74`.
3. Write a byte at fixed address `0x3a`, carrying the low eight bits of eight times the adjusted scalar plus `0x30`.
4. Read the word at double-leaf-base offset `0x14`.
5. Read the word at working-base offset `0x04`.

The last two reads occur **after** the `0x3a` write and may observe its effect through aliasing. They are not entry snapshots or values captured before all final mutations.

The function then performs byte writes involving the working base and retained global base, stores `0x6c` at working-base offset `0x08`, and writes the double-leaf base to its own offset `0x54`.

The saved word from `0xa9` and later reads at `0x67` have different lifetimes:

- Fixed byte address `0x67` receives the saved word’s numeric low byte.
- The word at double-leaf-base offset `0x44` is then read.
- Fixed address `0x21` receives a read of the byte at `0x67`.
- Subsequent mutations intervene before two further reads of `0x67`, each supplying a byte write at working-base offset `0x09`.

Between assignment to `0x67` and its first read, the represented intervening access is the double-leaf `0x44` word read. Between later reads, writes may alias `0x67`. The two later computed stores must therefore use the then-current byte, not an assumed saved low byte from `0xa9`.

Other ordered final effects include:

- A byte update at working-base offset `0x10`, followed later by a full word copied from its offset `0x40` to `0x10`.
- Two writes to retained-global-base offset `0x54`, first carrying the loaded double-leaf `0x44` word and then the working base.
- A byte at double-leaf-base offset `0x01` derived from a signed-looking right shift, with a count taken from the working base’s low eight bits.
- Publication of the saved working-base `0x04` word at retained-global-base offset `0x74`.
- A working-base `0x44` word write carrying the saved full word from `0xa9`, followed later by an overwrite with a freshly loaded address-like value.
- Restoration at double-leaf-base offset `0x54` using the word saved from its `0x14` field.
- Further fresh reads supplying byte writes and pointer-like traversal.

Near handoff, a word loaded at double-leaf-base offset `0x04` supplies a base whose doubled address receives halfword `0x65`. A word from that base’s offset `0x64` is retained. Another loaded base receives decimal 100 at offset `0x54`. A later `0x74` read supplies the fourth final argument. A fresh double-leaf `0x64` read supplies a final linking destination, and twice the retained `0x64` word receives another word store.

F-DEP-003 receives, in order, `0x72`, the retained `0x64` word, the earlier saved double-leaf `0x14` word, and the later `0x74` word.

## Side effects

Direct fixed-address effects include the input read at `0x05`, byte writes at `0x01`, `0x15`, `0x3a`, `0x67`, and `0x21`, word writes at `0x54` and `0x60`, and the word read at `0xa9`. Address `0x67` is also read repeatedly after assignment.

Mixed-width accesses overlap: offset `0x15` lies within the word starting at `0x14`; the word at `0x10` includes bytes at `0x10` and `0x11`. Different bases may introduce additional overlap.

Purity, atomicity, idempotence, reentrancy, locking, and rollback are not established. No ownership transfer, allocation, or release is identified. Dependencies may allocate, release, retain, or invalidate memory; their unknown contracts provide no lifetime guarantee.

## Dependencies

| Opaque ID | Interaction | Status |
|---|---|---|
| F-DEP-001 | Initial context-derived indirect invocation | documented_only |
| F-DEP-002 | Ordered family of intervening indirect targets | documented_only |
| F-DEP-003 | Final four-argument indirect invocation | documented_only |

These are document-local IDs, not verified dependency identities. F-DEP-002 does not imply one shared implementation. No dependency is assumed to preserve globals, pointer validity, secondary channels, or normal control flow.

## Acceptance examples

These are conditional static criteria, not executed tests.

- **Initial duplication:** With incoming value 1 and a callable initial target, its two visible arguments are both 128.
- **Truncation:** A byte store from `0x123456ab` writes `0xab`. A word store cannot substitute for it.
- **Extraction:** Retaining twenty low bits of `0xffffffff`, then unsigned-shifting right ten, yields 1023. Shifting that result left six yields 65472.
- **Intermediate width:** Under 32-bit wrapping arithmetic, left shifts by twenty-five and then seven yield zero.
- **Fixed-input ordering:** If the `0x05` input yields `0x80`, the subsequent computed byte destination is `0x95`. The fixed `0x01` write occurs between the input read and that computed write.
- **Fixed versus relative:** The later fixed `0x15` write remains required even when small-base offset `0x15` denotes another address.
- **Retained global:** If the `0x60` word write overlaps the global-base input, publication still uses the value loaded before that write.
- **Interleaved final read:** If double-leaf-base offset `0x14` includes address `0x3a`, its saved word is read after the fixed byte mutation. A pre-write snapshot is not equivalent.
- **Fresh byte global:** If an intervening computed write changes `0x67`, the next read supplying working-base offset `0x09` uses the changed byte, not the original saved word’s low byte.
- **Invalid destination:** An inaccessible computed address has no established recovery result; successful continuation is not guaranteed.

## Uncertainties

The following remain documented_only:

- Application purpose, object layouts, and ownership.
- Complete calling convention and dependency contracts.
- Independently declared storage width of the fixed `0x05` symbol; its widened value and address-base use are established.
- Exact target arithmetic, signed overflow, variable-shift semantics, endianness, and alignment.
- Aliasing, device-memory behavior, and intra-expression ordering where not established.
- Independently certified publication-group count.
- Final-transfer classification, normal-return reachability, and reliability of the displayed boundary and types.

Related offsets suggest coordinated state mutation; this is inferred intent, not proof of a collection invariant or formatting operation.

## Evidence

This contract derives from the supplied static representation, not from the prior draft or reviewer assertions. The represented access dependencies support the fixed-input ordering, global-load-before-`0x60` ordering, final read/write interleaving, and distinction between the saved `0xa9` word and fresh `0x67` reads.

No dependency inspection, machine-level validation, implementation access, or runtime testing was performed. Acceptance examples describe conditional consequences, not observed execution.
