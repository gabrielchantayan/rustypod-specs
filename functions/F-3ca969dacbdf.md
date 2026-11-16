<!-- Trusted provenance: F-3ca969dacbdf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3ca969dacbdf

## Purpose

F-3ca969dacbdf performs an ordered sequence of indirect calls, fixed-width numerical transformations, and mixed-width memory mutations, followed by an indirect transfer with four displayed arguments.

Its application-level purpose is **documented_only**. The supplied representation does not establish a collection operation, initializer, formatter, or protocol implementation. Address-like values, repeated stores, and character-valued constants do not establish those meanings.

This contract distinguishes:

- **Established static evidence:** displayed access widths, value dependencies, and ordering.
- **[INFERENCE]:** deductions conditional on an explicitly stated arithmetic or memory model.
- **documented_only:** behavior not resolved by the available evidence.

The operation is not established as a portable, independently callable API. Incoming machine state, unresolved dependencies, and access to low numerical addresses are material parts of its interface.

## Interface

### Inputs and outputs

There are no displayed explicit parameters, return value, output buffer, capacity, or success status.

Implicit inputs include:

- An incoming return-link value used to construct the first dependency target.
- Incoming general-purpose machine state used without a visible assignment.
- Primary and secondary machine-state results of indirect calls.
- Existing storage at absolute and computed addresses.
- A global associated with absolute address `0x05`, whose value is converted to a 32-bit unsigned quantity.
- A separate global word later retained as an address base.
- An unresolved final transfer target.

Observable outputs comprise direct memory mutations, dependency effects, and the final transfer. The entry calling convention and required entry-state domain are **documented_only**.

### Widths and numerical interpretation

| Quantity | Established representation |
|---|---|
| Explicit byte access | 8 bits |
| Explicit halfword store | 16 bits |
| Explicit word access | 32 bits |
| Selected combined dependency results | 64 bits |
| Converted value from the global at `0x05` | 32-bit unsigned result |
| Source access for that global | Width and signedness unresolved |
| Explicit return value | None |

Explicit byte stores retain the low eight bits; halfword stores retain the low sixteen bits. Explicit unsigned-byte loads used numerically are zero-extended.

**The read of the global at `0x05` is not established as an unsigned-byte load.** Its declaration is absent. Conversion to a 32-bit unsigned quantity establishes the destination representation, not the source access width, source signedness, or a zero-extension operation. The retained result must not be restricted to `0` through `255` on this evidence.

Selected combined results are separated numerically into low and high 32-bit words. This does not establish memory byte order or a conventional scalar-return ABI.

**[INFERENCE — 32-bit arithmetic model]:** ordinary arithmetic, address scaling, address addition, and fixed left shifts operate on 32-bit bit patterns, truncating intermediate results modulo \(2^{32}\). Arithmetic deductions below use this model; portable signed-overflow behavior is not established.

### Preconditions, ownership, and errors

Completion requires every preceding dependency to return and every reached memory access to be supported. Low-address and potentially unaligned accesses are included, notably a word read at `0xA9`.

No displayed validation checks target identity, address validity, alignment, allocation boundaries, permissions, null values, lengths, or capacities.

No allocation, release, retain, or ownership-transfer semantics are established. Dependencies must not be assumed to preserve globals or other memory, or to acquire or release ownership.

There is no displayed error status, recovery branch, exception translation, or rollback mechanism. Unsupported accesses and nonreturning dependencies have environment-dependent consequences, classified as **documented_only**.

## Behavior

### Execution structure and dependency results

The displayed path contains an indirect-call region, a direct-memory region, and the final transfer. No explicit conditional branch or runtime loop controls these regions. Dependencies may themselves branch, mutate state, fail, or not return.

The first dependency target is the incoming return-link value plus `0xFFE`. Subsequent targets are predominantly distinct embedded position or code-reference values plus that displacement. These are computed call addresses, not displayed pointer loads through those addresses.

Calls remain ordered events even when no arguments or retained primary result are displayed. A call with no displayed arguments may still have implicit inputs and effects.

Arguments include shifted primary results, secondary machine-state results, retained earlier results, masked fields, and explicit zeros. Unsigned 64-bit extractions include shifts of 34, 36, and 44 bits. These must retain their 64-bit interpretation: extracting after a shift of 36 corresponds numerically to the high 32-bit word logically shifted right by four.

Under the 32-bit model, successive left shifts by 28 produce zero. Explicitly established zero values also remain zero through the displayed fixed shifts. Passing zero does not establish a zero dependency result.

Complete dependency signatures, call-site identities, and application meanings of the transformed values remain **documented_only**.

