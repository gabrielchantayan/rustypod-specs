<!-- Trusted provenance: F-8c9eb7c059dd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8c9eb7c059dd

## Purpose

F-8c9eb7c059dd performs an ordered chain of unresolved indirect calls, transforms their results, mutates fixed and indirectly addressed storage, and invokes a final unresolved target with four represented arguments.

The supplied representation does not establish an application-level purpose. Repeated offsets and numeric constants do not establish text output, initialization, serialization, object construction, or device control.

This contract distinguishes:

- **Established static evidence:** represented inputs, conversions, explicitly typed indirect accesses, captures, and statement ordering.
- **[INFERENCE]:** deductions conditional on a stated arithmetic or execution model.
- **documented_only:** unresolved behavior for which no executable guarantee is established.

No runtime execution or tests have been performed.

## Interface

### Explicit inputs and output

The represented interface has four positional inputs and no declared return value.

| Position | Represented width | Explicit use |
|---|---:|---|
| First | 32 bits | No explicit reference in the body. |
| Second | 32 bits | No explicit reference in the body. |
| Third | Signed 32 bits | Supplies an early call argument after a left shift by four. |
| Fourth | Signed 32 bits | Supplies an early call argument after a left shift by 25. |

The first two inputs’ absence from explicit expressions does not prove independence from them through implicit execution state or unresolved calling conventions.

None of these inputs is explicitly dereferenced. Later addresses come from dependency results, arithmetic, fixed-storage observations, and indirect loads. No caller-provided buffer layout is established.

There is no explicit success flag, error result, or returned scalar. Observable output consists of dependency interactions and memory effects. If the final target returns normally, the represented body returns without supplying a declared value.

### Implicit inputs and preconditions

Implicit inputs include entry control-transfer context, unresolved call targets, primary and secondary dependency results, fixed shared storage, and indirectly reachable memory.

Normal completion requires every executed transfer and access to be supported. The body contains no visible null, bounds, alignment, mapping, executable-permission, or dependency-success checks. Low numeric addresses are not rejected as sentinels.

No ownership, lifetime, initialization, or nonaliasing guarantee is established for reachable storage.

## Behavior

### Execution and dependency phase

The represented flow consists of a long indirect-call phase, a direct-memory phase without intervening dependency calls, and a final indirect invocation. No explicit loop or conditional branch is represented. Dependencies may themselves branch, mutate memory, fail, or never return.

The first target is derived from entry control-transfer context plus 0xFFE. Later targets repeatedly use internal location-derived values plus that displacement. Their identities remain unresolved.

Calls have zero through four explicitly represented arguments. Captured results include 32-bit values, combined 64-bit values, and incompletely specified secondary result state. Calls whose results are ignored remain interactions; their effects cannot be discarded.

Early arguments include the shifted third input and, later, a dependency result shifted left by eight paired with the fourth input shifted left by 25. Subsequent arguments repeatedly combine transformed primary and secondary results, identical pairs, explicit zeros, and earlier captures. Unknown secondary results must not be replaced with zero or presumed preserved across unrelated calls.

No dependency contract establishes preservation of globals or reachable memory. A later observation therefore concerns memory at that observation point, not necessarily entry state.

### Widths, conversions, and fixed storage

Explicitly typed indirect accesses establish represented widths:

- Byte accesses: 8 bits.
- Halfword stores: 16 bits.
- Word accesses: 32 bits.

An explicitly unsigned byte load converted to an unsigned 32-bit value establishes zero-extension from eight bits. A byte-sized destination establishes an eight-bit store even when its source is a named global of unknown type.

**Named fixed-storage globals are different. Their declarations are absent.** Their address labels identify storage references, not declared types or access widths. Neither a receiving 32-bit value nor a narrowed assignment operand establishes the global’s original access width.

| Fixed-storage label | Established value operation | Unresolved storage property |
|---|---|---|
| 0x05 | Observation explicitly converted to unsigned 32 bits; result retained as an address base | Source width and signedness; whether extension occurs |
| 0x01 | Assignment operand explicitly narrowed to eight bits | Destination width and destination conversion |
| 0x54 | Assignment operand is captured-link value shifted left by one | Destination width and destination conversion |
| 0x15 | Assignment operand explicitly narrowed to eight bits | Destination width and destination conversion |
| 0x087FBBBC | Observation retained in a signed 32-bit capture | Original read width and conversion |
| 0x60 | Assignment operand is 0x20 | Destination width and conversion |
| 0xA9 | Observation retained in a 32-bit capture | Original read width and conversion |
| 0x3A | Character-sized conversion participates in subsequent arithmetic | Character signedness, destination conversion, and store width |
| 0x67 | Receives an explicitly eight-bit-narrowed operand; later observed three separate times | Store width, read widths, and conversions |
| 0x21 | Assigned from a fresh observation of fixed 0x67 | Both globals’ declarations and resulting conversions |

