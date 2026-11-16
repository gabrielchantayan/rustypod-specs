<!-- Trusted provenance: F-efbaa895fb31; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-efbaa895fb31

## Purpose

F-efbaa895fb31 performs a long, ordered chain of unresolved indirect invocations, transforms captured results, updates interdependent memory locations using several access widths, and finishes with an unresolved indirect transfer carrying four displayed arguments.

No application-level purpose is established. Character-like constants, repeated offsets, and address-like values do not establish text processing, object construction, device initialization, or collection management.

This contract distinguishes:

- **Established static evidence:** interactions present in the supplied representation.
- **[INFERENCE]:** consequences requiring an explicitly stated arithmetic or execution interpretation.
- **documented_only:** unresolved behavior without a supported executable requirement.

The ordering described below belongs to an **abstract access-trace model**: reached invocation and memory-access expressions are recorded in their represented order, with captured values retained separately from memory. This model does not establish that every expression produces a distinct physical access, or that another observer can distinguish every store. Volatile qualification, synchronization, and an executable memory-model contract are not supplied.

No runtime tests have been performed.

## Interface

### Calling interface and implicit inputs

The represented interface has no explicit parameters and no declared return value. No status code, output buffer, or result object is exposed.

Execution nevertheless depends on incoming control context, dependency results, secondary result channels, and ambient memory. The first indirect target is derived from incoming control context with displacement `0xFFE`. Subsequent invocation targets generally derive from embedded locations with that displacement.

An invocation without displayed arguments is not established as having no implicit inputs. Displayed arguments likewise do not establish complete dependency signatures.

Results are represented as 32-bit quantities or 64-bit containers. A container’s low and high words mean its numerical low and high 32-bit components; this does not specify their byte order in memory. Additional 32-bit result channels are consumed without an established calling convention.

### Widths and layouts

| Interaction | Width | Interpretation |
|---|---:|---|
| Byte read | 8 bits | Zero-extended when used as an unsigned word or byte-derived address base. |
| Byte write | 8 bits | Retains the low eight bits of the supplied value. |
| Halfword write | 16 bits | Includes constants `0x003A` and `0x0065`. |
| Word read or write | 32 bits | May carry an integer-like or address-like value. |
| Paired result | 64 bits | Numerically separated into 32-bit components. |

Offsets are byte offsets. Recurring offsets do not establish structures, valid bounds, object sizes, or disjoint memory regions.

Fixed byte locations include `0x01`, `0x05`, `0x15`, `0x21`, `0x3A`, and `0x67`. Fixed word locations include `0x54`, `0x60`, `0xA9`, and `0x087FBBBC`. Their numeric addresses do not establish whether they designate ordinary memory, device registers, or inaccessible locations.

### Preconditions and errors

Every reached invocation target must support its invocation, and every reached memory access must be supported by the environment. Some word or halfword accesses may be unaligned.

These are execution prerequisites, not validations performed by the function. No explicit null checks, bounds checks, permission checks, allocation checks, or recovery interface are represented.

If an invocation does not return normally or an access cannot complete, subsequent interactions are not unconditional requirements. Exception type, error signaling, recovery, and rollback remain documented_only.

## Behavior

### Invocation chain and arithmetic

The invocation chain is straight-line in the supplied representation. No retry loop or explicit conditional branch is shown. Each later operation depends on earlier operations completing normally.

Displayed arguments include shifted primary results, numerical halves of paired results, explicit zeros, duplicate arguments, retained earlier results, and secondary-channel values. A retained result is not necessarily replaced by a newer result from an intervening invocation.

Arithmetic includes shifts, low-bit masks, multiplication by powers of two, and subtraction by `0x96`. Mask widths include **1 and 8 bits**, as well as wider fields from 11 through 24 bits. The wider masks are not the complete set.

Two narrower uses are significant:

- A retained unsigned quantity is masked to one bit and shifted left by 16 into invocation arguments. The resulting field occupies bit 16, rather than preserving an arbitrary wider portion of the quantity.
- Publication base is masked to eight bits to obtain the final-phase right-shift count. This mask limits the count to 0–255, not to the supported range of a 32-bit shift.

Unsigned right shifts introduce zeros. Shifting an unsigned 64-bit result right by 34, 36, or 44 bits selects its high word shifted right by 2, 4, or 12 bits, respectively.

**[INFERENCE]** Under 32-bit bitvector semantics, intermediate left shifts and multiplications retain only their low 32 bits. Intermediate truncation matters: two successive left shifts by 28 produce zero, not an unbounded 56-bit placement.

