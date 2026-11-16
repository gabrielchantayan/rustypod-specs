<!-- Trusted provenance: F-a026ceea6e4b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a026ceea6e4b

## Purpose

F-a026ceea6e4b performs an ordered sequence of unresolved indirect invocations, derives values from call results and residual machine state, modifies memory through computed addresses and named globals, and reaches a final unresolved indirect transfer.

The supplied representation does not establish an application-level purpose. It does not establish object construction, collection management, text output, or protocol processing. Constants resembling character values do not establish formatting behavior.

This contract distinguishes represented operations from inferred intent and unresolved execution semantics. It describes static evidence only; no runtime tests have been performed.

## Interface

### Effective inputs and outputs

The displayed interface has no explicit parameters and no explicit return value. Effective inputs nevertheless include:

- An incoming continuation-related value used to derive the first indirect target.
- An incoming 32-bit machine value used in early call arguments.
- Primary and paired results of unresolved calls.
- Residual secondary-register values consumed after calls.
- Memory and global values obtained during execution.
- The unresolved final-transfer target.

Observable outputs include dependency invocations, memory updates, and a final transfer with four represented argument words. Its first argument is 0x72. No explicit success code, failure code, or returned object is established.

A return is displayed after the final transfer. Whether that transfer returns, and whether a normal return is reachable in the actual execution environment, remain **documented_only**.

### Widths and arithmetic

| Category | Established treatment |
|---|---|
| Explicit byte-pointer access | 8 bits |
| Explicit halfword store | 16 bits |
| Explicit word access | 32 bits |
| Ordinary represented scalar quantities | 32 bits |
| Paired result | 64 bits, consumed as low and high 32-bit words |
| Narrowed byte value | Low eight bits |
| Explicit unsigned-byte promotion | Zero-extension to 32 bits |
| Address material | Derived from represented 32-bit quantities |
| Named global at address 0x05 | Value converted to unsigned 32 bits; source storage width and signedness unresolved |

**The conversion of the address-0x05 global is not evidence of an unsigned-byte load.** Its declaration is absent. The captured result is an unsigned 32-bit value, but no restriction to 0–255 is established. This differs from explicitly typed byte-pointer reads elsewhere in the memory phase.

Arithmetic examples use low-32-bit truncation for word additions, subtractions, multiplications, and left shifts. This is a conditional machine-word model, not a guarantee of portable signed-overflow semantics. Signed right shifts and excessive shift counts require separate execution semantics.

Offsets identify individual accesses, not proven fields of a record. Object sizes, pointer provenance, alignment guarantees, and endianness are not established.

### Preconditions and ownership

Every reached indirect target must be callable under the actual calling convention. Every reached memory access must be supported at its address and width. No displayed validation enforces these conditions.

Low addresses and potentially unaligned addresses occur. Their validity depends on the execution environment.

No allocation, deallocation, ownership transfer, retention, or lifetime convention is established. Unknown dependencies must not be assumed to preserve globals, preserve all machine state, or leave memory ownership unchanged.

## Behavior

### Ordered invocation phase

The represented body has a preliminary indirect-call phase, a direct memory-mutation phase, and a final unresolved transfer. No explicit input-dependent branch, loop condition, retry, rollback, or early error return is displayed. A dependency may introduce its own control flow or fail to return.

The first target is derived from the incoming continuation-related value plus 0xFFE. Later targets generally derive from embedded code-location values with the same displacement. Actual dependency identities remain unresolved.

An early paired result contributes its low word shifted left by eight and its unchanged high word to a subsequent invocation. The same invocation receives the incoming machine value shifted left by six and the paired low word shifted left by fourteen.

Later invocations repeatedly consume:

- Returned words shifted left by eight or other fixed counts.
- Low and high halves of paired results.
- Explicit zeros.
- Residual secondary-register values.
- The same transformed value in multiple argument positions.
- Saved results retained across intervening calls.

These are ordered invocation events. Equal arguments, zero arguments, or unused primary results do not make calls removable.

Field transformations include selecting bits 10–19 and shifting the selected ten-bit field left by five or six; masking a low word to 24 bits before shifting left by six; and masking a high word to 22 bits before shifting left by seven or eight.

Unsigned shifts of paired results by 34, 36, and 44 extract the high word shifted right by two, four, and twelve, respectively. Some extractions use an older saved paired result rather than the most recent paired return.

The complete invocation enumeration, exact invocation count, and full mapping of implicit machine state remain **documented_only** here. The recurring patterns do not authorize removal, substitution, or reordering of calls.

### Retained address roles

The following names describe value dependencies without asserting data structures:

