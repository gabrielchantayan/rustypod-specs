<!-- Trusted provenance: F-c7ec7ebb705f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c7ec7ebb705f

## Purpose

F-c7ec7ebb705f performs an ordered sequence of indirect interactions, derives address-like quantities from their results, modifies fixed and computed storage, and ends with a four-argument indirect interaction.

No application-level purpose is established. The representation does not establish allocation, collection management, serialization, text output, or initialization. Constants that coincide with printable character values do not establish character operations.

This contract distinguishes:

- Captured values from later observations of their source storage.
- Expression and result widths from underlying storage-access widths.
- Explicitly typed computed accesses from observations and assignments involving undeclared named globals.

Overlapping accesses may change subsequent observations. Repeated and subsequently overwritten assignments remain part of the represented operation history.

## Interface

### Invocation and outputs

The represented interface has no explicit parameters and no explicit scalar return value. Effective inputs nevertheless include an incoming 32-bit quantity, incoming control-address state, dependency results, additional result channels, and memory contents. A conventional parameterless invocation is not established as a sufficient calling protocol.

The first explicit dependency argument is the incoming quantity shifted left by 22 positions. Under the 32-bit arithmetic model below, only its original low ten bits contribute.

Outputs consist of interactions and memory effects. No success flag, returned resource, or ownership transfer is established.

The terminal interaction receives:

| Position | Argument |
|---|---|
| First | Constant 0x72 |
| Second | Captured 32-bit word from terminal-base offset 0x64 |
| Third | Captured 32-bit word from retained-base offset 0x14 |
| Fourth | Later captured 32-bit word from terminal-base offset 0x74 |

The third argument is captured after the completion-stage assignment to the named global at 0x3A. Subsequent changes to its original storage do not replace that captured value.

A return follows the terminal interaction in the representation. Normal return from the target is **documented_only**.

### Widths and prerequisites

Explicit computed byte, halfword, and word accesses have widths of 8, 16, and 32 bits. Paired dependency results have a represented width of 64 bits. Explicit unsigned-byte loads yield values from 0 through 255.

**Named globals have no supplied storage declarations.** This applies to addresses 0x01, 0x05, 0x15, 0x21, 0x3A, 0x54, 0x60, 0x67, 0xA9, and the publication-anchor source at 0x087FBBBC.

For those globals:

- An eight-bit conversion on an assignment’s right-hand side does not prove an eight-bit destination access.
- Capture into a 32-bit quantity does not prove a 32-bit source access.
- A later observation does not inherit the width of a previously assigned expression.
- Underlying access widths, declaration-dependent conversions, and affected byte ranges remain **documented_only**.

Offsets are byte offsets, not established object fields. No allocation bounds, alignment, separation, or non-aliasing guarantee is supplied.

Completion requires every reached target and memory access to be supported and every consumed result channel to be usable. No represented validation establishes these prerequisites.

## Behavior

### Arithmetic and value roles

Numerical deductions use 32-bit bit-vector arithmetic for address construction and word calculations: addition, subtraction, multiplication, and left shifts retain the low 32 bits. This is a conditional arithmetic model, not a claim that all represented signed operations have portable overflow semantics.

Paired-result halves are numerical lower and upper 32-bit words; their extraction does not establish a memory-endianness convention. Unsigned shifts of a 64-bit result by 34, 36, and 44 select its upper word shifted right by 2, 4, and 12 respectively.

Other deductions include:

- Left shift by eight followed by unsigned right shift by eight retains the original low 24 bits.
- Masking to 24 bits before scaling is material.
- Extraction of bits 10–19 yields a ten-bit unsigned quantity.
- Two successive 28-position left shifts of a 32-bit quantity produce zero under this model.

The following descriptive roles identify origins, not types or ownership:

| Role | Derivation |
|---|---|
| Pair-packed address | Earlier paired lower word scaled by 2²⁵ |
| Scalar-packed address | Independent scalar result multiplied by 0x02000000 |
| Payload | Another scalar result shifted left by eight |
| Small index | Returned word shifted left by eight, masked to fifteen low bits, then unsigned-shifted right by six |
| Small address | Twice the small index |
| Final pair | Paired result immediately preceding direct memory modification |
| Global-selected base | Observation of the named global at 0x05 converted to unsigned 32 bits |
| Link word | Word captured at global-selected-base offset 0x54 |
| Fourfold link | Four times the link word |
| Traversal word | Word selected through the later explicitly unsigned-byte traversal observation |
| Retained base | Twice the traversal word |
| Post-subtraction origin | Fourfold link minus 0x96 |
| Adjusted base | Four times the post-subtraction origin |
| Publication anchor | Named-global observation captured into a 32-bit quantity after adjusted-base replacements |
| Terminal base | Later word captured at retained-base offset 0x04 |

Both packed addresses have 25 low zero bits, but their independent origins do not require equality. The payload has eight low zero bits.

The small index is a multiple of four between 0 and 508; the small address is a multiple of eight between 0 and 1016. These bounds do not establish accessibility.

The global-selected base has an unsigned 32-bit result representation, not an established 0–255 range. The publication anchor likewise has a 32-bit captured representation without an established source-access width.

### Indirect interaction phase

A long, straight-line sequence of indirect interactions precedes direct memory modification. No explicit input-dependent branch, retry, or runtime-counted loop is represented.

The first target depends on incoming control-address state. Later target constructions use embedded control locations displaced by 0xFFE. These constructions do not establish conventional function identities or prove that distinct sites reach distinct dependencies.

Explicit argument lists contain zero through four values. Material distinctions include scalar scaling, numerical halves of paired results, masking before scaling, explicit zeros, duplicated arguments, retained earlier results, and additional result channels.

An empty explicit argument list does not exclude implicit inputs. An unused represented return does not make an interaction removable. Complete preliminary per-site argument contracts are **documented_only** here; no permission to omit or reorder interactions follows.

### Initial linkage and clearing

The final pair supplies the initial linkage. Its upper-word offset 0x14 receives the pair-packed address as a word. Pair-packed offset 0x09 receives the upper word’s low byte, and the word at 256 times the final lower word plus 0x54 receives the scalar-packed address. Pair-packed offset 0x09 is subsequently replaced with the pair-packed address’s low byte, zero under the arithmetic model.

The replaced byte remains a distinct represented write.

Clearing and unsigned-byte widening concern the scalar-packed address, not the pair-packed address. Scalar-packed offset 0x14 is cleared before an explicitly unsigned-byte load at its offset 0x05. Small-address offset 0x14 is then cleared, and the captured byte is widened into the word at scalar-packed offset 0x34. Scalar-packed offset 0x14 is read afterward.

For stable ordinary storage, that word observation is zero: intervening potentially overlapping clearing also writes zero, and scalar-packed offsets 0x34 and 0x14 are distinct. This deduction excludes external mutation and special-storage behavior.

Small-address offset 0x14 is cleared again. Two consecutive one-byte writes at small-address offset 0x15 use the captured word’s low byte.

### Global selection and cross-linking

The named global at 0x05 is observed and converted to unsigned 32 bits after those two one-byte writes. Its source width and extension behavior are **documented_only**.

Before the link word is captured at the selected base’s offset 0x54, the represented order includes:

1. Assignment to the named global at 0x01 from the small address converted to eight bits.
2. One-byte write of the small index’s low byte at selected-base offset 0x15.
3. One-byte write of the payload’s low byte at scalar-packed offset 0x11.
4. Word publication of the payload at offset 0x54 through the earlier scalar-packed offset-0x14 result.
5. One-byte replacement with zero at scalar-packed offset 0x11.

The link word is then captured, and its source location is replaced with the earlier scalar-packed offset-0x14 result. Preceding assignments may alias the link location. Later changes to the global’s source storage do not retroactively change the captured selected base.

Further linkage includes repeated self-valued word writes at link-word offset 0x24; word replacements at small-address offset 0x14; and payload and scaled-scalar publications through a destination captured at scalar-packed offset 0x74. The independent scalar scaling used here is by 2²⁶.