Some expressions instead have signed arithmetic or signed-shift representations. Their portable overflow behavior, negative-value shifting, excessive shift counts, and address wrapping are not established.

### Captured quantities used below

These descriptive names identify relationships, not source identifiers or semantic object types.

| Quantity | Meaning |
|---|---|
| **High base** | A retained paired-result low word shifted left by 25 bits. |
| **Scaled base** | Another retained result multiplied by `0x02000000`. |
| **Shifted value** | A later primary result shifted left by eight bits. |
| **Field value** | Another result shifted left by eight, masked to 15 bits, then logically shifted right by six. |
| **Small base** | Twice field value. |
| **Opening word** | The subsequent word capture at scaled base plus `0x14`. |
| **Old link** | The word captured from a fixed-byte-derived base plus `0x54`, before replacement. |
| **Fourfold base** | Four times old link. |
| **Adjusted value** | Fourfold base minus `0x96`. |
| **Publication base** | Four times adjusted value. |

**[INFERENCE]** Under 32-bit bitvector semantics, field value is a multiple of four from 0 through 508, and small base is a multiple of eight from 0 through 1016. High base and scaled base have zero low bytes. These properties do not establish accessible addresses.

### Opening memory updates

After the final invocation in the opening chain, high base is stored as a word at the final paired result’s high word plus `0x14`. That high word’s low byte is stored at high base plus `0x09`. Scaled base is then stored at 256 times the final low word plus `0x54`, before high base’s low byte replaces the earlier byte at high base plus `0x09`.

A zero word is stored at scaled base plus `0x14`. The byte at scaled base plus `0x05` is captured; after a zero word is stored at small base plus `0x14`, the captured byte is published as a zero-extended word at scaled base plus `0x34`.

Opening word is then captured from scaled base plus `0x14`. Another zero word is stored at small base plus `0x14`, followed by two byte stores of narrowed opening word at small base plus `0x15`.

Only after these calculated-address interactions is fixed byte `0x05` read. Its then-current, zero-extended value supplies the **byte-derived base**. It is not an entry-state snapshot.

The following updates place narrowed small base at fixed byte `0x01`, narrowed field value at byte-derived base plus `0x15`, and narrowed shifted value at scaled base plus `0x11`. Shifted value is also stored as a word at opening word plus `0x54`, before the byte at scaled base plus `0x11` is replaced with zero.

Old link is subsequently captured from byte-derived base plus `0x54`; that word is replaced with opening word. Earlier stores may alias either the fixed-byte read or this word capture.

### Interlinked propagation and fixed locations

The next region propagates field value, shifted value, old link, and newly loaded words through calculated destinations. It includes:

- A word store of field value at small base plus `0x34`.
- A word capture at scaled base plus `0x74`, used as a destination base.
- Two read expressions involving small base plus `0x74` within one relationship. Their evaluation order within that expression is not separately established.
- Repeated stores of old link at old link plus `0x24`, interleaved with narrowed stores at the loaded destination base plus `0x09`.

A word containing the scaled-base-producing result shifted left by 26 is stored at small base plus `0x14`. Fixed word `0x54` then receives old link shifted left by one, before two stores of old link at small base plus `0x14`.

Two narrowed stores of fourfold base at small base plus `0x15` precede a store of narrowed small base at fixed byte `0x15`. Thereafter, shifted value is stored at fourfold base plus `0x54`, and fourfold base is stored at small base plus `0x34`.

Later propagation uses doubled and eightfold captured values, a word read at fourfold base plus `0x64`, a word read at fourfold base plus `0x04`, and byte-derived word-read bases. Offset-`0x15` bytes are overwritten with different retained values.

A byte at a loaded base plus `0x05` is read twice with calculated writes between the reads. These are separate captures in the abstract trace; equality is not assumed.

The **terminal link** is the word obtained through the later byte-derived base plus `0x74`. The **terminal base** is twice terminal link. Terminal link is retained before the constant-write region rather than reread afterward.

### Constants and publication

The constant-write region places `0x3A` in three widths and locations:

- One byte at four times terminal link.
- One halfword at eight times terminal link.
- One word at four times terminal link plus `0x20`.

The subtraction defining adjusted value occurs between the byte and halfword stores.

At publication base plus `0x20`, four word-store expressions occur in this order: `0x20`, `0x2F`, a zero-extended byte read at publication base plus `0x04`, and `0x20` again. The byte read belongs after the first two stores and before the third store.

This is an established order in the abstract access-trace model. It does **not** establish four distinguishable runtime observations, four physical bus transactions, or a prohibition on optimization under an otherwise unspecified execution model.

