<!-- Trusted provenance: F-c2d8809dae5f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c2d8809dae5f

## Purpose

F-c2d8809dae5f performs an ordered sequence of indirect dependency invocations, transforms captured results, modifies memory using mixed-width accesses, and makes a terminal indirect invocation with four operands.

The supplied static representation establishes no application-level purpose. Character-like constants and address-like values do not establish formatting, serialization, initialization, or a particular data structure.

This contract distinguishes:

- **Established static evidence:** represented interactions, access widths, captured-value relationships, and relative access order.
- **Inferred intent:** possible interpretations, not acceptance requirements.
- **documented_only:** behavior not resolved here, including behavior requiring missing declarations, dependency contracts, or execution-model evidence.

No runtime tests have been performed.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters and no explicit return value. Its implicit inputs include:

- An incoming return-link value used to derive the first indirect target.
- Execution-location-derived values used for later targets.
- Primary and secondary dependency result channels.
- Memory reached through captured results, loaded words, scaled values, and explicit byte reads.
- Fixed-address storage, including an input associated with address `0x05`.
- A separate 32-bit global captured as an association base.
- An unresolved terminal indirect target.

Observable outputs are dependency interactions, memory accesses and mutations, and the terminal invocation. No explicit status result, validation branch, recovery operation, or rollback is represented.

Progression requires executable targets, returning dependencies wherever subsequent operations depend on their return, and accessible storage for reached accesses. These conditions are not checked by the represented function.

A return follows the terminal invocation; it is reachable only if that invocation returns.

### Widths and arithmetic

| Quantity | Represented width or qualification |
|---|---|
| Explicit byte-pointer access | 8 bits; numerical reads are zero-extended |
| Halfword store | 16 bits |
| Word access and address-like arithmetic | 32 bits |
| Paired dependency result | 64 bits, with separately consumed low and high words |
| Fixed-address `0x05` input | Storage width and signedness unresolved; its value is converted to unsigned 32 bits |

Byte and halfword stores retain their respective low-order bits. Numerical splitting of a paired result does not establish memory endianness.

Arithmetic examples use a **conditional 32-bit bit-vector model**: represented 32-bit addition, multiplication, and left shifts retain the low 32 bits, including intermediate results. This is an analysis model, not an established execution guarantee. Signed overflow, signed shifts, oversized shift counts, and actual address semantics remain unresolved where applicable.

Offsets are byte offsets. They establish neither field types nor allocation boundaries.

### Fixed-address input qualification

The value associated with address `0x05` is converted to unsigned 32 bits before use as a base. The supplied representation does not declare that global’s original type.

Consequently, this conversion alone does **not** establish:

- An eight-bit memory read.
- An unsigned source value.
- Zero-extension from eight bits.
- A resulting range of 0–255.

This contract calls the converted value **Fixed-input base**, without asserting those properties. If independent declaration evidence establishes an unsigned eight-bit source, the 0–255 range follows conditionally. Later explicit byte-pointer reads are distinct and do establish eight-bit zero-extended values.

## Behavior

### Indirect-invocation prefix

A long straight-line prefix performs indirect invocations with zero to four represented operands. No conditional branch or loop is represented within that prefix.

The first target is derived from the incoming return link plus `0xffe`. Later targets use execution-location-derived values with the same displacement. This does not establish object-field dispatch.

Operand construction includes:

- Left shifts by eight bits and other site-specific counts.
- Unchanged high words and shifted copies of high words.
- Multiple transformations of the same captured result.
- Repeated operands and explicit zeros.
- Masking before shifting.
- Logical right shifts of unsigned intermediates.
- Values retained across intervening invocations.

Masks include fields of 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, and one bit. Their placement relative to shifts matters.

Some extractions logically shift an unsigned 64-bit result by 34, 36, or 44 bits before narrowing. These select the high word shifted right by two, four, or twelve bits, respectively; they are not low-word shifts.