| Role | Definition |
|---|---|
| Shifted-call base | A retained paired-result low word shifted left by 25 |
| Saved call word | Another retained call result shifted left by eight |
| Scale seed | A later retained call result |
| Large scaled base | Scale seed multiplied by 0x02000000 |
| Small precursor | A later result shifted left by eight, masked to 15 bits, then shifted right by six |
| Small base | Twice the small precursor |
| Converted-global base | The address-0x05 global’s value after conversion to unsigned 32 bits |
| Link word | Word loaded at converted-global-base-plus-0x54 |
| Quadrupled link | Four times the link word |
| Adjusted value | Quadrupled link minus 0x96 |
| Adjusted base | Four times the adjusted value |
| Association base | A separately captured global address word |
| Terminal link | Word reached through a later explicit byte-derived address |
| Terminal four-times base | Four times the terminal link |
| Terminal doubled base | Twice the terminal link |

Under truncated 32-bit arithmetic, the small precursor ranges from zero through 508 in multiples of four. The small base ranges from zero through 1016 in multiples of eight.

Those bounds do not apply to the converted-global base. Its source width and signedness are **documented_only**, and its resulting 32-bit value is not established to be byte-sized.

### Initial stores and first global capture

The final paired return supplies the low and high words used by the first direct stores.

The shifted-call base is written as a word at high-word-plus-0x14. The high word’s low byte is written at shifted-call-base-plus-0x09. A word at low-word-times-256-plus-0x54 receives the large scaled base. The offset-0x09 byte is then overwritten with the shifted-call base’s low byte.

The large scaled base’s offset-0x14 word is cleared before an explicit unsigned byte read at its offset 0x05. The small base’s offset-0x14 word is cleared, and the loaded byte is stored, zero-extended, as a word at large-scaled-base-plus-0x34.

A subsequent read of large-scaled-base-plus-0x14 captures a retained word. The small-base-plus-0x14 clear is repeated, followed by two byte stores of that retained word’s low byte at small-base-plus-0x15.

Only then is the named global at **address 0x05** evaluated and converted to unsigned 32 bits. The resulting converted-global base is captured **before** the byte-valued write of the small base’s low byte to **address 0x01**. No byte mask or byte-pointer load is established for this global evaluation.

After the address-0x01 write:

- Converted-global-base-plus-0x15 receives the small precursor’s low byte.
- Large-scaled-base-plus-0x11 receives the saved call word’s low byte.
- Retained-word-plus-0x54 receives the saved call word as a word.
- Large-scaled-base-plus-0x11 is cleared.
- Converted-global-base-plus-0x54 is read as the link word, then overwritten with the retained word.
- Small-base-plus-0x34 receives the small precursor as a word.

The retained word must not be replaced unconditionally with zero. Intervening accesses may overlap the cleared range, and the representation does not establish isolated ordinary memory.

Likewise, later uses of the converted-global base use the captured converted value. They do not implicitly reload address 0x05 after the address-0x01 write.

### Linked mutation and remaining early globals

A word read at large-scaled-base-plus-0x74 supplies a linked base. Its offset-0x54 word first receives the scale seed shifted left by 26.

An intervening assignment involving small-base-plus-0x74 represents two reads of that location: one supplies a destination base and another supplies the stored word. Their internal evaluation order is not established.

The link word is stored at link-word-plus-0x24. Its low byte is written at linked-base-plus-0x09, and linked-base-plus-0x54 receives the saved call word. The offset-0x24 word store and offset-0x09 byte store are repeated.

Small-base-plus-0x14 next receives the scale seed shifted left by 26. A word store to **address 0x54** then writes twice the link word. Only afterward does small-base-plus-0x14 receive the link word twice.

Two byte stores at small-base-plus-0x15 carry the quadrupled link’s low byte. The byte-valued global write to **address 0x15** follows them, carrying the small base’s low byte. It precedes the word store of the saved call word at quadrupled-link-plus-0x54.

Consequently, the address-0x05 evaluation, address-0x01 write, address-0x54 word write, and address-0x15 write form an ordered sequence interleaved with indirect accesses, not an unordered list of global effects.

### Intermediate linked accesses

The quadrupled link is written at small-base-plus-0x34. A word read at small-base-plus-0x74 is doubled to produce another retained base. The quadrupled link’s offset-0x54 word is overwritten with the quadrupled link itself.

The doubled value participates in offset-0x44 associations. An explicit unsigned byte read at quadrupled-link-plus-0x05 supplies the address component for an offset-0x74 word read.

Further mutations involve offsets 0x09, 0x0D, 0x14, 0x15, 0x44, and 0x54. Reads at quadrupled-link-plus-0x64 and quadrupled-link-plus-0x04 introduce additional bases.

The base obtained through offset 0x04 has its offset-0x05 byte read twice, with stores between those reads. Both are explicitly typed byte reads, unlike the earlier named-global evaluation at address 0x05. The second read supplies an offset-0x74 word lookup whose result becomes the terminal link.