In particular, conversion of fixed 0x05 to unsigned 32 bits does **not** establish a range of 0–255. Assigning an eight-bit operand to fixed 0x67 does **not** establish an eight-bit store. Assigning fixed 0x67 to fixed 0x21 does **not** establish an eight-bit read or write.

### Arithmetic and layout

Combined dependency results are represented as 64 bits. Low-result extraction retains 32 bits. Upper-result extraction uses an unsigned 64-bit right shift by 32; other extractions shift by 34, 36, or 44 before narrowing.

Masks select fields of several widths, including 11–14, 16–22, and 24 bits. Masking before shifting is not interchangeable with masking afterward.

**[INFERENCE]:** Under wrapping 32-bit arithmetic, left shifts and products retain their low 32 bits, and explicit eight-bit narrowing retains the low eight bits. This is not an unconditional interpretation of signed overflow or signed left shifts.

One late byte store depends on a signed 32-bit right shift whose count is the low eight bits of a derived address. Negative operands and counts of at least 32 remain documented_only; no count normalization is established.

Offsets describe access relationships, not validated structures. Word accesses include offsets 0x04, 0x14, 0x40, 0x44, 0x54, 0x64, and 0x74. Byte and word accesses can overlap, including at offset 0x10. Endianness, supported alignment, address arithmetic, and complete alias relationships remain unresolved.

### Opening direct-memory effects

The following roles describe arithmetic captures, not object types:

| Role | Derivation |
|---|---|
| High result; low result | Upper and lower 32 bits of the final combined result before direct mutations |
| Packed base | An earlier low result shifted left by 25 |
| Scaled base | A separate 32-bit dependency result multiplied by \(2^{25}\) |
| Small index | Another result shifted left by eight, masked with 0x7FFF, then unsigned-shifted right by six |
| Small base | Twice the small index |
| Shifted value | Another dependency result shifted left by eight |

The opening effects place the packed base in a word at high-result offset 0x14 and the high result’s low byte at packed-base offset 0x09. A word store at offset 0x54 from 256 times the low result contains the scaled base. A second byte store at packed-base offset 0x09 replaces the first with the packed base’s low byte.

Scaled-base offset 0x14 is cleared as a word. An explicit unsigned byte load from scaled-base offset 0x05 is captured and zero-extended for a word store at scaled-base offset 0x34. A small-base offset-0x14 word clear intervenes before that store.

A fresh word read at scaled-base offset 0x14 produces the **opening readback**. Another small-base offset-0x14 clear precedes two distinct byte stores of the opening readback at small-base offset 0x15. The readback remains an actual observation, not a substituted zero; aliases or supported memory semantics may affect it.

**[INFERENCE]:** With wrapping 32-bit shifts, the packed base’s low byte is zero. The small index is a multiple of four from 0 through 508, and the small base a multiple of eight from 0 through 1016. These ranges establish no address validity.

### Opening fixed-storage captures

After both small-base offset-0x15 stores, fixed 0x05 is observed and explicitly converted to unsigned 32 bits, producing the **captured fixed base**. Only afterward is fixed 0x01 assigned the explicitly eight-bit-narrowed small base. Both globals’ access widths remain unresolved.

The captured fixed base supplies a byte-store destination at offset 0x15; the stored value is the small index’s low byte. Subsequent effects place the shifted value’s low byte at scaled-base offset 0x11 and its word at opening-readback offset 0x54, then clear that scaled-base byte.

A word read at captured-fixed-base offset 0x54 produces the **captured link**. That location is subsequently overwritten with the opening readback. The small index is then stored as a word at small-base offset 0x34. These accesses reuse the captured base without rereading fixed 0x05.

A word from scaled-base offset 0x74 supplies another destination. Two distinct word observations at small-base offset 0x74 supply a destination relationship and a stored value; they must not be merged. Their within-expression evaluation order is unresolved.

Captured-link self-stores at offset 0x24, byte stores at the loaded destination’s offset 0x09, and intervening offset-0x54 mutations precede a small-base offset-0x14 word store containing the scaled-base producer shifted left by 26.

Fixed 0x54 is then assigned the captured link shifted left by one. Its store width is unknown. Two subsequent word stores place the captured link at small-base offset 0x14.

Two distinct byte stores next place the low byte of four times the captured link at small-base offset 0x15. Only after both is fixed 0x15 assigned the explicitly narrowed small base. Its store width is unknown. Offset-0x54 and offset-0x34 word mutations follow.

The fixed-0x54 assignment and captured-fixed-base offset-0x54 accesses remain distinct events, including when their effective storage overlaps.

### Linked mutations and repeated fields