### Opening memory region

The following labels describe retained numerical values, not validated object types:

| Label | Meaning |
|---|---|
| Pair-low, Pair-high | Low and high words of the last combined result before direct mutation |
| Shifted base | A retained dependency result shifted left by 25 |
| Scaled base | A separate retained result multiplied by \(2^{25}\) |
| Saved word | Another retained result shifted left by eight |
| Compact index | A retained result shifted left by eight, masked to `0x7FFF`, then logically shifted right by six |
| Small base | Twice Compact index |

Under the 32-bit model, Compact index is a multiple of four from 0 through 508. Small base is a multiple of eight from 0 through 1016. Neither is established as an offset into an allocated object.

The opening region has these ordered effects:

- A word containing Shifted base is stored at Pair-high plus `0x14`. The low byte of Pair-high is stored at Shifted base plus `0x09`. A word containing Scaled base is stored at 256 times Pair-low plus `0x54`. The byte at Shifted base plus `0x09` is then overwritten with the low byte of Shifted base.
- A zero word is stored at Scaled base plus `0x14`. An explicit unsigned byte is read at Scaled base plus `0x05`. A zero word is stored at Small base plus `0x14`, followed by a word store of the zero-extended byte at Scaled base plus `0x34`.
- A word is read at Scaled base plus `0x14`. Another zero word is stored at Small base plus `0x14`. Two consecutive byte stores at Small base plus `0x15` receive the low byte of the read word.

These are distinct accesses even where later stores overwrite earlier ones.

**[INFERENCE — ordinary-memory consequence]:** with 32-bit addressing, ordinary memory, and no external modification, the opening word read at Scaled base plus `0x14` returns zero. The intervening Small-base store also writes zero, so overlap cannot introduce nonzero bits. The word store at Scaled base plus `0x34` is disjoint from that word, including under modular address arithmetic.

Consequently, both opening byte stores at Small base plus `0x15` write zero. The second byte store at Shifted base plus `0x09` also writes zero because Shifted base has a zero low byte.

Device-backed storage, external modification, or unusual address mappings remain **documented_only** environmental cases. Aliasing alone does not defeat this zero deduction.

### Global conversion and subsequent address-derived mutation

After the duplicate opening byte stores, the operation reads the global associated with `0x05`, converts its value to a 32-bit unsigned quantity, and retains that result as **Converted base**.

The source access width and signedness are unresolved. Whether the conversion widens, truncates, preserves a numerical value, or maps a negative source value into an unsigned representation cannot be selected without the missing source type. In particular, Converted base is not established as byte-derived.

This read and conversion precede the explicit byte write at absolute `0x01`, which receives the low byte of Small base. Converted base is not subsequently replaced by rereading the global. Earlier dependency effects and computed stores may already have affected the global when it is read.

The operation writes the low byte of Compact index at Converted base plus `0x15`. At Scaled base plus `0x11`, it writes the low byte of Saved word and later overwrites that byte with zero.

Between those two Scaled-base byte stores, Saved word is stored as a word at the opening read-word value plus `0x54`. Under the ordinary-memory deduction above, this destination is absolute `0x54`; otherwise it remains result-derived.

A word at Converted base plus `0x54` is then retained as **Link word**, before that location is overwritten with the opening read-word value. Link word is the value observed at that read, not necessarily an initial-memory value.

Subsequent mutation uses Link word, Small base, and words loaded through offsets including `0x04`, `0x14`, `0x24`, `0x44`, `0x54`, `0x64`, and `0x74`. Stored values include retained words, scaled addresses, self-valued addresses, and low-byte projections. These offsets do not establish a validated record layout or graph invariant.

A later explicit word write at absolute `0x54` stores twice Link word. It follows a word store at Small base plus `0x14` containing a shifted retained result, and precedes two word stores there containing Link word. Two subsequent byte stores at Small base plus `0x15` receive the low byte of four times Link word. Only afterward does an explicit byte write at absolute `0x15` receive the low byte of Small base.

Thus the opening ordinary-memory write at `0x54` and the later explicit write there are separate events with different sources. The later explicit `0x54` write precedes the explicit `0x15` write. Computed aliases can prevent either from determining final contents.

Repeated loads and duplicate stores remain distinct where displayed. Exact machine-level ordering of multiple loads within one displayed expression, and correspondence to physical reads, are **documented_only**.

### Constant stores and the repeated region

A subsequently loaded word, **Late base**, determines three distinct constant stores:

| Destination | Width | Value |
|---|---:|---:|
| Four times Late base | 8 bits | `0x3A` |
| Eight times Late base | 16 bits | `0x3A` |
| Four times Late base plus `0x20` | 32 bits | `0x3A` |

