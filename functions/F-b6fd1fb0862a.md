<!-- Trusted provenance: F-b6fd1fb0862a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b6fd1fb0862a

## Purpose

F-b6fd1fb0862a performs a finite sequence of indirect dependency invocations, retains and transforms selected results, modifies fixed and computed memory locations, and ends with an unresolved indirect invocation carrying four explicit arguments.

No application-level purpose is established. Repeated offsets, self-referential writes, and constants corresponding to characters do not establish object construction, list operations, text output, serialization, or cleanup.

This contract separates:

- **Established static evidence:** represented accesses, widths, conversions, value provenance, capture boundaries, and sequencing.
- **Conditional interpretation:** consequences requiring an explicitly stated arithmetic or execution model.
- **documented_only:** details not resolved here.

The explicit arguments do not determine the complete behavior. Incoming execution context, dependency effects, secondary machine results, and memory contents also contribute.

## Interface

### Explicit arguments and result

| Element | Width | Established use |
|---|---|---|
| First argument | Opaque 32-bit value | Forwarded unchanged to the opening dependency. |
| Second argument | Signed 32-bit value | Forwarded unchanged and separately shifted left six bits for that invocation. |
| Return | No explicit value | No status, success indicator, or output-size result is represented. |

The body does not directly dereference either argument. Dependencies may interpret their values differently; their interfaces are unavailable.

### Implicit context and preconditions

The opening invocation target derives from incoming return-link context plus 0xFFE. Later chain targets generally derive from represented code-location values plus 0xFFE. The terminal target remains unresolved.

Reaching each subsequent effect requires all preceding dependencies to return and every reached target and memory access to be executable or accessible as appropriate. No validation of bounds, alignment, allocation extent, address range, or argument domain is represented.

Small fixed addresses are used directly. Their validity and meaning are **documented_only**, not grounds for inventing rejection behavior.

### Memory and arithmetic model

Explicit direct accesses distinguish 8-bit bytes, 16-bit halfwords, and 32-bit words. Narrow writes retain the corresponding low bits. Explicit unsigned-byte reads contribute numerical values from 0 through 255.

The global associated with address 0x05 is an exception: its value is explicitly converted to unsigned 32-bit form, but its original declaration, read width, and signedness are unavailable. It must not be described unconditionally as a zero-extended byte.

Represented 64-bit dependency results undergo unsigned numerical extraction after shifts of 32, 34, 36, and 44 bits. Whether these are scalar returns or combinations of machine results is unresolved.

Numerical examples assume **32-bit bit-vector arithmetic**, retaining the low 32 bits after overflowing word additions, multiplication, and left shifts. Signed expressions alone do not establish that execution model. Address wrapping, signed overflow, signed right shifts, and excessive shift counts remain target-dependent.

Offsets identify accesses, not verified object layouts. Computed regions may alias each other or fixed storage. A word at offset 0x14 overlaps a byte at offset 0x15. Byte order, alignment rules, atomicity, and external visibility are not established.

### Ownership

No direct allocation, release, or ownership-transfer primitive is identifiable. Dependencies are not assumed to preserve ownership, storage lifetime, global values, or mappings.

## Behavior

### Dependency invocation phase

The opening dependency receives the first argument, the second argument, and the second argument shifted left six bits, in that order. Two more invocations follow; the primary result of the latter is shifted left eight bits and passed onward.

The remaining finite chain combines primary results, secondary machine-result values, extracted high words, shifted low words, retained captures, repeated arguments, and explicit zeros. Some captures survive intervening invocations before reuse.

Represented argument relationships include:

- A low word shifted left eight bits alongside an unchanged high word.
- Independently shifted high words.
- Three- and four-argument forms containing repeated captures.
- Masked fields of 24, 22, 21, 20, 19, 18, 17, 16, 14, 13, 12, and 11 bits.
- Transformations of explicitly zero-valued captures.

These relationships do not establish an encoding. Invocations lacking explicit arguments may still consume implicit state. Neither such calls nor calls with zero arguments are established to be removable.

The exact per-invocation interfaces and secondary-result semantics remain **documented_only** in this contract. The represented chain is finite, not an input-controlled loop.

### Captures used by direct memory effects

The following descriptive terms distinguish independent values:

| Term | Definition |
|---|---|
| Shifted base | Low word of one dependency result shifted left 25 bits. |
| Original scale result | A separately retained, unscaled primary result. |
| Scaled base | Original scale result multiplied by 0x02000000. |
| Original shifted value | Original scale result shifted left 26 bits. |
| Field value | Another primary result shifted left eight bits, masked with 0x7FFF, then unsigned-shifted right six bits. |
| Small base | Twice the field value. |
| Link value | A separately captured primary result shifted left eight bits. |
| Final pair | Low and high words of the last represented 64-bit result before direct mutations. |

