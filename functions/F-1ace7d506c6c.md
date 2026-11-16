<!-- Trusted provenance: F-1ace7d506c6c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1ace7d506c6c

## Purpose

F-1ace7d506c6c invokes an ordered sequence of indirect dependencies, transforms their results, performs mixed-width memory mutations and fixed-address global accesses, and finishes with a four-argument indirect invocation.

The supplied static representation establishes numerical relationships and access ordering, not a business purpose. Descriptions such as “link” and “publication” identify observable relationships only. Object management, formatting, initialization, and synchronization are **inferred intent**, not contractual requirements.

This contract distinguishes **established static evidence**, **inferred intent**, and **documented_only** limitations. No runtime tests have been performed.

## Interface

### Inputs and outputs

The displayed interface has no explicit arguments and no explicit return value. No success indicator, error code, returned allocation, or output length is established.

Effective inputs include incoming execution context, an incoming machine-held value used in shifted arguments, primary and secondary dependency results, existing memory, and dependency-modified state. The initial invocation destination depends on incoming context; the final destination is unresolved.

Normal completion requires the preceding operations and final invocation to complete. The displayed interface does not establish a portable calling convention or prove recovery of every effective input.

### Widths and conversions

Explicitly width-qualified computed accesses establish:

| Access or value | Width |
|---|---:|
| Byte access | 8 bits |
| Halfword access | 16 bits |
| Word access | 32 bits |
| Ordinary displayed integer/address-bearing value | 32 bits |
| Combined dependency result | 64 bits |
| Each component of a combined result | 32 bits |

Explicit unsigned-byte reads widen by zero extension. Explicit byte destinations receive the low eight bits of the supplied value. These rules do **not** establish the widths of undeclared named globals.

In particular, the global at `0x05` is converted to a 32-bit unsigned value. Its original type, load width, signedness, and extension behavior are **documented_only**. The resulting value need not lie between 0 and 255.

Likewise, supplying an eight-bit-converted value to a named global does not prove an eight-bit destination access. Global reads and writes at `0x01`, `0x15`, `0x3A`, `0x67`, and `0x21` lack declarations establishing their access widths. The named globals at `0x54`, `0x60`, `0xA9`, and `0x087FBBBC` also lack declarations. Their displayed assignment values or receiving values establish numerical uses, not independently verified storage widths.

Unsigned 32-bit arithmetic discards bits outside that width. Signed left shifts, signed multiplication, address overflow, and excessive variable shift counts require target semantics not supplied here. Numerical examples using modulo-\(2^{32}\) arithmetic are conditional.

A logical shift of a 64-bit combined result by 34, 36, or 44 bits equals its high component logically shifted by 2, 4, or 12 bits. Component extraction alone establishes no memory byte order.

### Preconditions and ownership

Every reached indirect destination must be executable and support its effective invocation interface. Every reached memory access must have suitable mapping, permissions, lifetime, and alignment. No visible validation establishes these conditions.

No direct allocation, release, retention, or ownership-transfer operation is identified. Dependencies may alter globals, memory lifetimes, or resource ownership; preservation and ownership behavior must not be assumed.

## Behavior

### Indirect dependency phase

The opening phase contains a long straight-line sequence of indirect invocations. No visible input-controlled branch or loop selects among these occurrences.

Arguments include shifted primary results, separate low and high components of combined results, secondary machine results, an incoming machine-held value, explicit zeros, and duplicated values in different positions. Some earlier results remain live across intervening invocations.

Masks select fields of 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, and 11 bits. These operations do not establish a serialized layout or named logical fields.

Intermediate widths matter. Under truncating 32-bit semantics, a left shift followed later by another shift cannot recover bits discarded by the first. Known-zero arguments do not justify removing invocations. Separate occurrences remain distinct even when their supplied arguments match.

A complete occurrence-by-occurrence dependency ledger is **documented_only** here; the summarized patterns are not permission to reorder or consolidate calls.

### Descriptive values

The following labels denote captured numbers, not allocations or disjoint memory regions:

| Label | Origin or relationship |
|---|---|
| Retained shift | An earlier primary result shifted left by eight |
| Root | An earlier combined result’s low component shifted left by 25 |
| Shifted base | A later primary result multiplied by \(2^{25}\) |
| Compact index | A later result shifted left by eight, masked to 15 bits, then logically shifted right by six |
| Paired base | Twice Compact index |
| Initial capture | Word captured at Shifted base plus `0x14` |
| Global-derived base | Value obtained by converting the undeclared global at `0x05` to unsigned 32 bits |
| Link | Word captured at Global-derived base plus `0x54` |
| Fourfold link / Eightfold link | Four / eight times Link |
| Adjusted unit | Fourfold link minus `0x96` |
| Adjusted base | Four times Adjusted unit |
| Publication base | Value captured from the global at `0x087FBBBC` into a displayed 32-bit signed integer |
| Later captured base | Word captured at Fourfold link plus `0x04` |
| Tail source | Word captured using the second explicit byte-derived address described below |
| Doubled tail / Scaled tail | Twice / four times Tail source |