The widths are not interchangeable.

Let **Adjusted base** mean four times Link word minus `0x96`, and **Block base** mean four times Adjusted base, under the 32-bit model.

At Block base plus `0x20`, four word stores occur in order: `0x20`, `0x2F`, a zero-extended unsigned byte read from Block base plus `0x04`, and `0x20`. That byte read occurs after the `0x2F` store and before the byte-derived word store. No character-output interface is established.

Afterward, the separate global word is read and retained as **Global base**. This read precedes the explicit word write of `0x20` at absolute `0x60`. Later overlap with that global's storage does not replace the retained value.

The repeated region preserves the ordering of four effects within each displayed group:

- A word containing four times Late base is stored at a scaled destination base plus `0x54`.
- The low byte of Block base is stored at Global base plus `0x19`.
- The low byte of Global base is stored at Block base plus `0x11`.
- The scaled destination base is stored as a word at Global base plus `0x34`.

The opening group uses eight times Link word as the destination base; subsequent groups use eight times Adjusted base. Repetition is explicitly displayed rather than controlled by a visible runtime loop.

Exact repetition cardinality is **documented_only** in this contract; no invented loop bound is assigned. Repeated events must not be replaced with a single final assignment merely because their numerical operands repeat.

### Final retained reads and absolute-byte propagation

For this region, **Anchor** means twice Late base.

Before the final mutation sequence, the operation retains:

- A word at absolute `0xA9`, called **Absolute word**.
- A word at four times Late base plus `0x74`, called **Earlier tail word**.

An explicit byte write at absolute `0x3A` receives the low eight bits of eight times Adjusted base plus `0x30`.

The operation then retains a word at Anchor plus `0x14`, called **Saved anchor word**, and a word at Block base plus `0x04`. Computed writes affect Anchor and Block base, including a byte `0x6C` at Block base plus `0x08` and a self-valued word at Anchor plus `0x54`.

Absolute-byte propagation has this established order:

1. Absolute `0x67` receives the low byte of Absolute word.
2. A word is read at Anchor plus `0x44`.
3. A fresh byte read from absolute `0x67` supplies a byte write at absolute `0x21`.
4. After further computed writes, another byte read from `0x67` supplies a write at Block base plus `0x09`.
5. Following additional computed writes and a word read at Block base plus `0x54`, a third byte read from `0x67` supplies another write at Block base plus `0x09`.

Under ordinary memory without external modification, the first propagation read supplies the byte just written: only a word read intervenes. Later reads may differ because intervening computed stores can overlap `0x67`.

Between the latter two reads, the low byte of Block base is stored at Anchor plus `0x09`, and the full retained Absolute word is stored at Block base plus `0x44`. Retained Absolute word and fresh reads from `0x67` therefore have different preservation rules.

Other ordered final effects include:

- A signed-right-shift-derived byte at Anchor plus `0x01`, using the low eight bits of Block base as the shift count.
- A word read at Block base plus `0x40`, later stored at Block base plus `0x10`.
- A word write at Global base plus `0x74` using the retained Block-base-plus-`0x04` word.
- Restoration of Saved anchor word at Anchor plus `0x54`.
- Further overwrites at Block base plus `0x09` and `0x44`, using a retained word loaded from Block base plus `0x54`.

The signed variable-shift rule, especially for counts of 32 or greater, is **documented_only**.

### Final loaded bases and transfer

A word loaded from Anchor plus `0x04` is retained as **Final base**. A halfword `0x65` is stored at twice Final base. A word loaded from Final base plus `0x64` is retained as **Final link**.

A byte write through the previously loaded Block-base-derived address uses the low byte of a word read at Block base plus `0x14`; a subsequent word write through that address stores decimal 100 at offset `0x54`.

A later word at Final base plus `0x74` supplies the fourth transfer argument. Another word is loaded at Anchor plus `0x64`; it is stored through Final link at offset `0x54`, followed by a word store of that loaded value at twice Final link. A byte at Anchor plus `0x09` receives the low byte of Final base before those last word stores.

The final dependency receives, in order:

1. `0x72`.
2. Final link.
3. Saved anchor word.
4. The later word read at Final base plus `0x74`.

The fourth argument is not Earlier tail word. No dependency result is explicitly consumed. A return is displayed afterward; reaching it requires the target to return.

## Side effects

Direct stores use 8-, 16-, and 32-bit widths. Explicit absolute stores occur at `0x01`, `0x15`, `0x21`, `0x3A`, `0x54`, `0x60`, and `0x67`.

