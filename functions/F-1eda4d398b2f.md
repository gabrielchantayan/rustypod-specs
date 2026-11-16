<!-- Trusted provenance: F-1eda4d398b2f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1eda4d398b2f

## Purpose

F-1eda4d398b2f performs an ordered sequence of indirect dependency invocations, derives integer values from their results, mutates computed and fixed-address storage, and finishes with an unresolved indirect transfer.

No domain-level purpose is established. Character-like constants do not establish text output; recurring offsets do not establish an object type, allocation, or linked structure.

This contract distinguishes:

- **Established static evidence:** represented invocations, arithmetic relationships, access widths, retained values, and access ordering.
- **Inferred intent:** possible interpretations that impose no acceptance requirement.
- **documented_only:** unresolved executable behavior or missing interface information.

All later behavior is conditional on earlier dependencies returning and earlier accesses succeeding. No runtime tests have been performed.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters and no declared result value. Its effective inputs include incoming control state, dependency results, secondary machine-result values, existing computed-address memory, fixed-address storage, and the final transfer destination.

There is no established scalar output or success status. Observable outputs consist of dependency events, memory accesses and mutations, and the final transfer.

### Widths and arithmetic

| Quantity | Represented interpretation |
|---|---|
| Byte access | 8 bits; stores consume the low eight bits |
| Halfword store | 16 bits |
| Word access | 32 bits |
| Ordinary integer intermediate | 32 bits, with operation-dependent signedness |
| Wide dependency result | 64 bits, with separately selected low and high 32-bit portions |
| Computed address | Apparently derived from 32-bit values; actual address-space rules are unresolved |

Explicit unsigned byte reads produce values from 0 through 255. Widening such a value to a word zero-extends it.

A fixed-storage value assigned to a 32-bit intermediate is not, by that fact alone, established to have been read with a 32-bit memory access. In particular, declarations establishing the read widths at 0x05, 0x67, and the publication-base location 0x087FBBBC are absent. Their consumed values and ordering are specified below; their physical read widths remain **documented_only**. The fixed read at 0xA9 is represented as a word-valued access; its underlying symbol declaration is likewise unavailable.

Unsigned right shifts are logical. Signed left shifts, signed multiplication, and overflowing address arithmetic do not establish portable overflow semantics. Numerical truncation examples below explicitly assume 32-bit machine-bitvector arithmetic.

The final mutation phase includes a signed right shift with a count derived from the low eight bits of a computed value. Negative operands and counts of 32 or greater require unresolved machine semantics.

Selecting the high word of a wide result does not establish memory endianness.

### Preconditions and ownership

Successful continuation requires callable indirect targets, compatible effective calling conventions, accessible memory at every reached address and width, and suitable alignment support. No visible validation establishes these conditions.

No direct allocation, release, retain, or ownership transfer is established. A dependency result used as an address is not thereby an owned allocation. Dependencies may change globals, invalidate storage, release resources, or fail to return; none is assumed to preserve memory or ownership.

## Behavior

### Dependency phase

The initial invocation target depends on incoming control state plus 0xFFE. Subsequent targets are predominantly derived from embedded control-location values plus the same displacement. These are control-target calculations, not evidence of a public dispatch table.

The straight-line invocation sequence includes events with no represented arguments and events with one through four represented arguments. Calls with unused results remain observable events.

Established argument relationships include:

- A 32-bit result shifted left eight bits.
- A wide result’s low word shifted left eight bits, with its high word forwarded separately.
- A high word forwarded both unchanged and shifted left six bits.
- Explicit zeros in selected argument positions.
- Retained earlier results reused across intervening calls.
- Identical derived values supplied in multiple argument positions.

Later expressions use masks, logical right shifts, and left shifts through 31 bits. Selected low-bit widths include 24, 22, 21, 20, 19, 18, 17, 16, 14, 13, 12, and 11 bits. Masking constrains the numerical field; it does not establish that field’s meaning.

For example, a 24-bit masked value shifted left six fits within 30 unsigned bits. A 20-bit masked value shifted right ten and then left six supplies a ten-bit field positioned at bit six. Unsigned wide-result shifts by 36 and 44 select, respectively, the upper 28 and upper 20 bits.