The named global at 0x54 receives an expression representing twice the link word. The named global at 0x15 receives an eight-bit conversion of the small address. Neither assignment establishes its destination access width.

Fourfold-link offset 0x54 receives the payload before self-valued replacements. Intermediate linkage also involves small-address offsets 0x34 and 0x74, link-word offset 0x44, and destinations selected through further word observations. These operations cannot be reduced to final-state values without losing ordering and alias effects.

### Computed-byte traversal

Explicitly unsigned-byte computed loads remain distinct from undeclared named-global observations.

An unsigned byte at fourfold-link offset 0x05 selects an offset-0x74 word used for intermediate linkage. Later traversal captures a word at fourfold-link offset 0x04 and observes an unsigned byte at that loaded base’s offset 0x05 twice.

Memory changes occur between those byte observations. The later byte selects the offset-0x74 word designated as the traversal word. Reusing the earlier byte is not established as equivalent.

These explicitly byte-derived selectors have the 0–255 bound. Evaluation order among multiple loads within one expression is **documented_only**; order between surrounding completed operations remains established.

### Constant replacement and repeated publication

The traversal word controls three explicitly sized publications of 0x3A:

| Destination | Access width |
|---|---|
| Four times the traversal word | 8 bits |
| Eight times the traversal word | 16 bits |
| Fourfold-traversal offset 0x20 | 32 bits |

These are computed accesses, not the later named-global assignment at address 0x3A.

Adjusted-base offset 0x20 receives successive words: 0x20, 0x2F, a widened unsigned byte observed at adjusted offset 0x04, and 0x20 again. The byte observation follows the first two replacements.

The publication anchor is then captured from named global storage at 0x087FBBBC. Its captured representation is 32 bits; its underlying read width and declaration-dependent conversion remain **documented_only**. The named global at 0x60 subsequently receives constant 0x20, with destination width **documented_only**.

Each occurrence of the repeated region preserves this order:

| Order | Explicit computed write |
|---|---|
| 1 | Fourfold traversal value as a word at scaled-destination offset 0x54 |
| 2 | Adjusted base’s low byte at anchor offset 0x19 |
| 3 | Anchor’s low byte at adjusted offset 0x11 |
| 4 | Scaled destination as a word at anchor offset 0x34 |

The first scaled destination is eight times the original link word. Subsequent destinations are eight times the post-subtraction origin: subtraction precedes scaling.

Repetition is finite and statically expanded, not input-counted or convergence-driven. Exact multiplicity is not enumerated in this contract and is **documented_only** here. Repeated operations are not thereby optional or coalescible.

### Completion capture boundary

After repetition, the following order is established independently of named-global access widths:

| Order | Operation |
|---|---|
| 1 | Observe the named global at 0xA9 and capture its converted 32-bit result |
| 2 | Read a 32-bit word at fourfold-traversal offset 0x74 |
| 3 | Assign the completion expression to the named global at 0x3A |
| 4 | Read a 32-bit word at retained-base offset 0x14 |
| 5 | Read a 32-bit word at adjusted-base offset 0x04 |

The completion expression scales a character-converted post-subtraction origin by eight and adds 0x30. It is not established as an eight-bit expression merely because it includes a character conversion. Under the stated bit-vector treatment of narrowing, its low eight bits equal those of eight times the post-subtraction origin plus 0x30. Its full value and destination conversion depend on character and global declarations.

The assignment at order 3 precedes both later captures. Its affected storage range is **documented_only**, so exact overlap effects require an additional declaration assumption. Earlier captured values are not replaced when their source storage changes.

### Completion updates and terminal publication

Retained offset 0x0D receives the adjusted base’s low byte. Adjusted offset 0x08 receives byte 0x6C, while adjusted offsets 0x0D and 0x09 receive the anchor’s low byte. Retained offset 0x54 receives the retained base as a word.