Under valid 32-bit bit-vector semantics, Compact index equals four times the extraction input’s low seven bits: 0 through 508, in multiples of four. Paired base ranges from 0 through 1016 in multiples of eight.

Global-derived base has no corresponding eight-bit range guarantee. Its dependent accesses are a byte write at displacement `0x15`, followed later by a word capture and word replacement at displacement `0x54`.

### Initial mutations and captures

The direct phase begins with a word write of Root at the latest combined result’s high component plus `0x14`. Root plus `0x09` receives that high component’s low byte. A word write at 256 times the latest low component plus `0x54` then supplies Shifted base, before Root plus `0x09` receives Root’s low byte.

The initial clearing sequence preserves several important boundaries. Clearing Shifted base plus `0x14` precedes an explicit unsigned-byte capture at Shifted base plus `0x05`. Clearing Paired base plus `0x14` precedes storing that byte, zero-extended, as a word at Shifted base plus `0x34`.

Only after those writes is Initial capture read. A second word clear at Paired base plus `0x14` follows the capture, then two byte writes at Paired base plus `0x15` supply Initial capture’s low byte. Initial capture is not unconditionally zero: intervening overlapping writes or access-sensitive memory can affect its origin.

The global at `0x05` is next read and converted to form Global-derived base. The global at `0x01` is assigned Paired base after explicit eight-bit conversion. This is a global assignment of unresolved storage width, not an established byte store. The following computed byte write at Global-derived base plus `0x15` supplies Compact index’s low byte.

Retained shift supplies a byte at Shifted base plus `0x11` and a word at Initial capture plus `0x54`; the byte location is then cleared. Link is captured afterward from Global-derived base plus `0x54`. That word location is replaced with Initial capture, and Paired base plus `0x34` receives Compact index.

### Computed linking

A word captured at Shifted base plus `0x74` supplies another base. Its `0x54` word receives the earlier primary result scaled by \(2^{26}\), then later Retained shift. Between these writes is a self-linking assignment involving Paired base plus `0x74`, followed by a word self-store of Link at Link plus `0x24` and a byte write of Link at the captured base plus `0x09`.

The Link self-store and associated byte write each occur twice. The self-linking assignment contains separate word-read occurrences at Paired base plus `0x74` for destination formation and stored value. Their internal evaluation order is **documented_only**.

Paired base plus `0x14` receives the scaled earlier result, Link, and Link again, in that order. Between the first and second writes, the named global at `0x54` is assigned twice Link. That global assignment’s storage width is unresolved.

Two byte writes at Paired base plus `0x15` supply Fourfold link’s low byte. The global at `0x15` is then assigned Paired base after eight-bit conversion; its destination width remains unresolved.

Further linking uses Fourfold link as a base:

- Its `0x54` word receives Retained shift, then Fourfold link; Paired base plus `0x34` also receives Fourfold link.
- A fresh word capture at Paired base plus `0x74`, doubled, establishes an intermediate base. Link plus `0x44` receives this doubled value.
- An explicit unsigned-byte read at Fourfold link plus `0x05`, followed by a word read at that byte’s value plus `0x74`, establishes a byte-derived word base. Its `0x44` word receives the intermediate doubled value.
- Fourfold link plus `0x15` receives low bytes derived from Eightfold link and Fourfold link at distinct occurrences. Its `0x0D` byte receives the byte-derived word base’s low byte; its `0x14` word receives Fourfold link.
- A word captured at Fourfold link plus `0x64` supplies another base whose `0x0D` byte receives the same low byte and whose `0x14` word receives that base itself.
- Later captured base is then read at Fourfold link plus `0x04`. The intermediate base’s `0x09` byte receives the byte-derived word base’s low byte, and its `0x44` word receives Eightfold link.

These are numerical linking relationships, not proof of a conventional object layout. The complete intermediate access ledger is **documented_only** here.

### Two separate explicit byte observations

Unlike the undeclared global at `0x05`, both reads at Later captured base plus `0x05` are explicitly unsigned-byte accesses.

Between the first and second observations, the function writes the intermediate doubled value’s low byte at Fourfold link plus `0x09`, stores the first byte zero-extended at the byte-derived word base plus `0x44`, and stores Eightfold link at Fourfold link plus `0x14`. It also writes Later captured base’s low byte at the intermediate base plus `0x15` and Fourfold link at that base plus `0x44`.

Only then does the second byte observation occur. An intervening computed write may alias its origin.