The original shifted value is derived from the **original scale result**, not from the scaled base.

### Initial publications and captures

The first direct word write places the shifted base at final-pair high word plus 0x14. Shifted base plus 9 then receives the high word’s low byte. Before that byte is overwritten with the shifted base’s low byte, a word containing the scaled base is written at 256 times final-pair low word plus 0x54.

Scaled base plus 0x14 is cleared before an unsigned-byte capture at scaled base plus 5. Small base plus 0x14 is then cleared, and the captured byte is published as a word at scaled base plus 0x34.

A word is next captured from scaled base plus 0x14. Small base plus 0x14 is cleared again, followed by two separate byte writes of that captured word’s low byte at small base plus 0x15. The captured word is not unconditionally zero: intervening computed writes may alias its source.

The global at 0x05 is then captured and converted to unsigned 32-bit form, producing the **global-derived base**. Fixed byte 0x01 receives the small base’s low byte, followed by a byte write of the field value at global-derived base plus 0x15.

### Early linked mutations

Scaled base plus 0x11 temporarily receives the link value’s low byte. The link value is then stored as a word at the earlier captured word plus 0x54, before scaled base plus 0x11 is cleared.

The word at global-derived base plus 0x54 is captured as the **saved word**. That location is subsequently overwritten with the earlier captured word. Small base plus 0x34 receives the field value.

A word captured at scaled base plus 0x74 becomes the **early link**. Early link plus 0x54 receives the original shifted value. A separate self-referential update reads small base plus 0x74 for both destination-link construction and stored value. Those are represented memory reads, not an established single capture; their cross-operand evaluation order is unresolved.

Saved word plus 0x24 then receives the saved word, early link plus 9 receives its low byte, and early link plus 0x54 receives the link value. The saved-word self-store and early-link byte store repeat.

Small base plus 0x14 next receives the original shifted value. Fixed word 0x54 receives the saved word shifted left one bit. Small base plus 0x14 then receives the saved word twice.

Two separate byte writes at small base plus 0x15 contain the low byte of four times the saved word. Fixed byte 0x15 subsequently receives the small base’s low byte.

### Intermediate linked state

Define the **quadrupled base** as four times the saved word and the **eightfold value** as eight times the saved word. These retained arithmetic values are not reloads from locations subsequently modified.

The first word write at quadrupled base plus 0x54 contains the link value. **Before that location receives the quadrupled base**, small base plus 0x34 receives the saved word shifted left two bits, and the word at small base plus 0x74 is read and doubled. This doubled capture is the **doubled link**. Thus, an alias between small base plus 0x74 and quadrupled base plus 0x54 exposes the earlier link-value publication to this read.

After the doubled-link capture, quadrupled base plus 0x54 receives the quadrupled base. A byte containing the eightfold value’s low bits is then written at quadrupled base plus 0x15. Only afterward does saved word plus 0x44 receive the doubled link.

The next dependent lookup first reads an unsigned byte at quadrupled base plus 5, then reads a word at that byte’s numerical value plus 0x74. Call the resulting word the **byte-derived link**. Its offset-0x44 word receives the doubled link. These reads occur after the preceding offset-0x15 byte write and saved-word offset-0x44 word write; they cannot be moved ahead of them without additional aliasing guarantees.

Quadrupled base plus 0x15 is then overwritten with the quadrupled base’s low byte. Its offset-0x0D byte receives the byte-derived link’s low byte, and its offset-0x14 word receives the quadrupled base. Next, its offset-0x54 word receives the saved word shifted left two bits. **That offset-0x54 publication precedes the word capture at quadrupled base plus 0x64.**

The offset-0x64 capture becomes another base. Its offset-0x0D byte receives the byte-derived link’s low byte, followed by a self-referential word write at its offset 0x14. A subsequent word read at quadrupled base plus 4 supplies the **further base**.

Doubled link plus 9 then receives the byte-derived link’s low byte, and doubled link plus 0x44 receives the eightfold value. An unsigned byte is captured at further base plus 5. Quadrupled base plus 9 receives the doubled link’s low byte before the captured byte is published as a word at byte-derived link plus 0x44.

Quadrupled base plus 0x14 next receives the eightfold value. Doubled link plus 0x15 receives the further base’s low byte, and doubled link plus 0x44 receives the quadrupled base.

A **second, fresh unsigned-byte read** occurs at further base plus 5. Quadrupled base plus 0x15 receives the eightfold value’s low byte before a dependent word is read at this second byte’s numerical value plus 0x74. That word is the **publication link**.

