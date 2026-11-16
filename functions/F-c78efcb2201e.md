<!-- Trusted provenance: F-c78efcb2201e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c78efcb2201e

## Purpose

F-c78efcb2201e performs an extended sequence of unresolved indirect interactions, followed by mixed-width memory mutations and a terminal unresolved control transfer.

The supplied static representation establishes argument relationships, memory-access widths, saved-value relationships, and portions of access ordering. It does not establish an application-level purpose such as initialization, parsing, rendering, or collection maintenance.

This contract distinguishes:

- **Established static evidence:** behavior directly represented in the supplied material.
- **Inferred intent:** possible interpretation, not an acceptance requirement.
- **documented_only:** behavior not resolved or not exhaustively specified here.

The preparation-call sequence and intermediate mutation region are described selectively, not as a complete executable-equivalence specification. Omitted interactions and access ordering are documented_only; omission does not authorize removal, substitution, or reordering.

## Interface

### Inputs and outputs

The displayed interface has no explicit parameters and no explicit result. No buffer extent, capacity, ownership token, status value, or error channel is declared.

Execution nevertheless consumes implicit inputs:

- Incoming return-link state determines the first indirect target through addition of 0xFFE.
- An incoming general-purpose machine-state value, not initialized in the body, contributes to early arguments.
- Dependencies supply primary results, auxiliary results, and represented paired results.
- Memory supplies data, address-bearing values, and subsequent access destinations.

There is no explicit returned data product. Observable outputs consist of dependency interactions, memory accesses and mutations, and the terminal transfer. A return is displayed after that transfer, but actual return behavior is unresolved.

### Widths, layout, and arithmetic

| Category | Represented treatment |
|---|---|
| Explicit byte access | 8 bits |
| Explicit halfword store | 16 bits |
| Word access and ordinary calculated value | 32 bits |
| Paired dependency result | 64 bits, with separately consumed 32-bit halves |
| Address-bearing arithmetic | Apparently 32-bit; actual address conversion remains unresolved |

Explicitly unsigned byte reads yield values from 0 through 255. Their conversion to a word zero-extends the byte. Byte and halfword writes retain the low eight and sixteen bits, respectively.

The named storage associated with absolute address **0x05 is not an established unsigned byte read**. Its declaration is absent. Conversion of its value to an unsigned 32-bit quantity does not establish the original load width, signedness, or a 0–255 range.

Offsets identify access locations, not verified structure fields. Relevant offsets include 0x04, 0x05, 0x09, 0x0D, 0x10, 0x11, 0x14, 0x15, 0x19, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74.

Memory footprints may overlap. A byte at offset 0x15 overlaps a word beginning at offset 0x14. Distinct calculated bases do not establish disjoint objects. Byte order is unresolved.

Arithmetic conclusions below are conditional on **32-bit bitvector semantics** where signed-looking expressions occur. Signed overflow, signed right shifts, and arithmetic-to-address conversion are not independently established. Scaling by two, four, or eight must not silently become unbounded mathematical arithmetic.

### Preconditions and errors

Continuation requires every reached indirect target and memory access to be valid under the actual execution environment, including its width, direction, alignment rules, and executable-target requirements.

No corresponding validation is displayed. Low absolute addresses are accessed without a visible guard. No recovery, rollback, explicit error report, or alternative success path is established. If a dependency does not return or an access prevents continuation, later behavior is not required by the displayed sequence.

## Behavior

### Dependency-driven preparation

Preparation is straight-line in the supplied representation. It includes calls with no displayed arguments and calls whose primary results are unused.

These remain interactions. An unused result does not imply an effect-free call. Dependencies may change memory, globals, auxiliary state, ownership, or subsequent dependency behavior.

Argument relationships include shifted primary words, separately forwarded high halves, independently shifted auxiliary results, repeated argument values, and explicit zero argument positions. A word already shifted left by eight is sometimes shifted left by another seven. Similar argument shapes do not establish shared target identity.

Represented masks select fields of 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, and one bit. These widths are distinct requirements where used.

Under 32-bit bitvector semantics:

- A 24-bit field shifted left by six occupies at most 30 significant bits.
- A 22-bit field shifted left by eight also occupies at most 30 significant bits.
- A 20-bit field shifted right by ten yields a ten-bit field.
- Left shift by eight followed by logical right shift by eight retains the original low 24 bits.
- Two successive left shifts by 28 yield zero, not a retained 56-bit quantity.