After a byte write of Eightfold link at Fourfold link plus `0x15`, the second observed byte supplies the base for a word read at displacement `0x74`, establishing Tail source.

Subsequent mutations write Later captured base’s low byte at Fourfold link plus `0x15`, Fourfold link at its own `0x54`, Doubled tail’s low byte at Fourfold link plus `0x09`, and Eightfold link at Fourfold link plus `0x14`.

### Constants and publication

Three explicitly width-qualified stores supply `0x3A`: a byte at Scaled tail, a halfword at eight times Tail source, and a word at Scaled tail plus `0x20`.

Adjusted base plus `0x20` receives successive word values `0x20`, `0x2F`, an explicitly unsigned byte captured from Adjusted base plus `0x04` and widened to a word, then `0x20`. The byte capture follows the first two writes.

Publication base is captured before the named global at `0x60` is assigned `0x20`. The latter assignment has unresolved storage width. Publication uses the captured base, not a later reread.

The first publication group comprises a word write of Scaled tail at Eightfold link plus `0x54`, a byte write of Adjusted base at Publication base plus `0x19`, a byte write of Publication base at Adjusted base plus `0x11`, and a word write of Eightfold link at Publication base plus `0x34`, in that order.

Subsequent straight-line groups preserve this four-access pattern, substituting eight times Adjusted unit for Eightfold link in the first destination and last stored value. The byte values remain captured numerical values. Exact repetition multiplicity is **documented_only** here; these are not an input-controlled loop or merely final-state assignments.

### Final captures and global observations

Following publication, the global at `0xA9` is captured into a 32-bit value, then Scaled tail plus `0x74` is captured as a word. The original type and access width of the `0xA9` global remain unresolved.

The global at `0x3A` is assigned an expression formed by an eight-bit character conversion of Adjusted unit, multiplication by eight, and addition of `0x30`. Character signedness and destination conversion remain unresolved. Neither a byte-wide destination nor an eight-bit intermediate arithmetic result is established; under ordinary truncating byte-destination semantics, the low-byte result would equal eight times Adjusted unit plus `0x30`, modulo 256.

A word capture at Doubled tail plus `0x14` establishes the **retained tail-state word**. A separate word capture at Adjusted base plus `0x04` is also retained.

Before assigning the global at `0x67`, byte writes supply Adjusted base at Doubled tail plus `0x0D`, `0x6C` at Adjusted base plus `0x08`, and Publication base at Adjusted base plus `0x0D` and `0x09`. Doubled tail plus `0x54` receives Doubled tail as a word.

The `0x67` global is assigned the `0xA9` capture after explicit eight-bit conversion. Its destination width and assignment conversion are **documented_only**. A word capture from Doubled tail plus `0x44` follows. Only afterward is the `0x67` global freshly read and assigned to the global at `0x21`.

That transfer does not explicitly truncate the read to eight bits. Neither global’s access width or signedness is established.

The captured `0x44` word supplies a byte at Adjusted base plus `0x10` and a word at Publication base plus `0x54`. Its right shift by Adjusted base’s low eight bits supplies a byte at Doubled tail plus `0x01`; signed-right-shift and excessive-count behavior remain unresolved. Publication base plus `0x54` is subsequently replaced with Adjusted base.

A word capture at Adjusted base plus `0x40` precedes storing the earlier `0x04` capture at Publication base plus `0x74`, replacing Adjusted base plus `0x10` with the `0x40` word, and storing Doubled tail at Publication base plus `0x44`.

A fresh read of the `0x67` global then supplies an explicitly byte-wide destination at Adjusted base plus `0x09`. The destination truncates to eight bits; this does not establish the global read width.

After writing Adjusted base’s low byte at Doubled tail plus `0x09` and the retained `0xA9` value at Adjusted base plus `0x44`, a word capture at Adjusted base plus `0x54` establishes the **final link base**. Another separate `0x67` global read supplies another byte write at Adjusted base plus `0x09`.

### Final linking and invocation

The Scaled-tail `0x74` capture supplies low bytes at Final link base plus `0x0D` and, later, Doubled tail plus `0x11`. Between these writes, Doubled tail plus `0x54` receives the retained tail-state word, and Adjusted base plus `0x14` is captured.

Adjusted base plus `0x09` and `0x44` are then replaced using Final link base. A fresh word read at Doubled tail plus `0x04` establishes the **final argument base**.

A halfword store of `0x65` at twice Final argument base precedes capturing its `0x64` word. Final link base plus `0x11` receives the Adjusted-base `0x14` capture’s low byte, and its `0x54` word receives decimal 100. Final argument base plus `0x74` is then read as a word.

The last linking operations capture Doubled tail plus `0x64`, write Final argument base’s low byte at Doubled tail plus `0x09`, store the captured final-argument `0x64` word through the newly captured base plus `0x54`, and store that newly captured base at twice the final-argument `0x64` value.