The next region follows word-derived and byte-derived addresses, uses doubled, quadrupled, and eightfold arithmetic, and performs self-referential stores. No list, tree, ownership, or reference-count invariant is established.

Explicit unsigned byte loads at offset 0x05 supply bases for word reads at offset 0x74. These loads genuinely establish eight-bit source values and zero-extension, unlike the named fixed-0x05 observation.

A later lookup supplies the **linked base**. Its double is the **doubled link**; its quadruple is the **constant base**. The body writes 0x3A as a byte at the constant base, as a halfword at eight times the linked base, and as a word at constant-base offset 0x20.

The **adjusted base** is four times the captured link minus 0x96. Four times that value is the **repeated-field base**.

Repeated-field offset 0x20 receives four successive word stores: 0x20, 0x2F, a freshly observed unsigned byte from repeated-field offset 0x04 zero-extended to 32 bits, and 0x20. That byte observation occurs after the second store and before the third.

After the fourth store, fixed 0x087FBBBC is observed into a signed 32-bit **shared base** capture. Fixed 0x60 is then assigned 0x20 before the repeated-store region. Neither global’s access width is established. The shared base remains the captured value despite subsequent mutations.

The opening group places the constant base in a word at offset 0x54 from eight times the captured link. Its associated byte stores place the repeated-field base’s low byte at shared-base offset 0x19 and the shared base’s low byte at repeated-field offset 0x11. A word at shared-base offset 0x34 receives eight times the captured link.

Later groups retain those captured byte values but use eight times the adjusted base for both the offset-0x54 destination base and the shared-base offset-0x34 value. The byte values are arithmetic captures, not observations of the opposite fields. Repeated stores remain distinct events. Exact group multiplicity is documented_only here.

### Final shared-state captures

Fixed 0xA9 is observed into the 32-bit **captured shared word**. “Word” describes the capture, not the unknown global read width. An explicit word read at constant-base offset 0x74 then produces the **captured linked word**.

Only afterward is fixed 0x3A assigned an expression involving the adjusted base converted to a character-sized value, multiplied by eight, and increased by 0x30. The expression does not establish an eight-bit global store or character encoding. Character signedness and destination conversion remain unresolved.

The body next captures a **saved word** from doubled-link offset 0x14 and a **saved field** from repeated-field offset 0x04, both through explicit word accesses.

Before the next fixed assignment, byte stores place the repeated-field-base byte at doubled-link offset 0x0D, 0x6C at repeated-field offset 0x08, and the shared-base byte at repeated-field offsets 0x0D and 0x09. A word self-store places the doubled link at its offset 0x54.

Fixed 0x67 then receives an explicitly eight-bit-narrowed captured shared word. An explicit word read at doubled-link offset 0x44 intervenes before fixed 0x21 is assigned from a fresh observation of fixed 0x67. This assignment must not substitute the earlier capture. The fixed accesses’ widths remain unknown.

### Later observations, restoration, and termination

The doubled-link offset-0x44 capture supplies a byte store at repeated-field offset 0x10, a word store at shared-base offset 0x54, and the signed-shift-dependent byte store at doubled-link offset 0x01. Shared-base offset 0x54 is subsequently overwritten with the repeated-field base.

A fresh word from repeated-field offset 0x40 supplies a word overwrite at repeated-field offset 0x10. Between that capture and overwrite, shared-base offset 0x74 receives the saved field. Shared-base offset 0x44 subsequently receives the doubled link.

A fresh observation of fixed 0x67 supplies an explicitly byte-sized store at repeated-field offset 0x09. Later effects place the repeated-field-base byte at doubled-link offset 0x09 and the captured shared word at repeated-field offset 0x44.

A word read at repeated-field offset 0x54 produces the **loaded destination**. Only afterward does another fresh observation of fixed 0x67 supply another byte store at repeated-field offset 0x09.

There are therefore **three separate observations of fixed 0x67**: one for fixed 0x21 and two for indirect byte stores. None has an established source-read width. The two indirect destinations nevertheless have established eight-bit store widths.

The restoration boundary preserves this order:

- The captured linked word’s low byte is stored at loaded-destination offset 0x0D before the saved word is restored by a word store at doubled-link offset 0x54.
- A word read at repeated-field offset 0x14 captures the **late field** after restoration but before the captured linked word’s low byte is stored at doubled-link offset 0x11.
- The loaded destination is subsequently stored as a byte at repeated-field offset 0x09 and as a word at repeated-field offset 0x44.

These boundaries cannot be reordered on an assumption of nonaliasing.

A word from doubled-link offset 0x04 captures the **terminal base**. A halfword store of 0x65 at twice that base precedes a word read at terminal-base offset 0x64, producing the **terminal link**.

The late field’s low byte is stored at loaded-destination offset 0x11, and loaded-destination offset 0x54 receives the word value decimal 100.