Invocations whose results are ignored remain interactions. Zero represented operands do not establish absence of machine-state inputs. Dependencies are not assumed pure, globally preserving, or ownership-neutral.

The complete site-by-site prefix interaction schedule remains **documented_only**.

### Captured values entering memory mutation

The following names describe numerical values, not recovered objects:

| Name | Definition |
|---|---|
| Anchor | A captured result multiplied by `0x02000000` |
| Link value | A later captured result shifted left eight bits |
| Shifted token | A captured paired result’s low word shifted left 25 bits |
| Extracted field | A captured result shifted left eight bits, masked to 15 bits, then logically shifted right six bits |
| Small base | Twice Extracted field |
| Final pair | The last paired dependency result before explicit stores |

Under the conditional arithmetic model, Extracted field is a multiple of four from 0 through 508. Small base is a multiple of eight from 0 through 1016. Neither range establishes valid storage.

The opening mutation group has this access order:

- A word containing Shifted token is stored at Final pair’s high word plus `0x14`.
- That high word’s low byte is stored at Shifted token plus `0x09`.
- Anchor is stored as a word at 256 times Final pair’s low word plus `0x54`.
- Shifted token’s low byte is stored at Shifted token plus `0x09`.

The function then clears the word at Anchor plus `0x14`, captures the byte at Anchor plus `0x05`, clears the word at Small base plus `0x14`, and stores the captured byte as a zero-extended word at Anchor plus `0x34`.

It rereads Anchor plus `0x14` as a word, clears Small base plus `0x14` again, and writes the reread word’s low byte twice at Small base plus `0x15`.

The reread and duplicate stores are represented accesses, not merely final-state descriptions. Aliasing, external mutation, or device semantics can distinguish them.

### Fixed input and loaded-address mutation

The Fixed-input base capture precedes the byte store of Small base’s low byte at fixed address `0x01`. Its source access width remains unresolved.

Extracted field’s low byte is then stored at Fixed-input base plus `0x15`. Link value’s low byte is stored at Anchor plus `0x11`; an intervening word store places Link value at the previously reread word plus `0x54`; Anchor plus `0x11` is then cleared.

The word at Fixed-input base plus `0x54` is captured before replacement with the previously reread word. Call this retained value **Captured link**. Subsequent operations use it as data, an address base, and a scaling operand. No byte-sized bound on Fixed-input base is required for these relationships.

The following region includes:

- A word capture at Anchor plus `0x74`, followed by stores through that loaded address.
- A store whose destination base and stored word each involve a read at Small base plus `0x74`.
- Repeated self-value word stores at Captured link plus `0x24`.
- A fixed word store of twice Captured link at `0x54`.
- Repeated stores at Small base plus `0x14` and duplicate byte stores at Small base plus `0x15`.
- A fixed byte store of Small base’s low byte at `0x15`.

The offset-`0x74` reads are not established as one cached observation. For the store expression containing two such reads, no independent ordering between those reads is established here; both contribute to that store.

Further updates combine four times Captured link, eight times Captured link, doubled loaded words, explicit byte-derived bases, and words loaded at offsets `0x04`, `0x64`, and `0x74`. Repeated explicit byte reads at offset `0x05` remain separate observations.

These updates include byte stores at offsets `0x09`, `0x0d`, and `0x15`, and word stores at offsets `0x14`, `0x34`, `0x44`, and `0x54`. Loaded values are reused as both addresses and data. The exhaustive intermediate access schedule remains **documented_only**.

**Inferred intent:** association or linking is possible. No list, tree, ownership graph, or structural invariant is established.

### Constant stores and association capture

The remaining regions use these retained values:

| Name | Definition |
|---|---|
| Adjusted value | Four times Captured link minus `0x96` |
| Adjusted base | Four times Adjusted value |
| Colon seed | The last word loaded through offset `0x74` before the constant-store group |
| Colon base | Four times Colon seed |
| Terminal base | Twice Colon seed |
| Association base | The value captured from the separate 32-bit global |