After the last store in that progression, fixed word `0x087FBBBC` is captured as the **publication link**. Fixed word `0x60` then receives `0x20`.

The repeated publication region uses four times terminal link as its published word value. Its initial group places that value at eight times old link plus `0x54`; places narrowed publication base at publication link plus `0x19`; places narrowed publication link at publication base plus `0x11`; and places eight times old link at publication link plus `0x34`.

Subsequent groups retain those two byte destinations and captured byte values. They instead use eight times adjusted value for the offset-`0x54` destination base and for the word written at publication link plus `0x34`. Each group retains the represented word–byte–byte–word order.

The representation contains explicitly repeated groups, not an established counter-controlled loop. This contract does not quantify their exact repetition count; exact-count acceptance is documented_only. Collapsing them to final-state assignments would change the abstract trace, but physical distinguishability and permissible optimization remain unresolved.

### Final capture boundary

After the repeated region, fixed word `0xA9` is captured, followed by the word at four times terminal link plus `0x74`.

Fixed byte `0x3A` is then written with the narrowed result of eight times adjusted value plus `0x30`. Only afterward are the words at terminal base plus `0x14` and publication base plus `0x04` captured.

The fixed-byte store may overlap either later word read. Treating all four word captures as preceding that store would change the abstract trace and could change captured values under an aliasing interpretation.

### Fixed-byte copies and fresh observations

The next updates place narrowed publication base at terminal base plus `0x0D`, constant `0x6C` at publication base plus `0x08`, and narrowed publication link at publication base plus `0x0D` and plus `0x09`. Terminal base is stored as a word at terminal base plus `0x54`.

Fixed byte `0x67` receives the low byte of the earlier fixed-`0xA9` word capture. A word is then read at terminal base plus `0x44`; afterward, fixed byte `0x21` receives a byte read from `0x67`.

The offset-`0x44` word contributes a narrowed store at publication base plus `0x10`, a word store at publication link plus `0x54`, and a shifted-byte store at terminal base plus `0x01`. The right shift is represented as signed, with the low eight bits of publication base supplying its count. Counts outside the supported shift range remain documented_only.

Publication link plus `0x54` is subsequently overwritten with publication base. After the word at publication base plus `0x40` is captured, intervening stores publish the earlier offset-`0x04` capture at publication link plus `0x74`, the offset-`0x40` capture at publication base plus `0x10`, and terminal base at publication link plus `0x44`.

A fresh read of fixed byte `0x67` supplies a store at publication base plus `0x09`. Narrowed publication base is then stored at terminal base plus `0x09`, and the original fixed-`0xA9` word capture is stored at publication base plus `0x44`.

The word at publication base plus `0x54` is captured as **final base**. Another fresh read of `0x67` supplies another store at publication base plus `0x09`.

Neither later fixed-byte read is interchangeable with the originally stored low byte: intervening calculated writes may overlap `0x67`. The earlier copy to fixed `0x21` likewise has its own read point.

### Remaining updates and final transfer

The low byte of the earlier four-times-terminal-link-relative `0x74` capture is stored at final base plus `0x0D`. Terminal base plus `0x54` is restored from the retained terminal-base-relative `0x14` capture.

A new word capture at publication base plus `0x14` supplies a later narrowed store at final base plus `0x11`. Before that store, the retained offset-`0x74` byte is also placed at terminal base plus `0x11`; narrowed final base replaces publication base plus `0x09`; and final base replaces publication base plus `0x44` as a word.

A word read at terminal base plus `0x04` supplies **last link**. Halfword `0x0065` is stored at twice last link before the word at last link plus `0x64` is captured. The narrowed publication-base-relative `0x14` capture is then stored at final base plus `0x11`, followed by word 100 at final base plus `0x54`.

The remaining captures are the words at last link plus `0x74` and terminal base plus `0x64`, in that order. Narrowed last link is stored at terminal base plus `0x09`. The last-link-relative `0x64` capture is stored at the terminal-base-relative `0x64` capture plus `0x54`; the latter capture is also stored at twice the former capture.

The final unresolved transfer receives four displayed arguments, in order:

1. `0x72`.
2. The last-link-relative `0x64` capture.
3. The earlier terminal-base-relative `0x14` capture.
4. The last-link-relative `0x74` capture.

A return is represented afterward. Return to the original caller depends on that transfer returning normally.

## Side effects

Direct effects comprise mixed-width memory updates, fixed-address accesses, repeated stores, and reads whose captured values determine later destinations.