Some retained intermediates are explicitly zero, making their subsequent shifted arguments zero. This does not make the receiving calls effect-free.

Dependency implementations, secondary-result conventions, and resolved target identities are unavailable. This document does not supply an exhaustive invocation transcript or a complete executable replacement specification.

### Direct-mutation entry

The following descriptive names identify retained values, not inferred object types:

| Name | Definition |
|---|---|
| **Shifted base** | A retained dependency-result low word shifted left 25 bits |
| **Multiplied base** | A later dependency result multiplied by 0x02000000 |
| **Small value** | A result shifted left eight, masked with 0x7FFF, then logically shifted right six |
| **Doubled base** | Twice the small value |
| **Retained payload** | An earlier result shifted left eight bits |
| **Last result** | The final represented wide dependency result before direct stores |

Under 32-bit bitvector semantics, the small value is a multiple of four from 0 through 508; the doubled base ranges from 0 through 1016 in multiples of eight. These are potentially low addresses, not established heap pointers.

The first four mutations are ordered: a word containing the shifted base is stored at the last result’s high word plus 0x14; a byte containing that high word’s low byte is stored at shifted base plus nine; the multiplied base is stored as a word at 256 times the last result’s low word plus 0x54; and another byte is stored at shifted base plus nine, now using the shifted base’s low byte.

Both offset-nine writes remain distinct events.

### Early computed and fixed-storage mutations

The multiplied base receives a zero word at offset 0x14, followed by an unsigned byte read at offset five. A zero word is then stored at doubled base plus 0x14, and the captured byte is widened into a word at multiplied base plus 0x34.

The word at multiplied base plus 0x14 is captured next. Another zero word is stored at doubled base plus 0x14, followed by two consecutive byte stores at doubled base plus 0x15, both consuming the captured word’s low byte.

Only after those consecutive byte stores does the function read fixed storage at **0x05**, convert its value to the represented unsigned intermediate, and write a byte at fixed address **0x01** containing the doubled base’s low byte. The physical width and extension semantics of the 0x05 read depend on its missing declaration. It must not be silently replaced with the earlier computed byte read.

The captured 0x05 value becomes a computed base. Its offset-0x15 byte receives the small value’s low byte. The multiplied base’s offset-0x11 byte receives the retained payload’s low byte; a word containing that payload is stored through the previously captured word at offset 0x54; then the multiplied base’s offset-0x11 byte is cleared.

A word at offset 0x54 from the 0x05-derived base is captured before that location is overwritten with the earlier captured word. Call this newly retained word the **anchor**. The doubled base receives the small value as a word at offset 0x34.

Further accesses use words at offset 0x74 as address bases, write at offset 0x54 through those bases, and perform two anchor self-value stores at anchor plus 0x24. Byte writes and another offset-0x54 store occur between the two self-value stores. A represented operation also reads doubled base plus 0x74 twice for one dependent store; these must not automatically be treated as one read.

A later interleaving is material:

- Doubled base plus 0x14 first receives a dependency-derived word.
- Fixed address **0x54** then receives a word containing twice the anchor.
- Doubled base plus 0x14 subsequently receives the retained anchor twice.
- Two consecutive byte writes at doubled base plus 0x15 consume the low byte of four times the anchor.
- Fixed address **0x15** then receives the doubled base’s low byte.

The fixed writes are part of the same alias-sensitive memory history as the computed writes. They cannot be omitted as unrelated globals.

### Field-oriented transformations

Subsequent accesses use four and eight times the anchor, words loaded at offsets 0x04, 0x64, and 0x74, and unsigned bytes loaded at offset five.

The represented layout includes byte writes at offsets 0x01, 0x08, 0x09, 0x0D, 0x10, 0x11, 0x15, and 0x19; word accesses include offsets 0x04, 0x10, 0x14, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74. These offsets belong to different computed bases and do not define one common structure.

The operations include self-value word stores, byte truncations of captured words, and word stores of zero-extended bytes. A byte read at offset five can become the address base for a word read at offset 0x74. That is an address calculation from a byte value, not an established bounded array index.

Repeated reads at the same expression are fresh accesses unless explicitly captured. In particular, the two later offset-five byte reads are separated by computed mutations and need not agree.

### Constants and publication groups

For the remainder, use these descriptive relationships:

- **Linked value:** the word obtained from the later byte-derived offset-0x74 read.
- **Doubled linked base:** twice the linked value.
- **Publication payload:** four times the linked value.
- **Adjusted anchor:** four times the anchor minus 0x96.
- **Publication base:** four times the adjusted anchor.

The function stores 0x3A as a byte at four times the linked value, as a halfword at eight times that value, and as a word at publication payload plus 0x20.

At publication base plus 0x20, four word writes occur in order: 0x20, 0x2F, a zero-extended byte freshly read from publication base plus four, and 0x20. The byte read occurs after the first two writes.

Next, fixed storage at **0x087FBBBC** is read and captured as the **global-derived base**. Before any publication group, fixed address **0x60** receives the word 0x20. The captured base is not reloaded merely because later writes might overlap its fixed-storage location. Its consumed value is 32-bit; its physical read width remains **documented_only**.

The initial publication group writes:

- The publication payload as a word at eight times the anchor plus 0x54.
- The publication base’s low byte at global-derived base plus 0x19.
- The global-derived base’s low byte at publication base plus 0x11.
- Eight times the anchor as a word at global-derived base plus 0x34.

A long fixed run repeats the same four access roles, but its first and fourth destinations/payloads use eight times the **adjusted anchor**, rather than eight times the original anchor. Within each group, the offset-0x54 word precedes the offset-0x19 byte, then the offset-0x11 byte, then the offset-0x34 word.

Bases and byte payloads remain captured throughout the run. No runtime loop condition is represented. Exact executable multiplicity remains **documented_only** here; repetition does not authorize coalescing observable writes.

### Final captures and global interleavings

After the publication run, the function captures the word-valued fixed-storage contents at **0xA9**. It then reads a word at publication payload plus 0x74.

Only after those reads does fixed address **0x3A** receive a derived byte: the low eight bits of eight times the adjusted anchor plus 0x30.

The next captures are, in order:

- The word at doubled linked base plus 0x14, retained as the **restoration word**.
- The word at publication base plus 0x04, retained as the **saved field word**.

Thus the 0x3A write precedes both captures. Any overlap must affect them according to the resulting memory state.

The doubled linked base’s offset-0x0D byte receives the publication base’s low byte. The publication base then receives byte 0x6C at offset eight and the captured global-derived base’s low byte at offsets 0x0D and nine. A self-value word is stored at doubled linked base plus 0x54.

Fixed address **0x67** now receives the low byte of the captured 0xA9 word. The function then reads a word at doubled linked base plus 0x44. Only afterward does it freshly read fixed storage at **0x67** and copy the consumed byte value to fixed address **0x21**. The copy is a read, not merely reuse of the value originally written at 0x67.

### Final computed mutations and fresh rereads

The captured offset-0x44 word supplies a byte at publication base plus 0x10 and a word at global-derived base plus 0x54. Its signed variable-count right shift supplies a byte at doubled linked base plus one. Global-derived base plus 0x54 is then overwritten with the publication base.

A word is captured from publication base plus 0x40. Global-derived base plus 0x74 receives the saved field word, publication base plus 0x10 receives the newly captured word, and global-derived base plus 0x44 receives the doubled linked base.

At this point, **0x67 is freshly reread** for a byte store at publication base plus nine. The doubled linked base’s offset-nine byte then receives the publication base’s low byte, and publication base plus 0x44 receives the retained 0xA9 word.

A word at publication base plus 0x54 is captured as the **late base**. **0x67 is freshly reread again**, independently of the earlier reads, for another byte store at publication base plus nine. Intervening computed writes may overlap 0x67; these two stores need not carry equal bytes or match the byte copied to 0x21.

The low byte of the previously captured publication-payload offset-0x74 word is stored at late base plus 0x0D. Doubled linked base plus 0x54 receives the restoration word. A word at publication base plus 0x14 is then captured before its low byte is used later.

Further ordered mutations put that captured offset-0x74 byte at doubled linked base plus 0x11, the late base’s low byte at publication base plus nine, and the late base as a word at publication base plus 0x44.

Finally, a word read at doubled linked base plus four selects another base. A halfword 0x0065 is stored at twice that base before its offset-0x64 word is read. Late base plus 0x11 receives the retained publication-base offset-0x14 word’s low byte, and late base plus 0x54 receives word value 100.