These arithmetic relationships are conditional on the stated model. No overflow, underflow, or address-range check is represented.

Numeric `0x3a` is stored as:

- A byte at Colon base.
- A halfword at eight times Colon seed.
- A word at Colon base plus `0x20`.

Adjusted base plus `0x20` then receives four word stores: `0x20`, `0x2f`, the zero-extended byte read from Adjusted base plus `0x04`, and `0x20` again. The byte read occurs between the second and third stores.

Association base is captured **after these computed stores and before** the fixed word store of `0x20` at address `0x60`.

Earlier computed stores may overlap the association global. Later mutations may change its backing storage, but subsequent association operations use the captured value rather than reloading it.

### Repeated association updates

Following the fixed store at `0x60`, repeated groups each perform four stores in this order:

1. A word containing Colon base at a scaled base plus `0x54`.
2. A byte containing Adjusted base’s low byte at Association base plus `0x19`.
3. A byte containing Association base’s low byte at Adjusted base plus `0x11`.
4. A word containing the scaled base at Association base plus `0x34`.

The first group uses eight times Captured link as its scaled base. Later groups use eight times Adjusted value.

The destinations do not advance. These groups do not establish array traversal, and collapsing them to one final-state-equivalent group is not generally valid. The exact repetition count remains **documented_only** in this contract.

### Terminal capture boundary and numerical overlap

The transition into terminal mutation has this established order:

| Relative position | Access |
|---|---|
| Before the byte store | Capture the word at fixed address `0xa9` |
| Before the byte store | Capture the word at Colon base plus `0x74` |
| Boundary mutation | Store a byte at fixed address `0x3a` |
| After the byte store | Capture the word at Terminal base plus `0x14` |
| After the byte store | Capture the word at Adjusted base plus `0x04` |

The byte stored at `0x3a` is the low eight bits of eight times Adjusted value plus `0x30`.

The four captured words are called **Fixed capture**, **Earlier terminal link**, **Retained field**, and **Retained offset-four word**, respectively.

**Among the two captures after the byte store, only Terminal base plus `0x14` can numerically include byte address `0x3a` under the conditional model.** This is not an exclusivity claim about all four captures.

In particular, the earlier capture at Colon base plus `0x74` can overlap `0x3a` through address wraparound. Its occurrence before the store determines capture timing, not numerical disjointness.

The exclusion for Adjusted base plus `0x04` follows from:

- Adjusted base equals 16 times Captured link minus `0x258`, modulo \(2^{32}\).
- Its offset-four word therefore starts at 16 times Captured link minus `0x254`.
- That starting address has residue `0x0c` modulo 16.
- Its four numerical byte addresses have residues `0x0c` through `0x0f`.
- Address `0x3a` has residue `0x0a`.

Thus that particular later word access cannot directly include `0x3a`. The fixed capture at `0xa9` is also numerically disjoint from `0x3a`.

These statements concern numerical addresses. They do not resolve physical-memory aliases or device mappings.

### Terminal mutations and live reads

Initial terminal stores place Adjusted base’s low byte at Terminal base plus `0x0d`; byte `0x6c` at Adjusted base plus `0x08`; and Association base’s low byte at Adjusted base plus `0x0d` and `0x09`. Terminal base is stored as a word at its own offset `0x54`.

Fixed capture’s low byte is stored at fixed address `0x67`. Three subsequent uses of that address are separate byte reads:

| Consumer | Timing |
|---|---|
| Byte store at fixed address `0x21` | After capturing Terminal base plus `0x44` |
| First subsequent byte store at Adjusted base plus `0x09` | After intervening computed writes, including Association base plus `0x44` |
| Second subsequent byte store at Adjusted base plus `0x09` | After writing Fixed capture at Adjusted base plus `0x44` and capturing Adjusted base plus `0x54` |