A later word from terminal-base offset 0x74 captures the final fourth argument. Another word from doubled-link offset 0x64 supplies a destination. After a byte store of the terminal base at doubled-link offset 0x09, that destination’s offset 0x54 receives the terminal link, and twice the terminal link receives the destination as a word.

The final unresolved target receives:

| Position | Represented value |
|---|---|
| First | 0x72 |
| Second | Captured terminal link |
| Third | Saved word from doubled-link offset 0x14 |
| Fourth | Later captured word from terminal-base offset 0x74 |

These arguments use captures, not replacement reads.

## Side effects

Direct effects include mixed-width indirect mutations and named-global assignments of unresolved storage width. Dependencies may have additional unknown effects.

No transaction, rollback, synchronization, explicit error reporting, or cleanup path is represented. A fault or nonreturning dependency may prevent later events while leaving earlier effects intact.

Final-memory equality alone does not establish equivalence: overwritten values, repeated stores, separate observations, and ordering may be observable under supported memory semantics. A restoration denotes a store event, not guaranteed persistence.

No explicit allocation or release is established. Unknown dependencies may allocate, retain, transfer, or release resources. Ownership, lifetime, reentrancy, and thread safety remain documented_only.

## Dependencies

F-8c9eb7c059dd is the only supplied resolved opaque function ID. No opaque ID resolves any indirect dependency; none is invented here.

Entry-context-derived, intermediate, and final targets remain unresolved as to identity, calling convention, argument interpretation, secondary results, errors, global preservation, memory effects, ownership, and return behavior.

Ignored represented results do not make calls pure or removable.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

### Shifted input

Under wrapping 32-bit shifts, third inputs differing only in their highest four bits produce the same explicitly represented shift-by-four argument. Whole-function equivalence does not follow.

### Fixed-0x05 conversion and capture

If the fixed-0x05 observation, after its explicit unsigned 32-bit conversion, yields 0x000000E3, that is the captured base. Its capture precedes the fixed-0x01 assignment, and later changes to fixed 0x05 do not replace it.

This condition does not imply an eight-bit source read. If a supplied declaration instead permits a source value of 0x000001E3 that converts unchanged, the captured base is 0x000001E3, not 0xE3.

### Explicit byte-load extension

If the explicit unsigned byte load at scaled-base offset 0x05 observes 0xE3, the later word store at scaled-base offset 0x34 receives 0x000000E3. This extension is established by the indirect access type.

### Fixed-global assignments

When fixed 0x01 receives the explicitly narrowed small base, the operand is eight bits. Without its declaration, acceptance must not require that only one byte of fixed storage changes. The same distinction applies to the narrowed assignments to fixed 0x15 and fixed 0x67.

### Separate fixed-0x67 observations

If intervening supported effects change fixed 0x67, fixed 0x21 receives its fresh observation subject to the globals’ conversions. The two later indirect stores likewise use their own observations and store eight-bit destination values. Their source-read widths remain unresolved, and their stored bytes need not match.

### Captures and restoration

Repeated groups use the captured shared base even if subsequent effects alter fixed 0x087FBBBC.

If doubled-link offset 0x11 overlaps repeated-field offset 0x14, the late-field capture occurs before that byte store but after saved-word restoration. Moving the byte store earlier can change the capture and its later use.

### Termination

If an earlier dependency never returns, the represented flow does not reach later direct effects or the final invocation. No recovery is established.

If the final invocation is reached, its third argument remains the saved capture regardless of later changes to its original storage.

## Uncertainties

The following remain documented_only:

- Application intent and completeness of the represented interface and flow.
- Indirect dependency identities, calling conventions, secondary-result provenance, and preservation guarantees.
- Named fixed-storage declarations, access widths, signedness, destination conversions, and overlap extent.
- Exact repeated-store group multiplicity.
- Signed overflow, signed shifts, excessive shift counts, and address arithmetic.
- Character signedness in the fixed-0x3A expression.
- Endianness, alignment support, address mapping, and memory kind.
- Complete aliases and within-expression evaluation order where not independently established.
- Ownership, lifetime, concurrency, fault behavior, and final-target completion.

Missing global declarations do not erase established capture boundaries, statement ordering, or explicitly typed indirect-access widths.

## Evidence

This contract rests on static examination of the supplied representation. Prior prose, comments, and reviewer interpretations are not independent proof.

The representation establishes four positional input widths, unresolved indirect interactions, result transformations, explicitly typed mixed-width indirect accesses, fixed-storage observation boundaries, restoration ordering, and captured final arguments.

The fixed-storage declarations are not supplied. Conversions establish receiving values or assignment operands, not the underlying globals’ access widths. In contrast, explicit indirect byte, halfword, and word accesses establish their represented widths.

Arithmetic deductions are conditional where marked. No runtime execution or tests have been performed.