Absolute reads include the word at `0xA9`, repeated byte reads at `0x67`, and the global read associated with `0x05`. **That global read has unresolved source width and signedness; only its conversion to a 32-bit unsigned result is established.**

Computed accesses may overlap absolute storage, global storage, and other computed regions. Word stores may overlap byte fields. Byte-level effects of mixed-width overlap generally require the target byte order.

There is no established transaction, atomicity, locking, rollback, ownership change, idempotence, reentrancy, or thread-safety guarantee. Dependency effects may extend beyond displayed direct accesses.

## Dependencies

| Opaque ID | Role | Unresolved properties |
|---|---|---|
| F-U001 | Incoming-link-derived initial target | Identity, signature, effects, return behavior |
| F-U002 | Embedded-reference-derived target family | Individual identities, signatures, effects, return behavior |
| F-U003 | Final four-argument indirect target | Target selection, effects, return behavior |

F-U002 designates a family of call targets, not one established shared function.

Displayed argument counts do not establish complete ABI signatures. Secondary machine-state results and combined results require unresolved calling-convention information. No dependency receives an assumed memory-preservation or ownership guarantee.

## Acceptance examples

These are conditional static examples, not executed tests.

### Opening zero survives ordinary overlap

With ordinary memory, supported accesses, the 32-bit model, and no external modification, the opening word read at Scaled base plus `0x14` returns zero even if the intervening Small-base zero store overlaps it. Both following byte stores at Small base plus `0x15` therefore write zero.

### Explicit unsigned-byte conversion

If the explicit unsigned-byte read at Scaled base plus `0x05` returns `0xE7`, the corresponding word store at Scaled base plus `0x34` receives `0x000000E7`.

This example does not describe the separately named global at absolute `0x05`, whose source type is unresolved.

### Converted global base is retained without a byte-range assumption

If the global read and conversion produce `0x00000120`, the subsequent Compact-index byte store targets `0x00000135`, and the Link-word read targets `0x00000174`, under the 32-bit model.

This condition specifies the converted result, not a source access width. Restricting it to `0x20` by retaining only eight bits would not preserve the established value dependency.

The global read precedes the byte write at `0x01`. Replacing the retained result with a later reread is not equivalent when intervening writes affect the global's source storage.

### Compact-index alignment

If the index-producing result has its low seven bits set, Compact index is 508 and Small base is 1016. Values of 511 and 1022 would ignore zeros introduced by the initial left shift.

### Absolute access order

The explicit word store of twice Link word at `0x54` precedes the explicit byte store at `0x15`. The Global-base read precedes the word store of `0x20` at `0x60`. These orderings remain required even if computed stores overlap the same locations.

### Fresh reads from `0x67`

Suppose Absolute word has low byte `0xA5`. With ordinary memory and no external modification, immediate propagation to `0x21` writes `0xA5`.

If an intervening computed store changes `0x67` to `0x3C`, the next write sourced from a fresh read there uses `0x3C`. Reusing retained `0xA5` is not equivalent.

### Failure does not imply rollback

If a dependency does not return, subsequent operations are not reached through the displayed sequence. If a later access fails, no undo of earlier writes or returned failure status is established.

## Uncertainties

The following remain **documented_only**:

- Application purpose and valid entry-state domain.
- Accuracy and completeness of the supplied representation.
- Dependency identities, signatures, calling convention, effects, and return conditions.
- Source width, source signedness, and consequent conversion semantics of the global associated with `0x05`.
- Scalar versus paired-machine-result interpretation.
- Memory byte order, alignment rules, low-memory mapping, and device behavior.
- Signed arithmetic and variable-shift semantics.
- Allocation boundaries, ownership, and lifetime.
- Exact repeated-store cardinality and complete call-site mapping.
- Machine-level ordering and physical-read interpretation within compound expressions.
- Final target selection and failure behavior.

Ordinary-memory deductions do not automatically extend to device-backed or externally changing storage.

## Evidence

The supplied static representation supports the displayed widths, value dependencies, and relative ordering described here. Comments, warnings, the prior draft, and reviewer feedback are not independent behavioral evidence.

The global associated with `0x05` is used through conversion to a 32-bit unsigned quantity, but its declaration is absent. This establishes neither an 8-bit source read nor unsigned-byte zero-extension. Its read-before-write relationship with the explicit `0x01` byte store remains established independently of that missing type information.

Arithmetic deductions are conditional on the stated 32-bit model. Ordinary-memory deductions separately exclude external modification and device effects.

No implementation access, runtime execution, smoke checks, or runtime tests have been performed.