Each consumer receives its occurrence’s read value. Those values need not equal Fixed capture’s low byte or one another.

The captured word from Terminal base plus `0x44` supplies a low-byte store at Adjusted base plus `0x10` and a word store at Association base plus `0x54`. Its signed right-shift result supplies a byte at Terminal base plus `0x01`; the count is Adjusted base’s low eight bits. Association base plus `0x54` subsequently receives Adjusted base.

The shift count can exceed 31. Signed right-shift and oversized-count behavior remain **documented_only**.

Further terminal operations:

- Capture Adjusted base plus `0x40`, store Retained offset-four word at Association base plus `0x74`, then store the newly captured word at Adjusted base plus `0x10`.
- Store Terminal base at Association base plus `0x44`.
- Forward Adjusted base’s low byte to Terminal base plus `0x09`.
- Store Fixed capture at Adjusted base plus `0x44`, then capture Adjusted base plus `0x54` as **Loaded destination**.
- Forward Earlier terminal link’s low byte to Loaded destination plus `0x0d`.
- Restore Retained field at Terminal base plus `0x54`.
- Capture Adjusted base plus `0x14` before forwarding its low byte later.
- Forward Earlier terminal link’s low byte to Terminal base plus `0x11`.
- Store Loaded destination’s low byte at Adjusted base plus `0x09`, and its full word at Adjusted base plus `0x44`.

A word loaded from Terminal base plus `0x04` becomes **Loaded base**. A halfword containing `0x65` is stored at twice Loaded base.

The word at Loaded base plus `0x64` is then captured as **Forwarded word**. After this capture, the previously retained offset-`0x14` word’s low byte is stored at Loaded destination plus `0x11`, followed by word value 100 at Loaded destination plus `0x54`.

The function captures Loaded base plus `0x74` as **Final link**, then captures Terminal base plus `0x64` as **Final destination base**. It stores Loaded base’s low byte at Terminal base plus `0x09`, Forwarded word at Final destination base plus `0x54`, and Final destination base at twice Forwarded word.

### Terminal invocation

F-unknown-02 receives four represented operands, in order:

1. Numeric `0x72`.
2. Forwarded word.
3. Retained field.
4. Final link.

Final link is captured before the later read at Terminal base plus `0x64` and the final linking stores.

These are retained operands. Reloading their backing memory immediately before invocation is not generally equivalent.

## Side effects

The function exposes an access sequence, not merely a final memory image. Repeated writes, overwritten writes, and live rereads can remain observable.

No general disjointness is established:

- Byte and halfword stores can affect later word reads.
- Computed addresses can coincide with fixed addresses.
- Derived bases can overlap.
- Computed writes can change the association global or fixed byte `0x67`.
- Loaded addresses can reflect earlier mutations.

The specific offset-four exclusion above does not imply broader disjointness. Conversely, general aliasing uncertainty does not invalidate that numerical exclusion.

No allocation, release, or ownership transfer is identified in the function’s explicit operations. Dependencies may allocate, release, invalidate storage, or mutate globals; their contracts do not exclude these effects.

No locking, atomicity, transaction, synchronization barrier, recovery path, or rollback is represented. Invalid targets, inaccessible memory, unsupported alignment, or interruption can prevent completion after partial mutation. No clean error-return behavior is established.

## Dependencies

| Opaque ID | Established interaction | Unresolved properties |
|---|---|---|
| F-unknown-01 | Collective label for ordered prefix invocation sites, with zero to four represented operands and primary, paired, or secondary results | Individual identities, calling convention, side effects, preservation, ownership, failures, and return guarantees |
| F-unknown-02 | Terminal indirect invocation with the four specified operands | Target origin, dispatch meaning, memory effects, ownership, and return behavior |

The collective prefix label does not assert a shared implementation. The terminal target is not established to differ from every prefix target.

## Acceptance examples

These are conditional static examples, not executed tests.