The intervening mutations are observable: they may change later reads through aliasing. The two byte reads must not be collapsed into a single entry-time capture.

### Constant writes and repeated associations

After the intermediate linked mutations, 0x3A is written in three widths:

- As a byte at four times the terminal link.
- As a halfword at eight times the terminal link.
- As a word at terminal-four-times-base-plus-0x20.

The adjusted-base-plus-0x20 word receives, in order, 0x20, 0x2F, a zero-extended explicit byte read from adjusted-base-plus-0x04, and 0x20 again.

The separate global address word is captured **after** that sequence and **before** the word store of 0x20 to **address 0x60**. Its captured value becomes the association base. The address-0x60 store cannot retroactively replace the captured value, even if it overlaps the global’s storage.

A repeated association region follows. Each group has the same ordered effects:

- A derived base’s offset-0x54 word receives the terminal four-times base.
- Association-base-plus-0x19 receives the adjusted base’s low byte.
- Adjusted-base-plus-0x11 receives the association base’s low byte.
- Association-base-plus-0x34 receives the derived base as a word.

The first group uses eight times the original link word. Subsequent groups use eight times the adjusted value. Those later bases do not advance between groups.

Exact repetition multiplicity remains **documented_only** in this contract. Repeated equal-valued stores remain distinct represented events; matching only the final memory state does not establish equivalent behavior.

### Final captures, global interposition, and overwrites

The final region first captures a word from **address 0xA9**, then a word from terminal-four-times-base-plus-0x74.

Next, **address 0x3A** receives a computed byte. Under truncated arithmetic, its value is the low byte of eight times the adjusted value plus 0x30.

Only after that global write are words captured from terminal-doubled-base-plus-0x14 and adjusted-base-plus-0x04. These captures therefore occur against storage after the address-0x3A update.

The terminal doubled base’s offset-0x0D byte receives the adjusted base’s low byte. The adjusted base receives 0x6C at offset 0x08 and the association base’s low byte at offsets 0x0D and 0x09. Terminal-doubled-base-plus-0x54 receives the terminal doubled base itself.

The low byte of the captured address-0xA9 word is written to **address 0x67**. A word read at terminal-doubled-base-plus-0x44 intervenes before the later evaluation of **address 0x67** whose value is copied to **address 0x21**. That copy must use the later global evaluation, not an unconditional reuse of the original address-0xA9 capture.

The loaded offset-0x44 word supplies a byte at adjusted-base-plus-0x10 and a word at association-base-plus-0x54. It also supplies a shifted byte at terminal-doubled-base-plus-0x01. The shift count is the adjusted base’s low eight bits. Signed-right-shift behavior and counts outside the word width remain **documented_only**.

Association-base-plus-0x54 is then overwritten with the adjusted base.

A new word read at adjusted-base-plus-0x40 precedes stores of:

- The saved adjusted-base-plus-0x04 word at association-base-plus-0x74.
- The newly read offset-0x40 word at adjusted-base-plus-0x10.
- The terminal doubled base at association-base-plus-0x44.

The word write at adjusted-base-plus-0x10 can overwrite the earlier byte there.

A fresh evaluation of **address 0x67** supplies another byte-valued write at adjusted-base-plus-0x09. The terminal doubled base’s offset-0x09 byte receives the adjusted base’s low byte, and adjusted-base-plus-0x44 receives the captured address-0xA9 word.

A word read at adjusted-base-plus-0x54 supplies a new indirect base. Another fresh evaluation of **address 0x67** then supplies another write at adjusted-base-plus-0x09. Intervening stores may alias address 0x67; neither evaluation may be replaced by the originally captured low byte.

### Late linked writes and final transfer

The saved terminal-four-times-base-plus-0x74 word supplies its low byte at new-indirect-base-plus-0x0D. Terminal-doubled-base-plus-0x54 receives its previously saved offset-0x14 word.

A new word is read at adjusted-base-plus-0x14. The saved offset-0x74 word’s low byte is written at terminal-doubled-base-plus-0x11. The new indirect base’s low byte then overwrites adjusted-base-plus-0x09, and that base is stored at adjusted-base-plus-0x44.

A word read at terminal-doubled-base-plus-0x04 supplies the late link. A halfword store writes 0x65 at twice that link. Its offset-0x64 word is then captured as the late payload.

The newly read adjusted-base-plus-0x14 word supplies its low byte at new-indirect-base-plus-0x11. New-indirect-base-plus-0x54 receives decimal 100 as a word.

The late link’s offset-0x74 word is captured before the destination word read at terminal-doubled-base-plus-0x64. The terminal doubled base’s offset-0x09 byte receives the late link’s low byte. The late payload is stored at destination-plus-0x54, and the destination is stored as a word at twice the late payload.

F-unknown-02 receives these four represented argument words, in order:

1. 0x72.
2. The late payload.
3. The saved terminal-doubled-base-plus-0x14 word.
4. The captured late-link-plus-0x74 word.

No displayed cleanup, recovery, or success reporting follows.

## Side effects

Direct effects include ordered byte, halfword, and word stores, plus named-global evaluations and updates. Repeated stores remain separate events even when their addresses and values are equal.

Global and indirect storage are not established to be disjoint. An indirect write may affect a later global evaluation; a global update may affect a later indirect read.

For address 0x05 specifically, the unsigned conversion establishes the captured result’s width, not the underlying global access width or source signedness. Its possible overlap range therefore remains unresolved.

Unknown dependencies may mutate memory, perform I/O, allocate or release storage, block, fault, or fail to return. These are possibilities, not established individual effects.

No atomicity, synchronization, rollback, lock, or reentrancy guarantee is visible. If execution stops, an executed prefix of effects may remain observable. Actual hardware visibility and concurrency semantics remain **documented_only**.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-unknown-01 | Preliminary ordered indirect invocations; returned and residual machine values are consumed | Individual identities, signatures, preservation, ownership, effects, errors, termination |
| F-unknown-02 | Final unresolved transfer with four represented argument words | Target, calling convention, return behavior, effects, result convention |

F-unknown-01 denotes an unresolved family, not one proven shared target. F-unknown-02 is not established to be distinct from every preliminary target.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Word truncation.** Under the 32-bit truncation model, shifting 0x12345678 left by eight produces 0x34567800.

2. **Paired extraction.** A paired result with high word 0x80000000 yields 0x20000000 when extracted through an unsigned 64-bit right shift by 34.

3. **Masked transformation.** Masking 0xAB123456 to 24 bits and shifting left by six produces 0x048D1580.

4. **Explicit unsigned-byte promotion.** An explicitly typed byte read yielding 0xE7 contributes 0x000000E7 when promoted to an unsigned word.

5. **Address-0x05 is not byte-bounded.** If evaluating that global and converting its value yields 0x00001000, its later offset-0x15 byte destination is 0x00001015 and its offset-0x54 word lookup is at 0x00001054. Replacing the captured value with its low byte changes both addresses.

6. **Conditional signed source.** If the absent declaration supplies a signed value of −1, conversion to unsigned 32 bits yields 0xFFFFFFFF, not 0x000000FF. This example does not establish that the actual source is signed.

7. **Capture ordering.** The address-0x05 evaluation occurs after the duplicate small-base-plus-0x15 stores and before the address-0x01 write. If that write affects the global’s storage, subsequent computed accesses still use the earlier captured converted value.

8. **Association capture.** If the address-0x60 word store overlaps the separate address global, association operations still use the preceding capture. Reloading after the store is not equivalent.

9. **Fresh global evaluation.** If an intervening indirect store changes address 0x67, a later copy from that global uses the later value rather than the address-0xA9 word’s original low byte.

10. **Repeated overwrites.** Adjusted-base-plus-0x20 receives 0x20, 0x2F, the explicit byte-derived word, then 0x20. Final-state equality alone is insufficient.

11. **Interrupted execution.** A fault after earlier stores does not establish that those stores are undone. No rollback is represented.

## Uncertainties

The following remain **documented_only**:

- Application purpose and intended data structures.
- Completeness of the recovered interface and control-flow boundary.
- Exact indirect-target identities and complete invocation enumeration.
- Calling convention, implicit inputs, and register preservation.
- Address-0x05 global declaration, storage width, source signedness, and resulting overlap range.
- Pointer validity, mappings, alignment, and endianness.
- Signed overflow, signed shifts, and excessive-count semantics.
- Internal evaluation order of multiple accesses within a combined expression.
- Exact multiplicity of the repeated association region.
- Ownership, dependency effects, synchronization, and reentrancy.
- Error signaling, successful termination, and final-transfer return behavior.

[INFERENCE] Recurring reciprocal-looking stores may indicate association maintenance. That interpretation adds no ownership, layout, or consistency guarantee.

## Evidence

Established static evidence consists of the supplied representation’s indirect invocations, retained result dependencies, explicit widths, masks, shifts, ordered memory operations, repeated stores, and final-transfer arguments.

The address-0x05 distinction follows from its named-global evaluation followed by an unsigned 32-bit conversion, without a supplied declaration or explicit byte-pointer access. Its captured result feeds both the offset-0x15 store address and the offset-0x54 word lookup. No intervening operation restricts that captured value to eight bits.

The ordering requirements follow from represented access placement: the address-0x05 capture precedes the address-0x01 write; the association-address capture precedes the address-0x60 store; the address-0x3A update precedes later saved-word reads; and address 0x67 is evaluated again after intervening mutations.

No implementation inspection, executable inspection, runtime tests, or dynamic access traces were performed. Arithmetic examples are conditional calculations, not runtime observations.