Finally, quadrupled base plus 0x15 receives the further base’s low byte; its offset-0x54 word receives the saved word shifted left two bits; its offset-9 byte receives the low byte of twice the publication link; and its offset-0x14 word receives the eightfold value. Constant publication follows these effects.

### Constants and repeated publication

Define:

- **Terminal base:** twice the publication link.
- **Adjusted base:** quadrupled base minus 0x96.
- **Work base:** four times the adjusted base.

Numerical 0x3A is published, in order, as a byte at four times the publication link, a halfword at eight times that link, and a word at four times that link plus 0x20.

Work base plus 0x20 receives successive words containing 0x20 and 0x2F. An unsigned byte is then read at work base plus 4 and stored as a word at work base plus 0x20. Another word containing 0x20 follows there.

Only afterward is the **publication base** captured from fixed word location 0x087FBBBC. Fixed word 0x60 next receives 0x20.

Each subsequent publication group preserves this internal order:

1. A word containing four times the publication link is published at a scaled destination plus 0x54.
2. Publication base plus 0x19 receives the work base’s low byte.
3. Work base plus 0x11 receives the publication base’s low byte.
4. Publication base plus 0x34 receives the scaled destination value.

The first scaled destination is eight times the saved word; later groups use eight times the adjusted base. The publication base and both byte values remain captures even if writes overlap their original sources.

Repetition is finite and explicit. Exact multiplicity is **documented_only** here; no unbounded traversal is implied.

### Fixed-state captures and later updates

After publication, a word is captured at fixed address 0xA9, followed by a word at four times the publication link plus 0x74. These are the **fixed-state word** and **publication-link word**.

Fixed byte 0x3A then receives the low eight bits of eight times the adjusted base’s low-byte value plus 0x30. Ordinary signed-byte versus unsigned-byte promotion does not change this calculation’s final low eight bits.

Only after that write are words captured at terminal base plus 0x14 and work base plus 4: the **saved terminal word** and **saved work word**.

Terminal base plus 0x0D receives the work base’s low byte. Work base plus 8 receives 0x6C, followed by writes of the captured publication base’s low byte at work base plus 0x0D and plus 9.

Terminal base plus 0x54 receives the terminal base. Fixed byte 0x67 receives the fixed-state word’s low byte. A word is then captured at terminal base plus 0x44, before the represented fixed-byte value at 0x67 is copied to fixed byte 0x21.

That offset-0x44 capture supplies a byte at work base plus 0x10 and a word at publication base plus 0x54. Its signed-right-shifted value supplies a byte at terminal base plus 1. The shift count is the work base’s low byte, potentially 0–255; excessive-count behavior remains unresolved. Publication base plus 0x54 is then overwritten with the work base.

A word captured at work base plus 0x40 precedes publication of the saved work word at publication base plus 0x74. The offset-0x40 capture is stored as a word at work base plus 0x10, followed by a word containing the terminal base at publication base plus 0x44.

Two later writes at work base plus 9 each use a **fresh read** of fixed byte 0x67. Between those reads, terminal base plus 9 receives the work base’s low byte, work base plus 0x44 receives the fixed-state word, and a word is captured at work base plus 0x54. Aliasing prevents substituting the originally published byte for either fresh read.

### Terminal updates and invocation

The work-base offset-0x54 capture becomes the **further update base**. Its offset-0x0D byte receives the publication-link word’s low byte. Terminal base plus 0x54 is restored from the saved terminal word, followed by a word capture at work base plus 0x14.

Terminal base plus 0x11 receives the publication-link word’s low byte. Work base plus 9 receives the further update base’s low byte, and work base plus 0x44 receives that base as a word.

A word at terminal base plus 4 supplies the **terminal object base**. A halfword containing 0x65 is written at twice that base before a word is captured at terminal object base plus 0x64.

Further update base plus 0x11 then receives the earlier work-base offset-0x14 capture’s low byte. Further update base plus 0x54 receives decimal 100. A word is next captured at terminal object base plus 0x74, followed by a word at terminal base plus 0x64, the **last link**.

Terminal base plus 9 receives the terminal object base’s low byte. Last link plus 0x54 receives the terminal-object offset-0x64 capture. Twice that capture then receives the last link as a word.

The terminal dependency receives, in order:

1. 0x72.
2. The terminal-object offset-0x64 capture.
3. The saved terminal word.
4. The terminal-object offset-0x74 capture.

A return is represented afterward. Actual return behavior depends on the unresolved dependency.

## Side effects