### Paired extraction and intermediate width

For low word `0x00123456` and high word `0x89abcdef`, an applicable eight-bit low-word forwarding operand is `0x12345600`. An unchanged high-word operand is `0x89abcdef`. Logical extraction of the paired result by 36 bits yields `0x089abcde`.

Under the conditional model, retaining 24 bits of `0xffabcdef` and shifting left six yields `0x2af37bc0`. A 32-bit intermediate shifted left 28 bits and then another 28 bits becomes zero; retaining a wider intermediate is not equivalent.

### Fixed-input declaration alternatives

If the missing declaration establishes an unsigned eight-bit source at `0x05`, source value `0x80` gives Fixed-input base `0x00000080`, and its offset-`0x54` word starts at `0xd4`.

If instead it establishes a signed eight-bit source with value −128, unsigned 32-bit conversion yields `0xffffff80`; under the conditional address model, adding `0x54` gives `0xffffffd4`.

Neither declaration is established. An implementation cannot select the first case solely because the source is associated with address `0x05`.

### Association capture timing

If a computed store changes the association global before capture, the capture observes the changed value under ordinary memory semantics. A subsequent change to its backing storage does not replace the retained Association base.

The capture precedes the fixed word store at `0x60`.

### Later overlap at `0x3a`

For Captured link `0x2e`, Adjusted value is `0x22`, Adjusted base is `0x88`, and the byte stored at `0x3a` is `0x40`.

With Colon seed `0x13`, Terminal base is `0x26`; its offset-`0x14` word starts at `0x3a`. Under ordinary little-endian memory, with the other three bytes zero at capture and no intervening change, Retained field is `0x00000040`.

Adjusted base plus `0x04` is `0x8c`, occupying `0x8c` through `0x8f`; it is not directly overwritten by the byte at `0x3a`.

### Earlier overlap through wraparound

For Colon seed `0x3ffffff1`, Colon base is `0xffffffc4`. Adding `0x74` modulo \(2^{32}\) gives `0x38`; that four-byte capture includes `0x3a`.

Earlier terminal link therefore captures overlapping storage **before** the byte store. The later store does not retroactively change the retained operand. Numerical overlap and capture timing are separate properties.

### Live reads and widths

If the three byte reads at `0x67` return `0x11`, `0x22`, and `0x33`, their respective consumers receive those values.

If Retained field’s backing storage changes after capture, the terminal third operand remains the captured word.

The three constant stores of `0x3a` require byte, halfword, and word widths respectively. Three byte stores are not equivalent.

## Uncertainties

The following remain **documented_only**:

- Complete prefix invocation schedule, exhaustive intermediate mutation schedule, and exact association repetition count.
- Dependency identities, calling conventions, secondary-channel semantics, effects, and return guarantees.
- The fixed-address `0x05` input’s declaration, storage width, signedness, and resulting value range.
- Endianness, alignment, memory mappings, and device semantics.
- Signed overflow, signed shifts, oversized counts, and actual address arithmetic.
- Object boundaries, field meanings, lifetimes, ownership, concurrency, and reentrancy.
- Terminal target origin and terminal return behavior.

Packed-field processing, linking, and character construction remain inferred intent only.

## Evidence

The supplied static representation is the sole behavioral evidence. Embedded comments, the prior draft, and reviewer feedback are not independent evidence.

The represented accesses establish the association-global capture before the fixed store at `0x60`, the byte store at `0x3a` between two pairs of word captures, and three separate later byte reads at `0x67`.

The offset-four exclusion follows from its address residue modulo 16. The earlier overlapping capture follows independently from 32-bit wraparound at Colon base plus `0x74`.

The unsigned 32-bit conversion of the fixed-address `0x05` input establishes the converted value’s width, not the source storage width or signedness.

No external implementation inspection, dependency inspection, runtime execution, or tests have been performed. Claims are conditional on the fidelity of the supplied representation.