The captured 32-bit result originating at 0xA9 supplies an eight-bit converted expression assigned to the named global at 0x67. Its destination access width is unknown. After capture of the word at retained offset 0x44, the named global at 0x21 receives a fresh observation of the global at 0x67. Neither side of that assignment has a declared storage width.

The retained-offset-0x44 word supplies subsequent byte and word writes, including a shifted byte at retained offset 0x01. The operand is represented as signed, and the shift count is the adjusted base’s low eight bits. Signed-right-shift behavior and out-of-range counts remain **documented_only**.

Under the arithmetic model, the adjusted base is congruent to 8 modulo 16. Possible low-byte counts are 8, 24, 40, and so on through 248; only 8 and 24 are below 32.

Adjusted offset 0x40 is captured after preceding updates, then written as a word at adjusted offset 0x10, replacing an earlier byte there. Anchor offsets 0x44, 0x54, and 0x74 participate in linkage. The post-0x3A adjusted-offset-0x04 capture supplies anchor offset 0x74.

Later one-byte copies into adjusted offset 0x09 use fresh observations of the global at 0x67, converted for their explicitly one-byte destinations. Their source access widths remain unknown; their values are not necessarily the initial assigned expression.

A word captured at adjusted offset 0x54 supplies a separate update base. Retained offset 0x54 receives the post-0x3A retained-offset-0x14 capture. Adjusted offset 0x14 is captured for a later byte write. Adjusted offset 0x44 is replaced first with the captured 0xA9-derived result and later with the separate update base.

Retained offset 0x04 selects the terminal base. A halfword write of 0x65 at twice that base precedes capture of terminal offset 0x64.

The separate update base receives a byte at offset 0x11 and word 100 at offset 0x54 before terminal offset 0x74 is captured. Aliasing may therefore affect the fourth terminal argument.

Finally, retained offset 0x64 selects another destination. Retained offset 0x09 receives the terminal base’s low byte; the selected destination’s offset 0x54 receives the captured terminal-offset-0x64 word; and the word at twice that captured value receives the selected destination. These writes precede the terminal interaction.

## Side effects

### Named-global operations

| Address | Established expression or captured-result property | Underlying storage-access width |
|---|---|---|
| 0x01 | Assigned eight-bit conversion of the small address | **documented_only** |
| 0x05 | Observation converted to unsigned 32-bit selected base | **documented_only** |
| 0x15 | Assigned eight-bit conversion of the small address | **documented_only** |
| 0x21 | Assigned a fresh observation of global 0x67 | **documented_only** for source and destination |
| 0x3A | Assigned completion arithmetic expression | **documented_only** |
| 0x54 | Assigned twice-link expression | **documented_only** |
| 0x60 | Assigned constant 0x20 | **documented_only** |
| 0x67 | Initially assigned eight-bit conversion of captured 0xA9-derived result; subsequently observed afresh | **documented_only** |
| 0xA9 | Observation captured into a 32-bit result | **documented_only** |
| 0x087FBBBC | Observation captured into the 32-bit publication anchor | **documented_only** |

Consequently, these operations do not establish unconditional byte or word accesses, exact neighboring-byte preservation, or an eight-bit range for later global observations.

Explicitly typed computed accesses retain their stated widths even when their numerical addresses coincide with named-global addresses.

### Ownership, interruption, and errors

No directly identifiable allocation or release establishes an ownership protocol. Address-like results are not thereby owned resources or lifetime-guaranteed borrows.

Dependencies may modify globals, retain addresses, allocate, or release storage. Preservation and continued validity must not be assumed.

No validation branch, explicit error return, cleanup handler, rollback, lock, barrier, or transaction is represented. Invalid targets, inaccessible storage, unsupported alignment, and unusable result channels have no established recovery behavior. Later effects require earlier operations to complete; completed effects are not represented as reversed after interruption.

## Dependencies

F-c7ec7ebb705f is the only supplied authoritative opaque function ID. No opaque IDs or independent contracts are supplied for its indirect dependencies. Their identities remain **documented_only**; private labels are not dependency identities in this contract.

Dependency roles include the entry-state-selected target, embedded-location-selected targets, scalar and paired-result suppliers, additional-channel suppliers, and the terminal target.