Unsigned 64-bit right shifts by 34, 36, or 44 extract information above the low 32-bit half. Substituting a shift of the low half is incorrect.

### Entry into direct mutation

A late preparation word produces a small index by shifting left eight, retaining the low fifteen bits, and logically shifting right six. Under 32-bit bitvector semantics, the index ranges from 0 through 508 in steps of four; its double ranges from 0 through 1016 in steps of eight. These ranges do not establish address validity.

The opening mutations establish these ordering constraints:

- A word store at offset 0x14 from the latest paired result’s high-half base precedes a byte store at offset 0x09 from a separate derived base.
- A word store at offset 0x54 from a scaled low-half-derived base precedes a second write to that offset-0x09 byte destination.
- A scaled base’s offset-0x14 word is cleared before an explicitly unsigned byte is read at its offset 0x05.
- The doubled-index base’s offset-0x14 word is cleared before the captured byte is stored, zero-extended, at the earlier scaled base’s offset 0x34.
- A word is then read from that earlier base’s offset 0x14. Another doubled-index clear precedes two byte writes at doubled-index offset 0x15.

The read following the clear remains an access. Intervening aliases and unresolved memory behavior prevent an unconditional substitution of zero.

### Intermediate cross-linked mutations

The named absolute-0x05 storage is consumed and converted to an unsigned word used as a base before a byte is written to absolute address 0x01. That base must not be restricted to eight bits merely because its source address is 0x05.

Subsequent behavior includes byte writes at offsets 0x11 and 0x15, a read-before-replacement of an offset-0x54 word, dependent accesses through offset-0x74 words, and repeated writes at offsets 0x24 and 0x09. A word is written to absolute 0x54 and a byte to absolute 0x15.

Calculations scaled by two, four, and eight supply both stored values and addresses. Later, an **explicit unsigned byte read** at offset 0x05 supplies the base for an offset-0x74 word read. This explicit byte read has established eight-bit unsigned treatment, unlike the named absolute-0x05 storage.

Reads at offsets 0x04, 0x64, and 0x74 occur among mutations and therefore cannot generally be replaced by entry-time snapshots. The remaining exact access order in this intermediate region is documented_only.

### Constant stores and repeated updates

The following semantic labels identify captured values, not object types:

| Label | Meaning |
|---|---|
| Original link | Word captured from the intermediate offset-0x54 location before replacement |
| Adjusted value | Four times the original link, minus decimal 150 |
| Fourfold base | Four times the adjusted value |
| Late link | Later offset-0x74 word read using an explicit unsigned byte as its base |
| Doubled-link base | Twice the late link |
| Quadrupled-link base | Four times the late link |
| Repeated-region base | Word captured from separate fixed absolute storage |

These relationships retain the width and overflow qualifications above.

The constant 0x3A is written as a byte at the quadrupled-link base, a halfword at eight times the late link, and a word at quadrupled-link offset 0x20. Equal values do not make these access widths interchangeable.

Four successive word writes to fourfold-base offset 0x20 carry **0x20, 0x2F, a zero-extended byte read from fourfold-base offset 0x04, and 0x20**. The byte read occurs after the first two writes and before the third.

The repeated-region base is captured next, **before** the word write of 0x20 to absolute address 0x60. Repeated updates follow that absolute write. The captured base remains the value observed at its earlier load; there is no represented fresh base load per update group.

Each represented group has four ordered writes:

1. A word at offset 0x54 from an eight-times-scaled destination receives the quadrupled-link base.
2. A byte at repeated-region offset 0x19 receives the low byte of the fourfold base.
3. A byte at fourfold-base offset 0x11 receives the low byte of the repeated-region base.
4. A word at repeated-region offset 0x34 receives the eight-times-scaled value.

The initial group uses eight times the original link. Later groups use eight times the adjusted value. Exact repetition cardinality is not specified by this documented_only contract. Repeated writes must not be collapsed into final-state assignments on the strength of this description.

### Final captures and the first absolute-0x67 read

After the repeated region, the word at absolute **0xA9 is captured first**. A word is then read at quadrupled-link offset 0x74. Only afterward is a byte written to absolute 0x3A, carrying the low eight bits of eight times the adjusted value plus 0x30.