No non-aliasing guarantee exists. A byte at offset `0x15` overlaps the word beginning at offset `0x14` when both use the same base. Numeric overlap effects depend on endianness. Captured values remain distinct from subsequent contents of their original locations.

No direct allocation, release, reference-count protocol, or ownership transfer is established. Unknown dependencies may allocate, retain, invalidate, or release resources; the contract assumes none of these possibilities absent.

No rollback or transactional behavior is represented. Atomicity, synchronization, reentrancy, thread safety, memory barriers, device effects, physical access counts, and observer visibility remain documented_only. Abstract trace order alone supplies none of those guarantees.

## Dependencies

F-efbaa895fb31 is the only supplied opaque function ID. No opaque IDs are provided for its indirect targets. Assigning additional identities or semantic names would be unsupported.

| Dependency surface | Established interaction | Unresolved information |
|---|---|---|
| Incoming-context-derived target | Initial indirect invocation | Identity, full signature, incoming-state requirements |
| Embedded-location-derived targets | Ordered invocations with transformed arguments | Semantics, failures, ambient effects |
| Secondary result channels | Values consumed as 32-bit quantities | Calling convention and production guarantees |
| Final indirect target | Four displayed arguments beginning with `0x72` | Identity, resource effects, return behavior |

Unknown callees are not assumed to preserve globals, ambient memory, or resource validity.

## Acceptance examples

These are conditional static obligations, not executed tests.

1. **Word truncation:** Under 32-bit bitvector semantics, shifting `0x00123456` left by eight produces `0x12345600`.

2. **Unsigned paired extraction:** If a paired result’s high word is `0x80000008`, its logical right shift by three produces `0x10000001`, not a sign-extended result.

3. **One-bit argument field:** If the retained unsigned quantity presented to the one-bit mask is 2, its shifted field is zero; if it is 3, the field is `0x00010000`. Using the entire retained quantity would not satisfy this transformation.

4. **Eight-bit shift-count mask:** Publication base `0x00000104` supplies count 4. A low byte of `0x24` supplies count 36; the mask does not establish defined 32-bit shifting for that count.

5. **Small-base boundary:** A field-producing result whose low seven bits are `0x7F` produces small base 1016 under bitvector semantics. Zero is also possible. Neither establishes accessible memory.

6. **Fixed-byte timing:** In the abstract memory model, if earlier calculated stores alter fixed byte `0x05`, its later read uses then-current memory, not an entry snapshot.

7. **Publication capture:** The fixed-slot read follows the four offset-`0x20` stores and precedes the fixed word store at `0x60`.

8. **Ordered stores, not guaranteed visibility:** In the explicitly defined abstract access-trace model, publication base plus `0x20` has four ordered word stores: `0x20`, `0x2F`, the intervening byte-read result, and `0x20`. A single final store is not trace-equivalent. Whether a runtime observer can distinguish all four stores, or an implementation may combine them, remains documented_only.

9. **Final fixed-byte value:** If adjusted value is 2 and arithmetic is supported, fixed byte `0x3A` receives `0x40`, after the first two final-phase word captures and before the next two.

10. **Fresh fixed-byte reads:** Under the abstract memory interpretation, a calculated store that changes `0x67` can change a subsequent copy from that location. Reusing the original fixed-word capture is insufficient.

11. **Interrupted execution:** If a dependency does not return, later interactions are not required. No particular exception, status, recovery, or restoration is established.

## Uncertainties

The following remain documented_only:

- Application intent and semantic object types.
- Indirect-target identities, complete signatures, and secondary-result guarantees.
- Correspondence between the representation and executable behavior.
- Signed overflow, negative or excessive-count shifts, and address wrapping.
- Endianness, alignment rules, permissions, and memory mapping.
- Exact repeated-publication count as an acceptance requirement.
- Ownership, lifetime, concurrency, and device interaction.
- Volatile semantics, physical access visibility, and permissible optimization.
- Error signaling and final-transfer return behavior.

These limits prevent treating this document as a complete executable replacement specification.

## Evidence

The sole evidence is the supplied static representation. Prior prose, reviewer feedback, comments, and diagnostic annotations are not independent proof.

The representation supports the mixed-width accesses, invocation-result transformations, one-bit invocation fields, eight-bit shift-count mask, ordered publication-store expressions, interleaved captures and writes, separate fixed-byte rereads, and final four-argument transfer.

Arithmetic consequences are conditional where machine interpretation is required. Represented access order is separated from physical observability. No external implementation or dependency inspection, runtime execution, or runtime testing was performed.