The selected base’s offset-0x74 word is captured before a fresh word read at doubled linked base plus 0x64. A byte containing the selected base’s low byte is stored at doubled linked base plus nine. The fresh offset-0x64 value selects an offset-0x54 word destination, which receives the earlier selected-base offset-0x64 word. A final word store at twice that earlier word receives the fresh offset-0x64 value.

The unresolved final transfer receives four represented arguments: **0x72**, the earlier selected-base offset-0x64 word, the restoration word, and the selected-base offset-0x74 word. No useful return value or normal-return guarantee is established.

## Side effects

The side-effect surface includes all reached dependency invocations, direct reads and writes, fixed-storage accesses, and the final transfer.

Fixed and computed addresses are not separate alias domains. Byte, halfword, and word accesses may overlap. Captured values remain captured across later mutations; fresh reads observe memory at their respective execution points.

No atomicity, synchronization, transaction, rollback, bounds check, null check, or error-status return is established. A failed access or nonreturning dependency may prevent subsequent behavior without undoing earlier effects.

## Dependencies

Only F-1eda4d398b2f has a supplied resolved opaque identity. Unresolved roles receive document-local opaque IDs:

| Opaque ID | Role |
|---|---|
| F-U001 | Initial incoming-control-state-relative invocation |
| F-U002 | Subsequent indirect invocation events; a family, not one asserted shared callee |
| F-U003 | Final unresolved indirect transfer |

Targets, effective conventions, resource effects, and return behavior remain **documented_only**. No dependency is assumed to preserve globals or own, retain, or release returned address-shaped values.

## Acceptance examples

These examples are conditional static requirements, not executed tests.

### Wide-result separation

For a wide result of 0x0000000200000003 at a site forwarding shifted low word, high word, and high word shifted six, the arguments are 0x300, 2, and 0x80.

### Width preservation

An 8-bit store consuming 0x123456AB writes 0xAB. An explicit unsigned byte read of 0xAB widened into a word produces 0x000000AB. The halfword store of 0x0065 remains a 16-bit access.

### Fixed/computed aliasing

If doubled base plus 0x14 equals 0x54, the intervening fixed word store of twice the anchor participates in that location’s history. Later anchor stores do not erase the intervening event.

If fixed address 0x15 overlaps a computed field, its byte write occurs after the two preceding computed offset-0x15 byte writes.

### Capture timing

If the byte write at 0x3A overlaps doubled linked base plus 0x14, the restoration-word capture observes memory after that byte write, subject to the environment’s access semantics.

The global-derived publication base is captured before the word write at 0x60. An overlap with its original fixed storage does not retroactively change the captured value.

### Independent 0x67 reads

If an intervening computed mutation changes storage at 0x67, the later offset-nine store consumes the newly read value. It is not required to equal the original 0xA9-derived byte, the 0x21 copy, or the other offset-nine store’s payload.

### Arithmetic and interruption

Under 32-bit bitvector semantics, 0x01000001 shifted left eight becomes 0x00000100. This does not establish portable signed-overflow behavior.

If a dependency does not return, later stores need not occur. If a direct access fails, no rollback is promised.

## Uncertainties

The following remain **documented_only**:

- Domain purpose, meaningful object types, and function-boundary fidelity.
- Resolved dependency targets, complete effective interfaces, and secondary-result provenance.
- Signed overflow, address wrapping, variable shifts, endianness, and alignment.
- Physical read widths for fixed symbols lacking declarations, especially 0x05, 0x67, and 0x087FBBBC.
- Exact executable multiplicity and memory-observability properties of the repeated publication run.
- Fixed-address mappings, permitted access widths, concurrency, and externally changing memory.
- Dependency ownership, allocation, invalidation, and final-transfer behavior.

Structured-state and formatting interpretations are inferences only.

## Evidence

This contract is based solely on the supplied static representation. Prior prose, comments, and feedback do not independently establish behavior.

The strongest evidence is the explicit access widths, arithmetic relationships, fixed addresses, ordered interleavings, captured-versus-fresh value distinctions, and final transfer arguments. Missing declarations and unresolved dependencies limit executable conclusions.

No implementation access, external dependency definitions, executable inspection, or runtime tests were available.