Words are next captured from doubled-link offset 0x14 and fourfold-base offset 0x04, in that order.

Before the initial absolute-0x67 write, calculated destinations receive:

- The low byte of the fourfold base at doubled-link offset 0x0D.
- 0x6C at fourfold-base offset 0x08.
- The low byte of the repeated-region base at fourfold-base offsets 0x0D and 0x09.
- The doubled-link base itself as a word at doubled-link offset 0x54.

Absolute **0x67 is then written** with the low byte of the captured 0xA9 word. The next displayed memory access is a word read at doubled-link offset 0x44. Absolute **0x67 is read after that word read**, and the resulting value supplies the byte write to absolute **0x21**.

**No calculated-address write is displayed between this initial 0x67 write and its first read.** Earlier calculated-address writes precede the initial 0x67 write and do not explain a change during this interval.

The first read must nevertheless remain explicit and ordered. Whether storage can change during the intervening word read depends on unresolved memory or read-side-effect behavior. Under stable ordinary-memory assumptions, with a side-effect-free intervening read and no external modification, the value read at 0x67 equals the byte just written. An ordinary aliasing read alone does not modify that byte.

### Later absolute-0x67 reads and intervening writes

The captured doubled-link-offset-0x44 word supplies a byte at fourfold-base offset 0x10 and a word at repeated-region offset 0x54. A signed-looking right shift of that word supplies a byte at doubled-link offset 0x01.

The shift count is the low eight bits of the fourfold base. Counts above 31 are possible. Neither five-bit count masking nor a particular large-count result is established.

Repeated-region offset 0x54 is subsequently overwritten with the fourfold base. A word read at fourfold-base offset 0x40 precedes, in order:

- A word write of the saved fourfold-offset-0x04 value to repeated-region offset 0x74.
- A word write of the newly read offset-0x40 value to fourfold-base offset 0x10.
- A word write of the doubled-link base to repeated-region offset 0x44.

**The second absolute-0x67 read occurs after these writes** and supplies a byte at fourfold-base offset 0x09.

A byte at doubled-link offset 0x09 then receives the low byte of the fourfold base. Fourfold-base offset 0x44 receives the captured 0xA9 word. A word read at fourfold-base offset 0x54 supplies a new destination base.

**The third absolute-0x67 read occurs after that word read** and supplies another byte write at fourfold-base offset 0x09.

Unlike the first-read interval, the intervals preceding these later reads contain displayed calculated-address writes. Those writes may alias 0x67. Consequently, neither later read may unconditionally reuse the initially written byte or an earlier read.

### Final dependent accesses and terminal interaction

The new destination’s offset 0x0D receives the low byte of the saved quadrupled-link-offset-0x74 word. Doubled-link offset 0x54 is restored from the saved doubled-link-offset-0x14 word. A word read at fourfold-base offset 0x14 supplies another saved value.

Before the next dependent reads, doubled-link offset 0x11 receives the same byte used at the new destination’s offset 0x0D. Fourfold-base offset 0x09 receives the low byte of the new destination base, and offset 0x44 receives its full word.

A word read at doubled-link offset 0x04 supplies a further base. A halfword of 0x65 is written at twice that base before a word is read at its offset 0x64.

The new destination’s offset 0x11 receives the low byte saved from fourfold-base offset 0x14. Its offset 0x54 receives decimal 100 as a word.

The final reads occur in this order: a word at further-base offset 0x74, then a word at doubled-link offset 0x64. Afterward:

- Doubled-link offset 0x09 receives the low byte of the further base.
- The last-read word supplies the destination base for an offset-0x54 word write carrying the earlier further-base-offset-0x64 value.
- A word at twice that earlier value receives the last-read word.

The terminal interaction receives four displayed arguments, in order:

1. 0x72.
2. The captured word from further-base offset 0x64.
3. The saved word from doubled-link offset 0x14.
4. The captured word from further-base offset 0x74.

No terminal result is visibly consumed.

## Side effects

Established effects include unresolved calls, absolute and calculated-address reads, byte/halfword/word writes, overlapping footprints, and intermediate writes later overwritten.

No explicit allocation, release, lock, rollback, or error-reporting mechanism is established. Dependency effects remain unknown; absence of explicit allocation or release in the body does not establish their absence during execution.