Direct fixed-address writes occur at 0x01, 0x15, 0x21, 0x3A, 0x54, 0x60, and 0x67. Computed writes may overlap these locations, the publication-base source, or other computed regions.

Effects are non-transactional. No explicit rollback, fault handler, validation failure, or error-reporting path is represented. A fault or nonreturning dependency can prevent later effects without reversing completed ones.

Represented sequencing does not establish volatile access, synchronization, atomicity, or external observation of every intermediate value. It also does not justify eliminating duplicate accesses or replacing fresh reads with retained captures.

## Dependencies

The following document-scoped opaque IDs identify roles, not resolved implementations.

| ID | Role | Unresolved behavior |
|---|---|---|
| F-unknown-entry | Opening context-derived dependency | Full interface, results, effects, preservation, and return behavior. |
| F-unknown-chain | Family of subsequent indirect dependencies; not necessarily one callee | Individual identities, implicit inputs, secondary results, and resource effects. |
| F-unknown-terminal | Final four-argument dependency | Target, argument meanings, effects, and return behavior. |

No dependency is assumed to preserve globals, memory mappings, ownership, or storage lifetime.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Opening arguments and arithmetic provenance

For explicit arguments 0x12345678 and 3, the opening arguments are 0x12345678, 3, and 192. Under 32-bit bit-vector shifting, second-argument pattern 0x04000000 produces zero in the third position.

An original scale result of 1 produces scaled base 0x02000000 and original shifted value 0x04000000. The early-link offset-0x54 publication and the small-base offset-0x14 publication immediately before fixed word 0x54 both use 0x04000000. Shifting the already-scaled base by 26 would produce zero and violate the established provenance.

### Intermediate alias-sensitive ordering

Assume the intermediate phase is reached with saved word 8, small base 0, link value 0x11223344, accessible storage, and bit-vector arithmetic. Quadrupled base is 0x20, making its offset-0x54 location identical to small base plus 0x74.

The doubled-link capture therefore reads 0x11223344 and becomes 0x22446688 before that shared location is overwritten with 0x20. Grouping both offset-0x54 writes ahead of the read would instead produce 0x40 and violate the required order.

The byte publication of the eightfold value at quadrupled base plus 0x15 must precede the saved-word offset-0x44 publication and the subsequent unsigned-byte lookup. Likewise, the later offset-0x54 publication of the saved word shifted left two bits must precede the offset-0x64 capture.

### Typed reads and retained captures

An explicit unsigned-byte read of 0xFE supplies numerical base 254. For the undeclared global at 0x05, a signed 8-bit declaration holding −2 would instead convert to unsigned 32-bit value 0xFFFFFFFE; an unsigned 8-bit declaration would yield 254. Neither declaration is established.

If a computed write modifies 0x087FBBBC after publication-base capture, later groups still use the capture. In contrast, a change to fixed byte 0x67 between its later reads can change the second supplied byte.

### Fixed writes and termination

For saved word 0x1234, fixed word 0x54 receives 0x2468. For small base 0x120, the fixed byte publication at 0x15 contains 0x20.

If the adjusted base’s low byte is 0xFE, fixed byte 0x3A receives 0x20 under low-eight-bit arithmetic.

A fault after completed writes has no established rollback. If the terminal invocation is reached, its four arguments use the specified captures, not fresh reloads.

## Uncertainties

The following remain **documented_only**:

- Application intent, valid argument domains, and object layouts.
- Dependency identities, calling convention, implicit inputs, and secondary-result semantics.
- The original width and signedness of the global at 0x05.
- Scalar versus paired interpretation of represented 64-bit results.
- Byte order, alignment restrictions, fault behavior, synchronization, and visibility.
- Signed overflow, address wrapping, signed-right-shift semantics, and excessive shift counts.
- Cross-operand evaluation order in the self-referential update using multiple reads.
- Exact invocation-chain interfaces and publication-group multiplicity.
- Meaning of small fixed addresses and actual terminal return behavior.

These limitations prevent an unconditional success postcondition. They do not weaken the established interleaving of intermediate reads and writes.

## Evidence

The supplied static representation supports the access widths, arithmetic provenance, capture boundaries, and sequencing described above. Prior prose, comments, and reviewer feedback are not independent evidence.

In particular, the intermediate doubled-link capture lies between the link-value publication and the self-publication at quadrupled base plus 0x54. The eightfold-value byte publication precedes the subsequent dependent lookup. The later shifted-saved-word publication at offset 0x54 precedes the offset-0x64 capture.

No runtime tests have been performed. Dependency implementations, execution traces, verified calling-convention information, and independent memory-layout declarations were not available.