No dependency is established as pure, normally returning, global-preserving, or ownership-preserving.

## Acceptance examples

These are conditional static deductions, not executed tests.

1. **Entry truncation.** Under 32-bit arithmetic, incoming value 1 produces first argument 0x00400000; value 0x400 produces zero.

2. **Independent packed addresses.** Paired lower word 1 and independent scalar result 2 produce 0x02000000 and 0x04000000 respectively. Clearing concerns the latter.

3. **Small-address boundary.** Relevant returned low seven bits 0x7F produce small index 508 and small address 1016.

4. **Conditional global range.** If global 0x05 is unsigned eight-bit storage observed as 0xFF, its converted base is 255 and offset-0x54 address is 0x153. If a supported wider declaration instead supplies 0x100, unsigned 32-bit conversion retains 0x100 and selects 0x154.

5. **Expression width is not destination width.** An eight-bit converted expression assigned at global 0x01 does not establish that adjacent bytes are preserved. That conclusion requires an eight-bit destination declaration and ordinary supported storage semantics.

6. **Overwritten computed write.** A final upper word ending in 0xAB causes pair-packed offset 0x09 to receive byte 0xAB before its later zero replacement. Reporting only zero omits a represented write.

7. **Traversal publication widths.** The three traversal-controlled 0x3A publications use byte, halfword, and word accesses respectively. They do not establish the width of the separate named-global assignment at 0x3A.

8. **Completion alias with explicit declaration assumption.** Suppose completion is reached with traversal word 0x13 and link word 0x26. The retained base is 0x26, retained offset 0x14 is address 0x3A, and the post-subtraction origin is 2. **Additionally assume an eight-bit destination at global 0x3A**, ordinary little-endian storage, supported accesses, and no external mutation. The assignment stores 0x40. A word 0x11223344 immediately beforehand becomes 0x11223340 for the following retained-word capture. Subject to continued execution, that capture supplies the third terminal argument. Little-endian storage alone is insufficient to establish this one-byte effect.

9. **Fresh global observations.** An intervening overlapping write affecting global 0x67 may change a later one-byte copy into adjusted offset 0x09. That destination is explicitly one byte; the global source read is not established as one byte.

10. **Capture stability and shifts.** Changes after terminal-offset-0x64 capture do not replace the second argument. Shift count 24 is within the 32-bit operand width; count 40 has no specified result here. Modulo-32 masking is not established.

## Uncertainties

The following remain **documented_only**:

- Application intent, meaningful success, and complete invocation protocol.
- Dependency identities, implicit inputs, calling conventions, and additional result-channel semantics.
- Preliminary per-site argument contracts and repetition multiplicity not enumerated here.
- Every named global’s storage width, signedness, declaration-dependent conversion, and affected byte range.
- Character-conversion behavior and the full value stored by the completion expression.
- Memory mapping, endianness, alignment, atomicity, allocation bounds, ownership, and lifetime.
- Signed overflow, signed right shifts, and out-of-range shift counts.
- Evaluation order among multiple loads within one expression.
- Concurrent, dependency-induced, and special-storage effects.
- Normal return from the terminal target.

Unknown named-global widths do not erase established operation ordering or alter explicitly typed computed-access widths.

## Evidence

The supplied representation supports the ordered interactions, explicit computed-access widths, capture boundaries, and terminal argument origins described here. Prior prose, comments, and reviewer feedback are not independent evidence.

Named-global assignments and observations lack storage declarations. Right-hand-side narrowing and 32-bit result captures establish expression or result properties, not underlying access widths. This qualification applies consistently to all named globals, including the publication-anchor source.

The completion region establishes the order of the 0xA9 observation, fourfold-traversal word capture, 0x3A assignment, retained-word capture, and adjusted-word capture without establishing the named globals’ access widths.

Arithmetic examples are deductions conditional on the stated model and memory assumptions. No application intent is inferred to fill missing behavior. No tools, independent implementation access, runtime observations, or tests were used.