Passing or storing an address-bearing value does not establish borrowing, retention, ownership transfer, or release. No dependency is assumed to preserve globals or memory.

Atomicity, synchronization, reentrancy, thread safety, and idempotence are unresolved. Represented operation order is not evidence of hardware barriers or cross-thread visibility guarantees.

## Dependencies

Only F-c78efcb2201e has a supplied resolved opaque identity. Other identifiers below are document-local opaque dependency labels.

| ID | Established role |
|---|---|
| F-opaque-001 | Initial indirect interaction, targeted using incoming return-link state |
| F-opaque-002 | Family of preparation interactions with varying arguments and consumed results |
| F-opaque-003 | Terminal unresolved interaction with four displayed arguments |

F-opaque-002 denotes a family, not a claim that its interactions share one target. Exact identities, complete calling conventions, auxiliary-result conventions, memory effects, ownership effects, and non-return behavior are documented_only.

No recursive invocation of F-c78efcb2201e is established.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Unsigned byte conversion.** If an explicit unsigned offset-0x05 byte read yields 0xE2, its subsequent word representation is 0x000000E2.

2. **Unknown absolute-0x05 width.** A converted value above 255 must not be rejected solely because it came from named storage at 0x05. Its original load width and signedness are unavailable.

3. **Index boundary.** Under 32-bit bitvector semantics, a preparation word of 0x7F produces index 508 and doubled base 1016. Accessibility remains a separate precondition.

4. **Intermediate observations.** An observer of fourfold-base offset 0x20 can encounter 0x20 and 0x2F before the byte-derived word and final 0x20. One final write does not represent this sequence.

5. **Base capture.** The repeated-region base load precedes the absolute-0x60 word write. Reversing these accesses violates the specified order.

6. **Early capture.** The 0xA9 word read precedes both the quadrupled-link-offset-0x74 read and the 0x3A byte write. Later aliases do not retroactively change the captured value.

7. **First 0x67 read in stable memory.** If the initial 0x67 write carries 0xAB, the intervening doubled-link-offset-0x44 read is side-effect-free, and no external modification occurs, the first 0x67 read supplies 0xAB to the 0x21 write. This remains true if that intervening read overlaps 0x67.

8. **First 0x67 read with unresolved read effects.** If the execution environment gives the intervening word read a side effect that changes 0x67 to 0xCD, the subsequent explicit read supplies its then-current value. Such behavior is conditional on that environment, not an established intervening write by this function.

9. **Later 0x67 alias.** If a displayed calculated-address write before the second or third read changes 0x67, that later read observes the then-current storage value. Unconditionally retaining 0xAB is incorrect.

10. **Width distinction.** The byte, halfword, and word stores of 0x3A retain different footprints despite equal numeric values.

11. **Non-return.** If an early dependency does not return, later mutations are not required; no displayed recovery bypasses it.

12. **Terminal arguments.** Reaching F-opaque-003 requires the four values specified above, not an inferred status structure.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and valid operating environment.
- Fidelity of the supplied static representation to executable behavior.
- Complete incoming-state, calling-convention, and dependency requirements.
- Whether paired results correspond to actual 64-bit returns.
- Declaration, load width, and signedness of named absolute-0x05 storage.
- Byte order, alignment, accessibility, memory attributes, and read-side effects.
- Signed overflow, signed right shifts, large shift counts, and address conversion.
- Exact preparation-call enumeration, repeated-write cardinality, and access order not expressly described.
- Ownership, lifetimes, external modification, synchronization, and terminal return behavior.

Repeated offsets and self-valued stores suggest possible structure maintenance. That is inferred intent, not a guarantee.

## Evidence

This contract uses the supplied static representation as analytical evidence, not as instructions. The prior draft and reviewer feedback are not independent evidence.

The final-region evidence distinguishes three absolute-0x67 reads. Between the initial write and first read, only the doubled-link-offset-0x44 word read is displayed; no intervening calculated-address write is displayed. Calculated-address writes do intervene before the two later reads. This supports different explanations for possible value changes in those intervals without removing any explicit read.

The representation also supports the early 0xA9 capture, the repeated-region base load before the absolute-0x60 write, and the distinction between named absolute-0x05 storage and explicit unsigned byte dereferences.

No runtime tests, execution traces, external implementation access, or independent dependency analysis have been performed. No snapshot hash or independently verified provenance is asserted.