The final invocation receives, in order:

1. `0x72`.
2. The captured word from Final argument base plus `0x64`.
3. The retained tail-state word.
4. The later captured word from Final argument base plus `0x74`.

These captures are not one simultaneous memory snapshot.

## Side effects

Computed accesses may overlap each other and named globals. Explicit word accesses at displacement `0x14` overlap byte accesses at `0x15`; words at `0x10` overlap bytes at `0x11`. Byte order determines which word bits those overlaps affect.

Named-global accesses retain their occurrence ordering despite unresolved widths. In particular, the `0x67` assignment, intervening word capture, transfer to `0x21`, and two later separate reads must not be collapsed into reuse of the original `0xA9` capture.

No complete structure layout, independent global regions, device-memory model, atomicity, thread safety, reentrancy, or idempotence is established.

There is no visible error branch, retry, rollback, or cleanup path. A fault or nonreturning dependency prevents later operations without visibly undoing earlier effects.

## Dependencies

| Opaque ID | Role | Unresolved properties |
|---|---|---|
| F-U01 | Initial context-derived indirect invocation | Destination, effective inputs, results, effects |
| F-U02 | Subsequent ordered indirect occurrences | Individual identities, signatures, secondary results, effects |
| F-U03 | Final four-argument indirect invocation | Destination, effects, return behavior |

F-U02 denotes a collection of occurrences, not one verified shared function. No dependency is assumed to preserve globals or memory, or to own or release resources.

## Acceptance examples

These are conditional static examples, not executed tests.

- **Independent component shifting:** Given low component `0x00123456` and high component `0x89ABCDEF`, an occurrence requesting low shifted by eight and unchanged high supplies `0x12345600` and `0x89ABCDEF`.
- **Compact boundary:** Under 32-bit bit-vector semantics, extraction input `0x7F` yields Compact index 508 and Paired base 1016; `0x80` yields zero.
- **Established widening:** An explicit unsigned-byte capture of `0xE7` at Shifted base plus `0x05` supplies word value `0x000000E7`. This rule cannot automatically be applied to the undeclared global at fixed address `0x05`.
- **Global-derived range:** If the `0x05` global’s declared value converts to `0x1234`, its dependent word capture uses `0x1288`, not an address derived from `0x34`.
- **Conditional signed input:** If that global is declared signed eight-bit and holds −1, ordinary conversion to unsigned 32 bits yields `0xFFFFFFFF`, not 255. Subsequent address formation still requires target arithmetic semantics.
- **Clear ordering:** The second paired-base clear follows Initial capture even if their locations alias.
- **Distinct byte observations:** If an intervening computed write changes Later captured base plus `0x05`, Tail source uses the second observation.
- **Global transfer:** If a fresh `0x67` read yields a value with nonzero bits above bit seven, the `0x21` assignment follows its declaration-dependent conversion. Only the later explicitly byte-wide computed destinations necessarily retain just the low eight bits.
- **Fresh observations:** Computed writes changing `0x67` affect subsequent reads according to its actual declaration and memory behavior; reusing the earlier capture is not equivalent.
- **Final arguments:** Later changes to the final-argument `0x64` origin do not replace the retained second argument. The fourth argument uses the later `0x74` capture.
- **Interruption:** If F-U03 never returns, preceding mutations may remain observable without normal completion.

## Uncertainties

The following remain **documented_only**:

- Business purpose and conventional object interpretation.
- Complete dependency ledger, identities, signatures, effects, and ownership obligations.
- Boundary and calling-convention accuracy, incoming context, and secondary-result semantics.
- Original types, widths, signedness, and conversion behavior of undeclared named globals, including `0x05` and `0x67`.
- Global assignment widths at `0x01`, `0x15`, `0x3A`, and `0x21`; an eight-bit supplied value is not sufficient evidence.
- Byte order, alignment, address mapping, and access-sensitive memory behavior.
- Signed arithmetic, character signedness, overflow, and excessive shift counts.
- Internal evaluation order of the identified self-linking reads.
- The unenumerated intermediate access ledger and publication multiplicity.
- Final destination, return behavior, and fidelity of the supplied static representation to executable behavior.

None authorizes assuming non-aliasing, eight-bit global inputs, state preservation, or harmless repeated accesses.

## Evidence

The sole behavioral evidence is the supplied static representation. Prior prose, embedded comments, and reviewer feedback are not independent authority.

Explicit width-qualified computed accesses support byte, halfword, word, and unsigned-byte-widening claims. Undeclared named-global accesses support occurrence ordering and displayed numerical conversions, but not storage-width or signedness claims absent their declarations.

No implementation access or runtime observation supports stronger conclusions. Trusted provenance is left to the publishing system.
